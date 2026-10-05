# Team Chat: MVP plan

Assumptions: two developers, full time, focused days. The frontend is generated with AI, and the sync layer is reviewed by hand. Idea doc: [team-chat.md](team-chat.md).

## Goal
One team chats in real time in public channels, private channels and DMs. Every message shows in the same order for everyone, with no duplicates. Messages are not lost after a disconnect. Unread counts, mentions and push notifications show what needs attention.

**Done means:** 10 people use it for a week instead of WhatsApp, and nobody reports a lost or out-of-order message.

## MVP features
1. Login, one team, invite by link/email
2. Public channels, private channels, DMs (1:1 and group)
3. Live messages: same order for everyone, no duplicates, nothing lost on reconnect
4. Edit, delete, reactions, pins
5. File and image uploads
6. Unread counts and @mentions
7. Browser push notifications, per-channel mute
8. Online status and "is typing"
9. Search in messages

**Not in MVP:** threads, `@assistant`, link previews, email digest, voice/video, custom emoji, multiple teams, SSO, mobile app.

## Architecture

```
React ──REST──► Django (chat-core)      users, team, channels, files
  │   ──REST──► FastAPI (chat-realtime) messages, reactions, pins, read state, search
  └───WS─────► FastAPI gateway(s) ◄── Redis pub/sub ◄── FastAPI writes
Postgres: core DB (Django) + chat DB (FastAPI)  ·  MinIO: files
```

The design rules:
- **Writes go over REST, reads come over WS.** A POST request gets a clear answer: success, error or retry. The WebSocket only pushes events, typing and presence.
- **Django signs JWTs (RS256). FastAPI checks them with the public key.** There are no calls between the services on each request.
- **FastAPI needs to know memberships.** It calls Django `GET /internal/users/{id}/channels` and caches the answer in Redis. Django publishes `membership.changed` on Redis, and FastAPI clears that user's cache when it arrives.
- **Order comes from the database.** Each channel has a counter. Every change gets the next `seq` inside the same transaction that saves it.

## Django: chat-core

### Models
| Model | Fields (main) |
|---|---|
| `User` | email, display_name, avatar_file, timezone |
| `Team` | name, icon (one row in MVP) |
| `Invite` | token, email (optional), created_by, expires_at, used_at |
| `Channel` | name, kind (`public`/`private`/`dm`), topic, created_by, archived_at |
| `Membership` | channel, user, role (`owner`/`member`), muted, notify_level (`all`/`mentions`/`none`), joined_at |
| `File` | owner, object_key, filename, mime, size, status (`pending`/`ready`) |
| `PushSubscription` | user, endpoint, p256dh, auth |

### Endpoints
- Auth: `POST /auth/login`, `/auth/refresh`, `/auth/logout`, `GET /me`, `PATCH /me`
- Invites: `POST /invites`, `GET /invites/{token}`, `POST /invites/{token}/accept` (creates the user)
- Users: `GET /users` (team directory)
- Channels: `GET /channels` (mine plus public ones), `POST /channels`, `PATCH /channels/{id}`, `POST /channels/{id}/join`, `/leave`, `/members` (add/remove)
- DMs: `POST /dms {user_ids}`, which returns the existing DM if one already exists
- Membership settings: `PATCH /channels/{id}/me {muted, notify_level}`
- Files: `POST /files` (returns a presigned PUT URL), `POST /files/{id}/complete`, `GET /files/{id}` (returns a presigned GET URL)
- Push: `POST /push/subscriptions`, `DELETE /push/subscriptions/{id}`
- Internal: `GET /internal/users/{id}/channels`, `GET /internal/channels/{id}/members` (protected by a service token)
- Django admin for users, channels and invites

## FastAPI: chat-realtime

### Tables
| Table | Purpose |
|---|---|
| `channel_counters(channel_id PK, last_seq)` | Next number per channel |
| `channel_events(channel_id, seq, type, payload jsonb, created_at)` PK `(channel_id, seq)` | Log of every change, used for resync |
| `messages(id, channel_id, seq, author_id, body, file_ids[], client_msg_id, edited_at, deleted_at, created_at, tsv)` | Current state; unique `(author_id, client_msg_id)`; GIN on `tsv` |
| `reactions(message_id, user_id, emoji)` PK all three | |
| `pins(channel_id, message_id, pinned_by, pinned_at)` | |
| `mentions(message_id, user_id, channel_id, seq)` | Mention badges and push |
| `read_state(user_id, channel_id, last_read_seq)` | Unread counts |

### Write path (every change works the same way)
```
BEGIN
  UPDATE channel_counters SET last_seq = last_seq + 1 WHERE channel_id = $1 RETURNING last_seq   -- row lock = per-channel order
  INSERT INTO channel_events (...)  /  apply change to messages/reactions/pins
COMMIT
PUBLISH ch:{channel_id} {event}
```
- **Idempotency:** the client sends a `client_msg_id` (UUID). If that ID was already sent, the server returns the existing message and does not insert a new one.
- **Event types:** `message.created`, `message.edited`, `message.deleted`, `reaction.added`, `reaction.removed`, `pin.added`, `pin.removed`

### REST endpoints
- `POST /channels/{id}/messages {client_msg_id, body, file_ids}`
- `PATCH /messages/{id}`, `DELETE /messages/{id}` (soft delete)
- `GET /channels/{id}/messages?before_seq=&limit=50`: history, scrolling up
- `GET /channels/{id}/events?after_seq=N&limit=500`: resync. Returns `gap_too_large` if there are more than 500 events, and the client then reloads the latest page.
- `PUT /messages/{id}/reactions/{emoji}`, `DELETE ...`
- `POST /channels/{id}/pins/{message_id}`, `DELETE ...`, `GET /channels/{id}/pins`
- `POST /channels/{id}/read {seq}`: only moves forward
- `GET /unread`: returns `[{channel_id, unread, mentions}]` for all my channels
- `GET /search?q=&in=&from=&before=&after=`: Postgres full-text search, limited to my channels

