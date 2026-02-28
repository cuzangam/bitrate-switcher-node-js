# BSNJS

Bitrate-based OBS scene switcher with Twitch chat control, automatic Twitch token refresh, and optional raid auto-stop.

## Features

- Connects to OBS via WebSocket v5
- Auto-switches scenes based on bitrate
- Supports stream stats from:
  - NGINX RTMP stats XML
  - IRLHosting stats JSON
  - Node-Media-Server API
  - Nimble Streamer API (bitrate + RTT trigger)
  - SRT Live Server stats JSON
- Twitch IRC chat commands with configurable prefix
- Moderator-only control commands (`fix`, `ss`, `start`, `stop`, `status`)
- Twitch OAuth access token validation + refresh
- Optional auto-stop stream on raid

---

## Requirements

- Node.js 18+
- OBS Studio with WebSocket enabled (default port `4455`)

Install dependencies:

```bash
npm install
```

Run:

```bash
node index.js
```

---

## Configuration

Edit `config.json`.

### OBS

```json
"OBS": {
  "addr": "localhost",
  "password": "YOUR_OBS_WEBSOCKET_PASSWORD",
  "port": 4455,
  "record-when-streaming": true
}
```

### Switcher

```json
"control": {
  "ingest": {
    "switcher": {
      "enabled": true,
      "only-switch-when-streaming": false,
      "auto-stop-on-raid": true,
      "scenes": {
        "normal": "LIVE",
        "low": "LOW",
        "brb": "BRB"
      },
      "trigger": {
        "msTriggerCheck": 1000,
        "low": 1500,
        "offline": 400,
        "highRtt": 250
      },
      "server": {
        "type": "irlhosting"
      }
    }
  }
}
```

Notes:
- Use only `server` for stream stats configuration.
- Switching behavior:
  - `kbps <= offline` -> `brb`
  - `kbps <= low` OR Nimble RTT >= `highRtt` -> `low`
  - otherwise -> `normal`

### Twitch Chat + OAuth

```json
"chat": {
  "access-token": "YOUR_ACCESS_TOKEN",
  "refresh-token": "YOUR_REFRESH_TOKEN",
  "client-id": "YOUR_CLIENT_ID",
  "client-secret": "YOUR_CLIENT_SECRET",
  "bot-username": "YOUR_BOT_USERNAME",
  "channel": "YOUR_CHANNEL",
  "prefix": "!"
}
```

Token behavior:
- Access token is validated on startup.
- If expired/invalid, token is refreshed automatically.
- Updated tokens are saved back to `config.json`.

---

## Stream Server Config Examples

### 1) NGINX RTMP

```json
"server": {
  "name": "NGINX LIVE",
  "type": "nginx",
  "log-bitrate-in-console": true,
  "statsUrl": "http://localhost:9999/stat",
  "application": "live",
  "key": "live"
}
```

### 2) IRLHosting

```json
"server": {
  "name": "IRLHosting",
  "type": "irlhosting",
  "log-bitrate-in-console": true,
  "statsUrl": "http://host:8181/stats",
  "publisher": "publish/live/feed1"
}
```

### 3) Node-Media-Server

```json
"server": {
  "type": "NodeMediaServer",
  "statsUrl": "http://localhost:8000/api/streams",
  "application": "publish",
  "key": "live",
  "auth": {
    "username": "admin",
    "password": "admin"
  }
}
```

`auth` is optional.

### 4) Nimble Streamer (SRT)

```json
"server": {
  "type": "Nimble",
  "statsUrl": "http://nimble:8082",
  "id": "0.0.0.0:1234",
  "application": "live",
  "key": "srt"
}
```

Nimble can trigger low scene by bitrate or RTT (`trigger.highRtt`).

### 5) SRT Live Server

```json
"server": {
  "type": "SrtLiveServer",
  "statsUrl": "http://localhost:8181/stats",
  "publisher": "publish/live/feed1"
}
```

---

## Chat Commands

All commands use the configured `chat.prefix`.

### Public

- `!b`
- `!bitrate`

Returns current bitrate value.

### Moderator+ only

- `!f`
- `!fix`
  - Restarts media inputs (`ffmpeg_source`, `vlc_source`)
- `!ss <scene name>`
  - Switches current program scene
- `!start`
  - Starts OBS stream
- `!stop`
  - Stops OBS stream
- `!status`
  - Reports OBS connection, stream state, current scene, switcher state, and bitrate

Moderator+ means Twitch moderator or broadcaster.

---

## Raid Auto-Stop

If enabled:

```json
"auto-stop-on-raid": true
```

The app attempts to stop stream when a raid event is detected (IRC/PubSub handling).

---

## Troubleshooting

- **OBS not connected**
  - Verify OBS WebSocket password, address, and port.
- **Commands not responding**
  - Check `chat.prefix`, `channel`, and `bot-username`.
- **Auth/token errors**
  - Confirm `client-id`, `client-secret`, and `refresh-token` are valid.
- **No bitrate switching**
  - Verify selected `server` type and stats endpoint data.

---

## Security Notes

- Keep `config.json` secret (contains OAuth credentials).
- Do not commit real tokens/client secret to public repositories.
