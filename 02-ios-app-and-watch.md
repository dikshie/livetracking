# 02. iOS App and Apple Watch App

Native stack: **Swift, SwiftUI, Core Location, HealthKit, WatchConnectivity, BackgroundTasks, URLSession, Keychain, DeviceCheck (App Attest), APNs.**

Minimum targets (recommended): iOS 17 / watchOS 10 to use mirrored workout sessions. Older versions need the WatchConnectivity fallback (section 5).

## 1. Modules

```
App/
  Auth/            Enrollment, token storage (Keychain), App Attest
  Tracking/        LocationTracker, TrackingSession, TrackingState
  Health/          HeartRateReceiver (mirrored session), HealthAuthorization
  Storage/         SQLite (GRDB) or SwiftData: samples, outbox, settings
  Upload/          Uploader, BatchBuilder, RetryPolicy
  Config/          RemoteConfig (intervals, thresholds, consent versions)
  SOS/             SOSService
  UI/              Onboarding, Consent, Dashboard, Settings
WatchApp/
  WorkoutManager   HKWorkoutSession + HKLiveWorkoutBuilder
  Mirroring        Send heart-rate to phone
  UI               Start/stop, live HR, status
```

## 2. Permissions and capabilities

**Xcode capabilities:** Background Modes (Location updates), HealthKit, Push Notifications, App Attest (via entitlement), Associated Domains (optional deep links).

**Info.plist keys (iPhone):**

| Key | Purpose |
|---|---|
| `NSLocationWhenInUseUsageDescription` | First location prompt |
| `NSLocationAlwaysAndWhenInUseUsageDescription` | Upgrade to "Always" for background tracking |
| `NSHealthShareUsageDescription` | Read heart rate |
| `NSHealthUpdateUsageDescription` | Save the workout |
| `UIBackgroundModes` = `location` | Keep tracking when screen is off |

Write the usage strings in plain language: why the app needs it, who can see it, and that the runner can stop at any time.

**Permission flow (onboarding):**
1. Explain tracking and show the consent screen (see doc 06). Store the consent version and timestamp.
2. Request When-In-Use location, then request Always.
3. Request HealthKit authorization (heart rate read, workout write), only if the runner opted in to heart-rate sharing.
4. Request notification permission.
5. Show a battery checklist (charge, power bank, Low Power Mode guidance).

## 3. Location tracking (Core Location)

```swift
final class LocationTracker: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    private let store: SampleStore

    func start(config: TrackingConfig) {
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.activityType = .fitness
        manager.distanceFilter = config.distanceFilterMeters      // e.g. 5-10 m
        manager.pausesLocationUpdatesAutomatically = false
        manager.allowsBackgroundLocationUpdates = true
        manager.showsBackgroundLocationIndicator = true
        manager.startUpdatingLocation()
        // Relaunch the app if the OS terminated it
        manager.startMonitoringSignificantLocationChanges()
    }

    func locationManager(_ m: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        for loc in locations where loc.horizontalAccuracy >= 0 {
            let sim = loc.sourceInformation?.isSimulatedBySoftware ?? false
            let accessory = loc.sourceInformation?.isProducedByAccessory ?? false
            store.insert(Sample(
                ts: loc.timestamp,                 // GNSS fix time (authoritative)
                lat: loc.coordinate.latitude,
                lon: loc.coordinate.longitude,
                alt: loc.altitude,
                hAcc: loc.horizontalAccuracy,
                vAcc: loc.verticalAccuracy,
                speed: loc.speed >= 0 ? loc.speed : nil,
                course: loc.course >= 0 ? loc.course : nil,
                mock: sim,
                accessory: accessory,
                battery: UIDevice.current.batteryLevel,
                seq: store.nextSeq()
            ))
        }
        Uploader.shared.kick()
    }
}
```

Notes:
- `loc.timestamp` is the fix time. Never replace it with `Date()`.
- `sourceInformation` (iOS 15+) exposes whether the location was simulated by software, which is the iOS equivalent of Android's mock flag.
- iOS 17+ alternative: `CLLocationUpdate.liveUpdates(.fitness)` in an async loop, together with a `CLBackgroundActivitySession` to keep it alive in the background. Either approach is valid; the delegate API above works on older iOS versions.
- Filter in the app only for obviously bad fixes (`horizontalAccuracy < 0`). Keep everything else and let the server apply the quality rules (doc 05).

