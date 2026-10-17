# 06. Heart Rate Monitoring, Privacy, and Consent

Heart rate is **health data**. This document defines how it is collected, monitored, shown, and protected. The design assumes runners give explicit, informed consent, and still applies data minimization and access controls.

> This is an engineering design, not legal advice. Have counsel review it against GDPR (if any EU runners), Indonesia's Personal Data Protection Law (UU PDP) and other applicable rules, plus current Apple and Google store policies.

## 1. Principles

1. **Consent first, per purpose.** Separate consents for location tracking, heart-rate collection, and public display of heart rate.
2. **Least privilege.** Default visibility of heart rate: the runner and the medical team only.
3. **Minimization.** Collect heart rate only during the race; stop collection on finish or Stop.
4. **Transparency.** Plain-language explanation of what is collected, who sees it, how long it is kept.
5. **Revocable.** Runner can withdraw heart-rate consent at any time; collection stops and (per policy) data is deleted or anonymized.
6. **No secondary use.** No advertising, no sale, no use for unrelated analytics. Apple's App Store rules prohibit using HealthKit data for advertising or data mining, and Google Play and Health Connect have similar restrictions.

## 2. Consent scopes

| Scope | Required? | What it allows |
|---|---|---|
| `location` | Required to be tracked | Collect position and time; show position on the public map |
| `heart_rate` | Optional | Collect heart rate from watch/sensor; visible to medical staff and the runner |
| `public_heart_rate` | Optional, off by default | Show current heart rate on the public page |
| `public_profile` | Optional | Show name and photo instead of bib only |
| `core_temp` | Optional | Collect core temperature from an external sensor; visible to the runner and medical staff only |

**Consent record fields:** runner ID, scope, policy version, text shown (hash), granted time, revoked time, device ID, app version. Re-prompt when the policy version changes.

**Consent UX:**
- A dedicated screen per scope (not buried in terms of service).
- Clear options: "Share with race medical team only", "Also show on public tracking page".
- Short explanations with examples ("Medical staff see your heart rate so they can spot signs of trouble").
- A visible, always-available "Privacy and data" screen to review or revoke.
- Minors: apply the organizer's age policy and require guardian consent where law demands it.

## 3. Collection

| Platform | Source | Details |
|---|---|---|
| iPhone + Apple Watch | HealthKit workout session (`HKWorkoutSession` + `HKLiveWorkoutBuilder`) mirrored to the phone | See doc 02 |
| Android + Wear OS | Health Services `ExerciseClient` via Data Layer | See doc 03 |
| Android (fallback) | Health Connect `HeartRateRecord` | Delayed |
| Either | BLE chest strap | Optional |

Rules:
- Request HealthKit / Health Connect / body-sensor permissions **only after** the runner opts in to the `heart_rate` consent.
- Stop collection and delete local buffers when consent is revoked.
- Do not write heart rate to iCloud, analytics SDKs, or crash logs. Strip heart rate from logs entirely.
- HealthKit data must not be stored in iCloud and must not be shared with third parties beyond what the runner agreed to.

## 4. Storage and access

- Heart rate lives in the separate `heart_rates` table with its own DB role. The public API role has **no** grant on it.
- Encrypt at rest (disk encryption + application-level encryption of `bpm` or the table via pgcrypto/KMS-managed keys).
- Medical dashboard access: role `medical` only, with MFA. Every read of a runner's heart rate or alert details writes an `audit_log` row (who, which runner, when).
- Support staff and developers do not get production access to heart-rate data; use anonymized samples for debugging.
- Backups containing heart rate follow the same retention policy.

**Retention (suggested defaults; set per event and legal advice):**
- Live window: Redis keeps only the latest value (TTL few minutes).
- Raw heart rate: delete or anonymize 30 days after the event (shorter if possible).
- Alerts and SOS records: keep per the organizer's incident-reporting requirements, separately from raw heart rate.
- Runner can request export and deletion at any time before the retention deadline.

## 5. Visibility matrix

| Data | Public | Runner | Organizer | Medical | Admin (ops) |
|---|---|---|---|---|---|
| Position / progress | Yes (consent: location) | Yes | Yes | Yes | Yes |
| Name/photo | If `public_profile` | Yes | Yes | Yes | Yes |
| Current heart rate | Only if `public_heart_rate` | Yes | No | Yes | No |
| Heart-rate history | No | Yes | No | Yes (event period, audited) | No |
| Heart-rate alerts | No | Notified (own) | Summary count only | Yes | No |
| SOS location/notes | No | Yes | Yes | Yes | Yes |

