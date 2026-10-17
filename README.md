# Live Race Tracker: Documentation Set

A live tracking platform for ultra-marathon events, modelled on public tracking sites such as Strackking. Runners carry their own phones (and optionally an Apple Watch). The apps use **native Apple and Android location and health APIs** to send time-stamped positions and heart rate to a backend. Spectators watch on a web map; organizers and medical staff watch on an admin dashboard.

## Documents

| # | File | Contents |
|---|------|----------|
| 1 | [01-system-architecture.md](01-system-architecture.md) | Components, data flow, tech stack, deployment, scaling |
| 2 | [02-ios-app-and-watch.md](02-ios-app-and-watch.md) | iPhone app + Apple Watch app: Core Location, HealthKit, workout mirroring |
| 3 | [03-android-app.md](03-android-app.md) | Android app: Fused Location, foreground service, Health Services / Health Connect |
| 4 | [04-backend-and-data-model.md](04-backend-and-data-model.md) | API contract, ingestion pipeline, database schema, realtime fan-out |
| 5 | [05-race-logic.md](05-race-logic.md) | Route snapping, checkpoints, progress, pace/ETA, rankings, runner state machine |
| 6 | [06-heart-rate-privacy-consent.md](06-heart-rate-privacy-consent.md) | Heart-rate monitoring logic, alerts, consent, privacy, store compliance |
| 7 | [07-web-viewer-and-admin.md](07-web-viewer-and-admin.md) | Public map site, weather layers, elevation profile, admin/medical dashboards |

## Key decisions

1. **Native, not cross-platform.** Swift (iOS/watchOS) and Kotlin (Android/Wear OS). Background location and health sensors are the hardest parts of this product, and native APIs give the most reliable behavior.
2. **Device time is authoritative.** Each sample carries the GNSS fix time (`CLLocation.timestamp` / `Location.time`), not the upload time. The backend sorts by sample time, so late uploads after a signal gap are fine.
3. **Offline-first clients.** Every sample is written to a local database first, then uploaded in batches. Nothing is lost in dead zones.
4. **Heart rate comes from the watch workout session.** On Apple Watch this is `HKWorkoutSession` + `HKLiveWorkoutBuilder`, mirrored to the iPhone. On Android it comes from Wear OS Health Services (live) or Health Connect (best-effort).
5. **Health data is separate and consented.** Heart rate is stored in its own table, visible only to runner and medical staff by default, and shown publicly only if the runner explicitly opts in.
6. **Server-driven configuration.** Reporting intervals, battery modes, and thresholds are delivered by the server so they can change during a race without an app update.

## Platform support summary

| Capability | iPhone | Apple Watch | Android phone | Wear OS watch |
|---|---|---|---|---|
| Position and time | Core Location | (optional, workout GPS) | Fused Location Provider | (optional) |
| Heart rate (live) | via watch mirroring | HealthKit workout | via watch / Health Connect | Health Services |
| Background tracking | Location background mode | Workout session | Foreground service | Exercise session |
| Offline buffering | SQLite/SwiftData | local buffer | Room | local buffer |
| Push | APNs | n/a | FCM | n/a |

> **Note:** An Apple Watch only pairs with an iPhone, so Apple Watch heart rate is available to iOS runners only. Android runners can use a Wear OS watch (e.g. Pixel Watch, Galaxy Watch) through the equivalent path described in `03-android-app.md`.

## Suggested build order

1. Backend ingestion + database + simulator that replays a GPX as fake runners
2. Web viewer (map, runners, checkpoints)
3. iOS app (location only), then Android app (location only)
4. Race logic (snapping, checkpoints, ETA)
5. Apple Watch / Wear OS heart rate
6. Medical dashboard + alerts + consent flows
7. Weather, elevation profile, sponsors, donation
8. Field test, battery test, load test

## Disclaimer

Heart-rate monitoring in this system is an **awareness aid for race staff**, not a medical device and not a diagnostic tool. Legal and store-policy details (GDPR, Indonesia's PDP Law, Apple and Google health-data policies) change over time and should be verified with counsel before launch.
