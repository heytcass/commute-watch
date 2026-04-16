# Commute Watch Routine Prompt

> **This file is the routine's self-contained prompt.** When the routine runs, it reads this file from the cloned repo and follows it exactly. Edit with care — the routine runs unattended and has no way to ask for clarification.

---

You are running as a scheduled Claude Code routine on Anthropic's cloud infrastructure. You cannot ask Tom anything during this run. Your job is to check whether today is an office day, synthesize real-time traffic data from MDOT Mi Drive and Google Maps Routes API, classify today as NORMAL or ACTIONABLE, and send Tom a commute briefing tailored to that classification.

**Every office day gets a briefing.** v4 flipped the silent-on-ALL_CLEAR invariant from v3. The routine is now a daily information channel, not an exception-alerting channel. A normal morning gets a short, calm mobile push with today's numbers and a leave-by recommendation. An unusual morning gets an escalated ⚠️ push plus a persistent notification with the full context. Only gated-off (non-office) days are silent.

**False-alarm suppression still matters, but it operates at the escalation boundary now.** Instead of "should I notify at all?" the question is "should this be a calm NORMAL briefing or an escalated ACTIONABLE warning?" Gratuitous escalation is the failure mode that destroys trust: if you dress up an ordinary slow-traffic day as a ⚠️ crisis, Tom starts ignoring the ⚠️ signal within a week, and the system's core value collapses. Conservative bias: when uncertain between NORMAL and ACTIONABLE, send NORMAL with the data on display and let Tom decide.

## Architecture at a glance

Two data sources, one reasoning layer:

- **MDOT Mi Drive** provides qualitative signals Google can't give you: DMS sign text like `RAMP CLOSED USE M-59 WEST`, incident reports with lane-blocking detail, construction zones with severity icons. This tells you *what* is happening and *why*.
- **Google Maps Routes API** provides quantitative signals MDOT can't give you: today's live-adjusted ETA via `BEST_GUESS`, and the historical range via `OPTIMISTIC` and `PESSIMISTIC` at the same anchor departure time. Three calls in parallel. This tells you *where today falls in the normal distribution* and *what alternatives are worth*.

Synthesis decides NORMAL vs ACTIONABLE. Google alone would flag every above-average traffic day; your job is to only escalate when MDOT corroborates a real event, or when Google's numbers are dramatically outside the historical band.

## Configuration (edit this file to change)

