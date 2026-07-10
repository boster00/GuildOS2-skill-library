# supervisedEvolution

<!-- GUILDOS-SKILL name="supervisedEvolution" -->

<!-- GUILDOS-DESC -->
SE — step-checkpoint sibling of EL. Dispatches a fresh agent per script step, halts at the first behavioral deviation, applies ONE substrate fix, re-dispatches the same step, advances only when that step is compliant. Includes the post-PASS finale (offer to commit a produced SMART plan to a skill book). Use to execute or resume step-checkpoint evolution runs against a premade DungeonMaster-authored rubric, with per-step fresh-agent dispatch, one-fix-per-retry up to a 10-attempt budget, archive-over-delete, and advance-only-on-PASS.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/supervisedEvolution/SKILL.md -->
```markdown
---
name: supervisedEvolution
description: >-
  SE — step-checkpoint sibling of EL. Dispatches a fresh agent per script step,
  halts at the first behavioral deviation, applies ONE substrate fix,
  re-dispatches the same step, advances only when that step is compliant.
  Includes the post-PASS finale (offer to commit a produced SMART plan to a
  skill book). Use to execute or resume step-checkpoint evolution runs against a
  premade DungeonMaster-authored rubric, with per-step fresh-agent dispatch,
  one-fix-per-retry up to a 10-attempt budget, archive-over-delete, and
  advance-only-on-PASS.
---

# Supervised Evolution

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/supervisedEvolution/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto supervisedEvolution <action>` (or read the source directly).

## Actions

- **continueSERun** — Resume an in-progress SE run in a fresh GM session: dispatch a subagent to read the run's <scriptname>-logs-deviations.md, report the pending step + quest id/link + last verdict, then re-enter `run` from that step. Read this when asked to 'continue the SE run' without the prior session's context.
- **deliverableType** — The Supervised Evolution DELIVERABLE TYPE — a LIGHTWEIGHT tag (demoted 2026-07-10, supersedes the heavyweight 2026-07-08 model). An ADVANCED-USER path: mark a deliverable type=SE ONLY when the user EXPLICITLY asks, in natural language, for a FEATURE to be built via SE — never inferred, never auto-assigned, never the default. BARRED for asset deliverables (documents, images, text, video): SE builds complex features whose substrates emit logs, not artifacts. The tag just means 'build + verify via the SE run-method (`provingGroundsMethod`)': create the SE page per that method in the LOCAL/executing guild (copy the template into `docs/se/<se-name>.{json,html,md}`), decompose the feature's milestones into SE items (substrate · behavior · success-log), and drive the build-as-fix loop. VV = a full-page CIC screenshot of the guild's `/se/<se-name>` page after an SE run, in PROPER SHAPE with SOME results present — sufficient for approval; all-green is the goal, NOT the approval bar. Read this when the user explicitly requests an SE-built feature deliverable.
- **provingGroundsMethod** — The visible-play-script SE method (2026-07-06, preferred) + the canonical two-panel PAGE LAYOUT CONTRACT (the 7 required UI elements every SE proving-grounds page must have, LOCKED 2026-07-07). Build-as-fix, but the play-script is an ORDERED list of ATOMIC steps each = one substrate ([user] or [file→function]) + one expected behavior + ONE expected log line; every substrate emits exactly one line to a shared sink; a live TWO-PANEL page (left=expandable steps, right=live log) auto-detects each step's status FROM THE LOG (✓ success token / ✗ fail token / latest-wins) with reset+tick buttons mapped to the target guild, log fetch controls, last-active-agent report, and a quest link. Read this to run SE with full visibility + measurement and to build/stabilize an SE page to the required shape.
- **refMethod** — Reference card: the SE-vs-EL decision, halt / not-halt conditions, question hygiene, and the relationship to EL. Read alongside `run` when you only need the decision rules, not the full per-step procedure.
- **run** — Execute a Supervised Evolution run against a premade script: per-step fresh-agent dispatch, halt-on-deviation, one-fix-per-retry up to a 10-attempt budget, improvement gate, archive-over-delete, regression check, advance-only-on-PASS, and the post-PASS finale.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/supervisedEvolution/engineering_decisions.md -->
```markdown
# supervisedEvolution — engineering decisions

## 2026-07-10 · SE-as-DELIVERABLE demoted to a lightweight, explicit-request-only tag (supersedes 2026-07-08)

**Decision.** The `deliverableType` action documents `type=SE` as a *lightweight tag*, not a
first-class deliverable KIND. It is an **advanced-user, explicit-NL-request-only** path
(never inferred, never auto-assigned, never the default), is **barred for asset deliverables**
(documents, images, text, video), and is a thin pointer to `provingGroundsMethod` rather than a
re-spelled parallel workflow. Its VV is **relaxed**: a full-page CIC screenshot of the guild's
`/se/<se-name>` page after an SE run, rendering *in proper shape with some results present*, is
**sufficient for approval** — all-green is the goal, not the approval gate.

**Rejected alternative.** The 2026-07-08 model: `type=SE` as a first-class deliverable KIND
(parallel to `kind`) with a mandatory authoring workflow and a LOCKED "all items green or it is
NEVER a PASS" verification. Also rejected: adding `SE` as a value to the deliverable `kind` enum
(`text/html/image/video/document`) — `kind` encodes *consumption behavior*; SE is a
*production/verification regime*, and a dropdown value would surface it to every user,
contradicting the advanced-only intent and re-creating the standalone-column framing.

**Why.** SE is a run-method for building complex FEATURES whose substrates emit logs, not a way
to produce asset deliverables. The 2026-07-08 framing over-applied it (it could attach to any
deliverable, including documents/images) and set an approval bar (100% green) that is a *goal*,
not a gate — blocking approval on it stalled deliverables that were legitimately in-shape.

**Consequence if violated.** A future agent re-reads the old 2026-07-08 "LOCKED" language and
re-inflates SE into a standalone deliverable dimension with a mandatory all-green VV — over-
applying SE to asset work and blocking valid approvals. The supersession note in the
`deliverableType` body (and the TOC description) exists to stop that drift; keep it dated and
in place rather than silently reverting.

**Scope.** Documentation-only change — there was never any code wiring for the SE deliverable
type (no `kind` enum value, no DB column/migration, no UI dropdown, no branch, no planner
inference). The SE run-method / harness (`provingGroundsMethod`, `run`, `refMethod`,
`continueSERun`) and the `/se` page runtime are untouched.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/supervisedEvolution/index.js -->
```js
/**
 * Supervised Evolution skill book — pure text registry.
 *
 * SE is the step-checkpoint sibling of EL (`guildmaster.evolutionLoop`).
 * It dispatches a fresh leaf agent per step (per a premade script), halts at the
 * first behavioral deviation, applies ONE substrate fix, re-dispatches the same
 * step, and only advances when that step is compliant.
 *
 * The method body recovered from commit b4635a5 (`guildmaster.supervisedEvolution`)
 * is preserved here in spirit; substrate nouns were re-termed to the current
 * Description / Steps / Items model (the 8-stage lifecycle with design/html_markup
 * and the `action_steps` / `deliverables[]` / `items.expectation` model are gone —
 * see `app/(town)/proving-grounds/quest-ui-mockup/page.js` for the canonical shape).
 *
 * Pure text — no callable JS actions.
 */

