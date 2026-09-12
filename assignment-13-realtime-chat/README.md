# Real-Time Group Chat & Messaging Engine

> A multi-room, real-time chat server with private messaging, live typing indicators, per-room presence, and message-history replay — built on **Node.js + Express + Socket.io** with a zero-dependency dark-theme browser client.

<p align="left">
  <img alt="Node.js" src="https://img.shields.io/badge/Node.js-18%2B-339933?logo=node.js&logoColor=white" />
  <img alt="Express" src="https://img.shields.io/badge/Express-4.19-000000?logo=express&logoColor=white" />
  <img alt="Socket.io" src="https://img.shields.io/badge/Socket.io-4.7-010101?logo=socket.io&logoColor=white" />
  <img alt="License" src="https://img.shields.io/badge/License-MIT-blue" />
</p>

---

## Overview

This project is a self-contained real-time messaging engine. Users pick a username (and an optional emoji avatar), enter a room, and exchange messages that are broadcast instantly to everyone in that room. Messages stay isolated to their room, private direct messages are routed only to the target socket, and each room shows a live roster of who is online. All runtime state is held **in memory** — there is no database — which keeps the project easy to run and easy to reason about.

### Features

- **Multi-room chat** — three seeded rooms (`#general`, `#developers`, `#random`) with instant switching from the sidebar.
- **Real-time broadcast** — a message is pushed to every member of a room the moment it is sent.
- **Room isolation** — a message sent to one room never leaks into another (covered by an integration test).
- **Direct (private) messaging** — click a user in the roster to DM them; the message is delivered only to that user's socket, never to the room.
- **Typing indicators** — with a **server-side 3-second auto-stop** timer and an additional **client-side 1.5-second debounce**, so a disconnecting client never leaves a stuck "typing…" state.
- **Per-room presence roster** — a live online-user list per room, recomputed on every join, leave, and disconnect.
- **In-memory history replay** — a user joining a room instantly receives the last **50** messages of that room.
- **Dark-theme browser UI** — a responsive three-column layout (rooms · chat · online users) with chat bubbles, avatars, unread badges, and an animated typing indicator, served straight from `public/`.
- **Robust validation** — empty and oversized messages are rejected, duplicate usernames within a room are prevented (case-insensitive), and errors surface as an `error:message` event rather than a crash.

---

## Tech Stack

| Purpose               | Package                    | Notes                                             |
| --------------------- | -------------------------- | ------------------------------------------------- |
| HTTP server           | `express` ^4.19            | Serves the static client and a health endpoint    |
| Real-time transport   | `socket.io` ^4.7           | WebSocket-based event engine for all messaging    |
| Cross-origin support  | `cors` ^2.8                | Configurable CORS origin allow-list               |
| Configuration         | `dotenv` ^16.4             | Loads `PORT` / `CORS_ORIGIN` from `.env`          |
| Dev auto-reload       | `nodemon` ^3.1 (dev)       | Restarts the server on file changes               |
| Integration testing   | `socket.io-client` ^4.7 (dev) | Drives real socket clients against the server |
| Client runtime        | Vanilla HTML / CSS / JS    | No front-end build step or framework              |

---

## Project Structure

```
assignment-13-realtime-chat/
├── server.js                       # Express + Socket.io bootstrap (default port 5000)
├── package.json                    # Scripts, dependencies, metadata
├── .env.example                    # Sample environment configuration
├── .gitignore
├── README.md
├── public/                         # Static client (served by Express)
│   ├── index.html                  # Multi-room chat markup + login overlay
│   ├── app.js                      # Socket.io client + DOM/state logic
│   └── style.css                   # Dark theme: bubbles, animations, responsive layout
├── sockets/
│   └── index.js                    # All Socket.io event handlers + typing timers
├── utils/
│   ├── constants.js                # MAX_HISTORY, default rooms, length limits, typing timeout
│   ├── messageStore.js             # roomHistories + append / get / trim (50-msg cap)
│   ├── userStore.js                # connectedUsers Map + presence helpers
│   └── validators.js               # username / room / message validation
└── test/
    ├── messageStore.test.js        # Unit tests (plain Node assert)
    └── socket.integration.test.js  # End-to-end socket.io-client tests
```

---

## Prerequisites

- **Node.js 18 or newer** (uses modern JavaScript and current `socket.io`).
- **npm** (bundled with Node.js).

---

## Installation & Setup

```bash
# 1. Clone the repository and check out the branch
git clone https://github.com/Adityac17/Assignment_13.git
cd Assignment_13
git checkout assignment-13

# 2. Move into the project directory
cd assignment-13-realtime-chat

# 3. Install dependencies
npm install

# 4. (Optional) create a local .env — the defaults work out of the box
cp .env.example .env
```

