# MDOT Mi Drive API Reference

Undocumented internal endpoints powering the Mi Drive website. No authentication, no cookies, no CORS required (server-side only). Stable as of 2026-04-15 but could change without notice — build defensively, handle 4xx/5xx gracefully, tolerate single-endpoint failures.

**Base URL**: `https://mdotjboss.state.mi.us`

## Commute bounding box

All spatial filtering for Tom's commute uses this box:

- Latitude: **42.30** (south, downtown Detroit) to **42.65** (north, Rochester Hills / M-59)
- Longitude: **−83.25** (west) to **−83.03** (east)

Any incident, closure, or DMS sign within this box AND on I-75 (or feeding into I-75 via I-696, I-94, M-10) is potentially relevant.

## Endpoint 1: DMS Signs (Dynamic Message Signs)

**The highest-value data source.** These are the electronic highway signs that display real-time travel times, warnings, closure notices, crash alerts, and weather messages. MDOT updates them constantly.

### List all signs

```
GET /MiDrive/dms/AllForMap
```

Returns an array of `{latitude, longitude, id, title, icon}` objects, ~487 total statewide. The routine does **not** call this endpoint — the SB I-75 sign IDs are hardcoded below.

### Get sign message detail

```
GET /MiDrive/dms/getDMSInfo/{id}
```

Returns a 2-element JSON array: `[htmlMessage, signTitle]`.

Example for ID 16539:

```json
[
  "<div class='dmsMessage'> TRAVEL TIME TO<br> 14 MILE RD  7 MI<br> 5 MIN</div><div class='dmstimeStamp'>Apr 15 2026, 1:54 PM</div>",
  "SB I-75 @ N of Crooks Rd."
]
```

The message text lives inside `<div class='dmsMessage'>`, line breaks are `<br>`, and the update timestamp is in `<div class='dmstimeStamp'>`. **No regex extraction is needed by the routine** — Claude reads the raw HTML and reasons over it directly.

### What "normal" looks like

- `TRAVEL TIME TO / B. BEAVER 7 MI 5 MIN / I-696 15 MI 13 MIN`
- `TRAVEL TIME TO / 14 MILE RD 7 MI / 5 MIN`
- `TRAVEL TIME TO / 14 MILE 4 MI 4 MIN / I-696 8 MI 8 MIN`

### What "abnormal" looks like

