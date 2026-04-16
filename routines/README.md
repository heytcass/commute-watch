# Routines

Self-contained prompt files for Claude Code Routines that run this repo as their working directory.

## Files

- **`commute-watch.md`** — weekday morning commute intelligence for I-75 SB from Rochester Hills to downtown Detroit. Gates on a `$CALENDAR_KEYWORD` event in the primary calendar, fetches MDOT Mi Drive data (DMS sign text, incidents, closures) and Google Routes API data (today's ETA, typical baseline, alternatives), synthesizes both signals, and notifies only if actionable.

## How routines are configured

Routines are configured at [claude.ai/code/routines](https://claude.ai/code/routines). The cloud-side prompt should be minimal and just point at the prompt file in this repo:

> Follow the instructions in `routines/commute-watch.md`.

This keeps the real logic in git, versioned and visible, while the cloud-side config stays stable.

## Required environment variables

Set these in the routine's cloud environment on claude.ai/code:

| Variable | Value | Purpose |
|---|---|---|
| `HA_URL` | Your HA external URL (e.g., `https://ha.example.com`) | External HA URL. Internal/local URLs are not reachable from Anthropic's cloud. |
| `HA_TOKEN` | Long-lived HA access token | Auth for `persistent_notification/create` and `notify.mobile_app_*` REST calls |
| `HA_NOTIFY_SERVICE` | Your HA companion app notify service (e.g., `notify.mobile_app_your_phone`) | Target for the short push notification |
| `GOOGLE_MAPS_API_KEY` | Google Maps Platform API key with Routes API enabled | Auth header for `computeRoutes` calls. Restrict the key in the GMP console to the Routes API only. |
| `COMMUTE_ORIGIN` | Home address (e.g., `123 Main St, Rochester Hills, MI 48309`) | Passed to Google Routes as the route origin. Not committed to the repo. |
| `COMMUTE_DESTINATION` | Office address (e.g., `456 Larned St, Detroit, MI 48226`) | Passed to Google Routes as the route destination. Not committed to the repo. |
| `CALENDAR_KEYWORD` | Keyword in calendar event summaries that signals an office day (e.g., `@Office`) | The routine searches today's primary calendar events for this string. No match = gated off (silent). |

## Required MCP connectors

Enable these cloud MCPs on the routine config at claude.ai/code:

- **Google Calendar** — for the office-day gate (matching `$CALENDAR_KEYWORD` against event summaries). Must have read access to Tom's primary calendar. No write access needed.

HA notifications go via `curl` and the REST API, **not** the HA MCP. The HA MCP's tools are all domain-specific (lights, climate, media, broadcast, lists) and do not expose a generic `notify` service or `persistent_notification.create`. `HassBroadcast` is TTS, not a push notification. The ha-management routine uses the same `curl` pattern for the same reason.

## Suggested schedule

Weekdays at **7:45 AM America/Detroit**. This is 30 min before Tom's latest normal departure (08:15), which puts Google's BEST_GUESS call firmly in its real-time-weighted regime — the prediction for 08:15 is heavily informed by what traffic actually looks like at 7:45. That's enough lead time for Tom to eat breakfast faster, leave earlier, take an alternate, or decide to WFH without compressing his morning into chaos.

v3 ran at 06:45, but that was too early: Google's BEST_GUESS for 08:15 at 06:45 was 1.5h out and Google's live-data weighting at that distance was only partial. Worse, events that developed between 06:45 and 08:15 (the rush-hour ramp-up, when bad things tend to happen) were invisible to the routine. v4 shifted later to capture those.

## Network allowlist

Restrict outbound network access to exactly:

- `your-ha-instance.example.com` — notifications
- `mdotjboss.state.mi.us` — MDOT Mi Drive APIs
- `routes.googleapis.com` — Google Maps Routes API

No other hosts. The routine has no reason to reach anything else.

## Hardcoded configuration

Some config lives in the routine prompt itself rather than as env vars, because it's stable enough not to justify cloud-side config churn:

- **Origin / destination addresses**: moved to env vars (`COMMUTE_ORIGIN`, `COMMUTE_DESTINATION`) so they're not committed to the repo
- **Anchor departure time**: `08:15` America/Detroit (latest time Tom normally leaves)
- **Target arrival time**: `09:00` America/Detroit

To change any of these, edit the "Configuration" block at the top of `routines/commute-watch.md` and commit. The next routine run picks up the change automatically — no redeploy needed.

## State and notification modes

**v4 is fully stateless.** No baseline file, no last-run tracking, no git commits from the routine. Each run is check-calendar → fetch-both-sources → reason → classify → notify → exit.

An earlier v2 maintained a hand-rolled baseline learning loop. v3 replaced that with a `departureTime=+7d` hack for historical comparison via Google Routes. v4 replaced the `+7d` hack with the proper `trafficModel` parameter (BEST_GUESS / OPTIMISTIC / PESSIMISTIC at the same anchor departure time), which is Google's documented surface for "today's live-adjusted ETA" vs "historical good/bad day references."

v4 also flipped the silent-on-ALL_CLEAR invariant from v3. The routine is now a daily information channel — every `$CALENDAR_KEYWORD` day produces at least a NORMAL mobile push with today's numbers, and ACTIONABLE days escalate to `⚠️` push + persistent notification.

| Run outcome | Mobile push | Persistent notification | State | Commit |
|---|---|---|---|---|
| Gated off (no `$CALENDAR_KEYWORD` event) | none | none | n/a | none |
| NORMAL | calm daily briefing | none | none | none |
| ACTIONABLE | `⚠️` escalation with specific action | full markdown briefing | none | none |

## Safety

Routines cannot ask the user questions during a run. The `commute-watch.md` prompt includes explicit safety rails:

- **Conservative bias toward NORMAL** — when the classification is ambiguous, send a calm NORMAL briefing with the numbers on display and let Tom decide. Trust outranks coverage.
- **Gratuitous-escalation suppression is the primary value** — Google alone would flag every slow morning with a `⚠️`; Claude's job is to only escalate when MDOT corroborates a real event or the quantitative reading is extreme.
- **Fail closed on calendar errors** — missing MCP or failed query aborts the run, does not assume office day
- **Abort on missing env vars** — better to fail loudly than send notifications to nowhere
- **Graceful degradation on partial data loss** — a single DMS sign, single MDOT endpoint, or one or two of the three Google Routes calls failing is tolerable. Only total MDOT + Google failure aborts.

**A calm NORMAL push is the default successful run.** Silence is reserved for gated-off days. The failure mode to avoid is gratuitous `⚠️` escalation on a day that's actually normal — that's what trains Tom to dismiss the signal, and once that happens the system's value collapses.