## 6. Heart-rate monitoring logic

Goals: help medical staff notice runners who may need attention. Avoid false alarms (alert fatigue) and avoid claiming medical diagnosis.

### 6.1 Data prep

- Discard impossible values (< 25 or > 240 bpm) as sensor errors.
- Smooth with a short median filter (e.g. 5 samples) for display and rule evaluation; keep raw values stored.
- Track **signal quality**: `ok`, `noisy`, `missing`. Sensor-off or gaps are a different state from low heart rate.
- Use `ts` from the sample, not receive time; late data is re-evaluated against its own time.

### 6.2 Personal baselines (optional, improves accuracy)

- Runner may provide age or max heart rate in the app (or leave it blank). If not provided, estimate `max_hr ≈ 220 - age` only as a rough guide, and mark it as an estimate.
- Learn a personal **race baseline** from the first 30-60 minutes: typical bpm range at their pace. Alerts compare against the baseline plus fixed absolute limits.

### 6.3 Alert rules (all thresholds configurable per event, reviewed by the medical lead)

| Alert | Condition (example defaults) | Severity |
|---|---|---|
| `hr_high_sustained` | Smoothed bpm > 95% of max_hr for ≥ 5 min | warn |
| `hr_very_high` | Smoothed bpm > 100% of max_hr or > 200 for ≥ 2 min | critical |
| `hr_high_at_rest` | bpm > 120 while speed < 0.3 m/s for ≥ 5 min | warn |
| `hr_low` | bpm < 40 for ≥ 2 min while the runner is moving | warn / critical if symptoms flagged (e.g. stopped) |
| `hr_sudden_change` | Change > 40 bpm within 30 s without matching speed change | info |
| `hr_missing` | No heart rate for ≥ 10 min while position data continues | info (sensor/watch problem) |
| `hr_missing_and_stationary` | No heart rate and no movement ≥ 10 min | warn (escalate to critical with SOS or off-route) |
| `combined_risk` | Stationary + high or low heart rate + off-aid-station | critical |

**Escalation:** `info` goes to the dashboard only; `warn` highlights the runner and notifies the medical lead; `critical` plays a sound, shows a pop-up with last position and nearest aid station, and sends push/SMS to on-duty responders.

**Hysteresis and cool-down:** an alert closes only after the value stays normal for a recovery period (e.g. 3 min); repeat alerts for the same runner and type are suppressed for N minutes unless severity increases.

### 6.4 Medical dashboard

- List of runners with a color status: green (normal), yellow (warn), red (critical), grey (no data).
- Per-runner panel: heart-rate chart (last 2 hours), pace and elevation overlay, last position on map, battery, last-seen time, nearest aid station and distance, quick actions (acknowledge, call runner via the app, dispatch, resolve).
- Filters: only alerts, only a section of the route, only runners without heart-rate data.
- Everything shown is read-audited.

### 6.5 What the system does **not** do

- Does not diagnose or predict medical conditions.
- Does not replace medical staff, course marshals, or emergency services.
- Does not auto-dispatch emergency services without human confirmation (unless the organizer explicitly designs that process).

### 6.6 Heat illness risk score (combined signals)

Heart rate alone **cannot** detect heat stroke. Exertional heat stroke is defined by a very high core body temperature plus central nervous system changes (confusion, irrational behavior, collapse), and neither is measured by a wrist heart-rate sensor. Heart rate can be high, normal, or low in an overheating runner, and it is naturally high during hard running.

The system therefore uses heart rate as **one input** to a combined "needs a check" score. It does not diagnose heat stroke. Diagnosis and core-temperature measurement happen at the medical tent.

#### Inputs

| Input | Source | Notes |
|---|---|---|
| Heat level (environment) | Weather worker + aid-station readings | Temperature, humidity, wind, cloud; adjusted for elevation along the route; WBGT (wet-bulb globe temperature) estimate or heat index |
| Pace collapse | Position data | Moving pace vs. the runner's own earlier baseline |
| Heart-rate drift | Heart-rate data | Rising bpm at the same or slower speed |
| Stationary time | Position data | Stopped outside an aid-station geofence |
| Possible confusion | Position data | Off route, wrong direction, erratic speed changes |
| Heart-rate missing | Heart-rate data | Sensor off or watch problem while the runner stops |
| Signal lost | Position data | No samples in a known risky section |
| Time exposed | Position + weather | Hours spent in high heat levels |
| Runner self-report | App | "I feel unwell" button |
| SOS | App | Always critical |
| Core temperature (optional) | Dedicated Bluetooth sensor, if the event provides one | Separate `core_temps` source and consent scope; thresholds set by the medical lead |
| Impact then immobility (optional) | Watch motion analysis | Sudden impact followed by no movement outside an aid station; runner is prompted to confirm they are OK |