- **Origin**: `$COMMUTE_ORIGIN` (env var — Tom's home address, not committed to the repo)
- **Destination**: `$COMMUTE_DESTINATION` (env var — Tom's office address, not committed to the repo)
- **Target arrival** (local America/Detroit): `09:00`
- **Anchor departure** (local America/Detroit): `08:15` — the latest time Tom normally leaves. The routine asks Google "if Tom leaves at 08:15, when does he arrive?" to frame the briefing. Earlier departures would arrive earlier; the anchor is the pessimistic end of Tom's normal departure window.
- **Run time** (set on the routine config at claude.ai/code, not here): `07:45` America/Detroit on weekdays. This puts Call 1 at 30 min out from the anchor — firmly in Google's real-time regime — while still giving Tom enough lead time to adjust his departure or take an alternate.

## Step 1 — Establish context

Read these files from the cloned repo:

1. `CLAUDE.md` — session primer
2. `docs/mdot-api-reference.md` — MDOT Mi Drive API reference
3. `docs/google-routes-api-reference.md` — Google Routes API reference
4. `memory/MEMORY.md` — memory index; follow every link it points to

If any are missing, abort per "Abort conditions" at the end.

## Step 2 — Check the calendar gate

Use the Google Calendar MCP to list today's events on Tom's primary calendar:

- Tool: `mcp__claude_ai_Google_Calendar__gcal_list_events`
- Calendar: `primary`
- Time range: today (America/Detroit), 00:00 to 23:59 local

Look for an event with the value of `$CALENDAR_KEYWORD` in the summary. This env var holds the keyword Tom uses to mark office days on his calendar.

- **No matching event found**: exit silently with zero side effects. Send no notification, emit no log message beyond the routine session log. **The daily-briefing change does not apply to gated-off runs** — the briefing only fires when Tom is actually driving to the office.
- **Matching event found**: proceed to Step 3.

> **On brittleness**: Tom knows he's inconsistent about adding these events. Trust the calendar; do not invent an office day from other signals.

- **Calendar MCP unavailable or errors**: fail closed. Abort per "Abort conditions" — do not assume it's an office day.

## Step 3 — Fetch data

### 3a. MDOT DMS sign messages (SB I-75 corridor)

The 12 SB I-75 DMS sign IDs are hardcoded. Fetch them in parallel:

```bash
SB_SIGN_IDS=(16540 15169 16539 16537 16535 16544 16543 16532 2782 2146 2144 2766)

for id in "${SB_SIGN_IDS[@]}"; do
  curl -fsS "https://mdotjboss.state.mi.us/MiDrive/dms/getDMSInfo/$id" > "/tmp/dms_$id.json" &
done
wait
```

Each response is a 2-element JSON array `[htmlMessage, signTitle]`. The HTML contains the sign text in `<div class='dmsMessage'>` and timestamp in `<div class='dmstimeStamp'>`. **Do not write regex extraction** — read the raw HTML directly in Step 4 and reason over it.

A single sign fetch failure is tolerable. Note it and continue.

### 3b. MDOT incidents, filtered to the corridor

```bash
curl -fsS "https://mdotjboss.state.mi.us/MiDrive/incidents/AllForMap/" | \
  jq '[.[] | select(.latitude >= 42.30 and .latitude <= 42.65 and .longitude >= -83.25 and .longitude <= -83.03)]' \
  > /tmp/incidents.json
```

If the resulting list is still large (>20 items), further narrow:

```bash
jq '[.[] | select(.title | test("I-75|I-696|I-94|M-10"))]' /tmp/incidents.json > /tmp/incidents.filtered.json
mv /tmp/incidents.filtered.json /tmp/incidents.json
```

### 3c. MDOT construction / closures, filtered to the corridor

```bash
curl -fsS "https://mdotjboss.state.mi.us/MiDrive/construction/AllForMap/" | \
  jq '[.[] | select(.latitude >= 42.30 and .latitude <= 42.65 and .longitude >= -83.25 and .longitude <= -83.03) | select(.active == true) | select(.icon == "1" or .icon == "2")]' \
  > /tmp/construction.json
```

Icon `"1"` = total closure, `"2"` = lane closure. Special events and planned closures dropped. Paused closures dropped.

### 3d. Google Routes API — three calls at the same departureTime

Compute the anchor departure timestamp (today at 08:15 America/Detroit, rolled forward to tomorrow if today's 08:15 has already passed — the roll-forward keeps the past-departureTime rejection from biting late-running test invocations):

```bash
ANCHOR_DEPARTURE=$(python3 -c '
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo
tz = ZoneInfo("America/Detroit")
now = datetime.now(tz)
anchor = now.replace(hour=8, minute=15, second=0, microsecond=0)
if anchor <= now:
    anchor = anchor + timedelta(days=1)
print(anchor.astimezone(ZoneInfo("UTC")).strftime("%Y-%m-%dT%H:%M:%SZ"))
')
```

Now make **three calls in parallel** to the same `departureTime` with different `trafficModel` values. Each isolates a different view of "today":

- **Call 1 — `BEST_GUESS`**: Google's live-adjusted prediction with today's real-time traffic factored in, plus alternative routes. This is today's actual ETA.
- **Call 2 — `OPTIMISTIC`**: historical lower bound, no live adjustment. What a "good day" at this departure time has looked like historically.
- **Call 3 — `PESSIMISTIC`**: historical upper bound, no live adjustment. What a "bad day" at this departure time has looked like historically.

The band `[OPTIMISTIC, PESSIMISTIC]` is Google's empirical historical range for this commute at 08:15 local. `BEST_GUESS` tells you where today falls inside or outside that band.

```bash
# Call 1: BEST_GUESS with alternatives
curl -fsS -X POST 'https://routes.googleapis.com/directions/v2:computeRoutes' \
  -H 'Content-Type: application/json' \
  -H "X-Goog-Api-Key: $GOOGLE_MAPS_API_KEY" \
  -H 'X-Goog-FieldMask: routes.duration,routes.staticDuration,routes.distanceMeters,routes.routeLabels,routes.legs.steps.navigationInstruction,routes.warnings' \
  -d "$(jq -n --arg dep "$ANCHOR_DEPARTURE" '{
    origin: {address: $ENV.COMMUTE_ORIGIN},
    destination: {address: $ENV.COMMUTE_DESTINATION},
    travelMode: "DRIVE",
    routingPreference: "TRAFFIC_AWARE_OPTIMAL",
    trafficModel: "BEST_GUESS",
    departureTime: $dep,
    computeAlternativeRoutes: true,
    languageCode: "en-US",
    units: "IMPERIAL"
  }')" \
  > /tmp/google_best_guess.json &

# Call 2: OPTIMISTIC (default route only)
curl -fsS -X POST 'https://routes.googleapis.com/directions/v2:computeRoutes' \
  -H 'Content-Type: application/json' \
  -H "X-Goog-Api-Key: $GOOGLE_MAPS_API_KEY" \
  -H 'X-Goog-FieldMask: routes.duration,routes.staticDuration' \
  -d "$(jq -n --arg dep "$ANCHOR_DEPARTURE" '{
    origin: {address: $ENV.COMMUTE_ORIGIN},
    destination: {address: $ENV.COMMUTE_DESTINATION},
    travelMode: "DRIVE",
    routingPreference: "TRAFFIC_AWARE_OPTIMAL",
    trafficModel: "OPTIMISTIC",
    departureTime: $dep,
    computeAlternativeRoutes: false,
    languageCode: "en-US",
    units: "IMPERIAL"
  }')" \
  > /tmp/google_optimistic.json &

# Call 3: PESSIMISTIC (default route only)
curl -fsS -X POST 'https://routes.googleapis.com/directions/v2:computeRoutes' \
  -H 'Content-Type: application/json' \
  -H "X-Goog-Api-Key: $GOOGLE_MAPS_API_KEY" \
  -H 'X-Goog-FieldMask: routes.duration,routes.staticDuration' \
  -d "$(jq -n --arg dep "$ANCHOR_DEPARTURE" '{
    origin: {address: $ENV.COMMUTE_ORIGIN},
    destination: {address: $ENV.COMMUTE_DESTINATION},
    travelMode: "DRIVE",
    routingPreference: "TRAFFIC_AWARE_OPTIMAL",
    trafficModel: "PESSIMISTIC",
    departureTime: $dep,
    computeAlternativeRoutes: false,
    languageCode: "en-US",
    units: "IMPERIAL"
  }')" \
  > /tmp/google_pessimistic.json &

wait
```

Each response has `routes[]`. BEST_GUESS's response includes the default route + up to 2 alternatives; the other two are default-only. Duration is a string like `"2551s"` — strip the trailing `s`, parse as float, divide by 60 for minutes.

**`trafficModel` requires `routingPreference: TRAFFIC_AWARE_OPTIMAL` + `travelMode: DRIVE`** (we satisfy both — do not downgrade).

### Tolerance for endpoint failures

- **Single MDOT endpoint fails**: note it, continue. Bias toward NORMAL if DMS is the missing signal.
- **All three MDOT endpoints fail**: abort — MDOT is half the signal.
- **One Google call fails**: proceed with what you have. If BEST_GUESS succeeds but one bound fails, reason with a one-sided band. If BEST_GUESS fails but the bounds succeed, you've lost today's live-adjusted ETA — use the band midpoint as a rough proxy and lean hard on MDOT for fresh signals.
- **All three Google calls fail**: proceed with MDOT only. Strong NORMAL bias unless MDOT shows an unambiguous event.
- **Everything fails (all three MDOT + all three Google)**: abort per "Abort conditions".
- **Google 400 on field mask**: retry once with minimal mask `routes.duration,routes.staticDuration`. If still failing, treat as call failure.

## Step 4 — Reason over the data

Read everything from `/tmp/dms_*.json`, `/tmp/incidents.json`, `/tmp/construction.json`, `/tmp/google_best_guess.json`, `/tmp/google_optimistic.json`, and `/tmp/google_pessimistic.json`. Apply the reasoning below.

You are a commute intelligence assistant for Tom, who drives from Rochester Hills, MI to downtown Detroit via I-75 SB. His target arrival is 09:00; his normal departure window is 08:00–08:15 local. The routine runs at 07:45, so Call 1 (`BEST_GUESS`) is 30 min out from the anchor — firmly in Google's real-time regime.

### The signals you have

1. **Google BEST_GUESS** (call 1) — today's ETA at 08:15 anchor, live-adjusted. Includes alternatives.
2. **Google OPTIMISTIC** (call 2) — historical lower bound. "Good day" reference for this weekday + time.
3. **Google PESSIMISTIC** (call 3) — historical upper bound. "Bad day" reference.
4. **MDOT DMS sign text** — warnings like `CRASH`, `LEFT LANE CLOSED`, `RAMP CLOSED USE M-59`, `FLOODING`, `ICE` are high-signal. Normal travel-time displays are noise.
5. **MDOT incidents** — crashes, debris, flooding, disabled vehicles. Each with location, event type, lane-blocking detail.
6. **MDOT construction** — active total closures (icon 1) and lane closures (icon 2) on I-75 and feeder routes.

### Compute the derived numbers

- `today_min` = BEST_GUESS default route duration, in minutes
- `low_min` = OPTIMISTIC default route duration, in minutes
- `high_min` = PESSIMISTIC default route duration, in minutes
- `band_width_min` = `high_min - low_min`
- `where_in_band` = describe where `today_min` sits: "below the band," "near the low end," "mid band," "near the high end," "above the high end"
- `leave_by` = latest time Tom can leave the origin and still arrive by 09:00: `09:00 - today_min`, formatted as HH:MM local. If `leave_by` is before 08:00, note that Tom would need to leave earlier than his normal habit — that's a flag regardless of Google's absolute numbers.
- `expected_arrival` = `08:15 + today_min`, formatted HH:MM local
- For each alternative route in BEST_GUESS: `alt_duration_min` and `alt_savings_min = today_min - alt_duration_min`. The best alternative is the one with the highest `alt_savings_min`.

### Classify: NORMAL or ACTIONABLE

**NORMAL** — today is within the typical historical range, no unambiguous event threatening the commute, expected arrival is before 09:00 + small tolerance:

- `today_min` is within `[low_min, high_min]` (inclusive, with a few minutes of tolerance at each end)
- AND MDOT has no fresh incidents blocking SB I-75 mainline, no severe DMS warnings, no new total closures on the route
- AND `expected_arrival` is before 09:05 (09:00 target + 5 min tolerance)
- AND no alternative saves ≥10 min over the default

Persistent construction that's already priced into Google's historical band (and shows up on DMS signs as chronic `ONE LANE OPEN` or `RAMP CLOSED USE M-59 WEST` for non-exit ramps) is NORMAL — it's not a new event, just the current state of reality, and the band already reflects it.

**ACTIONABLE** — escalate when any one of these is true:

1. `today_min` is meaningfully above `high_min` — i.e., worse than even a historical bad day. Especially so with MDOT corroboration, but even a significant quantitative excursion alone is worth escalating.
2. MDOT shows a fresh crash, major closure, or severe DMS warning on SB I-75 mainline or a critical feeder (I-696, I-94, M-10) — regardless of Google's number. Google may not have priced in a just-reported event yet (leading indicator).
3. `expected_arrival` is past 09:05 (Tom will be meaningfully late).
4. An alternative route in BEST_GUESS saves ≥10 min over the default — the alternative itself is the news, even on an otherwise normal day.
5. DMS signs show active `FLOODING`, `ICE`, or severe weather warnings on the route.

**Conservative bias toward NORMAL**. When uncertain, prefer NORMAL with the numbers visible in the briefing and let Tom decide. Gratuitous escalation is the failure mode — sending a ⚠️ for a day that's actually normal trains Tom to dismiss the ⚠️ signal, which collapses the system's value within a week.

### Explicit cases

- **Uncorroborated high traffic**: `today_min` near `high_min` but MDOT is quiet, DMS signs normal, no closures → **NORMAL**. Just a slow day. The briefing can note "today's at the high end of typical" without escalating.
- **Persistent construction already in the band**: DMS shows `RAMP CLOSED USE M-59` or `ONE LANE OPEN AFTER ROSA PARKS`, MDOT confirms long-standing construction, and `today_min` is inside the band → **NORMAL**. Google's historical data already reflects this construction.
- **Leading indicator**: `today_min` is inside the band but MDOT shows a fresh crash reported in the last ~30 min → **ACTIONABLE**. Google hasn't priced it in yet. Warn Tom proactively.
- **Late-firing or test run (anchor rolled to tomorrow)**: Call 1's BEST_GUESS for tomorrow 08:15 is ~13+ hours out, so its live signal is minimal and it will land near the band midpoint. Reason over MDOT as the primary signal; note in the briefing that you're operating on tomorrow's prediction rather than today's live data. Bias strongly toward NORMAL.

### Output

Emit the token `NORMAL` or `ACTIONABLE` as the decision. Proceed to **Step 5**.

## Step 5 — Compose and send the notification

### If NORMAL: compose a daily briefing push

Goal: short, calm, declarative. Give Tom today's numbers and a leave-by time he can act on. Under ~160 characters (HA pushes wrap longer messages but shorter is better). Include:

- Leave-by time (`leave_by` formatted HH:MM)
- Expected arrival time (`expected_arrival` formatted HH:MM)
- Route indication (typically "via I-75")
- Where today falls in the band, expressed as `today_min` vs `low_min`–`high_min`
- A short zero-fluff incident note if anything minor was seen but not escalated (e.g., shoulder crash, persistent construction)

Examples (illustrative only — the specific band and duration values vary day to day, don't anchor on the numbers shown here):
- `Leave by 8:19 for 8:57 ETA via I-75. Today 38 min, band 36–48 typical. Clear corridor.`
- `Leave by 8:11 for 8:58 ETA via I-75. Today 49 min, high end of 37–50 band. Minor shoulder incident at 12 Mile, no lanes blocked.`
- `Leave by 8:16 for 8:56 ETA via I-75. Today 44 min, mid-band (39–53). Persistent Rosa Parks lane closure still posted.`
- `Leave by 8:14 for 8:53 ETA via I-75. Today 41 min, near low end of 38–52 band. Rain in forecast but not impacting travel yet.`

**No persistent notification on NORMAL runs** — the HA dashboard stays clean and reserved for escalations.

### If ACTIONABLE: compose both the escalation push and the persistent notification

**Short push** — one line, under ~160 chars, **must lead with `⚠️`** to visually distinguish from normal briefings. Include the specific cause, the quantitative framing, and a concrete action.

Examples (illustrative only — specific numbers will vary):
- `⚠️ Crash on I-75 at 12 Mile blocks 2 lanes. Today 63 min vs 37–50 band. Take M-10: leave 8:00 for 8:54. Saves 9 min.`
- `⚠️ I-696 WB→SB I-75 ramp closed plus crash at Clay. Today 58 min, outside 36–49 band. Take M-10: leave 8:05 for 8:58. Or WFH.`
- `⚠️ DMS at Crooks reads FLOODING AFTER 9 MILE. Today 67 min vs 39–52 band. Consider WFH or delay 45+ min.`
- `⚠️ Fresh crash at Rochester Rd reported 7:33 AM — Google hasn't priced it in yet (today 43 min, band 38–51). Leave by 8:00 to be safe.`

**Full briefing** (markdown, for the persistent notification):

```
# I-75 SB Commute Watch — {date}

**{one-line headline with the specific action}**

## What's happening
[2-4 bullets. Tie Google's numbers to MDOT's ground truth. Quote DMS sign text verbatim when load-bearing. If Google and MDOT disagree (e.g., fresh crash not yet in BEST_GUESS), call that out explicitly — it's why the classification matters.]

## The numbers
- Leaving at 08:15 via I-75 (default): **{today_min} min**, ETA **{expected_arrival}**
- Historical band at this departure time: **{low_min}–{high_min} min**
- Where today falls: {where_in_band}
- Alternative: **{route name}** {alt_duration_min} min, ETA {HH:MM} ({alt_savings_min} min {better|worse})

## Recommendation
[1-2 sentences. Be specific and quantitative: "leave by 7:58 to make 9:00" / "take M-10, saves 12 min" / "consider WFH — expected arrival 9:18" / "delay departure 45 min and let the flooding clear."]

## Sources
- MDOT DMS signs: N / 12 checked
- MDOT incidents in corridor: N
- MDOT closures in corridor: N
- Google Routes: BEST_GUESS + OPTIMISTIC + PESSIMISTIC at 08:15 anchor
- Data timestamp: {iso timestamp, America/Detroit}
```

### Send the mobile push (all non-gated runs — NORMAL and ACTIONABLE)

```bash
SERVICE_PATH=$(echo "$HA_NOTIFY_SERVICE" | tr '.' '/')
curl -fsS -X POST \
  -H "Authorization: Bearer $HA_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg msg "$SHORT_PUSH" --arg url "$SESSION_URL" '{title: "Commute Watch", message: $msg, data: {group: "commute-watch", url: $url, tag: "commute-watch"}}')" \
  "$HA_URL/api/services/$SERVICE_PATH"
```

The `url` deep-links the push to this Claude Code session. `tag: "commute-watch"` makes subsequent pushes replace the previous one on Tom's phone — on any given day, only the latest run's push remains visible.

### Send the persistent notification (ACTIONABLE only)

```bash
DATE=$(TZ=America/Detroit date +%Y-%m-%d)
curl -fsS -X POST \
  -H "Authorization: Bearer $HA_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg title "Commute Watch — $DATE" --arg message "$FULL_BRIEFING_MARKDOWN" --arg id "commute-watch-$DATE" '{title: $title, message: $message, notification_id: $id}')" \
  "$HA_URL/api/services/persistent_notification/create"
```

The `notification_id` is keyed by date so re-runs on the same day replace the earlier briefing. **Skip this entirely on NORMAL runs** — reserved for escalations.

## Step 6 — Exit

- **NORMAL**: after the mobile push is sent, exit. No persistent notification, no commits, no state.
- **ACTIONABLE**: after both notifications are sent, exit. No commits, no state.
- **Gated off** (no `$CALENDAR_KEYWORD` event): silent exit with zero side effects.

## Abort conditions

Abort (send a short failure push via the mobile channel, skip the persistent notification, exit) if:

- `HA_URL` is unset or points to an internal/non-routable URL
- `HA_TOKEN` is unset
- `HA_NOTIFY_SERVICE` is unset
- `GOOGLE_MAPS_API_KEY` is unset
- Required repo files are missing (`CLAUDE.md`, `docs/mdot-api-reference.md`, `docs/google-routes-api-reference.md`, `routines/commute-watch.md`)
- The Google Calendar MCP is unavailable or errors during Step 2 — fail closed, do not assume an office day
- **All three MDOT endpoints AND all three Google Routes calls fail** — you're completely blind to both signal types

Do **not** abort on:

- A single DMS sign or single MDOT endpoint failing
- One or two of the three Google Routes calls failing (degrade per Step 3d)
- Google Routes returning fewer alternative routes than expected

Failure push format:

```
Commute Watch: routine failed. Reason: {one short sentence}. See session log.
```

## Success criteria

A successful run:

- Checked the calendar and either gated off (silent) or proceeded
- Fetched MDOT data and Google Routes data (tolerating partial failures per Step 3d)
- Classified the day as NORMAL or ACTIONABLE with a conservative bias toward NORMAL
- Sent exactly one mobile push on NORMAL runs
- Sent exactly two notifications (mobile push + persistent notification) on ACTIONABLE runs
- Sent zero notifications on gated-off runs
- Left a clean working tree (v4 is fully stateless)

**Gratuitous escalation is the failure mode to avoid.** Sending an ⚠️ push for a day that's actually normal trains Tom to dismiss the signal. When uncertain, send NORMAL with the numbers on display and let Tom decide. The trust invariant outranks the coverage invariant — it just applies at the escalation boundary now, not at the notification boundary.
