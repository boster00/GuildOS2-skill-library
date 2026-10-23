# worker

<!-- GUILDOS-SKILL name="worker" -->

<!-- GUILDOS-DESC -->
Adventurers whose role is shipping quest deliverables: claim, execute, ship per deliverable, submit for purrview, address feedback or escalate honestly.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/worker/SKILL.md -->
```markdown
---
name: worker
description: "Execute quest deliverables: claim and orient, execute one objective at a time, ship as deliverable, run execute→purrview→review gates."
---

# Worker

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/worker/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto worker <action>` (or read the source directly).

## Agent portability

This skill applies to any capable agent. Native tool names describe adapters, not agent eligibility. Follow `worker.refAgentCapabilities` through the shared action runner; preserve the required outcome, evidence, authorization and configured backend.

## Actions

- **addressFeedbackOrBounce** — When a quest bounces back to execute (user feedback via cron, Cat purrview bounce, or local review bounce), read the feedback, fix the named objectives/assets, then resubmit.
- **claimQuest** — Orient on the assigned quest before doing any work. Call questmaster.summarizeQuest FIRST for a one-shot digest (stage, next objective, open/answered questions, deliverable fill); then deep-read ONLY what it flags. Read worker.refContextHygiene first.
- **executeStep** — New objectives-model execution. The dispatcher loaded ONE objective into `running`; read that objective row's input/method/output and do its concrete job, honoring [IMPORTANT DETAIL] verbatim and halting at any [co-develop-with-user] anchor. Read worker.refContextHygiene first.
- **refAgentCapabilities** — Shared agent-agnostic execution contract: resolve native equivalents by capability while preserving evidence, authorization, surface and configured backend requirements.
- **refContextHygiene** — Reference: the worker's load-bearing discipline rule. Prefer dispatching sub-agents to absorb noise (file reads, DOM scans, batch comparisons) rather than pulling content into your main session context. Read this on every quest claim.
- **refZoomOut** — Reference: the abstraction-mapping reflex. When you are unfamiliar with a section of code, a substrate, or a quest's surrounding system, zoom out one layer before drilling in.
- **reportAnchorResolution** — Compose the confirmation after a [co-develop-with-user] anchor is resolved: name the anchor, the chosen method (with any fallback condition), the source comment, and confirm execution resumed. GM voice, no follow-up question.
- **reportClosure** — Compose the terminal closure reply after the forge + closeQuest land: confirm the archive by file path, name the new/patched skill book + its parameter, confirm the quest is at complete. GM voice, no follow-up question.
- **reportMutation** — Compose the user-facing confirmation after a questUpdate mutation: quote the exact change applied; no follow-up question when the instruction was complete. GM voice.
- **reportStartup** — Compose the user-facing startup confirmation after the dispatcher loads an objective: name the objective now executing (n + name) and give the live quest page URL to monitor it. GM voice.
- **resolveCoDevAnchor** — Handle a [co-develop-with-user] anchor reached mid-execution: halt the objective, check for a prior user decision in comments/notes, escalate if none — then on the user's answer, record the decision in the objective's notes (citing the source comment) and resume.
- **resolveComment** — Mark a quest_comment resolved=true once its requested change has actually landed. The closing half of an audited mutation pair (update_request → applied).
- **runLifecycleGates** — After the last pre-Checkpoint objective lands, drive the execute→purrview→review gate arc: questExecution.submit (lockphrase), then a REAL per-objective purrview review (not a rubber-stamp), then questPurrview.approve (lockphrase). New objectives/deliverables model.
- **shipItemArtifact** — Per-asset shipping flow: produce the asset an objective's output[].asset names, upload to GuildOS_Bucket, UPSERT the deliverables row (name/kind/ref), and write ONE quest_comment with your worker rationale.
- **submitForPurrview** — When every objective up to the Checkpoint is status='done' and the assets those objectives produce exist as deliverables rows with a ref, call housekeeping.submitForPurrview. The gate enforces the new objectives/deliverables model.
- **surfaceCheckpoint** — At a Checkpoint (quest now at review), open the View Results carousel via the connected operator-browser adapter focused on the Checkpoint objective's assembled asset, pre-filtered to every objective with view_results != 'No', and ask the user to approve or bounce.
- **workerSelfAudit** — Before submitForPurrview, run a 30-second self-audit pass on every gated objective + the assets it produces.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/worker/agent-capabilities.js -->
```js
// Canonical execution contract. Seed/template copies are generated distributions.
export const AGENT_CAPABILITY_CONTRACT = `AGENT CAPABILITIES (applies to this procedure and inherited handoffs): This workflow is available to any agent with the required capabilities. Agent names, native tool prefixes, UI labels and local transcript paths in examples describe adapters, not eligibility gates. Discover the current runtime's documented equivalent for skill loading, shell/files, browser control, session records, delegation and reporting; exercise that capability before declaring it absent. Never invent tool names or treat another harness's missing namespace as a blocker. Use the shared skill runner when native skill loading is unavailable. A configured integration is not proof that it works.