---

## Environment Variables

Configuration is read from a `.env` file via `dotenv` (see `.env.example`). Both variables are optional.

| Variable      | Default | Description                                                            |
| ------------- | ------- | ---------------------------------------------------------------------- |
| `PORT`        | `5000`  | Port the HTTP + Socket.io server listens on.                          |
| `CORS_ORIGIN` | `*`     | Allowed CORS origin(s). Use `*` to allow all, or a specific origin.   |

> **macOS note:** On recent macOS the AirPlay Receiver often occupies port **5000**. If you see `EADDRINUSE :::5000`, either disable *System Settings → General → AirDrop & Handoff → AirPlay Receiver*, or start the server on a different port by overriding `PORT`, e.g. `PORT=5055 npm start`.

---

## Running the App

```bash
# Development — auto-reloads on file changes (nodemon)
npm run dev

# Production — plain Node
npm start
```

On startup the console prints the listening URL and the default rooms:

```
Chat server listening on http://localhost:5000
Default rooms: #general, #developers, #random
```

### Opening the client

Open a browser at **http://localhost:5000**. The Express static middleware serves `public/index.html` (the Socket.io client is loaded automatically from `/socket.io/socket.io.js`). Enter a username, optionally an emoji avatar, and start chatting.

A health/info endpoint is also available for smoke tests:

```bash
curl http://localhost:5000/api/health
# { "status": "ok", "rooms": [...], "defaultRooms": ["#general","#developers","#random"] }
```

---

## Socket Event Reference

All real-time communication flows through Socket.io events. Many client→server events accept an optional acknowledgement callback that receives `{ ok, ... }` or `{ ok: false, error }`.

### Client → Server

| Event          | Payload                                | Description |
| -------------- | -------------------------------------- | ----------- |
| `user:login`   | `{ username, avatar? }`                | Registers the user in `connectedUsers`. Falls back to a deterministic emoji avatar if none is given. Ack: `{ ok, socketId, username, avatar }`. Also triggers `user:login:ok`. |
| `room:join`    | `{ room }`                             | Joins a room (auto-leaving the previous one). Server replies with `room:history` to the joiner, a `room:system` join notice, and `room:userlist` to the room. Ack: `{ ok, room }`. |
| `room:leave`   | `{ room }`                             | Leaves a room; server emits a `room:system` leave notice and re-broadcasts `room:userlist`. Ack: `{ ok, room }`. |
| `chat:send`    | `{ room, message, timestamp }`         | Sends a group message. Validated, stored (max 50/room), then broadcast to the room as `chat:receive`. Requires membership in `room`. Ack: `{ ok, id }`. |
| `direct:send`  | `{ recipientId, message }`             | Sends a private message to a single online socket. Delivered only to the recipient as `direct:receive`. Ack: `{ ok, id }`. |
| `typing:start` | `{ room }`                             | Announces typing to the room (excluding the sender) via `typing:update`; arms a 3-second server-side auto-stop timer. |
| `typing:stop`  | `{ room }`                             | Clears the typing state and timer, broadcasting `typing:update` with `isTyping: false`. |

### Server → Client

| Event            | Payload                                                        | Description |
| ---------------- | ------------------------------------------------------------- | ----------- |
| `user:login:ok`  | `{ socketId, username, avatar }`                              | Login confirmation for the connecting client. |
| `room:history`   | `{ room, messages: Message[] }`                              | The last 50 messages of the room, sent **only to the joiner**. |
| `room:userlist`  | `{ room, users: [{ socketId, username, avatar }] }`         | The current online roster for a room, sent to everyone in it. |
| `room:system`    | `{ room, text, timestamp }`                                  | System notice such as `"alice joined #general"` / `"alice left #general"`. Not persisted to history. |
| `chat:receive`   | `{ id, room, username, avatar, senderId, message, timestamp }` | A broadcast group message delivered to every room member. |
| `direct:receive` | `{ id, from: { socketId, username, avatar }, to: { socketId, username }, message, timestamp }` | A private message delivered only to the recipient socket. |
| `typing:update`  | `{ room, username, isTyping }`                              | Typing state change, broadcast to room members **except** the sender. |
| `error:message`  | `{ scope, error }`                                          | A validation or operational error for the client to surface (e.g. failed login, empty message, DM to an offline user). |

*`Message` shape (as stored and replayed):* `{ id, room, username, avatar, senderId, message, timestamp }`.

---

## In-Memory Data Model

All runtime state lives in two structures — nothing touches disk.

