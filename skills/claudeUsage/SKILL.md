# claudeUsage

<!-- GUILDOS-SKILL name="claudeUsage" -->

<!-- GUILDOS-DESC -->
Read Claude Code SUBSCRIPTION usage (rolling 5h + weekly rate-limit windows) and decide whether to hold/pace work. Authoritative source = Anthropic's own rate_limit_event; token sums are a calibrated fallback. Actions: quickCheck (sub-second HOLD/OK line), checkUsage (the caravan pacing gate), readRateLimit (the raw authoritative verdict), plus ad-hoc usageWindows / categorizeUsage analysis. Use for caravan dispatch pacing or any 'am I rate-limited / how much Claude quota is left' question.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/claudeUsage/SKILL.md -->
```markdown
---
name: claudeUsage
description: >-
  Read Claude Code SUBSCRIPTION usage (rolling 5h + weekly rate-limit windows)
  and decide whether to hold/pace work. Authoritative source = Anthropic's own
  rate_limit_event; token sums are a calibrated fallback. Actions: quickCheck
  (sub-second HOLD/OK line), checkUsage (the caravan pacing gate), readRateLimit
  (the raw authoritative verdict), plus ad-hoc usageWindows / categorizeUsage
  analysis. Use for caravan dispatch pacing or any 'am I rate-limited / how much
  Claude quota is left' question.
---

# Claude Usage

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/claudeUsage/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto claudeUsage <action>` (or read the source directly).

## Actions