Preserve the required outcome, authentication, operator browser/window, actual rendered evidence, provenance, approval boundaries and any explicitly chosen backend/model. Another adapter controlling the same required surface is valid; a different surface is not. Runtime alternatives do not authorize provider fallback, more spending, external messages, new threads or delegation. Delegate only when the user or applicable instructions authorize it and the runtime supports it; otherwise complete the work directly. Use the actual runtime's session identity and transcript resolver rather than assuming another agent's home directory. Report the specific missing capability after a bounded real check, not an agent-brand mismatch.

For operator-browser work, use tabs in the existing signed-in Chrome window, preferably background tabs; visible tabs there are allowed. Do not create new windows or substitute a separate/headless browser. Record the real adapter and capture provenance without relabeling it. Treat a vendor name in an old handoff as an implementation example; an explicit user-required driver or backend remains a contract constraint and must be changed through the authorized workflow before claiming completion.`;

export function withAgentCapabilities(instruction) {
  if (!instruction || instruction.includes("AGENT CAPABILITIES (")) return instruction;
  return `${AGENT_CAPABILITY_CONTRACT}\n\n${instruction}`;
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/worker/agent-capabilities.test.js -->
```js
import test from "node:test";
import assert from "node:assert/strict";
import { spawnSync } from "node:child_process";
import { fileURLToPath } from "node:url";
import { readFileSync } from "node:fs";
import { skillBook as browser } from "../browsercontrol/index.js";
import { AGENT_CAPABILITY_CONTRACT } from "./agent-capabilities.js";

const runner = fileURLToPath(new URL("../../adventurer/run-action.mjs", import.meta.url));

test("registered procedure loading exposes portable browser and discovery actions", () => {
  for (const [book, action] of [["browsercontrol", "validateBrowser"], ["browsercontrol", "validateCIC"], ["skillDiscover", "addToStartupInstructions"], ["worker", "refAgentCapabilities"]]) {
    const run = spawnSync(process.execPath, ["--no-warnings", runner, "--howto", book, action], { encoding: "utf8" });
    assert.equal(run.status, 0, `${book}.${action}: ${run.stderr}`);
    const payload = JSON.parse(run.stdout);
    assert.equal(payload.ok, true);
    assert.ok(payload.howTo.startsWith("AGENT CAPABILITIES ("));
    assert.equal(payload.howTo.split("AGENT CAPABILITIES (").length, 2, "contract appears once");
  }
});

test("legacy browser gate preserves evidence while accepting equivalent adapters", () => {
  assert.equal(browser.toc.validateBrowser.howTo, browser.toc.validateCIC.howTo);
  const text = browser.toc.validateBrowser.howTo;
  assert.match(text, /Another connected adapter driving that SAME browser is valid/);
  assert.match(text, /Do not invent equivalent API names/);
  assert.match(text, /HTTP 200.*not sufficient/);
  assert.doesNotMatch(text, /screenshot must be.*mcp__claude-in-chrome__/);
});