- **`connectedUsers`** — a `Map<socketId, { username, avatar, currentRoom }>` in `utils/userStore.js`. It is the presence registry: helpers add/remove users, move them between rooms, list a room's occupants, and enforce per-room username uniqueness.
- **`roomHistories`** — an `Object<roomName, Message[]>` in `utils/messageStore.js`. Each room keeps at most **`MAX_HISTORY = 50`** messages; when a 51st message arrives, the oldest is trimmed from the front so only the newest 50 remain.

The store is **seeded with three default rooms** — `#general`, `#developers`, and `#random` — so `getHistory()` on a known room always returns an array. Direct messages are intentionally **not** stored in `roomHistories`; they are routed straight to the recipient and are ephemeral.

Key `messageStore` functions:

| Function                          | Behaviour                                          |
| --------------------------------- | -------------------------------------------------- |
| `addMessageToHistory(room, msg)`  | Appends a message, trims to the newest 50, returns it. |
| `getHistory(room)`                | Returns a copy of the last 50 messages for a room. |
| `listRooms()`                     | Lists all rooms that currently have a history bucket. |
| `clearHistory(room?)`             | Clears one room (or all rooms) — used mainly in tests. |

---

## Manual Testing (3+ browser tabs)

The multi-user behaviour is easiest to verify with several tabs open side by side.

1. Run `npm run dev` and open **http://localhost:5000** in **three** browser tabs.
2. In each tab, log in with a **different username** (an emoji avatar is optional).
3. **Group chat & room isolation** — In tabs A and B, select `#general`; in tab C, select `#developers`. Send a message from A: it appears in A and B but **not** in C.
4. **Presence roster** — Watch the right-hand "Online in #general" panel update as tabs join, switch rooms, or close.
5. **Typing indicator** — Start typing in tab A without sending. Tab B shows "alice is typing…". Stop for ~3 seconds and it clears automatically.
6. **Direct message** — In the online-users panel, click another user and enter a private message. It appears only for the sender and recipient, tagged **DM**; nobody else in the room sees it.
7. **History replay** — Send a few messages in `#general`, then open a fresh tab, log in, and join `#general`: the recent messages load immediately.

---

## Testing

```bash
npm test        # runs the unit suite, then the integration suite
```

The test script runs two suites in sequence (no external test framework — plain Node `assert`):

- **Unit — `test/messageStore.test.js`** verifies append behaviour, the 50-message cap and front-trimming, that `getHistory` returns the newest 50 in insertion order, that an unknown room returns `[]`, and that the default rooms are pre-seeded.
- **Integration — `test/socket.integration.test.js`** spins up a real Socket.io server on an ephemeral port, connects multiple `socket.io-client` instances, and asserts five end-to-end scenarios:
  1. **Room isolation** — `chat:send` in `#general` reaches other `#general` members but not a client in `#developers`.
  2. **Typing** — `typing:update` reaches room members except the sender.
  3. **Direct message** — `direct:send` reaches only the target socket, never the room.
  4. **Presence** — `room:userlist` updates correctly on join, leave, and disconnect.
  5. **History** — a late joiner receives the room's `room:history`.

---

## Design Choices & Limitations

- **Single active room per socket.** Each user has exactly one `currentRoom`; joining a new room automatically leaves the previous one. This keeps the presence roster and typing indicators unambiguous and matches the tab-style UI. (Socket.io supports multi-room membership natively — the single-room model is a deliberate product choice enforced in the handlers.)
- **Per-room, case-insensitive duplicate-username prevention.** The same display name can exist in different rooms but cannot collide within one room.
- **Server-enforced typing auto-stop.** A per-user, per-room timer (`TYPING_TIMEOUT_MS = 3000`) clears the indicator server-side, so a client that disconnects mid-type never leaves a stuck "typing…" state. The client additionally debounces (1.5 s idle) to avoid event spam.
- **Ephemeral direct messages.** DMs are routed directly to the recipient socket via `io.to(recipientId)` and are never persisted in history.
- **Input validation limits.** Message ≤ 2000 chars, username ≤ 32 chars, room name ≤ 40 chars; empty values are rejected. Invalid input produces an `error:message` event rather than throwing.
- **Deterministic default avatars.** When no avatar is supplied, one is derived from the username so users stay visually distinct without extra input.

**Limitations (by design):**

- **No persistence.** All history and presence live in process memory and are **lost on restart** — there is no database.
- **Single-node only.** State is per-process; running multiple instances behind a load balancer would require a shared adapter such as `@socket.io/redis-adapter`.
- **Bounded history.** Only the newest 50 messages per room are retained and replayed.

---

## Author

**Aditya S Chouksey**

## License

Released under the **MIT** License.
