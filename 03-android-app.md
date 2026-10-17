# 03. Android App (and Wear OS)

Native stack: **Kotlin, Jetpack Compose, Google Play services Fused Location Provider, Foreground Service, Room, WorkManager, Health Services (Wear OS), Health Connect, Play Integrity API, FCM, Android Keystore.**

Recommended: `minSdk` 26+, `targetSdk` current stable.

## 1. Modules

```
app/
  auth/        Enrollment, token store (Keystore-backed), Play Integrity
  tracking/    TrackingService (foreground), LocationSource, TrackingState
  health/      HealthConnectReader, WearHeartRateReceiver
  data/        Room: SampleDao, HeartRateDao, OutboxDao
  upload/      UploadWorker (WorkManager), BatchBuilder, RetryPolicy
  config/      RemoteConfig (intervals, thresholds, consent versions)
  sos/         SosService
  ui/          Compose screens: onboarding, consent, dashboard, settings
wear/
  ExerciseService   Health Services ExerciseClient
  DataLayerSender   Sends heart rate to phone
```

## 2. Permissions and manifest

```xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION"/>
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION"/>
<uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION"/>
<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
<!-- Heart rate via Health Connect (only if runner opts in) -->
<uses-permission android:name="android.permission.health.READ_HEART_RATE"/>

<service
    android:name=".tracking.TrackingService"
    android:foregroundServiceType="location"
    android:exported="false"/>
```

**Permission flow:**
1. Consent screen (doc 06).
2. Request fine location (foreground), then notifications (Android 13+).
3. Start the tracking service while the app is visible. A foreground service of type `location` started from the foreground keeps receiving location in the background.
4. Request background location only if testing shows it is needed for your restart scenarios. Google Play requires a declaration and an in-app disclosure for background location, so design the disclosure screen early.
5. If the runner opted in to heart rate, launch the Health Connect permission request (see section 6).
6. Show the battery checklist (section 7).

## 3. Location tracking (Fused Location Provider)

```kotlin
class TrackingService : Service() {
    private lateinit var fused: FusedLocationProviderClient
    private val callback = object : LocationCallback() {
        override fun onLocationResult(result: LocationResult) {
            result.locations.forEach { loc ->
                repo.insertSample(
                    Sample(
                        ts = loc.time,                       // UTC ms of the fix
                        elapsedNanos = loc.elapsedRealtimeNanos,
                        lat = loc.latitude, lon = loc.longitude,
                        alt = if (loc.hasAltitude()) loc.altitude else null,
                        hAcc = if (loc.hasAccuracy()) loc.accuracy else null,
                        vAcc = if (loc.hasVerticalAccuracy()) loc.verticalAccuracyMeters else null,
                        speed = if (loc.hasSpeed()) loc.speed else null,
                        course = if (loc.hasBearing()) loc.bearing else null,
                        mock = if (Build.VERSION.SDK_INT >= 31) loc.isMock else loc.isFromMockProvider,
                        battery = batteryLevel(),
                        seq = repo.nextSeq()
                    )
                )
            }
            UploadScheduler.kick(applicationContext)
        }
    }

    fun startTracking(cfg: TrackingConfig) {
        val req = LocationRequest.Builder(Priority.PRIORITY_HIGH_ACCURACY, cfg.intervalMs)
            .setMinUpdateIntervalMillis(cfg.minIntervalMs)
            .setMinUpdateDistanceMeters(cfg.minDistanceM)
            .setWaitForAccurateLocation(false)
            .build()
        fused.requestLocationUpdates(req, callback, Looper.getMainLooper())
    }
}
```

Start in the foreground:

```kotlin
startForeground(NOTIF_ID, notification,
    ServiceInfo.FOREGROUND_SERVICE_TYPE_LOCATION)   // Android 10+ typed overload
```

Notes:
- `Location.time` is the fix time in UTC milliseconds. `elapsedRealtimeNanos` is monotonic and helps detect clock changes.
- Set a persistent notification that shows status ("Tracking active, last sync 12 s ago") and a Stop action.
- Use `START_STICKY` and a watchdog (a periodic WorkManager job) that restarts the service if the race is active and the service is not running.
- Keep a `PARTIAL_WAKE_LOCK` only if testing shows the CPU sleeping between updates; foreground location services usually do not need it.
- If Google Play services is unavailable (some devices), fall back to `LocationManager` with `GPS_PROVIDER`.

**Adaptive intervals** driven by remote config, battery level, `PowerManager.isPowerSaveMode`, and proximity to checkpoints (shorter interval near them).

## 4. Local storage and upload

**Room entities:** `Sample`, `HeartRateSample`, `OutboxBatch`.

**Upload strategy:**
- Foreground sender in the tracking service: tries a batch every N seconds when connected.
- `UploadWorker` (WorkManager) with `NetworkType.CONNECTED` constraint flushes whatever is left. It runs periodically and is also enqueued when connectivity returns.
- Batch size ~100 samples; oldest first; idempotency key = `runnerId + deviceId + seq`.
- Retry with exponential backoff and jitter; refresh token on `401`; quarantine on validation errors.
- Cap the local buffer (~7 days); delete uploaded rows after the event.

## 5. Heart rate options on Android

An Apple Watch cannot pair with Android. For Android runners, use one of these:

