# butlerPlanning

<!-- GUILDOS-SKILL name="butlerPlanning" -->

<!-- GUILDOS-DESC -->
Strategy planning and objective tracking threads — the user's main strategy thread where initiatives are discussed and objectives tracked before they formalize into quests. Covers: keeping the objectives table, reading notes/status from OTHER sessions/threads token-efficiently, and delegating scans to cheap-model leaf agents. Use for 'strategy planning', 'objective tracking', 'main strategy thread', 'track these objectives', or 'what is the other thread doing'.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/butlerPlanning/SKILL.md -->
```markdown
---
name: butlerPlanning
description: >-
  Strategy planning and objective tracking threads — the user's main strategy
  thread where initiatives are discussed and objectives tracked before they
  formalize into quests. Covers: keeping the objectives table, reading
  notes/status from OTHER sessions/threads token-efficiently, and delegating
  scans to cheap-model leaf agents. Use for 'strategy planning', 'objective
  tracking', 'main strategy thread', 'track these objectives', or 'what is the
  other thread doing'.
---

# Butler Planning

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/butlerPlanning/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto butlerPlanning <action>` (or read the source directly).

## Actions

- **delegateScans** — Scope an objective for tracking: delegate any inventory/survey/scan (branches, worktrees, quests, files, usage) to a cheap-model leaf agent that returns only a compact table row for the objectives tracking table — never run wide scans inline in a strategy planning thread.
- **diagnoseDispatch** — Diagnose why GuildOS cron/dispatch is stalled — quests not getting picked up, a guild (Hermes / main / carbon) not taking jobs, or a manual tick doing nothing. Answers 'cron health check', 'dispatch stalled', 'quests aren't being dispatched', 'is Hermes/main taking jobs', 'try a manual tick'. THE fast path is the caravan TICK API, not reading cron code or running runCron inline.
- **digestOtherThread** — Read another session/thread's notes token-efficiently: what is the other thread doing, what did it find, what is its status. The transcript IS the note — but never load it raw into the strategy thread.
- **dispatchObjective** — What to do when the user ADDS an objective: dispatch a local thread via spawn_task when specific instructions are given, or just track it when not. Direct mode vs quest mode.
- **refTokenEfficiency** — Reference: token-efficiency findings from the 2026-07-03 strategy-thread analysis — which steps to omit and which to push down to cheaper models.
- **trackObjectives** — Objective tracking: maintain the objectives tracking table in a strategy planning thread. Track objectives as table rows; tracking and discussion ONLY — no execution in this thread. THE action when the user says 'keep track of these objectives' or 'add to the objectives table'.
- **trackSIOs** — Maintain the SIO (Substrate Improvement Opportunities) table — improvement ideas observed during objective review, placed BELOW the objective tables. Strict token-saving limits; SIOs are adopted ONLY on user command.
- **watchSubstrate** — Strategy-thread substrate duty: keep the GuildOS server and cron running, and include a small report of recently active cron-launched leaf sessions. The server/cron health summary is PART OF the BCS Health line.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/butlerPlanning/engineering_decisions.md -->
```markdown
# Engineering Decisions — butlerPlanning

## 2026-07-03 — Strategy threads are tracking-only; scans delegate to cheap-model leaves

**Decision:** In a strategy planning / objective tracking thread, the main model never executes objectives or runs wide scans inline — it tracks (objectives table), discusses, and delegates every inventory/digest to a haiku-tier leaf agent that returns a compact digest.

**Rejected alternative:** The native default — the aggressive-proceed rules (Sage WWCD, "just do it") push the main model to execute scoped work immediately in-thread, and it *can* (it did: a ~100-branch inventory ran inline on 2026-07-03).

**Why:** (a) The strategy thread is long-lived; every KB of raw scan output pollutes its context for the rest of its life and is re-paid on every subsequent turn. (b) The main thread runs the most expensive model; extract-and-compress jobs are haiku-shaped. (c) Execution in the strategy thread duplicates sibling threads' work — the 2026-07-03 branch inventory duplicated a cleanup another running session was already executing.

**Consequence-if-violated:** Strategy thread context fills with dead survey output, per-turn cost compounds, and parallel threads collide on the same work (double-deletes, conflicting commits).

## 2026-07-03 — Cross-thread notes = transcript digest, not a new notes subsystem

> **Superseded in part, 2026-07-03 (adopted SIO 2):** the "found by mtime" clause is
> replaced — the mapping is now DETERMINISTIC via the `ccSessions` weapon
> (`node libs/weapon/ccSessions/cli.mjs "<id-or-title>"`), which reads the desktop
> app's session store where `cliSessionId` = transcript filename. Mtime ordering
> survives only as the fallback when that store is unreadable. The load-bearing
> decision below — transcript-as-note, no parallel notes store — stands unchanged.

**Decision:** "Session notes" from another thread are obtained by digesting its transcript jsonl (resolved via the ccSessions weapon; mtime fallback), via a leaf. No dedicated cross-session notes store is built.

**Rejected alternative:** Building a structured shared-notes file/table that every session appends to (leaf-agent-notes style).

**Why:** The transcript already contains everything a notes file would, with zero write-side discipline required from other sessions; a notes convention only pays off if transcript digestion proves too noisy in practice. Don't build the second system until the first one fails.

**Consequence-if-violated:** A parallel notes surface that sessions forget to update becomes a stale, misleading second source of truth.

## 2026-07-03 — SIO sweep extracts BATCH quests per leaf and captures root-cause fields (meta-SIOs)

**Decision:** The quest-review SIO sweep batches ~7 quests per haiku extract leaf (not one leaf per quest), and each digest must capture timestamps/duration, the last comment VERBATIM, and the gate/hook name+version on any bounce/escalate — not just category labels.

**Rejected alternative:** The literal recipe as first written — one haiku leaf per quest, digests carrying only category labels + counts.

**Why:** One-leaf-per-quest is ~3× the dispatch overhead for the same cross-quest visibility (proven on the 2026-07-03 21-quest sweep: 3 batched leaves sufficed). And the label-only digests summarized away the exact data the sonnet judge needed — several quests came back "escalated for unknown reason" because the one comment holding the cause was compressed out; duration was unknowable so impact ranking guessed.

**Consequence-if-violated:** Sweeps cost 3× more at daily cron frequency (the whole point is affordability), and the judge ranks impact blind while missing root causes that live in the last comment / gate version.

## 2026-07-03 — Every adopted SIO records an ED in each affected module

**Decision:** Every SIO dispatch brief instructs the implementing thread to write an engineering_decisions.md entry at the root of each module it touches; inline-applied SIOs get their ED too.

**Rejected alternative:** Letting SIO code changes land with only a commit message (the native default — a spawned thread wouldn't write EDs unprompted for a change it sees as routine).

**Why:** An SIO is by construction a non-native substrate change — it overrides a current default, which is exactly the ED trigger. Without the standing instruction, the drift-prevention rationale for each substrate fix is lost the moment the implementing session ends.

**Consequence-if-violated:** Future agents re-litigate or naively revert substrate improvements whose reasoning was never recorded, re-opening the friction the SIO closed.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/butlerPlanning/index.js -->
```js
/**
 * butlerPlanning skill book — HOW to run a "strategy thread": the user's
 * main planning/objective-tracking conversation where initiatives are
 * discussed and tracked BEFORE they formalize into quests.
 *
 * PURE TEXT REGISTRY: prompt strings only, no imports/functions.
 *
 * THE ONE THING TO KNOW: a strategy thread is a TRACKING-AND-DISCUSSION
 * surface, not an execution surface. The main (expensive) model does
 * judgment + table upkeep only; every scan/survey/digest is delegated to a
 * cheap-model leaf agent that returns a compact digest. See
 * engineering_decisions.md in this folder for why.
 */

