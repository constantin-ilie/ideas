# Team Chat (mini Slack)

## The problem
Teams talk in too many places: email, WhatsApp groups, ticket comments. Information gets lost and people who were away don't know what was decided. Slack and Teams fix this, but they are paid, hosted outside the company and hard to extend with our own integrations.

## The solution
- a team has public and private channels, plus direct messages.
- messages arrive live, in the same order for everyone, on every open tab.
- unread counts, @mentions and notifications show the user what needs attention.
- messages can be edited, deleted, reacted to and pinned. Files and images can be shared.

## Other features
- `@assistant` AI bot: answers questions, "catch me up" on unread messages, summarizes a thread
- threads
- link previews
- email digest of unread mentions
- voice/video calls

## How it works
Every client keeps a WebSocket open to a gateway. When a message is sent, the server gives it the next number in that channel and saves it, so everyone sees the same order. Then it publishes it on Redis and every gateway forwards it to the users connected to it, so we can run more than one gateway. Edits, deletes, reactions and pins travel the same way. Each user keeps one "last read" number per channel, which gives the unread count. After a disconnect the client asks for everything after its last number, so nothing is lost. Push notifications go out only for mentions and DMs.

## Stack
- Django (users, team, channels, memberships, files, admin)
- FastAPI (WebSocket gateway, messages, unread counts, notifications)
- Postgres, Redis, MinIO (files)
- React web app

## MVP
- login, one team, invite users
- public/private channels and direct messages
- live messages, same order for everyone, no duplicates
- edit, delete, reactions, pinned messages
- file and image uploads
- unread counts and @mentions
- browser push notifications, mute per channel
- online status and "is typing"
- search in messages

## Risks
- the frontend is generated with AI, but the sync logic (reconnect, missed messages, unread counts) must be reviewed and tested by us.
- messages sent at the same time or retried can break the order or get duplicated. The order is set in the database.