- **categorizeUsage** — Ad-hoc analysis: classify local Claude token spend into develop / restart-scripts / analyze-decide / do-work buckets by a per-turn tool-use heuristic. Answers 'what is my Claude budget actually being spent on'.
- **checkUsage** — The Claude usage pacing GATE: how much of the 5h and weekly subscription rate limit is used, and should work HOLD. Returns {over, signal, rateLimit, fiveHour, week, reason}. AUTHORITATIVE-first (Anthropic's own rate_limit_event status/utilization), calibrated token estimate as fallback. What the caravan calls before dispatching a Claude leaf.
- **quickCheck** — Fastest check of current Claude usage — how much of the 5h and weekly subscription rate limit is used right now, and whether to pace/hold. One line (HOLD/OK + 5h status + weekly % + signal), sub-second via a script. Answers 'am I rate-limited / how much Claude quota is left'.
- **readRateLimit** — Am I rate-limited right now? Read Anthropic's own authoritative Claude subscription rate limit verdict — five_hour + weekly (seven_day) {status, resetsAt, utilization} — from the leaf stream-json logs. Ground truth, no token-ceiling guessing. Use when you want the real limit status, not an estimate.
- **usageWindows** — Ad-hoc analysis (NOT the gate): break local Claude token usage into the last N consecutive 5h windows, to see whether pacing is bending the consumption curve over time.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/claudeUsage/index.js -->
```js
/**
 * claudeUsage skill book — WHEN/HOW to read Claude Code SUBSCRIPTION usage (the
 * rolling 5h + weekly rate-limit windows) and decide whether to pace/hold work.
 *
 * PURE TEXT REGISTRY: prompt strings only, no imports/functions. All executable
 * logic lives in the paired weapon (libs/weapon/claudeUsage/index.js) and its CLI
 * (cli.mjs). This book is the discovery + judgment layer; the weapon's function
 * JSDoc is the API, and libs/weapon/claudeUsage/engineering_decisions.md is the WHY.
 *
 * THE ONE THING TO KNOW: the AUTHORITATIVE signal is Anthropic's own
 * `rate_limit_event` (status / utilization), emitted by `claude --output-format
 * stream-json` into the leaf logs — NOT a token count. The token sums are a
 * cache_read-excluded FALLBACK/telemetry, calibrated to the real /usage page.
 */

export const skillBook = {
  id: "claudeUsage",
  title: "Claude Usage",
  description:
    "Read Claude Code SUBSCRIPTION usage (rolling 5h + weekly rate-limit windows) and decide whether to hold/pace work. Authoritative source = Anthropic's own rate_limit_event; token sums are a calibrated fallback. Use for caravan dispatch pacing or any 'how much of my Claude limit is left / am I rate-limited' question.",
  steps: [],
  toc: {
    quickCheck: {
      description:
        "Fastest check of current Claude usage — how much of the 5h and weekly subscription rate limit is used right now, and whether to pace/hold. One line (HOLD/OK + 5h status + weekly % + signal), sub-second via a script. Answers 'am I rate-limited / how much Claude quota is left'.",
      howTo:
        "Run `node libs/weapon/claudeUsage/cli.mjs --brief` (~0.5s). Output e.g. `OK  5h=allowed  week~11.1%  (rate_limit)`. Exits non-zero when over, so it composes in shell (`node …/cli.mjs --brief && echo clear`). USE PLAIN `node`, NOT `npx tsx` — tsx adds ~2.5s of runtime resolution for zero benefit here (a harmless MODULE_TYPELESS warning on stderr is expected). For the full JSON drop `--brief`; for just the authoritative status use `--ratelimit`.",
      input: {},
      output: { line: "string — e.g. 'OK  5h=allowed  week~11.1%  (rate_limit)'" },
    },

    checkUsage: {
      description:
        "The Claude usage pacing GATE: how much of the 5h and weekly subscription rate limit is used, and should work HOLD. Returns {over, signal, rateLimit, fiveHour, week, reason}. AUTHORITATIVE-first (Anthropic's own rate_limit_event status/utilization), calibrated token estimate as fallback. What the caravan calls before dispatching a Claude leaf.",
      howTo: [
        "PRIMARY PATH: call weapon claudeUsage.checkUsage({ thresholdPct? }) in-process (or `node libs/weapon/claudeUsage/cli.mjs` for JSON). Read `over` (should this tick HOLD) and `reason` (human summary); `signal` is 'rate_limit' (authoritative event used) or 'token_fallback' (no valid event, token estimate used).",
        "HOW THE GATE DECIDES, per window: 5h HOLDs when the authoritative five_hour status is 'rejected' (or, absent a valid event, the 5h token estimate ≥ thresholdPct, default 70). Weekly HOLDs ONLY on a fresh seven_day event ('rejected', or utilization ≥ CARAVAN_WEEK_UTIL_HOLD_PCT/100, default 0.90). The weekly window NEVER holds on a token sum — a rolling-7d sum spans the weekly reset and false-reads high (see ED).",
        "VALIDITY: an authoritative event only counts while its window is OPEN (resetsAt > now). A re-surfaced event from an expired window (an old log re-read, or an event echoed into a transcript) has a PAST resetsAt and is correctly ignored — never trust a rate_limit_event by file mtime alone.",
        "The weekly token % in the result is TELEMETRY, calibrated to the operator's real /usage page (billable-since-reset ÷ ceiling). It tracks reality but is not the gate for the weekly window.",
        "Fail-open: on a monitor error the result is over:false with signal:'error' — a broken monitor must never silently halt all dispatch.",
      ].join("\n"),
      input: { thresholdPct: "int 1–100, default 70 — 5h token-fallback hold threshold" },
      output: {
        over: "boolean — should this tick HOLD",
        signal: "'rate_limit' | 'token_fallback' | 'error'",
        rateLimit: "object — {fiveHour, sevenDay, source, ageMs}",
        fiveHour: "object — {tokens, ceiling, pct}",
        week: "object — {tokens, ceiling, pct, windowStart, anchor}",
        reason: "string — one-line human summary",
      },
    },

    readRateLimit: {
      description:
        "Am I rate-limited right now? Read Anthropic's own authoritative Claude subscription rate limit verdict — five_hour + weekly (seven_day) {status, resetsAt, utilization} — from the leaf stream-json logs. Ground truth, no token-ceiling guessing. Use when you want the real limit status, not an estimate.",
      howTo: [
        "Call weapon claudeUsage.readRateLimit({ maxAgeMs?, workerLogsDir? }) (or `node libs/weapon/claudeUsage/cli.mjs --ratelimit`, ~0.25s). Returns the most-recent event per type, chosen by (file mtime, then resetsAt).",
        "five_hour carries only a binary status ('allowed' | 'rejected') + resetsAt — no percentage. seven_day carries status ('allowed_warning' | 'rejected') + a REAL utilization fraction + resetsAt.",
        "BLIND SPOT (important): Anthropic emits seven_day events ONLY at surpassedThreshold 0.75 (warning) or rejection — never a plain 'allowed' below 75%. So below 75% weekly there is NO authoritative weekly %, and sevenDay comes back null. Absence of any warning since a reset therefore MEANS weekly < 75% — use checkUsage's calibrated token telemetry (or the /usage page) for the exact number.",
        "Source is 'worker-logs' (leaf stream-json, the reliable native source) or 'transcripts' (fallback). Always validity-check resetsAt > now before treating a status as current.",
      ].join("\n"),
      input: { maxAgeMs: "int — freshness window, default 6h", workerLogsDir: "string — default <cwd>/.worker-logs" },
      output: {
        fiveHour: "object|null — {status, resetsAt}",
        sevenDay: "object|null — {status, utilization, resetsAt}",
        source: "'worker-logs' | 'transcripts' | 'none'",
        ageMs: "int|null — age of the latest event",
      },
    },

    usageWindows: {
      description:
        "Ad-hoc analysis (NOT the gate): break local Claude token usage into the last N consecutive 5h windows, to see whether pacing is bending the consumption curve over time.",
      howTo:
        "Call weapon claudeUsage.usageWindows({ count?, windowHours? }) (or `node libs/weapon/claudeUsage/cli.mjs --windows [N]`). Returns per-window token totals (window 0 = most recent). NOTE these totals include cache_read (raw 'consumed' tokens), so they are much larger than the gate's billable numbers — this view is for trend-spotting, not for comparing to a limit.",
      input: { count: "int, default 6", windowHours: "int, default 5" },
      output: { windows: "array of {index, hoursAgoStart, hoursAgoEnd, tokens}" },
    },

    categorizeUsage: {
      description:
        "Ad-hoc analysis: classify local Claude token spend into develop / restart-scripts / analyze-decide / do-work buckets by a per-turn tool-use heuristic. Answers 'what is my Claude budget actually being spent on'.",
      howTo:
        "Call weapon claudeUsage.categorizeUsage({ hoursBack? }) (or `node libs/weapon/claudeUsage/cli.mjs --categorize [hoursBack]`). Reports each bucket BOTH by total tokens (cache-read-inclusive, 'consumed') and by output tokens (active-generation 'effort'). The restart-vs-work split is inherently fuzzy (both are exec calls) — treat those two as a blend, not a hard line.",
      input: { hoursBack: "int, default 30" },
      output: { categories: "object — {develop, restart, analyze, work} each {tokens, pctTotal, outTokens, pctOut, turns}" },
    },
  },
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/claudeUsage/cli.mjs -->
```js
#!/usr/bin/env node
/**
 * CLI: print the local Claude token-usage estimate (5h + 7d windows) vs the
 * configured ceilings/threshold. Reads CARAVAN_TOKENS_5H_CEILING,
 * CARAVAN_TOKENS_WEEK_CEILING, CARAVAN_USAGE_THRESHOLD_PCT from env.
 *
 * QUICK: plain `node` runs it in ~0.6s (no tsx startup) — this is the fast path:
 *   node libs/weapon/claudeUsage/cli.mjs            # full JSON
 *   node libs/weapon/claudeUsage/cli.mjs --brief    # one-line glance (~0.6s)
 *   node libs/weapon/claudeUsage/cli.mjs --ratelimit # authoritative status (~0.25s)
 * (`npx tsx …` also works but adds ~2.5s of runtime resolution; node reparses the
 * ESM-syntax .js with a harmless MODULE_TYPELESS warning on stderr.)
 * Dynamic import (not a static `import {}`) so the .mjs entry doesn't choke on the
 * .js module's interop during the instantiation phase.
 */
const m = await import("./index.js");
const mod = m.checkUsage ? m : m.default;
const args = process.argv.slice(2);

// `--windows [N]`: per-5h-window token breakdown (ad-hoc analysis).
if (args[0] === "--windows") {
  const count = Number(args[1]) || 6;
  const r = mod.usageWindows({ count });
  if (r.ok) {
    console.log(`Token usage by ${r.windowHours}h window (window 0 = most recent), ${r.files} files / ${r.lines} usage records:`);
    for (const w of r.windows) {
      console.log(`  win ${w.index} (${w.hoursAgoStart}h–${w.hoursAgoEnd}h ago): ${w.tokens.toLocaleString()} tokens`);
    }
  } else {
    console.log(JSON.stringify(r, null, 2));
  }
  process.exit(0);
}

// `--categorize [hoursBack]`: token spend by work type (develop/restart/work/analyze).
if (args[0] === "--categorize") {
  const hoursBack = Number(args[1]) || 30;
  const r = mod.categorizeUsage({ hoursBack });
  if (r.ok) {
    const LABEL = { develop: "1. Develop scripts/tools", restart: "2. Start/restart scripts", analyze: "3. Analyze & decide", work: "4. Actually do work" };
    console.log(`Token spend by work type — last ${r.hoursBack}h, ${r.files} files / ${r.turns} assistant turns:`);
    console.log(`(% by total tokens [consumed] | % by output tokens [active effort])`);
    for (const k of ["develop", "restart", "analyze", "work"]) {
      const c = r.categories[k];
      console.log(`  ${LABEL[k].padEnd(26)} ${String(c.pctTotal).padStart(5)}% | ${String(c.pctOut).padStart(5)}%   (${c.tokens.toLocaleString()} tok, ${c.turns} turns)`);
    }
  } else {
    console.log(JSON.stringify(r, null, 2));
  }
  process.exit(0);
}

// `--ratelimit`: print just the authoritative rate_limit_event reading.
if (args[0] === "--ratelimit") {
  console.log(JSON.stringify(mod.readRateLimit(), null, 2));
  process.exit(0);
}

// `--brief`: one-line human glance — 5h + weekly + hold verdict.
if (args[0] === "--brief") {
  const r = mod.checkUsage();
  const five = r.rateLimit?.fiveHour?.status
    ? `5h=${r.rateLimit.fiveHour.status}`
    : `5h~${r.fiveHour.pct}%`;
  const week = r.rateLimit?.sevenDay?.status
    ? `week=${r.rateLimit.sevenDay.status}${typeof r.rateLimit.sevenDay.utilization === "number" ? ` ${Math.round(r.rateLimit.sevenDay.utilization * 1000) / 10}%` : ""}`
    : `week~${r.week.pct}%`;
  console.log(`${r.over ? "HOLD" : "OK"}  ${five}  ${week}  (${r.signal})`);
  process.exit(r.over ? 1 : 0);
}

const r = mod.checkUsage();
console.log(JSON.stringify(r, null, 2));
process.exit(r.over ? 1 : 0);
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/claudeUsage/engineering_decisions.md -->
```markdown
# Engineering Decisions — claudeUsage weapon

### 2026-07-01 · Weekly token % is anchored to the real reset boundary; ceiling re-calibrated to `/usage` (was ~9× low)

**Decision.** The weekly token telemetry now (a) counts billable tokens since the
CURRENT week's reset boundary — not a rolling 7d — and (b) divides by a 366M ceiling
calibrated to the operator's real `/usage` reading. `DEFAULT_WEEK_CEILING` 40M → 366M.

**Why.** The weekly limit resets on a FIXED weekly cadence (observed:
Mon 07:00 America/LA, epoch seed `1782741600`). A naive rolling-7d token sum spans
that boundary and keeps counting pre-reset tokens Anthropic already cleared, so it
read ~124% at a real ~11%. Two fixes: anchor the window to the most recent boundary
(`seed ± k·7d`, or a live seven_day `resetsAt` when one is in scan range — self-updating);
and fix the ceiling. The old 40M was mis-anchored on a PRE-reset event whose
`status:"rejected"` was `overageDisabledReason:"out_of_credits"` — an OVERAGE-CREDIT
exhaustion at the end of a prior budget cycle, NOT the base weekly cap — and it
undercounted that cycle. The trustworthy anchor is the live `/usage` page: 40.5M
billable since the 2026-06-29 reset read ~11% there → cap ≈ 366M billable.

**Scope / why safe.** The weekly % is TELEMETRY ONLY — the weekly window never gates
on the token sum (it gates on the authoritative seven_day event; see below). So this
only makes the reported number match reality; it changes no hold behaviour. Verified:
weekly now reads 11.1% (was 124%), windowStart = the Mon-07:00 boundary.

**Note on the authoritative weekly signal's blind spot.** Anthropic emits `seven_day`
events only at `surpassedThreshold: 0.75` (warning) or rejection — never a plain
"allowed" below 75%. So below 75% there is NO authoritative weekly %; the token
estimate is the only weekly number available, which is exactly why it must track
`/usage`. Absence of any seven_day warning since a reset authoritatively means weekly < 75%.

**Consequence if violated.** Reverting to a rolling-7d weekly sum re-introduces the
post-reset over-count; mis-calibrating the ceiling makes the reported weekly % diverge
from `/usage` again (re-tune the ceiling against `/usage`, don't anchor on an
overage-credit rejection).

### 2026-06-29 · The `rate_limit_event` IS the authoritative read; token sums are fallback-only (cache_read EXCLUDED)

**Decision.** Drive the caravan usage gate from Anthropic's OWN per-account
`rate_limit_event` — emitted on every turn by `claude --output-format stream-json`
(which every caravan leaf runs) and written natively to the leaf logs
(`.worker-logs/*.log`). Shape: `{"type":"rate_limit_event","rate_limit_info":{
status:"allowed"|"allowed_warning"|"rejected", resetsAt:<epoch-sec>,
rateLimitType:"five_hour"|"seven_day", utilization?:<0–1> }}`. `checkUsage` reads the
latest event per type (`readRateLimit`), and gates **per-window with the best signal**:
- **5h** — `five_hour` carries only a binary `status` (no `utilization`): `rejected`
  → HOLD. When no *valid* five_hour event exists, fall back to the token estimate.
- **7d** — `seven_day` carries `status` AND a real `utilization` fraction:
  `rejected` OR `utilization ≥ CARAVAN_WEEK_UTIL_HOLD_PCT/100` (default 0.90) → HOLD.
  The weekly window NEVER falls back to a token sum (see below).

**This SUPERSEDES the 2026-06-23 decision below** ("do NOT try to read the real
subscription %") — that decision was correct *given what was known then* (no
programmatic read existed), but the `rate_limit_event` in the stream-json output IS
that read. It was hiding in the leaf logs the whole time.

**Why the old token-sum gate broke (the bug this fixes).** `lineTokens` summed
`cache_read_input_tokens` at FULL weight against a guessed 100M ceiling. At heavy
continuous dispatch cache_read is ~95% of the raw total (the same large cached prompt
prefix re-read every turn across many parallel leaves), so a calm-real 5h of single
digits read as **73.7%, and the peak window as 140%** — wrongly HOLDing ALL dispatch
overnight (operator incident 2026-06-29). Measured: peak 5h ALL=139.9M vs
billable(excl cache_read)=6.4M — a ~22× inflation.

**Two non-obvious guards (a naive implementation gets these wrong):**
1. **Validity by `resetsAt`, not file mtime.** An event is the verdict for its window
   only while that window is OPEN (`resetsAt > now`). A re-surfaced event from an
   expired window — an old log re-read into a freshly-touched file, or echoed into a
   transcript by an agent investigating — carries a PAST `resetsAt`; honoring it by
   mtime alone would produce a stale hold. Filter on `resetsAt > now`.
2. **The 7d window never gates on a token sum.** A rolling-7d token sum spans weekly
   reset boundaries: right after a reset it keeps counting pre-reset tokens Anthropic
   no longer charges to the current week, reading false-high (observed 86% minutes
   after a real reset). So weekly HOLD comes ONLY from a fresh `seven_day` event;
   absent one, the weekly is left to the five_hour-reject exhaustion backstop. The 7d
   token number is reported as telemetry only.

**Fallback calibration (token estimate = `billableTokens` = input+output+cache_creation,
cache_read EXCLUDED).** `DEFAULT_5H_CEILING`=15M (peak observed billable 6.4M → ~43%,
consistent with the `five_hour`="allowed" seen at that peak). `DEFAULT_WEEK_CEILING`=40M
is REJECTION-ANCHORED: 7d billable hit ~34M over a window whose `seven_day` reached
utilization 0.98 then "rejected", so ~40M ≈ the real weekly cap. These are FALLBACK
ceilings only; the authoritative signal needs no ceiling.

**Verification.** Same machine that read 73.7%/over=true now reads 5h≈10% / over=false
(matches the real ~7% within margin) with `five_hour`="allowed"; crafted-fixture tests
confirm `rejected`→HOLD, `utilization 0.95`→HOLD, stale-event→ignored. Gate re-enabled
(`WORKER_PACING.claude.active=true` in `libs/caravan/index.js`).

**Consequence if violated.** Reverting to cache_read-inclusive sums, or gating the
weekly on a rolling token sum, reinstates the false-hold that halted all dispatch.
Trusting an event by mtime without the `resetsAt > now` check lets an expired-window
verdict (or an investigator's transcript echo) wrongly hold or release the gate.

---

### 2026-06-23 · Self-account local token usage; do NOT try to read the real subscription 5h/weekly % · SUPERSEDED 2026-06-29 (see above)

**Decision.** Estimate Claude usage by summing the per-message `usage` token counts
Claude Code already writes to the per-session transcripts under `~/.claude/projects`,
bucketed into rolling 5h and 7d windows, and compare to an operator-configured token
**ceiling** (env `CARAVAN_TOKENS_5H_CEILING` / `CARAVAN_TOKENS_WEEK_CEILING`). The
gate trips at `CARAVAN_USAGE_THRESHOLD_PCT` (default 70).

**Rejected alternatives.**
- *Read the true subscription limit %* via a `claude` subcommand, an HTTP API, or a
  local cache file — **none exist** (verified 2026-06-23). `/usage` is interactive-TUI
  only; the Rate-Limits/Usage APIs are API-KEY BILLING, a different thing from the
  Pro/Max **subscription** 5h+weekly caps; no local file stores the ceiling/reset.
- *Scrape the claude.ai / Claude Code usage webpage via CIC* — accurate but expensive
  (a browser round-trip every tick), and on Carbon there is no captured browser
  profile, so it is not even available there. Going this route would have forced the
  cron tick from 5 min → 20 min. Self-accounting is cheap, so the 5-min tick stays.

**Why it is a proxy, not the truth.** The ceiling is an operator-tuned stand-in for
the plan limit (the real number is account-side and drifts). Calibrate it from the
logged actuals (`node`/`tsx` `libs/weapon/claudeUsage/cli.mjs` prints the live
numbers). `cache_read_input_tokens` ARE included in the sum on purpose → it
over-counts vs billed tokens, i.e. the brake errs toward throttling EARLY (safe).

**Consequence if violated.** A future agent who "simplifies" this to call an Anthropic
usage/rate-limit API will read API-key billing numbers (wrong population) or get
nothing for a subscription account — silently disabling the brake. And anyone who
treats the ceiling as the real limit will mis-set the threshold; it must be calibrated
from observed usage, per-machine (limits are per-ACCOUNT, so each box gets a share).

**Fail-open.** A monitor error returns `over:false` (never halts all dispatch on a
read glitch); the claude-heavy cap still applies as the independent second brake.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/claudeUsage/index.js -->
```js
/**
 * claudeUsage weapon — local, on-demand read of Claude Code SUBSCRIPTION usage in
 * the rolling 5-hour and 7-day windows, for PACING the caravan dispatch against the
 * Pro/Max rate limits.
 *
 * TWO SIGNALS, AUTHORITATIVE-FIRST (non-native — see engineering_decisions.md):
 *
 *   1. AUTHORITATIVE — `rate_limit_event` (primary gate). `claude --output-format
 *      stream-json` (which every caravan leaf runs) emits, on each turn, a
 *      `{"type":"rate_limit_event","rate_limit_info":{ status, resetsAt,
 *      rateLimitType, utilization? }}` line carrying Anthropic's OWN per-account
 *      verdict. `five_hour` gives a binary status ("allowed"|"rejected"); `seven_day`
 *      gives status PLUS a real `utilization` fraction (0–1). These land natively in
 *      the leaf stream-json logs (`.worker-logs/*.log`). This is the true limit — no
 *      ceiling guessing — so when present-and-fresh it drives the gate directly.
 *
 *   2. FALLBACK — token self-accounting (only when no fresh rate_limit_event exists,
 *      e.g. a box that hasn't dispatched a stream-json leaf recently). Sum the
 *      per-message `usage` BILLABLE tokens (input + output + cache_creation) Claude
 *      Code writes to `~/.claude/projects`, in rolling windows, vs a CALIBRATED
 *      ceiling. `cache_read_input_tokens` is EXCLUDED — at heavy continuous dispatch
 *      it is ~95% of the raw total (the same large cached prefix re-read every turn
 *      across many leaves), and including it made the old gate read ~140% of a 100M
 *      ceiling when the real subscription 5h was single digits → it wrongly HELD all
 *      dispatch. The fallback is still a proxy; the authoritative signal is the truth.
 *
 * This counts ALL Claude usage on the local machine (operator session + every leaf),
 * which is exactly the right signal for "how much of THIS account's Claude budget is
 * spent" (limits are per-ACCOUNT). With several machines on one account, the
 * authoritative rate_limit_event is account-wide and identical everywhere; only the
 * token fallback is per-machine.
 */

import { readdirSync, statSync, openSync, readSync, closeSync } from "fs";
import { join } from "path";
import { homedir } from "os";

export const toc = {
  checkUsage:
    "Read Claude SUBSCRIPTION usage for the rolling 5h + 7d windows — AUTHORITATIVE rate_limit_event status (from leaf stream-json logs) first, billable-token estimate as fallback; returns {over, fiveHour, week, rateLimit}. Used to pace caravan dispatch.",
  readRateLimit:
    "Read Anthropic's own per-account rate_limit_event (status/resetsAt/utilization for five_hour + seven_day) from the latest leaf stream-json logs — the authoritative subscription-limit signal, no token-ceiling guessing.",
  usageWindows:
    "Break local Claude token usage into the last N consecutive 5h windows (ad-hoc analysis: is pacing reducing consumption over time?).",
  categorizeUsage:
    "Classify local Claude token spend into develop / restart-scripts / analyze-decide / do-work buckets by per-turn tool-use heuristic (fuzzy: 'start a script' vs 'work' blur).",
};

const HOUR_MS = 3600 * 1000;
const DAY_MS = 24 * HOUR_MS;
const WEEK_MS = 7 * DAY_MS;
/** Observed weekly-reset boundary (epoch ms): Mon 2026-06-29 07:00 America/LA. The
 *  weekly cap resets on this fixed 7-day cadence, so ANY boundary = seed ± k·7d.
 *  Used to align the weekly token window to the current week when no live seven_day
 *  event is in scan range; a live event's resetsAt supersedes it. */
const WEEKLY_ANCHOR_SEED_MS = 1782741600 * 1000;

/** Env-driven ceilings — BILLABLE-token (cache_read-excluded) proxies for the plan
 *  limit, mapping the token sums to a %. Calibrated from logged actuals:
 *   - 5h = 15M: peak observed 5h billable 6.4M while `five_hour` stayed "allowed"
 *     → ~43% (consistent with allowed). Used as the gate FALLBACK when no valid
 *     five_hour event exists (else the authoritative status gates).
 *   - week = 366M: anchored to the operator's real `/usage` weekly reading
 *     (2026-07-01: 40.5M billable since the reset read ~11% there → 366M). The
 *     earlier 40M was mis-anchored on a pre-reset OVERAGE-credit exhaustion, not the
 *     base weekly cap, and read ~124% at a real ~11%. The weekly % is TELEMETRY only
 *     (the weekly window never gates on the token sum — it gates on the authoritative
 *     seven_day event), so this ceiling just makes the reported number track reality.
 *  Re-tune from `cli.mjs` against `/usage` if they drift. */
export const DEFAULT_5H_CEILING = Number(process.env.CARAVAN_TOKENS_5H_CEILING) || 15_000_000;
export const DEFAULT_WEEK_CEILING = Number(process.env.CARAVAN_TOKENS_WEEK_CEILING) || 366_000_000;
export const DEFAULT_THRESHOLD_PCT = Number(process.env.CARAVAN_USAGE_THRESHOLD_PCT) || 70;
/** seven_day `utilization` fraction at/above which the authoritative weekly signal
 *  HOLDs (real %, not a proxy). 0.90 leaves headroom for the operator's interactive
 *  session below the hard reject. Env-tunable. */
export const DEFAULT_WEEK_UTIL_HOLD = Number(process.env.CARAVAN_WEEK_UTIL_HOLD_PCT) / 100 || 0.9;
/** A rate_limit_event older than this is "stale" → fall back to the token estimate.
 *  6h > the 5h window, so a still-current 5h verdict is never treated as stale. */
const RATE_LIMIT_MAX_AGE_MS = Number(process.env.CARAVAN_RATE_LIMIT_MAX_AGE_MS) || 6 * HOUR_MS;

/** BILLABLE tokens on one usage object — input + output + cache_creation. EXCLUDES
 *  cache_read_input_tokens on purpose (the dominant inflator: ~95% of the raw total
 *  at heavy load; see file header). This is the gate's fallback metric. */
function billableTokens(u) {
  if (!u || typeof u !== "object") return 0;
  return (
    (u.input_tokens || 0) +
    (u.output_tokens || 0) +
    (u.cache_creation_input_tokens || 0)
  );
}

/** TOTAL tokens incl. cache_read — the raw "consumed" figure. Used by the ad-hoc
 *  analysis fns (usageWindows / categorizeUsage), NOT the gate. */
function lineTokens(u) {
  if (!u || typeof u !== "object") return 0;
  return billableTokens(u) + (u.cache_read_input_tokens || 0);
}

/** Read the last `maxBytes` of a file as utf8 (transcripts can be large; the lines
 *  we care about are appended at the end, within the window). */
function readTail(path, maxBytes) {
  let fd;
  try {
    fd = openSync(path, "r");
    const size = statSync(path).size;
    const want = Math.min(maxBytes, size);
    const buf = Buffer.alloc(want);
    if (want > 0) readSync(fd, buf, 0, want, size - want);
    return buf.toString("utf8");
  } catch {
    return "";
  } finally {
    if (fd !== undefined) try { closeSync(fd); } catch { /* ignore */ }
  }
}

/** Pull every `rate_limit_event` object out of a chunk of text. Handles the NATIVE
 *  form (a top-level stream-json line in a leaf log) AND the ESCAPED form (a
 *  stream-json line captured inside a transcript string field, where `"` became
 *  `\"`). Returns the parsed `rate_limit_info` objects in file order. */
function extractRateLimitInfos(text) {
  const infos = [];
  if (!text || text.indexOf("rate_limit_event") < 0) return infos;
  // Match a brace-balanced JSON object starting at "type":"rate_limit_event",
  // in either the native or backslash-escaped quoting.
  const re = /\{\\?"type\\?":\\?"rate_limit_event\\?".*?\\?"rate_limit_info\\?":\s*(\{.*?\})/g;
  let m;
  while ((m = re.exec(text))) {
    let frag = m[1];
    // Un-escape the escaped form (\" → ", \\ → \) so JSON.parse accepts it.
    if (frag.indexOf('\\"') >= 0) frag = frag.replace(/\\"/g, '"').replace(/\\\\/g, "\\");
    try {
      const info = JSON.parse(frag);
      if (info && typeof info === "object") infos.push(info);
    } catch { /* skip a fragment we can't parse */ }
  }
  return infos;
}

/**
 * Read Anthropic's OWN per-account rate-limit verdict — the authoritative
 * subscription-limit signal — from the most recent leaf stream-json logs (and, as a
 * secondary source, the captured transcripts). Returns the latest event per
 * `rateLimitType`, chosen by (file mtime, then resetsAt) so the current window wins.
 *
 * @param {{ workerLogsDir?:string, projectsDir?:string, maxAgeMs?:number, maxFiles?:number, tailBytesPerFile?:number }} [opts]
 * @returns {{ ok:boolean, fiveHour:{status:string,resetsAt:number|null}|null, sevenDay:{status:string,utilization:number|null,resetsAt:number|null}|null, latestMs:number, ageMs:number|null, source:string, error?:string }}
 */
export function readRateLimit(opts = {}) {
  const workerLogsDir = opts.workerLogsDir || join(process.cwd(), ".worker-logs");
  const projectsDir = opts.projectsDir || join(homedir(), ".claude", "projects");
  const maxAgeMs = opts.maxAgeMs || RATE_LIMIT_MAX_AGE_MS;
  const maxFiles = opts.maxFiles || 40;
  const tailBytes = opts.tailBytesPerFile || 256 * 1024; // events sit near the file end
  const now = Date.now();
  const cutoff = now - maxAgeMs;

  // best[type] = { resetsAt, mtime, info } — most-recent by (mtime, resetsAt).
  const best = {};
  let latestMs = 0;
  let source = "none";

  /** Scan one flat dir of files, newest-mtime first, for rate_limit_event lines. */
  const scanDir = (dir, label, recurse) => {
    let entries;
    try { entries = readdirSync(dir, { withFileTypes: true }); } catch { return; }
    const files = [];
    for (const e of entries) {
      const fp = join(dir, e.name);
      if (e.isDirectory()) { if (recurse) scanDir(fp, label, false); continue; }
      let mtime;
      try { mtime = statSync(fp).mtimeMs; } catch { continue; }
      if (mtime < cutoff) continue;
      files.push({ fp, mtime });
    }
    files.sort((a, b) => b.mtime - a.mtime);
    for (const { fp, mtime } of files.slice(0, maxFiles)) {
      const infos = extractRateLimitInfos(readTail(fp, tailBytes));
      for (const info of infos) {
        const type = info.rateLimitType || "unknown";
        const resetsAt = Number(info.resetsAt) || 0;
        const cur = best[type];
        if (!cur || mtime > cur.mtime || (mtime === cur.mtime && resetsAt >= cur.resetsAt)) {
          best[type] = { resetsAt, mtime, info };
          if (mtime > latestMs) { latestMs = mtime; source = label; }
        }
      }
    }
  };

  // Worker-logs are the PRIMARY source (native stream-json lines, always fresh while
  // the caravan dispatches). Only fall through to the transcripts if they yield nothing.
  try {
    scanDir(workerLogsDir, "worker-logs", false);
    if (!Object.keys(best).length) scanDir(projectsDir, "transcripts", true);
  } catch (err) {
    return { ok: false, error: err?.message || String(err), fiveHour: null, sevenDay: null, latestMs: 0, ageMs: null, source: "none" };
  }

  const five = best.five_hour?.info || null;
  const seven = best.seven_day?.info || null;
  return {
    ok: true,
    fiveHour: five ? { status: five.status || "unknown", resetsAt: Number(five.resetsAt) || null } : null,
    sevenDay: seven
      ? { status: seven.status || "unknown", utilization: typeof seven.utilization === "number" ? seven.utilization : null, resetsAt: Number(seven.resetsAt) || null }
      : null,
    latestMs,
    ageMs: latestMs ? now - latestMs : null,
    source,
  };
}

/**
 * Read 5h + 7d subscription usage and decide whether the caravan should HOLD.
 *
 * AUTHORITATIVE-FIRST: when a fresh `rate_limit_event` is present (a leaf has run
 * stream-json within `RATE_LIMIT_MAX_AGE_MS`), Anthropic's OWN verdict drives the
 * gate — `five_hour`/`seven_day` "rejected" HOLDs, and `seven_day` utilization at/above
 * the hold fraction HOLDs (a real %, headroom below the hard reject). No ceiling
 * guessing. The BILLABLE-token estimate (cache_read-excluded) is computed alongside
 * for telemetry, and becomes the GATE only as a FALLBACK when no fresh event exists.
 *
 * @param {{ fiveHourCeiling?:number, weekCeiling?:number, thresholdPct?:number, weekUtilHold?:number, projectsDir?:string, workerLogsDir?:string, tailBytesPerFile?:number }} [opts]
 * @returns {{ ok:boolean, over:boolean, reason:string, signal:"rate_limit"|"token_fallback"|"error", rateLimit:object, fiveHour:{tokens:number,ceiling:number,pct:number}, week:{tokens:number,ceiling:number,pct:number}, error?:string }}
 */
export function checkUsage(opts = {}) {
  const fiveHourCeiling = opts.fiveHourCeiling || DEFAULT_5H_CEILING;
  const weekCeiling = opts.weekCeiling || DEFAULT_WEEK_CEILING;
  const thresholdPct = opts.thresholdPct || DEFAULT_THRESHOLD_PCT;
  const weekUtilHold = opts.weekUtilHold || DEFAULT_WEEK_UTIL_HOLD;
  // Read at most this many trailing bytes per file (a week of one session is well
  // under this; bounds the per-tick cost regardless of total transcript size).
  const tailBytes = opts.tailBytesPerFile || 8 * 1024 * 1024;
  const root = opts.projectsDir || join(homedir(), ".claude", "projects");

  const now = Date.now();
  const cut5 = now - 5 * HOUR_MS;
  let tok5 = 0, tok7 = 0;

  // ── AUTHORITATIVE signal — Anthropic's own rate_limit_event ────────────────
  // Read FIRST so its resetsAt can anchor the weekly token window (below).
  // VALIDITY by resetsAt, NOT just file mtime: an event is the verdict for its
  // window only while that window is still OPEN (resetsAt in the future). A
  // re-surfaced event from an expired window (e.g. an old log re-read into a fresh
  // file, or echoed into a transcript) carries a PAST resetsAt and is correctly
  // ignored — so it can never produce a stale hold.
  let rl;
  try { rl = readRateLimit({ workerLogsDir: opts.workerLogsDir, projectsDir: root }); }
  catch (e) { rl = { ok: false, error: e?.message || String(e), fiveHour: null, sevenDay: null, ageMs: null, source: "none" }; }
  const nowSec = now / 1000;

  // WEEKLY WINDOW START — anchor to Anthropic's real reset boundary, NOT a naive
  // rolling 7d. The weekly limit resets on a FIXED weekly cadence (observed:
  // Mon 07:00 America/LA); a seven_day event's `resetsAt` is one such boundary, so
  // the current window began at the most recent boundary ≤ now (anchor ± k·7d). This
  // makes the weekly token number reflect the CURRENT week (usage since the last
  // reset) instead of double-counting pre-reset tokens Anthropic already cleared.
  // Prefer a live seven_day resetsAt (self-updating); else the observed seed below.
  const weeklyResetMs = rl.sevenDay?.resetsAt ? rl.sevenDay.resetsAt * 1000 : WEEKLY_ANCHOR_SEED_MS;
  let weekStart = weeklyResetMs;
  while (weekStart + WEEK_MS <= now) weekStart += WEEK_MS; // forward to the boundary ≤ now
  while (weekStart > now) weekStart -= WEEK_MS;            // (or back, if the anchor is future)
  const tokFileCut = Math.min(cut5, weekStart);

  // ── billable-token estimate (telemetry always; gate only on fallback) ──────
  try {
    const dirs = readdirSync(root, { withFileTypes: true });
    for (const d of dirs) {
      if (!d.isDirectory()) continue;
      const sub = join(root, d.name);
      let files;
      try { files = readdirSync(sub); } catch { continue; }
      for (const name of files) {
        if (!name.endsWith(".jsonl")) continue;
        const fp = join(sub, name);
        let mtime;
        try { mtime = statSync(fp).mtimeMs; } catch { continue; }
        if (mtime < tokFileCut) continue; // nothing in either window
        const text = readTail(fp, tailBytes);
        let nl = text.indexOf("\n"); // skip a possibly-truncated first line
        const body = nl >= 0 ? text.slice(nl + 1) : text;
        for (const ln of body.split("\n")) {
          if (ln.indexOf('"usage"') < 0) continue;
          let o;
          try { o = JSON.parse(ln); } catch { continue; }
          const ts = Date.parse(o.timestamp || "");
          if (!ts) continue;
          const u = o.message && typeof o.message === "object" ? o.message.usage : o.usage;
          const t = billableTokens(u);
          if (!t) continue;
          if (ts >= weekStart) tok7 += t;
          if (ts >= cut5) tok5 += t;
        }
      }
    }
  } catch (err) {
    // Fail-OPEN: a broken monitor must not silently halt all dispatch (the
    // claude-heavy cap still applies). Surface the error so it is visible.
    return {
      ok: false,
      error: err?.message || String(err),
      signal: "error",
      rateLimit: { ok: false, fiveHour: null, sevenDay: null },
      fiveHour: { tokens: 0, ceiling: fiveHourCeiling, pct: 0 },
      week: { tokens: 0, ceiling: weekCeiling, pct: 0 },
      over: false,
      reason: `usage monitor error (fail-open): ${err?.message || err}`,
    };
  }

  const pct5 = fiveHourCeiling > 0 ? Math.round((tok5 / fiveHourCeiling) * 1000) / 10 : 0;
  const pct7 = weekCeiling > 0 ? Math.round((tok7 / weekCeiling) * 1000) / 10 : 0;
  const tokenBlock = {
    fiveHour: { tokens: tok5, ceiling: fiveHourCeiling, pct: pct5 },
    week: { tokens: tok7, ceiling: weekCeiling, pct: pct7, windowStart: new Date(weekStart).toISOString(), anchor: rl.sevenDay?.resetsAt ? "live" : "seed" },
  };

  const fiveValid = !!(rl.ok && rl.fiveHour && rl.fiveHour.resetsAt && rl.fiveHour.resetsAt > nowSec);
  const sevenValid = !!(rl.ok && rl.sevenDay && rl.sevenDay.resetsAt && rl.sevenDay.resetsAt > nowSec);

  // PER-WINDOW best signal:
  //   • 5h  — authoritative status when valid; else the calibrated token estimate
  //           (the 5h token sum is reliable: short window, cache_read-excluded).
  //   • 7d  — authoritative status/utilization ONLY. The 7d token sum is NEVER a
  //           hold trigger: a rolling-7d sum spans weekly reset boundaries (it keeps
  //           counting pre-reset tokens Anthropic no longer charges to the current
  //           week), so it false-reads high right after a reset. The weekly limit is
  //           caught authoritatively whenever a fresh seven_day event exists (active
  //           dispatch emits them), with the five_hour reject as the exhaustion backstop.
  const fiveSig = fiveValid ? `5h=${rl.fiveHour.status}` : `5h~token ${pct5}%`;
  const fiveOver = fiveValid ? rl.fiveHour.status === "rejected" : pct5 >= thresholdPct;

  const sevenUtil = sevenValid && typeof rl.sevenDay.utilization === "number" ? rl.sevenDay.utilization : null;
  const sevenOver = sevenValid && (rl.sevenDay.status === "rejected" || (sevenUtil != null && sevenUtil >= weekUtilHold));
  const weekSig = sevenValid
    ? `wk=${rl.sevenDay.status}${sevenUtil != null ? ` util ${Math.round(sevenUtil * 1000) / 10}%` : ""}`
    : `wk~unmetered (token est ${pct7}%)`;

  const over = fiveOver || sevenOver;
  const signal = fiveValid || sevenValid ? "rate_limit" : "token_fallback";
  const reason = `usage ${over ? "gate HOLD" : "OK"} (${signal}) — ${fiveSig}, ${weekSig}`;
  return { ok: true, over, signal, rateLimit: rl, ...tokenBlock, reason };
}

/**
 * Bucket local Claude token usage into `count` consecutive `windowHours` windows
 * back from now (window 0 = most recent). Ad-hoc analysis (NOT the per-tick gate) —
 * lets you see whether pacing is bending the consumption curve over time. Uses a
 * generous tail read per file to bound cost while covering the lookback.
 *
 * @param {{ windowHours?:number, count?:number, projectsDir?:string, tailBytesPerFile?:number }} [opts]
 * @returns {{ ok:boolean, windowHours:number, count:number, files:number, lines:number, windows:Array<{index:number, hoursAgoStart:number, hoursAgoEnd:number, tokens:number}>, error?:string }}
 */
export function usageWindows({ windowHours = 5, count = 6, projectsDir, tailBytesPerFile } = {}) {
  const root = projectsDir || join(homedir(), ".claude", "projects");
  const tailBytes = tailBytesPerFile || 32 * 1024 * 1024;
  const now = Date.now();
  const cutoff = now - count * windowHours * HOUR_MS;
  const buckets = new Array(count).fill(0);
  let files = 0, lines = 0;
  try {
    for (const d of readdirSync(root, { withFileTypes: true })) {
      if (!d.isDirectory()) continue;
      const sub = join(root, d.name);
      let names;
      try { names = readdirSync(sub); } catch { continue; }
      for (const name of names) {
        if (!name.endsWith(".jsonl")) continue;
        const fp = join(sub, name);
        let mtime;
        try { mtime = statSync(fp).mtimeMs; } catch { continue; }
        if (mtime < cutoff) continue;
        files++;
        const text = readTail(fp, tailBytes);
        const nl = text.indexOf("\n");
        const body = nl >= 0 ? text.slice(nl + 1) : text;
        for (const ln of body.split("\n")) {
          if (ln.indexOf('"usage"') < 0) continue;
          let o;
          try { o = JSON.parse(ln); } catch { continue; }
          const ts = Date.parse(o.timestamp || "");
          if (!ts || ts < cutoff || ts > now) continue;
          const u = o.message && typeof o.message === "object" ? o.message.usage : o.usage;
          const t = lineTokens(u);
          if (!t) continue;
          const idx = Math.floor((now - ts) / (windowHours * HOUR_MS));
          if (idx >= 0 && idx < count) { buckets[idx] += t; lines++; }
        }
      }
    }
  } catch (err) {
    return { ok: false, error: err?.message || String(err), windowHours, count, files, lines, windows: [] };
  }
  const windows = buckets.map((tokens, i) => ({
    index: i,
    hoursAgoStart: (i + 1) * windowHours,
    hoursAgoEnd: i * windowHours,
    tokens,
  }));
  return { ok: true, windowHours, count, files, lines, windows };
}

/** The 4 work categories (operator's taxonomy 2026-06-23). */
const CATS = ["develop", "restart", "work", "analyze"];
const CODE_RE = /\.(js|mjs|cjs|ts|tsx|py|sh|sql)$/i;
const RESTART_RE = /(schtasks|taskkill|npm run|\btsx\b|\bnode\b|start-process|stop-process|restart-|restart\b|pg_ctl|enable-scheduledtask|start-scheduledtask|\/run\b|\/end\b|\bstart\b)/i;
const MUTATE_RE = /(insert\s|update\s|delete\s|upsert|merge\s|-x\s*(post|patch|put|delete)|prefer:\s*return|\.from\([^)]*\)\.(insert|update|delete))/i;

/**
 * Classify ONE assistant turn into a work category by the tools it invoked.
 * Priority: code-write → develop; start/restart a script → restart; data mutation
 * or non-code file output → work; read/query/reason → analyze (the catch-all,
 * incl. tool-less "deciding" turns). The restart-vs-work split is inherently fuzzy
 * (both are exec calls; the real work is non-LLM), so treat those two as a blend.
 * @param {Array|string} content message.content
 * @returns {"develop"|"restart"|"work"|"analyze"}
 */
function classifyTurn(content) {
  const blocks = Array.isArray(content) ? content : [];
  let codeWrite = false, otherWrite = false, restart = false, mutate = false, readQuery = false;
  for (const b of blocks) {
    if (!b || b.type !== "tool_use") continue;
    const name = (b.name || "").toLowerCase();
    const inp = b.input || {};
    if (name.includes("write") || name.includes("edit") || name.includes("notebook")) {
      if (CODE_RE.test(String(inp.file_path || ""))) codeWrite = true;
      else otherWrite = true;
    } else if (name === "bash" || name.includes("powershell") || name.includes("exec") || name.includes("shell")) {
      const cmd = String(inp.command || "");
      if (MUTATE_RE.test(cmd)) mutate = true;
      else if (RESTART_RE.test(cmd)) restart = true;
      else readQuery = true; // select / get-content / findstr / ls / git status / cat …
    } else if (name.includes("read") || name.includes("grep") || name.includes("glob") || name.includes("search")) {
      readQuery = true;
    } else {
      readQuery = true; // TodoWrite, web fetch, misc → lump into analyze
    }
  }
  if (codeWrite) return "develop";
  if (restart) return "restart";
  if (mutate || otherWrite) return "work";
  return "analyze"; // reads/queries, or a tool-less reasoning/deciding turn
}

/**
 * Categorize local Claude token spend into develop / restart / work / analyze by
 * per-assistant-turn tool-use heuristic. Attributes each assistant message's usage
 * tokens to the category of the tools it called. % is reported BOTH by total tokens
 * (cache-read-inclusive — what's consumed) and by OUTPUT tokens (active generation,
 * less cache-dominated, a cleaner "effort" signal).
 *
 * @param {{ hoursBack?:number, projectsDir?:string, tailBytesPerFile?:number }} [opts]
 * @returns {{ ok:boolean, hoursBack:number, files:number, turns:number, totalTokens:number, totalOut:number, categories:Object, error?:string }}
 */
export function categorizeUsage({ hoursBack = 30, projectsDir, tailBytesPerFile } = {}) {
  const root = projectsDir || join(homedir(), ".claude", "projects");
  const tailBytes = tailBytesPerFile || 32 * 1024 * 1024;
  const now = Date.now();
  const cutoff = now - hoursBack * HOUR_MS;
  const acc = {}; for (const c of CATS) acc[c] = { tokens: 0, out: 0, turns: 0 };
  let files = 0, turns = 0;
  try {
    for (const d of readdirSync(root, { withFileTypes: true })) {
      if (!d.isDirectory()) continue;
      const sub = join(root, d.name);
      let names; try { names = readdirSync(sub); } catch { continue; }
      for (const name of names) {
        if (!name.endsWith(".jsonl")) continue;
        const fp = join(sub, name);
        let mtime; try { mtime = statSync(fp).mtimeMs; } catch { continue; }
        if (mtime < cutoff) continue;
        files++;
        const text = readTail(fp, tailBytes);
        const nl = text.indexOf("\n");
        const body = nl >= 0 ? text.slice(nl + 1) : text;
        for (const ln of body.split("\n")) {
          if (ln.indexOf('"usage"') < 0) continue;
          let o; try { o = JSON.parse(ln); } catch { continue; }
          const ts = Date.parse(o.timestamp || "");
          if (!ts || ts < cutoff || ts > now) continue;
          const msg = o.message;
          if (!msg || typeof msg !== "object" || msg.role !== "assistant") continue;
          const t = lineTokens(msg.usage);
          if (!t) continue;
          const cat = classifyTurn(msg.content);
          acc[cat].tokens += t;
          acc[cat].out += (msg.usage.output_tokens || 0);
          acc[cat].turns += 1;
          turns++;
        }
      }
    }
  } catch (err) {
    return { ok: false, error: err?.message || String(err), hoursBack, files, turns, categories: acc };
  }
  const totalTokens = CATS.reduce((s, c) => s + acc[c].tokens, 0) || 1;
  const totalOut = CATS.reduce((s, c) => s + acc[c].out, 0) || 1;
  const categories = {};
  for (const c of CATS) {
    categories[c] = {
      tokens: acc[c].tokens,
      pctTotal: Math.round((acc[c].tokens / totalTokens) * 1000) / 10,
      outTokens: acc[c].out,
      pctOut: Math.round((acc[c].out / totalOut) * 1000) / 10,
      turns: acc[c].turns,
    };
  }
  return { ok: true, hoursBack, files, turns, totalTokens, totalOut, categories };
}

export default { checkUsage, readRateLimit, usageWindows, categorizeUsage, toc };
```
<!-- /GUILDOS-FILE -->
