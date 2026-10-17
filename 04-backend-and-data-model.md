# 04. Backend, API, and Data Model

## 1. Services

| Service | Role |
|---|---|
| `ingest-api` | Authenticates runners, validates and de-duplicates batches, enqueues them |
| `worker` | Consumes the queue: quality filter, route snapping, checkpoints, pace/ETA, alert rules; writes to DB and Redis |
| `public-api` | Read-only endpoints and live stream for spectators (no health data) |
| `admin-api` | Event setup, runner management, medical views (role-based) |
| `notifier` | APNs, FCM, optional SMS for SOS and alerts |
| `simulator` | Replays GPX tracks as fake runners for testing |

Language: Go or Node.js/TypeScript. Queue: Redis Streams (simple) or NATS JetStream. DB: PostgreSQL 16 + PostGIS + TimescaleDB. Cache and pub/sub: Redis 7.

## 2. Authentication

**Enrollment** (`POST /v1/auth/enroll`)
- Request: `{ bib, code, platform, deviceModel, osVersion, appVersion, attestation }`
- `code` is a one-time code printed on the bib or sent by email/SMS.
- `attestation` is the App Attest object (iOS) or Play Integrity token (Android).
- Response: `{ accessToken, refreshToken, runner: {id, name, bib, category}, config }`
- Tokens are bound to one device. Re-enrolling on a new device revokes the old one (organizer can override).

**Access tokens:** short-lived JWT (e.g. 1 h). **Refresh tokens:** rotating, stored hashed.

**Roles:** `runner`, `organizer`, `medical`, `admin`. Public endpoints need no auth.

## 3. API contract (v1)

### 3.1 Runner endpoints

| Method | Path | Purpose |
|---|---|---|
| POST | `/v1/auth/enroll` | Enroll device |
| POST | `/v1/auth/refresh` | Refresh token |
| GET | `/v1/runner/me/config` | Server-driven config (intervals, thresholds, consent versions) |
| POST | `/v1/consents` | Record consent (location, heart rate, public sharing) |
| DELETE | `/v1/consents/{scope}` | Revoke a consent |
| POST | `/v1/tracking/start` | Mark tracking started (runner pressed Start) |
| POST | `/v1/tracking/stop` | Mark tracking stopped |
| POST | `/v1/telemetry/batch` | Upload samples |
| POST | `/v1/sos` | Raise SOS (priority path) |
| GET | `/v1/runner/me/status` | Race status, last checkpoint, messages |

### 3.2 Telemetry batch

```http
POST /v1/telemetry/batch
Authorization: Bearer <accessToken>
Idempotency-Key: <batchId>
Content-Type: application/json
```

```json
{
  "batchId": "b7f0c0c2-4f64-4f6a-9a1d-7d1d7a7e0f11",
  "deviceId": "d-123",
  "sentAt": "2026-10-17T05:41:12Z",
  "positions": [
    {
      "seq": 10231,
      "ts": "2026-10-17T05:40:58.412Z",
      "lat": -6.9147,
      "lon": 107.6098,
      "alt": 812.4,
      "hAcc": 6.5,
      "vAcc": 4.0,
      "speed": 2.9,
      "course": 118.0,
      "mock": false,
      "battery": 0.62
    }
  ],
  "heartRate": [
    {
      "ts": "2026-10-17T05:40:57Z",
      "bpm": 152,
      "source": "applewatch",
      "quality": "ok"
    }
  ],
  "deviceStatus": {
    "lowPower": false,
    "watchBattery": 0.71,
    "network": "cell",
    "appState": "background"
  }
}
```

**Response**

```json
{
  "accepted": { "positions": 25, "heartRate": 20 },
  "duplicates": { "positions": 0, "heartRate": 0 },
  "rejected": [],
  "serverTime": "2026-10-17T05:41:12.730Z",
  "config": { "version": 14 }
}
```

If `config.version` is newer than the app's, the app fetches `/v1/runner/me/config`.

### 3.3 SOS

```json
POST /v1/sos
{ "ts": "2026-10-17T05:44:10Z", "lat": -6.9151, "lon": 107.6102, "hAcc": 8.0,
  "note": "optional free text", "lastHeartRate": { "ts": "...", "bpm": 181 } }
```

Response: `{ "sosId": "…", "status": "received" }`. The app polls or receives a push for `acknowledged`.

### 3.4 Public endpoints (no health data)

| Method | Path | Purpose |
|---|---|---|
| GET | `/v1/public/events/{slug}` | Event info, routes, checkpoints, sponsors, weather location |
| GET | `/v1/public/events/{slug}/snapshot` | Latest position and progress for all (or filtered) runners; CDN-cached 3-5 s |
| GET | `/v1/public/events/{slug}/stream` | SSE stream of position updates |
| GET | `/v1/public/runners/{id}/track` | Historical track (simplified) |
| GET | `/v1/public/routes/{id}/profile` | Elevation profile and gradient stats |

