# 08. Appendix: Glossary of Terms and Abbreviations

Plain-language explanations of the technical terms used in this documentation set. Terms are grouped by topic. Use your browser's search (Ctrl/Cmd + F) to find a term quickly.

**Contents**

1. [Race and event terms](#1-race-and-event-terms)
2. [Location, maps, and geography](#2-location-maps-and-geography)
3. [Data processing and algorithms](#3-data-processing-and-algorithms)
4. [Apple platform (iPhone and Apple Watch)](#4-apple-platform-iphone-and-apple-watch)
5. [Android platform (phone and Wear OS)](#5-android-platform-phone-and-wear-os)
6. [Backend, servers, and databases](#6-backend-servers-and-databases)
7. [Networking, web, and APIs](#7-networking-web-and-apis)
8. [Web app, design, and map software](#8-web-app-design-and-map-software)
9. [Health, heart rate, and heat illness](#9-health-heart-rate-and-heat-illness)
10. [Privacy, security, and compliance](#10-privacy-security-and-compliance)
11. [Testing and operations](#11-testing-and-operations)
12. [General abbreviations and units](#12-general-abbreviations-and-units)

---

## 1. Race and event terms

| Term | Meaning |
|---|---|
| **Ultra marathon (ultra)** | A foot race longer than a standard marathon (42.2 km), often on trails and over mountains. |
| **Bib / bib number** | The numbered cloth a runner wears. Used here as the runner's public ID. |
| **Checkpoint** | A defined point on the route where passing is recorded (start, aid station, cut-off point, finish). |
| **Aid station** | A staffed stop with water, food, and sometimes medical help. |
| **Cut-off** | The latest time a runner may reach a checkpoint to stay in the race. |
| **DNF** | "Did Not Finish." A runner who withdrew or was stopped before the finish. |
| **Split** | The time taken for one section (for example, between two checkpoints). |
| **Pace** | Time per distance, usually minutes per kilometre (min/km). The inverse of speed. |
| **Moving pace** | Pace calculated only while the runner is moving (stops excluded). |
| **Grade-adjusted pace** | Pace corrected for slope, so uphill and downhill sections can be compared fairly. |
| **ETA** | "Estimated Time of Arrival" at a checkpoint or the finish. |
| **Gun time / chip time** | Gun time starts at the start signal; chip time starts when the runner crosses the start line. |
| **Wave start** | Runners starting in groups at different times. |
| **Loop route / out-and-back route** | A route that returns to its start, or that goes to a turnaround point and comes back along the same path. Both make "where is the runner on the route?" harder to compute. |
| **Elevation gain / loss** | Total metres climbed / descended along a route. |
| **Gradient (grade)** | Steepness as a percentage. A 10% gradient rises 10 m for every 100 m travelled. |
| **Marshal** | A volunteer or official along the course. |
| **Medical lead** | The person responsible for the event's medical team, who should approve alert thresholds and response procedures. |

## 2. Location, maps, and geography

| Term | Meaning |
|---|---|
| **GPS** | "Global Positioning System." The US satellite positioning system; the term is also used loosely for any satellite positioning. |
| **GNSS** | "Global Navigation Satellite System." The umbrella term for GPS, GLONASS, Galileo, BeiDou and others. Phones use several together. |
| **Fix** | One position calculation from satellite signals (latitude, longitude, accuracy, time). |
| **Latitude / longitude (lat/lon)** | Coordinates on Earth: north-south and east-west angles in degrees. |
| **Altitude / elevation** | Height above sea level. GPS altitude is less accurate than horizontal position. |
| **hAcc / vAcc** | Horizontal / vertical accuracy: the estimated error radius in metres. Smaller is better. |
| **Speed / course (bearing)** | Speed in metres per second; direction of travel in degrees from north. |
| **GPX** | "GPS Exchange Format." A common file format (XML-based) for GPS tracks, routes, and waypoints. Race organizers use GPX files to define the course. |
| **GeoJSON** | A JSON-based format for geographic shapes (points, lines, areas). Web maps use it to draw routes. |
| **Polyline** | A line made of connected points, used to draw a route on a map. |
| **LineString / LineStringZ** | A line geometry in a database. The "Z" version also stores elevation for each point. |
| **Geofence** | A virtual circle or area around a location. Entering it triggers an action (for example, recording a checkpoint pass). |
| **Route snapping** | Matching a noisy GPS position to the nearest point on the official route to get a clean "distance along route." |
| **Off-route distance** | How far the runner is from the official route line. |
| **WGS84 / SRID 4326** | The standard coordinate system used by GPS and most databases for lat/lon data. |
| **Web Mercator** | The flat map projection used by most web maps (Google Maps, OpenStreetMap, MapLibre). |
| **UTM** | "Universal Transverse Mercator." A metric (metres-based) coordinate system, useful for accurate distance calculations over small areas. |
| **Basemap** | The background map (roads, terrain, labels) under your routes and markers. |
| **Overlay / layer** | Extra graphics drawn on top of the basemap (weather, routes, markers). |
| **Tiles (raster / vector)** | Maps are delivered in small square pieces. Raster tiles are images; vector tiles are drawing data styled in the browser (smaller, work well in light and dark themes). |
| **XYZ tiles** | A common addressing scheme for tiles (zoom level, column X, row Y). Weather tile providers usually use it. |
| **Zoom level / bounds** | How close the map is; the geographic limits the user can pan to. |
| **Bounding box (bbox)** | The smallest rectangle that contains an area, used to limit which map data you download or host. |
| **Style JSON** | A file describing how a vector map looks (colours, fonts, which features to show). |
| **OpenStreetMap (OSM)** | A free, community-built map database. |
| **DEM** | "Digital Elevation Model." A grid of ground elevation values. SRTM and Copernicus are well-known open DEM datasets. |
| **Hillshade / contour lines** | Map shading that shows relief from a DEM; lines joining points of equal elevation. |
| **Attribution** | Required credit text or logo shown on a map for its data or tile provider. |
| **Dead zone** | A section of the route without mobile signal. |
| **Mock location** | A fake position injected by a developer tool or app. Phones can flag it, which helps detect cheating. |

## 3. Data processing and algorithms

| Term | Meaning |
|---|---|
| **Sample** | One recorded measurement (a position fix or a heart-rate reading) with its timestamp. |
| **Timestamp / `ts`** | The time the measurement was taken, as opposed to the time it was received by the server. |
| **UTC / ISO 8601 / "Z"** | UTC is the world reference time. ISO 8601 is a standard text format for timestamps, like `2026-10-17T05:40:58Z`; the trailing "Z" means UTC. |
| **Monotonic clock** | A clock that only counts forward (time since the phone started), unaffected by changing the phone's date or time. |
| **Buffer / offline-first** | Storing data on the device first and uploading later, so nothing is lost when there is no signal. |
| **Batch** | Many samples sent together in one request. |
| **Back-fill** | Uploading older data after a gap, so history is completed. |
| **Out-of-order data** | Data arriving in a different order than it was measured. The system sorts by measurement time. |
| **De-duplication** | Detecting and dropping repeated copies of the same sample. |
| **Idempotency / idempotency key** | Sending the same request twice has the same effect as sending it once. An idempotency key is a unique ID that lets the server recognize repeats. |
| **Sequence number (`seq`)** | A counter that increases with every sample from a device, used for ordering and duplicate detection. |
| **Quality filter** | Rules that reject or flag bad samples (very inaccurate, impossible speed, suspicious jumps). |
| **Kalman filter / moving average / median filter** | Smoothing methods that reduce noise in measurements. A Kalman filter estimates the true value from noisy readings over time. |
| **Hysteresis** | Using different thresholds for entering and leaving a state (for example, off-route at 60 m, back on-route at 30 m) so statuses do not flicker. |
| **Cool-down** | A pause during which the same alert is not repeated. |
| **State machine** | A model where something (a runner) is always in exactly one state (Tracking, Finished, DNF...) and moves between states by defined events. |
| **Rule engine** | Software that evaluates configurable rules (for example, alert conditions). |
| **Rolling window** | Calculations over the most recent period (for example, the last 2 minutes). |
| **Monotonic progress** | Assuming a runner's distance along the route does not go backwards (apart from small noise). |
| **Interpolation** | Estimating a value between two known points, such as the exact time a runner crossed a checkpoint between two samples. |
| **Weighted score** | A total made by adding points for each rule that fires, used for the heat risk score. |
| **Server-driven (remote) config** | Settings (intervals, thresholds) sent from the server so they can change without updating the app. |
| **Exponential backoff and jitter** | When a request fails, wait progressively longer before retrying, with some random variation, to avoid overloading the server. |
| **Shadow mode** | Running a new algorithm and recording its results without using them to trigger actions, so it can be evaluated safely. |

## 4. Apple platform (iPhone and Apple Watch)

| Term | Meaning |
|---|---|
| **iOS / watchOS** | Apple's operating systems for iPhone and Apple Watch. |
| **Swift / SwiftUI** | Apple's programming language and its modern user interface framework. |
| **Xcode** | Apple's development tool for building iOS and watchOS apps. |
| **Core Location** | The iOS framework that provides GPS position, speed, and accuracy. |
| **HealthKit** | Apple's framework for reading and writing health data such as heart rate and workouts. |
| **HKWorkoutSession / HKLiveWorkoutBuilder** | HealthKit classes. The session represents a running workout; the builder collects live data such as heart rate during it. |
| **Mirrored workout session** | A watchOS 10 / iOS 17 feature where the watch's workout session is mirrored on the iPhone so live data (heart rate) streams to the phone. |
| **WatchConnectivity** | Framework for sending messages and data between iPhone and Apple Watch apps. Used as a fallback. |
| **Core Motion** | Framework for accelerometer and gyroscope data and certain motion events, including fall detection (with special permission). |
| **Core Bluetooth** | Framework for talking to Bluetooth devices such as chest straps or sensors. |
| **Background mode (Location updates)** | An app setting that lets the app keep receiving location while the screen is off. |
| **Info.plist** | The app's settings file. It holds the text shown when asking for permissions (usage descriptions). |
| **Capability / entitlement** | Permissions built into the app's signing that unlock special features (HealthKit, Push Notifications, special APIs). Some require approval from Apple. |
| **"When In Use" / "Always" location** | Two location permission levels. "Always" allows tracking when the app is in the background. |
| **APNs** | "Apple Push Notification service." Sends push notifications to iPhones. |
| **App Attest / DeviceCheck** | Apple services that let the server verify a request comes from a genuine, unmodified copy of your app on a real Apple device. |
| **Keychain** | iOS secure storage for secrets such as login tokens. |
| **URLSession / background session** | Apple's networking API; a background session can finish uploads even if the app is suspended. |
| **BackgroundTasks / `BGProcessingTask`** | Framework to request background time from the system, for example to flush the upload queue. |
| **`NWPathMonitor`** | Detects network connectivity changes. |
| **SQLite / GRDB / SwiftData** | SQLite is a small embedded database; GRDB is a Swift library for it; SwiftData is Apple's data persistence framework. Used for the offline buffer. |
| **Low Power Mode** | iOS battery-saving mode that reduces background activity. |
| **App Store review notes** | Explanations given to Apple's reviewers (for example, why background location is needed). |

## 5. Android platform (phone and Wear OS)

| Term | Meaning |
|---|---|
| **Android / Wear OS** | Google's phone operating system and its smartwatch version. |
| **Kotlin / Jetpack Compose** | Google's preferred programming language for Android and its modern UI toolkit. |
| **Fused Location Provider** | A Google Play services API that combines GPS, Wi-Fi, and other sources to give the best location efficiently. |
| **Foreground service** | An Android background task that shows a persistent notification, which lets it keep running reliably (used for tracking). |
| **AndroidManifest** | The app's settings file, listing permissions and services. |
| **Doze / battery optimization** | Android power-saving behavior that can pause apps. Some phone brands (Xiaomi, Oppo, Vivo, Huawei, Samsung) add stricter limits. |
| **OEM** | "Original Equipment Manufacturer": the phone brand (Samsung, Xiaomi...), whose custom Android versions may kill background apps. |
| **Room** | Android's database library built on SQLite. Used for the offline buffer. |
| **WorkManager** | Android's scheduler for reliable background jobs (for example, uploading data when the network returns). |
| **Health Services (`ExerciseClient`)** | Wear OS API to run an exercise session and read live sensor data such as heart rate. |
| **Health Connect** | An Android hub where fitness apps and watches store health data (such as `HeartRateRecord`). Often delayed rather than live. |
| **Wearable Data Layer (`MessageClient`, `DataClient`)** | Wear OS APIs for sending data between watch and phone: messages are live; data items are queued. |
| **Play Integrity API** | Google service that lets the server check that the app is genuine and running on a real, trustworthy device. |
| **FCM** | "Firebase Cloud Messaging." Sends push notifications to Android devices. |
| **Android Keystore** | Secure hardware-backed storage for keys and secrets. |
| **Google Play Console / Data Safety form** | Where apps are published on Google Play, and the form declaring what data the app collects and shares. |
| **`minSdk` / `targetSdk`** | The oldest Android version the app supports / the version it is built and tested against. |

## 6. Backend, servers, and databases

| Term | Meaning |
|---|---|
| **Backend** | The server-side software that receives data, processes it, and serves the apps and website. |
| **Ingestion** | The step where incoming data is received, validated, and queued. |
| **Worker** | A background program that processes tasks from a queue (here: route snapping, checkpoints, alerts). |
| **Queue / stream** | A waiting line for data between services. **Redis Streams** and **NATS JetStream** are queue technologies. |
| **Redis** | A very fast in-memory data store, used for latest positions, caching, and message passing. |
| **Pub/sub** | "Publish/subscribe." One part publishes messages, other parts subscribed to a channel receive them. |
| **Fan-out** | Sending one update to many receivers (all spectators). |
| **PostgreSQL** | A widely used open-source relational database. |
| **PostGIS** | A PostgreSQL extension that adds geographic data types and functions. |
| **TimescaleDB / hypertable** | A PostgreSQL extension for time-series data; a hypertable is a table automatically split into time chunks for speed. |
| **Schema** | The structure of a database (tables, columns, types). |
| **SQL** | The language used to query and define relational databases. |
| **Spatial index (GiST)** | A database index that makes location searches fast. |
| **Retention policy** | A rule that deletes or anonymizes data after a set period. |
| **Compression (Timescale)** | Shrinking older data chunks to save storage. |
| **Column-level encryption / pgcrypto / KMS** | Encrypting specific database fields. pgcrypto is a PostgreSQL encryption extension; KMS ("Key Management Service") stores and controls encryption keys. |
| **Go / Node.js / Fastify** | Programming platforms suggested for the backend. Fastify is a fast Node.js web framework. |
| **VPS** | "Virtual Private Server." A rented virtual machine. |
| **vCPU** | A virtual processor core allocated to a server. |
| **Object storage (S3 / R2)** | File storage service for large files (maps, GPX, images). S3 is Amazon's; R2 is Cloudflare's. |
| **CDN** | "Content Delivery Network." A network of servers around the world that cache and deliver content quickly. Cloudflare and CloudFront are CDNs. |
| **Cache / TTL** | A stored copy of a response to avoid recomputing it; TTL ("time to live") is how long the copy stays valid. |
| **Snapshot / delta** | A snapshot is the full current state; a delta is only what changed since. |
| **Latency (p50 / p95)** | Delay between an event and its appearance. p50 is the typical (median) value; p95 means 95% of values are faster than this. |
| **Scale-out** | Handling more load by adding more servers. |
| **Kill switch** | A control that instantly turns a feature off (for example, public tracking). |
| **Environment (dev / staging / prod)** | Separate copies of the system for development, testing, and live use. |
| **Observability / metrics / logs** | Tools and data that show how the system is behaving. |

## 7. Networking, web, and APIs

| Term | Meaning |
|---|---|
| **API** | "Application Programming Interface." The set of requests one program can make to another. |
| **REST / endpoint** | A common style of web API; an endpoint is one specific URL path of the API (for example, `/v1/telemetry/batch`). |
| **HTTP / HTTPS / TLS** | The web protocol; HTTPS is HTTP encrypted using TLS. |
| **JSON** | A simple text format for structured data used by most APIs. |
| **Payload** | The data carried in a request or message. |
| **Header (`Authorization`, `Idempotency-Key`)** | Extra information sent with an HTTP request. |
| **HTTP status codes** | `200` = OK; `401` = not authenticated (token expired or invalid); `429` = too many requests; `4xx` = client error; `5xx` = server error. |
| **Token (access / refresh)** | A credential proving who the user is. Access tokens are short-lived; refresh tokens get new access tokens. |
| **JWT** | "JSON Web Token." A signed token that carries identity information. |
| **Authentication / authorization** | Proving who you are / deciding what you are allowed to do. |
| **Webhook** | A URL on your server that another service calls when something happens. |
| **Rate limiting** | Restricting how many requests a client can make in a time period. |
| **SSE (Server-Sent Events)** | A one-way live stream from server to browser over HTTP. |
| **WebSocket** | A two-way live connection between browser and server. |
| **Polling** | The client repeatedly asking the server for updates at intervals. |
| **CORS** | "Cross-Origin Resource Sharing." Browser rules about which websites may request data from your server. |
| **HTTP range request** | Asking for only part of a file. PMTiles uses this to read small pieces of one large map file. |
| **Proxy (server proxy)** | A server in the middle that forwards requests, used here to hide API keys and cache weather tiles. |
| **Push notification** | A message delivered to the phone by Apple (APNs) or Google (FCM), even when the app is closed. |
| **SMS gateway** | A service that sends text messages programmatically. |

## 8. Web app, design, and map software

| Term | Meaning |
|---|---|
| **UI** | "User Interface." What the user sees and taps: screens, buttons, maps. |
| **UX** | "User Experience." How easy, clear, and pleasant the product is to use overall, including flows, wording, and speed. |
| **Mobile-first / responsive** | Designing for phones first and adapting to larger screens. |
| **PWA** | "Progressive Web App." A website that can be installed on a phone and work like an app, with some offline support. |
| **Service Worker** | A background script in the browser that caches files and enables offline use and installability. |
| **Frontend** | The part users see and interact with (website or app). |
| **TypeScript / React / Svelte / Vite** | A typed version of JavaScript; two popular UI frameworks; and a fast build tool. |
| **DOM / DOM markers** | The browser's page structure; DOM markers are map pins built from ordinary HTML elements, which become slow in large numbers. |
| **WebGL** | Browser technology for GPU-accelerated graphics, used to draw thousands of map points smoothly. |
| **Code splitting** | Loading only the code needed at first so the page starts faster. |
| **i18n** | "Internationalization." Supporting multiple languages (here, Indonesian and English). |
| **Dark mode / theme** | A dark colour scheme vs. a light one. |
| **Bottom sheet** | A panel that slides up from the bottom of a mobile screen. |
| **Clustering** | Grouping nearby map markers into one numbered marker when zoomed out. |
| **Adapter (map adapter)** | A small layer of code that hides which map library is used, so you can switch libraries without rewriting the app. |
| **MapLibre GL JS** | A free, open-source library for drawing interactive vector maps in the browser. |
| **Leaflet** | A lightweight open-source map library, mostly for raster tiles. |
| **Google Maps JavaScript API** | Google's map library for websites (paid, billed per map load). |
| **deck.gl / MarkerClusterer** | A WebGL layer library for large point sets; a Google Maps library for clustering markers. |
| **Protomaps / PMTiles** | An open system that stores a map as a single file (`.pmtiles`) you can host on cheap storage, with no map server needed. |
| **MapTiler / Mapbox / Stadia Maps / Thunderforest / Geoapify** | Hosted map tile providers with free and paid plans. |
| **RainViewer / OpenWeatherMap** | Services providing weather radar tiles and weather data. |
| **OpenTopoMap** | A ready-made topographic map style based on OSM data. |
| **Mermaid** | A text-based diagram syntax used for diagrams in these documents. |
| **Markdown (`.md`)** | The simple text format used for these documents. |
| **CSV** | "Comma-Separated Values." A plain spreadsheet-style text file, used for importing runner lists. |
| **QR code / NFC / RFID** | Ways to scan or tap a tag to identify a runner at a checkpoint (camera code, short-range wireless tap, radio tag). |
| **QRIS** | Indonesia's standardized QR code for payments, used for donations. |

## 9. Health, heart rate, and heat illness

| Term | Meaning |
|---|---|
| **Heart rate / bpm** | Heartbeats per minute. |
| **Max HR (maximum heart rate)** | The highest heart rate a person can reach. Often estimated as 220 minus age, which is only a rough guide. |
| **HRV** | "Heart Rate Variability." The variation in time between heartbeats; sampled sporadically by watches and unreliable during running. |
| **Heart rate drift** | Heart rate rising over time at the same effort or speed, often due to heat, fatigue, or dehydration. |
| **Heart rate recovery** | How quickly heart rate drops after exercise stops. |
| **PPG (optical heart rate)** | "Photoplethysmography." The green-light sensor on a watch's underside that detects pulse from blood-volume changes. It can be disturbed by sweat, motion, or a loose fit. |
| **BLE** | "Bluetooth Low Energy." Short-range low-power wireless used by chest straps and sensors. |
| **Chest strap** | A heart-rate sensor worn around the chest; more accurate than wrist sensors during running. |
| **SpO2 / blood oxygen** | Oxygen level in blood; unreliable when measured on a moving wrist. |
| **Core body temperature** | The temperature deep inside the body. Heat stroke is defined by a very high value (around 40 degrees C / 104 degrees F or above) together with brain-function changes. |
| **Skin / wrist temperature** | Temperature at the skin surface. It does not track core temperature well during exercise. |
| **Exertional heat stroke** | A life-threatening overheating illness caused by hard exercise, with high core temperature and symptoms such as confusion or collapse. |
| **Heat illness / heat exhaustion** | The range of heat-related conditions; heat exhaustion is less severe than heat stroke but can progress. |
| **Heat index** | A "feels-like" temperature calculated from air temperature and humidity. |
| **WBGT** | "Wet-Bulb Globe Temperature." A heat-stress measure that combines air temperature, humidity, wind, and solar radiation; widely used for event heat safety. |
| **Heat strain index (HSI)** | A vendor-defined measure combining core and skin temperature to express how hard the body is working to cool itself. |
| **Lapse rate** | How much air temperature falls with altitude (roughly 6.5 degrees C per 1,000 m), used to adjust weather data for mountain trails. |
| **Baseline** | A runner's own normal values (pace, heart rate) used as a comparison for detecting change. |
| **Sustained** | A condition that lasts for a minimum time (for example, 5 minutes) before an alert fires. |
| **Sensor quality** | A label (ok, noisy, missing) showing how trustworthy a reading is. "Missing" is different from "low". |
| **False positive / false negative** | An alert when nothing is wrong / no alert when something is wrong. |
| **Alert fatigue** | Staff ignoring alerts because there are too many. |
| **Severity (info / warn / critical)** | Alert importance levels that decide who is notified and how. |
| **Escalation** | Raising an alert to a higher level or to more people. |
| **SOS** | An emergency help request sent from the app with the runner's position. |
| **Impact then immobility** | A motion pattern: a sudden jolt (fall or collapse) followed by no movement. |
| **Risk score levels (Normal / Watch / Check / Urgent)** | The four levels of the combined heat risk score described in doc 06. |

## 10. Privacy, security, and compliance

| Term | Meaning |
|---|---|
| **Consent (scope)** | The runner's explicit permission for a specific purpose (location, heart rate, public display). Each purpose is a separate scope. |
| **Data minimization** | Collecting only the data you need, for only as long as you need it. |
| **Least privilege** | Giving each user or system only the access it needs. |
| **PII** | "Personally Identifiable Information." Data that can identify a person (name, bib linked to a person, location history). |
| **Health data** | Data about a person's health, such as heart rate. It has stricter legal and store requirements. |
| **GDPR** | The European Union's data protection law. |
| **UU PDP** | Indonesia's Personal Data Protection Law (Undang-Undang Pelindungan Data Pribadi). |
| **Anonymization / pseudonymization** | Removing or replacing identifying details in data so it can no longer (or not easily) be linked to a person. |
| **Retention** | How long data is kept before deletion or anonymization. |
| **Encryption at rest / in transit** | Encoding stored data / data travelling over the network so others cannot read it. |
| **RBAC** | "Role-Based Access Control." Access determined by a user's role (public, runner, organizer, medical, admin). |
| **MFA / SSO** | "Multi-Factor Authentication" (a second proof of identity, such as a code) / "Single Sign-On" (using one organizational login). |
| **Audit log** | A record of who accessed or changed what and when, especially for health data. |
| **App attestation / integrity check** | Proof that requests come from a genuine app on a genuine device (App Attest, Play Integrity). |
| **Privacy policy / App Privacy label / Data Safety form** | The legal document describing data use; Apple's and Google's store disclosures about data collection. |
| **Breach notification** | The duty to inform affected people and authorities if data is exposed. |
| **Certificate pinning** | The app only trusts your server's specific certificate, which makes interception harder. |

## 11. Testing and operations

| Term | Meaning |
|---|---|
| **Simulator / replay** | A tool that replays recorded or generated tracks as if runners were running, for testing without a real race. |
| **Load test** | Testing the system with many simulated users or runners to see if it keeps up. |
| **Fuzz test** | Testing with random, extreme, or malformed inputs to find weaknesses. |
| **Field test** | Testing on a real trail with real phones and conditions. |
| **Staged exercise** | A rehearsal where volunteers act out scenarios (for example, a runner who stops) to test alerts and response. |
| **Contract test** | An automated check that an API follows its agreed rules, for example that public endpoints never return heart rate. |
| **Back-pressure / queue depth** | Signs that processing is falling behind the incoming data. |
| **Runbook** | A written step-by-step guide for handling incidents. |
| **MVP** | "Minimum Viable Product." The smallest version that is useful to real users. |

## 12. General abbreviations and units

| Term | Meaning |
|---|---|
| **OS** | Operating system. |
| **SDK** | "Software Development Kit." Tools and libraries for building software for a platform. |
| **CLI** | "Command-Line Interface." A tool used by typing commands. |
| **DB** | Database. |
| **UUID** | A long random unique identifier, for example for a batch of uploads. |
| **KB / MB / GB** | Kilobyte / megabyte / gigabyte (data sizes). |
| **ms / s** | Millisecond (one thousandth of a second) / second. |
| **m / km** | Metre / kilometre. |
| **m/s** | Metres per second. Multiply by 3.6 for km/h (for example, 3 m/s is 10.8 km/h). |
| **min/km** | Minutes per kilometre (pace). |
| **°C / °F** | Degrees Celsius / Fahrenheit. |
| **PWA, UI, UX, API, GPX, GNSS, WBGT, DNF, ETA** | Each is explained in its own section above. |