### WebSocket gateway
- `GET /ws?token=...`. On connect it loads the user's channels and subscribes to the Redis channels `ch:{id}`. One subscription per gateway is shared by all users on that gateway, with ref counting.
- Server → client: `event` (from the log, with seq), `typing`, `presence`, `unread` (count changes)
- Client → server: `typing {channel_id}` (throttled to 1 every 3 s), `heartbeat` (every 20 s, includes `visible: true/false`)
- On `membership.changed` the gateway subscribes or unsubscribes live.
- Several gateways run behind a load balancer. They share no state except Redis.

### Presence, typing, push
- **Presence:** the Redis key `presence:{user}` has a 60 s TTL and is refreshed by heartbeat. Online/offline changes are broadcast to the team.
- **Typing:** published only and never stored. The client hides it after 5 s.
- **Push:** a background task runs after `message.created`. It finds DM members and mentioned users and skips muted users, `notify_level` blocks and users with a visible tab. It sends through `pywebpush` (VAPID). It loads subscriptions from Django's internal API.

## Frontend: React (Vite + TS)
- **Generated with AI:** layout, sidebar, channel view, composer, modals, settings, search page
- **Written and reviewed by hand (the risky part):**
  - `ws-client.ts`: connect, reconnect with backoff, heartbeat
  - `sync-store.ts`: messages per channel kept by `seq`, events applied in order, resync after reconnect, deduplication by `client_msg_id`, optimistic send with a "pending/failed/retry" state
  - `unread.ts`: counts from `/unread` plus live events, mark as read when the channel is visible
- TanStack Query for REST and Zustand for the sync store.

## Team split
- **Dev A: realtime.** FastAPI messages, gateway, Redis, the sync layer in the frontend (`ws-client.ts`, `sync-store.ts`, `unread.ts`), push sending, search backend.
- **Dev B: core + UI.** Docker setup, Django (auth, invites, channels, files, push subscriptions), the AI-generated React UI, service worker, deploy.
- **Contract first.** On days 1 and 2 both developers write the OpenAPI specs for both services and the WS event schema (`docs/events.md`). After that, each developer builds against mocks of the other's side.
- **Cross-review.** Dev B reviews all of Dev A's sync code, and Dev A reviews Django permissions (who can see which channel).

## Build order and estimates

Total work is about 38 days. Split between two developers that is about **20 days each**, and the two tracks run in parallel.

| Calendar days | Dev A (realtime) | Dev B (core + UI) |
|---|---|---|
| 1–2 | **Together:** API contract, event schema, docker-compose, JWT keys, CI | |
| 3–6 | Messages backend: seq, idempotency, history, events-after-N, concurrency tests | Django core: login, invites, channels, DMs, memberships, admin, internal API |
| 7–10 | Gateway: WS auth, Redis fan-out, 2 gateways, live membership changes | UI shell: login, sidebar, channel view, composer (against mocks) |
| 11–13 | `ws-client` + `sync-store`: reconnect, resync, optimistic send | Wire UI to real APIs, edit/delete, reactions, pins UI |
| | **Milestone 1 (about day 13): two people chat live. Killing a gateway loses nothing.** | |
| 14–16 | Unread + mentions backend, `unread.ts`, presence + typing backend | Files: presigned upload, progress, images inline; mention autocomplete, badges |
| 17–19 | Push sender (VAPID, notify levels, mute), search backend | Presence/typing UI, service worker, mute settings, search UI |
| | **Milestone 2 (about day 19): feature-complete MVP** | |
| 20–22 | Load test, 2-gateway test, sync bug fixes | Deploy to one server, backups, UI bug fixes |
| | **MVP done (about day 22)** | |

- **Full time:** about 22 working days ≈ **4.5 weeks**. Add about 15% for coordination between two people and 30% buffer: about **6 to 7 weeks**.
- **Half time (about 4 h/day each):** about **3 months**.
- **Dev A's track is the critical path** (messages, then gateway, then sync store). If it takes longer, Dev B picks up the search backend and push sender. Cut search before cutting tests.

## Tests that must exist (sync logic)
1. 50 concurrent sends to one channel give seq 1..50 with no gaps and no duplicates.
2. Sending the same `client_msg_id` twice gives one message.
3. Client misses events while offline, reconnects and gets exactly the missing events in order.
4. With 2 gateways, a user on gateway A receives a message sent by a user on gateway B.
5. The unread count is correct after: new message, own message, deleted message, read on another tab.
6. A user removed from a private channel stops getting its events right away.

## Risks
- **AI-generated frontend:** the sync store must be written or reviewed by hand and covered by the tests above.
- **Membership cache becomes stale:** a removed user could still see messages. Fix: invalidate the cache on `membership.changed`, plus a 5-minute TTL as a backstop.
- **One busy channel slows down:** the row lock on `channel_counters` makes sends in one channel run one at a time. This is fine at MVP scale, which is hundreds of messages per second per channel.
- **Redis pub/sub drops a message while a gateway restarts:** clients resync from `channel_events`, so this is safe by design.

## After MVP (next order)
1. Threads, replies/quotes, markdown + code blocks, `Ctrl+K`
2. Custom status, DND, `@here`, user groups
3. Slash commands, reminders, scheduled messages, polls
4. Roles, guests, audit log, webhooks/bot API
5. `@assistant`, then voice via Jitsi/LiveKit
