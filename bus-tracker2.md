# Suceava Bus Tracker (PWA)

## 1. Problem and user
- **Who**: Suceava commuters who take TPL Suceava buses.
- **Pain**: they don't know when the bus arrives, so they wait at the stop or miss it.
- **Today**: TPL's own map app shows raw vehicle positions only. No ETAs, no alerts, no favorites.

## 2. Core loop
**Commute plans.** The user sets up a plan once: "be at work by 08:00, board at stop X, get off at stop Y, weekdays". The backend picks the best bus every day and pushes "get ready" and "leave now" at the right time. The user does nothing after setup.

Main endpoints: `POST /plans/preview` (dry run), `GET /plans/{id}/today` (today's computed result).
Secondary: `GET /stops/{id}/arrivals` for ad-hoc "when is the next bus".

### Commute plan: inputs
- **Destination deadline**: arrive by `HH:MM` (at the destination, not the stop).
- **Board stop** and **alight stop** (picked on the map or by search).
- **Walk to board stop**: minutes. Manual input, or estimated from a home GPS point (see below).
- **Walk from alight stop** to the destination: same, manual or estimated.
- **Safety buffer**: minutes of slack (default 3), plus a risk level: normal (plan on median ride time) or safe (plan on p80).
- **Allowed lines**: all lines that serve both stops, or a restricted list.
- **Schedule**, flexible:
  - weekdays (Mon-Fri), or any custom set of weekdays
  - one-off date and time (e.g. next Saturday to the train station)
  - skip a single date, pause the plan, delete the plan
- **Notifications**: "get ready" N min before leaving (default 10), "leave now", optional "bus is late, new leave time".

### Commute plan: what the backend computes each run
1. Activation: a scheduler marks the plan active for the day, and starts live evaluation ~45 min before the planned leave time. Before that, nothing is polled for this plan (cheap).
2. Candidates: every live vehicle on an allowed line heading from the board stop towards the alight stop, plus the next expected one by headway.
3. For each candidate: predicted time at the board stop, predicted time at the alight stop (live ETA plus learned ride time for that hour and day type), then add walk from alight stop.
4. Pick the latest bus that still reaches the destination by `deadline - buffer`. Keep the one before it as a backup.
5. `leave_at = board_time - walk_to_stop - small margin`. `prepare_at = leave_at - prepare_minutes`.
6. Re-evaluate every cycle. If the chosen bus is delayed or disappears, switch to the backup and send an updated notification ("take the earlier bus, leave now").
7. Store the result as a `PlanRun` row (chosen vehicle, predicted times, actual outcome later) for debugging and ML training.

### Walk-time estimate (when the user doesn't type it)
- v1: straight-line distance x 1.3 / 1.25 m/s. Good enough.
- v2: a self-hosted OSRM or Valhalla with the foot profile on the Romania OSM extract. Exact walking time between GPS points. Cache per (point, stop).
- The user can always override with a manual number.

## 3. Data source
Upstream: `https://info.tplsv.ro:7443/?ajax=get_bus_data&t=<ms>` (unofficial, no auth).

Response shape:
```
{"thoreb": "<json string>", "karsan": ""}
inner: { "<vehicleId>": {"1":"on|off", "2":"lat", "3":"lon", "4":"line"} }
```
- `4` can be `M114`, `NO_LINE`, `NO_LINE`-like, or empty (off route).
- Some ids are numeric (`0464`), some are plates (`SV67PMS`).
- Lat/lon `0.00000` = no GPS fix, discard.
- Response is ~8.7 kB, the official app polls every ~1.3 s.

**No GTFS exists.** Stops and routes must be built by us:
1. Only `get_bus_data` exists. No lines, stops or timetable endpoints.
2. Pull bus stops and bus route relations for Suceava from OpenStreetMap (Overpass API). Fix gaps by hand.
3. Fallback: learn stops and routes from our own recorded traces (cluster the points where buses dwell, build route polylines per line from history).
4. Email TPL Suceava for official stops/timetables and permission to use the feed.

## 4. Entities
- **User**: id, email, password hash, push subscriptions, locale
- **Line**: code (`M114`), name, color, active
- **Stop**: id, name, lat/lon (PostGIS point), source (osm/learned/manual)
- **LineStop**: line, stop, direction, sequence
- **RouteShape**: line, direction, polyline
- **Place**: user, label (home/work/other), lat/lon
- **CommutePlan**: user, name, origin place, destination place, board stop, alight stop, arrive_by, walk_to_stop_min (nullable = estimate), walk_from_stop_min (nullable = estimate), buffer_min, risk (normal/safe), prepare_min, allowed lines, active
- **PlanSchedule**: plan, weekdays bitmask or one-off date, skip dates, timezone (Europe/Bucharest)
- **PlanRun**: plan, date, chosen vehicle, predicted board/alight times, leave_at, backup vehicle, status (pending / notified / done / missed), actual board/alight times (from positions)
- **Favorite**: user, stop, line (nullable) for quick access
- **SegmentStat**: line, from stop, to stop, day type (weekday/sat/sun/holiday), hour bucket, n samples, median seconds, p80 seconds
- **Prediction**: model version, line, segment, predicted seconds, actual seconds, ts (for model evaluation)
- **Vehicle**: id, plate, last known line
- **PositionLog**: vehicle, line (inferred), lat/lon, ts (partitioned by day)
- **Report**: user, vehicle/line/stop, type (ghost / crowded / delayed), ts
- **Notification**: user, rule, sent_at, status

## 5. API surface
Django/DRF (writes, users):
- `POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`
- `GET/PATCH /me`
- `GET/POST/DELETE /favorites`
- `GET/POST/PATCH/DELETE /places`
- `GET/POST/PATCH/DELETE /plans` (includes schedule)
- `POST /plans/{id}/skip?date=`, `POST /plans/{id}/pause`, `POST /plans/{id}/resume`
- `POST /push/subscribe`, `DELETE /push/subscribe`
- `POST /reports`
- Django admin: lines, stops, shapes, reports moderation

FastAPI (live, read-heavy):
- `GET /vehicles` (latest positions, optional `?line=`)
- `WS /live` or `GET /live` (SSE): position stream
- `GET /lines`, `GET /lines/{code}` (shape + stops)
- `GET /stops`, `GET /stops/nearby?lat&lon`
- `GET /stops/{id}/arrivals` (next buses + ETA)
- `POST /plans/preview` (dry run: board, alight, arrive_by, walks, date -> options with leave time, bus, confidence)
- `GET /plans/{id}/today` (live computed result for the plan)
- `GET /lines/{code}/stats` (on-time %, by hour)
- `GET /segments/stats?line=&from=&to=` (learned ride times)
- `GET /health`

## 6. Architecture
- **Poller** (FastAPI process or arq worker): fetch upstream every 2-3 s, parse the nested JSON string, drop bad fixes, infer missing lines, write latest state to Redis, append to PositionLog every ~15 s.
- **Redis**: latest positions, pub/sub for live stream, ETA cache, job queue.
- **ETA engine**: snap vehicle to route shape, distance along shape to stop / average speed from history for that segment and hour.
- **Plan scheduler**: arq/Celery cron. Every minute, find plans whose evaluation window opens now (schedule + timezone + skip dates), create the `PlanRun`. Only active windows are evaluated each cycle.
- **Plan evaluator**: described in section 2. Pure function `(plan, live state, SegmentStat) -> options`, so it is unit-testable and reused by `/plans/preview`.
- **Notifier**: sends Web Push once per (plan run, stage: prepare / leave / update). Dedupe key `plan_run+stage`. Stores a `Notification` row.
- **Learning job** (nightly): turns `PositionLog` into per-segment travel times, refreshes `SegmentStat`, fills `Prediction.actual`, also measures plan outcomes (did the user's chosen bus actually arrive on time).

### Learned speed / ETA model (the "ML" part)
- **Phase 1, no ML**: `SegmentStat` medians and p80 by (line, segment, day type, hour bucket). Hour buckets of 15 min. Needs ~2 weeks of logs. This alone captures "7:00-8:00 is slow on line X".
- **Phase 2, ML**: gradient boosted trees (LightGBM) predicting segment travel time. Features: line, segment, hour, weekday, school holiday, weather (free API), current speed of the previous vehicle on the same segment. Train nightly, compare against Phase 1 on `Prediction` rows, and only ship if error is lower.
- **Output**: always a range (p50 and p80), so "safe" mode can plan on p80.
- **Data needs**: positions every ~15 s per vehicle. Segment times come from where the vehicle was nearest each stop on the route shape.
- **Postgres + PostGIS**: shared DB. Django owns schema and migrations. FastAPI reads the user/rule tables and writes only PositionLog.
- **Frontend**: PWA (React or Svelte + Vite), MapLibre GL with OSM tiles, service worker, Web Push (VAPID), installable.
- **Auth**: Django issues JWT, FastAPI verifies it.
- **Deploy**: Docker Compose (django, fastapi, poller, postgres, redis, nginx), one VPS to start.

## 7. Features
**MVP (v1 ships)**
- Register/login
- Commute plans: create, schedule (weekdays / one-off / skip / pause)
- Plan evaluator with manual walk times, "prepare" and "leave now" web push
- Stops + arrivals with ETA
- Live map with vehicles filtered by line
- Installable PWA

**Later**
- Walk-time estimate from GPS (OSRM/Valhalla)
- Learned ride times (Phase 1 stats, then Phase 2 ML)
- "Bus late, new leave time" updates and backup bus switching
- Plan outcome stats ("your bus was on time 18 of 20 days")
- Ghost bus / crowded reports
- Journey planner A to B (one transfer)
- Service-change alerts and offline timetable
- Public API / B2B widget for local businesses


## 8. Stack choice
- Django/DRF: users, admin, CRUD, billing, moderation.
- FastAPI: poller, live stream, ETA, alerts.
- Postgres+PostGIS, Redis. Python 3.12.
- Reason for both: Django gives auth/admin/migrations quickly; FastAPI handles the async, high-frequency, read-heavy live path.

## 9. Risks (top 5)
1. **Upstream is unofficial**: may change or block. Mitigation: single poller, adapter layer, email TPL for permission.
2. **No stops/routes**: ETAs impossible without them. Mitigation: OSM first, learned fallback, test in week 1.
3. **Missing/wrong line on vehicles** (`NO_LINE`): wrong ETAs. Mitigation: infer line from route shape matching.
4. **Push on iOS**: needs iOS 16.4+ and the PWA installed to the home screen. Mitigation: in-app install prompt, email fallback.
5. **ETA accuracy**: bad ETAs kill trust. Mitigation: show "approx", log predicted vs actual, tune.
6. **Plans before the bus is live**: at 06:30 the 07:40 bus may not be on the road yet, so there is no position. Mitigation: use learned headway and typical time at the stop for the early "prepare" estimate, and switch to live data once the vehicle appears.
7. **Long notification lead times vs. delays**: a "leave in 10 min" push may be wrong 5 min later. Mitigation: send the update notification only on a change above a threshold (e.g. 2 min), and never send more than one update per 5 min.
8. **Direction detection**: the feed has no direction field. Mitigation: infer from the position sequence on the route shape (moving towards the alight stop, not away).

## 10. First week
1. Day 1: log raw responses for 24 h (script saving one response per 10 s).
2. Day 2: pull Suceava bus stops and route relations from Overpass, view on a map.
3. Day 3: poller + Redis + `GET /vehicles`.
4. Day 4-5: minimal map PWA showing live vehicles.