**Adaptive intervals:** adjust `distanceFilter` and, when needed, switch between `kCLLocationAccuracyBest` and `kCLLocationAccuracyNearestTenMeters` based on remote config, battery level, and `ProcessInfo.processInfo.isLowPowerModeEnabled`. Never drop below the minimum accuracy the event requires.

**Auto-stop:** stop updates when the runner finishes (server says `FINISHED`), when the event time limit passes, or when the runner taps Stop.

## 4. Heart rate on Apple Watch (HealthKit workout session)

Heart rate is sampled at a high rate only while an **active workout session** is running. So the watch app starts an outdoor running workout, reads the live heart rate from the builder, and sends it to the phone.

### 4.1 Authorization

```swift
let hr = HKQuantityType(.heartRate)
let toRead: Set<HKObjectType> = [hr]
let toShare: Set<HKSampleType> = [HKObjectType.workoutType()]
try await healthStore.requestAuthorization(toShare: toShare, read: toRead)
```

Authorization for read access cannot be queried directly (iOS hides whether read was denied). Handle the "no data arrives" case in the UI with a "check Health permissions" hint.

### 4.2 Workout session on the watch

```swift
final class WorkoutManager: NSObject, ObservableObject,
    HKWorkoutSessionDelegate, HKLiveWorkoutBuilderDelegate {

    let store = HKHealthStore()
    var session: HKWorkoutSession?
    var builder: HKLiveWorkoutBuilder?

    func start() async throws {
        let cfg = HKWorkoutConfiguration()
        cfg.activityType = .running
        cfg.locationType = .outdoor

        let s = try HKWorkoutSession(healthStore: store, configuration: cfg)
        let b = s.associatedWorkoutBuilder()
        b.dataSource = HKLiveWorkoutDataSource(healthStore: store, workoutConfiguration: cfg)
        s.delegate = self
        b.delegate = self
        session = s; builder = b

        try await s.startMirroringToCompanionDevice()   // watchOS 10+
        let t = Date()
        s.startActivity(with: t)
        try await b.beginCollection(at: t)
    }

    func workoutBuilder(_ b: HKLiveWorkoutBuilder, didCollectDataOf types: Set<HKSampleType>) {
        guard types.contains(HKQuantityType(.heartRate)),
              let stats = b.statistics(for: HKQuantityType(.heartRate)),
              let q = stats.mostRecentQuantity(),
              let end = stats.mostRecentQuantityDateInterval()?.end else { return }
        let bpm = q.doubleValue(for: HKUnit.count().unitDivided(by: .minute()))
        let payload = HRSample(ts: end, bpm: bpm)
        try? session?.sendToRemoteWorkoutSession(data: JSONEncoder().encode(payload))
        // Also keep a small local ring buffer on the watch in case mirroring drops
    }
    func workoutBuilderDidCollectEvent(_ b: HKLiveWorkoutBuilder) {}
    func workoutSession(_ s: HKWorkoutSession, didChangeTo to: HKWorkoutSessionState,
                        from: HKWorkoutSessionState, date: Date) {}
    func workoutSession(_ s: HKWorkoutSession, didFailWithError error: Error) {}
}
```

### 4.3 Receiving on the iPhone (mirrored session)

```swift
healthStore.workoutSessionMirroringStartHandler = { mirrored in
    mirrored.delegate = self      // HKWorkoutSessionDelegate
    self.mirroredSession = mirrored
}

func workoutSession(_ s: HKWorkoutSession,
                    didReceiveDataFromRemoteWorkoutSession data: [Data]) {
    for d in data {
        if let hr = try? JSONDecoder().decode(HRSample.self, from: d) {
            store.insertHeartRate(hr)         // local DB; uploaded with next batch
            Uploader.shared.kick()
        }
    }
}
```

The iPhone app can also launch the watch app with `healthStore.startWatchApp(with:)` when the runner taps Start on the phone.

### 4.4 Behavior and limits