- `ONE LANE OPEN / AFTER ROSA PARKS` (major closure)
- `RAMP CLOSED / TO I-696 WEST / USE 8 MILE WEST` (ramp closure)
- `LEFT LANE CLOSED / AFTER SASHABAW` (lane closure upstream)
- Travel times dramatically above baseline (e.g., 25 min for a segment that's normally 5 min)
- Messages containing `CRASH`, `CLOSED`, `BLOCKED`, `FLOODING`, `DELAY`, `ICE`

### SB I-75 DMS signs (used by the routine, north → south)

| ID | Title | Latitude |
|---|---|---|
| 16540 | SB I-75 @ N of South Blvd. | 42.625454 |
| 15169 | SB I-75 @ Adams | 42.60962 |
| 16539 | SB I-75 @ N of Crooks Rd. | 42.60496 |
| 16537 | SB I-75 @ S of Wattles Rd. | 42.57371 |
| 16535 | SB I-75 @ Rochester Rd. | 42.559196 |
| 16544 | SB I-75 @ S of 13 Mile Rd. | 42.516388 |
| 16543 | SB I-75 @ 12 Mile Rd. | 42.505814 |
| 16532 | SB I-75 @ N of 9 Mile Rd. | 42.462307 |
| 2782 | SB I-75 @ McNichols | 42.422073 |
| 2146 | SB I-75 @ Clay | 42.378395 |
| 2144 | SB I-75 @ Canfield | 42.35687 |
| 2766 | SB I-75 @ Rosa Parks | 42.33476 |

### NB I-75 DMS signs (not used by the current routine — return trip is not routine-gated)

Retained for reference. Available via the same `getDMSInfo/{id}` endpoint if a future version adds an afternoon run.

| ID | Title | Latitude |
|---|---|---|
| 2149 | NB I-75 @ S of Bagley | 42.322216 |
| 2145 | NB I-75 @ Clay | 42.38253 |
| 2781 | NB I-75 @ 7 Mile | 42.431503 |
| 16531 | NB I-75 @ 8 Mile | 42.44976 |
| 16533 | NB I-75 @ S of Woodward Heights Blvd. | 42.468098 |
| 16542 | NB I-75 @ Gardenia Ave. | 42.495102 |
| 15170 | NB I-75 @ 12 Mile by split | 42.50943 |
| 16545 | NB I-75 @ 13 Mile Rd. | 42.518833 |
| 16534 | NB I-75 @ Maple Rd. | 42.551243 |
| 16536 | NB I-75 @ Livernois Rd. | 42.55924 |
| 16538 | NB I-75 @ Wattles Rd. | 42.576736 |
| 2187 | NB I-75 @ S of M-59 | 42.62694 |

## Endpoint 2: Active Incidents

### All incidents

```
GET /MiDrive/incidents/AllForMap/
```

Returns array of objects. Each has `latitude`, `longitude`, `id`, `title`, `icon`, and an HTML `message` field with striped divs containing `Location`, `Lanes Blocked`, `Event Type`, `County`, optional `Event Message`, and `Reported` fields.

Example:

```json
{
  "latitude": 42.239574,
  "longitude": -83.1996,
  "id": 1082482,
  "title": "Crash on SB  I-75",
  "message": "<div class='stripeLight'><strong>Location: </strong>SB I-75 after Dix Hwy</div><div class='stripeDark'><strong>Lanes Blocked: </strong>Right Lane, Right Shoulder</div><div class='stripeLight'><strong>Event Type: </strong> Crash</div><div class='stripeDark'><strong>County: </strong>Wayne</div><div class='stripeLight'><strong>Reported:</strong> 1:28 PM"
}
```

**Event types**: Crash, Debris, Flooding, Disabled Vehicle, Other. Crashes, flooding, and multi-lane blockages are the high-signal events.

### Filtering

The routine applies the bounding box first (lat 42.30–42.65, lon −83.25 to −83.03). If the in-box list is still long, it further narrows to items whose `title` contains `I-75`, `I-696`, `I-94`, or `M-10` — those are the corridor and its feeders.

### Alternative structured endpoint

```
GET /MiDrive/incidents/AllForPage
```

Returns `incidentId`, `incidentTitle`, `incidentText`, `latitude`, `longitude`, `gotoLink`. More detail but still HTML in `incidentText`. Either endpoint works; the routine uses `AllForMap`.

## Endpoint 3: Construction / Road Closures

### All closures (on map)

```
GET /MiDrive/construction/AllForMap/
```

Returns array of `{latitude, longitude, id, title, icon, coordinatePoints, active}` objects.

### Icon codes (severity)

- `"1"` — **Total Closure (RED)** — most impactful, always flag
- `"2"` — **Lane Closure (ORANGE)** — flag if multiple lanes or during rush hour
- `"3"` — **Special Event (BLUE)** — informational, usually not actionable
- `"4"` — **Future / Planned (GREEN)** — informational, not actionable

### `active` flag

`true` = currently in effect. `false` = paused (e.g., open for a holiday). The routine filters to `active == true` and `icon in ("1", "2")`.

### Detail by ID (not used by the current routine)

```
GET /MiDrive/construction/getConstructionInformation/{id}
```

Returns `[htmlDetail, title]`. The detail includes description, start date, end date, last updated date. The bulk list is enough for the routine's reasoning layer.

### Other construction endpoints (not used by the current routine)

- `GET /MiDrive/construction/getConstructionJobsFromSearch?route=SB+I-75&county=` — structured search by route
- `GET /MiDrive/construction/list/loadConstruction` — all ~629 closures statewide, structured JSON
- `GET /MiDrive/construction/Counties`, `/Routes`, `/counties/{x}/routes`, `/routes/{x}/counties` — filter helpers

## Cameras (not used by the current routine)

`GET /MiDrive/camera/AllForMap/` returns ~785 cameras. `GET /MiDrive/camera/getCameraInformation/{id}` returns the live JPEG URL and metadata. Images refresh every ~2 seconds.

Retained for possible future use (attach a snapshot URL to actionable notifications). Key camera IDs on the commute corridor:

| ID | Title |
|---|---|
| 1037 | I-75 @ Crooks |
| 2681 | I-75 @ Adams |
| 2413 | I-75 @ Big Beaver |
| 2411 | I-75 @ 14 Mile |
| 2212 | I-696 @ I-75 |
| 75 | I-75 @ S of I-696 |
| 5 | I-375 @ I-75 (downtown) |

## Endpoints NOT used by this routine

- `/MiDrive/plows/AllForMap/` — winter snowplow tracking. Refreshes every 90 sec. Candidate for a Dec–Mar seasonal enhancement.
- `/MiDrive/parking/getMapParking/` — truck parking. Only 5 statewide, none on corridor.
- `/MiDrive/tollBridges/allForMap/` — border crossings. Not on commute.

## Notes and caveats

- **No rate limiting observed**, but be respectful — one morning run, not continuous polling
- **HTML inside JSON** is the dominant pattern for detail endpoints. The routine reads the raw HTML via Claude and does not do regex extraction
- **Undocumented internal endpoints** — stable but could change. Handle 4xx/5xx gracefully and tolerate single-endpoint failures; only complete data loss should abort a run
- **No auth, no cookies, no CORS** — confirmed working with plain `curl` server-side
- **Spatial filter first, shape filter second** — always reduce with `jq` before handing data to the reasoning layer; statewide data in context is wasted tokens
