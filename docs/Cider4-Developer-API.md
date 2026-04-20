# Cider v2 API Reference

The v2 API provides a modern, RESTful interface for controlling Cider 4.x — playback, queue, audio features, library, and scoped authorization. All endpoints are prefixed with `/api/v2/`.

> **v1 Compatibility:** The original `/api/v1/playback/*` endpoints remain fully functional and are unscoped. v2 is a parallel API with better structure, scope-aware auth, and consistent response formatting. See the [migration guide](#v1-to-v2-migration-guide) at the end.

---

## Table of contents

1. [Quick start](#quick-start)
2. [Authentication & scopes](#authentication--scopes)
3. [Obtaining a token](#obtaining-a-token)
4. [Response format](#response-format)
5. [Error codes](#error-codes)
6. [Rate limiting](#rate-limiting)
7. [Endpoint reference](#endpoint-reference)
   - [Auth](#auth)
   - [Client](#client)
   - [Playback](#playback)
   - [Queue](#queue)
   - [Audio](#audio)
   - [Library](#library)
   - [Plugins](#plugins)
   - [Events catalog](#events-catalog)
8. [WebSocket channel](#websocket-channel)
9. [v1 to v2 migration guide](#v1-to-v2-migration-guide)

---

## Quick start

Every request carries an `apptoken` header:

```bash
curl -H "apptoken: YOUR_TOKEN" http://127.0.0.1:10767/api/v2/playback/state
# → { "data": { "state": "playing" } }
```

A token can be obtained three ways — see [Obtaining a token](#obtaining-a-token).

---

## Authentication & scopes

Every v2 endpoint except [Scope-less endpoints](#scope-less-endpoints) and [`POST /api/v2/auth/request`](#post-apiv2authrequest) requires a valid `apptoken` header. Tokens carry a list of **scopes** that determine which endpoints the token can reach.

### The 6 scopes

| Scope | Covers |
|---|---|
| `playback` | All `/api/v2/playback/*` — play, pause, seek, skip, rate, repeat, shuffle, autoplay, play-item/play-id |
| `queue` | All `/api/v2/queue/*` — list, position, jump, add-next, add-later, move, delete, shuffle, smart-optimizer |
| `library` | All `/api/v2/library/*` — playlists, albums, artists, songs; love/dislike; add/remove |
| `audio` | All `/api/v2/audio/*` — volume, crossfade, automix, atmos |
| `account` | `/api/v2/client/tokens` — reads your Apple Music developer + user tokens + storefront |
| `plugins` | `/api/v2/plugins/install` — request a plugin install. **Each install still requires explicit per-plugin user consent via a dialog**, so the scope is necessary but not sufficient. |

A token with only `playback` cannot read your library. A token with only `library` cannot set the volume. Users pick which scopes to grant at creation time.

### Scope-less endpoints

These three endpoints accept any valid token (or no token at all for `/auth/request`) regardless of its scope set:

- `GET /api/v2/client/info` — harmless telemetry; useful for preflight / version checks
- `GET /api/v2/events/catalog` — public documentation
- `POST /api/v2/auth/request` — **unauthenticated**; used to bootstrap a token

### Legacy tokens

Tokens created before scopes shipped do not have a `scopes` field. They are **grandfathered** to all scopes and continue working unchanged. Settings → Connectivity flags them with a "Legacy (all access)" badge; users can delete and re-issue with a specific scope set at their discretion.

### Scope errors

Calling an endpoint your token doesn't have the scope for returns `403 INSUFFICIENT_SCOPE`:

```json
{ "error": { "code": "INSUFFICIENT_SCOPE", "message": "Required scope: audio" } }
```

---

## Obtaining a token

Three ways, in order from manual to fully remote:

### Option A — Settings UI (manual)

User opens **Settings → Connectivity → API Tokens → Create New**, picks a name + scope checkboxes, and copies the generated token. Works entirely in-app; no external URLs involved.

### Option B — `cider://request-auth?…` deep link

External apps on the **same device** as Cider can request consent via the OS protocol handler:

```
cider://request-auth?app-name=<name>&scopes=<csv>&app-image=<https-or-data-url>
```

Query parameters:

| Param | Required | Notes |
|---|---|---|
| `app-name` | Yes | Display name shown in the dialog (URL-encoded, 1–64 chars) |
| `scopes` | Yes | Comma-separated subset of the 5 scopes |
| `app-image` | No | Icon URL. Must be `https://` or `data:image/{png,jpeg,gif,webp,svg+xml}` ≤ 128 KB. A bad/unreachable URL falls back to a generic lock icon. |

Opening the URL focuses Cider and shows the consent dialog. On Approve, the token is shown once on-screen for the user to copy into the requesting app. No programmatic return path — the user transfers the token manually.

```bash
# Linux
xdg-open 'cider://request-auth?app-name=My%20Widget&scopes=playback,library'
# macOS
open 'cider://request-auth?app-name=My%20Widget&scopes=playback,library'
# Windows
start "" "cider://request-auth?app-name=My%20Widget&scopes=playback,library"
```

### Option C — `POST /api/v2/auth/request` (remote clients)

For apps on a **different device** from Cider, there's an unauthenticated HTTP endpoint that pushes a dialog to the Cider device and blocks until the user decides. The token is returned in the HTTP response, so no manual copy-paste is needed.

See the full spec under [Auth → `POST /api/v2/auth/request`](#post-apiv2authrequest).

---

## Response format

### Success

```json
{
  "data": { ... },
  "meta": { "offset": 0, "limit": 50, "total": 120 }
}
```

`meta` is only present on paginated endpoints.

### Error

```json
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "Human-readable description"
  }
}
```

> Legacy auth failures from the pre-route hook (`UNAUTHORIZED_APP_TOKEN`) use a flat `{ "error": "UNAUTHORIZED_APP_TOKEN" }` shape for backward compatibility. All other v2 errors use the structured form above.

---

## Error codes

| Code | HTTP | Meaning |
|---|---|---|
| `INVALID_REQUEST` | 400 | Malformed body / query / path params |
| `UNAUTHORIZED_APP_TOKEN` | 403 | Missing or invalid `apptoken` header |
| `INSUFFICIENT_SCOPE` | 403 | Token does not carry the required scope |
| `AUTH_REQUEST_DENIED` | 403 | User denied the consent dialog |
| `AUTH_REQUEST_TIMEOUT` | 408 | User did not respond within 2 min |
| `AUTH_REQUEST_BUSY` | 409 | Another auth dialog is already pending |
| `AUTH_REQUEST_COOLDOWN` | 429 | Same IP fired another request within 5 s |
| `AUTH_REQUEST_BANNED` | 429 | Same IP hit 3 denials in 5 min → 15 min ban |
| `AUTH_RESPONSE_MALFORMED` | 500 | Dialog returned unexpected shape (bug; file an issue) |
| `PLUGIN_INSTALL_DENIED` | 403 | User denied the install dialog |
| `PLUGIN_INSTALL_TIMEOUT` | 408 | User did not respond within 3 min |
| `PLUGIN_INSTALL_BUSY` | 409 | Another install dialog is already pending |
| `PLUGIN_INSTALL_COOLDOWN` | 429 | Same IP fired another install request within 2 s |
| `PLUGIN_BAD_ZIP` | 422 | Downloaded file isn't a valid zip |
| `PLUGIN_BAD_MANIFEST` | 422 | `plugin.yml` missing / malformed / missing `identifier` |
| `PLUGIN_DOWNLOAD_FAILED` | 502 | URL unreachable or returned non-200 |
| `PLUGIN_INSTALL_FAILED` | 500 | Extraction / filesystem error after consent |
| `RPC_TIMEOUT` | 504 | Frontend did not respond within 30 s |

`429` responses include a `Retry-After: <seconds>` header.

---

## Rate limiting

Two endpoints are rate-limited. All other endpoints rely on token-based access control alone.

### `POST /api/v2/auth/request` (unauthenticated)

| Guard | Default | Error |
|---|---|---|
| Cooldown between requests (per IP) | 5 s | `AUTH_REQUEST_COOLDOWN` (429) |
| Denials in rolling window (per IP) | 3 in 5 min → 15 min ban | `AUTH_REQUEST_BANNED` (429) |
| Concurrent pending dialogs (global) | 1 | `AUTH_REQUEST_BUSY` (409) |
| User-response timeout | 2 min | `AUTH_REQUEST_TIMEOUT` (408) |

### `POST /api/v2/plugins/install` (scoped)

Lighter limits since callers already hold a token:

| Guard | Default | Error |
|---|---|---|
| Cooldown between requests (per IP) | 2 s | `PLUGIN_INSTALL_COOLDOWN` (429) |
| Concurrent pending dialogs (global) | 1 | `PLUGIN_INSTALL_BUSY` (409) |
| User-response timeout | 3 min | `PLUGIN_INSTALL_TIMEOUT` (408) |

State is in-memory and clears when Cider restarts.

---

## Endpoint reference

### Auth

#### `POST /api/v2/auth/request`

**Unauthenticated bootstrap endpoint.** Does not require an `apptoken` header — it's the one endpoint designed to hand one out. On call, Cider shows a consent dialog on the user's device; this HTTP request blocks until the user approves, denies, or the 2-minute timeout elapses.

**Scope:** none (open to unauthenticated callers, but rate-limited)

**Request body:**

```json
{
  "app_name": "Remote Dashboard",
  "app_image": "https://example.com/logo.png",
  "scopes": ["playback", "library"]
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `app_name` | string | yes | 1–64 chars; shown in the consent dialog |
| `app_image` | string | no | `https://` URL or `data:image/…` (≤ 128 KB); falls back to generic icon on error |
| `scopes` | string[] | yes | Non-empty subset of the 5 known scopes |

**Responses:**

| Status | Body | Meaning |
|---|---|---|
| 200 | `{ "data": { "token": "<new>", "scopes": ["playback","library"] } }` | User approved. The returned `scopes` may be a subset of the requested ones if the user unchecked some in the dialog. |
| 400 | `{ "error": { "code": "INVALID_REQUEST", … } }` | Body shape bad, unknown scope, or empty scopes array |
| 403 | `{ "error": { "code": "AUTH_REQUEST_DENIED", … } }` | User clicked Deny |
| 408 | `{ "error": { "code": "AUTH_REQUEST_TIMEOUT", … } }` | 2 min elapsed without user action |
| 409 | `{ "error": { "code": "AUTH_REQUEST_BUSY", … } }` | Another auth dialog is already pending |
| 429 | `{ "error": { "code": "AUTH_REQUEST_COOLDOWN"\|"AUTH_REQUEST_BANNED", … } }` | Rate limit (see `Retry-After` header) |

**Example:**

```bash
curl -X POST http://<cider-host>:10767/api/v2/auth/request \
  -H 'Content-Type: application/json' \
  -d '{
        "app_name": "Remote Dashboard",
        "scopes":   ["playback", "library"]
      }'
```

On the Cider device, a consent dialog appears. Once the user clicks Approve:

```json
{
  "data": {
    "token": "nhtvzkuhkxulg1w11zfd4ixq",
    "scopes": ["playback", "library"]
  }
}
```

Store this token and use it on subsequent calls via the `apptoken` header.

---

### Client

#### `GET /api/v2/client/info`

**Scope:** none (scope-less — any valid token works)

Client version and platform information. Safe to call on preflight to confirm reachability + auth without requiring any particular scope.

**Response:**

```json
{
  "data": {
    "framework": "dotnet",
    "platform": "win32",
    "osVersion": "10.0.22631",
    "port": 10767,
    "production": true,
    "version": "4.0.0"
  }
}
```

Framework values: `dotnet`, `genten`, `universal`. Platform values: `win32`, `darwin`, `linux`, `web`.

#### `GET /api/v2/client/tokens`

**Scope:** `account`

Apple Music authentication tokens (developer JWT, user token, storefront). Sensitive — only grant `account` to apps that genuinely need to call the Apple Music API on your behalf.

**Response:**

```json
{
  "data": {
    "developerToken": "eyJ…",
    "userToken":      "A…",
    "storefront":     "us"
  }
}
```

---

### Playback

All endpoints below require the `playback` scope.

#### `GET /api/v2/playback`

Full playback state snapshot.

```json
{
  "data": {
    "state": "playing",
    "nowPlaying": {
      "name": "Song Title", "artistName": "Artist", "albumName": "Album",
      "artwork": { "url": "https://…", "width": 600, "height": 600 },
      "durationInMillis": 240000,
      "inFavorites": true, "inLibrary": true, "flavor": "256"
    },
    "time":       { "currentTime": 45.2, "duration": 240, "remaining": 194.8 },
    "volume":     0.75,
    "repeatMode": "none",
    "shuffleMode": false,
    "autoplay":   true,
    "rate":       1.0
  }
}
```

#### `GET /api/v2/playback/now-playing`
Current track attributes with library/favorite status.

#### `GET /api/v2/playback/state`
`{ "data": { "state": "playing" | "paused" | "stopped" } }`

#### `GET /api/v2/playback/time`
`{ "data": { "currentTime": 45.2, "duration": 240, "remaining": 194.8 } }` (seconds)

#### `GET /api/v2/playback/audio-quality`

Audio quality + codec info. Includes `localPlayback` when playing local files.

```json
{
  "data": {
    "flavor": "256",
    "flavorLabel": "AAC 256kbps",
    "deviceAudioConfig": {
      "channelCount": 2, "sampleRate": 48000,
      "supportsAtmos": true,
      "supportedCodecs": ["AAC", "E-AC-3", "E-AC-3 JOC (Atmos)"]
    },
    "localPlayback": null
  }
}
```

Flavor values: `64`, `256`, `bin_64`, `bin_256`, `atmos`, `unk`, `local_lossless`, `local_<bitrate>`.

#### Transport controls

| Method | Path | Body | Effect |
|---|---|---|---|
| POST | `/playback/play` | — | Resume |
| POST | `/playback/pause` | — | Pause |
| POST | `/playback/stop` | — | Stop |
| POST | `/playback/toggle` | — | Toggle play/pause |
| POST | `/playback/next` | — | Skip forward |
| POST | `/playback/previous` | `{ "force"?: true }` | Previous track (or seek to 0 if > 3 s in; `force` skips the seek-first behavior) |
| POST | `/playback/seek` | `{ "position": 45.5 }` | Seek to position (seconds) |

#### Play specific content

| Method | Path | Body |
|---|---|---|
| POST | `/playback/play-item` | `{ "type": "songs", "id": "1234567890" }` |
| POST | `/playback/play-id` | Alias of `/playback/play-item` — same body, same semantics |
| POST | `/playback/play-url` | `{ "url": "https://music.apple.com/…" }` |
| POST | `/playback/play-href` | `{ "href": "/v1/catalog/us/songs/1234567890" }` |
| POST | `/playback/play-collection` | `{ "type": "albums", "id": "1234567890", "shuffle"?: false }` — async; takes a few seconds |

`play-collection` handles `albums`, `playlists`, and `stations`.

#### Playback rate / repeat / shuffle / autoplay

| Method | Path | Body / Response | Notes |
|---|---|---|---|
| GET | `/playback/rate` | `{ "data": { "rate": 1.0 } }` | |
| PATCH | `/playback/rate` | `{ "rate": 1.5 }` | Range: `0.25` – `4.0` |
| GET | `/playback/repeat` | `{ "data": { "mode": "none" } }` | Values: `none`, `one`, `all` |
| PUT | `/playback/repeat` | `{ "mode": "all" }` | |
| POST | `/playback/repeat/toggle` | — | Cycles none → one → all → none |
| GET | `/playback/shuffle` | `{ "data": { "enabled": false } }` | |
| PUT | `/playback/shuffle` | `{ "enabled": true }` | |
| POST | `/playback/shuffle/toggle` | — | |
| GET | `/playback/autoplay` | `{ "data": { "enabled": true } }` | |
| PUT | `/playback/autoplay` | `{ "enabled": false }` | |
| POST | `/playback/autoplay/toggle` | — | |

---

### Queue

All endpoints below require the `queue` scope.

#### `GET /api/v2/queue`

Paginated queue dump.

**Query:** `offset` (default 0), `limit` (default 50, max 200)

```json
{
  "data": {
    "items": [
      {
        "track": { "id": "123", "type": "songs", "attributes": { … } },
        "source": "user",
        "containerContext": {
          "type": "playlist", "id": "pl.abc", "name": "My Playlist"
        }
      }
    ],
    "position": 3
  },
  "meta": { "offset": 0, "limit": 50, "total": 120 }
}
```

#### Queue ops

| Method | Path | Body / Response |
|---|---|---|
| GET | `/queue/position` | `{ "data": { "position": 3, "total": 120 } }` |
| POST | `/queue/jump` | `{ "index": 5 }` — jump to a queue index |
| POST | `/queue/add-next` | `{ "type": "songs", "id": "…" }` — play next |
| POST | `/queue/add-later` | `{ "type": "songs", "id": "…" }` — add to end |
| DELETE | `/queue/items/:index` | Remove an item at index |
| POST | `/queue/move` | `{ "from": 5, "to": 2 }` — reorder |
| DELETE | `/queue` | Clear the queue |
| POST | `/queue/shuffle` | Toggle queue shuffle |

#### Smart Queue Optimizer

| Method | Path | Body / Response |
|---|---|---|
| GET | `/queue/smart-optimizer` | `{ "data": { "enabled": true, "busy": false, "pending": 0, "message": null } }` |
| POST | `/queue/smart-optimizer/toggle` | Toggle (requires Automix enabled) |

---

### Audio

All endpoints below require the `audio` scope.

| Method | Path | Notes |
|---|---|---|
| GET | `/audio/volume` | `{ "data": { "volume": 0.75 } }` |
| PATCH | `/audio/volume` | `{ "volume": 0.5 }` — range `0.0`–`1.0` |

#### Crossfade

```http
GET /api/v2/audio/crossfade
```

```json
{
  "data": {
    "enabled": true, "effectiveEnabled": true,
    "durationSec": 5, "skipEnabled": false,
    "tempoSensitivity": 0.6, "loudnessSensitivity": 0.75,
    "exclusions": {
      "albums": false, "playlists": false,
      "djMixes": true, "liveAlbums": true
    }
  }
}
```

```http
PATCH /api/v2/audio/crossfade
```

Partial update — include only fields to change:

```json
{ "enabled": true, "durationSec": 8, "exclusions": { "albums": true } }
```

#### Automix

```http
GET /api/v2/audio/automix
```

```json
{
  "data": {
    "enabled": true, "effectiveEnabled": true,
    "energyBoost": false, "normalizeDurationMs": 4000,
    "exclusions": { "albums": false, "playlists": false, "djMixes": true, "liveAlbums": true }
  }
}
```

```http
PATCH /api/v2/audio/automix
```

Partial update. Example: `{ "energyBoost": true, "normalizeDurationMs": 6000 }`

#### Atmos (Dolby)

```http
GET /api/v2/audio/atmos
```

```json
{
  "data": {
    "enabled": true, "binaural": true,
    "devicePreferences": { "default": "binaural" }
  }
}
```

```http
PATCH /api/v2/audio/atmos
```

Partial update. Example: `{ "enabled": true, "binaural": false }`

---

### Library

All endpoints below require the `library` scope.

#### Now-playing operations

| Method | Path | Notes |
|---|---|---|
| GET | `/library/now-playing/status` | `{ "data": { "inLibrary": true, "rating": 1 } }` — rating is `-1` / `0` / `1` |
| POST | `/library/now-playing/add` | Add current track to library |
| DELETE | `/library/now-playing` | Remove current track from library |
| POST | `/library/now-playing/love` | Rating = 1 |
| POST | `/library/now-playing/dislike` | Rating = -1 and skips to next |
| DELETE | `/library/now-playing/rating` | Clear rating |
| PUT | `/library/now-playing/rating` | `{ "rating": 1 }` — values: `-1`, `0`, `1` |

#### Library listing

All listing endpoints share the `{ data: { items }, meta: { offset, limit, total } }` envelope and accept `offset` / `limit` query params (same defaults as `/queue`: 0 / 50 / max 200). **Artwork URLs are pre-resolved** — no `{w}`/`{h}` placeholders remain, so clients can drop the URL straight into an `<img src>`.

##### `GET /api/v2/library/playlists`

User's library playlists.

```json
{
  "data": {
    "items": [
      {
        "id": "p.xxx", "name": "Coffeehouse",
        "description": "Slow mornings",
        "canEdit": true, "trackCount": 42,
        "artwork": { "url": "https://…", "width": 600, "height": 600 }
      }
    ]
  },
  "meta": { "offset": 0, "limit": 50, "total": 84 }
}
```

##### `GET /api/v2/library/playlists/:id/songs`

Songs in a specific library playlist. Each item has `id`, `name`, `artistName`, `albumName`, `durationMs`, `trackNumber`, `artwork`.

##### `GET /api/v2/library/albums`

User's library albums. Each item has `id`, `name`, `artistName`, `trackCount`, `releaseDate`, `artwork`.

##### `GET /api/v2/library/albums/:id/songs`

Songs on a specific library album. Same shape as playlist songs.

##### `GET /api/v2/library/artists`

User's library artists. Each item has `id`, `name`, `artwork`.

##### `GET /api/v2/library/artists/:id/albums`

Albums by a specific library artist. Same shape as `/library/albums`.

##### `GET /api/v2/library/songs`

All songs in the library (songs tab). Same shape as playlist songs.

---

### Plugins

All endpoints below require the `plugins` scope. **Each install also requires explicit per-plugin user consent via a dialog**, even with the scope — the scope is the gate for "this app can ask", the dialog is the gate for "this specific plugin gets installed".

#### `POST /api/v2/plugins/install`

**Scope:** `plugins`

Request a plugin install from a .zip URL. The server downloads the zip, validates `plugin.yml`, then pushes a consent dialog to the Cider device showing the real (not caller-claimed) manifest. The HTTP request blocks until the user approves, denies, or the 3-minute timeout elapses.

**Request body:**

```json
{ "url": "https://example.com/my-plugin.zip" }
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `url` | string | yes | http(s) URL to the .zip. ≤ 2048 chars. |

**Responses:**

| Status | Body | Meaning |
|---|---|---|
| 200 | `{ "data": { "identifier", "name", "version", "installPath" } }` | User approved; plugin installed. Cider usually needs a reload to activate new plugins. |
| 400 | `{ "error": { "code": "INVALID_REQUEST", … } }` | Body shape / URL bad |
| 403 | `{ "error": { "code": "PLUGIN_INSTALL_DENIED", … } }` | User clicked Cancel |
| 403 | `{ "error": { "code": "INSUFFICIENT_SCOPE", … } }` | Token lacks `plugins` scope |
| 408 | `{ "error": { "code": "PLUGIN_INSTALL_TIMEOUT", … } }` | 3 min elapsed without user action |
| 409 | `{ "error": { "code": "PLUGIN_INSTALL_BUSY", … } }` | Another install dialog is already pending |
| 422 | `{ "error": { "code": "PLUGIN_BAD_ZIP" \| "PLUGIN_BAD_MANIFEST", … } }` | Download succeeded but is not a valid plugin |
| 429 | `{ "error": { "code": "PLUGIN_INSTALL_COOLDOWN", … } }` | 2 s per-IP cooldown |
| 502 | `{ "error": { "code": "PLUGIN_DOWNLOAD_FAILED", … } }` | URL unreachable / 4xx / 5xx |

**Example:**

```bash
curl -X POST http://127.0.0.1:10767/api/v2/plugins/install \
  -H 'Content-Type: application/json' \
  -H 'apptoken: YOUR_TOKEN' \
  -d '{"url": "https://example.com/my-plugin.zip"}'
```

Dialog on the Cider device shows the plugin name, author, version, description, source host, and size. On approve:

```json
{
  "data": {
    "identifier": "com.example.my-plugin",
    "name": "My Plugin",
    "version": "1.2.3",
    "installPath": "/.../plugins/com.example.my-plugin"
  }
}
```

If a plugin with the same `identifier` is already installed, the dialog displays an **Upgrade** chip and the install replaces the existing folder on approve.

**Alternative: `cider://install-plugin?url=<zip-url>`** — on-device protocol URL that opens the same dialog with no scope / token required. See [Obtaining a token → Option B](#option-b--ciderrequest-auth-deep-link) for the launch pattern (same idea, different host).

---

### Events catalog

#### `GET /api/v2/events/catalog`

**Scope:** none (scope-less)

Machine-readable self-documentation: lists every WebSocket event, the scope matrix per HTTP endpoint, rate-limit settings, and aliases. Useful for generating clients or SDKs dynamically.

---

## WebSocket channel

Connect to the Socket.IO server at the same host/port as the HTTP API. Events fan out on the `API:Playback` channel as `{ type, data }` messages.

**Authentication:** the private room requires `CLIENT_INFO.capi` (an internal token, not a user API token) — but receiving broadcasts on `API:Playback` works for any connected socket, regardless of auth. No scope check is applied to WebSocket messages today.

### Event catalog

| Event | Payload | Trigger |
|---|---|---|
| `playbackStatus.nowPlayingItemDidChange` | Track attributes + artwork | Track changes |
| `playbackStatus.nowPlayingStatusDidChange` | `{ inLibrary, inFavorites }` | Library status changes |
| `playbackStatus.playbackStateDidChange` | `{ attributes, state }` | Play / pause / stop |
| `playbackStatus.playbackTimeDidChange` | `{ currentPlaybackTime, currentPlaybackDuration, currentPlaybackTimeRemaining, isPlaying }` | ~1 s during playback |
| `playbackStatus.transitionStarted` | `{ mode, from, to }` | Crossfade / gapless / automix starts |
| `playbackStatus.transitionCompleted` | `{ mode, from, to }` | Transition finishes |
| `playerStatus.volumeDidChange` | `number` (0–1) | Volume change |
| `playerStatus.repeatModeDidChange` | `number` (0 = none, 1 = one, 2 = all) | Repeat change |
| `playerStatus.shuffleModeDidChange` | `number` (0 = off, 1 = on) | Shuffle change |
| `audioStatus.crossfadeChanged` | Full crossfade state object | Crossfade settings change |
| `audioStatus.automixChanged` | Full automix state object | Automix settings change |
| `audioStatus.atmosChanged` | Full atmos state object | Atmos settings change |
| `audioStatus.qualityChanged` | `{ flavor, flavorLabel }` | Audio quality / codec changes |
| `queueStatus.queueChanged` | `{ position, total }` | Queue modified |
| `queueStatus.smartQueueChanged` | `{ enabled, busy, pending, message }` | Smart Queue status change |

### Minimal subscriber

```js
import { io } from 'socket.io-client';

const socket = io('http://127.0.0.1:10767', { transports: ['websocket', 'polling'] });
socket.on('API:Playback', ({ type, data }) => {
  if (type === 'playerStatus.volumeDidChange') {
    console.log('volume is now', data);
  }
});
```

---

## v1 to v2 migration guide

v1 endpoints are unscoped and remain fully functional. The v2 equivalents below require the listed scope.

| v1 endpoint | v2 equivalent | v2 scope |
|---|---|---|
| `GET /api/v1/playback/active` | `GET /api/v2/playback/state` | `playback` |
| `GET /api/v1/playback/is-playing` | `GET /api/v2/playback/state` | `playback` |
| `GET /api/v1/playback/now-playing` | `GET /api/v2/playback/now-playing` | `playback` |
| `POST /api/v1/playback/play` | `POST /api/v2/playback/play` | `playback` |
| `POST /api/v1/playback/pause` | `POST /api/v2/playback/pause` | `playback` |
| `POST /api/v1/playback/playpause` | `POST /api/v2/playback/toggle` | `playback` |
| `POST /api/v1/playback/stop` | `POST /api/v2/playback/stop` | `playback` |
| `POST /api/v1/playback/next` | `POST /api/v2/playback/next` | `playback` |
| `POST /api/v1/playback/previous` | `POST /api/v2/playback/previous` | `playback` |
| `POST /api/v1/playback/seek` | `POST /api/v2/playback/seek` | `playback` |
| `POST /api/v1/playback/play-url` | `POST /api/v2/playback/play-url` | `playback` |
| `POST /api/v1/playback/play-item-href` | `POST /api/v2/playback/play-href` | `playback` |
| `POST /api/v1/playback/play-item` | `POST /api/v2/playback/play-item` (or `/play-id`) | `playback` |
| `POST /api/v1/playback/play-next` | `POST /api/v2/queue/add-next` | `queue` |
| `POST /api/v1/playback/play-later` | `POST /api/v2/queue/add-later` | `queue` |
| `GET /api/v1/playback/queue` | `GET /api/v2/queue` | `queue` |
| `POST /api/v1/playback/queue/move-to-position` | `POST /api/v2/queue/move` | `queue` |
| `POST /api/v1/playback/queue/change-to-index` | `POST /api/v2/queue/jump` | `queue` |
| `POST /api/v1/playback/queue/remove-by-index` | `DELETE /api/v2/queue/items/:index` | `queue` |
| `POST /api/v1/playback/queue/clear-queue` | `DELETE /api/v2/queue` | `queue` |
| `GET /api/v1/playback/volume` | `GET /api/v2/audio/volume` | `audio` |
| `POST /api/v1/playback/volume` | `PATCH /api/v2/audio/volume` | `audio` |
| `POST /api/v1/playback/add-to-library` | `POST /api/v2/library/now-playing/add` | `library` |
| `POST /api/v1/playback/set-rating` | `PUT /api/v2/library/now-playing/rating` | `library` |
| `POST /api/v1/playback/toggle-repeat` | `POST /api/v2/playback/repeat/toggle` | `playback` |
| `GET /api/v1/playback/repeat-mode` | `GET /api/v2/playback/repeat` | `playback` |
| `POST /api/v1/playback/toggle-shuffle` | `POST /api/v2/playback/shuffle/toggle` | `playback` |
| `GET /api/v1/playback/shuffle-mode` | `GET /api/v2/playback/shuffle` | `playback` |
| `GET /api/v1/playback/autoplay` | `GET /api/v2/playback/autoplay` | `playback` |
| `POST /api/v1/playback/toggle-autoplay` | `POST /api/v2/playback/autoplay/toggle` | `playback` |
| `GET /api/v1/playback/library-status` | `GET /api/v2/library/now-playing/status` | `library` |

### New in v2 (no v1 equivalent)

- `GET /api/v2/playback` — full snapshot
- `GET /api/v2/playback/audio-quality`
- `POST /api/v2/playback/play-collection`
- `POST /api/v2/playback/play-id` — alias of `play-item`
- `GET` / `PATCH /api/v2/playback/rate`
- `PUT /api/v2/playback/repeat`, `/shuffle`, `/autoplay` (set directly, not just toggle)
- `GET` / `PATCH /api/v2/audio/crossfade`
- `GET` / `PATCH /api/v2/audio/automix`
- `GET` / `PATCH /api/v2/audio/atmos`
- `GET` / `POST /api/v2/queue/smart-optimizer`
- `GET /api/v2/library/playlists` + `/:id/songs`
- `GET /api/v2/library/albums` + `/:id/songs`
- `GET /api/v2/library/artists` + `/:id/albums`
- `GET /api/v2/library/songs`
- `GET /api/v2/client/info`
- `GET /api/v2/client/tokens`
- `GET /api/v2/events/catalog`
- `POST /api/v2/auth/request` — unauthenticated token bootstrap
- `POST /api/v2/plugins/install` — scoped, consent-gated plugin install
- `cider://request-auth?…` protocol URL — on-device token consent
- `cider://install-plugin?url=…` protocol URL — on-device plugin install consent
- Scoped token model — see [Authentication & scopes](#authentication--scopes)
