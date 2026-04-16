# Google Maps Routes API Reference

Google Maps Platform Routes API v2, `computeRoutes` endpoint. Used by commute-watch for today-vs-historical-band ETA comparison (via three parallel calls at the same `departureTime` with different `trafficModel` values) and alternative route scoring. Paid API but well within the free monthly credit at the routine's call volume.

**Endpoint**: `POST https://routes.googleapis.com/directions/v2:computeRoutes`

## Authentication

Send the API key as a header:

```
X-Goog-Api-Key: $GOOGLE_MAPS_API_KEY
```

The API key lives in the routine's cloud env vars. It should be scoped (HTTP referrer or IP restrictions don't apply for server-side calls; use "Restrict key > API restrictions" in the GMP console to limit to the Routes API only).

## Field mask (required)

The `X-Goog-FieldMask` header is **required** — requests without it return 400. This forces callers to opt into the response fields they actually need, reducing bandwidth and cost. The field mask is a comma-separated list of response paths.

The routine uses two field masks:

**Call 1 (today's ETA + alternatives)**:

```
routes.duration,routes.staticDuration,routes.distanceMeters,routes.routeLabels,routes.legs.steps.navigationInstruction,routes.warnings
```

**Call 2 (typical baseline only)**:

```
routes.duration,routes.staticDuration
```

If call 1 returns a 400 on the field mask (e.g., Google changes a field name), the routine retries once with the minimal mask `routes.duration,routes.staticDuration` and proceeds with degraded capability.

## Request body shape

```json
{
  "origin": {"address": "$COMMUTE_ORIGIN"},
  "destination": {"address": "$COMMUTE_DESTINATION"},
  "travelMode": "DRIVE",
  "routingPreference": "TRAFFIC_AWARE_OPTIMAL",
  "departureTime": "2026-04-15T12:15:00Z",
  "computeAlternativeRoutes": true,
  "languageCode": "en-US",
  "units": "IMPERIAL"
}
```

### Key fields

- **`origin` / `destination`** — Waypoint objects. The union supports `address` (plain string, what we use), `location` (lat/lng), `placeId`, and `navigationPointToken`. Plain addresses are simpler and don't require a separate geocoding step.
- **`travelMode`** — `DRIVE` for this routine.
- **`routingPreference`** — `TRAFFIC_AWARE_OPTIMAL` (best quality, DRIVE/TWO_WHEELER only; server auto-falls-back if it would cause latency), `TRAFFIC_AWARE` (default quality), or `TRAFFIC_UNAWARE` (free-flow, no traffic data). We use OPTIMAL.
- **`departureTime`** — RFC3339 timestamp with `Z` suffix (UTC). **Must not be in the past for DRIVE mode** — past values return 400 `"Timestamp must be set to a future time"` (only `TRANSIT` accepts past departures). The routine computes it from America/Detroit local time 08:15 and rolls forward one day if the current local time is already past 08:15. This is a real footgun: test runs during the day and late-firing scheduled runs will both hit it if you don't roll forward.
- **`computeAlternativeRoutes`** — when `true`, Google returns up to 2 alternatives alongside the default route. Only works when the request has no intermediate waypoints (we have none). Works with all routingPreference values for DRIVE.
- **`languageCode`** — affects navigation instruction text; we use `en-US`.
- **`units`** — `IMPERIAL` for miles in any distance fields. Duration is always returned in seconds regardless.

### Notes on `arrivalTime`

Routes API v2 has an `arrivalTime` field, but it is **ignored for DRIVE mode** — only TRANSIT routes support it. The reference explicitly states: *"This field is ignored when requests specify a RouteTravelMode other than TRANSIT."*

The routine works around this by anchoring on a fixed `departureTime` (08:15 local) and computing "expected arrival" as `08:15 + today_duration` to check against the 09:00 target. If Tom ever wants to change the anchor departure time, edit the "Configuration" block at the top of `routines/commute-watch.md` and commit.

### The `trafficModel` parameter

This is the parameter that unlocks v4's same-departureTime approach. Available values:

- **`BEST_GUESS`** (default) — combines historical patterns with live real-time traffic data. Google's docs: *"Live traffic becomes more important the closer the departure_time is to now."*
- **`OPTIMISTIC`** — models favorable traffic conditions from historical data only, no live adjustment. Effectively a historical lower-bound for this weekday + time of year.
- **`PESSIMISTIC`** — models poor traffic conditions from historical data only, no live adjustment. Historical upper-bound.

Google's docs include this telling sentence: *"It is possible the BEST_GUESS travel time prediction may be shorter than OPTIMISTIC or longer than PESSIMISTIC, due to the way BEST_GUESS integrates live traffic information."* This confirms that OPTIMISTIC and PESSIMISTIC are derived purely from the historical distribution for this departure time with **no live adjustment**, while BEST_GUESS can exceed the band in either direction when live conditions are unusual.

**Required companions**: `trafficModel` is only valid when `routingPreference: TRAFFIC_AWARE_OPTIMAL` and `travelMode: DRIVE`. We satisfy both.

## Response shape

The response is a `ComputeRoutesResponse` with a `routes` array. With `computeAlternativeRoutes: true` you get the default route first and up to 2 alternatives after.

Each `Route` object has (among other fields):

- **`duration`** — traffic-aware duration when `routingPreference` requests traffic. Format: string with trailing `s`, e.g., `"1534.5s"`. To get minutes: strip `s`, parse as float, divide by 60.
- **`staticDuration`** — free-flow duration (no traffic). Same string format. Useful as a floor reference.
- **`distanceMeters`** — integer meters. Divide by 1609.344 for miles.
- **`routeLabels`** — enum array. Values include `DEFAULT_ROUTE` (the primary), `DEFAULT_ROUTE_ALTERNATE`, `FUEL_EFFICIENT`. Use this to identify which route is which.
- **`legs[].steps[].navigationInstruction`** — each step has an `instructions` string like `"Head south on I-75 S"`. The first step's instruction typically reveals which highway the route takes, which lets you tell "via I-75" from "via M-10" without hardcoding.
- **`warnings`** — array of strings with route-level warnings (construction, road conditions). Rare; useful when present.

### Parsing duration to minutes

```bash
# jq inline
(.routes[0].duration | rtrimstr("s") | tonumber / 60)
```

Or in Python:

```python
float(route["duration"].rstrip("s")) / 60
```

## Why three calls at the same departureTime

Routes API v2 gives you "duration for a specific departure time." It does not directly expose a single-field "typical travel time for this weekday + time of year." But the `trafficModel` parameter lets you isolate live-adjusted vs purely-historical predictions for the **same** departure time, which is exactly what the abnormality signal wants.

The routine makes three parallel calls to `computeRoutes` at the same `departureTime` (today's 08:15 anchor, or tomorrow's if today's has passed):

- **Call 1 — `BEST_GUESS`** with `computeAlternativeRoutes: true` → today's live-adjusted ETA, plus alternatives
- **Call 2 — `OPTIMISTIC`** → historical lower bound (a "good day" at this time slot)
- **Call 3 — `PESSIMISTIC`** → historical upper bound (a "bad day" at this time slot)

The band `[OPTIMISTIC, PESSIMISTIC]` is Google's empirical historical range for this commute at this weekday + time. `BEST_GUESS` tells you where today actually sits in (or outside) that band. This is cleaner than v3's `departureTime=+7d` hack because:

1. All three calls use the **same** `departureTime`, so you're comparing like-for-like (no apples-to-oranges between "today at 08:15" and "next Wednesday at 08:15")
2. The historical band is **explicit** rather than a single median proxy
3. **No dependence on how far in the future the anchor is.** v3's approach degraded badly when the routine ran late in the day — both calls landed in Google's historical regime and the delta collapsed to zero. The trafficModel approach avoids that entirely because OPTIMISTIC and PESSIMISTIC are always purely historical regardless of run time
4. The parameter is Google's **documented** surface for this question, not a workaround

### Call 1's live signal still depends on run timing

BEST_GUESS weights live data by proximity. When the routine runs at 07:45 for an 08:15 anchor, Call 1 is 30 min out and heavily live-weighted — today's real conditions drive the prediction. When the routine runs much earlier (v3's 06:45 schedule was problematic for this reason) or when the anchor has rolled forward to tomorrow (late-running test invocations), BEST_GUESS lands deeper in historical territory and may not meaningfully differ from the band midpoint.

In that case, lean on MDOT for fresh signals. The band itself (OPTIMISTIC/PESSIMISTIC) remains a valid historical reference regardless of when the routine runs, because those calls don't depend on live traffic.

The reasoning layer in the routine (Step 4) handles this case explicitly: when BEST_GUESS is structurally close to the band midpoint because it's too far out, treat the quantitative signal as partially degraded and weight MDOT heavier.

## Pricing

Google Maps Platform pricing (as of 2026):

| SKU | Monthly free tier | $/1k (0-100k range) |
|---|---|---|
| Routes: Compute Routes Essentials | 10,000 | $5.00 |
| Routes: Compute Routes Pro | 5,000 | $10.00 |

`computeRoutes` with `TRAFFIC_AWARE_OPTIMAL` and `computeAlternativeRoutes: true` bills under the higher SKU (Pro). The routine makes **3 calls per run × ~22 weekdays/month ≈ 66 calls/month**, well under the 5,000 free-tier cap. Effective cost is ~$0.

Alternative routes do not add a separate billing line — they're part of the same request. `trafficModel` doesn't add a separate billing line either.

## Failure handling

- **400 Bad Request** — usually a field mask issue or malformed body. Retry call 1 once with minimal field mask `routes.duration,routes.staticDuration`.
- **401 / 403** — API key is bad, unauthorized for Routes API, or has restrictions that block this origin. Abort the run with a failure push.
- **429** — rate limit. Sleep 30 seconds, retry once. If it happens twice, abort. (Unlikely given call volume.)
- **500 / 503** — transient. Retry once with a brief delay, then degrade gracefully (treat as "Google call failed" per routine Step 3d).
- **Missing route in response** — if `routes` is empty, treat as a call failure.

The routine can proceed with partial Google results — one or two of the three calls failing is tolerable. If all three Google calls fail, it falls back to MDOT-only reasoning with a strong NORMAL bias. Only a total MDOT + Google failure (all three MDOT endpoints AND all three Google calls) aborts the run.

## Links

- Routes API overview: `developers.google.com/maps/documentation/routes`
- computeRoutes reference: `developers.google.com/maps/documentation/routes/reference/rest/v2/TopLevel/computeRoutes`
- Waypoint reference: `developers.google.com/maps/documentation/routes/reference/rest/v2/Waypoint`
- Pricing: `developers.google.com/maps/billing-and-pricing/pricing`
