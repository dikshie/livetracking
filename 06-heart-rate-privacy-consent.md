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
- [ ] Revocation flow tested (collection stops, data deleted per policy)
- [ ] Logs, analytics, and crash reports scrubbed of health data
- [ ] Retention jobs scheduled and tested
- [ ] Privacy policy and store disclosures published