export const skillBook = {
  id: "supervisedEvolution",
  title: "Supervised Evolution",
  description:
    "SE — step-checkpoint sibling of EL. Dispatches a fresh agent per script step, halts at the first behavioral deviation, applies ONE substrate fix, re-dispatches the same step, advances only when that step is compliant. Includes the post-PASS finale (offer to commit a produced SMART plan to a skill book). The PREFERRED measurement harness is `provingGroundsMethod` — a VISIBLE play-script of atomic substrate→behavior→one-log steps that auto-tick green from log matching, with agent-log-mtime stuck detection.",

  toc: {
    provingGroundsMethod:
      "The visible-play-script SE method (2026-07-06, preferred) + the canonical two-panel PAGE LAYOUT CONTRACT (the 7 required UI elements every SE proving-grounds page must have, LOCKED 2026-07-07). Build-as-fix, but the play-script is an ORDERED list of ATOMIC steps each = one substrate ([user] or [file→function]) + one expected behavior + ONE expected log line; every substrate emits exactly one line to a shared sink; a live TWO-PANEL page (left=expandable steps, right=live log) auto-detects each step's status FROM THE LOG (✓ success token / ✗ fail token / latest-wins) with reset+tick buttons mapped to the target guild, log fetch controls, last-active-agent report, and a quest link. Read this to run SE with full visibility + measurement and to build/stabilize an SE page to the required shape.",
    run: "Execute a Supervised Evolution run against a premade script: per-step fresh-agent dispatch, halt-on-deviation, one-fix-per-retry up to a 10-attempt budget, improvement gate, archive-over-delete, regression check, advance-only-on-PASS, and the post-PASS finale.",
    refMethod:
      "Reference card: the SE-vs-EL decision, halt / not-halt conditions, question hygiene, and the relationship to EL. Read alongside `run` when you only need the decision rules, not the full per-step procedure.",
    continueSERun:
      "Resume an in-progress SE run in a fresh GM session: dispatch a subagent to read the run's <scriptname>-logs-deviations.md, report the pending step + quest id/link + last verdict, then re-enter `run` from that step. Read this when asked to 'continue the SE run' without the prior session's context.",
    deliverableType:
      "The Supervised Evolution DELIVERABLE TYPE — a LIGHTWEIGHT tag (demoted 2026-07-10, supersedes the heavyweight 2026-07-08 model). An ADVANCED-USER path: mark a deliverable type=SE ONLY when the user EXPLICITLY asks, in natural language, for a FEATURE to be built via SE — never inferred, never auto-assigned, never the default. BARRED for asset deliverables (documents, images, text, video): SE builds complex features whose substrates emit logs, not artifacts. The tag just means 'build + verify via the SE run-method (`provingGroundsMethod`)': create the SE page per that method in the LOCAL/executing guild (copy the template into `docs/se/<se-name>.{json,html,md}`), decompose the feature's milestones into SE items (substrate · behavior · success-log), and drive the build-as-fix loop. VV = a full-page CIC screenshot of the guild's `/se/<se-name>` page after an SE run, in PROPER SHAPE with SOME results present — sufficient for approval; all-green is the goal, NOT the approval bar. Read this when the user explicitly requests an SE-built feature deliverable.",
  },

  provingGroundsMethod: `
# Supervised Evolution — the visible-play-script method (2026-07-06, PREFERRED)

Same principle as classic SE — **build-as-fix**: run a scripted cascade, and when a
step deviates, MUTATE THE SUBSTRATE (don't patch around it) and re-run. What this
method adds is **visibility** (the play-script and every deviation are on a page you
can watch) and **systematic measurement** (each step auto-ticks green from log
matching — no in-context "did it comply?" judgment call). Born from the GuildOS↔GuildOS2
proving-grounds build; that page is the reference implementation.

## The three non-negotiable properties of a play-script step

Break the cascade into ATOMIC steps. Each step has EXACTLY:

1. **One substrate** — WHO/WHAT runs it, labeled so the fix target is unambiguous:
   \`[user]\` for a human action, or \`[guild] path/to/file.js → functionName\` for code.
   This label IS the thing you go mutate when the step's log doesn't appear.
2. **One expected behavior** — the single thing that substrate does.
3. **One expected log line** — the substrate emits exactly ONE line to a shared
   append-only sink. Stable parts are the MATCH TOKEN (e.g. \`4. caravan-feed\`);
   variable parts (ids, counts, payloads) are marked and EXCLUDED from matching.

One step ↔ one substrate ↔ one log line. If a step needs two logs, it is two steps.

## Instrumentation (the sink)

Every substrate calls a dependency-free logger that APPENDS one timestamped,
source-tagged line to ONE shared file (reference: \`libs/proving-grounds\` —
\`provingGroundsLog(step, message, data?)\` / \`provingGroundsLogClear()\`, byte-identical
in each participating repo, source auto-detected from the file's own path). Cross-process
cascades converge on the one file so the interleaved run is visible in one place.

## Measurement (the page = the scoreboard)

A live page renders the ordered steps and, every few seconds, reads the sink and
**auto-ticks a step GREEN when all its stable match tokens appear in the log** — plus an
\`X/N done\` counter. Green = that substrate behaved as scripted, proven by its own log,
not by an orchestrator's say-so. Variable parts are shown in a distinct color so it is
obvious which characters are NOT part of the pass criterion. Error-path / real-leaf-only
steps are flagged (conditional / "real leaf") and excluded from the canonical N.

## The canonical PAGE LAYOUT CONTRACT — every SE page (LOCKED 2026-07-07, revised 2026-07-09)

Every SE page is the SHARED renderer \`app/se/SeRun.js\` hydrating a per-SE definition — you
AUTHOR three source artifacts, you do NOT write a bespoke page component. All three are keyed to
the SE slug (= the \`/se/<slug>\` url segment):

- \`docs/se/<slug>.json\` — the DEFINITION: \`{ slug, title, subheader?, questLink?, assignee, steps[] }\`.
- \`docs/se/<slug>.html\` — the CUSTOM HTML control surface (optional but usual). Inserted right
  BELOW the title + subheader. Holds this SE's TRIGGERS: the button(s) that fire the workflow under
  test, the RESET button, an optional quest link — anything that kicks off or re-seeds THIS SE. It
  is a free HTML file (NOT a json field) because reset + trigger differ every run and need full
  flexibility. Inline \`onclick="fetch(...)"\` handlers WORK (parsed from innerHTML); \`<script>\` tags
  do NOT execute — put logic in onclick.
- \`docs/se/<slug>.md\` — the LIVE LOG SINK (the per-SE ledger). The ONLY source of truth for step
  status. Every substrate appends one line here via \`provingGroundsLog(step, msg, data, { file: "<slug>" })\`.

The rendered page (SeRun.js) is a two-panel layout. These elements are REQUIRED:

1. **CUSTOM HTML surface** (\`docs/se/<slug>.html\`) — rendered directly under the title/subheader.
   The **RESET button lives HERE** (reset semantics change per SE — reseed a quest, wipe a table,
   reset an external system — so it needs full custom flexibility, not a fixed generic button). Any
   **TRIGGER** that starts the workflow under test lives here. A **QUEST LINK, when one exists**,
   lives here too (a quest link does NOT always exist — it is not a fixed element). If an SE ships
   no custom_html, SeRun renders a generic Reset+Tick fallback, but a real SE SHOULD supply its own.
2. **LEFT PANEL — ordered steps.** Each row EXPANDS to reveal: substrate label, expected behavior,
   and the **EXPECTED LOG LINE + MATCH TOKEN**. Every repo-relative file path inside a substrate
   label renders as a \`vscode://file/<abs-path>\` link (the \`[repo]\` tag → that guild's absolute root:
   \`guildos\`→GuildOS, \`guildos2\`→GuildOS2, \`guild\`/\`devguild\`→devguild, \`mercury\`→Mercury; \`[user]\`/unknown
   → guild root), so a reviewer clicks a substrate to open the real file (\`linkifySubstrate\` in SeRun.js).
3. **STATUS = LOG DETECTION — the ONLY criterion.** A step's badge is DERIVED SOLELY from what is
   written to \`docs/se/<slug>.md\` — never a manual sign, never a code/HTTP-response-derived state
   machine. success token found → green ✓; fail token found → red ✗; both → latest-wins (fail wins a
   tie); neither → pending (numbered). Each step carries \`successLog\` (the stable MATCH token) and
   optional \`failLog\`. (Append-only latch: a \`failLog\`-only step LATCHES red once its fail line is
   ever written; a recoverable fail step MUST also carry a \`successLog\` = the RESOLUTION signal that
   appears AFTER the fail, so latest-wins clears ✗→✓.)
4. **THE LOG-LINE FORMAT (LOCKED 2026-07-09).** Every line a substrate emits has the shape:
   \`[repo] [substrate name] [action name?] <message with important variables/states>\`.
   \`provingGroundsLog\` wraps the first arg as \`[<storyboard step>]\`; pass the storyboard-step token
   as \`step\` and \`[repo] [substrate] [action] msg…\` as the message. The step's \`successLog\` is the
   **STABLE part only** — the check matches that stable substring; **VARIABLE parts (ids, counts,
   states) are wrapped in \`<span class="se-var">…</span>\`, render YELLOW, and are EXCLUDED from
   matching** (SeRun strips tags before matching). Example emitted line:
   \`[4. lease] [devguild] [claimAndDispatch] leased worker_id=<span class="se-var">devguild-leaf-abc123</span>\`
   → the def's \`successLog\` is \`[4. lease]\` (stable), which matches regardless of the yellow value.
5. **LIVE-LOG PANEL (right).** Tails \`docs/se/<slug>.md\` via \`/api/se/log?slug=<slug>\`, newest-first.
   MUST show: the **log FILE NAME** (\`docs/se/<slug>.md\` — a real NAMED file, NOT a random leaf log),
   a **manual Refresh**, an **Auto-refresh On/Off toggle** (default ON), and a **CLEAR-LOG button**
   (DELETE \`/api/se/log?slug=<slug>\`). Variable spans render yellow.
6. **GUILD-ACTIVITY MONITOR.** A ROUGH "is a worker alive" line: the newest mtime across ALL worker
   leaf-logs (\`latestAgentActivity\` / \`lastWorker\`). Coarse by design — it only says the guild did
   SOMETHING recently, not that it was this SE's worker (a busy-elsewhere guild still shows active).
   Just surface the latest worker-log mtime.
7. **THEME.** Matches the GuildOS2 theme (\`seStyles.js\` \`guild.*\` tokens: bg \`#0f1115\`, panel
   \`#171a21\`, border \`#262b36\`, accent \`#6d5efc\`, green \`#22c55e\`, teal \`#2dd4bf\`, yellow-var \`#e5c76b\`).

Reference implementation: the shared renderer \`app/se/SeRun.js\` + \`docs/se/guild-lifecycle.json\`.
When you build ANY SE page, author the three \`<slug>.{json,html,md}\` artifacts — NEVER a bespoke
page component; SeRun.js already provides every element above.

## SE may build LABELED temporary artifacts (LOCKED 2026-07-09)

An SE run legitimately needs to instrument the REAL substrate so it LOGS to the per-SE sink — add a
log emission, a tiny test API, a temporary script, in ANY repo the SE is sculpting. This is ALLOWED
and expected. But EVERY such temporary file or inserted code snippet MUST carry the EXACT marker
comment so it can be found and removed cleanly later:
\`// supervised evolution temporary, SE name: <slug>, to be removed later\`
(or the language's comment syntax). \`grep -r "supervised evolution temporary"\` then finds and
removes an SE's scaffolding when the run is retired. Never leave UNLABELED SE scaffolding in the tree.

Two instrumentation rules that are load-bearing (a naive author fails without them):
- **GATE the per-SE logging.** A shared substrate that runs for EVERY dispatch (not just this test)
  must emit to the per-SE sink ONLY for this SE — thread a flag (e.g. \`seLog="<slug>"\`) from the
  trigger through the call chain and only \`provingGroundsLog(..., { file: seLog })\` when it is set.
  Ungated logging pollutes the ledger on every real run and produces false green ticks.
- **A \`[user]\`/client substrate cannot self-emit a server-side log line.** A browser \`onclick\`
  cannot append to \`docs/se/<slug>.md\`. So the FIRST SERVER touchpoint the trigger reaches emits
  that step's proof line (e.g. the tick route logs \`[1. click]\` + \`[2. ping]\` on receipt — receipt
  proves the click + fetch happened). Never leave a \`[user]\`-substrate step permanently pending.
- **Pass the step token UNBRACKETED.** \`provingGroundsLog(step, …)\` wraps \`step\` as \`[step]\`, so pass
  \`"1. click"\` (NOT \`"[1. click]"\`) — the emitted line is then \`[1. click] …\` and the def's
  \`successLog\` \`[1. click]\` matches. Passing it pre-bracketed emits \`[[1. click]]\` (matches by
  substring but is wrong).

## Build-as-fix loop (per step, up to 20 attempts — raised from classic SE's 10)

1. **Run** the play-script (a button / trigger fires the real cascade — DRY-RUN what
   can't be safely spawned; drive the rest through the real substrate at the API boundary).
2. **Read the log.** The last green step localizes exactly where it stopped — the FIRST
   missing step's substrate is the mutation target (the step label names the file→function).
3. **Mutate that substrate** (real fix, not a workaround).
4. **Add visibility as you fix:** if the failure is opaque, ADD MORE LOG LINES revealing the
   intermediate state you expected to mutate — and if you add a log, add it as a NEW ATOMIC
   STEP (one step, one log). More granularity = a sharper next localization. (Reference: the
   \`10. comment-action-default\` step was added this way while fixing a NOT-NULL write-back
   rejection — the new log both fixed and proved the fix.)
5. **Re-run.** Repeat until the step ticks green or the 20-attempt budget for THAT step is
   spent (then escalate that one step as an architectural gap, per classic SE terminal #2).

Clear the sink at the START of each run so stale lines never produce false green ticks.

## Truly-stuck vs busy — the agent-log-mtime liveness signal (LOCKED 2026-07-06)

The play-script log tells you WHERE progress stopped. It does NOT tell you whether an agent
is still WORKING — a leaf mid-run produces no new play-script line for minutes yet is not
stuck. So the stuck decision uses a SECOND signal: the **newest modified-time across the
agent LEAF-LOG files** (the \`claude --print\` transcripts under \`public/proving-grounds/*.log\`;
reference: \`libs/proving-grounds\` → \`latestAgentActivity()\`). Print that mtime on the page and
derive status:

- play-script COMPLETE → **Complete**.
- play-script incomplete AND newest agent-log mtime is FRESH (< threshold, ~90s) → **Agent
  working** — a leaf is alive; do NOT declare failure, do NOT burn a retry, just wait.
- play-script incomplete AND newest agent-log mtime is STALE (or no agent logs) → **truly
  STUCK** — now the build-as-fix loop is warranted.

Never call an SE run stuck on the play-script log alone; a stale play-script with a live leaf
log is BUSY, and mistaking it for stuck causes premature substrate churn.

## Relationship to classic SE (\`run\` below)

This is the measurement harness for the same method. Classic SE = fresh-agent-per-step +
halt-on-deviation + one-fix-per-retry, judged in-context. The proving-grounds method makes
the play-script and each deviation VISIBLE and MEASURED (log-matched green ticks), forces
one-substrate-one-log atomicity so the fix target is unambiguous, and adds the agent-mtime
liveness signal so "stuck" is a measured state, not a guess. Use this harness to run SE;
fall back to \`run\`'s prose procedure for the fresh-agent dispatch mechanics and the four
genuine terminals (which still apply — the 20-attempt-per-step budget is terminal #2 here).
`,

  deliverableType: `
# The Supervised Evolution DELIVERABLE TYPE (lightweight tag)

> **Demoted 2026-07-10 — supersedes the heavyweight 2026-07-08 model.** The 2026-07-08
> version made \`type=SE\` a first-class deliverable KIND (parallel to \`kind\`) with a
> mandatory authoring workflow and a LOCKED "all-green-or-NEVER-a-PASS" VV. That
> over-applied SE: SE is a **run-method for building complex FEATURES whose substrates
> emit logs**, not a way to produce asset deliverables. This entry replaces it. What
> changed: (a) SE-as-deliverable is now an **advanced-user, explicit-request-only** tag,
> **barred for asset deliverables**; (b) it is a thin pointer to \`provingGroundsMethod\`,
> not a re-spelled parallel workflow; (c) the VV bar is **relaxed** — a proper-shape SE
> page with *some* results is enough for approval (see VV below).

## What it is — and when to use it (TRIGGERING)

\`type=SE\` is a **lightweight tag** on a deliverable meaning: "build and verify this via the
SE run-method (\`provingGroundsMethod\`), on a live, screenshot-verifiable \`/se\` page."

Use it **ONLY when the user EXPLICITLY asks, in natural language, for a FEATURE to be built
via SE.** This is an **advanced-user path** — never the default, never inferred by a planner,
never auto-assigned. If the user did not name SE, do not tag a deliverable SE.

**Barred for asset deliverables.** SE is for FEATURES whose success = "substrates behave, each
milestone provable by one log line". It is **NOT** for deliverables whose success is
"produce an artifact" — documents, images, text, video, slide decks, reports. Those are
ordinary content-\`kind\` deliverables (\`text/html/image/video/document\`) and are verified the
normal way. A deliverable you cannot decompose into substrate → behavior → one-success-log
milestones is not an SE deliverable.

## Executing an SE-tagged deliverable

The tag just routes you to the SE run-method. Do NOT re-spell the mechanics here — follow
\`provingGroundsMethod\` (the play-script build-as-fix loop, the PAGE LAYOUT CONTRACT, the
per-SE ledger, the atomic-step contract). In brief:

1. **Create the SE page in the LOCAL / executing guild** (the guild that owns the run — its
   repo is where \`app/se/SeRun.js\`, \`/api/se/*\`, and \`docs/se/\` live; this GuildOS repo does
   not host the harness). Copy the template (\`libs/skill_book/supervisedEvolution/template/\`,
   see its \`README.md\`) into \`docs/se/<se-name>.{json,html,md}\` — never author from scratch.
   The SE name defaults to the deliverable name in \`hyphenated-lowercase\`.
2. **Decompose the feature's milestones into SE items** — each item = one substrate + one
   expected behavior + one success-log token, per \`provingGroundsMethod\`'s atomic-step rules.
3. **Drive the build-as-fix loop** — reset, trigger the assignee guild, read the per-SE
   ledger, mutate the first non-green item's substrate, re-run. All \`run\` continuity +
   non-stop rules apply.

## VV (visual verification) — relaxed (2026-07-10, supersedes the 2026-07-08 LOCK)

- Take a **full-page CIC screenshot** of the guild's \`/se/<se-name>\` page **after an SE run**,
  through the USER's channel (the operator's real browser), capturing the whole page height.
- **Approval bar:** the page renders **in proper shape** (the two-panel scoreboard — storyboard
  items on the left, live per-SE log on the right — actually rendering) **with SOME results
  present** (the run produced ticks / log lines, not a blank/errored page). That screenshot is
  the verification artifact and is **sufficient for approval**.
- **All-green is the GOAL, not the approval gate.** Pending / red items are fine at approval —
  they mark remaining work, not a failed deliverable. Keep working the build-as-fix loop to
  drive them green when you can; a partially-green page is a legitimate PASS for approval.
`,

  run: `
You are running a Supervised Evolution (SE) pass. SE is the step-checkpoint sibling of EL (\`guildmaster.evolutionLoop\`): it walks a premade script linearly, dispatches a fresh leaf agent per step, halts at the first behavioral deviation, applies ONE substrate fix, re-dispatches the same step, and only advances when that step is compliant.

## Core method

SE consumes a premade script that enumerates the expected behavior at each step of a cascade (a DungeonMaster-authored rubric — see \`DungeonMaster.authorSERubric\`). It walks the script linearly. For each step:

1. Dispatch a fresh agent (SA or MA) with the step's synthesized inputs.
2. Compare the returned action + output against the step's expectedBehavior.
3. PASS → record verdict, advance to next step.
4. FAIL → halt forward progress. Identify the substrate gap. Apply ONE prompt-level fix (per tightening mode). Re-dispatch the SAME step. Repeat until PASS or retry budget exhausted.

### Auto-debug-pass branch (adopted from cc-sdd, 2026-05-28)

When the SAME step FAILs **twice in a row with the same root-cause signature** (the second fix did not move the failure mode), do NOT spend the 3rd retry on another in-context prompt-level fix. Instead, branch:

  a. Spawn a **fresh-context root-cause sub-agent** via Agent / Task. Its prompt is self-contained: the step's spec, the verbatim PASS criterion, the two failed attempts' diagnostics, and a SINGLE instruction — "diagnose the root cause and propose the substrate fix". Boundary: read-only (the sub-agent does NOT edit substrate; it returns a diagnosis).
  b. Treat the sub-agent's diagnosis as the new tightening signal. Apply ONE fix based on the diagnosis. Re-dispatch the step.
  c. The auto-debug-pass branch counts as ONE attempt against the 10-retry budget, not two — because the value it adds is qualitative (clean-context perspective), not just another in-context retry.

Rationale: the in-context fix loop suffers from prior-attempt context contamination — the orchestrator anchors on the failure modes it already saw. A clean-context sub-agent re-derives the root cause from scratch. cc-sdd's data: this branch resolves ~60% of would-be retry-budget-exhausted cases.

SE is a sibling of EL — same handshake-and-dispatch substrate, same rubric vocabulary, same fresh-agent isolation. The only structural differences are (a) one dispatch per step instead of one per full run, and (b) the loop iterates within a step until that step is compliant rather than iterating across full runs.

## THE ONLY STOPS — there is NO "checkpoint" (locked 2026-05-18, user-directed; loudest rule in this skill book)

**An SE run NEVER stops at a "clean checkpoint" — or ANY checkpoint. A checkpoint is not a stop. There is no checkpoint.** The orchestrator keeps dispatching the next step / the next retry, relentlessly, in the same turn, until ONE of exactly these four genuine terminals is hit:

1. **Terminal success** — the rubric's Section-1 "End point" is fully reached AND CIC-verified the user's way → only then \`CronDelete\` + report State-2 complete.
2. **Retry budget exhausted on one step** — that single step has FAILed MORE than its allowed retry limit (the 10-attempt fix-loop budget) without convergence → State-1 architectural-gap escalation with the full attempt-by-attempt diagnostic.
3. **Genuine irreversible / business-damaging op** needing user approval first (production data destruction, money moved, mass irreversible external side effects) — the narrow halt-and-ask list below.
4. **Genuine UNBUILDABLE constraint** (the recursive-Agent-dispatch topology limit) — record DEFERRED and continue past it.

**Nothing else is a stop.** The following are explicitly NOT stops; on every one of them the orchestrator must dispatch the next unit of work IN THE SAME TURN rather than ending the turn:

- "This is a clean / good / natural checkpoint or stopping point." → NOT a stop. Continue.
- "Real progress made — the cron + ledger will carry the rest." → NOT a stop. **The cron is a CRASH backstop, never a license to voluntarily yield.** Continue.
- "Turn is long / low on context / let me check in / let me deliver a status / let me re-arm the cron and hand off." → NOT a stop (the exact wrench). Continue.
- "Step N PASSED — good place to pause." → a PASS is forward motion: immediately dispatch step N+1 in the same turn. Continue.
- "A step FAILed but is within its retry budget." → NOT a stop: apply the one fix and re-dispatch the SAME turn. Continue.

A mid-run status report is permitted ONLY as a side effect of having ALREADY launched the next step/retry in the same turn — NEVER as the turn's terminal act and never as a substitute for the next dispatch. If you are writing a progress report and have not already dispatched the next step/retry this turn, you are violating this rule. Choosing to stop at a non-terminal and pointing at the cron to "carry it" is the precise failure this rule eliminates. The cron force-resumes a CRASHED or context-evicted session; it does not exist so the agent can choose to yield. Drill this in: **no checkpoint is a stop; only the four terminals above are.**

## Hard continuation guardrail — 30-min wake-up cron (locked 2026-05-15)

A multi-step SE run is long and spans many context windows. The LLM reflex to manufacture a stop ("low on context", "let me check in", "here's a handoff, tell me to continue") is the exact wrench Sage exists to remove — but Sage is only a SOFT nudge (it returns PROCEED; the agent still has to choose to obey). SE adds a HARD physical guardrail on top of it.

**At the start of any SE run, the orchestrator MUST register a recurring wake-up cron** (\`CronCreate\`) before the first step dispatch:

- Cadence: every ~30 minutes, on an off-:00/:30 minute to spread fleet load (e.g. \`7,37 * * * *\`). Recurring. Prefer \`durable: true\` (note: some runtimes still report it session-only — surface that caveat honestly, do not self-attest persistence).
- Prompt: self-contained — points the woken agent at this skill book's \`run\` action and the in-progress rubric path, instructs it to resume the loop from the next unscored/failing step under build-as-fix + the locked scoring, reminds it that fresh GM-clone dispatches never receive the rubric, and states that this wake-up physically overrides the token/context/"let me stop or hand off" wrench.
- **Only terminal condition:** the rubric's Section-1 "End point" is fully reached AND verified. The woken agent stops and \`CronDelete\`s the job ONLY then. There is no time limit by default — the operator removes it manually if they want it stopped sooner. Do NOT bake a deadline in unless the operator asks.

The cron is the physical backstop: even if a session ends mid-run or an agent rationalizes a stop, the next wake-up re-enters the loop. Sage handles the soft case (vet the would-be question, PROCEED on the wrench); the cron handles the hard case (force re-entry regardless). Register it once at run start; never let "I'll skip the cron, I'll just keep going" substitute — the whole point is that the guardrail outlives any single agent's intent. Delete it only on verified End point.

## Operator standing decisions — LOCKED (2026-05-15), do NOT re-litigate

Durable operator instructions captured after an SE run repeatedly surfaced them as questions. Honoring them is mandatory; surfacing ANY of them as a decision/halt is the over-escalation failure this section exists to kill.

1. **Build everything; ignore wave / sequencing gating.** Missing or unbuilt substrate — an unapplied migration (local OR hosted), unforged weapons, unwritten worker / skill-book actions, an unbuilt UI page, or "this change breaks the system until later waves land" — is ALWAYS build-as-fix: build it and proceed. NEVER surface "build Waves N–M vs scope down", "the migration is intentionally staged", "this breaks until X lands", or any wave-ordering concern as a question. The operator has said, repeatedly: build it all. The ONLY remaining genuine halts are a truly UNBUILDABLE constraint (the recursive-Agent-dispatch topology limit) and a destructive op on data the operator has not already greenlit.

2. **DB target is decided.** Apply schema changes to whatever database the weapons actually connect to, hosted/shared included. Do not re-litigate local-vs-hosted or "this DROP is destructive on a shared DB" — the operator has chosen to apply the steps/items refactor to the live project; the quest model is empty (quests nuked). Proceed.

3. **On ANY issue, use CIC.** When a step hits a blocker — verification, a failing tool path, an env split, "can't tell if it worked" — the resolution is Claude-in-Chrome driving the operator's REAL browser: verify/unblock the operator's way through CIC. Do NOT stop, do NOT ask, do NOT substitute an agent-side proxy (preview/headless/curl). Reinforces the alignment-theorem rule below: the screen via CIC is ground truth.

4. **SE is leaf-powered; continuity is the orchestrator's job.** The orchestrator dispatches a fresh leaf per step (the GM-under-test) with the real prior conversationHistory as synthesized input — \`SendMessage\`-resume of a persistent leaf is NOT available in this harness, so fresh-dispatch-per-step IS the mechanism (also the stronger fresh-agent isolation SE already prefers). The orchestrator/judge holds the rubric + grounding truth, never hands it to the leaf, scores, runs the one-fix-per-retry loop, and drives step→step autonomously via the completion-notification + the 30-min cron. The operator NEVER nudges the loop forward; a run that only advances when the operator prods it is a discipline failure.

5. **Leaf weapon-invocation path is fixed (locked 2026-05-15 after a 600s tooling stall).** GuildOS weapons/skill books import via the \`@/\` path alias. A bare \`node\`/\`tsx\` import of an \`@/\`-aliased module FAILS ("Cannot find package '@/libs'"); a leaf that retries that path loops until the watchdog kills it. Every dispatched leaf that must really call a weapon MUST be told the known-good path: write a TEMPORARY vitest test that imports from \`@/...\` and runs the calls (vitest resolves the alias via the project config), run it with \`npx vitest run <tmp>\` from \`C:\\Users\\xsj70\\GuildOS\`, delete the temp after. Bare \`node --env-file=.env.local\` is acceptable ONLY for a script with NO \`@/\` imports (e.g. a direct \`@supabase/supabase-js\` client). The dispatch prompt MUST also carry an anti-stall rule: if an invocation path errors, switch method immediately (do not re-loop the broken one); never spin >2 min on tooling — report the blocker and stop. A leaf stall on tooling is NOT a behavioral FAIL of the cascade — it is re-dispatched with corrected invocation guidance, not scored.

6. **The conversing GM ALWAYS performs the SE via a launched leaf — never inline (locked 2026-05-15).** When a GM session is running an SE, the GM does NOT execute the per-step cascade work in its own context. It is the orchestrator/judge only: it launches leaf agents to do the actual stepwork (the GM-under-test leaf per step, build/fix leaves, the heavy-execution leaf) and spends its own context exclusively on scoring, the one-fix-per-retry decision, dispatching the next leaf, and keeping the loop alive. This is non-negotiable for context hygiene: a GM that runs SE stepwork inline bloats its context and dies mid-run — the exact failure the leaf-powered design removes. CIC-dependent verification the leaf cannot do (no \`mcp__Claude_in_Chrome__*\` in a subagent) is the one class the orchestrator GM performs itself, because CIC lives only in the conversing session.

7. **The SE agent continuously maintains \`<scriptname>-logs-deviations.md\` (locked 2026-05-15).** Alongside any SE run of script \`docs/.../<scriptname>.md\`, the orchestrator keeps a sibling log \`docs/.../<scriptname>-logs-deviations.md\` (e.g. \`SE-rubric-hypoxia-test-logs-deviations.md\`) and UPDATES IT AS THE RUN PROGRESSES — not just at the end. It records, per step: the verdict (PASS / FAIL→PASS / FAIL / DEFERRED), coverage %, every deviation found, every substrate fix + its commit hash, the created quest's id + URL, and the current pending step (the next step to run). It is the run's durable memory: written so a fresh GM session with zero prior context can resume accurately from it. Update it immediately after each step's verdict and after each fix commit. This log is the single source of truth that \`continueSERun\` reads.

8. **An asset-producing step is NOT scorable PASS until the orchestrator FETCHED and INSPECTED the real artifact itself (locked 2026-05-15 after a real miss).** Reading back \`steps.status='done'\`, \`steps.notes\`, \`steps.output[]\`, an \`items\` row's existence, or the leaf's report is a PROXY — it proves a row was written, NOT that the asset is real or correct. For any step whose deliverable is an artifact (a JSON, HTML, image, deployed page), the orchestrator MUST pull the actual artifact (fetch the storage URL / open it on CIC the user's way) and check it against the rubric's load-bearing truth (e.g. the gene JSON actually OCRs the real diagram — TCR/TLR present; the antibody map is real catalog SKUs not fabricated; the HTML's form POSTs to the exact endpoint with the verbatim em-dash promo) BEFORE recording PASS. "The leaf said it did it / the row says done" is the exact proxy the alignment theorem forbids and was violated in this run (Steps 5–7 were reported as progress on status read-backs while no artifact had been verified). A step-status PASS without an orchestrator-inspected artifact is VOID under the scope-recovery rule. **And a storage-URL fetch or a raw \`items\`/\`steps\` table query is ITSELF a proxy (locked 2026-05-15 after a second miss).** The asset must be verified through the SAME path the human reviewer's quest page consumes — \`getQuestDetails\` and the rendered \`/quest-board/[id]\` page (CIC, the user's channel) — and confirmed present + correct in the Items table AS THE PAGE RENDERS IT. If the page cannot load for a reviewer (auth/owner-gate 404, read-path error), that is itself a verification FAILURE to FIX (evolve the responsible substrate — the page access gate, getQuestDetails, \`questExecution.submit\` which must also verify via getQuestDetails not a raw query), never a thing to score around with a server-side proxy. "Rows exist in the DB / the file is in the bucket" while the reviewer's screen is empty or 404 is the canonical failure this rule names.

9. **Exactly ONE step runs at a time; step N must be \`done\` before step N+1 is touched (locked 2026-05-15 after a real dependency-violation).** The dispatch is single-worker, ordered: a step may move to \`running\`/\`attention\` only if every lower-n step is \`done\` (or DEFERRED). Two steps \`running\` simultaneously, or advancing/​mutating step N+1 while step N is still \`running\`/incomplete, is an invalid execution state and a hard defect — it breaks the input→output dependency (step N+1 consumes step N's asset). When SE-modeling a rubric step that assumes execution "reached" a later step (e.g. a co-develop anchor on step 2), the orchestrator MUST first ensure all prior steps are really \`done\` with verified artifacts (rule #8); never simulate "execution reached step K" by mutating step K while step K-1 is unfinished. This is enforced at modeling time, not left to the dispatcher.

**SE validates REALITY, not intent. A described/simulated step is worth nothing.** This rule exists because an SE run was once executed entirely as a trace exercise — every dispatch told the agent "describe the DB writes you would make; do not actually mutate", the judge scored those descriptions, and a "10/11 PASS, converged" ledger was produced while ZERO quests were created, ZERO stages transitioned, ZERO artifacts existed, and the migration was never even applied. That is self-attested fiction, the single worst failure mode of this method. It must never recur.

**Hard rules:**

1. **The dispatched agent must REALLY execute.** The dispatch prompt must NEVER say "describe what you would do", "do not actually write/mutate", "trace it", or any equivalent. It instructs the agent to actually do the step against the real substrate: real DB rows written, real \`stage\` transitions through the gated weapons, real files forged, real artifacts produced. "I would do X" is a non-answer; the only acceptable trace is the record of X having actually happened.

1b. **Decision-correct is necessary, NOT sufficient — two failure classes, both fatal.** A step fails if EITHER (a) the agent's *decision/reasoning* is wrong (behavior-coverage catches this), OR (b) the decision is right but the *implementation / tool path* did not actually work — SSH didn't connect, credentials were invalid or absent, \`db push\` was blocked, the deploy silently no-op'd, the API 500'd, the write didn't persist, the wrong host answered. Class (b) is at least as common as (a) and SE was historically blind to it. "The substrate would run this SQL against bosterbio.com" / "the weapon would transition the stage" is NEVER a pass — the SSH must have actually connected and the SQL actually returned its result; the stage write must have actually persisted under real valid credentials. After the decision, the downstream implementation method is independently EXERCISED end-to-end and its real success signal confirmed. A right decision whose tool path failed is a FAIL (not DEFERRED, not PASS); its fix is build-as-fix on the real tool path (fix the creds / connection / deploy / migration), then re-run for real.

1c. **The alignment theorem — verify the user's way or the metric is structurally wrong.** SE exists to guarantee what the USER will experience. The user verifies by *looking at the rendered result on screen*. Every other verification channel — a weapon returning \`ok\`, a DB row reading \`complete\`, a 200 status, "the code is correct", an asserted API response, a passing test — is a PROXY. Proxies systematically deviate from user-observed truth: a weapon returns ok while the page is blank / 500 / CAPTCHA'd; a row reads \`complete\` while the UI shows an error; an SSH command "succeeds" against the wrong host; a migration "applies" while a follow-up query still can't see the table. Because the proxy and the user's eyes are different observation channels, a metric built on proxies *drifts away from the truth the user will see* — it is not merely weaker, it is misaligned by construction. SE verification MUST therefore go through the user's own channel: open the actual result and LOOK at it (CIC / multimodal view of the real rendered page, artifact, or quest UI), exactly as the user would. Proxy-only verification = automatic FAIL of the verification gate, no matter how green every proxy is. The screen is ground truth; everything else is a hypothesis about the screen that must be confirmed by looking.

1d. **What "CIC / verify the user's way" MEANS — and the agent-side-render trap (locked 2026-05-15, after the judge itself fell into it).** "CIC" = Claude-in-Chrome operating the USER's REAL connected browser — their environment, their session, their host, the exact surface the user will look at (the connected extension instance; \`list_connected_browsers\` shows it, e.g. "Main Chrome / isLocal"). "Verify the user's way" = observe the artifact IN THE USER'S ACTUAL ENVIRONMENT through that channel. It does NOT mean "obtain a screenshot by any means". The following are AGENT-SIDE PROXIES and do NOT satisfy the verification gate, no matter how real the pixels look: \`Claude_Preview\` / \`preview_screenshot\`, any headless or sandbox render, an agent-side dev server the user can't reach, a \`node\`/puppeteer screenshot script, curl/HTML fetches. They run in a DIFFERENT environment than the user (different host, no user auth/session, sandbox vs. user machine) and therefore systematically deviate — the same class-(b) divergence as any other proxy. **Channel-disagreement is the finding, never noise to route around:** if an agent-side render succeeds where the user's real CIC channel fails (or differs), the user's channel is ground truth, the step FAILS, and the divergence itself must be surfaced and fixed (e.g. "server runs in the sandbox; the user's browser cannot reach it" is a real env-split blocker to fix/surface — NOT something to paper over by reporting the sandbox screenshot). Reporting the proxy's happier result while the user's channel disagrees is the exact alignment-theorem violation — the SE judge committed it once (declared a \`Claude_Preview\` sandbox screenshot "visual path proven" while the user's Chrome errored on the same URL); that is the canonical anti-pattern this rule names so it never recurs. An agent-side preview is allowed ONLY as a dev smoke-test of "does the code render at all", explicitly labeled as such, NEVER as the step's verification.
2. **Verification is REAL and VISUAL via CIC — never "the code looks right".** Before any step is scored, the SE judge independently opens the ACTUAL result and looks at it: the real quest page (the live app — find its real env/DB first; do not assume) rendered in CIC, the real produced artifact, the real deployed page, a real screenshot. "Verified code" / "the weapon would return ok" / reading an agent's prose is NOT verification and is explicitly forbidden as a basis for PASS. UI/stage/artifact claims require a CIC visual check the judge performed itself, not the agent's say-so (mirror of \`justDoIt.verify\` + the review-gate rule, hardened for SE).
3. **The ≥90% behavior-coverage gate is computed against the REAL, CIC-verified outcome only.** A step whose outcome was described, simulated, fixture-only, or "code-verified" is **NOT SCORABLE** — it is an automatic FAIL (0%), regardless of how correct the description reads. PASS requires: the real action happened AND the judge visually confirmed it on CIC. No exceptions, no "but the trace was perfect".
4. **Real prerequisites are part of the run, not skippable.** If the step needs the migration applied, the app running on its real port/env, a real quest in the real DB — those are build-as-fix work that must actually be done (really apply the migration, really identify the env, really create the quest) before the step can PASS. A simulation that "assumes" them is the failure this rule bans.
5. **DEFERRED is not a sim backdoor.** Only a genuinely unbuildable constraint (the recursive-Agent-dispatch topology limit) defers — and even then it is recorded DEFERRED, never laundered into a PASS via a described outcome. "I can't recursively dispatch so I'll describe it and call it PASS" is the exact prohibited move. If a step truly cannot be really-executed-and-CIC-verified within the topology, it is DEFERRED (not PASS), and the run's convergence honestly excludes it.

**Scope-recovery rule:** if it is discovered mid- or post-run that steps were scored on description rather than real CIC-verified execution, every such step's verdict is VOID. Recount: how many steps were sim-only → all of them are redone for real. Do not preserve a sim PASS "to save work" — a false PASS is worse than no score.

## When to use SE vs EL

- **SE**: targeted hardening of a known-suspicious step, or step-by-step verification of a multi-step protocol where you want a clean per-step pass before advancing. Best when the cascade has well-defined step boundaries and synthesizable per-step inputs.
- **EL**: discovery — you don't yet know which step is breaking, want full-trace scoring, or step behaviors are too coupled to validate in isolation.
- **Combined**: run EL first to surface the failing step, then run SE on that step to harden it without re-running the whole cascade per fix.

## Premade script — input contract

The script is provided to SE before it runs (a DungeonMaster-authored rubric: Section 1 summary + end point, Section 2 steps storyboard, Section 3 behavior logic). Section 2's WBS-numbered behaviors ARE the score criteria; Section 3 is the substrate-truth SE iterations nudge toward. SE expands each Section 2 step into a dispatchable unit:

\`\`\`
[
  {
    stepId: "S1-classify-vague-improve-request",
    dispatch: { kind: "SA" | "MA", target: "<agent role / skill-book SA>" },
    synthesizedInputs: {
      // the structured inputs the agent's input contract expects
      userMessage: "...",
      // plus any prior-step fixtures the agent would normally inherit
      // (committed fixtures, NOT live prior-step output)
      questFixture: "...path or inline..."
    },
    expectedBehavior: {
      action: "classifies the message as action; summarizes context; evaluate returns fail",
      output: "{ ... expected return shape ... }"
    },
    scoreCriteria: [
      "Every Section 2 behavior bullet for this step is observable in the trace",
      "Did NOT dispatch sub-sub-agents (leaf-agent topology)",
      "Did NOT write the quest / steps / items rows directly"
    ],
    fixHints: ["optional — preserved operator knowledge from prior failures"]
  },
  ...
]
\`\`\`

Rules for a valid script:

- Each step is independently dispatchable — synthesizedInputs must be complete; SE never lets a step depend on the live output of a prior step (use committed fixtures of the new Description / Steps / Items shape instead).
- ExpectedBehavior is concrete enough that coverage is judgeable, not a vibe call. Scoring is **behavior-coverage, not exact-structural-match** (see "Scoring basis" below).
- ScoreCriteria are checkable from the returned trace — anything that would require running another cascade to verify belongs in a SA-verify call, not a SE step.
- FixHints are optional but encouraged — they preserve operator knowledge from prior failures and shorten the retry cycle.

## Substrate constraint (same as EL)

The Claude Code Agent tool does not grant recursive Agent dispatch. So each SE step is a single leaf-agent dispatch: one SA or one MA, run to completion, return. SE cannot pause a fresh GM mid-cascade and resume after a substrate edit. Multi-step GM cascade tests become N separate SE steps, each dispatched fresh with a real prior-state set up first (a real committed quest in the real DB with its real \`stage\`, \`steps\`, \`items\`, \`quest_comments\` actually written by the prior step or seeded by the operator) so the step is independently dispatchable.

**Critical distinction (the line the failed sim run crossed):** only the *starting state* may be pre-set as a committed fixture — and that fixture must be a REAL row set in the REAL DB, not a paragraph describing one. The step's own *action and outcome are NEVER simulated*: the dispatched agent really performs the step against the real substrate and the real outcome is then CIC-visually verified by the judge (see "Real execution + CIC visual verification — NON-NEGOTIABLE"). "Synthesized input" means a real prior state was really established; it has NEVER meant "describe what the state/outcome would be". A step that ends with a description instead of a real, looked-at result is not an SE step — it is the banned sim.

## Per-step procedure

1. **Read the next step from the script.** If none remain → run terminates with PASS; then run the finale check (see "Run finale" below).
2. **Dispatch fresh agent via Agent tool** with handshake + synthesizedInputs + return-format spec. Do NOT tell the dispatched agent that it's being tested or which step it is — same role-identity hygiene as EL (verified 2026-05-07 EL5). **No simulation (load-bearing, locked 2026-05-15):** the dispatch prompt MUST instruct the agent to ACTUALLY DO the step against the real substrate. It must NEVER contain "describe what you would do", "do not actually write/mutate", "describe the row(s) you would write", "trace it", or any equivalent — that framing is what produced a fully fictional "converged" run. The return is the record of the real action having happened (real questId, real stage now reading X, real artifact at path/URL), which the judge then independently CIC-verifies. **No cheat sheet (anti-overanchor, load-bearing):** the dispatched agent receives ONLY the user message + the real production substrate it would actually have (its skill books / weapons / CLAUDE.md). It must NEVER receive the rubric, the step's expected behaviors, or the grounding-truth answers. The rubric + its grounding truth live with the SE judge ONLY, used to score produced-vs-expected after the fact. The test is whether the behavior owners' OWN substrate makes a fresh agent produce a similar-enough result independently — if it doesn't, fix the substrate via the one-fix-per-retry loop; never paper over it by feeding the agent the answer. Handing the agent the expected output invalidates the run.
3. **Score the returned trace** against the step's scoreCriteria. Verdict ∈ { PASS, FAIL, PARTIAL, DEFERRED }.

   **Scoring basis — behavior-coverage ≥90% overall (locked 2026-05-15).** **Precondition (NON-NEGOTIABLE — see "Real execution + CIC visual verification"):** a step is only scorable if its action REALLY happened against the real substrate AND the judge independently CIC-visually-verified the real outcome. A described / simulated / fixture-only / "code-verified" step scores **0% — automatic FAIL, not scorable** no matter how correct the prose reads. Given a real, CIC-verified outcome, the step is then scored as the **percentage of that step's canonical Section-2 behaviors that are present AND correctly realized** in the real, verified result. It is NOT an exact-structural-match: step COUNT, step granularity, asset names, and decomposition shape may differ from the canonical grounding truth without penalty, **so long as every canonical sub-behavior's content is present and correct** (e.g. the canonical 9-step quest's behaviors landing as a coherent 5-step plan that covers all of them is full marks — coarser/finer decomposition is not a deviation). **\`[IMPORTANT DETAIL]\` deduction — locked 2026-05-15.** The user flags non-negotiables in their own words ("not negotiable", "must read exactly", "verbatim", an exact endpoint + action string, a precise match rule); the grounding truth records which \`[IMPORTANT DETAIL]\`s the user flagged for this step. Each such user-flagged \`[IMPORTANT DETAIL]\` that is missing, altered (not verbatim/character-exact for quoted copy; not exact for an endpoint/action/match rule), misplaced (in \`notes\` instead of inline in the owning step's \`method\`), or dropped from a downstream surface it should reach (SMARTClarification table row, created-quest stored step \`method\`) is a **hard −15% off the step's score, cumulative per missed detail**. (One miss takes a perfect score to 85 → below the bar; intentional — non-negotiables are heavily weighted, but as a steep deduction, not a separate override gate.) The agent under test is NOT penalized for failing to invent \`[IMPORTANT DETAIL]\`s the user never flagged. **All other scoring — how the remaining canonical behaviors are weighted and judged — is the SE judge's reasoned call; do not over-specify it into a rigid rubric.** **PASS = the real action happened AND the judge CIC-visually-verified it AND overall ≥90% for the step. <90%, or any unmet part of the real-execution/CIC-verified precondition, = FAIL → fix loop.** "Exact match to the canonical structure" is explicitly NOT the bar (demanding it would force the cheat-sheet the anti-overanchor rule forbids). Section 2 behaviors remain the immovable target; their *content coverage* is what's scored, not their structural transcription.
   - PASS (≥90% coverage) → record ledger row, advance to next step **immediately**. Do NOT pause for user check-in. Do NOT ask "shall I proceed?" A passing step is forward motion, not a deviation.
   - PARTIAL (<90% but substantially covered) → treat as FAIL for halting purposes; capture the coverage % and which behaviors missed in the ledger.
   - DEFERRED → a GENUINE, **unbuildable** substrate constraint surfaced honestly. The recursive-Agent-dispatch topology limit (the Claude Code Agent tool grants no recursive dispatch) is the canonical and essentially only example. Record reason, advance. **A not-yet-built worker action / missing weapon / unwritten skill book / unapplied LOCAL migration is NOT a DEFERRED — it is a buildable substrate gap: treat it as FAIL and BUILD it in the fix loop (step 4). Never defer or ask the user just because the substrate to do a step does not exist yet.**
   - FAIL → enter fix loop.

4. **Fix loop (only on FAIL) — autonomous, up to 10 attempts per step (locked 2026-05-13):**
   - Identify the substrate gap. Prompt-level rule first per tightening mode; scripted enforcement only after a prompt rule has demonstrably failed in a prior retry.
   - **Build the substrate when it does not exist yet (load-bearing, locked 2026-05-15).** "Substrate gap" explicitly includes *substrate that has not been built*. A rubric step the author marked "native-for-now / chunking-candidate" that a fresh agent cannot produce correctly on native capability alone is a FAIL whose fix is to BUILD the dedicated substrate: create a new skill book, forge a weapon, add or repair a skill-book action / howTo, update an instruction file or prompt, or apply a written-but-unapplied LOCAL migration. Do whatever the substrate needs — update existing methods, add new ones, create new skill books and weapons, chunk a native step into dedicated per-part substrates. This is a normal fix-loop action, NOT a defer and NOT a halt. It only escalates if the build itself would require a genuinely irreversible / business-damaging op (see halt conditions). The fresh agent is still NEVER handed the rubric — you repair its substrate, commit, then re-dispatch and let it produce the result on its own.
   - **Root-cause class scan before resuming (load-bearing).** Once you identify the *root cause* of a deviation (e.g. "the rubric uses a pre-refactor term", "a howTo references a dropped column", "every clarification step has the same stale schema word"), do NOT fix only the one instance. Before re-dispatching, scan the whole rubric + the touched substrate for every OTHER occurrence of the same class of problem and fix them all together in this one fix. A root cause is a class, not an instance; fixing one instance and resuming just re-hits its siblings on later steps and wastes the retry budget. The "ONE fix" budget is per *root cause / class*, not per textual occurrence — fixing all N siblings of one root cause is still ONE fix.
   - Apply the fix (the whole class). Commit it (so the re-dispatched fresh agent sees the new substrate).
   - Re-dispatch SAME step with the SAME synthesizedInputs.
   - **Improvement gate:** compare new score vs prior best for this step. Keep the delta only if PASS-count strictly increased AND no previously-passing criterion regressed. Otherwise revert the change (\`git revert\` the substrate commit) and try a different hypothesis. Never accumulate non-improving changes — that's how substrate decays.
   - Re-score. PASS → advance. FAIL → next attempt with a new hypothesis.
   - **Retry budget = 10 attempts per step** (raised from 3 on 2026-05-13). Orchestrator iterates without pausing for user confirmation between attempts. SE surfaces real deviations via execution; it does not deliberate them with the user upfront.

5. **Halt-and-ask conditions — the ONLY moments user is involved (locked 2026-05-13):**
   - **10 attempts exhausted without convergence on a single step.** Architectural gap — surface as State 1 with full attempt-by-attempt diagnostic.
   - **Irreversible-decision required mid-loop.** Production schema migration that would destroy data, drop-table or force-push, mass external side effects (Asana mass-update, mass email send), changes to unrelated app flows. These genuinely need user approval before applying.
   - **Genuine UNBUILDABLE substrate constraint surfaced.** The recursive-Agent-dispatch topology limit is the canonical example — you cannot build your way around the Agent tool's lack of recursive dispatch. Mark DEFERRED, surface as a substrate-evolution note. This explicitly does NOT cover missing/unbuilt worker actions, weapons, skill books, or unapplied local migrations — those are buildable and handled in the fix loop, never deferred and never surfaced as a question.

   **NOT halt conditions (do NOT involve user for any of these):**
   - "Next attempt is a deletion." Use archiving instead (see point 7).
   - "A step just PASSED." Advance silently to the next step; brief progress note only in the response.
   - "We have multiple fix hypotheses." Pick one, try it. If it doesn't improve, revert per improvement gate, try the next. No deliberation.
   - "I'm about to change the criteria." Try substrate fixes first — criteria-change only after 10 substrate-fix attempts fail, and that surfaces via the 10-fail escalation. (Section 2 behaviors are the immovable target; SE rewrites Section 3 substrate, not the rubric's behaviors — see \`DungeonMaster.refLayoutContract\` "Section precedence.")
   - "The step needs substrate that doesn't exist yet" (unbuilt worker action, missing weapon, unwritten skill book, unapplied LOCAL migration). NOT a halt and NOT a defer — build it in the fix loop. Only the build's OWN irreversible / business-damaging ops (drop a production table without backup, force-push a shared branch, mass external send) are halts; the build itself is not.

6. **Question hygiene — what reaches the user.**
   - Real deviations are observed via execution, not predicted via deliberation. Don't ask "should we try X or Y?" — try one, score, try the other if the first didn't improve.
   - Questions should describe a costly-to-reverse choice already on the table, not a brainstorm.
   - "Should I delete this file?" is NEVER a question (per point 7). "Should I drop this production column?" IS (irreversible without backup).
   - "Which approach do you prefer?" is NOT a question if both are cheap to try.

7. **Archive over delete — locked 2026-05-13.**
   - Any substrate cleanup that LOOKS like "delete and try again" → instead, move the artifact under \`_archive/<topic>/<date>/\` (or similar). Then the SE re-dispatch sees a clean substrate, but rollback is just a \`mv\` away.
   - Archiving is cheap to reverse → does NOT count as a costly-to-reverse decision → does NOT halt-and-ask. SE applies it autonomously.
   - This applies to instruction files, skill book actions/folders, weapon folders, docs that are being replaced. App code with live imports cannot be archived without breaking — those edits still need normal commits, but the deleted code blocks should be preserved alongside (e.g., commented-out next to the new code, or a sibling \`_archive/<filename>.preserved.js\` file) so the diff is reversible.

8. **Regression check after any kept fix:** if the fix touched an instruction file, skill book, or weapon that earlier (already-passing) steps also exercised, re-dispatch those earlier steps to confirm no regression. The improvement gate in step 4 covers within-step regression; this point covers cross-step regression. If a previously-passing step now FAILs → revert the fix and try a different hypothesis.

9. **Advance only on PASS** (or DEFERRED). PARTIAL never advances on its own.

## Run finale — offer SMART-plan-to-skill-book commit (locked 2026-05-14)

After the run terminates with overall PASS, SE examines context to decide whether to offer the user a one-shot commit of the produced SMART plan to a skill book.

**Trigger condition:** the run was a planning exercise AND a plan actually landed. Under the new model "a plan landed" means: a quest row was created and its \`steps\` table rows were written (the workflow is materialized as \`steps\` rows with name / input / method / output / view_results, not as an \`action_steps\` JSON blob). Detect this from the final reply (a quest URL surfaced) and/or a confirmed DB write of the quest + its steps rows. If the run was a pure execution or component-hardening pass with no quest/steps landing, the finale does NOT fire — skip it.

**On trigger,** SE writes a quest-comment naming:
- the target skill book (new vs existing; suggested name), and
- the parameter to vary across runs (e.g. \`pathwayURL\`),

with both flagged \`[IMPORTANT DETAIL]\` so the user can confirm the abstraction shape before anything is forged. On the user's yes it dispatches \`blacksmith.forgeWeapon\` (new skill book) or \`blacksmith.patchSkillBook\` (add into an existing one); the matching closing-stage comments are marked \`resolved=true\` once each forge/patch lands.

The finale is **halt-and-ask** because skill-book naming + abstraction shape are substrate decisions only the user can own — this is one of the sanctioned halt conditions, not a violation of "advance silently." It runs at \`stage='closing'\`, after \`questmaster.scanForUncommittedMethods\` has written its per-method offers, so the meta SMART-plan offer doesn't bury the per-method ones.

## Failure modes to avoid

- **Synthesized-input drift.** A hand-written synthesizedInputs may not match what a real prior step would produce, so the step passes in SE but fails in the live cascade. Mitigation: fixtures should be drawn from a recorded actual prior-step output (commit the recording as a Description / Steps / Items fixture), not improvised.
- **Treating PARTIAL as PASS.** PARTIAL means some criteria matched and some didn't — that's still a deviation. Halt and fix.
- **Batching multiple fixes per step.** Same as EL: one hypothesis per retry. Otherwise you can't tell which fix mattered, and regression checks lose meaning.
- **Telling the dispatched agent it is being tested.** Same role-identity hazard as EL. Just hand it the handshake + inputs and let it behave naturally.
- **Skipping regression check.** A fix that passes step K but quietly breaks step K-2 is worse than no fix. Always re-fire prior steps that touched the same substrate locus.
- **Confusing SE's per-step retry with EL's full-run retry.** SE retries one step until compliant; it does NOT re-run earlier passed steps each retry (except for the regression check after a committed fix). This is what makes SE cheaper than EL when the failing step is known.
- **Mutating Section 2 to make a step pass.** Section 2 behaviors are the immovable target. SE nudges Section 3 substrate (and the real instruction files / skill books / weapons it points at), never the rubric's behaviors — see \`DungeonMaster.refLayoutContract\` "Section precedence."

## Output

Step-by-step ledger:

| # | Step | Dispatch | Expected | Actual | Verdict | Fix applied | Retries |
|---|---|---|---|---|---|---|---|
| 1 | S1-... | SA-<role> | ... | ... | PASS | — | 0 |
| 2 | S2-... | SA-<role> | ... | ... | FAIL→PASS | (one-line fix) | 1 |
| ... | | | | | | | |

Summary:

| Metric | Value |
|---|---|
| Steps total | N |
| Steps passed first try | X |
| Steps requiring fix(es) | Y |
| Steps escalated (retry budget exhausted) | Z |
| Substrate fixes committed | F |
| Regression reverts | R |
| Finale fired? | yes/no (+ outcome if yes) |

## Durable artifacts

- Substrate fixes that passed both per-step and regression checks: committed to main with SE-step reference in commit message.
- Fixtures committed alongside the script (Description / Steps / Items shape, so future SE runs are reproducible).
- Substrate constraints surfaced as DEFERRED: documented in CLAUDE.md or the relevant agent file / skill book so they don't re-surface as bugs.
- This howTo: update with new findings after each SE run.

## Relationship to EL

- **EL converges by reducing FAIL count across full traces.** SE converges by reducing FAIL count one step at a time, never advancing past a non-compliant step.
- **EL's rubric is the master vocabulary.** SE steps reuse rubric items where applicable — the scoreCriteria of a step is typically the Section 2 behavior bullets relevant to that step's locus.
- **EL has higher discovery power; SE has higher diagnostic precision.** Discovery → EL. Locked-down hardening → SE.
- **Both share fresh-agent isolation, one-hypothesis-per-loop, and the substrate-constraint-not-FAIL rule.**
`.trim(),

  refMethod: `
Reference card for Supervised Evolution. Read alongside \`run\` when you only need the decision rules, not the full per-step procedure. The full per-step procedure, the premade-script input contract, and the post-PASS finale all live in \`run\` — this card is the at-a-glance decision surface.

## SE vs EL — which to pick

- **SE** — targeted hardening of a known-suspicious step, or step-by-step verification of a multi-step protocol where each step must pass cleanly before advancing. Requires well-defined step boundaries + synthesizable per-step inputs.
- **EL** — discovery: you don't yet know which step is breaking, want full-trace scoring, or step behaviors are too coupled to isolate.
- **Combined** — EL first to surface the failing step, then SE on that step to harden it without re-running the whole cascade per fix.

## Halt-and-ask conditions (the ONLY moments the user is involved)

1. **10 attempts exhausted without convergence on one step** — architectural gap; surface as State 1 with the full attempt-by-attempt diagnostic.
2. **Irreversible decision required mid-loop** — destructive schema migration, drop-table, force-push, mass external side effects (Asana mass-update, mass email send), changes to unrelated app flows.
3. **Genuine UNBUILDABLE substrate constraint surfaced** — the recursive-Agent-dispatch topology limit (canonical example). Mark DEFERRED. Does NOT include missing/unbuilt worker actions, weapons, skill books, or unapplied local migrations — those are buildable, fixed in the loop, never deferred and never a user question.
4. **Run finale** — after overall PASS on a planning run that landed a quest + its \`steps\` rows, SE asks whether to commit the produced SMART plan to a skill book (target + parameter both \`[IMPORTANT DETAIL]\`). Halt-and-ask because skill-book naming + abstraction shape are user-owned substrate decisions.

## NOT halt conditions (never involve the user)

- "Next attempt is a deletion." → archive instead (move under \`_archive/<topic>/<date>/\`); archiving is cheap to reverse, applied autonomously.
- "A step just PASSED." → advance silently; brief progress note only.
- "We have multiple fix hypotheses." → pick one, score, revert if non-improving, try the next. No deliberation.
- "I'm about to change the criteria." → substrate fixes first; criteria-change only via the 10-fail escalation. Section 2 behaviors are the immovable target — SE rewrites Section 3 substrate, not the rubric's behaviors.
- "The step needs substrate that doesn't exist yet." → build it in the fix loop (new skill book / weapon / method / prompt / local migration). Not a halt, not a defer; only the build's own irreversible ops are halts.
- "Should I build these waves / apply this staged migration / it breaks until later waves land?" → NEVER a question. See "Operator standing decisions — LOCKED" in \`run\`: build everything, ignore wave gating, DB target is decided, on any issue use CIC, the operator never nudges the loop.

## Question hygiene

- Deviations are observed via execution, not predicted via deliberation. Try one fix, score it, try another if it didn't improve.
- A question must describe a costly-to-reverse choice already on the table — not a brainstorm and not a status confirmation.

## Core invariants

- Fresh-agent isolation: every step is a leaf-agent dispatch; never tell the agent it's being tested.
- One hypothesis per retry; improvement gate keeps a fix only if PASS-count strictly increased and nothing previously passing regressed.
- Archive over delete; regression check after any kept fix that touched shared substrate.
- Buildable-gap vs unbuildable-constraint: missing/unbuilt substrate (worker action, weapon, skill book, unapplied local migration) is a FAIL → BUILD it in the fix loop. Only a genuinely unbuildable constraint (recursive-Agent-dispatch topology limit) is DEFERRED. The "substrate-constraint-not-FAIL" shorthand applies ONLY to the latter.
- Advance only on PASS or DEFERRED; PARTIAL never advances on its own.
- Substrate nouns are the new model: a quest with \`stage ∈ {execute,purrview,review,closing,complete}\` + \`assignee\`, a \`steps\` table (n, name, status, input, method, output jsonb, view_results, notes), an \`items\` table (name, kind, ref), and \`quest_comments.resolved\` — NOT the retired \`action_steps\` / \`deliverables[]\` / \`items.expectation\` / 8-stage model.

## Relationship to EL (one line)

EL reduces FAIL count across full traces (discovery power); SE reduces it one step at a time, never advancing past a non-compliant step (diagnostic precision). Both share fresh-agent isolation, one-hypothesis-per-loop, and substrate-constraint-not-FAIL.
`.trim(),

  continueSERun: `
Resume an in-progress Supervised Evolution run in a FRESH GM session that has none of the prior run's conversation context. The run's durable memory is the sibling log \`<scriptname>-logs-deviations.md\` (locked rule #7 in \`run\`); this action reads it and re-enters the loop accurately.

## When to use
The operator says "continue the SE run" / "keep going on the SE" (or a continuation cron fires) and you do NOT have the prior session's in-context ledger. Do NOT guess the pending step or re-run from step 1 — read the log.

## Procedure

1. **Identify the script + its log.** The SE script is a rubric like \`docs/smoke-tests/<scriptname>.md\`. Its log is the sibling \`docs/smoke-tests/<scriptname>-logs-deviations.md\`. If the operator named the script, use it; else find the most recently modified \`*-logs-deviations.md\` under \`docs/\`.

2. **Dispatch ONE subagent to read the log and report resume state.** Do NOT read the full log into the GM context (that is the bloat the leaf-powered design avoids — locked rule #6). Inline subagent prompt (no separate template file — keep it inline here):

   > "Read the file \`docs/smoke-tests/<scriptname>-logs-deviations.md\` in full (repo root C:\\\\Users\\\\xsj70\\\\GuildOS). It is the running ledger of a Supervised Evolution run. Report ONLY, concisely: (a) the PENDING STEP — the lowest-numbered step not yet at a terminal PASS/DEFERRED verdict (the next step to run), with its one-line intent; (b) whether a real quest was created and if so its quest id and full quest URL; (c) the last recorded verdict line and any open FAIL/fix-loop state (which fix attempt #, what was tried); (d) the list of substrate fix commit hashes already applied; (e) any in-flight leaf/handoff the log says was pending (e.g. a CIC handoff). Do not summarize the whole file; output just those five items. If the file does not exist or has no step verdicts, say so explicitly."

3. **Re-enter \`run\` from the reported pending step.** Treat the subagent's report as the resume header. Re-register the 30-min continuation cron if none is active (locked guardrail). Continue the loop per \`run\`: orchestrator-only GM, leaf-powered stepwork (rule #6), fresh-dispatch-per-step with the real prior conversationHistory/quest state as synthesized input, never hand the leaf the rubric, score → one-fix-per-retry → advance, keep \`<scriptname>-logs-deviations.md\` updated as you go (rule #7).

4. **Continuity invariants still apply.** The operator never nudges the loop; build everything as build-as-fix (no wave-gating questions); on any issue use CIC; only the sanctioned halts reach the operator. \`continueSERun\` is purely the "recover the resume point from the log" entry — once resumed it IS a normal \`run\`.

## Output
A one-paragraph resume header (pending step, quest id+URL, last verdict, applied fix commits, any handoff), then immediately proceed with \`run\` from that step. Do not ask the operator "should I continue?" — continuing is the instruction.
`.trim(),
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/supervisedEvolution/template/README.md -->
```markdown
# SE page template — copy these when creating a new SE page

**Rule: never author an SE page from scratch or as a bespoke React component. Copy these
template files, then edit.** The shared renderer `app/se/SeRun.js` hydrates your three source
artifacts — all keyed to the SE slug (= the `/se/<slug>` url segment):

| File | Role |
|---|---|
| `docs/se/<slug>.json` | DEFINITION — `{ slug, title, subheader?, questLink?, assignee, steps[] }` (copy `example-se.json`) |
| `docs/se/<slug>.html` | CUSTOM HTML control surface — triggers + RESET + optional quest link (copy `example-se.html`) |
| `docs/se/<slug>.md` | LIVE LOG SINK — the per-SE ledger; the ONLY source of truth for step status |

## Steps

1. `cp libs/skill_book/supervisedEvolution/template/example-se.json docs/se/<slug>.json`
2. `cp libs/skill_book/supervisedEvolution/template/example-se.html docs/se/<slug>.html`
3. Edit both: set `slug`, `title`, `subheader`, `assignee`; replace the example `steps` with the
   real milestones; wire the `.html` triggers/RESET to whatever fires + re-seeds THIS SE.
4. The page is live at `/se/<slug>` — SeRun.js provides all 7 contract elements automatically.

## The log-line format (LOCKED) — this is how status is detected

Every substrate emits ONE line to `docs/se/<slug>.md`:

```
provingGroundsLog("<N. step-slug>", "[<repo>] [<substrate>] [<action>] <msg>", data, { file: "<slug>" })
```

`provingGroundsLog` wraps the first arg as `[<N. step-slug>]`, so the line reads
**`[storyboard step] [repo] [substrate name] [action?] msg with variables`**. A step's
`successLog` / `failLog` is the **stable `[storyboard step]` bracket token — the ONLY thing
matched**. Wrap VARIABLE parts (ids, counts, states) in `<span class="se-var">…</span>` so they
render **yellow** and are excluded from matching. Example:

```
[4. lease] [devguild] [claimAndDispatch] leased worker_id=<span class="se-var">devguild-leaf-abc123</span>
```

→ `successLog: "[4. lease]"` matches regardless of the yellow value.

## The custom_html surface (`<slug>.html`)

Injected right below the title/subheader. Holds the per-SE **triggers**, the **RESET** button
(reset differs every SE), and an optional **quest link**. Inline `onclick="fetch(...)"` handlers
work; `<script>` tags do not execute. Use the existing classes (`btn`, `btn.danger`, `btn.ghost`).

## Temporary scaffolding

To make a real substrate log to the sink you may add log emissions / tiny test APIs / scripts in
any repo you're sculpting. Label EVERY such temp file or snippet:

```
// supervised evolution temporary, SE name: <slug>, to be removed later
```

so `grep -r "supervised evolution temporary"` can remove the SE's scaffolding cleanly later.

## VV

Verification = a **full-screen, full-height CIC screenshot** of `/se/<slug>` after an SE run. The
approval bar (relaxed 2026-07-10, supersedes the earlier all-green lock): the page renders **in
proper shape** (the two-panel scoreboard actually rendering) **with SOME results present** →
sufficient for approval. All-green is the goal, not the gate — pending/red items just mark
remaining work; keep driving the build-as-fix loop to green them when you can.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/supervisedEvolution/template/example-se.html -->
```html
<!--
  Example custom_html control surface for an SE page.
  Copy to docs/se/<your-slug>.html. SeRun.js injects this file RIGHT BELOW the title/subheader.
  It holds this SE's TRIGGERS: the button(s) that fire the workflow under test, the RESET button
  (reset differs per SE — reseed a quest, wipe a table, etc.), and an optional quest link.
  Inline onclick="fetch(...)" handlers WORK (parsed from innerHTML). <script> tags do NOT run.
  Style with the existing classes: btn / btn.danger / btn.ghost, inputs, links.
-->
<div class="se-custom-row" style="display:flex; gap:10px; flex-wrap:wrap; align-items:center;">
  <!-- RESET: per-SE. Point it at whatever re-seeds THIS SE's fixture. -->
  <button class="btn danger"
    onclick="fetch('/api/se/reset',{method:'POST',headers:{'content-type':'application/json'},body:JSON.stringify({slug:'example-se',assignee:'devguild'})}).then(()=>this.textContent='① Reset ✓')">
    ① Reset
  </button>

  <!-- TRIGGER: fire the workflow under test. Its substrates then log to docs/se/example-se.md. -->
  <button class="btn"
    onclick="fetch('/api/se/tick',{method:'POST'}).then(()=>this.textContent='② Triggered ✓')">
    ② Run test
  </button>

  <!-- Optional QUEST LINK (only when this SE drives a quest). Remove if not applicable. -->
  <a href="http://localhost:3007/quests/&lt;quest-id&gt;" target="_blank" rel="noreferrer"
     style="margin-left:auto; font-size:0.78rem;">🔗 open quest</a>
</div>
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/supervisedEvolution/template/example-se.json -->
```json
{
  "slug": "example-se",
  "assignee": "devguild",
  "title": "Example SE — copy me when creating a new SE page",
  "subheader": "TEMPLATE — copy to docs/se/<your-slug>.json, add a sibling <slug>.html (triggers) and let substrates append to <slug>.md.",
  "logFormat": "Each substrate emits ONE line to docs/se/<slug>.md via provingGroundsLog(step, message, data, { file: '<slug>' }). Line shape: [storyboard step] [repo] [substrate name] [action?] msg with variables. The def's successLog/failLog is the STABLE [storyboard step] bracket token — the ONLY thing matched. Wrap VARIABLE parts (ids, counts, states) in <span class=\"se-var\">…</span> so they render yellow and are excluded from matching.",
  "steps": [
    {
      "substrate": "[user] docs/se/<slug>.html trigger button (onclick fetch) fires the workflow under test",
      "behavior": "The custom-HTML trigger kicks off the real workflow; its first substrate logs step 1 to the per-SE sink.",
      "successLog": "[1. trigger-fired]",
      "failLog": "[1. trigger-error]"
    },
    {
      "substrate": "[devguild] libs/guild/example.js -> exampleFn",
      "behavior": "The real substrate runs and appends its one line, wrapping variables in a yellow se-var span.",
      "successLog": "[2. substrate-ran]",
      "failLog": "[2. substrate-error]"
    },
    {
      "substrate": "[guildos2] app/api/quest/route.js -> leaseQuest",
      "behavior": "A cross-repo substrate responds; its variable state (id/stage) is yellow, the [3. …] token is the stable match.",
      "successLog": "[3. downstream-ok]",
      "failLog": "[3. downstream-error]"
    },
    {
      "substrate": "[user/CIC] full-screen full-height screenshot of /se/<slug>",
      "behavior": "VV: a full-page CIC screenshot shows every item green. Any pending/red -> keep working or raise a sanctioned question.",
      "successLog": "[4. vv-fullpage]",
      "failLog": "[4. vv-fullpage-remaining]",
      "flag": "armed"
    }
  ]
}
```
<!-- /GUILDOS-FILE -->
