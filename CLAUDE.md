# Commute Watch — Session Primer

You are working in Tom's **commute intelligence** workspace. This file is auto-loaded by Claude Code when a session starts here, which means **you are reading this and it applies to you**.

## What this directory is for

A scheduled Claude Code routine that runs each weekday morning at **07:45 America/Detroit**, checks Tom's calendar for an office day (an event named `$CALENDAR_KEYWORD` on his primary calendar), and synthesizes two real-time data sources along the I-75 SB corridor from Rochester Hills to downtown Detroit: **MDOT Mi Drive** (qualitative — DMS sign text, incident reports, closures) and the **Google Maps Routes API** (quantitative — three calls at the 08:15 anchor departure: `BEST_GUESS` for today's live-adjusted ETA with alternatives, `OPTIMISTIC` and `PESSIMISTIC` for the historical band). Claude's job is the synthesis: classify today as NORMAL or ACTIONABLE and tailor the notification accordingly.

**The daily briefing is the product.** On a normal morning the routine sends a short, calm push with today's numbers and a leave-by recommendation — data-backed, boring, useful. The routine should feel like a helpful daily ritual, not an alarm clock. Unusual days escalate visibly with a `⚠️` prefix and a specific recommendation; that's when the gratuitous-escalation-suppression judgment really earns its keep. Only gated-off (non-office) days are silent.

## How you run

You can be invoked in two modes:

1. **Interactive mode** — Tom opens a session here to edit the routine prompt, tune the reasoning, update docs, or investigate a past run. You can ask questions and propose changes normally.
2. **Routine mode** — Anthropic's cloud runs `routines/commute-watch.md` on a schedule. You cannot ask questions during a routine run. Follow the prompt exactly, bias conservative (silent over noisy), and never send a notification unless the data genuinely warrants it.

In both modes, the authoritative logic lives in `routines/commute-watch.md`. This file is just the primer pointing you there.

## Required reading

Before doing substantive work in this repo, read:

1. **`routines/commute-watch.md`** — the routine prompt. Source of truth for fetching, reasoning, and notifying. Any behavior change happens here.
2. **`docs/mdot-api-reference.md`** — the MDOT Mi Drive API reference (endpoints, sign IDs, HTML patterns, bounding box). Read this if you're touching MDOT-fetching logic.
3. **`docs/google-routes-api-reference.md`** — the Google Routes API reference (endpoint, auth, field mask, request/response shape, the three-call same-departureTime `trafficModel` pattern for today-vs-historical-band comparison, failure handling). Read this if you're touching the Google Routes logic or tuning the field mask.
4. **`memory/MEMORY.md`** — the memory index. Follow every link.

## The commute at a glance

- **Origin**: Rochester Hills area, I-75 SB on-ramp
- **Destination**: downtown Detroit (~42.33, −83.05)
- **Route**: I-75 SB, ~25–30 miles
- **Normal drive**: ~25–35 minutes in typical morning rush
- **Frequency**: ~3 days/week, driven by a `$CALENDAR_KEYWORD` event on the primary calendar
- **Bounding box for filtering**: lat 42.30–42.65, lon −83.25 to −83.03

## Notification philosophy

- **Every `$CALENDAR_KEYWORD` day produces a briefing.** v4 flipped the silent-on-ALL_CLEAR invariant. The routine is now a daily information channel: normal days get a short mobile push with today's numbers; unusual days get an escalated `⚠️` push plus a persistent notification with the full context. Only gated-off days are silent.
- **NORMAL vs ACTIONABLE**: the routine classifies every non-gated run into one of two modes. NORMAL is the default — a short "leave by X for Y ETA, today is Z in the band" push, no persistent notification. ACTIONABLE is the exception — a `⚠️`-prefixed push plus the persistent notification, triggered by MDOT corroboration of a fresh event, Google showing a reading meaningfully outside the historical `[OPTIMISTIC, PESSIMISTIC]` band, an expected arrival past 09:05, or an alternative route saving ≥10 min.
- **Conservative bias is now about escalation, not silence.** When uncertain between NORMAL and ACTIONABLE, prefer NORMAL with the numbers visible in the briefing. The trust invariant still outranks the coverage invariant; it just applies at the escalation boundary now. Gratuitously escalating a normal day into a `⚠️` push trains Tom to ignore the `⚠️` signal within a week, which is the failure mode that destroys the system's value.

## Non-negotiables

1. **Always send a mobile push on any non-gated `$CALENDAR_KEYWORD` run.** Gated-off runs are the only silent path. Every office day produces at least a calm NORMAL briefing. Silence on an `$CALENDAR_KEYWORD` day means the routine didn't run, not that "nothing was wrong."
2. **Never run on a day without a `$CALENDAR_KEYWORD` event on the primary calendar.** If the calendar check fails or the event is absent, exit silently with zero side effects. Tom is working on being better about adding these — the routine is not a safety net for his calendar hygiene.
3. **Never fabricate traffic data.** If data sources fail, note it in the reasoning and bias hard toward NORMAL. If everything fails, abort per the "Abort conditions" in the routine prompt — do not guess.
4. **Never escalate to ACTIONABLE without MDOT corroboration or a genuinely extreme quantitative reading.** The false-alarm-suppression rule still stands — it just applies at the escalation boundary now (NORMAL vs ACTIONABLE), not at the notification boundary (silent vs push). A gratuitous `⚠️` is worse than a missed edge case. When in doubt, send NORMAL with the data and let Tom decide.

## Where memories live

Per-project memories live in `memory/` **in this directory**, not in `~/.claude/projects/`. This mirrors the `ha-management` convention: git-tracked, portable, auditable. Write new memory files to `memory/` and index them in `memory/MEMORY.md`.