### 3.5 Admin and medical endpoints (role-based)

Event CRUD, GPX upload, checkpoint editing, runner CSV import, enrollment-code generation, device list with battery and last-seen, SOS queue with acknowledge/resolve, heart-rate views and alert acknowledgements, sponsor management, public-tracking kill switch.

## 4. Ingestion pipeline

```
HTTP batch
  -> auth (JWT) + rate limit (per runner)
  -> schema validation
  -> clock sanity: reject ts > now+5 min, ts < event.start - 1 day
  -> idempotency: batchId + (runnerId, deviceId, seq) unique
  -> consent gate: drop heartRate if consent not active (and report)
  -> enqueue to stream "telemetry"
  -> 200 with accepted/duplicate/rejected counts
```

**Worker per sample:**
1. Order by `ts` within the runner (small reorder window; late data triggers recompute of derived values).
2. Quality filter (doc 05): accuracy, impossible speed, jump detection.
3. Snap to route, compute `route_dist_m` and `off_route_m`.
4. Update progress, pace, ETA; detect checkpoint crossings.
5. Insert into `positions`.
6. If the sample is the newest for the runner, update Redis latest state and publish an event.
7. Evaluate alert rules (silence, off route, stationary; heart-rate rules go through the medical rule engine in doc 06).

Late back-filled data must **not** move a runner's "latest position" backward. Only update the latest if `ts` is newer than the stored latest.

## 5. Database schema (PostgreSQL)

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS timescaledb;

CREATE TABLE events (
  id            bigserial PRIMARY KEY,
  slug          text UNIQUE NOT NULL,
  name          text NOT NULL,
  starts_at     timestamptz NOT NULL,
  ends_at       timestamptz NOT NULL,
  timezone      text NOT NULL,
  public_enabled boolean NOT NULL DEFAULT true,
  weather_lat   double precision, weather_lon double precision
);

CREATE TABLE routes (
  id            bigserial PRIMARY KEY,
  event_id      bigint REFERENCES events(id),
  name          text NOT NULL,                 -- e.g. "100K"
  color         text,
  geom          geometry(LineStringZ, 4326) NOT NULL,
  length_m      double precision NOT NULL,
  elev_gain_m   double precision,
  elev_loss_m   double precision,
  profile       jsonb                          -- [[dist_m, elev_m, gradient_pct], ...]
);

CREATE TABLE checkpoints (
  id            bigserial PRIMARY KEY,
  route_id      bigint REFERENCES routes(id),
  name          text NOT NULL,
  kind          text NOT NULL,                 -- start | aid | cutoff | finish
  position      geometry(Point, 4326) NOT NULL,
  route_dist_m  double precision NOT NULL,
  radius_m      real NOT NULL DEFAULT 40,
  cutoff_at     timestamptz
);

CREATE TABLE runners (
  id            bigserial PRIMARY KEY,
  event_id      bigint REFERENCES events(id),
  route_id      bigint REFERENCES routes(id),
  bib           text NOT NULL,
  full_name     text NOT NULL,
  category      text,
  gender        text,
  birth_year    int,
  max_hr_bpm    int,                           -- optional, runner-provided
  status        text NOT NULL DEFAULT 'REGISTERED',
  enrollment_code_hash text,
  UNIQUE (event_id, bib)
);

CREATE TABLE devices (
  id            text PRIMARY KEY,              -- deviceId from app
  runner_id     bigint REFERENCES runners(id),
  platform      text NOT NULL,                 -- ios | android
  model         text, os_version text, app_version text,
  push_token    text,
  attestation_ok boolean,
  last_seen_at  timestamptz,
  last_battery  real,
  revoked_at    timestamptz
);

CREATE TABLE consents (
  id            bigserial PRIMARY KEY,
  runner_id     bigint REFERENCES runners(id),
  scope         text NOT NULL,                 -- location | heart_rate | public_heart_rate | public_profile
  policy_version text NOT NULL,
  granted_at    timestamptz NOT NULL,
  revoked_at    timestamptz,
  ip_or_device  text
);

CREATE TABLE positions (
  runner_id     bigint NOT NULL,
  ts            timestamptz NOT NULL,          -- sample (GNSS) time
  received_at   timestamptz NOT NULL,
  geom          geometry(Point, 4326) NOT NULL,
  alt_m         real, h_acc_m real, v_acc_m real,
  speed_ms      real, course_deg real,
  route_dist_m  double precision,              -- snapped progress
  off_route_m   real,
  quality       smallint NOT NULL DEFAULT 0,   -- 0 ok, 1 low accuracy, 2 suspect, 3 rejected
  mock          boolean NOT NULL DEFAULT false,
  battery       real,
  seq           bigint NOT NULL,
  device_id     text,
  PRIMARY KEY (runner_id, ts, seq)
);
SELECT create_hypertable('positions', 'ts');
CREATE INDEX ON positions USING gist (geom);

