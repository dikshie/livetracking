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

### 1.7 Option: Google Maps JavaScript API instead of MapLibre

The viewer's data flow (snapshot + SSE, privacy rules, runner cards, elevation chart) is identical. Only the map layer changes.

**Map adapter interface** (so either engine can be used per event):

```ts
interface MapAdapter {
  init(container: HTMLElement, opts: { theme: 'light' | 'dark'; mode: 'standard' | 'satellite' | 'terrain' }): void;
  drawRoute(routeId: string, geojson: GeoJSON.LineString, color: string): void;
  setRunners(runners: RunnerPoint[]): void;      // batched update, animated
  addCheckpoints(cps: Checkpoint[]): void;
  setWeatherLayer(kind: 'rain' | 'cloud' | 'wind' | 'temp' | null): void;
  setTheme(theme: 'light' | 'dark'): void;
  setMode(mode: 'standard' | 'satellite' | 'terrain'): void;
  focusRunner(bib: string): void;
  onRunnerClick(cb: (bib: string) => void): void;
}
```

**Google Maps implementation notes**

| Need | Approach |
|---|---|
| Load | Dynamic library import (`google.maps.importLibrary('maps')`, `'marker'`), created once per session |
| Map modes | `mapTypeId`: `roadmap`, `satellite`, `hybrid`, `terrain` |
| Light/dark | One cloud-styled Map ID per theme; recreate or switch Map ID when the theme changes |
| Routes | `google.maps.Polyline` (one per route) or Data layer `addGeoJson` |
| Runner markers (up to ~500) | `AdvancedMarkerElement` with a small custom DOM pin showing the bib; `@googlemaps/markerclusterer` for clustering |
| Runner markers (500+) | deck.gl `GoogleMapsOverlay` with a `ScatterplotLayer` / `IconLayer` (WebGL), picking for clicks |
| Checkpoints | Advanced markers + `InfoWindow` or the app's own bottom sheet |
| Weather tiles | `new google.maps.ImageMapType({ getTileUrl, tileSize: new google.maps.Size(256, 256), opacity })` pushed into `map.overlayMapTypes`; tile URLs go through your server proxy so API keys stay private |
| Marker animation | Interpolate positions in `requestAnimationFrame` between updates |
| Elevation hover | Chart hover sets a marker on the polyline at that route distance |

**Cost and policy checklist**

- Map loads are billed; load the map once per page session and avoid re-initializing it.
- Restrict the API key by HTTP referrer; set quotas and budget alerts before the event.
- Keep Google's logo and attribution visible; do not cache or store Google map tiles or content.
- Estimate cost from expected spectators x sessions before choosing Google for a large event; compare with MapLibre + self-hosted or provider tiles.
- Do not call Google Elevation or Directions per runner at race time; elevation is precomputed from the GPX/DEM.

### 1.8 Cost-effective map options (self-hosted and low-cost providers)

For a single event, a self-hosted vector basemap is usually the cheapest approach. Pricing and terms change, so re-check each provider before the event.

#### Recommended: Protomaps (PMTiles) + MapLibre

- Generate one vector-tile file (`.pmtiles`) covering only the event area from OpenStreetMap data (small compared with the whole planet).
- Host the file on object storage behind a CDN (e.g. Cloudflare R2, or S3 + CloudFront).
- MapLibre reads it directly with HTTP range requests (PMTiles protocol). No tile server, no per-map-load fees, no API key.
- Use Protomaps' light and dark basemap styles for the theme toggle.
- Cost is mostly storage and bandwidth; CDN caching keeps both low during spectator spikes.

**Build and hosting steps**

1. Choose the bounding box around all routes plus a margin.
2. Extract that area from a Protomaps/OSM build into a `.pmtiles` file using the `pmtiles` CLI.
3. Upload the file to object storage; enable CORS and range requests; put a CDN in front with long cache times.
4. In MapLibre, register the `pmtiles://` protocol and point the style's vector source at the file URL.
5. Set `maxBounds` and a limited zoom range on the map so users only request tiles that exist.
6. Keep light and dark style JSON files in your app; switch with `setStyle`.

#### Terrain and elevation without a paid provider

- Use open elevation data (Copernicus or SRTM-derived) as raster-DEM tiles.
- MapLibre can render hillshade from them; the `maplibre-contour` plugin can generate contour lines in the browser.
- To avoid depending on a public DEM host during the event, cache the DEM tiles for your bounding box (or package them as a PMTiles file) and serve them from your own CDN.
- OpenTopoMap exists as a ready-made topographic style, but its usage policy discourages heavy production traffic.

#### Satellite imagery (the hard part)

Good imagery is rarely free for commercial use. Options:

| Option | Notes |
|---|---|
| Optional paid layer from a provider (MapTiler, Mapbox, Stadia) | Load only when the user switches to satellite; set a hard budget cap |
| Open imagery (Sentinel-2 based layers) | Check license (some are non-commercial); low resolution for trail detail |
| Skip satellite | Terrain + route is often enough for a trail event |

Default view stays on the free vector basemap, so most viewers never trigger satellite costs.

#### Hosted providers (if not self-hosting)

| Provider | Notes |
|---|---|
| MapTiler | Free plan is non-commercial only, requires the MapTiler logo, and stops at about 5,000 sessions and 100,000 requests per month. A commercial event needs a paid plan; check current pricing. |
| Stadia Maps, Thunderforest, Geoapify, Mapbox | Free tiers and low-cost plans; compare per-session or per-request limits against expected spectators |
| OpenStreetMap public tile servers | **Do not use** for production events; their policy does not allow heavy use |

#### Other savings

- Limit the map to the event's bounds and a limited zoom range.
- Proxy and cache weather tiles on your server (RainViewer, OpenWeatherMap) so thousands of viewers share a few upstream requests; re-check their free-tier terms before the event.
- Serve the public snapshot through a CDN with a short TTL (3-5 s).
- Prefer vector tiles over raster tiles (smaller, cache better, work for both themes).

#### Comparison

| Option | Cost during spectator spikes | Setup effort | Satellite |
|---|---|---|---|
| Google Maps | Highest (billed per map load) | Low | Yes |
| Hosted provider (paid plan) | Medium (plan-based) | Low | Usually |
| Protomaps + CDN | Lowest | Medium (one-time setup) | No; optional paid layer |

**Recommendation:** Protomaps on MapLibre for streets and terrain, with satellite either skipped or limited to a budget-capped paid provider. Keep the map adapter interface from section 1.7 so Google Maps can be used for an event where budget is not a concern.

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