**Not used as a core-temperature proxy:** smartwatch wrist or skin temperature. Consumer wrist-temperature features are designed for trends (for example overnight), not for detecting overheating during exercise.

#### Environmental heat level

The weather worker keeps a heat level per route segment and time (for example every 5 km of route, refreshed every 10-15 minutes):

1. Fetch temperature, humidity, wind, and cloud cover for the segment location.
2. Adjust temperature for elevation difference from the weather station or grid point (temperature falls with altitude, roughly 6.5 degrees C per 1,000 m).
3. Compute the heat index, or a WBGT estimate if solar radiation data is available. Manual WBGT readings from aid stations (entered in the admin dashboard) **override** model values for nearby segments.
4. Map the value to a level 0-3 (low, moderate, high, extreme) using thresholds from the guideline your medical lead selects (for example a recognized sports-medicine WBGT flag scheme). Do not hard-code numbers without their approval.

```sql
CREATE TABLE heat_levels (
  route_id      bigint,
  segment_start_m double precision,   -- route distance where segment starts
  segment_end_m   double precision,
  ts            timestamptz NOT NULL,
  temp_c        real, humidity_pct real, wind_ms real, cloud_pct real,
  wbgt_c        real,                 -- estimated or measured
  source        text NOT NULL,        -- model | aid_station_meter
  level         smallint NOT NULL,    -- 0..3
  PRIMARY KEY (route_id, segment_start_m, ts)
);
```

#### Scoring (example weights, all configurable per event)

Each rule adds points over a rolling window. Points are summed with the heat level points.

| Rule | Example condition | Points |
|---|---|---|
| Pace collapse | Moving pace > 30% slower than the runner's own 60-minute-earlier baseline, not explained by a steeper gradient (use grade-adjusted pace) | +2 |
| Heart-rate drift | Smoothed bpm up by 15 or more over 20 min while speed is equal or falling | +1 |
| Stationary | Stopped 5 min or more outside an aid-station geofence | +2 |
| Stationary and heart rate missing | No heart rate and no movement for 10 min | +2 |
| Possible confusion | Off route, going the wrong way, or erratic speed changes (see doc 05) | +2 |
| Signal lost in risky section | No samples beyond `T_silent` in a flagged section | +1 |
| Time in heat | Each 2 hours spent at heat level 2 or higher (max +2) | +1 each |
| Runner self-report | "I feel unwell" pressed | +3 |
| Heat level | Level 0 / 1 / 2 / 3 | 0 / +1 / +2 / +3 |
| SOS | | Critical immediately (not scored) |

Risk levels (example):

| Level | Score | Meaning | Action |
|---|---|---|---|
| Normal | below 4 | No concern | None |
| Watch | 4-5 | Mild concern | Highlight on the medical dashboard |
| Check | 6-7 | Needs a check | Notify the medical lead; send responder to the nearest aid station to look out for the runner; app prompts the runner |
| Urgent | 8 or more | Needs attention now | Sound and pop-up for on-duty medical staff with last position and the nearest aid station, plus push/SMS to responders |

Use hysteresis (a level drops only after the score stays lower for a recovery period) and cool-downs so the same runner does not generate repeated alerts.

```sql
CREATE TABLE risk_scores (
  runner_id   bigint NOT NULL,
  ts          timestamptz NOT NULL,
  score       real NOT NULL,
  level       text NOT NULL,         -- normal | watch | check | urgent
  components  jsonb NOT NULL,        -- which rules fired and their points
  PRIMARY KEY (runner_id, ts)
);
```

`risk_scores` is health-derived data. Apply the same access rules, encryption, retention, and audit logging as `heart_rates` (sections 4 and 5).

#### Apple Watch sensor assessment for heat risk

The Apple Watch has no validated heat stroke indicator and does not measure core body temperature. Use only the signals below, and treat all of them as weak inputs to the combined score.

