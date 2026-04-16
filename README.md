# Commute Watch

Morning commute intelligence for I-75 SB from Rochester Hills to downtown Detroit. Runs as a scheduled Claude Code routine at **07:45 America/Detroit** on `$CALENDAR_KEYWORD` office days. Synthesizes MDOT Mi Drive (qualitative: DMS sign text, incidents, closures) with Google Maps Routes API (quantitative: today's live ETA plus a historical optimistic/pessimistic band via three parallel `computeRoutes` calls) and sends a daily briefing to Tom's phone.

**Daily briefing by design.** Every `$CALENDAR_KEYWORD` day produces a mobile push. Normal days get a short "leave by X for Y ETA, today is Z min in the 38–51 min typical band" briefing. Unusual days get an escalated `⚠️` push plus a persistent notification with the full context, triggered by MDOT corroboration, a reading meaningfully outside the historical band, a late-arrival flag, or a strong alternative-route save. Only non-office days are silent.

## What's in here

- `CLAUDE.md` — session primer, auto-loaded when Claude Code opens this directory
- `routines/commute-watch.md` — the self-contained routine prompt (authoritative logic)
- `routines/README.md` — routine configuration reference (env vars, schedule, MCPs, network allowlist)
- `docs/mdot-api-reference.md` — MDOT Mi Drive API reference (endpoints, sign IDs, filtering)
- `docs/google-routes-api-reference.md` — Google Routes API reference (endpoint, auth, field mask, request/response, failure handling)
- `memory/` — per-project memory index and files

## How it runs

1. Cloud-side routine config at [claude.ai/code/routines](https://claude.ai/code/routines) is minimal: environment variables (`HA_URL`, `HA_TOKEN`, `HA_NOTIFY_SERVICE`, `GOOGLE_MAPS_API_KEY`), MCP connectors, the 07:45 weekday schedule, and the one-line prompt `Follow the instructions in routines/commute-watch.md`
2. On schedule, Claude Code clones this repo, reads `routines/commute-watch.md`, and follows it
3. A calendar check gates the run — if there's no `$CALENDAR_KEYWORD` event on today's primary calendar, the routine exits silently
4. MDOT data is fetched via `curl`, pre-filtered with `jq` to the I-75 corridor. Google Routes is called three times in parallel at the same 08:15 anchor: `BEST_GUESS` (today's live-adjusted ETA with alternatives), `OPTIMISTIC` (historical lower bound), `PESSIMISTIC` (historical upper bound)
5. Claude reads everything and classifies today as **NORMAL** or **ACTIONABLE** based on where `BEST_GUESS` falls in the `[OPTIMISTIC, PESSIMISTIC]` historical band and what MDOT is showing on the route
6. Every non-gated run sends a mobile push. NORMAL runs send a calm "leave by X, today's Y, band is Z" daily briefing. ACTIONABLE runs send a `⚠️`-prefixed escalation push plus a full persistent notification with what happened, the numbers, and a specific recommendation.

## First-time setup

This repo must be a git repo pushed to a remote before the routine can clone it. From this directory:

```bash
git init
git add .
git commit -m "Initial commute-watch scaffold"
# create a remote (e.g. on GitHub) and push
```

Then configure the routine at [claude.ai/code/routines](https://claude.ai/code/routines) with the env vars and MCPs listed in `routines/README.md`.

## Architecture notes

**Two data sources, one reasoning layer.** MDOT Mi Drive supplies qualitative signals Google can't give us — DMS sign text like `RAMP CLOSED USE M-59 WEST`, incident reports with lane-blocking detail, construction zones with severity codes. Google Maps Routes API supplies quantitative signals MDOT can't give us — today's live-adjusted ETA via a `BEST_GUESS` call, the historical range via `OPTIMISTIC` and `PESSIMISTIC` calls at the same 08:15 anchor departure, plus alternative route durations via `computeAlternativeRoutes: true` on the BEST_GUESS call. Three Google calls in parallel. Claude synthesizes all of it.

**The magic is gratuitous-escalation suppression.** A dumb aggregator over Google's numbers would fire a `⚠️` push every day that's slightly above the median, and within a week Tom would ignore the `⚠️` signal. Claude's specific job is to decide whether to send a calm daily briefing or to escalate — based on whether MDOT corroborates a real event, whether today's reading is meaningfully outside the historical band, whether an alternative saves real time, and whether the expected arrival is past Tom's 9:00 target. If Google says "today 48 min, band 38–51" and MDOT shows no incidents, the routine sends a calm NORMAL briefing saying "today's at the high end of typical, nothing wrong." Not an escalation.

**The evolution, for posterity:**
- **v1** — stateless, silent-on-ALL_CLEAR, MDOT-only
- **v2** — added a hand-rolled baseline learning loop (committed state/baseline.json to main). Tom correctly flagged it as reinventing Google's historical data badly
- **v3** — replaced the baseline loop with a `departureTime=+7d` hack for historical comparison via Google Routes, but stayed silent-on-ALL_CLEAR
- **v4** — replaced the `+7d` hack with the proper `trafficModel` parameter (`BEST_GUESS`/`OPTIMISTIC`/`PESSIMISTIC` at the same anchor departureTime), flipped silent-on-ALL_CLEAR to a daily-briefing model, and shifted the run time from 06:45 to 07:45 to catch rush-hour developments in Google's real-time window

Still out of scope: NB return-trip run, camera snapshots in notifications, deeper calendar integration (e.g., WFH event types), weather API integration, adaptive anchor departure time. Candidates for v5+.