export const skillBook = {
  id: "butlerPlanning",
  title: "Butler Planning",
  description:
    "Strategy planning and objective tracking threads — the user's main strategy thread where initiatives are discussed and objectives tracked before they formalize into quests. Covers: keeping the objectives table, reading notes/status from OTHER sessions/threads token-efficiently, and delegating scans to cheap-model leaf agents. Use for 'strategy planning', 'objective tracking', 'main strategy thread', 'track these objectives', or 'what is the other thread doing'.",
  steps: [],
  toc: {
    trackObjectives: {
      description:
        "Objective tracking: maintain the objectives tracking table in a strategy planning thread. Track objectives as table rows; tracking and discussion ONLY — no execution in this thread. THE action when the user says 'keep track of these objectives' or 'add to the objectives table'.",
      howTo: [
        "A strategy thread tracks objectives; it does NOT execute them. If the user states an objective, scope it (cheaply — see delegateScans) and add a table row. Execution happens in dedicated threads/quests.",
        "USER-INSTRUCTION-ONLY ADOPTION (LOCKED 2026-07-03): objectives enter the tracker ONLY on explicit user instruction. Never self-adopt a discovered thread, quest, or piece of work as a tracked objective — however active or relevant it looks. Same for attaching a thread id to an existing objective: only when the user names the thread or tells you to find it.",
        "REPORT FORMAT = BCS (sage.questReport) merged with Thread-id and Notes columns: TWO tables — 'Done' then 'Pending' — each with columns Item | Thread id | Progress/Δ | Status | Notes. Thread id = short transcript id (first 8 hex chars) or '—', annotated with a state marker from ccd list_sessions (one cheap call per report refresh): ● running · ○ idle · ⊗ archived (isRunning/isArchived flags; include_archived:true to see archived). Status labels and the anti-BS Δ rule follow sage.questReport verbatim.",
        "MAIN-THREAD GROUPING (LOCKED 2026-07-03): objectives roll up under a small set (≤~5) of user-defined MAIN THREADS (themes the operator is actually working on, e.g. 'quest-lifecycle troubleshooting', 'quest review + SIO evolution', 'dispatch/update API', 'GuildOS-on-VM'). Each tracked objective/session is a sub-item under exactly one main thread; report as one section per main thread (its rolled-up status + the contributing sub-threads), not a flat list, once the operator has named their themes. The operator OWNS the theme set — adopt/rename only on their instruction.",
        "NUMBERING + MERGED PROGRESS/Δ (LOCKED 2026-07-03, real-estate save): every row's Item cell is PREFIXED with a running number (`1.`, `2.` …) continuous across BOTH tables (Done first, then Pending) so the user can say 'mutate 3' / 'drop 5' and point to a row unambiguously. Renumber each report so the sequence stays gap-free. Progress and Δ are ONE merged column 'Progress/Δ': the standing progress signal + this-turn's change in one cell (e.g. '6/7 · none this turn' or 'shipped · new→done 6bb7708'); the anti-BS rule still applies to the Δ part.",
        "NOTES COLUMN: a persistent one-line orientation cell (key artifact, where things stand, flagged remainder) written from digests ALREADY in context — never spend tool calls or extra reads just to fill Notes. Δ is per-turn change; Notes is standing context. Carry Notes forward verbatim until a digest updates them.",
        "Re-output BOTH full tables whenever anything changes; keep every cell to one line. Keep prose around the tables brief — the tables ARE the deliverable.",
        "DROPPED = GONE (LOCKED 2026-07-03): when the user drops an item, remove it PERMANENTLY — never render it again in any form (no 'dropped'/'⏸' status row, no strikethrough, no 'orphan'/historical mention). Dropped items simply cease to exist in every subsequent report. 'Drop' means drop.",
        "DONE ROWS PERSIST UNTIL USER CONFIRMS (LOCKED 2026-07-03): a ✅ Done row stays in the Done table and keeps appearing in EVERY subsequent report until the user explicitly confirms it (e.g. 'confirmed', 'clear the done ones'). Only user confirmation removes a row — never self-remove a Done row, even across many turns. On confirmation, drop the confirmed rows and note the drop in one line.",
        "When an objective graduates into a quest, record the quest id in Notes and stop tracking detail here — the quest becomes the durable record (CLAUDE.md rule 5).",
      ].join("\n"),
      input: { objective: "string — new/updated objective from the user" },
      output: { table: "markdown — the full updated objectives table" },
    },

    dispatchObjective: {
      description:
        "What to do when the user ADDS an objective: dispatch a local thread via spawn_task when specific instructions are given, or just track it when not. Direct mode vs quest mode.",
      howTo: [
        "WITH specific instructions (LOCKED 2026-07-03): if the user's objective comes with concrete instructions/specs, (1) write a fully self-contained spawn_task brief carrying those instructions (paths, contracts, deliverable formats, verify steps), (2) add the objective row to the BCS tables, thread id = the chip's task_id until the user starts it, then (3) once started, resolve the real thread id via the ccSessions weapon and associate it to the row.",
        "WITHOUT instructions: just add the row with thread id '—'. Do NOT dispatch, scope-hunt, or draft briefs unprompted — the user will supply instructions or kick it off later.",
        "DIRECT MODE vs QUEST MODE (LOCKED 2026-07-03): DEFAULT = direct mode — the spawned thread works the objective directly. QUEST MODE only when the user explicitly mentions doing it in/as a quest: then the spawn brief must instruct the thread to CREATE A QUEST instead of working directly — nail the description + user deliverables and create at stage='plan' per questmaster.clarifyQuest/createQuest (planQuest itemizes objectives; don't hand-author them). Record the quest id in the row's Notes once created.",
        "Deliverables in briefs: always concretely testable (a smoke-test command with PASS criteria, an exact file path string, a URL) — never 'it works'.",
      ].join("\n"),
      input: { objective: "string — the user's objective + any instructions" },
      output: { dispatched: "boolean", row: "the new BCS table row" },
    },

    digestOtherThread: {
      description:
        "Read another session/thread's notes token-efficiently: what is the other thread doing, what did it find, what is its status. The transcript IS the note — but never load it raw into the strategy thread.",
      howTo: [
        "1. List sessions via the ccd_session_mgmt MCP (list_sessions) to get titles + running state + last activity. This is cheap; do it in the main thread.",
        "2. RESOLVE THE TRANSCRIPT DETERMINISTICALLY via the ccSessions weapon (adopted SIO 2026-07-03): `node libs/weapon/ccSessions/cli.mjs \"<local_<uuid> | title substring | 8-char short id>\"` → returns the record incl. `transcriptPath` + `transcriptExists`. (The desktop app's store at %APPDATA%/Claude/claude-code-sessions maps ccd sessionId → cliSessionId = transcript filename.) Do NOT Glob for the ccd sessionId among transcript filenames — the id spaces differ. Fallback ONLY if the store is unreadable: order ~/.claude/projects/<project-slug>/*.jsonl by mtime and exclude the current session's id.",
        "3. DELEGATE THE DIGEST to a cheap-model leaf: spawn an Agent (model: haiku) whose prompt names the transcript path and asks for a <=10-line digest: thread purpose, current activity, key findings, open items. The leaf extracts text blocks only (user strings + assistant text[] entries — skip tool_use/tool_result), e.g. via an inline `node -e` that JSON-parses each line and prints first/last N text blocks truncated to ~400 chars. ~112-line jsonl → ~8 messages → 10-line digest.",
        "4. Only the digest enters the strategy thread. If the digest raises a specific question, a targeted follow-up (grep the jsonl for a keyword, read that slice) beats re-reading everything.",
        "Alternative when you only need WHICH thread discussed a topic: ccd_session_mgmt search_session_transcripts (full-text, returns one snippet per session) — no leaf needed.",
      ].join("\n"),
      input: { question: "string — what you need to know from the other thread" },
      output: { digest: "string — <=10-line summary: purpose, activity, findings, open items" },
    },

    diagnoseDispatch: {
      description:
        "Diagnose why GuildOS cron/dispatch is stalled — quests not getting picked up, a guild (Hermes / main / carbon) not taking jobs, or a manual tick doing nothing. Answers 'cron health check', 'dispatch stalled', 'quests aren't being dispatched', 'is Hermes/main taking jobs', 'try a manual tick'. THE fast path is the caravan TICK API, not reading cron code or running runCron inline.",
      howTo: [
        "DRIVE A TICK VIA THE API: `curl -s -X POST http://localhost:3002/api/caravan/tick -H 'Content-Type: application/json' -d '{\"guildName\":\"main\"}'`. Read `log.loadout` (candidate count per caravan class: planning/execute/derailed/purrview/closing) and `log.dispatch.dispatched` (what ACTUALLY dispatched). loadout>0 with dispatched=[] IS the stall signature. Do NOT use `npm run cj:cron:run-once` (stale — points at a deleted file) or inline `tsx -e` runCron (fights env-load order + export names).",
        "GOTCHA (LOCKED 2026-07-04): caravanTick DEFAULTS guildName='local', which is NOT in KNOWN_GUILDS ([main,carbon,production,hermes]) → resolves to ZERO adventurers → dispatches nothing (registry warns 'unknown guild local — dispatch will yield zero quests'). A bare `{}` tick is a FALSE NEGATIVE. ALWAYS pass an explicit real guildName, and tick each guild separately.",
        "GUILD↔WORKER: a guild only dispatches quests whose `assignee` is an adventurer whose `guilds[]` contains that guild. Check: GET /rest/v1/adventurers?select=name,guilds (Main_Claude→[main], Main_Hermes_Codex→[main,hermes], Carbon_Claude→[carbon]). So Hermes quests CAN dispatch under guild=main (Main_Hermes_Codex is in both).",
        "CRON vs API TICK: the persistent cron (`npm run cj:cron` → run5Min→runCron) drives ONE guild = LOCAL_GUILD_NAME (default 'main'); it does not tick other guilds. Confirm alive: PowerShell `Get-CimInstance Win32_Process -Filter \"Name='node.exe'\" | ? {$_.CommandLine -match 'run5Min|cj:cron'}`. Stale cron runs OLD dispatch/rescue code — restart (detached: `Start-Process cmd '/c cd /d <repo> && npm run cj:cron' -WindowStyle Hidden`) after edits, or it re-exhibits fixed bugs.",
        "READ the queue: GET /rest/v1/quests?stage=in.(plan,execute,purrview,review)&select=id,title,stage,assignee,worker_id. A quest dispatches ONLY if: dispatchable stage + an active pending/running objective + no worker_id lease + assignee in the ticked guild. A stage=escalated or all-objectives-done quest that isn't moving is a RESCUE/questTick problem, not dispatch. (Note: quests table has no `guild_name` column — guild lives on the assignee adventurer.)",
      ].join("\n"),
      input: { symptom: "string — the reported dispatch symptom (guild idle, tick no-op, quests stuck)" },
      output: { diagnosis: "string — which layer is stalled (cron liveness / guild resolution / dispatch filter / rescue) + the tick evidence" },
    },

    delegateScans: {
      description:
        "Scope an objective for tracking: delegate any inventory/survey/scan (branches, worktrees, quests, files, usage) to a cheap-model leaf agent that returns only a compact table row for the objectives tracking table — never run wide scans inline in a strategy planning thread.",
      howTo: [
        "Trigger: the user states an objective that needs scoping (e.g. 'clean up branches') or asks 'what is the status of X'. The scan itself is leaf work; only the verdict/table belongs in the strategy thread.",
        "Spawn Agent with subagent_type: 'Explore' (read-only) or 'general-purpose', model: 'haiku'. Prompt = the exact scan + the exact compact output shape you want back (e.g. 'return two lists: merged-branch count + names of unmerged branches with ahead-count and shortstat, one line each, nothing else').",
        "BEFORE scanning, check whether another running session already owns that work (digestOtherThread) — a duplicate scan is pure waste (this exact duplication happened 2026-07-03: strategy thread re-inventoried branches a sibling thread was already deleting).",
        "The leaf's raw output stays in the leaf. The strategy thread receives the digest and converts it into an objectives-table row or a one-paragraph brief.",
      ].join("\n"),
      input: { scan: "string — what to inventory and the compact shape wanted back" },
      output: { brief: "string — compact table/list, ready to paste into the objectives table" },
    },

    watchSubstrate: {
      description:
        "Strategy-thread substrate duty: keep the GuildOS server and cron running, and include a small report of recently active cron-launched leaf sessions. The server/cron health summary is PART OF the BCS Health line.",
      howTo: [
        "The main strategy thread owns substrate uptime. Each reporting turn, check cheaply (inline, no leaf needed): (1) SERVER — HTTP ping http://localhost:3002 (the `npm run dev` bundle serves Next on 3002); (2) CRON — bundled in `npm run dev` (`cj:cron` → tsx libs/council/cron/run5Min.js); alive if a run5Min/tsx process exists OR .worker-logs/ shows fresh mtimes.",
        "POWERSHELL-FIRST for process/file-mtime checks on Windows (LOCKED 2026-07-03, adopted SIO): use the PowerShell tool (Get-Process, Get-ChildItem | Sort LastWriteTime), NOT Git-Bash idioms — `find /c`, `tasklist | find` misparse in Git Bash (POSIX find walked the whole C: drive and burned a 2-min timeout).",
        "UNCONDITIONAL PER-REPORT (LOCKED 2026-07-03): EVERY message that outputs the objective tables MUST contain BOTH (a) the substrate summary appended to the Health line and (b) the leaf report section. No skipping on 'checked recently' / 'no refresh trigger' — the user expects it every report (this exact omission was corrected 2026-07-03). The check is one PowerShell call; run it fresh each report.",
        "WHEN TO REPORT AT ALL (LOCKED 2026-07-03): the BCS report (tables + substrate + leaves) fires ONLY on objective-touching turns — add/drop/merge/confirm a row, a status or Δ change, or the user asking for a refresh/report. Pure strategy-DISCUSSION turns (ideas, questions, evaluation, advice) get plain prose with NO report — do not re-emit unchanged tables as filler.",
        "HEALTH LINE FORMAT: append the substrate summary to the BCS Health line, e.g. `Health: 90% — <situation>. Substrate: server UP (3002) · cron UP · leaves: 3 active <2h`.",
        "LEAF REPORT: newest 5 of .worker-logs/*.log by LastWriteTime → one line per recent leaf (workerId stem, HH:mm). A labeled 'Cron leaves:' line/section below the objective tables. .worker-logs/<workerId>.log mtime = last activity (see libs/caravan/logs.js).",
        "IF DOWN: restart is pre-authorized (reversible): relaunch `npm run dev` (or only the dev:app subtree on OOM/Jest-worker 500s — leave a healthy cron running). Report the restart in the turn's Δ; escalate only if restart fails twice.",
      ].join("\n"),
      input: {},
      output: { healthSuffix: "string — 'Substrate: server UP/DOWN · cron UP/DOWN · leaves: N active <age>'", leafReport: "<=5 lines" },
    },

    trackSIOs: {
      description:
        "Maintain the SIO (Substrate Improvement Opportunities) table — improvement ideas observed during objective review, placed BELOW the objective tables. Strict token-saving limits; SIOs are adopted ONLY on user command.",
      howTo: [
        "WHEN: evaluate SIOs only when reviewing objectives AND a significant delta has occurred for some objective since the last SIO pass. No delta → skip the pass entirely (write nothing).",
        "HOW (token limit, LOCKED): evaluate ONLY from what the current workflow already surfaced — friction you directly hit, failures observed in this thread or digests. NEVER read additional docs/files/code to hunt for opportunities.",
        "TABLE (below the objective tables): SIO | Observed friction | Proposed improvement | Impact | Risk | Status. Status = 'proposed' until the user commands adoption; then it graduates to a tracked objective/quest and leaves this table. Never implement an SIO on your own initiative.",
        "IMPACT column: HIGH/MED/LOW — how much the friction costs across the substrate if left unfixed (how many quests/workers it blocks, and whether it fully halts them vs merely slows). This is the 'why bother' axis.",
        "RISK column (BINARY, LOCKED 2026-07-03): Risk = confidence a dispatched thread can complete this WELL autonomously if the user says 'go ahead, take care of it, I don't want the details'. LOW = clear bounded spec, extends an existing pattern, reversible, does NOT touch core enforcement semantics (the visual-verify/bounce moat), the audit-log contract, or make a policy/UX fork → the user AUTO-APPROVES low-risk SIOs (dispatch on their 'go' without further sign-off). HIGH = design fork the operator should own, ambiguous/unbounded spec, touches the enforcement moat or a multi-guild contract, or changes durable-record semantics → requires the user's explicit sign-off or a discussion first. Impact (blast radius) is SEPARATE from Risk (autonomous-doability) — a HIGH-impact SIO can be LOW-risk if the fix is clean and bounded.",
        "Carry proposed SIOs forward each report (one line each); drop only on user command ('drop SIO n') or adoption.",
        "ED-ON-ADOPTION (LOCKED 2026-07-03): every SIO dispatch brief MUST instruct the implementing thread to record the change as an engineering_decisions.md entry AT THE ROOT OF EACH AFFECTED MODULE'S FOLDER (per the global + GuildOS-6 ED rule) — one dated entry per module touched (Decision · Rejected alternative · Why · Consequence-if-violated). An SIO is by definition a non-native substrate change (it overrides a current default), so it is always ED-worthy. Applied SIOs handled inline (e.g. recipe edits in this book) get their ED here too. Non-negotiable line in every SIO brief.",
        "QUEST-REVIEW SIO SWEEP (model recipe, 2026-07-03): when the user commands a quest-review SIO sweep, run two stages — (1) haiku leaves fan out (BATCH ~7 quests per leaf, not one leaf per quest — proven 2026-07-03: same cross-quest visibility at ~1/3 the dispatch overhead), each returning a compact digest; (2) ONE sonnet leaf reads all digests together and hunts cross-quest friction patterns (one quest bouncing = noise; several bouncing at the same gate = an SIO) → proposed SIO rows. Sonnet is the judgment sweet spot: haiku SIOs come out generic, fable/opus is flagship rate for curated-anyway output. Never sonnet-per-quest — costlier AND loses the cross-quest visibility.",
        "EXTRACT-STAGE CAPTURE SPEC (refined 2026-07-03 from the sweep's own meta-SIOs): each per-quest digest MUST include — stage; objective count + status breakdown; retry/bounce signal (comment count, repeated actions); AND (M1) timestamps / wall-clock duration (so severity ranking reflects real elapsed cost, not comment count); (M2) the LAST comment VERBATIM, not just its category label (the root-cause reason for escalate/block lives in that text — summarizing it away blinds the judge); (M3) the specific gate/hook NAME+VERSION tagged on any bounce/escalate event (e.g. 'Bounce gate v2'), so the judge can tell same-code-path failures apart from look-alikes.",
        "EVOLUTION PATH (operator intent, 2026-07-03): this sweep is the seed of GuildOS's automated learning/evolving loop, adopting patterns from the Hermes harness. Planned rollout: (a) PER-QUEST at the quest CLOSING stage — the same extract→judge pass runs over the single closing quest and files SIO candidates as part of closure; (b) PERIODIC across recent quests — manual on user command now, later a weekly-then-daily cron tick. Adoption of any SIO remains operator-gated at every phase. When building the closing-stage or cron variant, keep this two-stage model split — it is what makes the loop affordable at daily frequency.",
      ].join("\n"),
      input: {},
      output: { table: "markdown — SIO | Observed friction | Proposed improvement | Status" },
    },

    refTokenEfficiency: {
      description:
        "Reference: token-efficiency findings from the 2026-07-03 strategy-thread analysis — which steps to omit and which to push down to cheaper models.",
      howTo: [
        "OMIT: Glob/search for a ccd sessionId among transcript files (id spaces differ; always a miss — resolve via the ccSessions weapon instead). OMIT: reading a raw session .jsonl into the main context (tool_use/tool_result lines dominate; text is typically <20% of lines). OMIT: re-scanning state a sibling running session already owns.",
        "PUSH DOWN to haiku leaves: transcript digests, branch/worktree inventories, quest-board status sweeps, file/usage surveys. These are extract-and-compress jobs; judgment stays in the main thread.",
        "KEEP INLINE (cheap enough): ccd list_sessions; single targeted greps; the objectives-table upkeep itself.",
        "EXTRACTION PRIMITIVE (run inside the leaf): JSON-parse each jsonl line; keep type==='user' string content and type==='assistant' text blocks; print first/last N truncated to ~400 chars. This turns a multi-MB transcript into a sub-KB digest deterministically before any model reads it.",
        "Cost intuition: one inline wide scan (~4KB) in the main thread costs more than an entire haiku leaf run that returns 10 lines — and pollutes the strategy context permanently.",
      ].join("\n"),
      input: {},
      output: { rules: "the omit/push-down/keep-inline lists above" },
    },
  },
};
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/ccSessions/cli.mjs -->
```js
#!/usr/bin/env node
// CLI for the ccSessions weapon.
//   node libs/weapon/ccSessions/cli.mjs                       → all records (compact)
//   node libs/weapon/ccSessions/cli.mjs "<query>" [--cwd <p>] → matching records (full JSON)
// <query> = ccd id (local_<uuid> or bare uuid), transcript uuid, a >=6-hex-char
// short id (objectives-table form), or a title substring.

import { listSessionRecords, resolveSession } from "./index.js";

const args = process.argv.slice(2);
const cwdIdx = args.indexOf("--cwd");
const cwd = cwdIdx >= 0 ? args[cwdIdx + 1] : undefined;
const query = args.filter((a, i) => i !== cwdIdx && i !== (cwdIdx >= 0 ? cwdIdx + 1 : -1))[0];

if (!query) {
  const rows = listSessionRecords().map(
    (r) =>
      `${r.ccdSessionId}  →  ${r.cliSessionId}  [${r.isArchived ? "archived" : "live"}] ${JSON.stringify(r.title)}  (${r.cwd})${r.transcriptExists ? "" : "  ⚠ transcript missing"}`,
  );
  process.stdout.write(rows.join("\n") + "\n");
  process.exit(0);
}

const idLike = /^(local_)?[0-9a-f]{6,}(-[0-9a-f-]+)?$/i.test(query);
const hits = idLike
  ? resolveSession({ sessionId: query, cwd })
  : resolveSession({ title: query, cwd });

if (!hits.length) {
  process.stderr.write(`no session record matched ${JSON.stringify(query)}\n`);
  process.exit(1);
}
process.stdout.write(JSON.stringify(hits, null, 2) + "\n");
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/ccSessions/engineering_decisions.md -->
```markdown
# Engineering Decisions — ccSessions weapon