- During an active workout the watch app keeps running in the background and the watch records heart rate automatically. Use the ordinary workout lifecycle: start, pause (optional), end.
- Heart rate has no guaranteed per-second rate; expect updates every few seconds. Store whatever arrives with its real timestamp.
- Sensor quality varies: loose strap, cold, sweat, or arm movement can produce gaps or noisy values. Attach a quality flag if the platform provides one, and let the backend treat missing data differently from low data (doc 06).
- For ultras longer than a watch battery lasts, show watch battery in the app if available and warn the runner in the pre-race checklist. Watch Low Power Mode may reduce sensor frequency.
- If the iPhone and watch disconnect, the watch workout continues locally. On reconnect, send buffered samples (or read them from HealthKit with an `HKAnchoredObjectQuery` for the time window that was missed).

## 5. Fallback paths

| Situation | Fallback |
|---|---|
| iOS < 17 / watchOS < 10 (no mirroring) | Watch sends heart rate with `WCSession.sendMessage` (live, when reachable) and `transferUserInfo` (queued) |
| Watch app not running | Phone reads recent heart-rate samples from HealthKit with `HKAnchoredObjectQuery` (not real time, but fills gaps) |
| Runner without a watch | Location-only tracking |
| Runner declined heart-rate consent | Never request HealthKit read permission; the upload payload has no heart-rate array |

## 6. Local storage and upload

**Tables (SQLite/GRDB or SwiftData):**
- `samples(seq, ts, lat, lon, alt, hAcc, vAcc, speed, course, mock, battery, uploaded)`
- `heart_rates(id, ts, bpm, quality, uploaded)`
- `outbox(batchId, createdAt, attempts, lastError)`
- `settings(key, value)`

**Uploader logic:**
1. Triggered on a timer (from remote config, e.g. every 15 s), on every N new samples, on connectivity regained (`NWPathMonitor`), and on app foreground.
2. Builds a batch of up to ~100 samples + heart-rate samples (oldest first).
3. `POST /v1/telemetry/batch` using a `URLSession` configured for background use where possible (so uploads can complete if the app is suspended).
4. On `200`, mark rows uploaded. On `401`, refresh token. On `429`/`5xx`, retry with exponential backoff and jitter. On `4xx` validation errors, quarantine the batch and report diagnostics.
5. Register a `BGProcessingTask` as an extra chance to flush the outbox.
6. Cap the local buffer (e.g. 7 days) and delete uploaded rows after the event ends.

## 7. App Attest

1. At enrollment, generate an App Attest key (`DCAppAttestService.generateKey`).
2. Request an attestation (`attestKey(_:clientDataHash:)`) over a server-provided challenge and send it with the enrollment request.
3. Server verifies it against Apple's attestation rules and stores the key ID with the device record.
4. For sensitive endpoints (SOS, enrollment refresh), optionally send assertions (`generateAssertion`).

App Attest raises the cost of automated fakes but does not replace the location plausibility checks on the server.

## 8. SOS

- Large, hard-to-trigger-accidentally button (press and hold 2 s).
- On trigger: read the latest fix, call `POST /v1/sos` directly with high priority, vibrate on acknowledgement, and show "Help requested; staff notified".
- If the network is unavailable, queue it, keep retrying, and show clear status. Make clear in the UI that SOS is **not** a substitute for local emergency services.
- Medical staff can respond via push (APNs) with an acknowledgement and instructions.

## 9. UI screens

1. Enrollment (bib + code)
2. Consent (location, optional heart rate, with details)
3. Permission setup + battery checklist
4. Ready (test GPS, test watch connection, test upload)
5. Tracking dashboard: status (GPS, signal, last sync), distance, pace, heart rate if enabled, battery, Stop button, SOS
6. Finished summary
7. Privacy and data (view consent, revoke heart-rate sharing, request deletion)

## 10. Testing checklist

- Airplane-mode tunnel test, then restore connectivity and verify back-fill
- Force-quit and relaunch during a run
- Low Power Mode on, then off
- Revoke Always permission mid-run: app must warn and notify the runner
- Revoke HealthKit permission mid-run: stop heart-rate path cleanly
- 8+ hour battery test with and without Apple Watch
- Simulated location (Xcode) to check the mock flag is sent
- Watch disconnect and reconnect during a workout
