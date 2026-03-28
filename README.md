# MineWeb Multiplayer

> Real-time multiplayer for MineWeb, powered by Node.js and WebSockets.

---

## Table of Contents

1. [Overview](#1-overview)
2. [How It Works](#2-how-it-works)
3. [Running the Server Locally](#3-running-the-server-locally)
4. [Hosting on Render](#4-hosting-on-render)
5. [Connecting from the Client](#5-connecting-from-the-client)
6. [Chat](#6-chat)
7. [Server Status Page](#7-server-status-page)
8. [Configuration](#8-configuration)
9. [Protocol Reference](#9-protocol-reference)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Overview

MineWeb multiplayer works by running a lightweight Node.js WebSocket server (`index.js`) that all players connect to. The server:

- Picks a random **seed** and sends it to every client so everyone generates the identical world
- Relays **block changes** (break/place) to all connected players in real time
- Broadcasts **player positions** 20 times per second so you can see others move
- Keeps a full **block change log** so players who join late see the same modified world as everyone else
- Handles **chat** between all connected players

---

## 2. How It Works

```
Player A ──┐
Player B ──┼──► index.js (Node.js WebSocket server) ◄──► All players
Player C ──┘
```

1. Server starts and picks a random seed.
2. A player opens MineWeb in their browser and enters the server URL.
3. The server sends an `init` packet containing the seed, all existing block changes, and a list of currently connected players.
4. The client generates the world from the seed, applies the block changes, and spawns the other players.
5. From that point, all block breaks/places and chat messages are sent to the server and relayed to everyone else.
6. Player positions are broadcast by the server every 50ms.

---

## 3. Running the Server Locally

### Requirements

- Node.js 16 or higher
- npm

### Setup

```bash
# 1. Put index.js and package.json in the same folder
# 2. Install dependencies
npm install

# 3. Start the server (default port 8080)
node index.js

# 4. Or specify a port
node index.js 3000
```

You should see:

```
╔══════════════════════════════════════╗
║     MineWeb Multiplayer Server       ║
╠══════════════════════════════════════╣
║  Port    : 8080                     ║
║  Seed    : 7431822                  ║
║  WRAD    : 38                       ║
╚══════════════════════════════════════╝

WebSocket : ws://localhost:8080
Status    : http://localhost:8080
```

### Connecting locally

In MineWeb, enter `ws://localhost:8080` in the server URL field (or just `localhost:8080` — the client fills in `ws://` automatically).

---

## 4. Hosting on Render

Render is the recommended free hosting option. The server runs as a **Web Service**.

### Files needed

```
your-repo/
├── index.js
└── package.json
```

**package.json:**

```json
{
  "name": "mineweb-server",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "ws": "^8.18.0"
  }
}
```

### Deploying

1. Push both files to a **GitHub repository**.
2. Go to [render.com](https://render.com) and create a new **Web Service**.
3. Connect your GitHub repository.
4. Fill in the settings:

| Setting | Value |
|---|---|
| **Runtime** | Node |
| **Build Command** | `npm install` |
| **Start Command** | `npm start` |
| **Environment Variables** | *(none — Render sets `PORT` automatically)* |

5. Click **Deploy**. Wait for the build to finish.
6. Your server URL will be something like `https://mineweb-server-xxxx.onrender.com`.

### Connecting to a Render server

Paste your Render URL directly into MineWeb — any of these formats work:

```
https://mineweb-server-xxxx.onrender.com
mineweb-server-xxxx.onrender.com
wss://mineweb-server-xxxx.onrender.com
```

The client automatically converts `https://` to `wss://` (secure WebSocket), which is required for Render.

> ⚠️ **Render free tier sleeps after 15 minutes of inactivity.** The first connection after a sleep will take ~30 seconds while the server wakes up. This is normal.

---

## 5. Connecting from the Client

Open MineWeb and find the **Multiplayer** card on the main menu.

| Field | Description |
|---|---|
| **Your name** | Display name shown above your character to other players. Letters, numbers, spaces and underscores only. Max 20 characters. |
| **Server URL** | The WebSocket URL of the server. Any format is accepted (see below). |

### Accepted URL formats

All of the following are valid — the client normalises them automatically:

```
mineweb-server-xxxx.onrender.com          → wss://mineweb-server-xxxx.onrender.com
https://mineweb-server-xxxx.onrender.com  → wss://mineweb-server-xxxx.onrender.com
wss://mineweb-server-xxxx.onrender.com    → wss://mineweb-server-xxxx.onrender.com
localhost:8080                            → ws://localhost:8080
ws://localhost:8080                       → ws://localhost:8080
```

### Connection states

The status line under the Join button shows what's happening:

| Status | Meaning |
|---|---|
| `Connecting to wss://…` | WebSocket handshake in progress |
| `Authenticating…` | Connected, waiting for server init |
| `Loading world…` | Generating world from server seed |
| *(empty)* | Connected and in-game |
| `❌ Could not connect (1006)` | Server unreachable or URL wrong |
| `Disconnected` | Lost connection after being in-game |

---

## 6. Chat

Press **T** to open the chat input while in-game.

| Key | Action |
|---|---|
| `T` | Open chat input |
| `Enter` | Send message and close input |
| `Esc` | Close input without sending |

- Messages are limited to 200 characters.
- Chat messages appear in the bottom-left of the screen and fade after 5 seconds.
- All players on the server see all messages.
- Your own messages are echoed back so you see them in the log too.

---

## 7. Server Status Page

The server exposes a JSON status page at its HTTP URL. Open it in a browser to check the server is alive:

```
http://localhost:8080
https://mineweb-server-xxxx.onrender.com
```

Example response:

```json
{
  "name": "MineWeb Multiplayer Server",
  "version": "1.0.0",
  "seed": 7431822,
  "wrad": 38,
  "players": 3,
  "changes": 142,
  "uptime": 3600
}
```

| Field | Description |
|---|---|
| `seed` | The world seed used for this session. Resets on server restart. |
| `wrad` | World radius. Must match the client's `WRAD` constant. |
| `players` | Number of currently connected players. |
| `changes` | Number of blocks modified since server start. |
| `uptime` | Server uptime in seconds. |

---

## 8. Configuration

All configuration is at the top of `index.js`:

```js
const PORT = parseInt(process.env.PORT || process.argv[2]) || 8080;
const WRAD = 38;   // world radius — must match client WRAD
const SEED = Math.floor(Math.random() * 0xffffff); // random each restart
const TICK = 50;   // position broadcast interval in ms (50ms = 20Hz)
```

### Fixing the world seed

By default the seed is random on every server restart, which means the world resets when Render's free tier restarts your server overnight. To persist the same world, set a fixed seed:

```js
const SEED = 1234567; // fixed seed — world stays the same across restarts
```

> Note: block changes are currently stored in memory and lost on restart. For persistent worlds, you would need to write `blockChanges` to a file or database.

---

## 9. Protocol Reference

All messages are JSON sent over WebSocket.

### Server → Client

#### `init`
Sent once immediately after connection. Contains everything the client needs to join.

```json
{
  "type": "init",
  "id": 1,
  "seed": 7431822,
  "wrad": 38,
  "color": "#3498db",
  "name": "Player1",
  "changes": { "10,48,5": 3, "10,49,5": 1 },
  "players": [
    { "id": 2, "name": "Alice", "color": "#e74c3c", "x": 12.3, "y": 50.0, "z": 8.1, "rx": 0, "ry": 1.2, "slot": 0 }
  ]
}
```

#### `playerJoin`
Another player connected.

```json
{ "type": "playerJoin", "id": 3, "name": "Bob", "color": "#2ecc71", "x": 0.5, "y": 50, "z": 0.5 }
```

#### `playerLeave`
A player disconnected.

```json
{ "type": "playerLeave", "id": 3 }
```

#### `playerName`
A player changed their display name.

```json
{ "type": "playerName", "id": 3, "name": "Bobby" }
```

#### `positions`
Broadcast every 50ms with all player positions.

```json
{
  "type": "positions",
  "list": [
    { "id": 2, "x": 12.3, "y": 50.0, "z": 8.1, "rx": -0.2, "ry": 1.2, "slot": 0 }
  ]
}
```

#### `setBlock`
A player broke or placed a block.

```json
{ "type": "setBlock", "x": 10, "y": 48, "z": 5, "id": 3, "by": 2 }
```

`id: 0` means the block was broken (set to AIR). `by` is the player ID who made the change.

#### `chat`
A chat message from any player.

```json
{ "type": "chat", "from": 2, "name": "Alice", "color": "#e74c3c", "text": "Hello!" }
```

---

### Client → Server

#### `move`
Sent every 3 frames (~20Hz) with the local player's current position and rotation.

```json
{ "type": "move", "x": 12.3, "y": 50.0, "z": 8.1, "rx": -0.2, "ry": 1.2, "slot": 0 }
```

#### `setBlock`
Break or place a block.

```json
{ "type": "setBlock", "x": 10, "y": 48, "z": 5, "id": 3 }
```

`id: 0` = break. Any other value = place that block ID.

#### `chat`
Send a chat message.

```json
{ "type": "chat", "text": "Hello everyone!" }
```

#### `setName`
Set the player's display name. Sent once after `init` is received.

```json
{ "type": "setName", "name": "Alice" }
```

---

## 10. Troubleshooting

**"Could not connect" immediately**
- Check the server URL — make sure it's correct and the server is running.
- If hosting on Render's free tier, the server may be sleeping. Wait 30 seconds and try again.
- Open the status page (`https://your-server.onrender.com`) in a browser to check if the server is awake.

**Players can see each other but blocks don't sync**
- Make sure all clients are on the same version of MineWeb.
- Check the browser console for errors.

**World looks different between players**
- The `WRAD` constant in `index.js` must match the `WRAD` constant in the client HTML. Both default to `38`.

**Server restarts and the world resets**
- This is expected on Render's free tier. Set a fixed `SEED` in `index.js` so at least the terrain is consistent. Block changes are lost on restart — this is a known limitation.

**Chat input stops responding**
- Press `Esc` then `T` to reopen the chat input. If pointer lock is interfering, press `Esc` to pause first.

**"Port already in use" locally**
- Another process is using port 8080. Run on a different port: `node index.js 3001`

---

*MineWeb Multiplayer · Made by IdeiGeniale · Powered by Node.js + ws*