| Path | Real time? | Best for |
|---|---|---|
| **Wear OS watch + Health Services `ExerciseClient`**, forwarded to the phone via the Wearable Data Layer | Yes (every few seconds) | Pixel Watch, Galaxy Watch (Wear OS), others |
| **Health Connect** on the phone (`HeartRateRecord`) | Near real time only if the watch's companion app writes frequently; often delayed | Fitness bands/watches that sync to Health Connect |
| Bluetooth LE Heart Rate Profile (chest strap) | Yes | Runners with BLE chest straps (standard 0x180D service) |

Recommended order: Wear OS app first (closest to the Apple Watch experience), Health Connect as a fallback, BLE chest strap as an optional add-on.

### 5.1 Wear OS exercise session

```kotlin
val exerciseClient = HealthServices.getClient(context).exerciseClient

val config = ExerciseConfig.builder(ExerciseType.RUNNING)
    .setDataTypes(setOf(DataType.HEART_RATE_BPM))
    .setIsAutoPauseAndResumeEnabled(false)
    .setIsGpsEnabled(false)          // phone provides position
    .build()

exerciseClient.setUpdateCallback(object : ExerciseUpdateCallback {
    override fun onExerciseUpdateReceived(update: ExerciseUpdate) {
        update.latestMetrics.getData(DataType.HEART_RATE_BPM).lastOrNull()?.let { p ->
            val bpm = p.value
            // DataPoint times are relative to boot; convert to wall-clock time
            val bootInstant = Instant.ofEpochMilli(
                System.currentTimeMillis() - SystemClock.elapsedRealtime())
            val ts = p.getStartInstant(bootInstant)
            sendToPhone(HrSample(ts.toEpochMilli(), bpm))
        }
    }
    override fun onLapSummaryReceived(lapSummary: ExerciseLapSummary) {}
    override fun onRegistered() {}
    override fun onRegistrationFailed(throwable: Throwable) {}
    override fun onAvailabilityChanged(dataType: DataType<*, *>, availability: Availability) {}
})
exerciseClient.startExerciseAsync(config)
```

Wear OS needs the `BODY_SENSORS` permission (and the Health Services capabilities check with `exerciseClient.getCapabilitiesAsync()`). The exercise runs in a foreground service on the watch with an ongoing-activity notification.

Send to the phone with `MessageClient` (live, when connected) and `DataClient` (queued). The phone's `WearableListenerService` receives them and writes to Room.

### 5.2 Health Connect fallback (phone-side)

```kotlin
val hc = HealthConnectClient.getOrCreate(context)
val response = hc.readRecords(
    ReadRecordsRequest(
        HeartRateRecord::class,
        timeRangeFilter = TimeRangeFilter.between(lastSync, now)
    )
)
```

- Use `HeartRateRecord` and read samples (`records.flatMap { it.samples }`) for the time window since the last sync.
- Reading from the background requires an extra permission (`READ_HEALTH_DATA_IN_BACKGROUND`) and is not available everywhere. Plan on reading while the app's service is active in the foreground-service context and treat results as delayed.
- Health Connect requires a privacy-policy rationale activity that explains why the app reads heart rate.

### 5.3 Behavior notes

- Mark the data `source` (`wearos`, `healthconnect`, `ble`) so the backend and medical dashboard know how fresh and reliable it is.
- Keep a local buffer on the watch; send queued data when the phone reconnects.
- Heart rate readings may drop out when the watch is loose; send a "no reading" marker so the backend can tell "sensor off" from "no data".

## 6. Play Integrity

1. At enrollment, the app requests an integrity token (Play Integrity API) with a server-provided nonce.
2. Backend decodes and verifies the verdict (device integrity, app recognized) and records the result with the device.
3. Repeat on token refresh or before high-value actions (SOS acknowledgement may skip this to avoid blocking an emergency).

## 7. Battery and OEM background restrictions

Many Android brands (Xiaomi, Oppo, Vivo, Huawei, Samsung "sleeping apps") aggressively kill background apps. Build a **battery checklist** screen:

1. Check `PowerManager.isIgnoringBatteryOptimizations(packageName)`; guide the runner to the system setting for unrestricted battery use. (Google Play restricts apps from requesting the exemption directly unless they qualify, so prefer guiding the user to the setting.)
2. Brand-specific instructions with screenshots (autostart, lock in recents, "no restrictions").
3. A "tracking health check" that runs for 2 minutes in onboarding with the screen off and reports whether samples kept arriving.
4. Detect gaps in the field: if the service was killed (gap > N minutes while the race is active), log it and show a notification to reopen the app.

## 8. SOS and notifications

- Press-and-hold SOS button; calls `POST /v1/sos` immediately with the last fix, even if the batch uploader is backed up.
- FCM high-priority messages for medical acknowledgement and organizer messages.
- Use a dedicated high-importance notification channel for SOS responses.

## 9. UI screens

Same flow as iOS: enrollment, consent, permissions + battery checklist, ready test, tracking dashboard, finished summary, privacy and data. Use Compose and Material 3. The tracking screen should work with large text and in bright sunlight (high contrast).

## 10. Testing checklist

- Screen-off, 2+ hour run on several brands (Samsung, Xiaomi, Oppo/Vivo, Pixel)
- Airplane mode then restore: back-fill order and de-duplication
- Swipe-away app from recents with the foreground service running
- Revoke location mid-run and verify the warning
- Mock-location app on: verify the flag arrives at the server
- Wear OS disconnect/reconnect, with and without Health Connect
- Doze mode and battery saver
- Play Integrity success and failure paths