| Signal | Use for heat risk? | Notes |
|---|---|---|
| Heart rate (workout session) | Weak, as one input | High during hard running anyway; wrist optical sensors get noisy with sweat and motion |
| Motion (accelerometer, gyroscope) | Yes, for behavior | Detect stopping, stumbling, erratic or collapsing movement; run Core Motion analysis in the watch workout app |
| Fall detection | Possible, with limits | Apple's Core Motion fall-detection API needs a special entitlement from Apple (check eligibility); designed for general falls, so a runner who slumps or sits down may not trigger it |
| Wrist temperature | No | Designed for overnight trend tracking and, as far as is known, not exposed as a live reading during workouts; skin temperature also tracks core temperature poorly during exercise |
| HRV, blood oxygen | No | Sampled sporadically or unreliable during running motion |
| Heart rate recovery | After the fact only | Calculated after the workout ends, so it cannot help during the race |

Design decisions that follow from this:

1. Keep the combined risk score (heart rate, pace collapse, stationary time, off-route behavior, heat level, self-report) as the main approach.
2. Add a **motion-based "impact then immobility" input** from the watch (a sudden impact followed by no movement for N seconds, outside an aid station). Suggested weight: +3, and the app prompts the runner to confirm they are OK before the alert escalates (a short on-watch and on-phone prompt with haptic feedback).
3. Do not use wrist temperature, HRV, or blood oxygen in the score.
4. Support an optional external core-temperature source (below).

#### Optional source: external core-temperature sensor

A dedicated core-temperature sensor is the only way to add a real temperature signal. An example is the CORE sensor, a wearable for endurance athletes that tracks core and skin temperature; the vendor recommends using it with a heart-rate monitor for best accuracy. Some third-party apps can store its data in Apple Health as body temperature. For this platform it would be read over Bluetooth by the runner app.

| Topic | Guidance |
|---|---|
| Who uses it | Not every runner (cost); consider high-risk groups, volunteers, elites, or a pilot group |
| Integration | Check the vendor's integration options, licensing, and protocol; read via Core Bluetooth (iOS) or Android BLE; keep the integration behind a source adapter so other sensors can be added |
| Consent | New consent scope `core_temp` (collection) with the same rules as `heart_rate`; never shown publicly |
| Storage | Separate `core_temps` table (below), same role restrictions, encryption, retention, and audit logging as `heart_rates` |
| Interpretation | High core temperatures are normal in hard racing (values above 38 degrees C can occur in a half marathon, and up to about 40 degrees C in vigorous exercise in heat), so a number alone is not heat stroke. Symptoms such as confusion matter as much. Thresholds and responses are set by the medical lead. |
| Scoring | Core temperature above the medical lead's "watch" and "urgent" levels adds points, or can escalate directly to "check" or "urgent" as they decide; combine with behavior signals (confusion, stopping) |
| Data quality | Mark sensor status (connected, warming up, lost); a lost sensor is "no data", not "normal" |

```sql
CREATE TABLE core_temps (
  runner_id   bigint NOT NULL,
  ts          timestamptz NOT NULL,
  core_c      real,                 -- estimated core temperature
  skin_c      real,
  strain_index real,                -- vendor heat strain metric, if provided
  source      text NOT NULL,        -- e.g. core_sensor
  quality     text,
  received_at timestamptz NOT NULL,
  PRIMARY KEY (runner_id, ts, source)
);
SELECT create_hypertable('core_temps', 'ts');
```

Add `core_temp` to the consent scopes table (section 2), to the visibility matrix (visible to the runner and medical staff only), and to the batch upload payload as an optional `coreTemp` array alongside `heartRate`.

#### Optional experimental input: heart-rate-based core-temperature estimate

Research and military systems estimate core temperature from heart-rate time series (for example, an MIT Lincoln Laboratory smartwatch system that alerts trainees to their heat strain risk). Caveats for a race setting:

- Developed and tested in military training with chest or arm heart-rate sensors
- Individual error can be large
- Wrist optical heart rate is noisier
- Not shown to work on Apple Watch heart rate in races

If you want to try it: run it server-side in shadow mode (compute and store, do not alert), compare against real outcomes and any core-temperature sensor data, and add it to the score at a low weight only after the medical lead approves.

#### Wording and actions

- Alerts say **"needs a check"**, never "heat stroke detected" or any diagnosis.
- The dashboard shows *why* the score is high (the `components` list), so staff can judge it quickly.
- Each alert shows the runner's last position, speed trend, heart-rate chart, current heat level for that segment, nearest aid station, and an estimated time for a responder to reach them.
- Staff actions: acknowledge, call the runner via the app, dispatch, record outcome (false alarm, checked and fine, treated, withdrawn).

#### Runner-facing features