CREATE TABLE heart_rates (                    -- separate table, stricter access
  runner_id     bigint NOT NULL,
  ts            timestamptz NOT NULL,
  bpm           smallint NOT NULL,
  source        text NOT NULL,                 -- applewatch | wearos | healthconnect | ble
  quality       text,
  received_at   timestamptz NOT NULL,
  PRIMARY KEY (runner_id, ts, source)
);
SELECT create_hypertable('heart_rates', 'ts');

CREATE TABLE checkpoint_passes (
  runner_id     bigint, checkpoint_id bigint,
  passed_at     timestamptz NOT NULL,
  method        text NOT NULL,                 -- gps | manual | scan
  PRIMARY KEY (runner_id, checkpoint_id)
);

CREATE TABLE alerts (
  id            bigserial PRIMARY KEY,
  runner_id     bigint, kind text NOT NULL,    -- silent | off_route | stationary | hr_high | hr_low | hr_missing | sos
  severity      text NOT NULL,                 -- info | warn | critical
  opened_at     timestamptz NOT NULL, details jsonb,
  acknowledged_by bigint, acknowledged_at timestamptz, resolved_at timestamptz
);

CREATE TABLE sos_events (
  id bigserial PRIMARY KEY, runner_id bigint, ts timestamptz NOT NULL,
  geom geometry(Point, 4326), h_acc_m real, note text, last_hr smallint,
  status text NOT NULL DEFAULT 'RECEIVED',     -- RECEIVED | ACKNOWLEDGED | DISPATCHED | RESOLVED
  handled_by bigint, handled_at timestamptz
);

CREATE TABLE audit_log (
  id bigserial PRIMARY KEY, actor_id bigint, actor_role text,
  action text NOT NULL, subject_runner_id bigint, at timestamptz NOT NULL DEFAULT now(), details jsonb
);

CREATE TABLE sponsors (
  id bigserial PRIMARY KEY, event_id bigint, name text, image_url text, link_url text, position int
);
```

**Retention:** heart-rate and position raw data deleted or anonymized after a defined period (e.g. 30-90 days), unless the runner asks to keep their own history. Use Timescale retention policies. Keep results (checkpoint passes, finish times) per event rules.

**Compression:** enable Timescale compression on chunks older than 1 day.

## 6. Redis keys

| Key | Value |
|---|---|
| `evt:{id}:runner:{id}:latest` | Latest position JSON (lat, lon, ts, routeDist, status, speed) |
| `evt:{id}:snapshot` | Full snapshot JSON (rebuilt every 2-3 s) |
| `evt:{id}:runner:{id}:hr` | Latest heart rate (medical access only) |
| `evt:{id}:updates` | Pub/sub channel of position updates (public-safe payload) |
| `rate:{runner}` | Rate limit counters |

**Public payload excludes heart rate unless the runner has `public_heart_rate` consent active.**

## 7. Realtime fan-out

- **Primary approach:** clients call the CDN-cached snapshot endpoint every 3-5 seconds, then optionally open an SSE stream for deltas. Most viewers can run on cached snapshots alone.
- **SSE** is simpler than WebSocket for one-way updates and works through proxies; use WebSocket if you later need two-way chat.
- Each SSE message: `{ id, bib, lat, lon, ts, routeDist, state }`, batched and throttled to at most one message per runner per second.
- Medical dashboard uses a separate authenticated stream that includes heart rate and alerts.

## 8. Config returned to apps

```json
{
  "version": 14,
  "tracking": {
    "intervalSec": 15, "minDistanceM": 10,
    "lowBattery": { "threshold": 0.20, "intervalSec": 60 },
    "nearCheckpoint": { "radiusM": 300, "intervalSec": 5 }
  },
  "upload": { "everySec": 15, "maxBatch": 100 },
  "heartRate": { "enabled": true, "uploadEverySec": 15 },
  "consents": { "locationVersion": "2026-10-01", "heartRateVersion": "2026-10-01" },
  "sos": { "holdMs": 2000 }
}
```

## 9. Rate limits and protection

- Per-runner: e.g. 10 batches/min, 1 SOS/10 s.
- Per-IP limits on public endpoints; CDN caching to absorb spikes.
- Body size cap (e.g. 256 KB).
- Reject tokens from revoked devices.
- Kill switch: admin can disable public tracking instantly (returns a static "tracking paused" message).

## 10. Testing

- Replay simulator with 2,000 virtual runners
- Out-of-order and duplicate batch tests
- Time travel tests (device clock wrong)
- Load test snapshot and SSE endpoints
- Permission tests: public endpoints must never return heart rate; medical endpoints must write audit records
