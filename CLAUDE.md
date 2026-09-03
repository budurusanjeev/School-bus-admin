# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository ("MAHATHI - Advanced Fleet Control & Live Panel") is a single static HTML file (`Index.html`) implementing an admin panel for a school bus fleet management system. There is no build system, package manager, or test suite — it is a plain HTML/CSS/JS file that can be opened directly in a browser or served by any static file server.

Note the file is capitalized `Index.html`, not `index.html`.

## Running / Testing

There is no build step. To view changes, serve the directory locally (Leaflet's assets are loaded relatively, so opening via `file://` directly also works but a local server is closer to real deployment), e.g.:

```
npx serve .
```

There are no automated tests, linter, or CI configuration in this repo.

## Architecture

Everything — markup, styles, and logic — lives in `Index.html` as a single self-contained page.

- **Backend**: all data operations hit a remote API defined by `BASE_URL` (`https://school-bus-backend-2.onrender.com/api/v1`). This admin panel is a pure frontend client for that backend — there is no server code in this repo.
- **Dependencies are vendored locally, not via CDN**: `leaflet.js`/`leaflet.css` (Leaflet 1.9.4) live next to `Index.html` in this directory. Do not switch these back to a CDN (`unpkg.com` etc.) — that CDN has been observed to be blocked on some networks, which silently breaks the map (this exact failure mode was hit and fixed in the sibling `Driverapp`/`ParentApp` Android projects first). Reverse geocoding on map clicks still uses the OpenStreetMap Nominatim API directly (no local alternative).
- **Auth is server-verified now**: the login gate (`#auth-gate`, `checkAuth()`) sends the entered password to `GET /api/v1/admin/auth-check` in the `x-admin-key` header; the backend compares it to its `ADMIN_PASSWORD` env var. On success the key is kept in `sessionStorage` (auto-unlock on reload of the same tab) and attached to **every** admin request via `adminHeaders()` — `submitJson()`, `tryFetchList()`, and the fleet-locations poll all include it, and all three treat a 401 as "key revoked" (`handleUnauthorized()` clears the key and reloads to the gate). No password lives in this file anymore. When adding any new admin fetch, always spread in `adminHeaders()` and handle 401 the same way, or the request will silently fail with the backend's auth middleware.
- **Layout gotcha**: `body` is `display: flex` with explicit `width: 100vw`/`height: 100vh`. `#app-root` (which wraps `#sidebar` + `#map-wrapper`, toggled from `display:none` to `display:flex` by `checkAuth()` once unlocked) is therefore itself a flex *item* of `body` and needs its own explicit `width: 100%; height: 100%` — without it, it shrinks to fit just the 380px sidebar and the map gets zero width. If you ever restructure the top-level layout, keep this in mind.
- **Sidebar workflow is a 5-step sequential form flow**, each step calling its own handler function and hitting a corresponding backend endpoint:
  1. Register Driver → `POST /admin/drivers`
  2. Create Bus Asset → `POST /admin/buses` (returns a bus UUID used in later steps)
  3. Open Route Container → `POST /admin/routes`
  4. Deploy Configuration (link driver + route to a bus) → `PUT /admin/buses/{uuid}/assign`
  5. Add Bus Stop → `POST /admin/routes/{routeId}/stops` — this step is tied to the map: clicking the map fills in lat/lng and reverse-geocodes an address into the stop name field via Nominatim.
- **Steps 4/5's bus/driver/route fields are `<select>` dropdowns, not free-text ID inputs** — this replaced the original design where the admin had to copy-paste a UUID/ID from a previous step's success alert, which was extremely error-prone. They're populated from `cachedBuses`/`cachedDrivers`/`cachedRoutes` (`Map<id, label>`), filled two ways: (1) whatever this panel itself creates during the session gets pushed into the relevant cache immediately after a successful create (in `registerDriver`/`createBus`/`createRoute`), and (2) `refreshAllOptions()` (called once from `startApp()`, and manually via the "↻ Refresh" link in Step 4) opportunistically fetches list endpoints and merges them in. Only `GET /api/v1/buses` is confirmed to exist (same endpoint Driverapp/ParentApp use) — `GET /api/v1/admin/drivers` and `GET /api/v1/admin/routes` are *not* confirmed to exist on the backend, so `tryFetchList()` silently no-ops on failure rather than breaking the UI. If the backend later adds proper list endpoints for drivers/routes, `refreshAllOptions()` will pick them up automatically once the URLs there are correct — check `BASE_URL + '/admin/drivers'` / `'/admin/routes'` still match. **Always add newly-created entities to the cache and call the matching `render*Options()`** when adding new create-flow steps — don't reintroduce a raw ID text input for anything the panel itself already knows how to create.
- **Shared submit helper**: all 5 handlers go through `submitJson(url, method, payload)`, which checks `response.ok` and throws with a message pulled from the response body (`error`/`message` fields) or the HTTP status on failure. Each handler validates its required fields are non-empty before calling it, disables its button and shows a loading label via `setLoading()` while in flight, and clears its form fields on success. Follow this same pattern for any new form step — don't add a fetch call that skips validation/error handling/loading state, since that was the state of every handler before this pattern was introduced and it silently showed "Success!" alerts even on failed requests.
- **Live fleet tracking**: `trackLiveFleetPipeline()` polls `GET /admin/fleet/locations` every 5 seconds (`setInterval`, started from `startApp()` after auth succeeds — not before) and reconciles the result against `activeLiveBusMarkers` (an in-memory map of `bus_id -> Leaflet marker`) — updating positions for buses still reporting, adding new markers for newly-seen buses, and removing markers for buses that stopped reporting. The `#live-fleet-counter` badge in the header reflects the current active bus count.
- Backend response shapes are assumed directly in the JS (e.g. `data.id`, `data.bus.id` fallback, `busLog.bus_id/latitude/longitude/bus_number/route_name/speed`) — there is no shared type/schema, so if the backend API changes, these field accesses need manual updates.

## Deploy Configuration

- Platform: Render static site (Blueprint in `render.yaml`)
- Service name: `school-bus-admin`
- Deploy trigger: auto-deploy on push to the connected Git branch (`main`)
- Build: `cp Index.html index.html` (Render is Linux; `/` looks for lowercase `index.html`)
- Publish path: repo root (also serves `leaflet.js` / `leaflet.css`)
- Do not add a catch-all SPA rewrite to `index.html` — that would hide the Leaflet assets.
- Production URL: `https://school-bus-admin.onrender.com` (confirm after first deploy; Render may suffix a random string if the name is taken)
- This site still calls `https://school-bus-backend-2.onrender.com`. If login or API calls fail in the browser with a CORS error, add this site's origin to the backend CORS allowlist.