test("the distributed seed and cloud lease carry the canonical capability contract", async () => {
  const seed = await import("file:///C:/Users/xsj70/GuildOS2/templates/guild/libs/skill_book/worker/agent-capabilities.js");
  const lease = await import("file:///C:/Users/xsj70/GuildOS2/libs/quest/lease/prompts.js");
  assert.equal(seed.AGENT_CAPABILITY_CONTRACT, AGENT_CAPABILITY_CONTRACT);
  assert.equal(lease.AGENT_CAPABILITY_CONTRACT, AGENT_CAPABILITY_CONTRACT);
  const compose = readFileSync("C:/Users/xsj70/GuildOS2/libs/quest/lease/index.js", "utf8");
  assert.match(compose, /\[AGENT_CAPABILITY_CONTRACT, handoff, continuity/);
});
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/worker/index.js -->
```js
/**
 * Worker class skill book — for adventurers whose role is shipping
 * deliverables on a quest (CJGEO Dev, BosterBio Website Dev, Nexus Armor
 * Dev, Researcher, etc.). Loaded in addition to codex + housekeeping +
 * dailies + the worker's project-specific book(s).
 *
 * Workers' lifecycle: claim → execute → ship per deliverable → submit for
 * purrview → if bounced, address feedback and resubmit; if escalated,
 * provide a clear unblock_path.
 */

import { AGENT_CAPABILITY_CONTRACT } from "./agent-capabilities.js";

export const skillBook = {
  id: "worker",
  title: "Worker",
  description: "Adventurers whose role is shipping quest deliverables: claim, execute, ship per deliverable, submit for purrview, address feedback or escalate honestly.",
  steps: [],
  toc: {
    refAgentCapabilities: {
      description: "Shared agent-agnostic execution contract: resolve native equivalents by capability while preserving evidence, authorization, surface and configured backend requirements.",
      howTo: AGENT_CAPABILITY_CONTRACT,
    },
    refContextHygiene: {
      description: "Reference: the worker's load-bearing discipline rule. Prefer dispatching sub-agents to absorb noise (file reads, DOM scans, batch comparisons) rather than pulling content into your main session context. Read this on every quest claim.",
      howTo: `Your context window is the most expensive resource you have. Bloat it and you make worse decisions, miss details, run out of budget mid-task. Treat the main worker agent (you) as an orchestrator that holds **decisions, state, and the next step** — not raw data, intermediate analysis, or large file dumps.

## Rules

1. **RAG everything you can.** Don't bulk-read large files when targeted retrieval works:
   - Grep first to find the lines you need, then Read with \`offset\` + \`limit\` to load only those lines.
   - Querying Supabase / Postgres? Select specific columns + tight \`where\` + limit. Never \`select *\` from large tables.
   - Searching docs? Grep for keywords, read the matching block.

2. **Dispatch sub-agents for noisy work.** If an objective involves:
   - Reading >300 lines of a file to extract a few facts
   - Scraping a DOM or scanning a large page
   - Crunching many records to find a pattern
   - Comparing two long artifacts
   - Reading a large CSV to extract a sample
   → spawn a sub-agent (\`Agent\` tool) with a focused prompt. Sub-agent returns the clean fact; raw content dies with its context. You stay lean.

3. **What stays in main worker context:**
   - Quest id, current deliverable name, current objective
   - Decisions made (with one-line rationale)
   - Pointers (file paths, URLs, record IDs) — not contents
   - Next action

4. **What goes to a sub-agent:**
   - "Look through X and find Y"
   - "Compare A and B"
   - Anything producing lots of intermediate state you don't carry forward
   - File reads where you only need a summary or extracted values
   - Verification reads of artifacts after upload

5. **No improvising on missing tools.** If an objective needs an action that doesn't exist, STOP and report \`SKILL_BOOK_GAP\` via \`housekeeping.escalate\` rather than improvising. Reusable substrate is the point.

6. **One decision, one objective, one return.** Return after each meaningful unit so the orchestrator (or Cat at purrview) can intervene before drift compounds.

## Anti-patterns

- ❌ "Let me read this 2000-line file to understand the migration" → Grep + targeted Read or sub-agent
- ❌ Loading the full Magento product CSV in context → Grep for SKU, Read 5 lines around match
- ❌ Walking 1000 rows in main session → dispatch sub-agent with rows + check criteria; it returns anomalies
- ❌ Holding draft + revised + final versions of HTML in the same session → write each to disk, keep paths only
- ❌ Re-fetching what a sub-agent already returned → trust the return; re-fetch only on legitimate reason

## When in doubt

"If my session crashed right now, what would I need from my context to resume?" That set is what stays in context. Everything else is noise — push to sub-agents, files, or DB.
`,
    },
    executeStep: {
      description: "New objectives-model execution. The dispatcher loaded ONE objective into `running`; read that objective row's input/method/output and do its concrete job, honoring [IMPORTANT DETAIL] verbatim and halting at any [co-develop-with-user] anchor. Read worker.refContextHygiene first.",
      howTo: `**FIRST** — read \`worker.refContextHygiene\`. One decision, one objective, one return.

You are executing exactly ONE \`objectives\` row (the one \`questmaster.tickWorkerDispatch\` flipped to \`running\`). Do not look ahead or batch objectives.

## Procedure

1. **Read the objective row** (\`objectives\` where status='running' for this quest): \`n\`, \`name\`, \`input\`, \`method\`, \`output\` (jsonb array of \`{asset,format,display,verification,verification_example}\`), \`view_results\`, \`notes\`, \`output_key\`. Read the quest's \`deliverables\` rows for any \`[name]\` the input references — fetch only the assets you need (RAG, don't bulk-load).
2. **Honor markers literally:**
   - \`[IMPORTANT DETAIL] …\` clauses in \`method\` are non-negotiable user constraints (an exact endpoint+action, a verbatim copy string, a precise match rule). Implement them character-exact. Never paraphrase or approximate them.
   - \`[co-develop-with-user]\` anchor in \`method\`: STOP forward progress on this objective and hand off to \`worker.resolveCoDevAnchor\` — do NOT guess the method. (Check existing comments/notes first: if the user already decided, proceed with that decision.)
3. **Do the concrete job** the \`method\` describes, producing the asset(s) named in \`output[].asset\`. Write each produced asset as a \`deliverables\` row (\`name\`=output asset, \`kind\`, \`ref\`=URL/storage path or inline ref) — UPSERT on \`(quest_id, name)\` so re-runs replace in place. Update the objective's \`output[]\` entries' produced refs if applicable. Verify the output against each \`output[].verification\` before claiming done — be the first reviewer (mirror of \`workerSelfAudit\`).
4. **Status transition** on the objective row:
   - success → \`running → done\`.
   - needs user eyes / Checkpoint → keep per \`view_results\` (see 5).
   - genuine blocker → **leave the objective OPEN** (do NOT mark it done), raise an \`unblocking\` quest_question (\`unblocks_objective_ids\` → this objective) under the quest-success escalation principle, and escalate the quest (\`housekeeping.escalate\` / \`moveQuestToEscalated\`) with a clear unblock_path; do not fake completion. The open question + \`escalated\` stage ARE the block — there is no \`attention\` objective status (retired 2026-06-15).
5. **Respect \`view_results\`:** if the just-finished objective's \`view_results='Checkpoint'\`, HALT execution past this objective (do not auto-advance to the next pending objective) — the lifecycle gate / user reviews it first. \`Show\`/\`No\` → the dispatcher's next tick may pick up the next pending objective.
6. **Record method developments** in the objective's \`notes\` (concise) when execution discovers a reusable method (e.g. an SSH/SQL retrieval approach) — the closing-stage \`questmaster.scanForUncommittedMethods\` reads notes for skill-book-capture candidates.

## Rules

- One objective per invocation. Return after the status transition so the orchestrator/gate can intervene before drift compounds.
- Never write \`quests.stage\`. Never skip a \`[co-develop-with-user]\` anchor. Never soften an \`[IMPORTANT DETAIL]\`.
- No improvising a missing capability — if the method needs a tool/action that doesn't exist, escalate \`SKILL_BOOK_GAP\` (don't fake the artifact).
`,
    },
    reportStartup: {
      description: "Compose the user-facing startup confirmation after the dispatcher loads an objective: name the objective now executing (n + name) and give the live quest page URL to monitor it. GM voice.",
      howTo: `After \`questmaster.tickWorkerDispatch\` returns \`{ ok:true, questId, url, title, objective:{n,name} }\`, compose ONE concise confirmation for the user:

- State that the local worker dispatcher was triggered, scanned for active Guildmaster quests, picked up the quest by title, and loaded its first pending objective into execution.
- Name the objective now running: \`objective <n> (<name>)\`, single worker.
- Give the live quest page URL (\`url\`) to watch progress.
- Promise to surface anything that needs the user's input; do not ask a question here (a started objective is forward motion, not a deviation).

Keep it to ~2 sentences. Do not re-summarize the plan or open a cascade. If the weapon returned \`{ ok:false, reason }\`, report that plainly instead (nothing to dispatch) — never invent a started objective.
`,
    },
    resolveComment: {
      description: "Mark a quest_comment resolved=true once its requested change has actually landed. The closing half of an audited mutation pair (update_request → applied).",
      howTo: `A \`quest_comments\` row written by \`questmaster.questUpdate\` (\`action='update_request'\`, \`resolved=false\`) is an OPEN audit record. Resolve it ONLY after the mutation it requested has actually been rendered on the targeted attribute and verified — never before, never speculatively.

1. Confirm the requested change is live (e.g. the objective's \`method\` now carries the refined rule + its \`[IMPORTANT DETAIL]\` marker).
2. Set that comment's \`resolved=true\` (the thin questmaster DB helper does the write).
3. One comment per request; do not bulk-resolve unrelated open comments (a co-develop escalation or a save-to-skill-book proposal stays open until ITS own thing lands).

Resolving before the change lands breaks the audit trail (a "done" marker with nothing done). If the change failed, leave it \`resolved=false\` and escalate.
`,
    },
    reportMutation: {
      description: "Compose the user-facing confirmation after a questUpdate mutation: quote the exact change applied; no follow-up question when the instruction was complete. GM voice.",
      howTo: `After \`questmaster.questUpdate\` renders a change and \`worker.resolveComment\` closes the request comment, compose ONE concise confirmation:

- State that the request was logged as a comment on the quest AND applied to the specific attribute (name the objective + attribute, e.g. "applied to objective 2's method").
- Quote the exact mutation in substance (what the rule/value now is).
- Explicitly note anything deliberately left intact that the user might wonder about (e.g. an unrelated open \`[co-develop-with-user]\` anchor still pending).
- Ask NO follow-up question when the instruction was complete — a finished, audited mutation is forward motion, not a deviation. Only surface a question if the request itself was genuinely ambiguous AND context could not resolve it.

Keep it tight (~2-3 sentences). Do not re-summarize the whole quest or open a cascade.
`,
    },
    resolveCoDevAnchor: {
      description: "Handle a [co-develop-with-user] anchor reached mid-execution: halt the objective, check for a prior user decision in comments/notes, escalate if none — then on the user's answer, record the decision in the objective's notes (citing the source comment) and resume.",
      howTo: `\`[co-develop-with-user]\` in an objective's \`method\` means: the objective admits more than one execution method and the user owns that choice. When \`worker.executeStep\` reaches this marker it MUST stop forward progress on that objective and come here. Two phases:

## Phase A — reached the anchor (no decision yet)

1. **Halt** the objective. Do NOT guess a method, do NOT silently self-resolve.
2. **Check for a prior decision.** Read the quest's \`quest_comments\` + this objective's \`notes\`. A decision counts ONLY if it actually answers THIS anchor's choice. A refinement to an unrelated clause (e.g. a match-rule tweak) is NOT a decision on the retrieval method — scope matters; do not treat an adjacent change as resolving this anchor.
3. If a real prior decision exists → apply it and resume (skip to Phase B step 3, no escalation needed).
4. If none → **escalate**: compose a user-facing message that names the blocker precisely (which objective, which anchor, what the choice is), gives the realistic options + why it matters (e.g. crawling ~N pages is slow/brittle vs a bulk method), and states execution is paused on this objective until they reply. Then the objective stays halted. (This escalation is a sanctioned user-facing question — it is a genuine user-owned fork, not a manufactured stop.)

## Phase B — user answered the anchor

1. The user's reply is an answer to the pending escalation. It is routed through \`questmaster.questUpdate\` (comment-first-then-render): the instruction is written as an \`update_request\` quest comment BEFORE any change, then the update agent rewrites this objective's \`method\` to specify the chosen method.
2. The rewritten \`method\` carries the chosen mechanism, flagged as an inline \`[IMPORTANT DETAIL]\` (verbatim constraints — exact access pattern, table/column rules, fallback conditions). It **supersedes-and-reconciles**: any earlier scoped refinement to a different clause of this objective (e.g. a match rule) is preserved; only the part the anchor governs (the method/mechanism) is now resolved.
3. **Record the decision in the objective's \`notes\`** (concise), explicitly citing the just-written user comment as the source ("retrieval method resolved per user comment <id>: …"). \`notes\` is the execution-artifact log; the instruction itself lives in \`method\`.
4. Resume execution of the objective (the dispatcher's next tick picks it up; status stays/returns to the running path).
5. The \`update_request\` comment is closed via \`worker.resolveComment\` once the mutation lands AND execution resumes — not before.
6. Confirm via \`worker.reportAnchorResolution\`.

## Rules

- Never skip or self-resolve an anchor on "it's obvious". The whole point is the user owns this fork.
- An unrelated prior refinement does NOT count as a decision for this anchor — verify scope before treating anything as a prior decision.
- The chosen method goes in \`method\` as \`[IMPORTANT DETAIL]\`; the *fact that it was decided + the source comment* goes in \`notes\`. Don't conflate the two.
- Reconcile, don't clobber: preserve earlier scoped refinements to other clauses of the same objective.
`,
    },
    reportAnchorResolution: {
      description: "Compose the confirmation after a [co-develop-with-user] anchor is resolved: name the anchor, the chosen method (with any fallback condition), the source comment, and confirm execution resumed. GM voice, no follow-up question.",
      howTo: `After \`worker.resolveCoDevAnchor\` Phase B lands, compose ONE concise confirmation:

- Name the anchor that was blocking (which objective, what the choice was).
- State the chosen method in substance, including any verbatim fallback/threshold condition the user gave (e.g. "only after N tries + doc research, fall back to CIC").
- Note what earlier refinement was preserved (the supersede-and-reconcile: e.g. "your earlier match rule is kept; only the retrieval mechanism changed").
- Confirm the instruction is logged as a comment, cited in the objective's notes, the comment is resolved, and the objective is running again.
- No follow-up question — the fork is resolved; resumed execution is forward motion.

Keep it tight (~3-4 sentences). Don't re-summarize the whole quest.
`,
    },
    runLifecycleGates: {
      description: "After the last pre-Checkpoint objective lands, drive the execute→purrview→review gate arc: questExecution.submit (lockphrase), then a REAL per-objective purrview review (not a rubber-stamp), then questPurrview.approve (lockphrase). New objectives/deliverables model.",
      howTo: `Run this when execution reaches the Checkpoint objective (the objective whose \`view_results='Checkpoint'\`) and its work has landed. The gate WEAPONS already exist with the correct lockphrases + transitions — you orchestrate them; you never write \`quests.stage\` directly.

## Arc

1. **Submit for purrview** — call \`questExecution.submit({questId})\`. It hard-gates and, on pass, writes \`stage='execute' → 'purrview'\` and posts the SUBMIT lockphrase comment \`"this quest now meets the criteria for purrview"\`. Under the new model the readiness signal is the \`objectives\`/\`deliverables\` state (every objective up to and including the Checkpoint is \`done\`; the produced \`deliverables\` assets exist with refs), NOT the retired deliverables.expectation/per-deliverable-comment/tier-check columns. If the live gate still validates the old columns and rejects, that is a substrate gap to fix in the weapon (scoped follow-up) — do not bypass it by writing \`stage\` directly.
2. **Real purrview review (NOT rubber-stamp)** — genuinely inspect each presentable objective's produced asset against that objective's \`output[].verification\`: the interactive-map anchors resolve to the right gene-card URLs; the quote-form POSTs to the exact \`[IMPORTANT DETAIL]\` endpoint+action and renders the verbatim promo copy; the trimmed description has no duplicate hero/CTA/footer; the deploy screenshot matches. Per-objective verdicts must be earned by looking, not assumed. A pass you didn't verify is a discipline failure the downstream gate/user will catch.
3. **Approve** — only if every objective's verdict is a genuine \`pass\`, call \`questPurrview.approve(...)\`. It writes \`stage='purrview' → 'review'\` and the APPROVE lockphrase \`"this quest now meets the criteria for review"\`. Any genuine \`fail\` → bounce instead (\`purrview → execute\` with a per-deliverable reason); never approve around a real defect.
4. Then surface the checkpoint to the user via \`worker.surfaceCheckpoint\`.

## Rules

- Lockphrases are written by the weapons, never hand-typed by you. Stage transitions are weapon-gated, never a direct \`quests.stage\` write.
- The review must be real. "All deliverables passed" is only sayable after you actually checked each against its \`output[].verification\`.
- One arc per Checkpoint. Do not skip the review to "save time" — that is the exact failure the layered gate exists to prevent.
`,
    },
    surfaceCheckpoint: {
      description: "At a Checkpoint (quest now at review), open the View Results carousel via the connected operator-browser adapter focused on the Checkpoint objective's assembled asset, pre-filtered to every objective with view_results != 'No', and ask the user to approve or bounce.",
      howTo: `The quest just reached \`stage='review'\` via \`questPurrview.approve\`. Surface it for the user's decision:

1. **Open the View Results carousel in the existing operator browser** focused on the Checkpoint objective's produced asset (the assembled \`final_pathway_page\` for the hypoxia quest — objective 8). Pre-filter the carousel to every objective whose \`view_results != 'No'\` (i.e. \`Show\` or \`Checkpoint\`), with the Checkpoint objective as the anchor so the user lands on the assembled/deployed result FIRST, then can page through the other presentable objectives.
2. **Compose the prompt**: state the checkpoint was reached AND that you ran a real purrview review (briefly name what you actually checked per objective), that all objective verdicts passed so the quest is at \`review\`, the carousel is open in the existing operator browser pre-filtered with the Checkpoint anchor, and ask the user to **approve** the result or **bounce it back to execute with a reason**.
3. This is a sanctioned user-facing decision point (review-stage approval is user-owned) — it is a real fork, not a manufactured stop. Surface it and wait; do not self-approve the review stage.

Keep the prompt tight. Do not re-summarize the whole plan; the carousel shows the work.
`,
    },
    reportClosure: {
      description: "Compose the terminal closure reply after the forge + closeQuest land: confirm the archive by file path, name the new/patched skill book + its parameter, confirm the quest is at complete. GM voice, no follow-up question.",
      howTo: `After \`questmaster.closeQuest\` returns \`{ ok:true, stage:"complete" }\`, compose ONE concise terminal confirmation:

- State the workflow was forged (or an existing book patched) and give the skill book file path (e.g. \`libs/skill_book/PathwayEnrichment/index.js\`).
- Name the skill book and the parameter it is parameterized over (e.g. "parameterized over \`pathwayURL\`"), and note the in-execution method that was included (e.g. the SSH/SQL antibody-retrieval method from objective 2).
- Confirm it loads via the skill book registry without error.
- Confirm the closing comment is resolved and the quest is now at \`complete\`.
- One forward-looking line on reuse (e.g. "re-run the whole flow on any other Boster pathway map by calling it with a different URL").
- No follow-up question — closure is terminal, this is the deliverable.

Keep it tight (~3-4 sentences). This is the run's end state; make it a clean State-2 report.
`,
    },
    claimQuest: {
      description: "Orient on the assigned quest before doing any work. Call questmaster.summarizeQuest FIRST for a one-shot digest (stage, next objective, open/answered questions, deliverable fill); then deep-read ONLY what it flags. Read worker.refContextHygiene first.",
      howTo: `**FIRST** — read \`worker.refContextHygiene\`. The discipline applies to every objective below.

0. **ORIENT FAST (do this before any raw read):** call \`questmaster.summarizeQuest({questId})\`. Its concise \`digest\` tells you the stage, the next actionable objective, every OPEN and answered-not-digested question, the deliverable fill ratio, and the latest note — in one call. Do NOT bulk-read all objectives/comments just to learn status. Use the digest to decide what to deep-read; pull full bodies (steps 1-4 below, or \`questmaster.getQuestDetails\`) ONLY for the specific rows the digest flags (the next objective, the open questions, any unresolved user comment).

1. Read \`quests.description\` literally — that's the OBJECTIVE. Do not mistake it for status text; status lives in \`stage\` + per-objective \`status\`.
2. Read every \`objectives\` row for this quest (ordered by \`n\`): \`name\`, \`input\`, \`method\`, \`output\` (jsonb array of \`{asset,format,display,verification,verification_example}\`), \`view_results\`, \`notes\`, \`output_key\`. The \`output[].verification\` is the literal claim Cat (T2) and the user judge each produced asset against — each asset you produce has to faithfully satisfy its objective's stated verification.
3. Read any pre-existing \`deliverables\` rows (assets are \`name\`/\`kind\`/\`ref\`) — these are external inputs created at planning that an objective's \`input\` may reference as \`[name]\`. The rest of the deliverables are produced BY execution.
4. Read recent \`quest_comments\` (latest 5–10). Anything from \`source='user'\` with \`action='feedback'\` is a hard requirement to address; a \`resolved=false\` \`update_request\` is an open change you must honor.
5. If anything is ambiguous, ask via \`housekeeping.seekHelp\` — don't infer. The objective phrasing is intentional; if it doesn't make sense, the spec is wrong, not your read.
`,
    },
    shipItemArtifact: {
      description: "Per-asset shipping flow: produce the asset an objective's output[].asset names, upload to GuildOS_Bucket, UPSERT the deliverables row (name/kind/ref), and write ONE quest_comment with your worker rationale.",
      howTo: `Per asset (one entry of an objective's \`output[].asset\`):

1. Produce the asset. Open it yourself first — does it actually satisfy that objective's \`output[].verification\`? If not, regenerate. Don't ship work that doesn't match its spec.
2. Upload to GuildOS storage bucket (\`GuildOS_Bucket\`, on the GuildOS project — sdrqhejvvmbolqzfujej.supabase.co). Path convention: \`cursor_cloud/<questId>/<assetName>.<ext>\` or \`general/<questId>/<assetName>.<ext>\` for non-screenshot artifacts. Helper: \`supabase_storage\` weapon. (For inline assets — markdown/json — \`ref\` can be a storage path or an inline reference; pick the \`kind\` accordingly.)
3. UPSERT the deliverables row (UNIQUE \`(quest_id, name)\` — resubmits replace in place; never invent \`_v2\` names). Prefer the \`libs/quest\` helper so the shape stays correct:
\`\`\`javascript
import { writeDeliverable } from "@/libs/quest";
await writeDeliverable({
  questId,
  name: "<the objective output[].asset>",
  kind: "image", // one of: html | image | markdown | json | link
  ref: "<public-url-or-storage-path-from-step-2>",
});
\`\`\`
4. Insert ONE \`quest_comments\` row with your rationale (what the asset shows + how it satisfies the objective's verification). Per-asset comments live on \`quest_comments\` now — there is no item_comments table under the new model:
\`\`\`javascript
import { recordQuestComment } from "@/libs/quest";
await recordQuestComment(questId, {
  source: "adventurer",
  action: "asset_rationale",
  summary: "<asset name>: <short rationale — what it shows and why it satisfies the objective verification>",
  actor_name: "<your-name>",
  detail: { asset_name: "<the objective output[].asset>" },
});
\`\`\`

Model reminder: deliverables are pure assets (\`name\`, \`kind\`, \`ref\`). The retired \`expectation\` / \`url\` / \`caption\` / \`self_check\` / \`*_check\` tier columns and the \`item_comments\` table are GONE — do not write them.
`,
    },
    submitForPurrview: {
      description: "When every objective up to the Checkpoint is status='done' and the assets those objectives produce exist as deliverables rows with a ref, call housekeeping.submitForPurrview. The gate enforces the new objectives/deliverables model.",
      howTo: `Use \`housekeeping.submitForPurrview\` directly — that action wraps \`questExecution.submit\` (the gate-enforced weapon). Gate failures are listed in the action's response; do not patch around them.

Common gate-fail causes (new objectives/deliverables model):
- An objective up to and including the Checkpoint is not \`status='done'\` → finish it via \`worker.executeStep\` until its status flips to 'done'.
- A produced asset named in a gated objective's \`output[].asset\` has no \`deliverables\` row, or its \`ref\` is null → \`worker.shipItemArtifact\` for that asset (UPSERT deliverables name/kind/ref).
- Quest not at \`stage='execute'\` → the gate only opens from execute; don't try to resubmit a quest already past it.

If the gate fails, re-read the failure list, fix the named objectives/assets, retry. Do NOT directly write to \`quests.stage\` to bypass — every transition is weapon-gated.
`,
    },
    addressFeedbackOrBounce: {
      description: "When a quest bounces back to execute (user feedback via cron, Cat purrview bounce, or local review bounce), read the feedback, fix the named objectives/assets, then resubmit.",
      howTo: `Three bounce paths land you back in execute:

1. **User feedback bounce** (cron-triggered): a fresh \`quest_comments\` row with \`source='user' / action='feedback'\` arrived after the quest entered review. The cron writes a system audit comment (\`action='review_bounce_user_feedback'\`) and sends you a \`[USER FEEDBACK]\` followup with the verbatim text. The user's feedback is the contract — do not re-submit until you've addressed it.

2. **Cat purrview bounce** (\`questPurrview.bounce\`): the SINGLE bounce comment (\`action='bounce'\`, top-level \`verdict='fail'\`) carries everything — \`detail.failing_item_keys\` lists the failing **asset names** and \`detail.perItemVerdicts\` carries the per-asset rationale (\`{item_key, verdict, text, judge_rationale}\`). (As of 2026-06-15 there is no longer a separate \`asset_verdict\` row per asset — it was collapsed into this one comment.) Re-produce those specific assets (UPSERT on \`(quest_id, name)\` replaces in place — never \`_v2\`) and/or re-run the objectives that produce them.

3. **Local review bounce** (\`questReview.bounce\`): same shape as #2 but \`source='questReview'\` — a stricter local re-judge failed an asset.

After addressing: write a worker \`quest_comments\` row summarizing what you changed (and set any related \`resolved=false\` request comment to \`resolved=true\` once its change has landed — see \`worker.resolveComment\`), then re-call \`housekeeping.submitForPurrview\`.

If the feedback is impossible to address (spec is genuinely infeasible), don't fake — escalate via \`housekeeping.escalate\` with detail.reason + detail.unblock_path.
`,
    },
    workerSelfAudit: {
      description: "Before submitForPurrview, run a 30-second self-audit pass on every gated objective + the assets it produces.",
      howTo: `For each objective up to and including the Checkpoint, and each asset it names in \`output[].asset\`:
1. Is the objective \`status='done'\`? A 'running'/'pending'/'active' objective blocks the gate.
2. Does the asset have a \`deliverables\` row with a non-null \`ref\`? Is the \`ref\` fetchable (HTTP 200, real bytes for a URL)? \`curl -I <ref>\` if unsure.
3. Did I open the asset myself and verify it shows what the objective's \`output[].verification\` claims? Don't outsource this to the judge — be the first reviewer.
4. Did I record the reusable method developments (if any) in the objective's \`notes\` so the closing-stage \`questmaster.scanForUncommittedMethods\` can find them?
5. Is there at least one worker \`quest_comments\` rationale row for each produced asset?

## CRITICAL: Proof-required deliverables

If the acceptance criteria contain any of these words — **proof**, **demonstrate**, **show**, **evidence**, **working implementation**, **verified** — the artifact MUST contain ACTUAL outputs, not documentation or pseudocode. Concretely:

- ❌ An API spec page or flow diagram does NOT satisfy "working implementation" — the endpoint must exist and respond.
- ❌ A pseudocode block or decision diagram does NOT satisfy "show fallback behavior" — an actual rendered page or curl response is required.
- ❌ A specification of what WOULD happen is not proof of what DOES happen.
- ✅ Actual curl output showing HTTP 200 + response body (with real \`report_id\`, \`public_url\`, etc.)
- ✅ A screenshot or rendered HTML showing the actual page at the actual URL
- ✅ A 401 curl response from an actual unauthenticated request

When in doubt: can I paste this artifact's content into a bug report and have a developer reproduce the behavior? If no → it is documentation, not proof.

If any answer is "no" or "kinda" — fix it before submit. The submit gate will reject anyway, but better you catch it than the gate.

This is the worker-side mirror of the WWCD visual-verification rule. The worker is T0; T0 honesty makes T1/T2/T3.5/T4 easier.
`,
    },
    refZoomOut: {
      description: "Reference: the abstraction-mapping reflex. When you are unfamiliar with a section of code, a substrate, or a quest's surrounding system, zoom out one layer before drilling in.",
      howTo: `Use this when you find yourself reading inside a module / file / table you don't have a mental map for.

## The reflex

Do not start editing or proposing changes from inside an unfamiliar layer. Go up one layer of abstraction first, then come back.

Produce a one-screen map of:

- The **deep modules** in the area (small interface, lots of behavior).
- Their **callers** — who depends on this surface.
- Their **collaborators** — what they depend on.
- The **seams** — where the interface lives and could be swapped.
- The **domain terms** from \`CONTEXT.md\` that name what's happening here (don't drift into synonyms).

## Output shape

A short labeled map, not prose. Example:

\`\`\`
Area: questmaster.upsertQuest
  callers:
    - GM step 4b (CLAUDE-GM.md)
    - questmaster.planRequestToQuest (when promoting a plan)
  collaborators:
    - questmaster.nameQuest (naming)
    - questmaster.evaluate (SMART firewall)
    - DB:quests, DB:quests.objectives, DB:quests.deliverables
  seam:
    - the persistence call — one adapter today (Supabase). Hypothetical seam.
  domain terms:
    - Quest, Objective, Deliverable, SMART evaluation, lifecycle stage
\`\`\`

## When to skip

Skip if the area is already on your active mental map from this same session and the step is a narrow edit (rename, typo fix, single-line change). Otherwise spend the 30 seconds — bad mental maps produce confidently wrong edits.

## When to dispatch a sub-agent

If building the map requires reading >300 lines or scanning many files, dispatch a sub-agent with this exact reflex as the prompt. The sub-agent returns the map; raw reads die with its context.
`,
    },
  },
};

export const definition = skillBook;
```
<!-- /GUILDOS-FILE -->