- **Heat advisory banner** in the app when the heat level for the runner's section is 2 or higher (hydration, pacing, and cooling reminders).
- **"I feel unwell" button** next to SOS, which adds points and notifies staff without being an emergency call.
- **Symptom guide:** short, plain-language list of heat illness warning signs (confusion, stumbling, headache, nausea, cessation of sweating, collapse) and clear advice to stop and ask for help.
- Messages from staff ("Please stop and check in at the next aid station") need wording approved by the medical lead.
- Runners with heart-rate consent withdrawn or no watch still receive scoring from position, heat, and self-report inputs.

#### Limitations

- **False positives:** hills, walking sections, rest stops, and sensor dropouts can add points. Grade-adjusted pace and aid-station geofences reduce this, but not completely.
- **False negatives:** a runner can collapse suddenly with no earlier warning pattern, and data can be missing exactly when it matters.
- **Environment model error:** forecast or grid weather can differ from conditions on the trail; aid-station meter readings correct it near stations only.
- **Not validated as a medical tool.** Treat it as an awareness aid.

#### Validation and tuning

1. Replay recorded data from past races (positions, pace, weather) and review alert counts per 100 runners with the medical lead.
2. Tune weights and thresholds so alert volume is manageable for the medical team on a hot day.
3. Run a staged exercise before the event (volunteer "runners" simulating pace collapse, stops, and off-route behavior) to test the dashboard and response process.
4. Record outcomes (the staff action "false alarm / checked / treated") to review score quality after each event.

## 7. Runner-facing behavior

- Runner sees their own heart rate in the app and can pause heart-rate sharing temporarily (the dashboard then shows "paused by runner").
- If a critical alert is raised, the app may show a message from medical staff ("Please stop and check in") and request acknowledgement. Use wording approved by the medical lead.
- After the race: option to download or delete the runner's data.

## 8. Platform compliance checklist

**Apple (iOS/watchOS)**
- HealthKit entitlement and clear `NSHealthShareUsageDescription` / `NSHealthUpdateUsageDescription` strings
- Privacy policy URL that covers health data; accurate App Privacy "nutrition label"
- No advertising or unrelated data mining with HealthKit data; no storing HealthKit data in iCloud
- Background location justified in review notes (live safety tracking for race participants)

**Google (Android/Wear OS)**
- Play Console declarations: background location (if used), foreground service type location, Health Connect data types
- In-app prominent disclosure for location and health data before the permission prompt
- Health Connect permission rationale activity and privacy policy link
- Data Safety form completed accurately (collection, sharing, encryption, deletion)

**Both**
- Privacy policy covering what is collected, purposes, retention, sharing, rights, contact
- Data deletion request path in-app and on the web

## 9. Incident and breach plan

1. Keep an incident response contact and runbook for the event.
2. If heart-rate data is exposed, follow notification duties under applicable law; document scope, time, cause, and fix.
3. Log review: audit log tells who accessed what.
4. Rotate keys and revoke tokens if credentials leak.

## 10. Implementation checklist

- [ ] Consent screens per scope, versioned and stored
- [ ] HealthKit / Health Connect prompts only after opt-in
- [ ] Separate DB role and table for `heart_rates`
- [ ] Column/disk encryption configured
- [ ] Public API verified to never serve heart rate unless `public_heart_rate` is active
- [ ] Medical dashboard with MFA and audit logging
- [ ] Alert rules reviewed by medical lead; thresholds in config
- [ ] Heat-level pipeline (weather by route segment, elevation adjustment, aid-station WBGT override)
- [ ] Risk score rules and thresholds reviewed and approved by the medical lead
- [ ] `risk_scores` and `heat_levels` tables created; `risk_scores` under the same access controls as heart rate
- [ ] "I feel unwell" button and heat advisory banner in both apps
- [ ] Alert wording ("needs a check") approved; response protocol written
- [ ] Replay and staged-exercise testing completed before the event
- [ ] Watch motion input ("impact then immobility") with on-watch/phone confirmation prompt
- [ ] Apple fall-detection API entitlement eligibility checked (if used)
- [ ] Optional `core_temp` consent scope, `core_temps` table, and sensor adapter (if the event provides sensors)
- [ ] Heart-rate-based core-temperature estimate, if used, runs in shadow mode until validated
- [ ] Revocation flow tested (collection stops, data deleted per policy)
- [ ] Logs, analytics, and crash reports scrubbed of health data
- [ ] Retention jobs scheduled and tested
- [ ] Privacy policy and store disclosures published