## 2026-07-03 — Read the desktop app's private session store; no maintained index, no mtime heuristic

**Decision:** sessionId↔transcript resolution reads the Claude desktop app's own
store (`%APPDATA%\Claude\claude-code-sessions\<org>\<user>\<ccdSessionId>.json`,
field `cliSessionId` = transcript filename) fresh on every call.

**Rejected alternatives:** (a) A GuildOS-maintained index file mapping ids —
requires write-side discipline from every session and goes stale the moment one
forgets; the app's store is already authoritative and always current. (b) The
prior mtime heuristic ("newest transcript that isn't me") — guesswork that
breaks with ≥2 concurrently-active siblings, exactly the multi-session case
this weapon exists for.

**Why:** the store is the only place the ccd id (`local_<uuid>`) and the CLI
transcript id coexist; reading it is O(store size) and needs zero cooperation
from other sessions.

**Consequence-if-violated:** re-introducing mtime guessing silently mis-attributes
digests across concurrent threads. NOTE the accepted risk: the store is an
UNDOCUMENTED app internal — if a desktop update changes its shape, this weapon
degrades to no-matches (never wrong-matches); the documented fallback is mtime
ordering per butlerPlanning.digestOtherThread step 2.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/ccSessions/index.js -->
```js
/**
 * ccSessions weapon — deterministic ccd sessionId ↔ transcript mapping
 * (adopted SIO, 2026-07-03).
 *
 * THE FACT THIS WEAPON ENCODES (verified 2026-07-03): the Claude desktop app
 * keeps one JSON record per Claude Code session at
 *   %APPDATA%\Claude\claude-code-sessions\<org-uuid>\<user-uuid>\<ccdSessionId>.json
 * whose `sessionId` field is the ccd id (`local_<uuid>`) that
 * ccd_session_mgmt.list_sessions returns, and whose `cliSessionId` field IS
 * the transcript filename under
 *   ~/.claude/projects/<cwd-slug>/<cliSessionId>.jsonl
 * (cwd-slug = cwd with every non-alphanumeric char replaced by "-").
 * So a title or ccd sessionId resolves to its transcript path with zero
 * mtime guessing.
 *
 * CLI: node libs/weapon/ccSessions/cli.mjs "<local_<uuid> | transcript-uuid |
 *      8-char short id | title substring>"
 */

export const toc = {
  listSessionRecords:
    "Scan the desktop app's claude-code-sessions store; returns [{ccdSessionId, cliSessionId, title, cwd, isArchived, lastActivityAt, model, transcriptPath, transcriptExists}] sorted by lastActivityAt desc.",
  resolveSession:
    "Resolve {sessionId (ccd 'local_<uuid>', bare uuid, transcript uuid, or a >=6-hex-char prefix like the objectives-table short id), title (substring, case-insensitive), cwd (optional filter)} to matching session record(s) incl. transcriptPath. CLI: node libs/weapon/ccSessions/cli.mjs \"<id-or-title>\"",
};

import { readdirSync, readFileSync, statSync, existsSync } from "node:fs";
import { join } from "node:path";
import { homedir } from "node:os";

// Env override exists ONLY for tests; production reads the real store.
function storeDir() {
  return (
    process.env.CCD_SESSIONS_STORE ||
    join(
      process.env.APPDATA || join(homedir(), "AppData", "Roaming"),
      "Claude",
      "claude-code-sessions",
    )
  );
}

/** cwd → the ~/.claude/projects/ directory slug ("C:\Users\x\GuildOS" → "C--Users-x-GuildOS"). */
export function projectSlug(cwd) {
  return String(cwd || "").replace(/[^a-zA-Z0-9]/g, "-");
}

export function transcriptPathFor(cwd, cliSessionId) {
  return join(homedir(), ".claude", "projects", projectSlug(cwd), `${cliSessionId}.jsonl`);
}

function collectJsonFiles(dir, depth, out) {
  let entries = [];
  try {
    entries = readdirSync(dir);
  } catch {
    return;
  }
  for (const name of entries) {
    const full = join(dir, name);
    let s;
    try {
      s = statSync(full);
    } catch {
      continue;
    }
    if (s.isDirectory()) {
      if (depth > 0) collectJsonFiles(full, depth - 1, out);
    } else if (name.endsWith(".json")) {
      out.push(full);
    }
  }
}

/**
 * All session records in the desktop store, newest activity first.
 * Records are ~10-100KB each (embedded MCP config) but parsing stays cheap at
 * the store's real scale (tens of sessions).
 */
export function listSessionRecords() {
  const files = [];
  collectJsonFiles(storeDir(), 3, files);
  const out = [];
  for (const f of files) {
    let rec;
    try {
      rec = JSON.parse(readFileSync(f, "utf8"));
    } catch {
      continue;
    }
    if (!rec || !rec.sessionId || !rec.cliSessionId) continue;
    const transcriptPath = transcriptPathFor(rec.cwd, rec.cliSessionId);
    out.push({
      ccdSessionId: rec.sessionId,
      cliSessionId: rec.cliSessionId,
      title: rec.title || "",
      cwd: rec.cwd || "",
      isArchived: !!rec.isArchived,
      lastActivityAt: rec.lastActivityAt || 0,
      model: rec.model || "",
      transcriptPath,
      transcriptExists: existsSync(transcriptPath),
    });
  }
  return out.sort((a, b) => (b.lastActivityAt || 0) - (a.lastActivityAt || 0));
}

/**
 * @param {{ sessionId?: string, title?: string, cwd?: string }} q
 *   sessionId: 'local_<uuid>', bare ccd uuid, transcript (cli) uuid, or a
 *   >=6-hex-char prefix of either (the objectives-table short id form).
 *   title: case-insensitive substring. cwd: optional exact-project filter.
 * @returns {Array<ReturnType<typeof listSessionRecords>[number]>}
 */
export function resolveSession(q = {}) {
  let hits = listSessionRecords();
  if (q.cwd) {
    const slug = projectSlug(q.cwd).toLowerCase();
    hits = hits.filter((r) => projectSlug(r.cwd).toLowerCase() === slug);
  }
  if (q.sessionId) {
    const want = String(q.sessionId).replace(/^local_/i, "").toLowerCase();
    hits = hits.filter((r) => {
      const ccd = String(r.ccdSessionId).replace(/^local_/i, "").toLowerCase();
      const cli = String(r.cliSessionId).toLowerCase();
      return ccd === want || cli === want || (want.length >= 6 && (ccd.startsWith(want) || cli.startsWith(want)));
    });
  } else if (q.title) {
    const t = String(q.title).toLowerCase();
    hits = hits.filter((r) => r.title.toLowerCase().includes(t));
  }
  return hits;
}
```
<!-- /GUILDOS-FILE -->
