# 07. Web Viewer and Admin

The spectator site reproduces the main features of public tracking pages such as Strackking. The admin and medical dashboards are separate, authenticated apps.

## 1. Public web viewer (PWA)

**Stack:** TypeScript, Vite, a light framework (React or Svelte), **MapLibre GL JS**, Service Worker for installability and offline shell.

**URL pattern:** `/live/{eventSlug}` (optionally `/live/{eventId}/{slug}`), plus `?bib=123` to deep-link a runner.

### 1.1 Features

| Feature | Implementation |
|---|---|
| Live map with runner markers | Load snapshot from CDN, then apply SSE deltas; animate markers between updates; cluster when zoomed out |
| Route display and filter | Routes as GeoJSON lines with per-route colors; filter chips by route/category |
| Runner search | Search by bib or name; click centers the map and opens the runner card |
| Runner card | Bib, name (if public), route, distance covered, pace, elapsed time, last checkpoint, ETA to next, "last seen" age, elevation profile with the runner's position |
| Checkpoints | Markers with name, cutoff time, runner list and times; checkpoint table view |
| Leaderboard | Live ordering by route progress + checkpoint results |
| Map modes | Standard, satellite, terrain; light and dark theme (separate basemap style per theme) |
| Weather layers | Rain radar, clouds, wind, temperature overlays + current-weather panel |
| Elevation profile and route analytics | Per-route chart, average and max gradient, distance by gradient band (e.g. > 10%) |
| Sponsors | Banner area and rotating logos from the sponsors table |
| Donation | Modal with a QR code (QRIS) and short message |
| Share | Link to a runner (`?bib=`), Web Share API |
| Mobile-first layout | Bottom sheet for runner list/details, large tap targets, installable PWA |
| Status banners | "Tracking paused", "Weather hold", or "Event finished" messages from the admin |

### 1.2 Data loading

```text
1. GET /v1/public/events/{slug}              (routes, checkpoints, sponsors, config)  -- cached 5 min
2. GET /v1/public/events/{slug}/snapshot     (latest positions)                         -- cached 3-5 s
3. Open SSE /v1/public/events/{slug}/stream  (deltas)  ; fall back to polling snapshot every 5 s
4. On runner select: GET /v1/public/runners/{id}/track  (simplified polyline)
```

Optimizations:
- Gzip/Brotli responses; snapshot uses compact arrays (`[bibIndex, lat, lon, routeDist, ts, state]`).
- Viewport-based filtering on the client; throttle rendering to 1-2 fps for marker updates when more than a few hundred runners are visible.
- Keep the SSE connection alive with heartbeat comments; reconnect with `Last-Event-ID`.

### 1.3 Weather layers

| Layer | Source | Notes |
|---|---|---|
| Rain radar | RainViewer API tiles | Free tier; animated frames; respect their attribution and terms |
| Cloud, wind, temperature tiles | OpenWeatherMap tile layers | API key required; limits apply; **proxy through your server** to hide the key and cache tiles |
| Current weather panel | OpenWeatherMap (or another provider) current-conditions endpoint | Fetched server-side every ~10 min for the event location, cached in Redis |

Always display the required attributions. Provide a layer toggle and a legend (rain intensity, wind speed, temperature scale). Check each provider's current terms and rate limits before the event.

### 1.4 Elevation profile and route analytics

Computed once when a GPX is uploaded (admin pipeline), stored in `routes.profile`:

1. Parse GPX points (lat, lon, ele). If elevation is missing or noisy, replace it from a DEM (SRTM/Copernicus) and smooth it (e.g. 30 m moving window).
2. Compute cumulative distance and gradient between samples: `grade% = Δelev / Δdist * 100` over ~50 m windows.
3. Statistics: total gain/loss, min/max elevation, average gradient (uphill and downhill separately), max gradient, distance above 10% (and other bands: 5-10%, > 15%).
4. Store `[distance_m, elevation_m, grade_pct]` as the profile array.

The chart shows elevation vs distance with color bands by gradient, checkpoints as markers, and live runner dots.

### 1.5 Map tiles and styles

- Choose a provider that allows your expected traffic: MapTiler, Stadia, Mapbox, or self-hosted OpenStreetMap vector tiles (Protomaps/PMTiles on a CDN is a cost-effective option).
- Terrain mode: contour lines/hillshade or a terrain style.
- Satellite mode: provider-licensed imagery (watch usage limits and cost).
- Each theme (light/dark) uses its own style JSON.

### 1.6 Privacy on the public site

- Show only runners with an active `location` consent.
- Name/photo only with `public_profile`; otherwise bib only.
- No heart rate unless `public_heart_rate` is active.
- Optional **delay** (e.g. 60-120 s) on public positions for security/safety, configurable per event.
- Optional rule: stop publishing positions after the runner finishes.
- Kill switch from admin.

## 2. Admin dashboard

**Roles:** `organizer`, `admin`. Auth with SSO or email + password with MFA.

### 2.1 Event setup
- Create event, set timezone, start/end, public switches, delay settings
- Upload GPX per route; preview on the map; compute profile and stats
- Place checkpoints on the route (snap to route, set radius and cut-off)
- Manage sponsors and the donation QR image
- Weather location and layer settings

### 2.2 Runner management
- Import runners (CSV: bib, name, category, route, wave, optional birth year/gender)
- Generate enrollment codes (printed on bibs, or emailed/SMS-ed)
- Device list: platform, app version, last seen, battery, attestation status, consents granted
- Reassign device, revoke tokens, mark DNF or withdraw, manual checkpoint entry
- Message runners (push) individually or by group

### 2.3 Live operations
- Live map with all runners, filters, search, and a "silent runners" list
- Alert list (silent, off-route, stationary, cut-off risk, missed checkpoint, integrity flags)
- SOS queue with acknowledge, dispatch, resolve, and notes
- Checkpoint view: arrivals, expected next arrivals (for aid-station staff), cut-off countdowns
- Kill switch and banner messages for the public page

### 2.4 Reports
- Splits and finish times (CSV export)
- Tracking health: data latency, silent time per runner, battery stats
- Incident log

## 3. Medical dashboard

A separate view with role `medical` and MFA. Contents are defined in `06-heart-rate-privacy-consent.md` (section 6.4): runner status board, alert queue, per-runner heart-rate chart with pace and elevation, last position and nearest aid station, acknowledgement actions. Every heart-rate read is audited.

## 4. Aid-station helper (optional)

A simple mobile-web page for aid-station volunteers:
- List of runners expected within the next 30-60 minutes with ETAs
- Scan runner QR/NFC to confirm arrival (manual checkpoint pass)
- Mark runner status (Taking a rest, Withdrawn, Needs medical)

This also provides backup timing when a runner's phone dies.

## 5. Accessibility and performance targets

- Works on mid-range phones over 4G; initial load < 200 KB JS before map (code splitting), map ready < 3 s on a typical connection
- Dark mode and high-contrast mode; keyboard-accessible controls; readable fonts in sunlight
- Languages: Indonesian and English (i18n files)
- Public snapshot endpoint can handle spikes via CDN (target: tens of thousands of viewers on cached responses)

## 6. Testing

- Visual tests with the simulator (2,000 runners)
- Slow network and offline (PWA shell)
- Theme switching and basemap changes
- Verify no heart rate in any public response (automated contract test)
- Cross-browser and low-end device checks
