# questmaster

<!-- GUILDOS-SKILL name="questmaster" -->

<!-- GUILDOS-DESC -->
Plan user requests into quest-shaped JSON and pick roster adventurers by capability.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/questmaster/SKILL.md -->
```markdown
---
name: questmaster
description: "Plan user requests into quest-shaped JSON; itemize objectives + deliverables and match roster adventurers by capability."
---

# Questmaster

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/questmaster/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto questmaster <action>` (or read the source directly).

## Agent portability

This skill applies to any capable agent. Native tool names describe adapters, not agent eligibility. Follow `worker.refAgentCapabilities` through the shared action runner; preserve the required outcome, evidence, authorization and configured backend.

## Actions

- **approveOrEscalate** — Handle requests from worker agents seeking approval or help.
- **assetInlineVsFileStorage** — When a produced asset/result is stored INLINE (text in the row's ref) vs in a FILE. Threshold: text < 1000 chars → inline; otherwise (or any binary) → file. LOCKED 2026-06-12.
- **assign** — Assign stage: load quest, scan adventurer roster, pick best-fit or return no-match.
- **assistAdventurer** — Assist an adventurer asking for help; scan the full skill-book registry, not just their loaded set.
- **auditLeafLogs** — Parse .worker-logs/*.log and emit per-leaf tool-call audit: ordered sequences, waste classification (getQuestDetails miss, import probing, --list discovery, skill-book source reads), and summary counts. Input: { logDir?, filterPrefix? }. No LLM.
- **checkpointQuestAsset** — One-call quest-asset checkpoint honoring assetInlineVsFileStorage: text < 1000 chars stores INLINE (ref = the text, no upload, filename optional, kind defaults 'text'); larger/binary content uploads to GuildOS_Bucket (kind defaults 'document') then inserts the quest_assets row. Replaces the two-step hand-rolled upload+REST-insert for ruleCheckpointAssets.
- **clarifyQuest** — THE action for creating a quest from a chat session (the pre-creation gate, run BEFORE createQuest). A quest-creating chat has ONE job: nail a high-fidelity DESCRIPTION plus the user-owned deliverables — then createQuest({ownerId,title,description}) births the quest at stage='plan' and the planQuest substrate itemizes the objectives. Do NOT author objectives in chat, and the approval proposal you show the user is Name + Description + Deliverables only (objectives are the planner's job, not the chat's). Gate: evaluate whether {name, deliverables, methods} are defined and feasible; if not, ask clarification questions instead of creating a half-formed quest.
- **closeQuest** — Terminal closure arc. The user said yes to the closing finale proposal — forge the skill book (blacksmith.forgeWeapon), resolve the finale comment once the forge lands, then questmaster.closeQuest weapon (gated closing→complete), then worker.reportClosure.
- **closeQuest** — Archive quest deliverables and summary to Asana, then mark complete.
- **commentDigestion** — How a worker handles quest_comments. A source='user' comment is a DIRECTIVE: follow it, then mark resolved=true (NEVER delete — comments are the durable audit log). Contrast questionDigestion (questions are removed). Comments render newest-first. LOCKED 2026-06-12.
- **createPR** — Create a pull request for the worker agent's branch after review approval.
- **deliverableStoragePolicy** — Where a worker stores a produced deliverable (local filesystem vs Supabase Storage). The BASE preference is owned here (questmaster skill book); each worker may hold its own preference on top. LOCKED 2026-06-11.
- **deliverableTacticalEvaluation** — Prompt template used by the questmaster.SMARTItemClarification weapon. Asks 3 questions per objective; returns ok or specific clarification questions.
- **dispatchRoledAgent** — Wraps the questmaster.dispatchRoledAgent weapon. Composes role-aware substrate priming (CLAUDE.md + CLAUDE-<role>.md) for the dispatched sub-agent and delegates to claudecli. Use this when a dispatch needs the sub-agent to think as a specific role.
- **getSecondOpinion** — Launch Claude CLI to independently evaluate a submission when you're unsure.
- **handleFeedback** — Act on feedback from a quest comment or direct message.
- **interpretIdea** — Turn a raw user request into a structured quest for a chosen adventurer.
- **nameQuest** — Sub-procedure used inline by upsertQuest. Produces a quest name from a SMART plan following the action-verb convention.
- **planQuest** — OWNS the plan-stage itemizing procedure: turn a quest's big description (summarized from the user's ask) into objectives + deliverables + quest_questions, shrinking the description until it is short and non-overlapping with the sub-items, then transition to execute. Run by the planning agent questTick deploys to plan-stage quests; the scripted itemizer is the questmaster WEAPON's planQuest action.
- **planRequestToQuest** — Translate a raw user request into a structured quest (title, description, deliverables, due date, stage).
- **questionDigestion** — Questions are TRANSIENT but their DECISIONS are permanent. Digestion = a 4-step capture pass: extract decisions from every answer → MAP each to description / a specific objective / new objective(s) / assets → VERIFY the main points are captured → only THEN remove (unblocking only). Deleting an answer before its decisions are migrated is a DATA-LOSS defect. LOCKED 2026-06-13.
- **questUpdate** — Apply a user refinement / escalation-answer to an EXISTING in-flight quest. Comment-first-then-render: summarize the request as a quest comment BEFORE any mutation, then a dedicated update agent classifies the target attribute and rewrites it. Used by GM Step 1.5's refinement branch — never for a new request.
- **raiseUnblockingQuestionDiplomatically** — Raise a question only when the quest is unlikely to succeed without the user's intervention; otherwise, keep working.
- **readLifecycleContract** — Canonical quest lifecycle — event-bus stages, cron dispatch, worker leasing, objectives model. Read this first when creating a quest, transitioning a stage, or interpreting a stage-related comment. Authoritative spec: docs/quest-lifecycle.md.
- **reportChaperonWork** — Report chaperon work by creating a review-stage quest on the Guildmaster.s Desk.
- **reviewCloudAgentWork** — Legacy: high-level review framing for cloud-agent deliverables. For new work, use reviewSubmission — the script-locked flow.
- **reviewSubmission** — Manual purrview review for non-image artifacts (text, JSON, doc). For image-only quests, use runReviewLoop instead. Script-locked end-to-end via questPurrview weapon: confirmSubmission → per-deliverable evaluation → approve OR bounce.
- **ruleAcceptableStopState** — The criteria a worker burst MUST reach before it may stop. Every objective either done, or open-and-blocked-by-a-linked-unblocking-question; every block externalized as an unblocking question; exactly one stop-summary comment (edit the prior failed-stop comment, don't pile up). LOCKED 2026-06-12; objective `attention` status retired 2026-06-15.
- **ruleCheckpointAssets** — Two-layer asset-archiving policy (decided 2026-06-12). LAYER 1 (primary): a worker proactively saves any result a LATER objective will consume as a quest_asset. LAYER 2 (recovery): if a later objective finds the asset missing, it REACTIVATES the prerequisite objective with a note to archive, and retries. Chosen over a hard move-to-review gate. LOCKED 2026-06-12.
- **ruleComments** — LOCKED 2026-06-12: quest_comments are the STRATEGIC, FUTURE-BEARING channel — NOT an action log. A comment earns its place only if a future reader (the next worker or the user) gains actionable insight from it. Write what was DECIDED + what to DO NEXT, never bookkeeping of what happened.
- **ruleCommentValue** — Every quest_comment must be INSIGHT-BEARING + ACTIONABLE — it tells the next step what to do. Pure mechanical event-logs (lease reclaim, leaf dispatch) are NOT comments; they live in worker_id / console / cron_log. Don't pile up: edit a repeat in place. LOCKED 2026-06-12.
- **ruleObjectiveClarity** — How an objective's NAME is phrased. Same convention as a quest name: future-focused, starts with a verb (or verb+object), one action per objective. No origin prefix, no combined actions. LOCKED 2026-06-12.
- **ruleUserQuestionAuthority** — AUTHORITY boundary: a worker may ANSWER user-type questions but may NEVER delete or resolve them, and must answer every open user question BEFORE executing objectives. Only the user removes a user question. Enforced by the deleteQuestQuestionById guard. LOCKED 2026-06-12.
- **runReviewApproval** — Approve a GuildOS2 quest waiting at stage='review' on the operator's behalf ('looks good, approve it'). Review's exit is an ATTESTATION, not a derived fact: it is a USER-CHANNEL write of stage='reflect' — the same op the GM Room's 'Approve → reflect' button sends. The worker channel refuses it (no_worker_stage_assertion), so POST /api/quest/update can NEVER approve; the machine route that can is POST /api/quest/test with action='mutate', channel='user'.
- **runReviewLoop** — One-call locked review: confirmSubmission → openai_images.judge per deliverable → approve OR bounce. Use this for any quest whose deliverables are all images. Falls back to manual_required for non-image deliverables.
- **scanForUncommittedMethods** — At stage='closing', walk every objective's notes for in-execution method developments not yet captured in a skill book; write a closing-stage quest comment proposing the SMART plan AND any such method be saved to a skill book for reuse, naming the source objective.
- **searchAdventurerForQuest** — Pick the best-fit adventurer for a quest by capability match.
- **searchQuestBySimilarName** — Prompt template used by the questmaster.searchQuestBySimilarName weapon. Picks the best-matching active quest, or null.
- **selectAdventurer** — Match a new request to a roster adventurer (returns the id or no-match).
- **SMARTClarification** — SA-dispatched action. Runs AFTER questmaster.evaluate has failed. Two-tier internal eval: T1 = no objectives → ask generic 'what is the workflow and how do I verify it'; T2 = objectives present but gaps → per-objective deep eval via questmaster.SMARTItemClarification weapon. Aggregates user-facing questions and a partial objectives table.
- **stepBoundaryAnnotations** — Convention for embedding sub-agent contract boundaries directly in quest objective definitions. Adopted from cc-sdd's `_Boundary:_` / `_Depends:_` annotations to stop sub-agent drift mid-run. Additive — existing objectives without annotations are unaffected.
- **summarizeRelevantContext** — SA-dispatched action. Reads the user's most recent message + prior conversation, extracts entities (URLs, page names, objective hints, external inputs), and produces a structured context bundle the rest of the cascade consumes. Output shape matches the new Description / Objectives / Deliverables model.
- **tickWorkerDispatch** — The local worker-dispatcher tick. GM fires this when the user gives an execution go-ahead on an existing execute-stage quest (e.g. "looks good, go"). One tick = scan execute-stage quests assigned to Guildmaster, load the lowest-n pending objective (or resume a running one), single worker, no parallel fan-out.
- **upsertQuest** — SA-dispatched action. Runs AFTER questmaster.evaluate has passed (smartJson available). Names the quest, searches for an existing match, creates a new one if no match. No orchestration-graph compute under the new model — objective ordering lives in objectives.n directly.
- **workScopePerTick** — Strategic: the scope a worker drives toward each tick — the acceptable stop state (ruleAcceptableStopState). Don't stop at a single objective or a natural pause. LOCKED 2026-06-12.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/questmaster/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/skill_book/questmaster`

Rationale layer (per CLAUDE.md §6). The WHY behind the questmaster procedures
(howTos). Append; supersede in place when a decision changes.

---

## 2026-09-15 — `runReviewApproval` names the GuildOS2 user-channel attestation; the `questReview` weapon is legacy

**Decision.** `runReviewApproval` now documents ONE route for approving a GuildOS2
quest at `stage='review'`: the user-channel op
`{table:"quest", mode:"update", fields:{stage:"reflect"}}` — verbatim what
`approveReviewAction` (`GuildOS2/app/(app)/gm-room/actions.js`) sends — reached over
the wire by `POST /api/quest/test` with `action:"mutate"`, `channel:"user"`, or by
the operator clicking **Approve → reflect** in the GM Room. The howTo additionally
names each dead end WITH ITS REASON, and labels `libs/weapon/questReview` legacy:
that weapon targets the original GuildOS (council database, retired steps/items
model, `FINAL_GATE_PASS` lockphrase, `enforce_stage_gate` trigger) and advances
`review → closing`, a stage GuildOS2 does not have.

**Rejected alternative (1).** Keep pointing at `questReview.confirmApproval` →
`pass` → `closing`. Rejected: it cannot approve a GuildOS2 quest at all, and on
2026-09-15 it was the matcher's TOP HIT for "approve a quest at review stage in
GuildOS2" — the skill was found and it misdirected, which is worse than a miss.

**Rejected alternative (2).** Keep the old rule "never a direct `quests.stage`
write; the stage is always derived." Rejected as false of exactly one stage:
`deriveStage` has no rule that leaves `review`. Review's exit is an ATTESTATION,
and writing facts instead — the advice the worker channel's refusal message gives —
can never produce it. The rule survives everywhere else and is restated as such.

**Rejected alternative (3).** Approve by driving the UI with a Playwright login as
`GUILDOS2_VERIFIER_*`. Rejected on measurement: RLS is owner-scoped, the verifier
owns nothing, and the quest reads back as `[]`. Minting a session for the
operator's own account stays forbidden (it evicts their browser —
browsercontrol/leafBrowserControl), and CIC cannot be inherited by a headless leaf.

**Why.** The approval is a one-line write whose whole difficulty is knowing WHICH
channel carries it. `no_worker_stage_assertion` is HARD but `channels: WORKER_ONLY`,
so the identical op is refused on one channel and lands on the other. Without that
named in the skill, a worker reads the refusal's own "write the facts instead"
advice, finds no fact that works, and either escalates or gives up. Recording the
dead ends is as load-bearing as recording the route.

**Consequence if violated.** A quest sits at `review` indefinitely while workers
re-derive the same investigation — quest `c9d6a1a1` waited six days — or a worker
takes the forbidden shortcut (service key, operator session) to get past it.

---

## 2026-06-13 — createQuest must capture the user's grants of autonomy into the description

**Decision.** The `clarifyQuest` howTo's PASS branch mandates that when composing
a quest description, the GM scans the request for **delegated/pre-approved/
range-granted decisions** and records them verbatim under an explicit
`AUTONOMY / PRE-APPROVED DECISIONS:` section — not buried mid-prose.

**Rejected alternative.** Pack only the positive spec (chapters, methods,
language) into the description and let grants of autonomy live only in chat.

**Why.** The chat is ephemeral; the quest is the durable record. A grant that
lives only in chat ("5–10 chapters, your call") is re-asked by the next stage. An
explicit, unmissable section lets the planner and executor honor the latitude
without re-litigating it. This is the upstream capture; the planner's
reversibility filter (`libs/weapon/questmaster/engineering_decisions.md`) is the
downstream net.

**Consequence if violated.** Delegated decisions get re-surfaced as questions
(observed 2026-06-13: chapter count asked despite "agent's discretion").

**Residual — RESOLVED 2026-06-13.** `planQuest`'s `conciseDescription` shrink used
to strip the `AUTONOMY / PRE-APPROVED DECISIONS:` block (grants absorbed into
objectives), so a re-plan lost the explicit signal. Fixed in
`libs/weapon/questmaster/index.js → planQuest`: (a) the planner systemPrompt rule 1
now mandates keeping the block verbatim, and (b) a deterministic net
(`extractAutonomyBlock` + `AUTONOMY_BLOCK_RE`) strips whatever block the shrink
emitted and re-attaches the operator's ORIGINAL block verbatim before the
description write — so the grants are always present, exact, and never duplicated.

---

## 2026-06-15 — unblocking questions go through `raiseUnblockingQuestionDiplomatically`; most candidates are bad and must be self-resolved

**Decision.** A worker/planner may bump a quest to `escalated` with an
`unblocking` question ONLY by following the `raiseUnblockingQuestionDiplomatically`
howTo: (1) the candidate must PASS `sage.askSage` *before* the row is written —
PROCEED ⇒ the agent decides and records, never asks; only an ESCALATE verdict
(irreversible/business-damaging, or user-only knowledge) earns a question; and
(2) the written body must state, labelled, **WHAT the user needs to do** and **WHY
the agent can't do it**. The default posture is *resolve it yourself* — a
well-specified quest should usually raise ZERO questions.

**Rejected alternative.** The prior loose instruction ("if genuinely blocked on a
user decision, create the unblocking question") with "genuinely" undefined and no
required shape — which let any deferred decision or self-answerable fact become a
to-user question.

**Why.** An unblocking question stalls a quest until a human acts; it is the most
expensive thing a worker can emit. Auditing the escalated quests (2026-06-15)
found the large majority were **manufactured stops** — decisions the agent could
and should have made. This is `sage.askSage` (the chat-side question gate) applied
to the quest substrate; previously the gate governed chat but not question
*generation*. (Complements the upstream autonomy-capture decision above and the
planner's reversibility net in `libs/weapon/questmaster/engineering_decisions.md`.)

**Consequence if violated.** Quests pile up bare "Which X?" questions and sit in
`escalated` waiting on calls the agent should have made; the user is forced to
adjudicate reversible defaults, answer facts the agent could look up, or — worst —
do work the agent falsely claimed it couldn't.

### Catalogue of BAD unblocking questions (anti-patterns — grow this list)

Each entry: the question, the failure class, the lesson. **This list is meant to
GROW** — when a bad unblocking question is caught in the wild, add it here (quote
+ class + lesson) so the catalogue keeps sharpening
`raiseUnblockingQuestionDiplomatically`. Seeded 2026-06-15 from the first audit:

1. **Supabase `db push` (quest "Develop the Caravan system").** Question: *"run
   supabase db push to apply migrations 20260612160000 + 20260612170000"*, marked
   *"OPERATOR ACTION (holds DB password) — escalate, do not run autonomously."*
   **Failure class: FALSE "agent can't."** The premise was wrong: the agent
   obtained full Supabase access on its own (minted a personal access token via
   CIC, reset the DB password, applied the migrations through the Management API)
   and resolved it without the user. **Lesson:** before writing "the agent can't /
   only the operator can," challenge the can't — scan for a capability path
   (skill book, weapon, get-access route) first; a missing credential is usually
   *obtainable*, not a true block. (See `~/.claude/CLAUDE.md` challenge-your-cant.)

2. **RAG primitive (quest "Add an explicit RAG / memory retrieval primitive").**
   Four questions: which corpora to search, BM25 vs embeddings, the snippet
   struct, the first integration target. **Failure class: manufactured
   design-deferral** — all reversible defaults the description *explicitly
   delegated* ("**Decide** the corpus scope… and the retrieval method…"); plus a
   self-answerable fact ("are embeddings available?" → go grep) and a ratification
   (the question proposed its own schema). **Lesson:** a delegated/reversible
   design choice is the agent's to make and record in the objective `notes`, not
   the user's to adjudicate.

3. **Rescuer escalation (quest "Set up cron for quests").** Question: *"execute is
   stalled — no active objective… how should we proceed — clarify/relax/accept/
   drop?"* on a quest whose 6 objectives were ALL `done`. **Failure class:
   undiagnosed mechanical escalation** — a vague how-should-we-proceed menu with
   no diagnosis; the quest was simply complete. **Lesson:** diagnose the actual
   state before escalating; never offer a "how to proceed?" menu in place of
   finding the answer.

## 2026-06-15 — a blocked objective is an OPEN objective + an unblocking question, NOT an `attention` status

**Decision.** The objective `status='attention'` is RETIRED. A blocked objective
stays **OPEN** (`active`/`pending`); the block is carried *solely* by an open
`unblocking` `quest_questions` row (`unblocks_objective_ids` → the objective) plus
the quest at `stage='escalated'`. That pair is the single source of truth for
"this objective is blocked." Enforced in the questmaster weapon: `planQuest` and
`patchObjective` whitelists no longer accept `attention` (a patch with that status
returns `invalid status 'attention'`). Migration `20260615210000` re-opened the
existing rows (`attention → active`); no `objectives.status` CHECK constraint
exists, so app-layer enforcement is the whole fence.

**Rejected alternative.** Keep `attention` as a third objective status alongside
`done`/`active`. It read as the obvious way to flag a blocked row, and the UI +
worker prose already used it.

**Why.** `attention` was a SECOND source of truth that *diverged* from the real
block signal. Two failure modes seen live (2026-06-15): (a) an objective marked
`attention` with **zero** unblocking questions and the quest left in `execute` — a
silent dead-end that never escalated, because nothing required the question to
exist (e.g. quest `988c9792` obj n2); (b) where used "correctly" it was pure
redundancy — the unblocking question + `escalated` stage already blocked the
quest, so the status added nothing. Worse, `attention` is **not** in the
dispatcher's actionable set nor the derail ladder's `ACTIVE_OBJECTIVE_STATUSES`,
so a quest whose only open objective was `attention` read as "no active objective"
— invisible to the healthy-execute dispatcher and a permanent derail-ladder churn
target. Keying solely on the open unblocking question + escalated stage (the
ladder's gate 1 already uses it) removes the divergence.

**Consequence if violated.** Re-introducing an objective "blocked" status
recreates the dual source of truth: rows that are "blocked" but un-escalated
(invisible blocks the user never sees) and quests stranded in `execute` because
their sole open objective doesn't count as active. To know whether an objective is
blocked, ask "does it have an open unblocking question?" — never add a status for
it.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/questmaster/index.js -->
```js
/**
 * Questmaster skill book — Cat uses these actions for planning and assignee matching.
 * Includes JSON extraction helpers shared with plan/find flows.
 */
import { runDungeonMasterChat } from "@/libs/council/ai/chatCompletion.js";

/** Server logs for tracing planRequestToQuest (grep: GuildOS:planRequestToQuest). */
const LOG = "[GuildOS:planRequestToQuest]";

/** @param {unknown} s @param {number} [max] */
function previewText(s, max = 280) {
  const t = s == null ? "" : String(s);
  return t.length <= max ? t : `${t.slice(0, max)}…`;
}

/**
 * Pull a JSON object from model text: tries ```json fences, then scans for balanced `{...}` and parses.
 * Expects objects with at least `title` and `deliverable` (quest-plan shape).
 * @param {string} text
 * @returns {Record<string, unknown>|null}
 */
function extractQuestPlanJson(text) {
  if (text == null || typeof text !== "string") return null;
  let s = text.trim();
  const fence = /^```(?:json)?\s*([\s\S]*?)```/im.exec(s);
  if (fence) s = fence[1].trim();

  const start = s.indexOf("{");
  if (start === -1) return null;

  for (let i = start; i < s.length; i++) {
    if (s[i] !== "{") continue;
    let depth = 0;
    for (let j = i; j < s.length; j++) {
      const ch = s[j];
      if (ch === "{") depth++;
      else if (ch === "}") {
        depth--;
        if (depth === 0) {
          const slice = s.slice(i, j + 1);
          try {
            const obj = JSON.parse(slice);
            if (
              obj &&
              typeof obj === "object" &&
              !Array.isArray(obj) &&
              "title" in obj &&
              "deliverable" in obj
            ) {
              return obj;
            }
          } catch {
            /* try next opening brace */
          }
          break;
        }
      }
    }
  }
  return null;
}

export const definition = {
  id: "questmaster",
  title: "Questmaster",
  description:
    "Plan user requests into quest-shaped JSON and pick roster adventurers by capability.",
  toc: {
    clarifyQuest: {
      description:
        "THE action for creating a quest from a chat session (the pre-creation gate, run BEFORE createQuest). A quest-creating chat has ONE job: nail a high-fidelity DESCRIPTION plus the user-owned deliverables — then createQuest({ownerId,title,description}) births the quest at stage='plan' and the planQuest substrate itemizes the objectives. Do NOT author objectives in chat, and the approval proposal you show the user is Name + Description + Deliverables only (objectives are the planner's job, not the chat's). Gate: evaluate whether {name, deliverables, methods} are defined and feasible; if not, ask clarification questions instead of creating a half-formed quest.",
      howTo: `
## clarifyQuest — is this quest well-defined and feasible? (run BEFORE createQuest)

A quest is only worth creating if a worker could actually plan and execute it. Before \`questmaster.createQuest\`, evaluate the user's instruction on THREE axes:

1. **name** — is there a clear, single objective? ("Create a picture book for Zhangfei" ✓; "make stuff better" ✗).
2. **deliverables** — is the concrete final artifact(s) named and content-typed? ("a PDF picture book" ✓ → kind=document/pdf). If the user named none, can you infer exactly one obvious deliverable? If not, ask.
3. **methods** — are the methods for PRODUCING each deliverable specified or obviously inferable, AND feasible with the guild's tools? (e.g. "ChatGPT-image-2 for image fill, native AI for chapter text, Chinese, 150–200 words/chapter, 5 chapters" ✓ — every method maps to a known capability). Flag any method that no skill book / weapon can perform.

### Decision
- **PASS** (all three defined + feasible) → proceed to \`questmaster.createQuest({ ownerId, title, description })\` with the full spec packed into the **description** (chapters, per-chapter constraints, image/text methods, language, the single deliverable). Do NOT pre-create objectives/deliverables — the quest is born at stage='plan'; planQuest itemizes later. Assignee defaults to Guildmaster unless a specific adventurer was requested.

  **THE CHAT APPROVAL PROPOSAL IS DESCRIPTION-FIRST — NEVER AUTHOR OBJECTIVES IN CHAT (LOCKED 2026-06-14).** When you show the user a quest for approval in a chat session, present exactly THREE blocks: **Name**, a high-fidelity **Description**, and the user-owned **Deliverables**. Do NOT enumerate objectives in the proposal. Objectives are the output of the LOCKED planning substrate (\`planQuest\`, run by the planning tick on the plan-stage quest); hand-authoring them in chat jumps that substrate, skips the engineering-decision best-practices it applies, and forces the user to mentally audit a plan they never asked you to write — wasted effort. So a quest-creating chat has ONE job: nail the DESCRIPTION with high fidelity (plus the user-owned deliverables). On approval, \`createQuest({ ownerId, title, description })\` lands the quest in \`plan\` and the planning tick itemizes the objectives. (The user owns deliverables + method; the planner owns objectives.)

  **CAPTURE THE USER'S GRANTS OF AUTONOMY (do not lose them).** The user often states, in their ask, decisions they have ALREADY delegated, pre-approved, or made — and they should never have to repeat these. Scan the request for them and record them VERBATIM in the description under an explicit, clearly-labelled section so every downstream stage (planning, execution) honors them and does NOT re-surface them as questions. Capture:
  - **Delegated decisions** — "up to the agent / your discretion / you decide / your call", or a stated RANGE that grants latitude ("5–10 chapters", "roughly a page each"). A range or a "you choose" is a DECISION ALREADY MADE (the agent picks within it), never an open question.
  - **Pre-approvals / standing permissions** — "you don't need to ask about X", "go ahead and …", "I pre-approve …", "always do Y", any blanket authorization to proceed without checking back.
  - **Fixed constraints the user has already settled** — values they gave so they won't be re-litigated (audience, language, tone, format, budget ceiling).
  Write these into a section the planner can't miss, e.g. \`AUTONOMY / PRE-APPROVED DECISIONS:\` followed by a bullet list. This is the durable record — the chat is ephemeral, so a grant that lives only in chat is a grant the next stage will re-ask. Burying "agent's discretion" mid-sentence is how it gets ignored; elevate it.
- **FAIL** (something underspecified or infeasible) → do NOT create a broken quest. Surface CLARIFICATION QUESTIONS to the user, each naming exactly what is missing and why it blocks creation (e.g. "How many chapters?", "What is the final format — PDF, web, slides?", "Image source — generate with ChatGPT-image-2, or supply your own?"). Only after the user answers, re-run this gate, then createQuest.

### Why a gate, not a guess
Creating a vague quest pushes the ambiguity downstream to a planning leaf that can only guess — producing the wrong deliverable. One clarification round at creation is cheaper than a full plan→execute→bounce cycle. This is a procedural rule (tunable here), not schema.
`,
    },
    deliverableStoragePolicy: {
      description:
        "Where a worker stores a produced deliverable (local filesystem vs Supabase Storage). The BASE preference is owned here (questmaster skill book); each worker may hold its own preference on top. LOCKED 2026-06-11.",
      howTo: `
A deliverable, once produced, must be stored somewhere its \`ref\` can point at. WHERE is **owned by the worker** — different workers have different storage preferences — but the BASE preference lives here so a worker with no opinion has a safe default.

## The base preference (LOCKED)
- **Guildmaster** (the local operator-run worker) MAY store **either locally** (a path under the repo / a served \`public/\` location) **OR in Supabase Storage** (\`GuildOS_Bucket\`). Its choice; local is fine because the operator reviews on localhost.
- **Every other worker** (Cursor cloud agents, dispatched leaf/SA/MA workers, any non-local executor) **MUST store in Supabase Storage** — they have no durable local filesystem the user can reach, so a local path would be a dead \`ref\`. Upload to \`GuildOS_Bucket\` (public) and set \`deliverables.ref\` to the public URL.

## How a worker applies it
1. Read your own storage preference (if your role/skill book defines one). If you have none, fall back to this base preference.
2. Produce the artifact, then store per the rule above:
   - Supabase Storage: upload to \`GuildOS_Bucket\`, get the public URL, UPSERT the \`deliverables\` row (name, kind, ref=publicURL, status='done').
   - Local (Guildmaster only): write under a served location, set ref to the local/served path, status='done'.
3. The deliverable's \`kind\` drives how the results carousel renders it (pdf/ppt/img embed; html/text iframe). A non-public local ref will NOT embed for remote reviewers — another reason non-Guildmaster workers must use Supabase Storage.

## verify_screenshot_ref storage (LOCKED — ALL workers, including Guildmaster)
\`verify_screenshot_ref\` MUST ALWAYS be a **public Supabase Storage URL** (https://*.supabase.co/storage/*) — no exceptions, not even for the Guildmaster. Upload to \`GuildOS_Bucket\` (path convention: \`quest-assets/<questId>/verify/<deliverableId>.png\`) via the \`supabase_storage\` weapon, then set \`verify_screenshot_ref\` to the public URL. A local file path, \`file://\` URL, or any non-Supabase URL is NEVER valid for \`verify_screenshot_ref\`. The golden path is \`questPurrview.autoVisualVerify\` (headless, always uploads); manual operator-browser screenshots must be uploaded the same way before setting the field.

## Why it lives here
Storage location is a *procedural* rule (tunable ad hoc), not schema — so it is a questmaster skill-book howTo, not a hardcoded column. A worker overrides the base only by holding an explicit preference in its own role/skill book; absent that, this base governs.
`,
    },
    assetInlineVsFileStorage: {
      description:
        "When a produced asset/result is stored INLINE (text in the row's ref) vs in a FILE. Threshold: text < 1000 chars → inline; otherwise (or any binary) → file. LOCKED 2026-06-12.",
      howTo: `
When a worker produces a result/asset, choose INLINE vs FILE by size:

- **< 1000 characters of text → store INLINE.** Put the text directly in the asset/deliverable \`ref\` (e.g. a short story, a one-paragraph summary, a small JSON snippet). The value is small enough to live in the row and render directly — no upload needed.
- **>= 1000 characters, OR any binary / non-text artifact (image, pdf, ppt, large html) → store in a FILE.** Write it out and set \`ref\` to the file's location. WHERE the file lives is the worker's call per the \`deliverableStoragePolicy\` howTo: Guildmaster may store locally OR in Supabase Storage; every other worker MUST use Supabase Storage (a public URL).

This keeps small results legible inline while large/binary ones get a durable, linkable file. The 1000-char threshold is a procedural rule (tunable here), not a schema constraint.
`,
    },
    ruleComments: {
      description:
        "LOCKED 2026-06-12: quest_comments are the STRATEGIC, FUTURE-BEARING channel — NOT an action log. A comment earns its place only if a future reader (the next worker or the user) gains actionable insight from it. Write what was DECIDED + what to DO NEXT, never bookkeeping of what happened.",
      howTo: `
## Comments are strategic, not a log (LOCKED 2026-06-12)

A \`quest_comments\` row must carry **forward insight** — a decision, a blocker the next actor must clear, what to do next, a verdict + what to fix, the user's directive. It is the durable hand-off the next worker and the user read.

**DO write a comment for:**
- The worker's end-of-burst summary — Done / Next / Blocked / Learned (the canonical hand-off shape).
- A review verdict + WHAT TO FIX (bounce reason), an escalation + what the user must decide, a transition-blocked note naming what's missing to proceed.
- The user's update request / directive (recorded comment-first before a mutation).
- A closure summary (what the quest accomplished, for the record).

**DO NOT write a comment for** (these are action-LOGS — the event is recorded elsewhere; the reader gains nothing forward):
- "Dispatched a worker", "reclaimed a stale lease", "bounced stage X→Y", "decomposed into N objectives", "stage advanced". The stage change lives in \`quest_stage_log\`; the lease in \`worker_id\`; the run in \`cron_log\`; the plan IS the objectives/deliverables rows. A line restating those carries no forward insight — omit it.

Litmus: *would the next worker or the user act differently because this comment exists?* If no → don't write it. Bookkeeping belongs in the audit tables (quest_stage_log / cron_log) and the field values themselves, not in the comment thread.`,
    },
    ruleCheckpointAssets: {
      description:
        "Two-layer asset-archiving policy (decided 2026-06-12). LAYER 1 (primary): a worker proactively saves any result a LATER objective will consume as a quest_asset. LAYER 2 (recovery): if a later objective finds the asset missing, it REACTIVATES the prerequisite objective with a note to archive, and retries. Chosen over a hard move-to-review gate. LOCKED 2026-06-12.",
      howTo: `
## The decided policy (2026-06-12): two soft layers, NOT a hard gate
We deliberately did NOT add a hard \`moveQuestToReview\` block on \`quest_assets\` count (too blunt — many quests have no intermediates). Instead, two prompt-level layers:

**LAYER 1 — save forward (primary).** When a worker produces a result that a LATER objective will consume, it saves that result as a \`quest_asset\` proactively, before moving on. This is the main path and closes the common case.

**LAYER 2 — recover backward (fallback for the edge case where Layer 1 was missed).** When a worker reaches an objective that NEEDS an upstream artifact with NO \`quest_asset\`, it does NOT silently regenerate or fabricate. It instead:
1. **Reactivates the prerequisite objective** — set that objective's \`status\` back to \`active\`.
2. **Adds a note** on the prerequisite objective (its \`notes\`) that it MUST archive its result as a \`quest_asset\` this time.
3. **Leaves a comment** recording the gap, then lets the next tick re-run the prerequisite (which now archives the asset). Only once the asset exists does the consuming objective proceed.

This self-heals the missing-asset edge case without a blunt global gate. Both layers live in \`buildLeafExecutePrompt\` step 4.

## Layer-1 mechanics
A result a later objective will consume — especially each component artifact before an **assembly or composition step** (PDF build, deck merge, multi-image collage, multi-chapter document) — that took a non-trivial tool call to produce MUST be saved as a \`quest_assets\` row.

## What counts as "non-trivial"
Any artifact produced by: image generation, long text generation (>100 words), file creation, API call to an external service, or any operation that would take >30 seconds to reproduce. Quick inline computations or string formatting do NOT require checkpointing.

## The procedure (one call — \`checkpointQuestAsset\` handles inline-vs-file for you)
1. **Produce** the artifact (image, text chunk, data file).
2. **Call \`questmaster.checkpointQuestAsset({ questId, label, content, filename?, kind?, source, actorName })\`.** It applies \`assetInlineVsFileStorage\` automatically: short text (< 1000 chars) is written INLINE into the row's \`ref\` (no upload, \`filename\` optional, \`kind\` defaults to \`"text"\`); larger/binary content uploads to GuildOS_Bucket and \`ref\` becomes the public URL (\`kind\` defaults to \`"document"\`). Do NOT hand-roll the upload+insert.
3. **\`kind\` uses the 5-kind consumption taxonomy** — \`"text"\` (read it — incl. a URL/link, json, markdown, a short story), \`"html"\`, \`"image"\`, \`"video"\`, \`"document"\`. The retired \`"link"/"json"/"markdown"/"file"/"screenshot"\` kinds all collapse into one of these five; never write \`kind:"link"\`.
4. **Then** proceed to the assembly step that consumes the artifact.

## Enforcement points
- **buildLeafExecutePrompt** injects this rule so every leaf worker sees it.
- **moveQuestToReview soft gate**: if the quest has composite deliverables (pdf/html) but zero quest_assets, the gate writes a warning comment ("intermediates may be missing — checkpoint artifacts for reassembly"). This is a soft gate (warning, not block) because some quests genuinely have no intermediates.

## Why
The Guan Yu quest produced 8 images + 8 stories, assembled a PDF, but saved zero quest_assets. When the PDF needed reassembly, all images had to be regenerated. Checkpointing intermediates eliminates this waste.
`,
    },
    raiseUnblockingQuestionDiplomatically: {
      description: "Raise a question only when the quest is unlikely to succeed without the user's intervention; otherwise, keep working.",
      howTo: `Raise a question only when the quest is unlikely to succeed without the user's intervention; otherwise, keep working.
State the blocker, why user intervention is necessary, and the concrete action or decision needed.
Use the current lease's question fields and write-back API. Reuse an open question covering the
same blocker, keep incomplete objectives open, and link them with unblocks_objective_ids.
No separate sage eligibility verdict or escalation scenario is required.`,
    },
    ruleAcceptableStopState: {
      description:
        "The criteria a worker burst MUST reach before it may stop. Every objective either done, or open-and-blocked-by-a-linked-unblocking-question; every block externalized as an unblocking question; exactly one stop-summary comment (edit the prior failed-stop comment, don't pile up). LOCKED 2026-06-12; objective `attention` status retired 2026-06-15.",
      howTo: `
A worker burst may STOP only when the quest is in an **acceptable stop state**. All three criteria must hold:

1. **Every objective is either \`done\` or a genuinely-blocked OPEN objective** — none left \`pending\`/\`active\` and unworked. You either finished it, or it is blocked (left OPEN, NOT marked done) and carries a linked unblocking question. There is no \`attention\` status — a blocked objective is just an open objective with a linked open unblocking question.
2. **Every BLOCKED objective is mapped to a \`quest_questions\` row of type \`unblocking\` RAISED VIA \`raiseUnblockingQuestionDiplomatically\`** — a blocked objective MUST have a corresponding unblocking question (put its id in the question's \`unblocks_objective_ids\`), and that question MUST have been authored through \`raiseUnblockingQuestionDiplomatically\` (met the quest-success escalation principle, and its body states WHAT-you-need-to-do + WHY-I-can't). An OPEN objective whose only "block" fails that gate is NOT a real block — resolve it yourself and finish the objective. An open, unfinished objective with NO conforming unblocking question is NOT an acceptable stop (it is workable — keep going).
3. **Exactly ONE stop-summary comment per stop** — on reaching an acceptable stop, write ONE \`quest_comments\` row summarizing what happened (done / blocked + why / what next). **If the previous agent FAILED to reach an acceptable stop and left a stop-summary comment, EDIT that comment in place instead of creating a new one** — no comment pile-up; the latest stop summary is the single durable handoff.

Routing once the state holds: all objectives \`done\` + deliverables filled → \`moveQuestToReview\`. One or more objectives blocked by a linked unblocking question and nothing else workable → \`moveQuestToEscalated\`. If objectives are still workable, you have NOT reached an acceptable stop — keep going.
`,
    },
    workScopePerTick: {
      description:
        "Strategic: the scope a worker drives toward each tick — the acceptable stop state (ruleAcceptableStopState). Don't stop at a single objective or a natural pause. LOCKED 2026-06-12.",
      howTo: `
**Each worker drives toward ONE end state per tick: the acceptable stop state** (see \`ruleAcceptableStopState\`). Do NOT stop at a "natural pause" or after a single objective — push the quest until EVERY objective is \`done\` or blocked-by-a-linked-unblocking-question, every block has an \`unblocking\` question, and the single stop-summary comment is written (or the prior failed-stop comment edited).

Per tick:
1. Pick the lowest-n workable objective (open — \`pending\`/\`active\` — with no open unblocking question) and work it to \`done\`. If you think it is blocked, do NOT just create a question — run \`raiseUnblockingQuestionDiplomatically\`: the quest-success principle decides whether user intervention is necessary (record in \`notes\` and finish the objective). Only when the quest is unlikely to succeed without user intervention do you author the \`unblocking\` question with the needed action and why intervention is necessary, **leave the objective OPEN**, and link its id (\`unblocks_objective_ids\`) — the open question, not a status, marks it blocked.
2. Repeat until no objective is workable (all \`done\` or blocked-by-a-linked-unblocking-question).
3. Reach the acceptable stop state: write/edit the one summary comment, then \`moveQuestToReview\` (all done + deliverables filled) OR \`moveQuestToEscalated\` (blocked) OR release the lease for the next tick only if progress is still possible but your budget is spent.

The worker's strategic objective: maximize progress toward done-or-blocked-with-a-question every tick, never leave a workable objective untouched, and externalize every block as an unblocking question rather than stalling silently.
`,
    },
    questionDigestion: {
      description:
        "Questions are TRANSIENT but their DECISIONS are permanent. Digestion = a 4-step capture pass: extract decisions from every answer → MAP each to description / a specific objective / new objective(s) / assets → VERIFY the main points are captured → only THEN remove (unblocking only). Deleting an answer before its decisions are migrated is a DATA-LOSS defect. LOCKED 2026-06-13.",
      howTo: `
A \`quest_questions\` row is **transient**, but the **DECISIONS its answer carries are permanent** and MUST end up in the quest's durable state (description / objectives / deliverables / quest_assets) BEFORE the question is removed. Deleting an answered question whose answer was not migrated destroys a decision the user gave — a DATA-LOSS defect (this is exactly what failed on quest 5d29b94f: the age/style/layout answers were deleted without migration and lost).

Run digestion as the FIRST thing in a burst, before any objective work, as four explicit steps:

**1. EXTRACT.** Read EVERY answered question together with its answer (both \`unblocking\` and \`user\` rows). Build a flat **list of decisions** the answers establish — each a concrete, load-bearing fact or instruction (e.g. "target age 7–10", "illustration style = chibi, consistent across chapters", "no source-episode preference", "one image + story per PDF page"). One answer may yield several decisions.

**2. MAP + APPLY.** Route each decision to where it belongs and apply it as a mutation:
- **Global, or affects multiple objectives → the DESCRIPTION.** Edit the quest description (the global-instructions field every worker reads).
- **Affects ONE specific objective → that OBJECTIVE.** Write it into that objective's \`method\`/\`notes\`/\`name\` as appropriate (and resolve/modify its status), linked via \`unblocks_objective_ids\` where set.
- **Introduces NEW work (one or a few new things) → ADD new objective(s)** for them.
- **Establishes or changes a needed artifact/input → the DELIVERABLES / quest_assets.** Add or modify the deliverable or quest_asset the decision implies (answer digestion can change assets too).
Map each decision individually; a single answer can land in several places.

**3. VERIFY CAPTURE (double-check before removal).** Before deleting any question, re-read it and confirm its MAIN POINTS are now actually present in the quest (in the description / the objective / a new objective / a deliverable or asset). If any point is missing, capture it first. Only when every main point is durably recorded may you proceed to remove.

**4. REMOVE — unblocking only, after step 3.**
- **\`unblocking\` (agent→user):** once fully captured + verified, remove it ONLY via \`questmaster.digestAndRemoveQuestion({questId, questionId, migrationNote})\` — the migrationNote names where each decision landed. That action is the SOLE sanctioned removal path: it REFUSES without a migrationNote and archives the Q+A+note into quest_comments BEFORE deleting (and a DB BEFORE-DELETE trigger archives the row to \`quest_question_archive\` no matter the caller). **NEVER raw-delete \`quest_questions\` (\`client.from('quest_questions').delete()\`) and NEVER call \`DELETE /api/quest/questions\` directly** — both bypass the digest guard and risk destroying the user's decision (the 5d29b94f DATA-LOSS defect).
- **\`user\` (user→agent):** the worker's ONLY job is to **answer** it (write the \`answer\` field), BEFORE executing any objective; still EXTRACT + APPLY its decisions in steps 1–3. The worker has **NO authority to delete or resolve a user question** — only the user removes it after reading the answer. Enforced: \`deleteQuestQuestionById\` rejects a worker delete of a \`qtype='user'\` row (the authed UI passes \`allowUserType:true\`). NEVER work around the guard with raw service-role REST. (See \`ruleUserQuestionAuthority\`.)

Invariant (LOCKED): **a question is removed ONLY after its decisions are migrated into the quest's durable state and verified present** — never delete first and hope. And **no user question is ever deleted by a worker**. Questions accumulate only while OPEN; once answered they exist only until digested (unblocking) or until the user removes them (user). See \`ruleAcceptableStopState\`, \`workScopePerTick\`, and \`ruleUserQuestionAuthority\`.
`,
    },
    ruleUserQuestionAuthority: {
      description:
        "AUTHORITY boundary: a worker may ANSWER user-type questions but may NEVER delete or resolve them, and must answer every open user question BEFORE executing objectives. Only the user removes a user question. Enforced by the deleteQuestQuestionById guard. LOCKED 2026-06-12.",
      howTo: `
A \`quest_questions\` row with \`qtype='user'\` is a question the USER asked the worker. It defines an **authority boundary**:

1. **A worker may only ANSWER it.** Write the reply into the question's \`answer\` field (PATCH /api/quest/questions or \`updateQuestQuestion\`). Answering is the worker's entire job on a user question.
2. **A worker may NEVER delete or resolve a user question.** Removal is the USER's act, done after they read the answer. A worker that deletes a user question has overstepped its authority and destroyed a record the user had not yet seen.
3. **Answer user questions BEFORE executing objectives.** They are inbox items the user is waiting on; clear them first (see the executeQuest step-3 checklist + \`questionDigestion\`). An objective must not be worked while an open user question sits unanswered.

This is **enforced, not advisory**: \`libs/council/database/serverQuest.js\` → \`deleteQuestQuestionById\` refuses to delete a \`qtype='user'\` row unless the caller passes \`allowUserType:true\`. Only the authed UI delete route (\`DELETE /api/quest/questions\`, gated by \`requireUser\`) passes it — i.e., only a real user action. A worker calling the helper (or the route, which it can't auth to) gets an error. A worker MUST NOT bypass the guard with raw service-role REST against \`quest_questions\` — that violates \`ruleDoNotImprovise\` and this rule.

Contrast \`unblocking\` questions (the worker's own questions to the user): those the worker DOES remove, but only after digesting the user's answer (see \`questionDigestion\`). The delete-guard only fences \`user\`-type rows.
`,
    },
    commentDigestion: {
      description:
        "How a worker handles quest_comments. A source='user' comment is a DIRECTIVE: follow it, then mark resolved=true (NEVER delete — comments are the durable audit log). Contrast questionDigestion (questions are removed). Comments render newest-first. LOCKED 2026-06-12.",
      howTo: `
\`quest_comments\` is the quest's **durable audit + handoff log** — the narrative of what the user asked and what each worker did. Unlike \`quest_questions\` (transient Q&A that is REMOVED once digested), a comment is KEPT. Two comment lifecycles:

## User comments are directives — digest, then RESOLVE (don't delete)
A \`quest_comments\` row with \`source='user'\` and \`resolved=false\` is a **directive the worker must follow**. On a burst, BEFORE working objectives:
1. Read every comment (the list renders newest-first; the UI \`GET /api/quest/comments\` and \`getQuestDetails\` both order \`created_at\` DESC).
2. For each unresolved user comment: do exactly what it says — mutate objectives / deliverables / code / schema as the directive implies.
3. Mark it **resolved** via \`questmaster.resolveQuestComment\` (sets \`quest_comments.resolved=true\`). **Do NOT delete it.** The record of "user asked X → it was done" is the point; a resolved comment greys out in the UI but stays as history.
4. Record what you changed in a NEW comment (your stop-summary or a dedicated note).

**Why resolve, not remove (LOCKED recommendation):** a question's whole value migrates into objectives, so the question is disposable. A comment's value is partly the record itself — the audit trail of intent and action. Removing it loses history; resolving preserves it while signalling "handled". So: questions → remove on digest; user comments → resolve on digest.

**Provisional reinforcement (2026-06-12):** the "never delete, only resolve" stance is doubly anchored WHILE the quest workflow is still stabilizing — the full comment trail (even resolved rows) is debugging signal for the workflow itself, so nothing is pruned for now. This is **revisitable once the workflow is proven stable**: a later iteration may archive or prune long-resolved comments. Until then, resolve-and-keep is the rule; do not add a delete-on-digest path for user comments.

## Worker/system comments are pure log — never resolve, never delete
\`source\` of \`adventurer\` / \`guildmaster\` / \`system\` comments (stop-summaries per \`ruleAcceptableStopState\`, tick-dispatch notes) are append-only log entries. They carry no \`resolved\` affordance in the UI; leave them.

Ownership: the WORKER digests user comments (this rule). The USER reads worker/system comments. See \`questionDigestion\` for the question half of the inbox, and \`ruleAcceptableStopState\` (the one-stop-summary-comment rule).
`,
    },
    ruleCommentValue: {
      description:
        "Every quest_comment must be INSIGHT-BEARING + ACTIONABLE — it tells the next step what to do. Pure mechanical event-logs (lease reclaim, leaf dispatch) are NOT comments; they live in worker_id / console / cron_log. Don't pile up: edit a repeat in place. LOCKED 2026-06-12.",
      howTo: `
A \`quest_comments\` row earns its place ONLY if it bears value for the NEXT step — insight (what was learned/verified) AND/OR an instruction (what to do next). The test: *would the next worker/reviewer/user act differently because this comment exists?* If no, it is noise — do not write it.

## What is NOT a comment (mechanical bookkeeping — never post these)
- **Lease reclaim** ("Reclaimed stale lease … cleared worker_id"). Recorded in the \`worker_id\` clear + console + cron_log. The next worker gains no forward insight. \`unstuckQuests\` deliberately posts NO comment (see its code).
- **Leaf dispatch** ("Cron:questTick deployed leaf worker X"). Recorded in \`worker_id\` + the tick's \`dispatched[]\` return + console. Not a comment.
- **Bare status** ("gate passed", "stage advanced", "N/N objectives done") with no verdict insight and no next-step. A pass that carries a real verdict ("Cat note: PDF has 9 zh-CN chapters matching the storyboard") IS insight; a bare "gate passed" is not.

## What IS a comment (write these)
- **Stop-summaries** with a real \`Next:\` ("Next: user reviews the PDF; if tone is off, reopen ch3-5 objectives").
- **Blocks / bounces** that name the fix ("Worker MUST call questExecution.submit() before resubmitting").
- **Verdicts** carrying what was verified + how (\`asset_verdict\` with judge_origin).
- **Digestion records** — what an answered question/directive changed in the quest.
- **Lockphrase handoffs** — the script-locked gate phrase IS the next-stage instruction; keep the phrase, but make the surrounding text a real verdict, not filler.

## Don't pile up (edit-in-place)
If a prior comment already states the SAME unresolved block/stop for the SAME reason (a repeat attempt), **EDIT it in place** (PATCH — bump an attempt counter) rather than inserting a near-identical row. Three copies of "confirmSubmission failed — worker must submit()" is two rows of noise. This is the comment-side of \`ruleAcceptableStopState\` (exactly one stop-summary per stop).

Enforced in the leaf prompts (\`buildLeafExecutePrompt\` / \`buildLeafReviewPrompt\` step 5) and in the deterministic substrates (e.g. \`unstuckQuests\` posts nothing). See \`commentDigestion\` + \`ruleAcceptableStopState\`.
`,
    },
    planQuest: {
      description:
        "OWNS the plan-stage itemizing procedure: turn a quest's big description (summarized from the user's ask) into objectives + deliverables + quest_questions, shrinking the description until it is short and non-overlapping with the sub-items, then transition to execute. Run by the planning agent questTick deploys to plan-stage quests; the scripted itemizer is the questmaster WEAPON's planQuest action.",
      howTo: `
You are planning ONE quest at \`stage='plan'\`. A freshly created quest is a big description and nothing else — no objectives, no deliverables. Your one job is **itemizing**: migrate every itemizable element OUT of the description into the typed sub-item tables, until the description holds only what is global.

## The itemizing pass — three destinations

Read the description and classify every element into exactly one of:

1. **Work items → \`objectives\`.** Every discrete action the quest requires becomes one objectives row, ordered by dependency (lowest \`n\` = do first). Author each name per \`ruleObjectiveClarity\`: verb-led, future-focused, ONE action per objective — split bundled independent actions, never carry an origin prefix. Per-objective detail (constraints, file paths, acceptance notes) moves into that objective's \`notes\`, NOT left in the description.
2. **Final artifacts → \`deliverables\`.** Every artifact the quest must PRODUCE becomes one deliverables row with a content-typed \`kind\` — how a human consumes it, not the file format: \`text\` (read it), \`html\` (view a rendered page), \`image\` (view a picture/screenshot/diagram), \`video\` (watch it), \`document\` (flip through a paged file — pdf/ppt). One row per distinct artifact; \`ref\` stays null (the worker fills it). Deliverables track OUTPUT; objectives track WORK — never write an objective that merely restates a deliverable.
3. **Ambiguities → \`quest_questions\`.** Every load-bearing decision the user must make becomes one open question (\`qtype='unblocking'\`, \`direction='to_user'\`), linked to the objectives it blocks via \`unblocks_objective_ids\`. Ask ONLY load-bearing questions — if the answer is discoverable from the description, the repo, or a skill book, discover it instead of asking.

## The done-condition (when planning is finished)

Planning is complete when BOTH hold:
- **Short**: the description retains only GLOBAL instructions — context every objective needs (links, references, cross-cutting constraints, precedence rules). A description that still narrates work items or artifacts is not done.
- **Non-overlapping**: nothing in the description duplicates an objective, deliverable, or question. An element lives in exactly ONE place; the sub-item is the canonical copy, so delete its trace from the description when you migrate it.

Rewrite the description to this concise form as the final mutation of the pass.

## Mechanics

- The scripted itemizer \`questmaster.planQuest\` (weapon action) performs this whole pass in one call — run it FIRST. Then VERIFY its output against the done-condition above and patch surgically (\`patchObjective\`, description update, missing rows) rather than re-running the itemizer over good rows. It is idempotent: a quest that already has objectives skips straight to the transition.
- Gear scan: planQuest stamps each objective's \`notes\` with a "Gear: <book.action, …>" line from the registry scan — planning is where skill-book/weapon research happens ONCE, so execute leaves don't re-run it.
- Transition: when the done-condition holds, \`questmaster.moveQuestToExecute\` — the guard requires >=1 objective, >=1 deliverable, and an assignee, and is the ONLY legal stage write. If the description is too vague to itemize at all, write ONE open question naming what you need and leave the quest in plan (the tick re-plans after the user answers).
- Do NOT execute objectives, produce deliverables, or answer your own questions — planning ends at the transition.
`,
    },
    ruleObjectiveClarity: {
      description:
        "How an objective's NAME is phrased. Same convention as a quest name: future-focused, starts with a verb (or verb+object), one action per objective. No origin prefix, no combined actions. LOCKED 2026-06-12.",
      howTo: `
An objective's \`name\` is a **command to a future worker**, not a record of where it came from. Phrase it like a quest name: a verb-led, concise statement of the one thing to do. Three rules — all three must hold.

## 1. No origin prefix — future-focused only
Describe ONLY what is to be done. Where the objective came from (a user message, a question, a refactor request) is irrelevant to the worker who executes it. Strip any \`Questions:\`, \`User asked:\`, \`Refactor:\`, \`From chat:\` lead-in. The objective communicates the future, not its history.

## 2. Start with a verb — verb, or verb+object, then optional clarification
The first word is a verb so the action is clear at a glance. Shape: \`<verb>\` or \`<verb> <object>\`, then a period, then an OPTIONAL clarifying sentence that defines exactly what "done" means. Make the object specific enough to disambiguate (prefer \`Add answer button.\` over a bare \`Add button.\` when more than one button is in play).

## 3. One action per objective — decompose independent actions
Each objective is exactly ONE action. If a user instruction bundles multiple actions that have **no dependency** on each other, split them into separate objectives. Combining them produces an **unclean close** (you can't mark it \`done\` when only half is finished) or an **unclean stuck** (half blocks, half doesn't, and the row can't honestly be left open-and-blocked or marked \`done\`). Dependent steps of a single deliverable may stay together; independent deliverables must not.

## Bad → good
**Bad** (origin prefix, noun-led, two actions combined):
\`Questions: Answer button for empty unblocking answers + questionDigestion rule (worker digests unblocking, user digests user; transient, removed once insights migrated)\`

**Good** (decomposed, verb-led, future-focused):
- \`Add answer button. Add an answer button to the answer column of unblocking questions that have no answer.\`
- \`Solidify digestion behavior. For unblocking questions that have an answer, the worker migrates the decisions in the answer into objectives and other parts of the quest, then removes the question and answer.\`
- \`Add resolve button. Add a resolve button to user questions so the user can mark them resolved.\`

Apply this when authoring objectives in \`planQuest\`, when a worker \`add\`s an objective mid-execution, and when externalizing an objective shift per CLAUDE.md §5.
`,
    },
    stepBoundaryAnnotations: {
      description:
        "Convention for embedding sub-agent contract boundaries directly in quest objective definitions. Adopted from cc-sdd's `_Boundary:_` / `_Depends:_` annotations to stop sub-agent drift mid-run. Additive — existing objectives without annotations are unaffected.",
      howTo: `
You are writing or refining a quest objective that will be executed by a sub-agent (MA / SA / Cursor cloud agent). Embed the contract directly in the objective's \`method\` field using two literal markers so the sub-agent cannot drift past them and the parent can mechanically verify completion.

## The two annotations

**\`_Boundary:_\`** — what this objective is allowed to touch.
- Files / paths the sub-agent may read AND write.
- Tools / skill-book actions it may call.
- Hard out-of-scope items (it may NOT touch X, even if doing so seems related).
- One line per constraint.

**\`_Depends:_\`** — what this objective requires from prior objectives OR external state.
- Named outputs from earlier objectives (\`objective.n.output_key\`).
- External preconditions (a cloud agent must already exist; a row must already be in table X).
- One line per dependency.

## Example

\`\`\`
objective.method:
  Generate the biomarker-mockup CTA copy variants and write them to
  app/(town)/proving-grounds/biomarker-landing-page-mockup/copy.json.

  _Boundary:_
  - May read: docs/biomarker-mockup-spec.md
  - May write: app/(town)/proving-grounds/biomarker-landing-page-mockup/copy.json
  - May NOT touch: any pim_* table; any Cursor cloud-agent dispatch
  - Allowed tools: Write, Read

  _Depends:_
  - objective.1.output: target_persona (string)
  - objective.2.output: hypoxia_marker_list (array of HPA gene ids)
\`\`\`

## Why this works

1. **Sub-agent cannot drift past the boundary** — its preflight gate scans the boundary block and rejects out-of-scope edits.
2. **Parent can mechanically verify** — \`worker.executeStep\` can diff the objective's actual file touches vs the \`_Boundary:_\` allow-list and refuse to mark the objective complete if they don't match.
3. **Survives compaction** — the convention is in the objective text itself; the PreCompact hook re-injects objective text, so the contract survives long sessions.

## Rules

- These annotations are OPTIONAL but RECOMMENDED for any objective whose deliverable is non-trivial (≥ ~5 minutes of agent work) or that runs in a sub-agent.
- Do NOT mix \`_Boundary:_\` with prose constraints elsewhere in \`method\` — put all boundary constraints inside the block so they are mechanically parseable.
- For \`_Depends:_\`, only name dependencies that BLOCK execution. "Nice to have" context goes in prose.
- If an objective has no boundary or dependency worth naming, omit the block — empty blocks are noise.
`,
    },
    tickWorkerDispatch: {
      description:
        "The local worker-dispatcher tick. GM fires this when the user gives an execution go-ahead on an existing execute-stage quest (e.g. \"looks good, go\"). One tick = scan execute-stage quests assigned to Guildmaster, load the lowest-n pending objective (or resume a running one), single worker, no parallel fan-out.",
      howTo: `
You are running one worker-dispatcher tick. This is NOT a planning cascade — the quest already exists at \`stage='execute'\` with its \`objectives\` rows. The user just gave a go-ahead ("looks good, go", "go", "start it").

## Procedure

1. Call the \`questmaster.tickWorkerDispatch\` weapon (programmatic, no LLM). It:
   - scans quests where \`assignee='Guildmaster'\` AND \`stage='execute'\`, newest-first;
   - on the first quest with work, resumes its already-\`running\` objective OR loads its lowest-\`n\` \`pending\` objective, flipping that objective \`pending → running\` in \`quests.objectives\`;
   - returns \`{ ok:true, questId, url, title, objective:{n,name}, resumed? }\` or \`{ ok:false, reason }\`.
   Default is ONE next quest, ONE worker. Never fan out in parallel unless the user explicitly asked for it.
2. The flipped \`running\` objective now carries a clear \`input\`/\`method\`/\`output\` — so execution renders event-bus-stably. The worker (\`worker.executeStep\`) is what actually does the objective's concrete job; the dispatcher only loads + flips status, it does not do the work itself.
3. Compose the startup confirmation via \`worker.reportStartup\`: name the objective now executing (n + name) and give the live quest page URL to monitor it. Do NOT open a new planning cascade; do NOT re-summarize the request.

## Rules

- This action is the bridge between the planning cascade (Steps 1–4b) and execution. It is dispatched by GM ONLY for an execution go-ahead / refinement on an already-created quest, never for a new planning request.
- Do not write \`quests.stage\` here — the quest is already \`execute\`; the dispatcher only mutates the loaded objective's \`status\`.
- A \`{ ok:false }\` (no Guildmaster execute-stage quest with a pending/running objective) means there is nothing to dispatch — report that plainly, do not invent work.
- One decision, one objective, one return: load/resume exactly one objective per tick and report.
`,
    },
    questUpdate: {
      description:
        "Apply a user refinement / escalation-answer to an EXISTING in-flight quest. Comment-first-then-render: summarize the request as a quest comment BEFORE any mutation, then a dedicated update agent classifies the target attribute and rewrites it. Used by GM Step 1.5's refinement branch — never for a new request.",
      howTo: `
You are applying a user's mid-flight change to a quest that already exists (planning is done; the quest is at \`stage='execute'\`). This is NOT a planning cascade and NOT the dispatcher — it is a surgical, audited mutation.

## Comment-first-then-render (load-bearing ordering)

1. **FIRST, write the user request as a quest comment — before ANY change is rendered.** Summarize the user's instruction verbatim-in-substance and insert it into \`quest_comments\` (\`quest_id\`, \`source='user'\`, \`action='update_request'\`, \`summary=<summary>\`, \`resolved=false\`). This is the audit record; it MUST land before the mutation so the change is always traceable to its request. (Weapon: the thin \`questmaster.questUpdate\` DB helper writes the comment + returns its id and the quest's current objectives.)
2. **THEN dispatch a dedicated update agent** that:
   2.1. **Classifies which quest attribute the request targets.** Usually \`objectives\` (an objective's \`input\`/\`method\`/\`output\`); could be \`description\`, \`assignee\`, or \`deliverables\`. Pick exactly one; if genuinely ambiguous which quest, the GM resolves it inline from context (do not ask if context makes it unambiguous).
   2.2. **Renders the change on the targeted attribute itself — never as a workaround.** An objective-behavior change rewrites that objective's \`method\` (the \`objectives\` attribute), NOT the objective's \`notes\` and NOT a new comment. \`notes\` is for execution artifacts, not the instruction.
   2.3. **Scopes the edit to exactly what was asked.** Do not preempt unrelated open decisions. E.g. refining a match rule must leave an existing \`[co-develop-with-user]\` anchor (and any other unrelated method clause) intact.
   2.4. **Flags any new user-emphasized non-negotiable as an inline \`[IMPORTANT DETAIL]\`** in the rewritten \`method\` (same convention as \`summarizeRelevantContext\`: literal marker + verbatim constraint, in \`method\`, never \`notes\`), so the refinement is unambiguous downstream.
3. **Resolve the request comment.** Once the mutation lands, mark the step-1 user-request comment \`resolved=true\` (via \`worker.resolveComment\`). The audit pair (request → applied) is now closed.
4. **Report the mutation** via \`worker.reportMutation\`: quote the exact change applied; surface NO follow-up question when the instruction was complete.

## Rules

- Comment STRICTLY before mutation. A rendered change with no preceding \`update_request\` comment is an un-audited mutation — a discipline failure.
- One attribute, one scoped change per request. Never widen scope, never resolve an unrelated open anchor/decision as a side effect.
- Never write \`quests.stage\` here. Never soften or drop an existing \`[IMPORTANT DETAIL]\` / \`[co-develop-with-user]\` marker that the request didn't touch.
- If the request is actually a new objective (not a change to this quest), it is NOT a questUpdate — it belongs in the planning cascade. questUpdate is for refinements/answers to an unambiguous existing quest only.
`,
    },
    runReviewApproval: {
      description:
        "Approve a GuildOS2 quest waiting at stage='review' on the operator's behalf ('looks good, approve it'). Review's exit is an ATTESTATION, not a derived fact: it is a USER-CHANNEL write of stage='reflect' — the same op the GM Room's 'Approve → reflect' button sends. The worker channel refuses it (no_worker_stage_assertion), so POST /api/quest/update can NEVER approve; the machine route that can is POST /api/quest/test with action='mutate', channel='user'.",
      howTo: `The operator approved a quest sitting at \`stage='review'\` (purrview put it there — every deliverable \`review_approval_status='approved'\`). Review is the one stage the fact cascade cannot leave on its own: \`deriveStage\` has no rule that advances out of \`review\`, so the exit is an explicit attestation.

## The op — one line, and it is the product's own approve button

\`\`\`json
{ "table": "quest", "mode": "update", "fields": { "stage": "reflect" } }
\`\`\`

That is verbatim what \`approveReviewAction\` in \`GuildOS2/app/(app)/gm-room/actions.js\` sends. \`reflect\` is sticky (cascade rule 5) until a reflect-stage comment lands, which derives \`complete\` (rule 4).

## Which channel — this is the whole trap

\`no_worker_stage_assertion\` is HARD but \`channels: WORKER_ONLY\`. On the **worker** channel it REFUSES the write; on the **user** channel it runs as advisory and the attestation lands. So:

- ❌ \`POST /api/quest/update\` — hard-codes \`channel:"worker"\`. Returns 400 \`QUEST_UPDATE_AUDIT_FAILED\` on any stage op. This is by design, not a bug; do not try to work around it with facts, because no fact derives \`reflect\`.
- ❌ \`POST /api/quest/inject-verdict\` — covers \`review_plan\` and \`purrview\` verdicts only.
- ❌ a Playwright/leafBrowser login as \`GUILDOS2_VERIFIER_*\` — RLS is owner-scoped and the verifier owns nothing; the quest reads back as \`[]\`. Never mint a session for the operator's own account (it evicts their browser — see browsercontrol.leafBrowserControl).
- ✅ **\`POST /api/quest/test\`, \`action:"mutate"\`, \`channel:"user"\`** — the machine-key-gated door to \`quests.applyMutations\`, the same user-channel write the GM Room button takes. Owner-scoped by the API key. The route's own comment says it: applyMutations has no HTTP surface of its own, so this harness action is the only way to reach that path over the wire.
- ✅ the GM Room at \`/gm-room\` in the operator's own signed-in browser (CIC), review card → **Approve → reflect**. Available when this runtime has a working adapter to the required signed-in browser; verify the actual capability instead of excluding headless or non-Claude workers by name.

\`\`\`bash
curl -X POST "$GUILDOS2_URL/api/quest/test" -H "Authorization: Bearer $GUILDOS2_API_KEY" \\
  -H 'content-type: application/json' -d '{"action":"mutate","payload":{
  "questId":"<uuid>","channel":"user",
  "ops":[{"table":"quest","mode":"update","fields":{"stage":"reflect"}}]}}'
\`\`\`

## Arc

1. **Earn the gate.** Read the quest (\`GET /api/quest?id=<uuid>\`) and confirm it really is at \`review\` with every deliverable \`review_approval_status='approved'\`, no open question, no unresolved comment. An unresolved comment or an unanswered question outranks everything in the cascade and would bounce the quest to \`plan\`/\`escalated\` the moment you write.
2. **Real final read.** Re-verify the actual artifact, not a rubber-stamp. The operator said approve; the gate still earns it.
3. **Write the attestation** through the user channel above.
4. **Verify it stuck** — read the quest back and confirm \`stage === "reflect"\`. The executor self-heals the stage from the facts, so a write that "succeeded" can still reconcile elsewhere; only the read-back proves it.
5. Report the resulting stage.

## LEGACY — do not use the \`questReview\` weapon on a GuildOS2 quest

\`libs/weapon/questReview\` belongs to the ORIGINAL GuildOS: it talks to the council database, the retired steps/items model, a \`FINAL_GATE_PASS\` lockphrase and an \`enforce_stage_gate\` DB trigger, and it advances \`review → closing\`. GuildOS2 has no \`closing\` stage (its ladder is … purrview → review → reflect → complete), no items table and no such trigger. \`confirmApproval\` / \`pass\` / \`confirmFinalGate\` will not approve a GuildOS2 quest. Same for \`scanForUncommittedMethods\`, which is written against \`stage='closing'\`.
`,
    },
    scanForUncommittedMethods: {
      description:
        "At stage='closing', walk every objective's notes for in-execution method developments not yet captured in a skill book; write a closing-stage quest comment proposing the SMART plan AND any such method be saved to a skill book for reuse, naming the source objective.",
      howTo: `Runs once a quest reaches \`stage='closing'\` (after \`questReview.pass\`). The point: methods invented DURING execution (e.g. an SSH/SQL retrieval approach co-developed at a \`[co-develop-with-user]\` anchor) are valuable reusable substrate that would otherwise be lost when the quest closes.

## Procedure

1. **Walk every \`objectives\` row's \`notes\`** for this quest. \`notes\` is the execution-artifact log; a "method resolved per user comment …" / "retrieval method: …" style note signals an in-execution method development. (Recall: \`resolveCoDevAnchor\` records the decided method's *fact + source comment* in \`notes\`; the instruction itself is in \`method\`.)
2. **Detect uncommitted methods** — a developed method counts if it is reusable beyond this one quest AND is not already encoded in a skill book. The canonical example here: the SSH/SQL Magento-2 PicoBand antibody-retrieval method developed at objective 2's anchor (recorded in objective 2's notes, instruction in objective 2's method).
3. **Write ONE closing-stage quest comment** (\`quest_comments\`, \`source='guildmaster'\` or system, \`action='closing'\`, \`resolved=false\`) that proposes BOTH:
   - the quest's overall **SMART plan** be saved as a reusable skill book, and
   - the **detected in-execution method(s)** be saved alongside it,
   naming the **source objective** for each method (e.g. "the SSH/SQL antibody-retrieval method developed in objective 2"). Flag the abstraction-shape choices the user must confirm (target skill book name; the parameter to vary across runs) — these are \`[IMPORTANT DETAIL]\` for the user to confirm before anything is forged.
4. Do NOT forge anything here. This action only PROPOSES (writes the comment). The actual forge happens in the post-success finale only on the user's explicit yes (\`blacksmith.forge\` / \`blacksmith.patchSkillBook\`), after which the matching closing comment is marked \`resolved=true\`.

## Rules

- One consolidated closing comment, not one per method — the user confirms the abstraction shape once.
- Name the source objective for every proposed method; an unattributed "we made something reusable" is not actionable.
- Never auto-forge. Skill-book naming + abstraction shape are user-owned substrate decisions (sanctioned finale halt, not a manufactured stop).
- If NO uncommitted reusable method exists and the plan itself isn't worth abstracting, write no comment — do not manufacture a proposal.
`,
    },
    closeQuest: {
      description:
        "Terminal closure arc. The user said yes to the closing finale proposal — forge the skill book (blacksmith.forgeWeapon), resolve the finale comment once the forge lands, then questmaster.closeQuest weapon (gated closing→complete), then worker.reportClosure.",
      howTo: `The quest is at \`stage='closing'\` with an unresolved \`action='closing'\` finale comment (the save-to-skill-book proposal from \`scanForUncommittedMethods\`). The user just approved it with the abstraction shape. Run the closure arc.

## Arc

1. **Forge** — dispatch \`blacksmith.forgeWeapon\` with the user-confirmed abstraction shape: the target skill book (new vs patch; name), what the actions are (the quest's SMART plan parameterized over the user-named parameter — e.g. \`pathwayURL\`), and the in-execution method(s) the finale named (e.g. the SSH/SQL antibody-retrieval method from objective 2) included as action(s). Forge produces the skill book file (e.g. \`libs/skill_book/PathwayEnrichment/index.js\`) and MUST verify it loads via the skill book registry without error (per blacksmith.refForgeDiscipline). If the user asked to patch an existing book instead, use \`blacksmith.patchSkillBook\`.
2. **Resolve the finale comment** — once the forge lands AND the registry-load verification passes, mark the matching \`action='closing'\` finale comment \`resolved=true\` (via \`resolveQuestComment\`). Not before — an unresolved finale comment HARD-BLOCKS closeQuest by design.
3. **closeQuest weapon** — call the \`questmaster.closeQuest\` weapon. It re-checks (stage=closing, no unresolved \`action='closing'\` comment), archives a managerial-summary comment, writes \`stage='closing' → 'complete'\`, and SELECT-back-verifies. Never write \`quests.stage='complete'\` directly — the gate (resolved finale) is the whole point.
4. **reportClosure** — \`worker.reportClosure\`: confirm the archive by file path, name the new/patched skill book + its parameter, and confirm the quest moved to \`complete\`.

## Rules

- Forge BEFORE resolving the finale comment BEFORE closeQuest. Resolving the comment with nothing forged is a false "done"; closing with the comment unresolved is refused by the weapon.
- The skill book must be verified loading via the registry — a forged file that doesn't load is not a completed forge.
- closing→complete is weapon-gated and terminal; do not emulate it with a direct stage write.
- Honor the user's abstraction shape exactly (target book, parameter, one-book-vs-two) — these were the \`[IMPORTANT DETAIL]\` choices they confirmed.
`,
    },
    auditLeafLogs: {
      description: "Parse .worker-logs/*.log and emit per-leaf tool-call audit: ordered sequences, waste classification (getQuestDetails miss, import probing, --list discovery, skill-book source reads), and summary counts. Input: { logDir?, filterPrefix? }. No LLM.",
      input: { logDir: "string, default '.worker-logs'", filterPrefix: "string, default 'leaf-'" },
      output: { ok: "boolean", leafCount: "number", leaves: "array of per-leaf audit objects" },
    },
    checkpointQuestAsset: {
      description: "One-call quest-asset checkpoint honoring assetInlineVsFileStorage: text < 1000 chars stores INLINE (ref = the text, no upload, filename optional, kind defaults 'text'); larger/binary content uploads to GuildOS_Bucket (kind defaults 'document') then inserts the quest_assets row. Replaces the two-step hand-rolled upload+REST-insert for ruleCheckpointAssets.",
      input: { questId: "string", label: "string", content: "Buffer|string", filename: "string (required only for the file path; omit for inline text)", kind: "string in {text,html,image,video,document}, default 'text' inline / 'document' file", contentType: "string, default 'application/octet-stream'", source: "string, default 'worker'", actorName: "string|null" },
      output: { ok: "boolean", inline: "boolean (true when stored in-row)", publicUrl: "string (file path only)", assetId: "string", storagePath: "string (file path only)" },
    },
    readLifecycleContract: {
      description: "Canonical quest lifecycle — event-bus stages, cron dispatch, worker leasing, objectives model. Read this first when creating a quest, transitioning a stage, or interpreting a stage-related comment. Authoritative spec: docs/quest-lifecycle.md.",
      howTo: `
## Quest Lifecycle (CANONICAL — event-bus model, 2026-06-11+)

**The big mental model.** A quest is a durable thread on an event bus. A **cron (\`questTick\`)** scans quests by stage every ~10 min and dispatches a worker agent (leaf) to push it forward. The worker leases the quest, does one burst, summarizes, and releases. Two stoppable terminals only: **complete** (verified) or **escalated** (blocked on a user decision). The substrate guarantees a quest can **never sit in \`execute\` forever** — every burst either advances it, escalates it, or is reclaimed by \`unstuckQuest\`.

### Stages (in order)

\`plan → execute → purrview → review → closing → complete\`  (+ \`escalated\` as a blocked-state branch from \`execute\`)

| Stage | Owner | What happens |
|-------|-------|--------------|
| \`plan\` | Cron → planner leaf | Created with deliverables + verbose description; the planner distils it into objectives + questions. When objectives + deliverables + assignee present → \`moveQuestToExecute\`. |
| \`execute\` | Cron → worker leaf | The heavy-lifting loop. Workers lease via \`worker_id\` (compare-and-set), produce + verify deliverables, update objective Progress/Δ/Status. On completion → \`moveQuestToReview\`; on full block → \`moveQuestToEscalated\`. |
| \`purrview\` | review path | Automated/agentic review of the worker's submission. \`runReviewLoop\` (image-only) or \`reviewSubmission\` (mixed). |
| \`review\` | review path | Quality gate — script-check then agentic check. Pass → \`closing\`. Fail → back to \`execute\` with a helpful comment. |
| \`closing\` | Questmaster | Archive summary. |
| \`complete\` | terminal | Create a new quest rather than reopening. |
| \`escalated\` | user (blocked) | Every workable objective is blocked on an open user question. User answers bounce it back to \`execute\` on the next tick. |

### Key transitions

| From → To | Mechanism |
|-----------|-----------|
| \`plan → execute\` | \`QM:moveQuestToExecute\` (guard: ≥1 objective, ≥1 deliverable, assignee) |
| \`execute → review\` | \`QM:moveQuestToReview\` (script-gate: all objectives done + deliverables filled; then quality-gate) |
| \`execute → escalated\` | \`QM:moveQuestToEscalated\` (all workable objectives blocked on user questions) |
| \`escalated → execute\` | User answers gating questions; next tick bounces back |
| \`review → closing → complete\` | Review approval path |
| \`review → execute\` | Review bounce with helpful comment |

### Entity model (current)

- \`quests\`: stage, assignee (TEXT), worker_id, worker_claimed_at, description, title
- \`objectives\`: n, name, status, progress, delta, notes — the materialised plan
- \`deliverables\`: name, kind, ref, status, verify_screenshot_ref
- \`quest_comments\`: actor_name, source, action, summary, resolved, position
- \`quest_questions\`: qtype (user/unblocking), body, answer, answered
- \`quest_assets\`: label, ref, kind, source, actor_name

**RETIRED (do not use):** items/inventory model, action_steps, design/html_markup stages, the 5-tier review columns (self_check/openai_check/purrview_check/claude_check/user_feedback), expectation on deliverables, design_approved, assigned_to, assignee_id.

### Worker execution contract

1. Lease via \`toggleQuestActiveExecution\` (compare-and-set \`worker_id\`).
2. Handle open user questions + unresolved comments BEFORE touching objectives.
3. Work lowest-n active objective with no blocking question.
4. Save forward: checkpoint reusable artifacts as \`quest_assets\`.
5. On stop: write ONE structured closing comment (Done/Next/Blocked/Learned), then release lease.

### Locked rules

1. **\`description\` is strategic context, not status.** Set at creation, edited only to refine the OBJECTIVE itself. NEVER overwritten with status text.
2. **User questions are the user's to remove.** Workers answer them (write into the answer field) but never delete or resolve them.
3. **No scratch scripts.** Always use skill books + weapons; forge via blacksmith if missing.
4. **Retry-exhaustion at 4.** If last 4 comments show the same failure signature with no new approach → escalate or spawn a fresh-context sub-agent for a different hypothesis.

### Operational entry points

- **Creating a quest:** \`questmaster.createQuest\` (title + description at stage=plan; planner itemizes objectives).
- **Planning:** \`questmaster.planQuest\` (distil description → objectives + questions).
- **Executing:** follow \`docs/quest-lifecycle.md\` §2 \`executeQuest\` procedure.
- **Reviewing:** \`questmaster.runReviewLoop\` (image-only) or \`questmaster.reviewSubmission\` (mixed).
- **Escalating:** \`questmaster.moveQuestToEscalated\`.
- **Full spec:** \`docs/quest-lifecycle.md\`.
`,
    },

    planRequestToQuest: {
      description: "Translate a raw user request into a structured quest (title, description, deliverables, due date, stage).",
      input: {
        initialRequest: "string, the user's raw request text",
      },
      output: {
        title: "string",
        description: "string",
        deliverables: "string",
        due_date: "string, ISO 8601",
        stage: "string, one of: idea, plan",
      },
    },
    searchAdventurerForQuest: {
      description: "Pick the best-fit adventurer for a quest by capability match.",
      input: {
        quest: "object with title, description, deliverables",
        adventurers: "array of { id, name, capabilities }",
      },
      output: {
        adventurer_id: "string, UUID",
        name: "string",
      },
    },
    selectAdventurer: {
      description: "Match a new request to a roster adventurer (returns the id or no-match).",
      input: {
        quest: "object with id, title, description",
      },
      output: {
        result: "object { id, name } or false",
        msg: "string, rationale",
      },
    },
    assign: {
      description: "Assign stage: load quest, scan adventurer roster, pick best-fit or return no-match.",
      input: {
        questId: "string, quest UUID",
      },
      output: {
        result: "object { id, name } or false",
        msg: "string, rationale",
        meta: "object, debug trace",
      },
    },
    interpretIdea: {
      description: "Turn a raw user request into a structured quest for a chosen adventurer.",
      input: {
        initialRequest: "string, the user's raw request text",
        adventurerName: "string, name of the chosen adventurer",
      },
      output: {
        title: "string",
        description: "string",
        deliverables: "string",
        due_date: "string, ISO 8601 or null",
      },
    },
    runReviewLoop: {
      description: "One-call locked review: confirmSubmission → openai_images.judge per deliverable → approve OR bounce. Use this for any quest whose deliverables are all images. Falls back to manual_required for non-image deliverables.",
      howTo: `
**Use this whenever every deliverables.ref is an image** (.png/.jpg/.jpeg/.webp/.gif). One call replaces the entire reviewSubmission workflow — Cat does not write its own loop, choose its own judge, or assemble its own perItemVerdicts.

\`\`\`javascript
import { runReviewLoop } from "@/libs/skill_book/questmaster";
const result = await runReviewLoop({ questId });
\`\`\`

**Result shape:**
- \`{ ok: true, action: 'approved', stage: 'review', items_count, perItemVerdicts }\` — every deliverable passed gpt-4o judge; quest advanced.
- \`{ ok: true, action: 'bounced', stage: 'execute', failing_item_keys, perItemVerdicts }\` — at least one deliverable failed; quest sent back; bounce reason auto-built from judge rationales.
- \`{ ok: false, action: 'manual_required', verdicts }\` — at least one deliverables.ref is non-image; defer to the reviewSubmission howTo and judge those manually (text/JSON/doc judging is not yet automated in this loop).
- \`{ ok: false, action: 'confirm_failed' | 'approve_failed' | 'bounce_failed', failed_at, reason }\` — a gate weapon refused; the report tells you exactly what's wrong.

**Why this exists:** Cat's prior sessions (pre-2026-04-28) kept inventing their own loops, posting verdicts without invoking the vision judge (judge_origin null), and drifting into infra PR work when the queue went quiet. Removing the choice is the highest-leverage fix.

**Banned**: do NOT call confirmSubmission/approve/bounce/judge directly when runReviewLoop covers your case. Save those for the manual reviewSubmission flow when deliverables are non-image or you need a tiebreaker.
`,
    },
    reviewSubmission: {
      description: "Manual purrview review for non-image artifacts (text, JSON, doc). For image-only quests, use runReviewLoop instead. Script-locked end-to-end via questPurrview weapon: confirmSubmission → per-deliverable evaluation → approve OR bounce.",
      howTo: `
**Prefer \`runReviewLoop\` when every deliverables.ref is an image** — it removes the workflow choice. Use this manual flow only when:
  - At least one deliverables.ref is non-image (text/JSON/doc/HTML), OR
  - You need a tiebreaker via \`getSecondOpinion\` before deciding, OR
  - runReviewLoop returned a gate failure you want to debug step-by-step.

This is the canonical purrview review flow. Both ends of every handoff are script-locked — never write \`stage\` directly, never open a screenshot before \`confirmSubmission\` passes.

### Step 1 — Verify the worker handoff (no shortcuts)

\`\`\`javascript
import { confirmSubmission } from "@/libs/weapon/questPurrview";
const confirm = await confirmSubmission({ questId });
if (!confirm.ok) {
  // confirm.reason: 'no_gate_comment' | 'lockphrase_missing' | 'wrong_stage' | ...
  // The quest didn't pass through questExecution.submit. Do NOT review it.
  // Either bounce it back (if a worker is owning it) or escalate to Guildmaster.
  return;
}
// confirm.deliverables = the produced assets (name, kind, ref); judge each one against its objective's output[].verification.
\`\`\`

### Step 2 — Invoke the vision judge per deliverable (mandatory)

**The judge is mandatory, not optional.** Layer-attribution finding (5f9b4c0): Cat alone (Composer 2.0 vision) has 0/2 catch rate on visual verification. The \`questPurrview.approve\` and \`bounce\` gates now refuse any verdict missing \`judge_origin\`. The judge's verdict binds — Cat's \`text\` is the wrapper note, not the decision.

For each \`item_key\` (asset name) returned by confirmSubmission:
- **Image URL** → call \`openai_images.judge({ imageUrl, claim, baselineUrl })\`. \`claim\` = the deliverable's \`expectation\` / objective \`output[].verification\` verbatim. \`baselineUrl\` = the deliverable's \`design_url\` (the AI-approved baseline from the design stage) — pass it whenever the deliverable went through the new design pipeline. With both, the judge requires (a) actual satisfies claim AND (b) actual is ≥90% similar to baseline OR contains all key elements. Default model is \`gpt-4o\` (full); never override to \`gpt-4o-mini\` (banned, see CLAUDE.md WWCD).
- **Text / JSON URL** → fetch the body and judge it yourself via Claude Sonnet (claudeCLI). Use \`judge_origin: 'claude-multimodal-read'\` for these.
- **Document / non-image URL** → same as text.

\`\`\`javascript
import { judge } from "@/libs/weapon/openai_images";

const perItemVerdicts = [];
for (const deliverable of confirm.deliverables) {
  const isImage = /\\.(png|jpg|jpeg|gif|webp)/i.test(deliverable.ref);
  if (isImage) {
    const j = await judge({ imageUrl: deliverable.ref, claim: deliverable.expectation });
    // j.verdict ∈ {'match','mismatch','inconclusive'}; map conservatively.
    const verdict = j.verdict === "match" ? "pass" : "fail";
    perItemVerdicts.push({
      item_key: deliverable.name,
      verdict,
      text: verdict === "pass"
        ? \`Judge confirmed: \${j.reasoning}\`
        : \`Judge flagged \${j.verdict} (confidence \${j.confidence}): \${j.reasoning}\`,
      judge_origin: \`openai_images.judge:\${j.model}\`,
      judge_rationale: j.reasoning,
      judge_model: j.model,
    });
  } else {
    // Text/JSON/doc — fetch + Claude-judge, then build the verdict with
    // judge_origin: 'claude-multimodal-read'.
  }
}
\`\`\`

### Step 3a — Approve (every judge says match)

\`\`\`javascript
import { approve } from "@/libs/weapon/questPurrview";
const result = await approve({ questId, perItemVerdicts, summary: "Optional Cat-level note" });
if (!result.ok) {
  // Common rejections:
  //   failed: ['judge_origin_missing_or_invalid'] → you forgot to invoke the judge
  //                                                  for one or more deliverables.
  //   failed: ['verdict_not_all_pass']            → at least one judge said
  //                                                  mismatch/inconclusive. Use bounce.
  // result.report.fix tells you exactly what to do.
}
\`\`\`

On success: stage moves to \`review\`, per-deliverable Cat verdict comments are written (text includes judge rationale), \`deliverables.purrview_check\` (T2 column) is filled per deliverable via the writeReview chokepoint, and the APPROVE lockphrase ('this quest now meets the criteria for review') lands as a quest_comments row. The GM-desk script greps for that phrase before showing the quest to the user.

### Step 3b — Bounce (≥1 judge said mismatch / inconclusive)

\`\`\`javascript
import { bounce } from "@/libs/weapon/questPurrview";
const result = await bounce({
  questId,
  perItemVerdicts, // same shape as approve — judge_origin still required even on bounce
  reason: "<one paragraph: which item_keys failed, what the judge flagged, what the worker should fix>",
});
\`\`\`

On success: stage returns to \`execute\`, per-deliverable Cat verdict comments are written, \`deliverables.purrview_check\` is filled per deliverable, and the BOUNCE lockphrase lands as a quest_comments row. The worker resubmits using \`writeItem\` with the SAME item_keys (UPSERT replaces in place) — same deliverables.length, same gate.

### What NOT to do

- Do NOT \`updateQuest({stage: 'review'})\` directly. There's no other path that writes the APPROVE lockphrase, and the GM desk will refuse to surface the quest.
- Do NOT skip \`confirmSubmission\`. If a quest reached purrview without passing through the worker gate, it's untrusted — bounce or escalate.
- Do NOT call approve with a 'fail' verdict to "let it through with a note." The gate refuses; bounce is the only valid path for any failure.
- Do NOT post a verdict without a \`judge_origin\`. Cat's standalone visual judgment has 0/2 catch rate — the gate will reject and tell you which item_keys are missing the judge call.
- Do NOT make up a \`judge_origin\` value. The gate accepts only \`openai_images.judge[:model]\` and \`claude-multimodal-read\`. Inventing labels like \`"cat"\` or \`"manual-review"\` will fail validation.

### Common content-quality bounces (history)

- Stock images uploaded with descriptions claiming they're real artifacts (Caribbean beach for sales chart, Swiss-Made badge for folder listing).
- Empty inventory passed via direct stage write (now blocked by the submit gate, but legacy quests may still appear).
- Wrong repo/workspace artifacts (quest is for bosterbio.com2026, screenshots are from GuildOS dev server).
- Dev overlays / 4xx pages captured as if they were the success state.
`,
    },
    reviewCloudAgentWork: {
      description: "Legacy: high-level review framing for cloud-agent deliverables. For new work, use reviewSubmission — the script-locked flow.",
      howTo: `
**Use reviewSubmission for any quest in \`purrview\`.** This action is kept as a meta-framing reference for cases where a cloud agent's output isn't yet on a quest (one-off investigations, exploratory dispatches), but the moment artifacts exist as deliverables on a quest, the script-locked flow is the only acceptable path.

**Common cloud agent pitfalls (still useful as a reference list — feed into reviewSubmission's per-deliverable verdict text):**
- Google/Bing CAPTCHA on headless cloud IPs → use DuckDuckGo or add \`--disable-blink-features=AutomationControlled\`
- Full-page screenshots that are blank/black → use viewport capture instead
- Secrets appearing in committed code → agent's repo has secret scanner
- \`refer to CLAUDE.md for port assignment\` unreachable → agent forgot to start dev server
- Stock image substitution masquerading as a real artifact (the dominant failure mode in the 2026-04-25 audit — 8 of 14 quests had bogus inventory)
`,
    },
    reportChaperonWork: {
      description: "Report chaperon work by creating a review-stage quest on the Guildmaster.s Desk.",
      howTo: `
Every completed chaperon engagement must produce a review task visible on \`/guildmaster-room/desk\`. If no GuildOS quest exists for the work, create one in \`review\` stage — the desk auto-shows all review-stage quests.

\`\`\`javascript
import { writeQuest, writeDeliverable, recordQuestComment } from "@/libs/quest";

// 1. Create the review quest
const { data: quest } = await writeQuest({
  userId,
  title: "Review: <what was done>",
  description: "<summary of work and success criteria>",
  stage: "review",
});

// 2. Upsert each deliverable into the deliverables table (UNIQUE(quest_id, item_key) enforces replace-not-pile-on)
await writeDeliverable({
  questId: quest.id,
  item_key: "screenshot_1",
  url: "https://...",
  description: "Login page after fix",
  source: "chaperon",
});

// 3. Post one hand-off comment at the quest level
await recordQuestComment(quest.id, {
  source: "chaperon",
  action: "deliver",
  summary: "Agent completed: built login page, tested with Playwright, 3 screenshots attached.",
  detail: { agentId: "bc-xxx", artifacts: ["screenshot_1", "screenshot_2"] },
});
\`\`\`

**Per-deliverable review notes (Cat annotating a specific screenshot):** use \`writeItemComment(itemId, { role, text })\` — quest-level comments (\`recordQuestComment\`) are a separate channel for hand-offs and major events.
`,
    },
    handleFeedback: {
      description: "Act on feedback from a quest comment or direct message.",
      howTo: `
When you receive feedback on a quest (comment ping or direct message): act on it immediately. Do not ask for confirmation or permission to implement the feedback. The feedback IS the instruction. Just do it, verify the result, and report back.
`,
    },
    assistAdventurer: {
      description: "Assist an adventurer asking for help; scan the full skill-book registry, not just their loaded set.",
      howTo: `
An adventurer's \`skill_books\` array is the common-use load, not a cap on capability. When helping, think comprehensively across the full registry at \`libs/skill_book/index.js\`.

**Process:**
1. Understand the block — read the quest description, the adventurer's latest comment, and what they tried.
2. Scan the skill book registry. Look for any book whose scope matches the problem, regardless of whether it's in the adventurer's assigned array.
3. If you find a fitting action in a book the adventurer doesn't carry: instruct them to load its \`toc\` (and the specific action's \`howTo\`) temporarily and retry. Do NOT modify their DB \`skill_books\` array for a one-off use.
4. If the adventurer will need that book repeatedly, recommend to the Guildmaster that the adventurer be recommissioned with the book added permanently.
5. Only after this comprehensive scan comes up empty should the adventurer escalate for human help.

**Anti-pattern:** telling an adventurer "you don't have that skill book, escalate" without first checking the registry yourself. The registry is the upper bound on what's available to the guild, not the adventurer's loaded set.
`,
    },

    // ── Merged from questmaster_registry on 2026-04-27 ─────────────────────
    // The old "questmaster_registry" book has been folded into this book.
    approveOrEscalate: {
      description: "Handle requests from worker agents seeking approval or help.",
      howTo: `**When another agent contacts you for approval or help:**

1. **First response:** Do NOT read their full report yet. Ask them:
   "Do you have everything you need to proceed? If so, proceed. If not, tell me specifically what you need from me."

2. **On their second response:** Judge whether you can provide what they need:
   - If YES: help them directly (provide info, credentials, guidance, whatever they asked for).
   - If NO: tell them to escalate the quest (move to 'escalated' stage with a structured comment — reason + unblock_path) and work on the next-priority quest if they have one.

**Key principle:** Don't be a bottleneck. Most agents are asking for permission they don't need. The first question filters those out.`,
    },

    getSecondOpinion: {
      description: "Launch Claude CLI to independently evaluate a submission when you're unsure.",
      howTo: `**When to use:** Complex deliverables where you're unsure about quality, correctness, or completeness. Use sparingly — your own per-deliverable vision verdicts are the canonical signal; this is a tiebreaker.

**How:**
\`\`\`bash
claude -p "Review this submission for quest '<quest-title>'. The deliverable should show: <expectation>. The screenshot is at: <url>. Is this acceptable? What could be improved?"
\`\`\`

Use Claude CLI's response to confirm your assessment, identify issues you missed, or provide more detailed feedback in the bounce comment. Do NOT let it override a clear pass/fail you already established — the per-deliverable verdicts you write to \`purrview_check\` are yours, not delegated.`,
    },

    createPR: {
      description: "Create a pull request for the worker agent's branch after review approval.",
      howTo: `**When:** After reviewSubmission moves the quest to \`review\` (all per-deliverable verdicts pass).

1. Identify the worker agent's branch from the quest comments or conversation.
2. Create a PR on GitHub targeting main (or the project's default branch).
3. Add the quest title and a short summary as the PR description.
4. Record the PR URL in a quest comment so the Guildmaster's desk can link to it.

Worker agents do NOT create PRs — only you do, after approval.`,
    },

    closeQuest: {
      description: "Archive quest deliverables and summary to Asana, then mark complete.",
      howTo: `**Closing flow:**
1. Read the quest description, deliverables, and key comments.
2. Write a managerial-level summary suitable for Asana (not technical detail).
3. Check if the quest references an Asana task (in description or comments).
4. If yes: update the Asana task with the summary using the asana weapon.
5. If no: escalate to the Guildmaster to identify the right Asana task.
6. After successful Asana archival: move quest to \`complete\` stage.
7. Add a comment: "Quest closed. Summary archived to Asana."`,
    },

    // ── New GM cascade (SE 2026-05-13) ──────────────────────────────────────
    // The six actions below are the questmaster-centric replacement for the
    // legacy AO + SMART-scan + MA-SMART-advisor cascade. See
    // docs/smoke-tests/SE-rubric-hypoxia-test.md for the test that exercises
    // them end-to-end. Old cascade kept dormant per decision D7.

    summarizeRelevantContext: {
      description:
        "SA-dispatched action. Reads the user's most recent message + prior conversation, extracts entities (URLs, page names, objective hints, external inputs), and produces a structured context bundle the rest of the cascade consumes. Output shape matches the new Description / Objectives / Deliverables model.",
      howTo: `
You are summarizeRelevantContext. Read the inputs and produce a JSON object that downstream cascade actions will consume.

## Inputs (provided in the dispatch prompt)
- userMessage: the current user message (string)
- conversationHistory: array of {role, content} entries from the prior turns of this planning conversation (may be empty for the first turn)

## Procedure
1. Concatenate conversationHistory + the current userMessage into a working context.
2. Identify:
   - Any URL the user references (e.g. the page they want changed) — these become entries in \`mentions\` and may become \`deliverables\` if they are concrete input assets (a source image, a source dataset, a template page to clone).
   - Any explicit numbered objectives the user lists, OR objectives you can infer with high confidence from a workflow they described.
   - **Partial-spec disambiguation (deterministic — resolves the Tier-1↔Tier-2 split).** If the user has stated one or more CONCRETE, buildable components / deliverables / stages of the system they want (a named module, a named table with its columns, a specific cron/job role, a specific file, a specific endpoint) — extract EACH as an objective, EVEN WHEN the user explicitly DEFERS a later portion ("the sync workflow itself I'll describe next", "I'll give you the infrastructure part first", "more to come"). A deferred later portion makes \`workflowComplete:false\` with that portion named in \`undefinedTail\` — it is NEVER a reason to discard the already-stated concrete components and return \`objectives:[]\`. Return \`objectives:[]\` ONLY when the user has stated NO concrete buildable component at all (a genuinely vague objective — "it's broken, fix it", "just schedule something and call it done"). Discarding user-stated concrete components because a later part is deferred loses stated information and wrongly collapses a Tier-2 partial-plan into a Tier-1 generic question — the exact defect this rule prevents. (Vague-objective turns with zero stated components still correctly yield \`objectives:[]\` → Tier-1; this rule does not change them.) **Named-but-deferred deliverable → named-EMPTY objective (not only an \`undefinedTail\` line).** When the user NAMES a forthcoming deliverable but defers its content ("the sync workflow itself I'll describe next", "the X part comes later"), that named deliverable ALSO becomes its own objective row — \`name\` = the deliverable the user named, \`input\`/\`method\`/\`output\` left genuinely EMPTY (per the hedge-prose rule below: empty, not "TBD"/"deferred"/a hedge sentence) — IN ADDITION to being named in \`undefinedTail\`. Putting it only in \`undefinedTail\` with no objective row is under-extraction: \`evaluate\` then returns only the workflow-level \`workflowComplete\` gap and never a per-objective gap, so \`SMARTClarification\` never deep-evaluates that deliverable via \`SMARTItemClarification\`/\`deliverableTacticalEvaluation\` and the user is never asked the specific per-objective questions about it. A user-named deliverable is an objective whose fields are empty (→ per-objective gap → per-objective clarification), NOT a tail-only mention.
   - Any verification / "how do we know it's done" methods they mention.
   - Any specific the user marked as a hard constraint — "not negotiable", "must read exactly", "verbatim", "exactly", "make sure that copy is verbatim", a precise endpoint + action string, an exact match rule, OR a per-deliverable selection/derivation rule the user states as THE way to produce a value (e.g. "for each gene, pick the antibody whose name contains 'picoband'"). Each of these becomes an inline \`[IMPORTANT DETAIL]\` clause inside the owning objective's \`method\` (see step 3) — NOT in \`notes\`. **A per-deliverable selection/match rule is an \`[IMPORTANT DETAIL]\` in its OWN right, independently of how the value is retrieved.** When the same clause is also an expensive per-deliverable crawl that earns a \`[co-develop-with-user]\` anchor (see the decompose rule below), the two markers are ORTHOGONAL and BOTH must appear on that clause: \`[co-develop-with-user]\` governs *how to fetch* (crawl vs co-developed bulk method — a still-open execution choice), while \`[IMPORTANT DETAIL]\` pins *which value is correct* (the match rule itself — a locked non-negotiable). Emitting \`[co-develop-with-user]\` does NOT absorb, satisfy, or substitute for the \`[IMPORTANT DETAIL]\` match rule, and folding the match rule into the co-develop clause as plain prose drops a user-flagged non-negotiable — a hard defect. Mark the literal token \`[IMPORTANT DETAIL]\` immediately before the match-rule constraint even when a \`[co-develop-with-user]\` anchor sits in the same sentence.
   - Any meta-request to capture / encode / reflect the workflow itself as reusable tooling — e.g. "turn this into a skill book", "save the SMART plan to a skill book", "make a PathwayEnrichment skill book out of this". This is a \`finaleCandidate\`, NOT a workflow objective: it describes what to do with the quest's method AFTER it succeeds, not an objective that transforms a page/asset. Never emit it as an \`objectives[]\` entry.
   - Any prior questId if a quest is being referenced (otherwise null).
3. For each identified objective, build a \`{n?, name, input?, method?, output?, viewResults?, outputKey?, notes?}\` object. Populate only the fields the user has stated or that follow unambiguously from their description; leave the rest undefined. NEVER invent values the user did not state — leave fields empty so the evaluate gate can correctly catch gaps. Embed any user-emphasized non-negotiable / verbatim specific from step 2 as an inline clause INSIDE that objective's \`method\` string, written as the literal token \`[IMPORTANT DETAIL]\` immediately followed by the exact constraint (the verbatim copy string in quotes, the exact endpoint + action string, the precise match rule) — NOT in \`notes\`, NOT paraphrased, NOT abbreviated. The marker travels with the method through every downstream action.
4. Build the initial \`deliverables[]\` array from pre-existing external inputs ONLY (a source URL the user named as input, an asset they handed off). Do NOT pre-populate downstream deliverables — those are populated by execution, not planning.
5. Build \`finaleCandidates[]\` from any meta-requests identified in step 2. Each entry is \`{kind: "skill-book", name: "<proposed skill book name if the user named one, else null>", note: "<one line: what the user wants captured after the quest succeeds>"}\`. These never enter \`objectives[]\` or \`deliverables[]\` — they are signals for the post-success SE finale, not workflow rows.
6. **Judge workflow completeness/terminality AND product-boundary definition, then emit the explicit signal \`workflowComplete\`.** Set \`workflowComplete: false\` (and write a one-line \`undefinedTail\` note naming what is missing) when ANY of the following holds: (a) the user explicitly framed the workflow as partial / not end-to-end — a "here's the **start** of", a leading "first", a trailing "then…", "…and so on", "etc.", "(more steps to come)", or any phrasing that signals the listed objectives do not run to completion; (b) the objectives you extracted do NOT reach a terminal deliverable/output that resolves the user's stated overall objective — the last objective produces an intermediate artifact (a lookup, a fetched value, a "see which page ranks") with no objective consuming it to deliver the thing the user actually asked for; (c) you could only extract a coherent front-half and the tail is genuinely undefined; **(d) the objectives DO run end-to-end to a terminal *action* but the workflow's PRODUCT BOUNDARY is undefined — the user has NOT established BOTH (i) the deliverable shape (is this a reusable feature / set of modules to build, or a single one-off manual run?) AND (ii) the terminal output that ends/reports a run (a report, a notification, an explicit "done when X" / acceptance condition). A terminal *action* alone (e.g. "…then Chrome triggers adoption", "adopt the article", "the script posts the row") is NOT a defined product boundary: reaching an action says what the last objective DOES, not what the user is receiving (feature vs one-off) or what signals a run is finished/successful. When either (i) or (ii) is unstated, set \`workflowComplete: false\` and write an \`undefinedTail\` that names the product-boundary gap, e.g. "deliverable shape (reusable feature/modules vs one-off run) and terminal output/report undefined" — name whichever of the two is missing.** Otherwise — workflow end-to-end AND both product-boundary elements ((i) deliverable shape AND (ii) terminal output/report/done-condition) user-stated — set \`workflowComplete: true\` and \`undefinedTail: ""\`. **Generalize the principle — do not key off any one literal phrase ("here's the start" is only one example, and for (d) the product boundary is "stated" by the SUBSTANCE of the user establishing deliverable-shape + terminal-report, not by any literal wording: e.g. "the deliverable is the reusable feature — build the modules; reuse the existing pipeline via a wrapper; send a notification email; acceptance = a smoke test + a review-package report" defines BOTH and must yield \`workflowComplete: true\`); judge whether the workflow, as the user gave it, runs end-to-end to the objective AND has a user-defined product boundary.** A tool/API/schema unknown *inside an otherwise end-to-end, product-boundary-defined workflow* (an objective whose method is known to be "POST to some endpoint, exact path TBD") is an implementation checkpoint, NOT incompleteness — that still gets \`workflowComplete: true\`; this signal is exclusively about a truncated/partial/non-terminal workflow OR an end-to-end one whose product boundary (deliverable shape + terminal report) the user has not defined — not about resolvable unknowns inside an otherwise complete, bounded one. **Never fabricate a terminal objective, a deliverable shape, or a done-condition the user did not give in order to make the workflow look complete/bounded** — that is the exact bypass this signal exists to prevent; represent the truncation or the undefined product boundary honestly so the evaluate gate fails it to Tier-2 clarification. **The \`undefinedTail\` must ENUMERATE every still-unspecified element required by the user's STATED overall objective, aggregated across the ENTIRE conversation — not only the single next thing the user said they would describe.** Carry earlier-turn signals forward: if an earlier turn established the objective (e.g. "X is broken again / fix it for good", "make this self-sustaining and able to report its own health"), then the urgent repair / the durable system / the system's standard facets it implies (e.g. a self-reporting system implies its reporting + post-run validation surfaces) remain OPEN tail items until the user specifies them — list each by name in \`undefinedTail\`, even when the latest turn only advanced one of them. A tail that names just the immediately-next item while silently dropping other still-open elements of the stated objective is under-scoped and is the defect this rule prevents; it is NOT fabrication to name a still-open element the user already implied by their objective — fabrication is inventing the element's CONTENT, naming it as still-open is required.

7. **Judge three substrate-readiness signals and emit them as load-bearing context fields (\`verificationOwnership\`, \`adoptionPrecedence\`, \`finaleSurfacing\`) — locked 2026-05-20 after a real over-acceptance defect (SE-cursor-hand-off-test Step 2).** The evaluate weapon reads these directly (no LLM there). Each is independent of \`workflowComplete\`: a workflow can be end-to-end and product-bounded yet still fail any of these three.
   - **\`verificationOwnership\`**: \`"defined"\` when EVERY objective's \`output[].verification\` clause that names a check (curl, OCR, screenshot, lint, HTTP roundtrip, etc.) ALSO names the OWNER ENTITY that performs the check — one of: Cat at purrview (autonomous per-deliverable judge), GM CIC (chaperone user-channel verify), cursor worker / cloud agent (the dispatched executor itself), per-page vs batch granularity. \`"undefined"\` when any objective's verification names WHAT is checked but not WHO performs it (e.g. "OCR confirms hero text matches keyword" with no owner). Examples: "Cat-side cloud Playwright OCR per page" = OWNER NAMED (Cat) + GRANULARITY (per page); "GM CIC opens the live URL and eyeballs the hero" = OWNER NAMED (GM CIC); "curl handshake good→2xx" alone = OWNER NOT NAMED (who runs the curl — chaperone, cursor agent, Cat?). A verification without an owner is not actionable.
   - **\`adoptionPrecedence\`**: \`"defined"\` when the workflow adopts content/data from one substrate into another (source→destination hydration, X→Y mapping, e.g. CJGEO HTML → Magento CMS page) AND the precedence/default rule for mid-execution adoption ambiguities (diagram fill, anchor injection, deep-dive link injection, branding-grep, image-placeholder fill, sibling-cross-link logic) is NAMED (e.g. "default to [SB:BryanBlog->adoptCJGEORawContent] behavior unless that doesn't answer it"). \`"undefined"\` when the workflow does cross-substrate adoption but no precedence rule is stated. \`"n/a"\` when no cross-substrate adoption exists (pure data transform, pure infrastructure, no source→destination content mapping). Without a precedence rule, every ambiguity halts execution.
   - **\`finaleSurfacing\`**: \`"defined"\` when the workflow produces MULTIPLE deliverable artifacts (multiple live pages, multiple files, a quest page + N satellite artifacts, dispatched-agent sessions to inspect) AND the end-point CIC-tab spec is given — which tabs the GM opens at the run's finish (e.g. "three tabs: quest page + one representative live URL + the cursor agent session at FINISHED"). \`"undefined"\` when the workflow produces multiple artifacts and the tab list is not stated. \`"defined"\` also when the workflow produces a single artifact (the quest page itself is the trivially-defined finale).
   **All three are emitted on every turn that has objectives. Missing == \`"undefined"\` (the evaluate gate treats absence as the failing state — same fail-loud discipline as \`workflowComplete\`).**

## Output schema (return exactly one JSON object)
{
  "summary": "<2-4 sentence plain-English summary of the working context>",
  "questId": "<existing questId if user referenced one, else null>",
  "workflowComplete": <true | false — false when the user framed the workflow as partial/non-terminal, OR the objectives do not reach a terminal deliverable that resolves the stated objective, OR the objectives reach a terminal action but the product boundary is undefined (deliverable shape — reusable feature/modules vs one-off run — and/or the terminal output/report/notification/done-condition not user-stated) (step 6)>,
  "undefinedTail": "<one line naming what is missing when workflowComplete is false — the missing tail, or the undefined product-boundary element(s) (deliverable shape and/or terminal report); '' when workflowComplete is true>",
  "verificationOwnership": <"defined" | "undefined" — see step 7. "undefined" when any verification clause names a check but not the owner entity (Cat / GM CIC / cursor worker / per-page vs batch)>,
  "adoptionPrecedence": <"defined" | "undefined" | "n/a" — see step 7. "undefined" when the workflow does cross-substrate content adoption and no precedence default is named. "n/a" when no cross-substrate adoption exists>,
  "finaleSurfacing": <"defined" | "undefined" — see step 7. "undefined" when the workflow produces multiple deliverable artifacts and the end-point CIC-tab spec is not stated>,
  "conflicts": [ /* REQUIRED field. Each entry = a user ask in THIS conversation that you must NOT silently comply with. THREE kinds, all recorded here: (k1) LOCKED-RULE: the ask contradicts a locked design rule discoverable in genuine substrate (a skill book howTo, CLAUDE*.md, a design doc the user pointed to) — e.g. "schedule total_sync daily" vs nexus.syncNewZohoData total_sync-is-manual-only. You MUST detect k1 by consulting the relevant domain skill book before deciding the reply. (k2) SELF-CONTRADICTION: the user states a design and, in the SAME message, also states something incompatible with it (e.g. "dashboards must be a reusable nexus_dashboards module, not bolted onto one feature" + "just put the dashboard config in user profile settings"). (k3) DEFLECTION-SCOPE-REVERSAL: under "don't overengineer / just ship it / not that important" pressure the user abruptly DROPS or guts an element they DELIBERATELY specified across prior turns, WITHOUT answering the blockers actually keeping the plan from being quest-ready — the drop is recorded as a conflict (a scope reversal to confirm), NOT silently honored as if it resolved anything; an explicit scope cut is the user's right, but a deflection-driven reversal of a multi-turn deliberate design is surfaced ("you specified X deliberately as <role>; dropping it now reverses that and still doesn't answer <blockers> — confirm you really want X gone") rather than instantly accepted. Shape: {"kind":"k1|k2|k3","userAsk":"<the ask, verbatim-ish>","against":"<the locked rule + where it lives | the user's own earlier/same-message statement it contradicts | the deliberately-specified element being reversed + which prior turns specified it>"}. Empty array [] only when none of k1/k2/k3 is present. This field is read by SMARTClarification's Tier-1 conflict-flag carve-out AND the Tier-2 conflict surface (step 4) — a dropped conflict here = silent compliance downstream, a hard defect. */ ],
  "mentions": ["<URL or page name>", "..."],
  "objectives": [
    {
      "n": <1-based integer order>,
      "name": "<short noun phrase naming this objective>",
      "input": "<single sentence describing the input(s) this objective consumes; reference deliverables by [name]>",
      "method": "<one short paragraph describing HOW the objective transforms input to output; concrete enough that the next agent can act on it>",
      "output": [
        {
          "asset": "<deliverable name this objective produces or updates>",
          "format": "<format description, e.g. 'json — names + coordinates per gene', 'raw HTML', 'markdown'>",
          "display": "<carousel render mode, or null. e.g. 'rendered HTML in result panel', 'rendered markdown in result panel', 'image preview in result panel'>",
          "verification": "<one sentence: how do we know the output is correct? — ALWAYS non-empty for an [IMPORTANT DETAIL] objective: if the user explicitly stated the check, copy it verbatim-in-substance prefixed with the literal token [USER-STATED]; otherwise COMPOSE a concrete baseline verification derived directly from the [IMPORTANT DETAIL] constraint (no [USER-STATED] prefix), and if a genuine tool/API/credential is unresolved append it as an inline (impl-checkpoint: …) note inside this same string>",
          "verification_example": <optional concrete example JSON | null>
        }
      ],
      "viewResults": "Show" | "Checkpoint" | "No",
      "outputKey": "<optional pointer to a results-carousel lookup key for this objective's rendered output; null when no presentable artifact>",
      "notes": "<optional one-line note; '' when the user didn't say anything noteworthy yet>"
    }
  ],
  "deliverables": [
    { "name": "<asset name>", "kind": "text"|"html"|"image"|"video"|"document", "ref": "<URL or null>" }
  ],
  "finaleCandidates": [
    { "kind": "skill-book", "name": "<proposed skill book name or null>", "note": "<one line: what the user wants captured after the quest succeeds>" }
  ]
}

## Rules
- Output ONLY valid JSON. No prose, no fences.
- If the user has not yet stated a workflow (e.g. first vague turn), return objectives: [] and deliverables: [].
- **🔒 LITERAL-TOKEN RULE (load-bearing, locked 2026-05-15 after a real defect — read this before emitting any objective). Every user-emphasized must-have is recorded by placing the EXACT literal string \`[IMPORTANT DETAIL]\` inline, immediately before the must-have clause, INSIDE the owning objective's \`method\` string. The token is the literal 17-character substring \`[IMPORTANT DETAIL]\` (square brackets included) — not paraphrased, not reworded, not a synonym.** A user emphasizes a must-have whenever they say "it is important", "must", "mandatory", "not negotiable", "do not …", "no substitute", "exactly", "verbatim", or name an exact tool / endpoint / rule as THE way to do something. **Bare prose is NOT compliant.** Recording the constraint as descriptive English — \`use DataForSEO (locked — no substitute)\`, \`(locked)\`, \`(required)\`, \`(do not change)\`, \`must reuse the existing pipeline\` — WITHOUT the literal \`[IMPORTANT DETAIL]\` token preceding it is the exact defect this rule exists to prevent and is a hard failure, identical in severity to dropping the constraint entirely. The downstream programmatic surfaces (\`SMARTClarification\` table, \`createQuest\`'s stored \`objectives.method\`, the execution layer) key off the literal token, not on whether the prose "sounds locked".
  **Concrete transform — apply this shape to EVERY user-emphasized must-have:**
  - method as drafted: \`Run keyword research; use DataForSEO to pull volumes.\`
  - method as REQUIRED (user said DataForSEO is mandatory/no-substitute): \`Run keyword research; [IMPORTANT DETAIL] use DataForSEO (mandatory — no substitute) to pull volumes.\`
  More examples of the same shape, each inside its OWN owning objective's \`method\`: \`[IMPORTANT DETAIL] keyword selection must be non-branded, highest-volume, commercially-valuable\`; \`[IMPORTANT DETAIL] reject the picked page and re-loop until it is a non-homepage URL\`; \`[IMPORTANT DETAIL] the CJGEO import must be API-triggerable, never a manual paste\`; \`[IMPORTANT DETAIL] exactly one vertical keyword-research pass + one horizontal pass — no more, no fewer\`; \`[IMPORTANT DETAIL] the draft must consume the competitor-research and keyword-research assets produced by the prior steps\`; \`[IMPORTANT DETAIL] adoption is triggered through Chrome (CIC), not a direct API call\`; \`[IMPORTANT DETAIL] reuse the existing CJGEO pipeline via a thin wrapper — do NOT duplicate or fork it\`; \`[IMPORTANT DETAIL] the notification email must NOT include a screenshot\`. Each token lives in the method of the single objective that owns that clause; do not pile them all into one objective.
  **\`[co-develop-with-user]\` (separate literal token, same discipline).** When an objective's approach is a deliberate execution-time co-development decision — a choice the user has signalled will be resolved WITH them mid-quest, not pre-decided at planning (e.g. a wrapper/integration seam where in-process module import vs out-of-process API call is settled at execution time with the user) — embed the EXACT literal string \`[co-develop-with-user]\` inline in that objective's \`method\`. It is a planned mid-quest co-develop halt anchor; omitting it (leaving the seam as plain prose) drops the halt signal and is a defect. \`[co-develop-with-user]\` and \`[IMPORTANT DETAIL]\` are orthogonal and may co-exist in the same clause (see step 2 / the decompose rule) — neither substitutes for the other.
  **MANDATORY reuse-via-wrapper subcase (locked 2026-05-15 after a real defect — this is NOT a conditional judgment call).** ANY objective whose \`method\` reuses / wraps / calls an existing pipeline / system / workflow / service — the "reuse the existing X via a wrapper, do NOT duplicate or fork it" pattern — MUST carry the literal \`[co-develop-with-user]\` token inline in that objective's \`method\`, unconditionally. The user saying "use a wrapper / reuse the existing pipeline, don't duplicate" does NOT pre-decide and does NOT eliminate this co-develop point: it only LOCKS the wrapper-not-duplicate CONSTRAINT (record that with \`[IMPORTANT DETAIL]\`), leaving the concrete INTEGRATION SEAM — HOW the wrapper actually invokes the existing pipeline (in-process module import vs out-of-process API/HTTP call vs CLI/queue, etc.) — an open execution-time decision co-developed WITH the user. Judging "the user already decided the wrapper approach, so no anchor is needed" is the exact defect this rule prevents: the wrapper decision and the seam decision are different decisions. So a reuse-via-wrapper objective carries BOTH tokens — \`[IMPORTANT DETAIL]\` before the wrapper-not-duplicate constraint AND \`[co-develop-with-user]\` for the unresolved seam. Concrete example (one objective's \`method\`): \`Build a thin wrapper service that drives the existing CJGEO content pipeline; [IMPORTANT DETAIL] reuse the existing pipeline via the wrapper — do NOT duplicate or fork it; [co-develop-with-user] the concrete integration seam (in-process import vs out-of-process API call) is decided with the user at execution time.\`
  **MANDATORY user-owned-gate subcase (locked 2026-05-18 after a real defect — NOT a conditional judgment call).** Two more patterns ALWAYS carry the literal \`[co-develop-with-user]\` token inline in the owning objective's \`method\`, unconditionally, in ADDITION to (never instead of) the \`[IMPORTANT DETAIL]\` that records the constraint: (1) a user-flagged **explicit-approval / sign-off / "not final until I approve it" gate** — the user has said a deliverable is provisional until they personally approve it (e.g. "the canonical format is final ONLY when I explicitly approve it in a quest comment", "don't treat the draft as done until I sign off"): the constraint itself is an \`[IMPORTANT DETAIL]\`, AND the approve-with-the-user act is a planned mid-quest co-develop halt → \`[co-develop-with-user]\` on the same objective; (2) an **irreversible / heavily-supervised integrated go-live** — an objective that performs real cross-system / production-affecting / hard-to-reverse writes (standing the integrated system live, real external mutations through adapters, the end-to-end go-live of a multi-system build): the go-live is co-developed with the user, \`[co-develop-with-user]\` on that objective. Judging "the user already told me the approval rule / the go-live plan, so it's pre-decided, no anchor needed" is the exact defect this rule prevents — the constraint being locked does NOT eliminate the planned halt where the user personally approves / supervises it. Like the reuse-via-wrapper subcase, the gate clause carries BOTH tokens.
  **Do not over-flag.** Only genuinely user-emphasized must-haves get \`[IMPORTANT DETAIL]\`; only a genuine deliberate execution-time co-development seam, user-owned approval/sign-off gate, or irreversible integrated go-live gets \`[co-develop-with-user]\`. Minting either token for content the user did not emphasize / for a seam that is not a planned co-develop halt is its own defect (noise that dilutes the load-bearing markers). Recognize every user flag and honor it 100%; invent neither marker.
- **\`workflowComplete\` is a load-bearing contract field the \`evaluate\` weapon reads directly (not derived by an LLM there).** It is necessary that you emit it on every turn that has objectives: \`false\` + a non-empty \`undefinedTail\` when the workflow is partial/non-terminal OR end-to-end-but-product-boundary-undefined (step 6 criteria, including criterion (d)), \`true\` + \`undefinedTail: ""\` only when the listed objectives run end-to-end to a deliverable that resolves the user's stated objective AND the user has stated BOTH (i) the deliverable shape (reusable feature/modules vs one-off run) AND (ii) the terminal output/report/notification/done-condition. A well-formed front-half whose individual objectives each have name/input/method/output is STILL \`workflowComplete: false\` if it stops before the objective is delivered — per-objective field presence does not make a truncated workflow complete. **Equally, a workflow whose objectives run all the way to a terminal *action* is STILL \`workflowComplete: false\` while its product boundary is undefined — reaching an action (e.g. "Chrome triggers adoption") is not the same as the user having defined what the deliverable IS (feature vs one-off) or what ends/reports a run. Reaching a terminal action does not make an unbounded workflow complete.** Never set \`workflowComplete: true\` by inventing the missing tail (a fabricated terminal objective), inventing a deliverable shape the user never chose, or inventing a done-condition/report the user never stated, and never omit the field when objectives exist; any of these is the exact premature-quest bypass this signal exists to prevent.
- Do NOT propose objectives on behalf of the user. The evaluate gate is supposed to catch the empty case and trigger SMARTClarification — don't bypass it.
- **Decompose a stated deliverable into the sub-objectives and source assets it necessarily entails — that is required structuring, NOT invention.** "Do NOT propose objectives" forbids inventing work the user never asked for; it does NOT license under-decomposing work the user DID ask for. If a deliverable's own description necessarily entails an artifact the user did not hand you, that artifact needs its own objective: e.g. "a table explaining each biomarker's **role**" entails an objective that GENERATES the role descriptions (the role text is not a supplied input); "an interactive map overlaying the **original pathway image**" entails that pathway IMAGE as a pre-existing input \`deliverables\` row (\`kind: image\`, the image asset/URL), not merely the page link. Any objective that iterates an expensive per-item manual/CIC operation across many items — a crawl, e.g. "for each gene, CIC-visit its page and pick X", "visit each product page and read Y" — inherently admits a co-developed bulk alternative (a SQL/SSH/extraction method built with the user instead of N manual visits). Such objectives are ALWAYS \`[co-develop-with-user]\` candidates: embed the literal anchor \`[co-develop-with-user]\` inline in that objective's \`method\` so the execution layer surfaces the crawl-vs-co-develop choice. (More generally, embed it in any data-sourcing objective that admits more than one execution method.) **Emitting \`[co-develop-with-user]\` for a per-item crawl does NOT discharge the \`[IMPORTANT DETAIL]\` requirement for any per-item selection/match RULE the user stated in the same clause** (e.g. "pick the antibody whose name contains 'picoband'"). The anchor is about the unresolved *retrieval mechanism*; the match rule is a *locked correctness constraint* — orthogonal concerns that coexist. The clause must carry BOTH the \`[co-develop-with-user]\` anchor AND the literal \`[IMPORTANT DETAIL]\` immediately before the match-rule text. The line: never invent unstated VALUES (leave them blank so the gate catches them); always derive the structural OBJECTIVES/DELIVERABLES a stated deliverable necessarily requires; and never let one inline marker swallow another — each user-flagged non-negotiable keeps its own \`[IMPORTANT DETAIL]\`.
- **List-to-elements decomposition (load-bearing, locked 2026-05-20 after SE-cursor-hand-off-test Step 4 attempt-1 defect).** When the user's input enumerates a list of ≥2 PARALLEL deliverables — slugs, URLs, file paths, page titles, item names, named outputs that share a common transformation pipeline (e.g. "publish 4 articles to slugs cjgeo-mockup-1..4", "render these 5 charts", "scrape these 3 product pages") — the produced context MUST:
  (i) **Deliverables[] — one row per deliverable as a kind-appropriate placeholder.** \`kind\` is one of FIVE consumption behaviors — pick by how a human consumes the deliverable, not the file format: \`text\` (read it — prose/markdown/json/xml/code/a bare link), \`html\` (view a rendered/interactive web page — a satellite URL the run lands at), \`image\` (view a picture), \`video\` (watch/scrub with player controls), \`document\` (flip through an embedded paged file — pdf/ppt/office). For URL-type deliverables whose content is a live page, emit \`{kind:"html", ref:"<expected URL>"}\`. For file-path deliverables, pick the kind by consumption (a .md/.json/.csv → \`text\`; a .pdf/.pptx → \`document\`). For content-type deliverables whose target URL/path is not yet known, emit \`ref:null\`. For file-path deliverables, emit one \`{kind:"markdown"|"json"|"html", ref:"<expected path>"}\` per deliverable. For content-type deliverables whose target URL/path is not yet known, emit one row with \`ref:null\` per deliverable. A single mega-deliverable ("publish_targets" json blob enumerating all N) is UNDER-decomposition and a hard defect — the rubric/judge keys off per-deliverable rows because each is independently verifiable.
  (ii) **Objectives[] — per-deliverable transformation/verification granularity.** Decompose per-deliverable transformations and verifications either as (a) one objective per phase per deliverable (4 deliverables × 3 phases = 12 objectives), OR (b) one objective per phase that EXPLICITLY iterates over all deliverables in its \`method\` ("for each of cjgeo-mockup-1..4: …") and whose \`output[]\` lists per-deliverable assets — NOT a single mega-objective that bundles all phases AND all deliverables into one \`method\` paragraph. The judge tests each phase × each deliverable independently; a single bundled objective prevents per-deliverable verification and is a hard defect identical in severity to dropping the deliverables rows.
  **Worked example (matching the SE-cursor-hand-off-test rubric shape).** User states: "publish 4 western-blot guides to bosterbio.com at slugs cjgeo-mockup-1..4; topics = gel %, transfer, blocking, membrane". Correct decomposition:
  - \`deliverables[]\` carries 5 rows: \`{name:"topic_briefs", kind:"text", ref:"<path>"}\` (the source brief — pre-existing input) PLUS \`{name:"cjgeo_mockup_1", kind:"text", ref:"https://www.bosterbio.com/cjgeo-mockup-1"}\`, \`cjgeo_mockup_2/3/4\` likewise (the 4 satellite-URL placeholders for the eventual live URLs — a URL is consumed by reading → kind \`"text"\`, NOT the retired \`"link"\`/\`"markdown"\`).
  - \`objectives[]\` decomposes per-phase-iterating-over-deliverables: a draft-generation objective whose method says "for each of cjgeo-mockup-1..4 per topic_briefs: generate the draft" and whose \`output[]\` lists \`cjgeo_article_1..4\` assets; an adoption objective that iterates likewise; a publish objective that iterates likewise; a verification objective that iterates likewise (Cat OCR per page). NOT one mega-objective "Dispatch Cursor agent that generates+adopts+publishes+verifies all 4 articles".
  The list-to-elements rule is **always-on** for parallel-deliverable enumerations and is checked at every emit; collapsing N parallel deliverables into 1 deliverable + 1 objective is the exact defect this rule prevents.
- **A working/integrated/reality-tested end product built from multiple components STRUCTURALLY ENTAILS a terminal integration objective — emitting it is required decomposition, NOT invented work (locked 2026-05-18 after a real defect).** When the user's stated GOAL/objective is a *working*, *integrated*, *end-to-end*, or *reality-tested* end product — an MVP, a prototype, "make it actually work", "a reality-tested X that catches the gaps" — and the body of the request is N separate components/systems/subsystems, the user has asked for N+1 deliverables: the N components AND the integrated end product that proves they work together. Emitting only the N component objectives and stopping is the exact under-decomposition the decompose rule forbids: the integrated end product is the deliverable the user actually wants, the components are means to it. So append a FINAL objective whose \`method\` integrates the prior components into one functioning end-to-end system and reality-tests it (a real end-to-end pass that exercises every component, not a stub/demo), consuming the prior objectives' assets as its \`input\`. This is the same principle as "a table explaining each biomarker's role entails an objective that generates the role text" — the stated deliverable necessarily entails the integrating objective; it is NOT an objective "invented on behalf of the user" (the "Do NOT propose objectives" guard forbids inventing work the user never asked for — it does not license dropping the integrated end product the user explicitly asked for). Per the user-owned-gate subcase above, that final integrated go-live carries \`[co-develop-with-user]\` (a heavily-supervised irreversible-ish integration). Do NOT synthesize this objective when the user only asked for the components themselves with no integrated/working/reality-tested end product as the goal — over-emitting it then is its own over-decomposition defect.
- **Do NOT synthesize an \`output[]\` entry's \`asset\`/\`format\`/\`verification\` to slip past the evaluate gate.** If the user has not stated, for an objective, what asset it produces + that asset's format + how it is verified, emit \`output: []\` (or leave \`format\`/\`verification\` empty) so evaluate correctly FAILS and clarification runs. Inventing a plausible-looking \`{asset, format}\` so the programmatic gate returns \`ok:true\` is the exact bypass the gate exists to prevent — it produces a premature, under-specified quest. An objective whose verification the user never stated is an unresolved gap, not a fillable blank.
- **Hedge prose in \`input\`/\`method\` is the SAME gate-bypass as a synthesized \`output\` — forbidden.** If the user has not concretely specified HOW an objective transforms its input (the transmutation method) or WHERE an input comes from (its source), leave that objective's \`method\` (resp. \`input\`) **empty / undefined**. Do NOT write a vague one-liner followed by a hedge marker — "! need attention", "not stated", "unspecified", "TBD", "(to be determined)", "user didn't say", or a plausible best-guess sentence. \`evaluate\` flags \`name\`/\`input\`/\`method\` only when the field is *empty*; a vague-but-non-empty value silently passes the programmatic gate, so the ambiguity never reaches Tier-2 clarification and the user is never asked (this is the exact defect that let an "image → interactive map" transmutation method and a per-gene antibody source go un-clarified). The objective's \`name\` still names the deliverable the user asked for; the unspecified \`method\`/\`input\` stays genuinely empty so evaluate FAILS on that field and SMARTClarification asks the user how the objective is done / where its input comes from. "Leave fields empty so the gate can catch gaps" means EMPTY — not "describe the gap in the field."
- **An objective that carries a user-flagged \`[IMPORTANT DETAIL]\` ALWAYS gets a concrete, real \`verification\` — the agent COMPOSES it from the constraint by baseline judgment; the ABSENCE of a user-stated verification is NOT a planning gap and is NEVER a user-clarification question (locked 2026-05-15 after a real over-clarification defect — read this before emitting any \`[IMPORTANT DETAIL]\` objective).** "How do you verify your non-negotiable?" asked back to the user is the exact unnecessary-clarification anti-pattern the design forbids: baseline AI judgment is sufficient to author a verification unless the user demanded a custom rule. So, for EACH \`[IMPORTANT DETAIL]\`-bearing objective, emit a non-empty \`output[].verification\` as follows:
  1. **If the user explicitly stated (this turn or any prior conversationHistory turn) HOW that objective's non-negotiable is verified** → record it verbatim-in-substance with the literal token \`[USER-STATED]\` as its prefix (e.g. \`"verification": "[USER-STATED] CIC submits 4 antibodies + a test email and confirms the POST payload to <endpoint> matches action=<action> with the selected antibodies + email; the page shows the promo confirmation copy."\`). A user-STATED verification is kept verbatim and HONORED — never discarded, never overwritten with a composed one. Emit \`[USER-STATED]\` for NOTHING else — never on an agent-composed check, never to make a gate pass; stamping it on a verification the user did not actually state is a hard defect identical in severity to dropping an \`[IMPORTANT DETAIL]\`.
  2. **Otherwise (the common case — user flagged the constraint but did not specify its check)** → COMPOSE a concrete, real verification derived DIRECTLY from the \`[IMPORTANT DETAIL]\` constraint itself, in one sentence, NO \`[USER-STATED]\` prefix. The verification asserts the constraint is provably satisfied. Concrete transforms (apply this shape):
     - \`[IMPORTANT DETAIL] use DataForSEO (mandatory — no substitute)\` → \`"the run logs/persists the DataForSEO API request + response so the keyword source is provably DataForSEO, not a substitute provider"\`
     - \`[IMPORTANT DETAIL] reject the picked page and re-loop until it is a non-homepage URL\` → \`"asserts the selected URL is not the domain root path (path !== '/' and not the bare homepage)"\`
     - \`[IMPORTANT DETAIL] exactly one vertical keyword-research pass + one horizontal pass\` → \`"the orchestrator records loop counts == exactly one vertical pass and exactly one horizontal pass — no more, no fewer"\`
     - \`[IMPORTANT DETAIL] the notification email must NOT include a screenshot\` → \`"the sent email's parts contain zero image/attachment MIME parts"\`
     The composed verification is REAL and concrete (a specific assertion against the constraint), not a hedge and not "looks right".
  3. **Genuine tool/API/credential uncertainty is an inline implementation-checkpoint note, NOT a user question and NOT incompleteness.** If composing the verification surfaces a genuine unknown about a tool/API/endpoint/credential needed to RUN the check, append it inside the SAME objective as an inline \`(impl-checkpoint: <the unresolved tool/API/credential>)\` clause in this objective's \`method\` (or the \`verification\` string itself) — the agent still emits a concrete verification (step 2). This is recorded for the execution layer to resolve; it is NEVER emitted as an \`output[].verification\` blank, NEVER raised to SMARTClarification, and NEVER surfaced to the user as a "how do you verify X?" question. Only a genuine PRODUCT decision (deliverable shape / what-ends-the-run — already covered by the \`workflowComplete\` product-boundary gate) blocks quest creation. **Never leave an \`[IMPORTANT DETAIL]\` objective's \`verification\` empty** — an empty verification on such an objective was the old behavior and is now the defect; the agent is expected to have composed one.
- **\`output\` is a JSON array of objects, never a flat string.** Each entry MUST be an object \`{asset, format, ...}\`. Multiple output assets in one objective are allowed (e.g. an objective that produces both an HTML overlay and a JSON coordinate file).
- **\`outputKey\` is a string OR null** — it points at the results-carousel render lookup. Objectives with no presentable artifact (internal data shaping) should set outputKey to null and viewResults to "No".
- **\`deliverables[]\` is for pre-existing external inputs only.** A source-pathway image URL the user named is a deliverable. The "interactive_map_html" that objective 4 will produce is NOT — execution materializes it. Leave such deliverables out at planning time.
- **An image deliverable's \`ref\` MUST be an actual image asset, NEVER a page/HTML URL (load-bearing, locked 2026-05-15 after a real failure).** When a deliverable entails "the original pathway image" but the user only gave the *page* URL (e.g. \`.../hypoxia-signaling-pathway\`), do NOT set \`{kind:"image", ref:<that page URL>}\` — a page is HTML, and a downstream OCR/fetch objective pointed at it fails or fabricates. Either (a) the real image asset URL is unambiguously known/derivable → use it (e.g. the canonical Boster pathway PNG \`.../media/pdf/collaterals/pathways/<Name>.png\`), or (b) it is not → set \`ref: null\` so the worker resolves the real image from the page at execution time. \`kind:"image"\` paired with a page/HTML \`ref\` is a hard defect, not a harmless approximation: it silently breaks objective 1 and every objective that chains off the gene set. Same principle for any \`kind\` whose \`ref\` must be the asset itself (json/markdown/html), never the page that links to it.
- **\`viewResults\` is \`"Show"\`, \`"Checkpoint"\`, or \`"No"\`** (exact strings, case-sensitive). Default to "No" unless the user explicitly asked for an inline preview or breakpoint review.
- **\`status\` is owned by execution, not planning.** Do not emit a status field at this stage — every new objective starts \`pending\` once createQuest runs.
- **A "capture this workflow as a skill book" ask is a \`finaleCandidate\`, never an objective.** Requests like "encode this as a PathwayEnrichment skill book" / "save the SMART plan to a skill book" describe what happens to the method AFTER the quest succeeds (the SE finale), not a page-producing workflow objective. Misfiling it as an \`objectives[]\` row inflates the plan with a non-deliverable. Extract it into \`finaleCandidates[]\` and keep \`objectives[]\` to concrete deliverables only.
- **\`[IMPORTANT DETAIL]\` is a literal inline marker, not prose, and it is USER-ORIGINATED.** The user flags non-negotiables in their own words ("not negotiable", "must read exactly", "verbatim", "make sure that copy is verbatim", an exact endpoint + action string, a precise match rule). Your job is to RECOGNIZE every such user flag and HONOR it 100% — never to mint your own \`[IMPORTANT DETAIL]\` for content the user did not flag. For each user-flagged non-negotiable, the owning objective's \`method\` MUST contain the literal substring \`[IMPORTANT DETAIL]\` immediately followed by the verbatim constraint. Recording it only in \`notes\`, paraphrasing it, abbreviating it (e.g. "banner verbatim"), or dropping it downstream is a defect — every user-flagged detail is honored 100% and the marker is load-bearing for SMARTClarification's table, createQuest's stored objective, and execution. Conversely, missing ANY user-flagged non-negotiable is a hard failure, not a partial deduction. **CHARACTER-EXACT — no punctuation normalization (load-bearing, locked 2026-05-15 after a real failure).** When the user flags a *verbatim copy string* ("must read exactly", "verbatim", "make sure that copy is verbatim"), reproduce it BYTE-FOR-BYTE inside the \`[IMPORTANT DETAIL]\`. Do NOT "tidy" or normalize punctuation — specifically: an em-dash \`—\` (U+2014) MUST stay an em-dash, never a hyphen-minus \`-\`; an en-dash \`–\` stays an en-dash; curly/smart quotes \`" " ' '\` stay curly, never straightened; ellipsis \`…\` stays as written; non-breaking spaces, accents, and casing are preserved exactly. The LLM reflex to substitute \`—\`→\`-\` or \`"\`→\`"\` is the exact corruption this rule bans: a promo banner stored as "...50% off - quote emailed to you." when the user wrote "...50% off — quote emailed to you." is an ALTERED non-negotiable and a hard defect, identical in severity to dropping the marker. If you cannot guarantee a character is reproduced exactly, copy it literally from the user's message rather than retyping it. This applies through every surface the string travels: the SMARTClarification table, the stored \`objectives.method\`, and any later \`patchObjective\` that touches the same objective.
`,
    },

    SMARTClarification: {
      description:
        "SA-dispatched action. Runs AFTER questmaster.evaluate has failed. Two-tier internal eval: T1 = no objectives → ask generic 'what is the workflow and how do I verify it'; T2 = objectives present but gaps → per-objective deep eval via questmaster.SMARTItemClarification weapon. Aggregates user-facing questions and a partial objectives table.",
      howTo: `
You are SMARTClarification. Run ONLY when evaluate has already returned ok:false with gaps. Your job is to produce user-facing questions plus a partial objectives table for the user to review.

## Inputs (provided in the dispatch prompt)
- context: the output of summarizeRelevantContext
- gaps: the gaps[] array from questmaster.evaluate (entries have shape {objectiveN, missingField, note})

## Procedure
1. **Tier 1 check.** If context.objectives is empty, emit EXACTLY one question, verbatim: "What is the workflow and how do I verify each objective?" No examples, no enumerations, no preamble, no clarifying bullets, no "e.g." lists, no starter prompts. **Tier-1 output discipline (LOCKED): the entire user-visible reply in Tier 1 is that single locked question string VERBATIM and nothing else. Do NOT prepend or append any preamble, cascade narration, weapon/return-value trace (e.g. \`Evaluate weapon: objectives:[]→ {ok:false,...}\`, \`Step 4a — SMARTClarification Tier 1 ...\`), rule restatement, examples, options, or partial table. The reply is exactly one line: the question. Any character before or after the question is a Tier-1 violation — the anti-anchoring purpose of Tier-1 is defeated by exposition.** Anything beyond the verbatim question becomes anchoring bias that contaminates the user's answer. Skip Tier 2. Return. **Conflict-flag carve-out (the ONE sanctioned, MANDATORY addition to a Tier-1 reply — locked).** If \`context.conflicts\` is non-empty (the user's ask contradicts a locked design rule discoverable in genuine substrate — e.g. "schedule total_sync daily" vs the locked total_sync-manual-only rule), the Tier-1 reply is the verbatim locked question PRECEDED by a brief, mandatory conflict-flag: for each conflict, one or two plain sentences stating the locked rule and explicitly declining the contradicting ask (no workflow examples, no scoping menu, no proposed objectives — those remain forbidden). Surfacing the conflict is REQUIRED, not optional: the Tier-1 anti-anchoring lock forbids workflow-anchoring exposition, it does NOT license silently complying-by-omission with a request that violates a locked rule. Emitting only the bare question while a recorded conflict goes unsurfaced is silent compliance and a hard defect (it lets the user re-answer believing the rule-violating shortcut is on the table). When \`context.conflicts\` is empty, the reply is exactly the bare verbatim question and nothing else, as above.
2. **Tier 2 check.** For each objective in context.objectives that has at least one gap in the gaps[] array, call the questmaster.SMARTItemClarification weapon with {context, objective, measurementCriteria}. Collect any returned questions.
3. Compose the partial objectives table as a markdown table: one row per objective, columns = \`#\`, \`name\`, \`input\`, \`method\`, \`output (asset · format)\`, \`viewResults\`. Use \`! need attention\` in cells the user hasn't filled. The \`output\` column is a compact summary of the output[] array (e.g. \`[genes] · json\`, \`[interactive_map_html] · raw HTML\`).
4. Compose user-facing questions: list them numbered, each clearly tied to which objective (by \`n\`) + which missing field. Tier 2 questions MUST be derived from gaps[]; do NOT invent new lines of inquiry the gaps array didn't surface. **When gaps[] includes the \`{missingField:"workflowComplete"}\` gap, the questions MUST cover EVERY element named in the context's \`undefinedTail\` — one targeted question per still-open element, not just the immediately-next one. A Tier-2 set that asks about a subset of the \`undefinedTail\` while silently dropping other named-open elements is under-scoped and incomplete (this is the exact defect: asking only about the next item the user signaled while the urgent repair / reporting / validation it implied go un-asked). The \`undefinedTail\` is the checklist; every item on it gets a question.** **Tier-2 conflict surface (MANDATORY, mirrors the Tier-1 conflict-flag carve-out — locked).** If \`context.conflicts\` is non-empty, the Tier-2 reply MUST, in addition to the partial table + tail questions, explicitly surface EVERY \`conflicts[]\` entry: state the conflict (the locked rule it violates / the user's own statement it contradicts / the deliberately-specified element it reverses) and decline-or-confirm rather than silently complying. A k3 deflection-scope-reversal is NOT treated as having resolved or removed any open item — the dropped element is surfaced as a reversal to confirm AND the genuine blockers are still re-asked; "ship it / don't overengineer" never collapses the plan or silently deletes a multi-turn deliberate design. Emitting a Tier-2 reply that silently honors a k1/k2/k3 conflict (or omits it) is a hard defect — the same silent-compliance failure the Tier-1 carve-out exists to stop, on the Tier-2 path.**

## Output schema (return exactly one JSON object)
{
  "questions": ["<question 1>", "<question 2>", "..."],
  "partialSmartTable": "<markdown table string>",
  "tier": 1 | 2
}

## Rules
- Output ONLY valid JSON. No prose outside the JSON object.
- Tier 1 returns exactly ONE generic question. Do not pre-itemize what the user hasn't said yet.
- Tier 2 questions must be specific (mention the objective number + the missing field).
- Do NOT silently fill in gaps yourself. Surface every gap as a question.
- The partial table has **one row per concrete objective in \`context.objectives\` only**. \`context.finaleCandidates\` (a "capture this as a skill book" ask) is NOT an objective and NEVER becomes a table row or a Tier-2 question — it is a post-success SE-finale signal, out of scope for the planning table.
- **Preserve \`[IMPORTANT DETAIL]\` markers verbatim in the partial table.** When compacting an objective's \`method\` into the table's method cell, never drop or shorten an \`[IMPORTANT DETAIL] …\` clause — carry the literal marker plus its verbatim constraint (the exact copy string, the exact endpoint + action) into the row unchanged. Collapsing it to "banner verbatim" / "as specified" loses the load-bearing detail and is a defect.
`,
    },

    deliverableTacticalEvaluation: {
      description:
        "Prompt template used by the questmaster.SMARTItemClarification weapon. Asks 3 questions per objective; returns ok or specific clarification questions.",
      howTo: `
You are evaluating ONE objective (from the new Description / Objectives / Deliverables model). Answer the three questions below in order; if any of them is unresolved, return clarification questions.

## The three questions

1. **Is there a clear input?** \`objective.input\` is "clear" only if it names a concrete source — a prior deliverable by name (\`[genes]\`, \`[hypoxia_pathway_image]\`), a URL, a table/column, a file path, or "I'll provide a list." Descriptive nouns ("biomarkers in the pathway", "the recommended antibody") are NOT clear unless the user said where they come from. If unclear → ask where the data comes from.

2. **Is there a clear method that converts input → output, AND is the placement of the output clear?**
   - **Method:** \`objective.method\` is "clear" only if the user named a concrete process (a tool, an algorithm, an existing workflow, a specific UI flow). High-level verbs alone — "overlay", "convert", "render", "generate", "select", "enrich" — are NOT clear unless the user said HOW. If unclear → ask which method (offer plausible candidates).
   - **Placement subtlety:** if the objective writes to a live website / existing page (e.g. an objective like "deploy to live site"), you MUST surface the question "override the existing page content, or create a new/separate page (e.g., a proving-grounds demo page)?" — never infer override-vs-new from context, even if the user mentioned the existing URL. The choice changes blast radius and rollback cost.

3. **Is the verification of this objective clear?** Each entry in \`objective.output[]\` should carry a \`verification\` field. The objective is "clear" only if every output entry has a concrete check (operator-browser screenshot vs. expectation, file lint, API round-trip, schema validation, etc.) **that the USER stated**. "Looks right" or silence is NOT clear. **If the objective's \`method\` carries a user-flagged \`[IMPORTANT DETAIL]\`, the verification of that non-negotiable MUST be user-stated — an agent-supplied "obvious" check does NOT make it clear; treat it as unresolved and ask.** Consistent with the \`evaluate\` firewall and the \`summarizeRelevantContext\` provenance rule: on an \`[IMPORTANT DETAIL]\`-bearing objective, a \`verification\` is resolved ONLY if it carries the literal token \`[USER-STATED]\` (set by summarizeRelevantContext from an explicit user statement). A non-empty \`verification\` WITHOUT \`[USER-STATED]\` on such an objective is an agent-synthesized check — treat it as UNRESOLVED exactly as if it were blank, and ask the user how the flagged non-negotiable is verified. If unclear → ask which verification gate applies to which output asset.

## Decision rule
- All three questions are resolved (either explicitly defined by the user OR clearly within agent's native capability) → return { "ok": true }.
- Any question unresolved → return { "ok": false, "questions": ["<question text 1>", "<question text 2>"] } with one question per unresolved item, clearly tied to the objective number + which of the three areas (input / method+placement / verification) is unresolved.

## Output schema
Return ONLY one JSON object matching one of the two shapes above. No prose, no fences.

## Rules
- Be honest about native capability. "Convert image to interactive map" is NOT a native capability — it needs an explicit method (OCR + overlay coordinates, etc.).
- "Agent can do this with CIC" is acceptable for verification/screenshot objectives.
- If the user has stated a specific tool/method, treat as resolved even if it's unusual — don't second-guess.
- An objective that produces multiple output assets needs verification for EACH asset — not just one umbrella check.
`,
    },

    upsertQuest: {
      description:
        "SA-dispatched action. Runs AFTER questmaster.evaluate has passed (smartJson available). Names the quest, searches for an existing match, creates a new one if no match. No orchestration-graph compute under the new model — objective ordering lives in objectives.n directly.",
      howTo: `
You are upsertQuest. The SMART context is complete (evaluate passed). Your job is to create a quest, or surface an existing match for the user to confirm.

## Inputs (provided in the dispatch prompt)
- smartJson: the SMART plan object from questmaster.evaluate (title, summary, mentions, objectives[], deliverables[])
- ownerId: the userId who owns this quest
- assignee (optional): the human-readable agent name to assign; defaults to "Not assigned" when omitted

## Procedure
1. Call the nameQuest action's procedure (below in the prompt) to produce a quest name following the convention: <action verb> <target object> <optional descriptors>.
2. Call questmaster.searchQuestBySimilarName weapon with { candidateName, smartPlan: smartJson, ownerId }. Get back { match, reason }. **The weapon call is MANDATORY and UNCONDITIONAL on every \`upsertQuest\` run (locked 2026-05-20 after SE-cursor-hand-off-test Step 4 attempt-1 defect).** Skipping the call on the grounds that "this is obviously a first-run / novel workflow / no plausible match" is the exact defect this rule prevents: \`match: null\` is the load-bearing AUDIT RECORD the rubric/judge requires in trace ("the lookup happened and returned no match"), distinct from "the lookup was never performed". The weapon-side double-verification (getActiveQuestNames + getQuestDetails for any candidate) is ALSO part of the audit even when the candidate list is empty. First-run obviousness is NOT a valid skip reason — call the weapon, record the \`{match:null, reason}\` return, then proceed.
3. If match !== null → return { ok: true, mode: "existing", questId: match, reason }. (First run won't hit this branch.)
4. If match === null → call questmaster.createQuest weapon with:
   - ownerId
   - title = the produced quest name
   - description = markdown rendering of the SMART plan summary, **STRATEGIC-ONLY** (load-bearing, locked 2026-05-15 after a real defect): the feature's intent in prose — WHAT the feature is and WHY the user wants it — and nothing else. The description **MUST NOT** enumerate, list, or parenthetically recap the per-objective mechanics or objective names: no \`(objective 1 …, objective 2 …, …)\`, no \`(does X, then Y, then Z)\` parenthetical, no inline copy of the workflow. The objectives (with their \`method\` text and any \`[IMPORTANT DETAIL]\`/\`[co-develop-with-user]\` tokens) live in \`quests.objectives\`, NOT in the description. Litmus test: if a clause names an objective or describes a mechanic ("then it triggers adoption via Chrome", "objective that imports to CJGEO"), it belongs in an objective's \`method\`, not the description — strip it. Keep the description to the strategic what+why a stakeholder reads to understand the goal without reading the workflow.
   - assignee = the input assignee (or "Not assigned" if absent)
   - objectives = smartJson.objectives verbatim. The weapon assigns \`n\` 1-based when missing and defaults \`status='pending'\`, \`view_results='No'\`. Pass through the \`output\` JSON array exactly as emitted by summarizeRelevantContext / evaluate.
   - deliverables = smartJson.deliverables (initial external inputs only). Each carries \`{name, kind, ref}\`. May be an empty array when planning surfaced no pre-existing inputs.
5. Return the createQuest result with mode: "new" + a one-line objectives summary the GM can surface in chat (e.g. "9 objectives: objective 1 OCR → … → objective 9 deploy + verify").

## nameQuest sub-procedure (inline LLM step)
Read the smartJson. Produce ONE quest name following convention: "<action verb> <target> [<descriptors>]". Examples: "Update Hypoxia Pathway Page", "Refactor login auth flow", "Migrate billing to Stripe". 4-10 words. Title case.

## Output schema (return exactly one JSON object)
{
  "ok": true | false,
  "mode": "new" | "existing",
  "questId": "<uuid>",
  "url": "/quest/<uuid>",
  "name": "<quest name>",
  "reason": "<optional one-line context>"
}

## Rules
- Output ONLY JSON. No prose.
- Never write to DB directly — always go through the questmaster weapons.
- **\`description\` is STRATEGIC-ONLY (locked).** It is the feature's intent (what + why) in prose; it must NOT enumerate or parenthetically list the per-objective mechanics or objective names. Per-objective detail lives in \`objectives[].method\`, never the description column. (See step 4 for the litmus test.)
- **Pass objective \`method\` text through to \`createQuest\` VERBATIM.** Any \`[IMPORTANT DETAIL]\` / \`[co-develop-with-user]\` literal token (and the constraint text it precedes) that \`summarizeRelevantContext\`/\`evaluate\` placed in an objective's \`method\` is forwarded to \`createQuest\` byte-for-byte — never stripped, normalized, reworded, "tidied", or relocated to \`notes\`. \`objectives = smartJson.objectives verbatim\` means the \`method\` strings are untouched. Dropping or altering a token here defeats the entire load-bearing marker chain and is a hard defect.
- If createQuest returns an error, surface it as { ok: false, error: "..." }; do NOT retry silently.
- Do NOT compute a quest-level orchestration graph. Objective ordering lives in the objectives.n column; runtime dependency is implicit in the input/output asset references between objectives.
- Do NOT populate deliverables[] with assets that downstream objectives will produce. Only assets that exist BEFORE execution starts (source URLs, handoff files) belong in the initial deliverables set.
`,
    },

    nameQuest: {
      description:
        "Sub-procedure used inline by upsertQuest. Produces a quest name from a SMART plan following the action-verb convention.",
      howTo: `
Read the SMART plan (title, summary, deliverables). Produce ONE quest name following the convention "<action verb> <target object> [<optional descriptors>]". 4-10 words. Title case. No quotes, no trailing punctuation.

Good examples:
- "Update Hypoxia Pathway Page"
- "Refactor Login Auth Flow"
- "Migrate Billing To Stripe"
- "Build Interactive Pathway Map Demo"

Bad examples (avoid):
- "the hypoxia thing" (no action verb, lowercase)
- "Hypoxia pathway page enhancements with interactive map and biomarker table" (too long, no action verb leading)

Return ONLY a JSON object: { "name": "<quest name>" }.
`,
    },

    searchQuestBySimilarName: {
      description:
        "Prompt template used by the questmaster.searchQuestBySimilarName weapon. Picks the best-matching active quest, or null.",
      howTo: `
You are matching a candidate quest name against a list of currently active quests. Decide if any existing quest is "the same objective" as the candidate.

## Inputs
- candidateName: the proposed name for the new quest
- smartPlan: the SMART plan summary (optional context, may be missing)
- activeQuests: list of {id, title, stage} for non-terminal quests

## Decision rule
- Two quests are "the same objective" only if they target the same artifact AND have substantially overlapping scope. "Update hypoxia page" and "Refactor hypoxia API" are NOT the same — different artifacts.
- If unsure, return null. False-positive matches are worse than false-negative because they conflate distinct work.

## Output schema (return exactly one JSON object)
{ "match": "<questId or null>", "reason": "<one-line rationale>" }

## Rules
- Output ONLY JSON. No prose.
- "match" must be either a real quest id from activeQuests OR the literal string null / the JSON null value. Never invent ids.
`,
    },

    dispatchRoledAgent: {
      description:
        "Wraps the questmaster.dispatchRoledAgent weapon. Composes role-aware substrate priming (CLAUDE.md + CLAUDE-<role>.md) for the dispatched sub-agent and delegates to claudecli. Use this when a dispatch needs the sub-agent to think as a specific role.",
      howTo: `
You are dispatching a sub-agent that must operate under a specific role. The weapon you call (questmaster.dispatchRoledAgent) reads the cross-role fundamentals from CLAUDE.md and, if present, the role-specific rules from CLAUDE-<role>.md, and prepends both to the sub-agent's prompt as substrate priming.

## Inputs (call site fills these)
- role: short role name (e.g. "GM", "DungeonMaster"). MUST match the casing of the CLAUDE-<role>.md file name. No path separators, no "..".
- prompt: the task prompt the sub-agent must execute. Pass INPUTS only — per CLAUDE.md's dispatch-prompt convention, do not prescribe procedure. The sub-agent's substrate priming (the role file + its skill book howTo) owns procedure.
- handshake (optional): the handshake string this dispatch protocol requires, if any.

## Role-file resolution
- The weapon composes the role file path as repoRoot/CLAUDE-<role>.md.
- If CLAUDE-<role>.md exists → it's read and included verbatim in the priming.
- If CLAUDE-<role>.md does NOT exist → priming includes only CLAUDE.md plus a note pointing the sub-agent to the Role Index. This is intentional: not every role has its own file; some roles (SA, MA) are owned by their skill book howTo or by the dispatch prompt itself.
- The role string MUST NOT contain path separators or "..". The weapon rejects such inputs to prevent path traversal.

## When NOT to use this
- For dispatching an SA whose entire procedure is owned by its skill book howTo and which does not need CLAUDE-level priming — use the normal Agent-tool dispatch with the skill book howTo reference.
- For weapon calls (no LLM) — call the weapon directly.

## Output
The weapon returns { ok, output?, rawOutput?, roleFileLoaded, error? }. Surface output to the caller; surface error as a State 1 question to the user if you cannot recover.
`,
    },
  },
  steps: [],
};

/**
 * @param {unknown} v
 * @returns {string}
 */
function deliverablesToText(v) {
  if (v == null) return "";
  if (Array.isArray(v)) return v.map((x) => String(x).trim()).filter(Boolean).join("\n");
  return String(v).trim();
}

/**
 * @param {unknown} raw
 * @returns {string | null}
 */
function parseDueDateIso(raw) {
  if (raw == null || raw === "") return null;
  const s = String(raw).trim();
  const t = Date.parse(s);
  if (Number.isNaN(t)) return null;
  return new Date(t).toISOString();
}

/**
 * @param {string} text
 * @returns {Record<string, unknown> | null}
 */
function extractJsonObject(text) {
  if (text == null || typeof text !== "string") return null;
  return extractQuestPlanJson(text) || tryBalancedJson(text);
}

/**
 * @param {string} s
 */
function tryBalancedJson(s) {
  const start = s.indexOf("{");
  if (start === -1) return null;
  for (let i = start; i < s.length; i++) {
    if (s[i] !== "{") continue;
    let depth = 0;
    for (let j = i; j < s.length; j++) {
      const ch = s[j];
      if (ch === "{") depth++;
      else if (ch === "}") {
        depth--;
        if (depth === 0) {
          try {
            const obj = JSON.parse(s.slice(i, j + 1));
            if (obj && typeof obj === "object" && !Array.isArray(obj)) return /** @type {Record<string, unknown>} */ (obj);
          } catch {
            return null;
          }
          return null;
        }
      }
    }
  }
  return null;
}

/**
 * Query AI with the user's initial request; return quest-compatible JSON and normalized fields.
 *
 * @param {string} userId
 * @param {{ initialRequest: string, client: import("@/libs/council/database/types.js").DatabaseClient }} opts
 */
export async function planRequestToQuest(userId, { initialRequest, client }) {
  const req = String(initialRequest ?? "").trim();
  if (req.length < 3) {
    console.warn(LOG, "reject: initial request too short", { len: req.length });
    return { data: null, error: new Error("Initial request is too short (need at least 3 characters).") };
  }

  console.log(LOG, "start", {
    userIdPrefix: typeof userId === "string" ? `${userId.slice(0, 8)}…` : String(userId),
    initialRequestLen: req.length,
    initialRequestPreview: previewText(req, 400),
    hasClient: Boolean(client),
  });

  const prompt = `The user submitted this request:

${req}

Respond with ONLY one JSON object (no prose) using exactly these keys:
{
  "title": "verb-first short title",
  "description": "SMART description of the work",
  "deliverables": "what must be submitted (string, or array of strings if multiple)",
  "due_date": "ISO 8601 datetime or clear date string",
  "stage": "plan"
}

"stage" must be "plan" unless the request is too vague — then use "idea".`;

  let aiText = "";
  try {
    console.log(LOG, "calling runDungeonMasterChat");
    const out = await runDungeonMasterChat({
      userId,
      messages: [{ role: "user", content: prompt }],
      model: undefined,
      client,
    });
    aiText = out.text;
    console.log(LOG, "model raw response", {
      responseChars: typeof aiText === "string" ? aiText.length : 0,
      rawPreview: previewText(aiText, 500),
    });
  } catch (err) {
    const e = err instanceof Error ? err : new Error(String(err));
    console.error(LOG, "runDungeonMasterChat failed", {
      message: e.message,
      name: e.name,
    });
    return { data: null, error: e };
  }

  const parsed = extractJsonObject(aiText);
  if (!parsed || typeof parsed.title !== "string" || String(parsed.title).trim() === "") {
    console.warn(LOG, "parse failed or missing title", {
      hasParsed: Boolean(parsed),
      titleType: parsed && typeof parsed.title,
      extractPreview: previewText(aiText, 600),
    });
    return { data: null, error: new Error("Model did not return valid quest JSON (title required).") };
  }

  const title = String(parsed.title).trim();
  const description = parsed.description != null ? String(parsed.description) : "";
  const deliverables = deliverablesToText(parsed.deliverables ?? parsed.deliverable);
  const dueRaw = parsed.due_date ?? parsed.dueDate;
  const due_date = parseDueDateIso(dueRaw);
  const stageRaw = parsed.stage != null ? String(parsed.stage).trim().toLowerCase() : "plan";
  const stage = stageRaw === "idea" || stageRaw === "plan" ? stageRaw : "plan";

  console.log(LOG, "normalized plan", {
    titleLen: title.length,
    titlePreview: previewText(title, 120),
    descriptionLen: description.length,
    deliverablesLen: deliverables.length,
    due_date,
    stage,
    dueRaw: dueRaw != null ? previewText(dueRaw, 80) : null,
  });

  return {
    data: {
      title,
      description,
      deliverables,
      due_date,
      stage,
      raw_model_text: aiText,
    },
    error: null,
  };
}

/**
 * Read quest fields + adventurer names/capabilities; pick one adventurer (AI-assisted).
 *
 * @param {string} userId
 * @param {{
 *   quest: { title?: unknown, description?: unknown, deliverables?: unknown },
 *   adventurers: Array<{ id: string, name: string, capabilities?: unknown, skill_books?: unknown }>,
 *   client: import("@/libs/council/database/types.js").DatabaseClient
 * }} opts
 */
export async function searchAdventurerForQuest(userId, { quest, adventurers, client }) {
  const roster = Array.isArray(adventurers) ? adventurers : [];
  if (roster.length === 0) {
    return { data: null, error: new Error("No adventurers on roster.") };
  }

  const qTitle = String(quest?.title ?? "").trim() || "(untitled)";
  const qDesc = String(quest?.description ?? "").trim();
  const qDel = deliverablesToText(quest?.deliverables);

  const brief = roster.map((a) => {
    const cap =
      typeof a.capabilities === "string"
        ? a.capabilities
        : a.capabilities != null
          ? JSON.stringify(a.capabilities)
          : "";
    const books = Array.isArray(a.skill_books) ? a.skill_books.join(", ") : "";
    return {
      adventurer_id: a.id,
      name: a.name,
      capabilities_plain: cap,
      skill_books: books,
    };
  });

  const prompt = `You match guild quests to adventurers.

Quest:
Title: ${qTitle}
Description: ${qDesc || "—"}
Deliverables: ${qDel || "—"}

Adventurers (pick exactly one by id):
${JSON.stringify(brief, null, 2)}

Reply with ONLY: {"adventurer_id":"<uuid>"} using one of the listed adventurer_id values.`;

  let aiText = "";
  try {
    const out = await runDungeonMasterChat({
      userId,
      messages: [{ role: "user", content: prompt }],
      model: undefined,
      client,
    });
    aiText = out.text;
  } catch (err) {
    return { data: null, error: err instanceof Error ? err : new Error(String(err)) };
  }

  const parsed = extractJsonObject(aiText);
  const id =
    parsed && typeof parsed.adventurer_id === "string"
      ? parsed.adventurer_id.trim()
      : parsed && typeof parsed.assignee_id === "string"
        ? parsed.assignee_id.trim()
        : "";

  const picked = id ? roster.find((a) => a.id === id) : null;
  if (!picked) {
    return {
      data: null,
      error: new Error('Model did not return a valid {"adventurer_id":"..."} for a listed adventurer.'),
    };
  }

  return { data: { adventurer_id: picked.id, name: picked.name }, error: null };
}

/**
 * @param {unknown} parsed
 * @returns {{ result: unknown, msg: string } | null}
 */
function normalizeSelectAdventurerConclusion(parsed) {
  if (!parsed || typeof parsed !== "object" || Array.isArray(parsed)) return null;
  const o = /** @type {Record<string, unknown>} */ (parsed);
  if (!("result" in o)) return null;
  const msg = o.msg != null ? String(o.msg).trim() : "";
  if (!msg) return null;
  return { result: o.result, msg };
}

function selectResultMeansNoMatch(result) {
  if (result === false) return true;
  if (result === "false") return true;
  if (typeof result === "string" && result.trim().toLowerCase() === "false") return true;
  return false;
}

/**
 * Loads roster (dynamic import to avoid skill_book ↔ adventurer cycle), runs AI pick.
 *
 * @param {string} userId
 * @param {{
 *   quest: Record<string, unknown>,
 *   client: import("@/libs/council/database/types.js").DatabaseClient
 * }} opts
 * @returns {Promise<{ data: { result: false | { id: string, name: string }, msg: string, meta: { prompt: string, modelText: string } } | null, error: Error | null }>}
 */
export async function selectAdventurer(userId, { quest, client }) {
  const questRow = quest && typeof quest === "object" ? quest : null;
  const questId = questRow?.id;
  const initialRequest =
    String(questRow?.description ?? "").trim() || String(questRow?.title ?? "").trim();

  if (!questId || typeof questId !== "string") {
    return { data: null, error: new Error("selectAdventurer: quest.id is required") };
  }

  if (initialRequest.length < 3) {
    return { data: null, error: new Error("selectAdventurer: request text is too short.") };
  }

  const { listAdventurers } = await import("@/libs/adventurer_runtime/server.js");
  const { data: roster, error: rosterErr } = await listAdventurers(userId, { client });
  if (rosterErr) {
    return { data: null, error: rosterErr };
  }

  const rosterArr = Array.isArray(roster) ? roster : [];

  // Build brief with both static capabilities (abstract) and dynamic boast (specific).
  // Capabilities describes the type of work the adventurer handles.
  // Boast lists the exact skill book actions available — the binding contract.
  const { buildBoast } = await import("@/libs/adventurer/index.js");
  const brief = await Promise.all(rosterArr.map(async (a) => ({
    id: a.id,
    name: a.name,
    backstory: typeof a.backstory === "string" ? a.backstory : "",
    capabilities:
      typeof a.capabilities === "string"
        ? a.capabilities
        : a.capabilities != null
          ? JSON.stringify(a.capabilities)
          : "",
    boast: await buildBoast(a),
  })));

  const prompt = `You match a NEW guild request to at most one adventurer on the roster.

User request (raw):
${initialRequest}

Roster — each entry has:
  "id" / "name" — identifiers,
  "backstory" — the adventurer's intended purpose in a sentence or two; THIS IS THE PRIMARY SIGNAL for fit,
  "capabilities" — high-level description of work they handle,
  "boast" — exact list of skill book actions available to them.

Read backstory FIRST. The backstory tells you what the adventurer is FOR. Capabilities and boast are about HOW they do it. A task that fits an adventurer's backstory is a good match even if the boast is generic; a task that mismatches the backstory is a poor match even if the boast technically covers it.

${JSON.stringify(brief, null, 2)}

Reply with ONLY one JSON object (no markdown, no prose):

If one adventurer is a good match:
{
  "result": { "id": "<uuid from roster>", "name": "<exact name from roster>" },
  "msg": "<brief rationale>"
}

If NONE are a good match:
{
  "result": false,
  "msg": "<brief rationale why no one fits>"
}

IMPORTANT — triage priority (apply FIRST, before matching by boast):
1. Ask: "Can this task be done with natural language alone — no specialized API access, no database changes, no codebase modifications?"
2. If YES or UNSURE → pick the general-purpose agent. Most tasks (writing, research, reports, analysis, screenshots, creative work) fall here.
3. ONLY pick a specialized adventurer if the task CLEARLY requires their specific weapon (e.g. Zoho CRM data, forging new code weapons, Stripe billing).
4. Blacksmith/Runesmith are ONLY for creating new weapons and skill books — NOT for general tasks. Do not pick them for writing, research, or content.
5. If a quest returns to assign with a comment saying specialized access is needed, read the comments and re-route accordingly.

Rules:
- "msg" must be a non-empty string.
- "result" is either an object with both "id" and "name" from the roster, or boolean false (or the string "false").
- Match by what the actions DO (read their descriptions), not by action names.`;

  if (rosterArr.length === 0) {
    const msg = "No adventurers on roster; cannot assign without recruitment.";
    return {
      data: {
        result: false,
        msg,
        meta: { prompt, modelText: "" },
      },
      error: null,
    };
  }

  let aiText = "";
  try {
    const out = await runDungeonMasterChat({
      userId,
      messages: [{ role: "user", content: prompt }],
      model: undefined,
      client,
    });
    aiText = out.text;
  } catch (err) {
    return { data: null, error: err instanceof Error ? err : new Error(String(err)) };
  }

  const parsed = extractJsonObject(aiText);
  const conclusion = normalizeSelectAdventurerConclusion(parsed);

  if (!conclusion) {
    return {
      data: null,
      error: new Error(
        'selectAdventurer: model did not return valid JSON with "result" and non-empty "msg".',
      ),
    };
  }

  const meta = { prompt, modelText: aiText };

  if (selectResultMeansNoMatch(conclusion.result)) {
    return {
      data: { result: false, msg: conclusion.msg, meta },
      error: null,
    };
  }

  const resObj =
    conclusion.result && typeof conclusion.result === "object" && !Array.isArray(conclusion.result)
      ? /** @type {Record<string, unknown>} */ (conclusion.result)
      : null;
  const id =
    resObj && typeof resObj.id === "string"
      ? resObj.id.trim()
      : resObj && typeof resObj.adventurer_id === "string"
        ? resObj.adventurer_id.trim()
        : "";

  const picked = id ? rosterArr.find((a) => a.id === id) : null;
  if (!picked) {
    return {
      data: null,
      error: new Error("selectAdventurer: model id is not on the roster."),
    };
  }

  const nameFromModel = resObj && typeof resObj.name === "string" ? resObj.name.trim() : "";
  let msg = conclusion.msg;
  if (nameFromModel && nameFromModel !== picked.name) {
    msg = `${conclusion.msg} (model name "${nameFromModel}" normalized to roster name "${picked.name}")`;
  }

  return {
    data: {
      result: { id: picked.id, name: picked.name },
      msg,
      meta,
    },
    error: null,
  };
}

/**
 * @param {string} userId
 * @param {{
 *   initialRequest: string,
 *   adventurerName: string,
 *   client: import("@/libs/council/database/types.js").DatabaseClient
 * }} opts
 */
export async function interpretIdea(userId, { initialRequest, adventurerName, client }) {
  const req = String(initialRequest ?? "").trim();
  const adv = String(adventurerName ?? "").trim();
  if (req.length < 3) {
    return { data: null, error: new Error("Initial request is too short.") };
  }
  if (!adv) {
    return { data: null, error: new Error("Adventurer name is required for interpretIdea.") };
  }

  const prompt = `Evaluate this user request and decide if it is clear enough to execute.

User request:
${req}

Check two things:
1. Is the request clear enough that someone could act on it without asking questions?
2. Can you deduce when the task is considered done? (explicit or obvious completion criteria)

If BOTH pass, respond with:
{"clear":true,"title":"<short action-verb title, e.g. Write a poem about spring in Beijing>"}

If EITHER fails, respond with:
{"clear":false,"question":"<one specific clarification question to ask the user>"}

Rules:
- title MUST start with an action verb.
- Do NOT rewrite the user's request. The original description will be kept as-is.
- Be lenient — if the task is obviously completable (e.g. "write a poem about X"), it is clear. Only flag truly ambiguous requests.`;

  let aiText = "";
  try {
    const out = await runDungeonMasterChat({
      userId,
      messages: [{ role: "user", content: prompt }],
      model: undefined,
      client,
    });
    aiText = out.text;
  } catch (err) {
    return { data: null, error: err instanceof Error ? err : new Error(String(err)) };
  }

  const parsed = extractJsonObject(aiText);
  if (!parsed) {
    return { data: null, error: new Error("Model did not return valid JSON for interpretIdea.") };
  }

  if (parsed.clear === false) {
    // Request is unclear — return question for clarification
    return {
      data: {
        clear: false,
        question: String(parsed.question || "Could you clarify what you'd like done?"),
        raw_model_text: aiText,
      },
      error: null,
    };
  }

  // Request is clear — return title only, description stays as-is
  const title = typeof parsed.title === "string" ? parsed.title.trim() : "";
  if (!title) {
    return { data: null, error: new Error("interpretIdea: model returned clear=true but no title.") };
  }

  return {
    data: {
      clear: true,
      title,
      description: req, // keep original user request as description
      raw_model_text: aiText,
    },
    error: null,
  };
}

/**
 * Assign stage action — uses the quest from pipeline context (guildos.quest),
 * scans adventurer roster via AI, writes the assignment back to the quest.
 * Returns full debug trace (prompt, model response, quest context).
 *
 * @param {string} userId
 * @param {{
 *   questId?: string,
 *   guildos?: { quest?: Record<string, unknown> },
 *   client: import("@/libs/council/database/types.js").DatabaseClient
 * }} opts
 */
export async function assign(userId, { questId, guildos, client }) {
  // Quest comes from pipeline context (loaded by the trigger), or fall back to questId
  let questRow = guildos?.quest ?? null;
  if (!questRow && questId) {
    const { getQuestForOwner } = await import("@/libs/quest");
    const { data, error } = await getQuestForOwner(questId, userId, { client });
    if (error || !data) return { data: null, error: error || new Error("Quest not found.") };
    questRow = data;
  }
  if (!questRow || !questRow.id) {
    return { data: null, error: new Error("assign: quest is required (via pipeline context or questId)") };
  }

  // Run selectAdventurer — AI scans the roster
  const result = await selectAdventurer(userId, { quest: questRow, client });
  if (result.error) {
    return {
      data: {
        result: false,
        msg: result.error.message,
        meta: { quest: { id: questRow.id, title: questRow.title, stage: questRow.stage } },
      },
      error: null,
    };
  }

  const questMeta = { id: questRow.id, title: questRow.title, description: questRow.description, stage: questRow.stage };

  // If a match was found, write the assignment and advance to plan
  if (result.data && result.data.result && result.data.result !== false) {
    const match = result.data.result;
    const assignedTo = match.name || "";

    // Use quest module's resolveAssignee to confirm NPC vs adventurer
    const { resolveAssignee, updateQuest } = await import("@/libs/quest");
    const resolved = await resolveAssignee(assignedTo, client);

    const { updateQuestAssignee } = await import("@/libs/council/database/serverQuest.js");
    await updateQuestAssignee(questRow.id, {
      assigneeId: resolved.type === "adventurer" ? resolved.profile.id : null,
      assignedTo,
    }, { client });

    // Advance directly to execute — planning is done inline by the executing LLM
    await updateQuest(questRow.id, { stage: "execute" }, { client });

    return {
      data: {
        result: match,
        msg: result.data.msg,
        assigneeType: resolved.type,
        resolved: resolved.profile,
        meta: { ...result.data.meta, quest: questMeta },
      },
      error: null,
    };
  }

  // No match — trigger preparation cascade
  const { triggerPreparationCascade } = await import("@/libs/quest");
  const cascadeResult = await triggerPreparationCascade(questRow, { client });

  return {
    data: {
      result: false,
      msg: result.data.msg,
      assigneeType: "none",
      cascade: cascadeResult.error
        ? { ok: false, error: cascadeResult.error.message }
        : { ok: true, ...cascadeResult.data },
      meta: { ...result.data.meta, quest: questMeta },
    },
    error: null,
  };
}

// ---------------------------------------------------------------------------
// runReviewLoop — script-locked end-to-end purrview review for one quest.
// ---------------------------------------------------------------------------
//
// Why this exists: Cat's prior sessions kept inventing their own review
// loops, posting verdicts without invoking the vision judge (judge_origin
// null), and drifting into infra PRs when the queue was empty. The fix
// is to remove the choice — Cat calls this one function, which orchestrates
// the entire workflow against the script-locked gate weapons.
//
// One call per quest:
//   1. questPurrview.confirmSubmission → verify worker handoff lockphrase.
//   2. For each deliverable: route to a vision judge based on URL extension:
//      - image (png/jpg/webp/gif) → openai_images.judge (gpt-4o, mandatory).
//      - non-image (text/json/doc) → returned as 'manual_required' verdict;
//        the loop exits without writing a stage transition so Cat (or a
//        human) handles the non-image case via the existing reviewSubmission
//        howTo. Future work: add Claude-side text/JSON judging for full
//        automation here.
//   3. If every judge returns 'match' → questPurrview.approve.
//   4. If any judge returns 'mismatch' or 'inconclusive' → questPurrview.bounce
//      with a structured reason listing each failing item_key + judge
//      rationale.
//
// Returns one of:
//   { ok: true, action: 'approved' | 'bounced', stage, items_count, perItemVerdicts }
//   { ok: false, action: 'manual_required', failed_at, ... }
//
// @param {{ questId: string }} input
// @param {string} [userId] — owner id for credential-scoped weapon calls
async function runReviewLoop({ questId } = {}, userId) {
  if (typeof questId !== "string" || !questId.trim()) {
    return { ok: false, action: "input_invalid", reason: "questId required (non-empty string)" };
  }

  // Lazy-import the gate weapons + judge so the questmaster module doesn't
  // pull them in for non-review code paths (planRequestToQuest etc.).
  // These modules have NO default export — destructure the namespace directly.
  const { confirmSubmission, approve, bounce } = await import("@/libs/weapon/questPurrview/index.js");
  const { judge } = await import("@/libs/weapon/openai_images/index.js");

  // Step 1 — confirm worker handoff
  const confirm = await confirmSubmission({ questId });
  if (!confirm.ok) {
    return {
      ok: false,
      action: "confirm_failed",
      failed_at: "confirmSubmission",
      reason: confirm.reason,
      msg: confirm.msg,
    };
  }
  const deliverables = confirm.deliverables || [];
  // The acceptance-criteria source for the judge claim. confirmSubmission now
  // returns the quest title + description; feeding them into the claim is what
  // makes the judge verify "satisfies THIS brief" instead of "an image exists"
  // (the deliverable-name-only claim was the content-blind leak).
  const { buildJudgeClaim } = await import("@/libs/weapon/questPurrview/index.js");
  const questTitle = confirm.title || "";
  const questDescription = confirm.description || "";
  if (deliverables.length === 0) {
    return {
      ok: false,
      action: "no_deliverables",
      failed_at: "confirmSubmission",
      reason: "Quest has no deliverables rows after confirmSubmission. Cannot review an empty handoff.",
    };
  }

  // Step 2 — judge per deliverable
  const IMAGE_EXT = /\.(png|jpe?g|gif|webp)(\?|#|$)/i;
  const verdicts = [];
  let manualRequired = false;
  for (const deliverable of deliverables) {
    const url = deliverable.ref || "";
    // The judge claim now carries the quest's acceptance criteria (title +
    // description), not just the deliverable name — so a rendered-but-wrong
    // artifact (wrong style/language, missing/forbidden story elements) is
    // judged 'mismatch' instead of waved through. Source = quest.description
    // (the operator-chosen acceptance-criteria source; no per-deliverable
    // expectation column exists post-migration).
    const claim = buildJudgeClaim({
      deliverableName: deliverable.name,
      kind: deliverable.kind,
      questTitle,
      questDescription,
    });
    if (!IMAGE_EXT.test(url)) {
      // Non-image: bail out to manual review (text/json/doc judging not yet
      // automated in this loop). Cat or a human handles via reviewSubmission
      // howTo.
      manualRequired = true;
      verdicts.push({
        item_key: deliverable.name,
        verdict: "manual",
        text: `Non-image artifact at ${url}; runReviewLoop only auto-judges images. Defer to manual reviewSubmission flow.`,
        judge_origin: null,
      });
      continue;
    }
    let judgeRes;
    try {
      judgeRes = await judge({ imageUrl: url, claim }, userId);
    } catch (err) {
      verdicts.push({
        item_key: deliverable.name,
        verdict: "fail",
        text: `Judge call failed: ${err.message}`,
        judge_origin: "openai_images.judge",
        judge_rationale: `Error: ${err.message}`,
      });
      continue;
    }
    const passOrFail = judgeRes.verdict === "match" ? "pass" : "fail";
    verdicts.push({
      item_key: deliverable.name,
      verdict: passOrFail,
      text:
        passOrFail === "pass"
          ? `Judge confirmed match (confidence ${judgeRes.confidence}): ${judgeRes.reasoning}`
          : `Judge flagged ${judgeRes.verdict} (confidence ${judgeRes.confidence}): ${judgeRes.reasoning}`,
      judge_origin: `openai_images.judge:${judgeRes.model}`,
      judge_model: judgeRes.model,
      judge_rationale: judgeRes.reasoning,
    });
  }

  if (manualRequired) {
    return {
      ok: false,
      action: "manual_required",
      failed_at: "judge",
      reason: "One or more deliverables are non-image artifacts; runReviewLoop only auto-judges images. Use the reviewSubmission howTo for text/JSON/doc deliverables.",
      verdicts,
    };
  }

  // Step 3 — decide approve vs bounce
  const fails = verdicts.filter((v) => v.verdict === "fail");
  if (fails.length === 0) {
    const result = await approve({
      questId,
      perItemVerdicts: verdicts,
      summary: `runReviewLoop auto-approved: ${verdicts.length}/${verdicts.length} deliverables judged 'match' by gpt-4o.`,
    });
    if (!result.ok) {
      return { ok: false, action: "approve_failed", failed_at: "approve", reason: result.failed?.[0], report: result.report, verdicts };
    }
    return {
      ok: true,
      action: "approved",
      stage: result.stage,
      items_count: result.items_count,
      perItemVerdicts: verdicts,
    };
  }

  // Bounce path
  const failingKeys = fails.map((v) => `${v.item_key}: ${v.judge_rationale || v.text}`).join("; ");
  const reason = `runReviewLoop auto-bounced: ${fails.length}/${verdicts.length} deliverables failed gpt-4o judge. Failing deliverables — ${failingKeys}. Worker should re-capture these artifacts so they actually demonstrate the expectation, then resubmit with the same item_keys (UPSERT replaces in place).`;
  const result = await bounce({ questId, perItemVerdicts: verdicts, reason });
  if (!result.ok) {
    return { ok: false, action: "bounce_failed", failed_at: "bounce", reason: result.failed?.[0], report: result.report, verdicts };
  }
  return {
    ok: true,
    action: "bounced",
    stage: result.stage,
    items_count: result.items_count,
    failing_item_keys: fails.map((v) => v.item_key),
    perItemVerdicts: verdicts,
  };
}

const questmaster = {
  definition,
  planRequestToQuest,
  searchAdventurerForQuest,
  selectAdventurer,
  interpretIdea,
  assign,
  runReviewLoop,
};

export { runReviewLoop };
export default questmaster;
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/questmaster/closingArchive.js -->
```js
/**
 * closingArchive.js — the pure decision half of archiveQuestTakeaways (SIO #11).
 *
 * Split out of the giant questmaster/index.js so the closing→escalate-vs-close
 * fork is unit-testable in isolation (index.js itself can't be imported by Vite's
 * test transform).
 *
 * SIO #11 — routine quests at closing AUTO-ARCHIVE (closeQuest: closing→complete);
 * escalate ONLY on a *genuine decision fork*, defined conservatively (see the ED
 * in engineering_decisions.md):
 *
 *   A genuine decision fork = closing requires the OWNER to make an
 *   irreversible / externally-visible choice the system cannot safely default:
 *     1. a caller-supplied `pendingDecisions[]` entry (the archive leaf itself
 *        identified an owner-only choice), OR
 *     2. an attempted irreversible EXTERNAL side-effect FAILED and cannot be
 *        safely auto-retried (e.g. the Asana completion post threw).
 *
 *   NOT a fork (carries a safe default → auto-archive):
 *     - no external archive target (routine internal quest → complete);
 *     - unresolved self-improvement finale comments (uncommitted ED/insight
 *       offers) — optional enhancements; left open for separate action, they do
 *       NOT gate the quest's completion;
 *     - a missing completion report (auto-composed from the deliverables).
 */

/**
 * @param {{
 *   pendingDecisions?: string[],   // caller-identified owner-only choices
 *   asanaPostFailed?: boolean,     // an attempted external post threw
 *   asanaFailReason?: string,
 *   openMethodOfferCount?: number, // unresolved action='closing' finale comments
 * }} args
 * @returns {{ escalate: boolean, decisions: string[], openMethodOfferCount: number }}
 */
export function classifyClosingArchive({
  pendingDecisions = [],
  asanaPostFailed = false,
  asanaFailReason = "",
  openMethodOfferCount = 0,
} = {}) {
  const decisions = Array.isArray(pendingDecisions) ? pendingDecisions.filter((d) => String(d || "").trim()) : [];
  const out = [...decisions];
  // Fork #2: an irreversible external side-effect actually failed → a human must
  // fix the token / target and retry. This is NOT the same as "no target" (which
  // is routine); it is a real attempted-and-failed external op.
  if (asanaPostFailed) {
    out.push(
      `Auto-archive to the source Asana task FAILED${asanaFailReason ? ` (${asanaFailReason})` : ""} — archive the results manually or fix the Asana token, then retry.`,
    );
  }
  return {
    escalate: out.length > 0,
    decisions: out,
    // Surfaced so the caller can NOTE (not escalate) how many self-improvement
    // offers remain open when it auto-closes.
    openMethodOfferCount: Math.max(0, Number(openMethodOfferCount) || 0),
  };
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/questmaster/closingArchive.test.js -->
```js
import { describe, it, expect } from "vitest";
import { classifyClosingArchive } from "./closingArchive.js";

describe("SIO #11 — classifyClosingArchive (escalate only on a genuine decision fork)", () => {
  it("ROUTINE: no target, open self-improvement offers → auto-archive (no escalate)", () => {
    const r = classifyClosingArchive({ pendingDecisions: [], asanaPostFailed: false, openMethodOfferCount: 3 });
    expect(r.escalate).toBe(false);
    expect(r.decisions).toEqual([]);
    expect(r.openMethodOfferCount).toBe(3); // preserved so the caller can NOTE, not gate
  });

  it("ROUTINE: nothing pending at all → auto-archive", () => {
    expect(classifyClosingArchive({}).escalate).toBe(false);
  });

  it("FORK #1: a caller-supplied pendingDecisions entry → escalate", () => {
    const r = classifyClosingArchive({ pendingDecisions: ["Email vs Slack archive target — which?"] });
    expect(r.escalate).toBe(true);
    expect(r.decisions).toContain("Email vs Slack archive target — which?");
  });

  it("FORK #2: a FAILED external side-effect (Asana post threw) → escalate", () => {
    const r = classifyClosingArchive({ asanaPostFailed: true, asanaFailReason: "401 invalid token" });
    expect(r.escalate).toBe(true);
    expect(r.decisions.some((d) => /FAILED/.test(d) && /401 invalid token/.test(d))).toBe(true);
  });

  it("both forks accumulate into the decisions list", () => {
    const r = classifyClosingArchive({ pendingDecisions: ["owner call A"], asanaPostFailed: true });
    expect(r.escalate).toBe(true);
    expect(r.decisions.length).toBe(2);
  });

  it("blank/whitespace pendingDecisions entries are ignored (no phantom escalation)", () => {
    const r = classifyClosingArchive({ pendingDecisions: ["", "   ", null] });
    expect(r.escalate).toBe(false);
    expect(r.decisions).toEqual([]);
  });

  it("a missing report / no Asana target is NOT represented as a decision here (routine)", () => {
    // The caller only passes asanaPostFailed for a real failed post; 'no target'
    // and 'no report' never reach this helper as decisions.
    const r = classifyClosingArchive({ pendingDecisions: [], asanaPostFailed: false });
    expect(r.escalate).toBe(false);
  });
});
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/questmaster/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/weapon/questmaster`

Rationale layer (per CLAUDE.md §6). The WHY behind the quest stage machine and
planner. Append; supersede in place when a decision changes.

---

## 2026-07-03 — questTick dispatches leaves by the assignee's worker RUNTIME, but claude keeps the claudecli launcher

**Decision.** All three `questTick` leaf-dispatch sites (plan / execute /
purrview) route through a `dispatchLeaf` helper that branches on the assignee's
`worker_type` (resolved ONCE per tick — every select is `.eq("assignee",
assignee)`, so the runtime is uniform): `claude` workers keep the direct
`claudecliInvoke` launcher; every other worker_type (`hermes`, `codex`, …) goes
through `invokeAdventurerLeaf(prompt, assignee, …)`, which renders the
adventurer's own `system_prompt` start-command template. This is what makes the
manual "Trigger tick" button honor the quest's guild (GuildOS = claude, Hermes =
codex-runtime) instead of always spawning a Claude leaf.

**Rejected alternative A** — route EVERYTHING (including claude) through
`invokeAdventurerLeaf`, deleting the branch. Cleaner-looking, but it silently
drops `--mcp-config`: only `claudecliInvoke` threads `mcpConfigPath`, and the
claude `system_prompt` template has no mcp-config token. That would break
`[BROWSER]` leaves (headless Playwright injection) for every GuildOS quest.

**Rejected alternative B** — leave all three sites hardcoded to `claudecliInvoke`
(the original). Correct for claude, but a Hermes-assigned quest triggered
manually then runs a *Claude* leaf, not codex — the guild is ignored.

**Why.** The caravan cron already dispatches via `invokeAdventurerLeaf`
(worker-type-agnostic); the manual tick was the one path still hardcoded to
claude. Converging on the same launcher — while preserving the claude-only
browser path — is the minimal fix that (a) fixes Hermes routing and (b) keeps
`[BROWSER]` GuildOS quests working. Scope is intentionally limited to the two
live guilds (GuildOS + Hermes); Carbon was explicitly dropped.

**Consequence if violated.** Collapse the branch → lose `[BROWSER]` support;
revert the branch → the button never triggers a non-claude guild.

---

## 2026-06-13 — a plan with open unblocking questions ESCALATES; it must not enter execute

**Decision.** After `planQuest` itemizes, it transitions to `escalated` (via
`moveQuestToEscalated`) when any unblocking question is still open
(`qtype='unblocking', answered=false`), not to execute. `moveQuestToExecute`
additionally hard-blocks entry to execute while any open unblocking question
exists (defense-in-depth: no caller can bypass it). A re-plan after the user
answers (questions `answered=true`) proceeds normally.

**Rejected alternative.** Always call `moveQuestToExecute` after planning (the
original) — which only gated on objectives/deliverables/assignee, never on
questions.

---

## 2026-06-15 — raising an unblocking question goes through one named procedure (`raiseUnblockingQuestionDiplomatically`)

> Naming note: this started as the rule `ruleUnblockingQuestionBar` and was
> consolidated the same day into the action `raiseUnblockingQuestionDiplomatically`
> (questmaster skill book) — an action you INVOKE when you think you need to ask
> the user, rather than a passive bar. Same two requirements, now packaged as the
> single procedure every producer points to.

**Decision.** An unblocking question may be written ONLY by following
`raiseUnblockingQuestionDiplomatically`, referenced at every authoring point (the
`planQuest` planning prompt, the `buildLeafExecutePrompt` runtime prompt step 5,
and the skill-book `workScopePerTick` #1 / `ruleAcceptableStopState` #2):
- **Step 1 — pass `sage.askSage` FIRST:** the question is written only if the gate
  returns ESCALATE (irreversible/business-damaging OR user-only knowledge).
  Reversible defaults, self-answerable facts, and self-proposed answers return
  PROCEED → the agent decides and records, never asks. The askSage pass is a
  precondition of the row existing, not an afterthought.
- **Step 2 — required body:** (1) WHAT the user needs to do (concrete, with options
  + recommended default) and (2) WHY the agent can't do it (the irreversibility /
  user-only knowledge).

**Rejected alternative.** Keep the prior asymmetry — the planner had Gate A but no
Gate B, and the worker path (`workScopePerTick`/`ruleAcceptableStopState`) had
*neither* ("if genuinely blocked on a user decision, create the question"), with
"genuinely" left undefined and no required shape.

**Why.** Audit of the 3 escalated quests (2026-06-15) found most "unblocking"
questions were **manufactured stops**: the RAG-primitive quest had 4 questions
that were all deferred design defaults / self-answerable facts (e.g. "which
corpora?", "BM25 or embeddings (and are embeddings available)?", "what's the
snippet structure?" — the agent had even proposed the structure inside the
question). None named a reason the *agent* couldn't decide, because there wasn't
one. These stall quests for no reason and push the agent's own design work onto
the user. The bar is `sage.askSage` (the chat-side question gate) applied to the
quest substrate — the same discipline, previously absent from question *generation*.

**Consequence if violated.** Quests pile up bare "Which X?" questions and sit in
`escalated` waiting on decisions the agent should have made; the user is forced to
adjudicate reversible defaults and answer factual questions the agent could look
up — the exact low-value-interruption failure the question gate exists to prevent.

**Future hardening (not yet done).** Gate B is prompt-enforced on free text. A
stronger form would split `quest_questions.body` into structured
`why_agent_blocked` + `user_action_required` columns so a missing half is a schema
error, not a style miss — deferred (schema + UI + call-site churn).

**Why.** Executing past unresolved decisions produces the wrong work. A quest that
genuinely needs user input must rest in `escalated` with the question, not march
into execute. (`escalated` + an open unblocking question is the caravan's valid
resting state.)

**Consequence if violated.** Quests silently execute on unanswered decisions — the
exact behavior the operator caught (born→plan→execute with 5 open questions).

---

## 2026-06-13 — the planner mints a question ONLY for irreversible decisions; delegated/reversible → pick a default

**Decision.** `planQuest`'s question-generation prompt applies the `sage.askSage`
test: drop any candidate question that is (a) explicitly delegated by the user
("agent's discretion", a stated range like "5–10 chapters"), (b)
discoverable/inferable from the description, or (c) reversible/low-stakes
(layout, palette, styling, count-within-range). Only a costly-and-irreversible
unknown earns a question.

**Rejected alternative.** "Ask about any ambiguity" — which surfaced reversible
and already-delegated choices as questions.

**Why.** The planner must honor the same question discipline as the rest of
GuildOS. A user who said "5–10 chapters, your discretion" must not be asked "5 or
10?". Capture of those grants happens upstream at createQuest (see
`libs/skill_book/questmaster/engineering_decisions.md`); the planner is the
downstream safety net.

**Consequence if violated.** The user is re-asked decisions they already made →
spurious escalations, wasted round-trips.

---

## 2026-06-13 — `moveQuestToReview` delegates to `questExecution.submit`; it never re-implements the gate

**Decision.** `moveQuestToReview` does not duplicate the execute→purrview gate
logic — it calls `questExecution.submit({ questId })` internally (the canonical
gate) and only adds a soft-gate side effect around it.

**Rejected alternative.** Re-implement the objectives/deliverables checks inline, or
call `moveQuestToExecute` directly.

**Why.** The execute→purrview gate is the one canonical stage boundary. A second
implementation drifts from the first over time, so the two disagree and the boundary
becomes inconsistent. Delegation keeps a single gate.

**Consequence if violated.** Two gate implementations diverge — one passes a quest
the other would reject; the stage boundary stops being deterministic.

---

## 2026-06-13 — composite deliverables with no `quest_assets` get a SOFT warning, not a hard block

**Decision.** `moveQuestToReview` checks for composite deliverables (pdf/html) that
have zero `quest_assets` and, if found, writes a soft-gate **warning** comment — it
does NOT block the transition (`ruleCheckpointAssets`).

**Rejected alternative.** Hard-block review entry without assets, OR ignore the gap
entirely.

**Why.** Many quests are single-step (text→text) and legitimately have no
intermediates — a hard block would kill them. But composite deliverables usually
depend on intermediate artifacts (images, chapters), and a missing checkpoint forces
wasteful regeneration later. A warning surfaces the gap for the reviewer without a
false-positive block.

**Consequence if violated.** Either composite deliverables sail to review with no
checkpoint record (forcing re-generation), or a hard block kills legitimate
single-step quests.

---

**2026-06-13 · Derailment-rescue state lives in a `derail_probe` comment, not a new table.**
`detectDerailedQuests` needs cross-tick memory (is the progress fingerprint the
same as last tick?) to detect flatline/thrash. The native move would be a new
`quest_derail_state` table (or a column on `quests`). **Rejected** — chose to
carry the fingerprint+run-count in a self-written `source='system',
action='derail_probe'` quest_comments row, deleted-and-reinserted each tick.
**Why:** detection must be purely additive substrate (no migration coupling, no
schema surface to keep in sync with the detector heuristics, which are expected
to retune often per §7.5); the comments table already exists, is RLS-scoped to
the quest, and a probe row is naturally co-located with the quest's history.
**Consequence if violated:** moving this to a table re-introduces the migration
churn the heuristic-tuning is meant to avoid, and splits derail state away from
the quest thread a human reads. The probe is bookkeeping — it is filtered out of
the oscillation scan (`.neq("action","derail_probe")`) so it can't feed itself.

**2026-06-13 · `parked` is a distinct terminal, NOT a flavor of `escalated`.**
A derailed-but-unsalvageable quest has no clean user question. Native instinct is
to dump it in `escalated` with a reason flag. **Rejected** — added a separate
`parked` stage (migration 20260613130000). **Why:** `escalated` is the user's
ACTIONABLE inbox ("answer this question"); mixing in "genuinely stuck, nobody to
ask" rots that inbox (the escalationRot detector exists precisely because stale
escalated quests are already a problem). **Consequence if violated:** the triage
tray and the answer-this-question queue collapse into one list and the user
loses the signal of which quests actually need a decision from them.

---

**2026-06-14 · questTick's planner ADOPTED orphaned plan-stage quests (SUPERSEDED 2026-06-14).**
~~The planner's `.or()` filter claimed orphans on pickup.~~ **Superseded by**
`assignOrphans()` in `libs/caravan/index.js` — a standalone mutation at the front
of the rescue flow that covers ALL non-terminal stages, not just plan. The
planner now filters `.eq("assignee", assignee)` only; orphan adoption is the
caravan rescue's job. See `libs/caravan/engineering_decisions.md` (2026-06-14
entries) for the full rationale.

---

## 2026-06-16 — the digest is a CURATED worker read-surface, not the comment thread; compaction-by-deletion is rejected

**Decision.** `summarizeQuest` projects `quest_comments` into a worker-only
WORKING SET: unresolved `source='user'` directives (all), the single latest
worker/leaf/chaperon hand-off, open verification fails, and the earliest
`chaperon/deliver` origin — while `source='system'` telemetry (derail_kickback,
lease, unstuck, `*_digested`) is COUNTED but never quoted. Classification is on
the existing `source`/`action` (no schema change). The full thread stays the
durable, immutable audit log (`getQuestDetails` returns every row); the digest
filters noise OUT OF THE READ, never out of the record.

**Rejected alternative.** The prior `needsCompaction` flag, which told the worker
to "run `questmaster.compactComments` first (compact 5..N into one)." Rejected on
three counts: (1) `compactComments` never existed — not in the TOC, not exported,
a phantom action a worker can't run (scratch-guard blocks hand-rolling it);
(2) "compact 5..N into one" is DESTRUCTIVE collapse of the audit log, colliding
head-on with the locked "comments are the durable audit log, never delete" rule
(the same anti-pattern that caused the 5d29b94f data-loss defect, which
`digestAndRemoveQuestion` exists to prevent); (3) the worker never reads the whole
thread anyway (it reads the digest + unresolved user comments), so thread length
is a cost it doesn't pay — compaction optimized the wrong thing and would have
MASKED the real bloat sources (the derail loop + un-deduped purrview passes)
instead of fixing them.

**Why.** Observed on quest 988c9792 (23→25 comments): the old digest fed the
worker a SINGLE comment — the newest `system/derail_kickback` saying "there are
no deliverables" while the line above read "Deliverables (4/4 filled)" — i.e. the
one surfaced comment was actively false churn, while the user directives and the
origin hand-off were never surfaced. Signal in the read slice was ~0%.

**Consequence if violated.** Reintroducing a >N "compact the thread" flag sends
workers after a non-existent action and pressures destructive deletion of the
audit trail. Reverting the digest to "latest note only" re-buries real signal
under whatever telemetry was written most recently. Residual signal-loss sources
are write-side (idempotent system events; one UPSERT'd structured comment per
verify pass; a structured `verdict` field instead of regex-sniffing "fail" from
prose) — fix those at the write, not by deleting the read.

## 2026-06-15 — a verification pass writes ONE UPSERT'd structured comment; the digest classifies on a `verdict` FIELD, not prose

**Decision.** A purrview/review verification pass emits ONE `quest_comments` row,
not one-per-deliverable. The leaf path is `questmaster.recordVerifyPass`: it folds
every per-deliverable finding into a single `action='visual_verify'` row (findings
in `detail.findings`, a roll-up in `summary`, overall pass/fail in the new
top-level `verdict` column) and writes it THROUGH the centralized chokepoint
`libs/quest` `recordQuestComment` — `'visual_verify'` is registered in that
chokepoint's `UPSERT_ACTIONS`, so a second racing leaf (or a re-verify after
rework) REPLACES the SAME row in place (keyed on `quest_id` + `action`) instead of
appending. `recordVerifyPass` does NOT hand-roll its own select/update/insert —
that dedup policy lives in ONE place. `summarizeQuest.openVerifyFails` now
classifies on `c.verdict === 'fail'`, with the old prose regex demoted to a
fallback for legacy rows where `verdict IS NULL`. This is the write-side fix the
read-side digest ED (above) named as the residual.

**Rejected alternative.** Keep one `quest_comments` row per deliverable (the
shape a competent agent reaches by default — "loop the deliverables, write a
finding for each"), and keep detecting open fails by regex-sniffing "fail"/"bounce"
out of the `summary` prose.

**Why.** One-row-per-deliverable is the #2 comment-bloat source: a 4-deliverable
purrview pass wrote 5 rows, and two leaves racing the same pass wrote 10
near-identical rows (quest 988c9792). The prose regex is fragile — it mislabels any
comment that merely mentions the word "fail." A verdict that lives in a real column
is both leaner (one row) and machine-classifiable (no NLP on summaries). UPSERT (not
append) is the dedup; in-place UPDATE (not delete) keeps the locked "comments are
the durable audit log" rule — and the immutable pass/fail decision still lives on
the approve/bounce lockphrase comments, which `recordVerifyPass` never touches.
`depicts` + a verbatim `readback` stay MANDATORY per finding so the collapse does
not weaken the anti-rubber-stamp gate.

**Consequence if violated.** Revert to per-deliverable rows and the verify flood
returns (now amplified by the caravan running multiple verify leaves). Revert
`openVerifyFails` to the prose regex and the digest mislabels fails/passes on word
matches, re-burying the signal the read-side ED restored.

## 2026-06-16 — the checkpoint soft-gate is idempotent + advisory-only; it does NOT block, and it is at the wrong layer

**Decision.** `moveQuestToReview`'s "composite deliverables but zero quest_assets"
soft-gate writes its `checkpoint_warning` comment AT MOST ONCE per quest (guarded by a
prior-existence check), and it remains purely advisory — it never blocks the submit.

**Rejected alternative.** The original: insert the warning UNCONDITIONALLY on every
`moveQuestToReview` call. (And the tempting "make it a hard gate" — rejected below.)

**Why.** `moveQuestToReview` is retried every tick on any quest that can't yet advance, so
the unconditional insert spammed byte-identical rows — 40+ on quest `ec0cde6c` over 3 days
— which is the exact "looping comments" failure the caravan loop-breaker now escalates on.
Worse, the warning is UNACTIONABLE at this layer: checkpointing must happen DURING execution
(when an objective produces an artifact), but `moveQuestToReview` runs at submit time when
every objective is already `done` and the intermediate artifacts are gone. Telling the agent
to "consider checkpointing" then is too late — it can't act, so the condition persists and
(pre-fix) the comment re-fired forever. It must NOT be a HARD gate either: a missing
checkpoint is a future-reassembly inconvenience, not a correctness failure, and hard-blocking
review on it would strand finished quests. So: keep it advisory, but say it ONCE. The real
checkpoint enforcement lives at execution time — `summarizeQuest`'s `assetCheckpoints` flag
and the worker's `ruleCheckpointAssets`.

**Consequence if violated.** Restore the per-tick insert "so it's more visible" and you
recreate the comment loop (and trip the loop-breaker into escalating healthy quests).
Promote it to a hard gate and finished, reviewable quests get stranded in execute over a
non-correctness advisory.

---

## 2026-07-02 — hermes-leaf.mjs: GuildOS ceremony runs OUTSIDE Codex; only creative work enters the Codex sandbox

**Decision.** Hermes/Codex leaves are invoked through a thin wrapper
(`libs/weapon/questmaster/hermes-leaf.mjs`) that runs all deterministic GuildOS
reads/writes (getQuestDetails, patchObjective, writeWorkerSummaryComment,
toggleQuestActiveExecution) via `run-action.mjs` in a plain Node.js process
OUTSIDE the Codex sandbox. Only the creative/reasoning work goes to
`hermes -z` as a simplified prompt requesting JSON-structured output.

**Rejected alternative.** Pass the standard `buildLeafExecutePrompt` (with its
`node --no-warnings libs/adventurer/run-action.mjs` instructions) directly to
the Codex agent. This was the prior approach; it failed because Codex's sandbox
either narrates the commands instead of executing them, or the nested
`node run-action.mjs` dies in the heavy desktop-Codex sandbox ('no final
response' / timeout).

**Why.** The Codex sandbox is optimized for reasoning and code generation, not
for executing complex Node.js processes with custom import resolution (`@/`
aliases via `_alias-hook.mjs`). Splitting the work into
deterministic-outside-sandbox + creative-inside-sandbox plays to each
substrate's strengths: Node.js handles GuildOS plumbing reliably; Codex handles
thinking.

**Consequence if violated.** Sending the full leaf prompt to Codex (or any
future non-Claude substrate) re-introduces the narrate-instead-of-execute
failure. Every new worker type that can't reliably run `run-action.mjs` needs
its own thin wrapper in this pattern.

**Setup note.** The wrapper must add `%LOCALAPPDATA%/OpenAI/Codex/bin` to PATH
when spawning hermes — the `codex` binary lives there but isn't on the system
PATH by default. The adventurer's `system_prompt` template is:
`node --no-warnings libs/weapon/questmaster/hermes-leaf.mjs [prompt_file]`.

---

## 2026-07-03 — Plan-stage stale guard escalates never-itemized quests; runs BEFORE dispatch-plan (SIO #7)

**Decision.** A new `escalateStalePlanQuests` guard (its own module
`planStaleGuard.js`) runs as questTick STEP 2a — BEFORE the dispatch-plan step
(2b). It escalates any plan-stage, unleased quest owned by the assignee that has
0 objectives and is older than `PLAN_STALE_MINUTES` (30, ≈6 ticks): flip
stage→escalated, mint an unblocking question, write a queryable
`escalation_reason` comment.

**Rejected alternatives.**
- *Put the check AFTER dispatch-plan (the natural place — right beside the other
  plan handling).* This is the load-bearing ordering trap: dispatch-plan LEASES
  each unleased plan quest and fires a planning leaf every tick, so by the time a
  post-2b guard ran, the stale quest would be leased again and the guard's
  `worker_id IS NULL` filter would skip it forever — it would NEVER fire. Running
  before 2b, an escalated quest leaves the plan stage and 2b simply doesn't select
  it.
- *Auto-RE-PLAN on the threshold instead of escalating.* Rejected as the primary
  action — dispatch-plan ALREADY re-runs planQuest every tick (continuous
  re-plan). Re-planning a quest that has failed to itemize for 30 min just repeats
  the same failure; the missing behavior is a TERMINAL that surfaces it to the
  user. So the guard escalates (with an explicit reason); the ongoing re-plan is
  what already gave planning its ~6 attempts.
- *Inline the guard in questTick.* Rejected — questmaster/index.js is 3.5k lines
  with a heavy import graph (claudecli/leafBrowser/asana/questPurrview); a separate
  module keeps the guard unit-testable in isolation (same split rationale as the
  caravan weapon's loadquests.js / logs.js), and questTick just calls it.

**Why.** A quest created but never itemized sits at `plan` with 0 objectives
forever (evidence: 7fd07621). The caravan derail ladder treats "plan + no
objectives" as the CORRECT shape (gate: no objectives → plan), so its rescuer
never touches it — nothing else catches this rot. The `escalation_reason` tag is
deliberately the SAME action used by the caravan escalation sites (SIO #4), so a
reviewer answers "why did this escalate?" with one query across both subsystems.

**Consequence if violated.** Move the guard after dispatch-plan → the re-lease
hides every stale quest from it and it silently no-ops (the exact "rots forever"
bug it exists to kill). Swap escalate for re-plan-only → the never-itemizable
quest loops planning attempts indefinitely and still never reaches a human. Drop
the age threshold → freshly-created quests (legitimately 0 objectives for their
first few ticks, before planning runs) get wrongly escalated.

---

## 2026-07-03 - Hermes HTML deliverables use `data:text/html` refs for visual verification

**Decision.** When `hermes-leaf.mjs` receives an HTML deliverable as inline
content, it stores the deliverable ref as a `data:text/html` URL instead of
uploading the HTML file to Supabase Storage.

**Rejected alternative.** Upload the HTML to `GuildOS_Bucket` with
`contentType: "text/html; charset=utf-8"` and point the deliverable ref at the
public Supabase object URL.

**Why.** Supabase public object URLs for HTML are served as `text/plain` with a
sandbox CSP in this project. The headless visual gate then screenshots raw
source text, and `openai_images.judge` correctly returns mismatch. A
`data:text/html` ref renders as the intended page in `questPurrview.autoVisualVerify`
and still produces a normal uploaded `verify_screenshot_ref` PNG for review.

**Consequence if violated.** Replacing the data URL with a storage URL makes
Hermes-produced HTML look like source code during purrview, causing the real
vision judge and approve gate to fail even when the HTML content itself is
correct.

## 2026-07-03 — no external archive target is a clean SKIP, not a user decision (closing = review/evolve)

**Decision.** In `archiveQuestTakeaways`, a quest reaching `closing` with **no
linked source Asana task** (or any other archive target) is closed silently —
there is nothing to archive, so it is NOT raised as an unblocking question. Only
GENUINE decisions still escalate: an unresolved `action='closing'` finale comment
(a discovered ED/insight to apply to its substrate), a linked-Asana-but-no-report
case, or an Asana post that actually failed.

**Rejected alternative.** The original code pushed `"This quest has no linked
source Asana task … decide how its results should be archived (Asana? email?
other?)"` as a pending decision → `escalated`. (This stranded 6 no-Asana test
quests, e.g. "Tell a dog joke", on a spurious archiving question.)

**Why.** Closing's purpose is shifting to *reviewing the quest and evolving the
skills* (the Hermes model), not forcing every quest through an external archive.
Most quests have no Asana source and never will; treating "no target" as a
user-facing decision manufactures a stop for the common case and blocks
completion. Archiving is opportunistic (do it when a target exists), not
mandatory.

**Consequence if violated.** Re-adding the no-target escalation re-strands every
non-Asana quest at `escalated` on a question the user cannot meaningfully answer,
defeating auto-close and burying real decisions under noise.

---

## 2026-07-03 — the worker status writers UPSERT (they do NOT append per tick) (SIO #2)

**Decision.** `writeWorkerStatus` (new) is the canonical per-tick status heartbeat
(`action='worker_status'`), and `writeWorkerSummaryComment` keeps its
`action='worker_summary'`. Both now collapse to ONE row per quest because the
chokepoint treats those actions as WORKER_HEARTBEAT (see `libs/quest`
engineering_decisions.md). Neither writer changed its return contract — `commentId`
comes back on the first insert and on every in-place update alike.

**Rejected alternative.** Add a `writeWorkerStatus` that hand-rolls its own
`.update()` on quest_comments (a parallel upsert path), or keep appending and
"clean up later".

**Why.** The collapse policy belongs in the ONE chokepoint (`recordQuestComment`),
not re-implemented per writer — the same reason `recordVerifyPass` routes through it
rather than inserting directly. Making the writers thin callers of the chokepoint
means the heartbeat set is the single place that decides append-vs-upsert, and a new
status writer gets the behavior for free by choosing a heartbeat action. Keeping
`writeWorkerSummaryComment`'s signature identical means no caller (worker skill book,
`hermes-leaf.mjs`) needs a change to stop bloating — the collapse is transparent.

**Consequence if violated.** A hand-rolled per-writer upsert drifts from the
chokepoint's key/`created_at`-bump semantics (re-introducing the false-stale bug),
and duplicating the policy means the next status writer silently appends again.

## 2026-07-03 — the purrview leaf prompt branches its verification method by deliverable Type

**Decision.** `buildLeafPurrviewPrompt` gives the reviewer a TWO-BRANCH procedure:
VISUAL kinds (html/image/document/video) are verified by SCREENSHOT (open + look at
pixels → upload → set verify_screenshot_ref to the Supabase URL); the TEXT kind is
verified by READING the content and setting verify_screenshot_ref to a `PASS …`/`FAIL …`
verdict — no screenshot. The prompt also scopes to ACTIVE deliverables only, and the
VERIFY payload (libs/caravan/loadquests.buildVerifyPayload) now carries `status`.

**Rejected alternative.** The prior single-branch prompt: "CIC-screenshot EVERY
deliverable." 

**Why.** A text answer is verified by reading it; forcing a screenshot made the whole
gate hostage to the browser/auth channel and produced the "login-screen" false-fails.
This is the leaf-facing half of the questPurrview decision — see
`libs/weapon/questPurrview/engineering_decisions.md` (2026-07-03) for the gate/field
rationale and the `classifyPurrviewVerification` value contract.

**Consequence if violated.** Reverting the prompt to "screenshot everything" re-couples
text verification to the browser channel and reopens the login-screen false-fail class.

---

## 2026-06-24 — `checkpointQuestAsset` always uses the service client

**Decision.** `checkpointQuestAsset` injects an explicit service client into both
`insertQuestAsset` calls — it does NOT rely on `resolveServerClient`'s fallback
chain.

**Rejected alternative.** Letting `insertQuestAsset` resolve its own client via
`resolveServerClient` (the default), which is what every other quest write does.

**Why.** `scripts/_alias-hook.mjs` (the canonical CLI's loader) stubs
`next/headers` so `cookies()` returns an empty-but-never-throwing object. This
makes `createServerClient()` succeed with the publishable key + zero auth
cookies, so `resolveServerClient`'s try/catch fallback to the service client
never triggers — every worker-side checkpoint inserts under RLS as an
unauthenticated session and is rejected (`new row violates row-level security
policy for table "quest_assets"`). Worker-leaf cleaf-22deea9b-mqrfkoq0 hit
this and worked around it by hand-rolling a service-client REST insert; the
next tick would have repeated the work. Service-client injection is correct
ON ITS OWN MERITS here too: `checkpointQuestAsset` is a worker / cron / leaf
primitive — it has no user-facing caller — so respecting RLS is meaningless
overhead; the function is exactly the kind of "trusted server job" the service
client exists for.

**Consequence if violated.** Revert to `resolveServerClient` and every leaf
worker under `run-action.mjs` (the canonical CLI for all skill-book access)
hits the same RLS error, then either silently skips its save-forward
checkpoint or hand-rolls a service client around the wrapper — drifting back
to the exact two-step "upload + REST-insert" pattern this wrapper exists to
replace.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/questmaster/hermes-leaf.mjs -->
```js
#!/usr/bin/env node
/**
 * hermes-leaf.mjs — Thin wrapper that runs GuildOS quest ceremony OUTSIDE the
 * Codex sandbox and passes only the creative/reasoning work to Hermes.
 *
 * WHY THIS EXISTS: the standard leaf prompt tells the agent to call
 * `node --no-warnings libs/adventurer/run-action.mjs` for all quest mutations.
 * Codex can't reliably execute these commands (nested node dies in the Codex
 * sandbox, or the agent narrates instead of executing). This wrapper handles
 * all deterministic GuildOS reads/writes via run-action.mjs OUTSIDE the Codex
 * agent, and only sends the creative objective work to Hermes.
 *
 * Used as the system_prompt for hermes-type adventurers:
 *   node --no-warnings libs/weapon/questmaster/hermes-leaf.mjs {{PROMPT_FILE}}
 *
 * The flow:
 *   1. Read the leaf prompt file (standard buildLeafExecutePrompt output)
 *   2. Extract quest ID, worker ID from the prompt text
 *   3. Fetch quest details via run-action.mjs (deterministic, outside Codex)
 *   4. Find the next active objective
 *   5. Build a Codex-friendly prompt (no run-action.mjs commands)
 *   6. Call `hermes -z` with the simplified prompt
 *   7. Parse Codex's output and write mutations back (patch objective, add comment)
 *   8. Release the lease
 */

import "dotenv/config";
import { register } from "node:module";
import { readFileSync, writeFileSync, unlinkSync } from "node:fs";
import { execSync, spawnSync } from "node:child_process";
import { resolve, join } from "node:path";
import { tmpdir } from "node:os";
import { randomUUID } from "node:crypto";
import { pathToFileURL } from "node:url";
import { buildHermesInvocation, HERMES_PROMPT_ARGV_LIMIT } from "./hermesInvocation.js";

const REPO_ROOT = resolve(import.meta.dirname, "..", "..", "..");
register(
  pathToFileURL(resolve(REPO_ROOT, "scripts", "_alias-hook.mjs")).href,
  pathToFileURL(REPO_ROOT + "/").href,
);

// ── helpers ─────────────────────────────────────────────────────────────────

function runAction(skillBook, action, input) {
  // Write input JSON to a temp file to avoid shell escaping issues (especially on Windows)
  const inputFile = join(tmpdir(), `hermes-input-${randomUUID()}.json`);
  writeFileSync(inputFile, JSON.stringify(input), "utf8");
  const cmd = `node --no-warnings libs/adventurer/run-action.mjs ${skillBook} ${action} --input-file "${inputFile}"`;
  try {
    const out = execSync(cmd, { cwd: REPO_ROOT, timeout: 30_000, encoding: "utf8", stdio: ["pipe", "pipe", "pipe"] });
    return JSON.parse(out.trim());
  } catch (e) {
    const stderr = e.stderr ? e.stderr.toString().slice(0, 500) : "";
    console.error(`[hermes-leaf] run-action ${skillBook}.${action} failed: ${e.message}\n${stderr}`);
    return { ok: false, error: e.message };
  } finally {
    try { unlinkSync(inputFile); } catch { /* ignore */ }
  }
}

function callHermes(prompt) {
  // Ensure the codex binary is on PATH — it lives at %LOCALAPPDATA%/OpenAI/Codex/bin
  // and hermes's codex_app_server provider needs it to spawn the codex runtime.
  const codexBinDir = resolve(process.env.LOCALAPPDATA || "", "OpenAI", "Codex", "bin");
  const env = { ...process.env, PATH: `${codexBinDir};${process.env.PATH || ""}` };
  const { args, scratchPath, spilled } = buildHermesInvocation(prompt);
  if (spilled) {
    console.log(`[hermes-leaf] prompt ${Buffer.byteLength(prompt, "utf8")} bytes > ${HERMES_PROMPT_ARGV_LIMIT} — spilled to ${scratchPath}`);
  }
  try {
    const result = spawnSync("uv", args, {
      cwd: REPO_ROOT, timeout: 5 * 60_000, encoding: "utf8", stdio: ["pipe", "pipe", "pipe"], env,
    });
    if (result.error) throw result.error;
    if (result.status !== 0) {
      console.error(`[hermes-leaf] hermes exited ${result.status}: ${(result.stderr || "").slice(0, 500)}`);
    }
    return (result.stdout || "").trim();
  } finally {
    if (scratchPath) { try { unlinkSync(scratchPath); } catch { /* ignore */ } }
  }
}

/** Try to extract a JSON block from Hermes output (may be wrapped in markdown fences). */
function extractJson(text) {
  // Try bare JSON parse first
  try { return JSON.parse(text); } catch { /* continue */ }
  // Try extracting from ```json ... ``` fences
  const fenced = text.match(/```(?:json)?\s*\n?([\s\S]*?)```/);
  if (fenced) {
    try { return JSON.parse(fenced[1].trim()); } catch { /* continue */ }
  }
  // Try finding a { ... } block
  const braces = text.match(/\{[\s\S]*\}/);
  if (braces) {
    try { return JSON.parse(braces[0]); } catch { /* continue */ }
  }
  return null;
}

// ── main ────────────────────────────────────────────────────────────────────
// Guarded so importing this module (e.g. a unit test exercising
// buildHermesInvocation — SIO #5) does NOT run the leaf. main() is hoisted, so
// the guard can call it before the declaration below.
const __isDirectRun = import.meta.url === pathToFileURL(process.argv[1] || "").href;
if (__isDirectRun) main();

function safeStorageName(name, fallback = "deliverable") {
  return String(name || fallback)
    .toLowerCase()
    .replace(/[^a-z0-9._-]+/g, "-")
    .replace(/^-+|-+$/g, "")
    .slice(0, 80) || fallback;
}

function inferContentType(kind, extension) {
  const ext = String(extension || "").toLowerCase();
  if (ext === "html" || kind === "html") return "text/html; charset=utf-8";
  if (ext === "json" || kind === "json") return "application/json; charset=utf-8";
  if (ext === "md" || kind === "markdown") return "text/markdown; charset=utf-8";
  if (ext === "svg") return "image/svg+xml";
  if (ext === "png") return "image/png";
  if (ext === "jpg" || ext === "jpeg") return "image/jpeg";
  return "text/plain; charset=utf-8";
}

function inferExtension(kind, contentType) {
  if (kind === "html") return "html";
  if (kind === "json") return "json";
  if (kind === "markdown") return "md";
  if (kind === "image") return contentType === "image/svg+xml" ? "svg" : "png";
  return "txt";
}

async function shipParsedDeliverables({ questId, parsed }) {
  if (!Array.isArray(parsed?.deliverables) || parsed.deliverables.length === 0) return [];
  const { writeDeliverable } = await import("@/libs/quest/index.js");
  const { writeFile, readPublicUrl } = await import("@/libs/weapon/supabase_storage/index.js");
  const shipped = [];

  for (const [index, d] of parsed.deliverables.entries()) {
    const name = String(d?.name || "").trim();
    if (!name) {
      shipped.push({ ok: false, error: `deliverable ${index + 1} missing name` });
      continue;
    }

    const kind = String(d.kind || "text").trim();
    let ref = typeof d.ref === "string" && d.ref.trim() ? d.ref.trim() : null;

    if (!ref) {
      const content = d.content ?? d.html ?? d.text ?? null;
      const base64 = typeof d.base64 === "string" && d.base64.trim() ? d.base64.trim() : null;
      if (content == null && !base64) {
        shipped.push({ ok: false, name, error: "missing ref or content/base64" });
        continue;
      }

      const contentType = String(d.contentType || d.content_type || inferContentType(kind, d.extension)).trim();
      if (kind === "html" && !base64 && /^text\/html\b/i.test(contentType)) {
        // Supabase public storage serves uploaded HTML as text/plain+sandbox,
        // so the visual gate screenshots source text. data:text/html renders.
        ref = `data:text/html; charset=utf-8,${encodeURIComponent(String(content))}`;
      } else {
        const extension = String(d.extension || inferExtension(kind, contentType)).replace(/^\./, "");
        const filename = `${safeStorageName(name, `deliverable-${index + 1}`)}-${randomUUID().slice(0, 8)}.${extension}`;
        const path = `quest-assets/${questId}/hermes/${filename}`;
        const uploadContent = base64 ? Buffer.from(base64, "base64") : Buffer.from(String(content), "utf8");
        await writeFile({ path, content: uploadContent, contentType, upsert: true });
        ref = readPublicUrl({ path }).url;
      }
    }

    const write = await writeDeliverable({ questId, name, kind, ref, verify_screenshot_ref: null });
    if (write.error) {
      shipped.push({ ok: false, name, ref, error: write.error.message });
      continue;
    }
    shipped.push({ ok: true, name, kind, ref });
  }

  return shipped;
}

async function main() {
const promptFile = process.argv[2];
if (!promptFile) {
  console.error("[hermes-leaf] usage: hermes-leaf.mjs <prompt-file>");
  process.exit(1);
}

const promptText = readFileSync(promptFile, "utf8");

// Extract quest ID and worker ID from the standard leaf prompt format
const questIdMatch = promptText.match(/Quest id:\s*([0-9a-f-]{36})/);
const workerIdMatch = promptText.match(/Your worker_id[^:]*:\s*(\S+)/);
if (!questIdMatch) {
  console.error("[hermes-leaf] could not find quest ID in prompt");
  process.exit(1);
}

const questId = questIdMatch[1];
const workerId = workerIdMatch ? workerIdMatch[1] : "hermes-leaf";

console.log(`[hermes-leaf] quest=${questId} worker=${workerId}`);

// ── 1. Orient: fetch quest details ──────────────────────────────────────────

const details = runAction("questmaster", "getQuestDetails", { questId });
if (!details.ok) {
  console.error("[hermes-leaf] getQuestDetails failed:", details.error);
  // Release lease and exit
  runAction("questmaster", "toggleQuestActiveExecution", { questId, release: true, workerId });
  process.exit(1);
}

const quest = details.quest;
const objectives = details.objectives || [];
const deliverables = details.deliverables || [];
const comments = details.comments || [];
const questions = details.questions || [];

// ── 2. Find the next active objective ───────────────────────────────────────

// Handle open questions first (answer user questions, skip unblocking)
const userQuestions = questions.filter((q) => q.qtype === "user" && !q.answered);
const answeredUnblocking = questions.filter((q) => q.qtype === "unblocking" && q.answered);
const openUnblocking = questions.filter((q) => q.qtype === "unblocking" && !q.answered);

// Find lowest-n active objective not blocked by an unblocking question
const activeStatuses = ["pending", "active", "running", "todo"];
const nextObjective = objectives
  .filter((o) => activeStatuses.includes(o.status))
  .sort((a, b) => a.n - b.n)[0];

if (!nextObjective && userQuestions.length === 0 && answeredUnblocking.length === 0) {
  // All objectives done — check if we should move to review
  const allDone = objectives.every((o) => o.status === "done");
  const allDelivsFilled = deliverables.every((d) => d.ref);
  if (allDone && allDelivsFilled) {
    console.log("[hermes-leaf] all objectives done + deliverables filled → moving to review");
    runAction("questmaster", "moveQuestToReview", { questId });
  } else {
    console.log("[hermes-leaf] no active objective but not all done/filled. Skipping.");
  }
  // Add closing comment
  runAction("questmaster", "writeWorkerSummaryComment", {
    questId,
    summary: `Done: all workable objectives processed.\nNext: ${allDone && allDelivsFilled ? "moved to review" : "waiting for blocked objectives to unblock"}\nBlocked: ${openUnblocking.length > 0 ? "open unblocking questions" : "none"}`,
    workerId: "Main_Hermes_Codex",
  });
  runAction("questmaster", "toggleQuestActiveExecution", { questId, release: true, workerId });
  process.exit(0);
}

// ── 3. Build the Codex-friendly prompt ──────────────────────────────────────

const contextLines = [
  `You are a GuildOS worker. You are working on a quest.`,
  ``,
  `## Quest`,
  `Title: ${quest.title}`,
  `Description: ${(quest.description || "").slice(0, 2000)}`,
  `Stage: ${quest.stage}`,
  ``,
];

// Add user questions to answer
if (userQuestions.length > 0) {
  contextLines.push(`## Open User Questions (answer ALL before doing objective work)`);
  for (const q of userQuestions) {
    contextLines.push(`- Question ID: ${q.id}`);
    contextLines.push(`  Body: ${q.body}`);
  }
  contextLines.push(``);
}

// Add answered unblocking questions to digest
if (answeredUnblocking.length > 0) {
  contextLines.push(`## Answered Unblocking Questions (digest these decisions)`);
  for (const q of answeredUnblocking) {
    contextLines.push(`- Question ID: ${q.id}`);
    contextLines.push(`  Body: ${q.body}`);
    contextLines.push(`  Answer: ${q.answer}`);
  }
  contextLines.push(``);
}

// Add unresolved user comments
const unresolvedUserComments = comments.filter((c) => c.source === "user" && !c.resolved);
if (unresolvedUserComments.length > 0) {
  contextLines.push(`## Unresolved User Comments (follow the directive, then mark resolved)`);
  for (const c of unresolvedUserComments) {
    contextLines.push(`- Comment ID: ${c.id}`);
    contextLines.push(`  Summary: ${c.summary}`);
  }
  contextLines.push(``);
}

if (nextObjective) {
  contextLines.push(
    `## Your Objective (n=${nextObjective.n})`,
    `Name: ${nextObjective.name}`,
    `Status: ${nextObjective.status}`,
    `Notes: ${nextObjective.notes || "(none)"}`,
    ``,
    `## Deliverables`,
    ...deliverables.map((d, i) => `  ${i + 1}. ${d.name} [${d.kind}] — ref: ${d.ref || "(not yet filled)"}`),
    ``,
  );
}

contextLines.push(
  `## Instructions`,
  `Do the work described in the objective above. Think carefully and produce the result.`,
  ``,
  `OUTPUT your result as a JSON object with these fields:`,
  `{`,
  `  "objective_status": "done" or "pending" (if you completed it or not),`,
  `  "objective_delta": "brief description of what you did this session",`,
  `  "objective_notes": "updated notes for the objective (include prior notes + your additions)",`,
  `  "comment": "a closing summary: Done: <what> / Next: <what> / Blocked: <what or none>",`,
  `  "result": "the actual output/deliverable content (text, data, etc.)",`,
  `  "deliverables": [`,
  `    { "name": "<exact deliverable row name>", "kind": "html|text|image|document|video", "content": "<html/text/svg/etc to upload>", "contentType": "text/html; charset=utf-8", "extension": "html" }`,
  `  ]`,
  ...(userQuestions.length > 0
    ? [`, "question_answers": [{"question_id": "<id>", "answer": "<your answer>"}]`]
    : []),
  `}`,
  ``,
  `Output ONLY the JSON. No other text before or after.`,
);

const codexPrompt = contextLines.join("\n");
console.log(`[hermes-leaf] sending ${codexPrompt.length} chars to Hermes/Codex...`);

// ── 4. Call Hermes ──────────────────────────────────────────────────────────

let hermesOutput;
try {
  hermesOutput = callHermes(codexPrompt);
  console.log(`[hermes-leaf] Hermes returned ${hermesOutput.length} chars`);
} catch (e) {
  console.error(`[hermes-leaf] Hermes call failed: ${e.message}`);
  runAction("questmaster", "writeWorkerSummaryComment", {
    questId,
    summary: `Hermes/Codex leaf failed to execute: ${e.message}`,
    workerId: "Main_Hermes_Codex",
  });
  runAction("questmaster", "toggleQuestActiveExecution", { questId, release: true, workerId });
  process.exit(1);
}

// ── 5. Parse output and write mutations ─────────────────────────────────────

const parsed = extractJson(hermesOutput);

if (parsed && nextObjective) {
  const shipped = await shipParsedDeliverables({ questId, parsed });
  if (shipped.length > 0) {
    console.log(`[hermes-leaf] shipped deliverables: ${JSON.stringify(shipped)}`);
    const failed = shipped.filter((d) => !d.ok);
    if (failed.length > 0 && parsed.objective_status === "done") {
      parsed.objective_status = "pending";
      parsed.objective_delta = `Hermes produced output but ${failed.length} deliverable(s) failed to ship.`;
      parsed.objective_notes = `${parsed.objective_notes || nextObjective.notes || ""}\n\nDELIVERABLE SHIP FAILED: ${failed.map((d) => `${d.name || "unnamed"}: ${d.error}`).join("; ")}`.trim();
    }
  }

  // Patch the objective
  const patch = {};
  if (parsed.objective_status) patch.status = parsed.objective_status;
  if (parsed.objective_delta) patch.delta = parsed.objective_delta;
  if (parsed.objective_notes) patch.notes = parsed.objective_notes;

  if (Object.keys(patch).length > 0) {
    const patchResult = runAction("questmaster", "patchObjective", {
      objectiveId: nextObjective.id,
      patch,
    });
    console.log(`[hermes-leaf] patchObjective: ok=${patchResult.ok}`);
  }

  // Question answering is deferred — updateQuestQuestion is a server function
  // not exposed via run-action.mjs. The wrapper logs any answers Codex provided
  // in the closing comment for the user to action manually.
  if (Array.isArray(parsed.question_answers) && parsed.question_answers.length > 0) {
    console.log(`[hermes-leaf] Codex provided ${parsed.question_answers.length} question answer(s) — included in closing comment`);
  }
}

// Write closing comment
const commentText = parsed?.comment
  || `Hermes/Codex processed objective n=${nextObjective?.n || "?"}.\nRaw output: ${hermesOutput.slice(0, 500)}`;

runAction("questmaster", "writeWorkerSummaryComment", {
  questId,
  summary: commentText,
  workerId: "Main_Hermes_Codex",
});

// Check if all objectives are done now
if (parsed?.objective_status === "done") {
  const refreshed = runAction("questmaster", "getQuestDetails", { questId });
  if (refreshed.ok) {
    const allDone = (refreshed.objectives || []).every((o) => o.status === "done");
    const allFilled = (refreshed.deliverables || []).every((d) => d.ref);
    if (allDone && allFilled) {
      console.log("[hermes-leaf] all objectives done + deliverables filled → moveToReview");
      runAction("questmaster", "moveQuestToReview", { questId });
    }
  }
}

// ── 6. Release lease ────────────────────────────────────────────────────────

runAction("questmaster", "toggleQuestActiveExecution", { questId, release: true, workerId });
console.log(`[hermes-leaf] lease released. Done.`);
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/questmaster/hermes-leaf.test.js -->
```js
import { describe, it, expect } from "vitest";
import { readFileSync, existsSync, unlinkSync } from "node:fs";

// SIO #5 — large-deliverable passthrough. Pure builder split out of the CLI
// script (hermes-leaf.mjs) so it is importable without the leaf's main().
import { buildHermesInvocation, HERMES_PROMPT_ARGV_LIMIT } from "./hermesInvocation.js";

describe("SIO #5 — buildHermesInvocation large-payload passthrough", () => {
  it("passes a small prompt inline on argv (no behavior change for the common case)", () => {
    const prompt = "Do the objective and return JSON.";
    const { args, scratchPath, spilled } = buildHermesInvocation(prompt);
    expect(spilled).toBe(false);
    expect(scratchPath).toBeNull();
    expect(args).toContain(prompt); // the exact prompt rides on argv
    expect(args).toContain("-z");
  });

  it("spills a >200KB prompt to a scratch file and keeps every argv element tiny", () => {
    const big = "<html>" + "x".repeat(200_000) + "</html>";
    const { args, scratchPath, spilled } = buildHermesInvocation(big);
    try {
      expect(spilled).toBe(true);
      expect(scratchPath).toBeTruthy();
      // No argv element anywhere near the OS command-line limit.
      const largest = Math.max(...args.map((a) => Buffer.byteLength(a, "utf8")));
      expect(largest).toBeLessThan(2_000);
      // The full content is recoverable from the PATH we passed.
      expect(readFileSync(scratchPath, "utf8")).toBe(big);
      // The short goal must reference the scratch path so the agent reads it.
      const goal = args.find((a) => a.includes(scratchPath));
      expect(goal).toBeTruthy();
    } finally {
      if (scratchPath && existsSync(scratchPath)) unlinkSync(scratchPath);
    }
  });

  it("uses BYTE length for the threshold (multibyte content still spills)", () => {
    // Each '★' is 3 bytes in UTF-8; a char-count check would under-count.
    const chars = Math.ceil((HERMES_PROMPT_ARGV_LIMIT + 3_000) / 3);
    const multibyte = "★".repeat(chars);
    expect(multibyte.length).toBeLessThan(HERMES_PROMPT_ARGV_LIMIT); // char count is UNDER
    expect(Buffer.byteLength(multibyte, "utf8")).toBeGreaterThan(HERMES_PROMPT_ARGV_LIMIT); // bytes OVER
    const { spilled, scratchPath } = buildHermesInvocation(multibyte);
    try {
      expect(spilled).toBe(true);
    } finally {
      if (scratchPath && existsSync(scratchPath)) unlinkSync(scratchPath);
    }
  });

  it("honors a caller-supplied threshold and an injected writeFile (no disk I/O)", () => {
    let written = null;
    const { args, scratchPath, spilled } = buildHermesInvocation("abcdefghij", {
      threshold: 4,
      writeFile: (p, c) => { written = { p, c }; },
    });
    expect(spilled).toBe(true);
    expect(written.c).toBe("abcdefghij");
    expect(written.p).toBe(scratchPath);
    expect(args.some((a) => a.includes(scratchPath))).toBe(true);
  });
});

describe("SIO #5 — leaf worker guidance", () => {
  // index.js is the giant aggregator module and is deliberately NOT imported by
  // tests (Vite import-analysis chokes on it; existing suites test extracted
  // sibling modules instead). A source-text assertion is the regression guard
  // that buildLeafExecutePrompt keeps steering workers to --input-file for large
  // payloads.
  it("buildLeafExecutePrompt tells workers to spill large payloads to --input-file", () => {
    const src = readFileSync(new URL("./index.js", import.meta.url), "utf8");
    const start = src.indexOf("export function buildLeafExecutePrompt");
    expect(start).toBeGreaterThan(-1);
    const body = src.slice(start, start + 4_000);
    expect(body).toMatch(/LARGE PAYLOADS/);
    expect(body).toMatch(/--input-file/);
  });
});
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/questmaster/hermesInvocation.js -->
```js
/**
 * hermesInvocation.js — the pure, testable half of the Hermes leaf (SIO #5).
 *
 * Split out of hermes-leaf.mjs so it is importable by unit tests without
 * dragging in the CLI script's top-level main() / dotenv side-effects, and so
 * the large-payload passthrough logic can be exercised in isolation.
 */

import { writeFileSync } from "node:fs";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { randomUUID } from "node:crypto";

export const HERMES_PROJECT = "C:/Users/xsj70/hermes";

// SIO #5 — large-deliverable passthrough. Windows' CreateProcess caps a whole
// command line at ~32767 chars, so a large codexPrompt (accumulated objective
// notes, a big HTML deliverable pulled into context) passed as a spawnSync
// positional blows the arg limit before hermes ever starts. Above this
// threshold we spill the prompt to a scratch file and hand hermes a SHORT goal
// that tells the agent to read its full task from that PATH — passing the path,
// not the content. The threshold sits well under the OS cap to leave headroom
// for the rest of the argv + env the shell adds.
export const HERMES_PROMPT_ARGV_LIMIT = 24_000;

/**
 * Build the `uv run … hermes -z <goal>` argv for a prompt, spilling to a scratch
 * file when the prompt is too large to ride on the command line.
 *
 * @param {string} prompt  the full codex prompt
 * @param {{ threshold?: number, writeFile?: (path:string, content:string)=>void }} [opts]
 * @returns {{ args: string[], scratchPath: string|null, spilled: boolean }}
 */
export function buildHermesInvocation(prompt, opts = {}) {
  const threshold = opts.threshold ?? HERMES_PROMPT_ARGV_LIMIT;
  const write = opts.writeFile || ((p, c) => writeFileSync(p, c, "utf8"));
  const baseArgs = ["run", "--project", HERMES_PROJECT, "hermes", "-z"];
  const tail = ["--provider", "openai-codex", "--model", "gpt-5-codex", "--accept-hooks", "--yolo"];

  // Byte length, not char length — the OS limit is on bytes, and multibyte
  // content (emoji, non-ASCII in HTML) inflates past a char-count check.
  if (Buffer.byteLength(prompt, "utf8") <= threshold) {
    return { args: [...baseArgs, prompt, ...tail], scratchPath: null, spilled: false };
  }

  const scratchPath = join(tmpdir(), `hermes-prompt-${randomUUID()}.txt`);
  write(scratchPath, prompt);
  const goal =
    `Your full task is too large to pass on the command line, so it has been ` +
    `written to a file. Read the file at ${scratchPath} — it contains your ` +
    `complete instructions and the exact JSON output contract — then do the ` +
    `work and reply following that file's OUTPUT format. Do not ask for the ` +
    `task; it is in that file.`;
  return { args: [...baseArgs, goal, ...tail], scratchPath, spilled: true };
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/questmaster/index.js -->
```js
/**
 * questmaster weapon — Wave 1 refactor (2026-05-14).
 *
 * Refactored against the new Description / Objectives / Deliverables model encoded in
 * app/(town)/proving-grounds/quest-ui-mockup/page.js. The old deliverables-as-workflow
 * + action_steps jsonb on quests is replaced by:
 *   - an `objectives` table (per-quest workflow rows, ordered by n)
 *   - a repurposed `deliverables` table (pure asset storage, populated by execution)
 *   - quests gains `assignee` (text) and the stage CHECK tightens to the new
 *     5-value enum: execute / purrview / review / closing / complete.
 *
 * Seven actions:
 *   evaluate                  — programmatic gate (no LLM); decides whether SMARTClarification needs to run. Operates on objectives[] (was deliverables[]).
 *   SMARTItemClarification    — LLM eval per objective (uses skill_book/questmaster.deliverableTacticalEvaluation as prompt).
 *   searchQuestBySimilarName  — DB read + LLM pick among active quests.
 *   getActiveQuestNames       — DB read of non-terminal quests.
 *   getQuestDetails           — DB read of one quest + its objectives (ordered) + deliverables + quest_comments.
 *   createQuest               — DB write: insert quest (stage='plan'; big description, sub-items optional), insert any pre-planned objectives/deliverables rows, attempt plan→execute when pre-planned.
 *   dispatchRoledAgent        — Compose a role-aware priming preamble (CLAUDE.md + CLAUDE-<role>.md) and dispatch a sub-agent via claudecli.
 *
 * Pattern: thin code surface; LLM calls go through runGenericChat. Skill book howTos
 * are used as prompt templates for the LLM-bearing weapons.
 */

import { existsSync, readFileSync, readdirSync } from "fs";
import { join } from "path";
import { runGenericChat } from "@/libs/council/ai/chatCompletion.js";
import { invoke as claudecliInvoke } from "@/libs/weapon/claudecli/index.js";
import { invokeAdventurerLeaf } from "@/libs/adventurer/index.js";
import * as leafBrowser from "@/libs/weapon/leafBrowser/index.js";
import * as asana from "@/libs/weapon/asana/index.js";
import { autoVisualVerify } from "@/libs/weapon/questPurrview/index.js";
import { logWeaponCall } from "../weaponLog/index.js";
import { definition as questmasterSkillBook } from "@/libs/skill_book/questmaster";
// runReviewLoop's SOURCE lives in the questmaster skill book, but it is RE-EXPORTED
// here so `run-action.mjs questmaster runReviewLoop` resolves it via the CLI's
// weapon-fallback (which calls fn(input) with no context). Registering it instead
// as a skill-book adventurerAction would route it through buildAdventurerSkillBookView's
// wrapper, which throws unless an AsyncLocalStorage execution context is set — and the
// CLI never sets one, so the review-leaf's canonical call would fail. See ED 2026-07-03.
export { runReviewLoop } from "@/libs/skill_book/questmaster";
import { publicTables } from "@/libs/council/publicTables";
import { questPath } from "@/libs/council/questPath";
import { resolveAssignee } from "@/libs/quest";
import { escalateStalePlanQuests } from "./planStaleGuard.js";

// computeQuestionGate — inlined verbatim from GuildOS libs/caravan/loadquests.js
// during the GuildOS→devguild migration. devguild has no `libs/caravan/` (it uses
// the thin `libs/guild/` tick instead), so the migrated import dangled and broke
// the ENTIRE questmaster weapon (createQuest/planQuest/questTick) for every
// consumer. This is a pure, dependency-free function; inlining it restores the
// weapon without resurrecting the dropped caravan module. Behavior identical.
function computeQuestionGate(questions = []) {
  const qs = questions || [];
  const userToAnswer = qs.filter((q) => q.qtype === "user" && !q.answered);
  const answersToDigest = qs.filter((q) => q.qtype === "unblocking" && q.answered);
  const awaitingUser = qs.filter((q) => q.qtype === "unblocking" && !q.answered);
  const slim = (q) => ({ id: q.id, body: q.body });
  return {
    blocked: userToAnswer.length + answersToDigest.length + awaitingUser.length > 0,
    actionable: userToAnswer.length + answersToDigest.length > 0,
    userToAnswer: userToAnswer.map(slim),
    answersToDigest: answersToDigest.map((q) => ({ id: q.id, body: q.body, answer: q.answer })),
    awaitingUser: awaitingUser.map(slim),
  };
}

// ── toc ──────────────────────────────────────────────────────────────────────

export const toc = {
  evaluate: "Programmatic SMART-readiness gate (no LLM). Decides whether SMARTClarification needs to run. Operates on the new objectives[] shape.",
  SMARTItemClarification: "LLM eval for ONE objective; returns ok or clarification questions.",
  searchQuestBySimilarName: "Search active quests for a name match via LLM; double-verifies positive matches by reading the quest.",
  getActiveQuestNames: "Read non-terminal quests (stage NOT IN closing/complete).",
  getQuestDetails: "Read one quest plus its objectives (ordered by n), deliverables, and quest_comments.",
  summarizeQuest: "Rapid concise quest digest (no LLM, one pass): stage/lease, objective status counts + the next actionable objective, OPEN + answered-not-digested questions, deliverable fill ratio, and the latest note. Call this FIRST to orient instead of raw-reading every row; deep-read only what the digest flags.",
  createQuest: "Write a quest BORN AT stage='plan': ownerId + title + a big description (summarized from the user's ask) is the canonical input — objectives/deliverables are optional. The planning tick (questTick dispatch-plan → planQuest) itemizes the description and transitions to execute; pre-planned callers that DO pass objectives get an immediate moveQuestToExecute attempt.",
  planQuest: "Distill a plan-stage quest's verbose description into concise description + objectives + questions + DELIVERABLES. Moves per-objective detail into objectives rows, ambiguities into quest_questions, and INTERPRETS the final artifacts from the description into deliverables rows with content-typed kinds (text/html/image/video/document) — but ONLY when the deliverables table is empty (the user owns any deliverables they defined). Calls moveQuestToExecute on success.",
  moveQuestToExecute: "Guard: quest has >=1 objective, >=1 deliverable, and an assignee. If all present -> stage='execute'. Else stay in plan with a comment naming what's missing.",
  moveQuestToReview: "Guard: every objective status='done', every deliverable ref filled. Transitions execute -> purrview via questExecution.submit.",
  moveQuestToEscalated: "Guard: every workable objective is blocked on an open user question. If so -> stage='escalated'. Else stay in execute.",
  dispatchRoledAgent: "Compose role-aware priming (CLAUDE.md + CLAUDE-<role>.md) and dispatch a sub-agent via claudecli.",
  tickWorkerDispatch: "One worker-dispatcher tick (no LLM): scan execute-stage quests assigned to Guildmaster, resume a running objective or load the lowest-n pending objective, flipping it pending->running for a single worker (no parallel fan-out).",
  questUpdate: "Comment-first audit write for a mid-flight quest change: insert the user request as a quest_comments row (action='update_request', resolved=false) BEFORE any mutation; returns the comment id + the quest's current objectives so the update agent can target one.",
  patchObjective: "Render a scoped mutation on one objectives row (name/progress/delta/notes/status) by objective id. The 'render' half of comment-first-then-render.",
  resolveQuestComment: "Mark a quest_comments row resolved=true — call only AFTER the requested change has landed (closes the update_request audit pair).",
  addQuestComment: "Append ONE quest_comments row via the guarded recordQuestComment chokepoint. Use for the per-tick stop-summary (action='close_summary', source='worker', body in `summary`) that ruleAcceptableStopState requires. Input: { questId, summary, action?, source?, actorName?, detail?, resolved? }. Defaults: action='close_summary', source='worker', resolved=true. Never pass a top-level verdict (CHECK constraint quest_comments_verdict_chk rejects it) — put sage verdict inside detail.verdict.",
  raiseUnblockingQuestion: "Insert ONE quest_questions row (qtype='unblocking', direction='to_user') via insertQuestQuestion. CLI bridge for workers that need to block an objective on a user decision. Input: { questId, body, unblocks_objective_ids?, actorName? }. Returns { ok, questionId }. After calling this, call moveQuestToEscalated to move the quest to escalated stage.",
  patchQuestQuestion: "Patch fields on a quest_questions row via updateQuestQuestionById (service client). Allowed fields: body, answer, answered, unblocks_objective_ids. Input: { questId, questionId, patch: { body?, answer?, answered?, unblocks_objective_ids? } }. Returns { ok, question }.",
  runReviewLoop: "One-call script-locked purrview review for ONE quest (re-exported from the questmaster skill book so the review leaf can call it via run-action): confirmSubmission → openai_images.judge each image deliverable → approve (all match) or bounce (any mismatch). Non-image deliverables return action='manual_required' (fall back to the reviewSubmission howTo). Input: { questId }.",
  recordVerifyPass: "Write ONE structured visual-verification comment for a WHOLE purrview/review pass, UPSERT'd (replace-in-place, never appended) — collapses the old one-row-per-deliverable flood (#2 comment-bloat source) into a single verdict-bearing row. Input: { questId, passType?:'purrview'|'review', workerId?, findings:[{deliverable, depicts, readback, verdict:'match'|'mismatch', probes?, mismatch?}], note? }. Each finding MUST carry depicts (what is on screen) + a verbatim readback (anti-rubber-stamp proof-of-looking). Sets the top-level quest_comments.verdict the digest classifies on. A re-run by another leaf replaces the row in place; never deletes (the durable pass/fail audit lives on the approve/bounce lockphrase comments).",
  digestAndRemoveQuestion: "The ONLY sanctioned way for a worker to remove an answered unblocking question. REQUIRES migrationNote (where the answer's decisions were migrated); archives Q+A+note into quest_comments BEFORE deleting. REFUSES without the note. NEVER raw-delete quest_questions and NEVER the auth-gated DELETE route — this is the guarded path (questionDigestion).",
  closeQuest: "Terminal transition closing->complete. Hard-gated: REFUSES unless the quest is at 'closing' AND every action='closing' finale comment is resolved. Archives a managerial summary comment, then writes + SELECT-back-verifies stage='complete'.",
  archiveQuestTakeaways: "Closing-stage archiving engine (run by the caravan 'closing' archive-leaf). For a quest at stage='closing': IF the description links a SOURCE Asana task, posts the deliverables + completion report there (the only auto-approved external archive today); if there is NO archive target, that is a clean skip — nothing to archive is NOT a user decision. Then closes the quest via closeQuest (closing->complete). Only genuine decisions escalate: an unresolved action='closing' finale comment (a discovered ED/insight to apply to its substrate), a linked-Asana-but-no-report case, or an Asana post that failed. Input: { questId, report, deliverableRefs?, pendingDecisions? }.",
  toggleQuestActiveExecution: "Lease/release a quest's execution slot via worker_id compare-and-set (null->workerId or clear). Prevents two workers on one quest. Returns { ok, leased|released }.",
  unstuckQuests: "Reclaim stale leases: clear worker_id on any quest leased longer than staleMinutes (default 20) with no progress, writing a system 'unstuck' incident comment so the next tick re-dispatches it.",
  questTick: "The cron stage-router (no LLM to decide; deterministic). Renders the QUEST_TICK_ACTIONS registry, reclaims stale leases, deploys a PLANNING leaf to each unleased plan-stage quest (itemize description → objectives/deliverables/questions → execute), then for each owned execute-stage quest with an active objective and no lease, leases it and deploys a leaf claude agent (claudecli) to push it forward. Idempotent via the lease.",
  auditLeafLogs: "Parse .worker-logs/*.log and emit per-leaf tool-call audit: ordered sequences, waste classification (getQuestDetails miss, import probing, --list discovery, skill-book source reads), and summary counts. Input: { logDir?, filterPrefix? }. No LLM.",
  checkpointQuestAsset: "One-call quest-asset checkpoint that honors assetInlineVsFileStorage: text < 1000 chars is stored INLINE (ref = the text, kind defaults 'text', no upload, filename optional); larger/binary content uploads to GuildOS_Bucket (kind defaults 'document') then inserts the quest_assets row. Replaces the two-step hand-rolled upload+REST-insert pattern. Input: { questId, label, content, filename?, kind?, contentType?, source?, actorName? }. Returns { ok, inline?, publicUrl?, assetId }.",
  shipDeliverableArtifact: "Upload a base64 or text artifact to GuildOS_Bucket, upsert the matching deliverables row via libs/quest.writeDeliverable, and optionally checkpoint a quest_asset/comment. Input: { questId, deliverableName, kind, filename, contentBase64?|content?, contentType?, workerId?, rationale?, checkpointAsset? }. Returns { ok, deliverable, publicUrl, storagePath, assetId?, commentId? }.",
  writeWorkerSummaryComment: "Write the required structured worker handoff comment for an execute burst. Input: { questId, summary, workerId }. Writes source='worker', action='worker_summary', resolved=false. NOTE (SIO #2): 'worker_summary' is now a WORKER_HEARTBEAT action, so re-writes UPSERT one row per quest (edited in place) instead of appending a new row every burst — use this for the recurring status handoff; record a real state transition via its own append action.",
  writeWorkerStatus: "Write/refresh a worker STATUS HEARTBEAT (SIO #2) — the canonical per-tick 'here is where I am now' line. Input: { questId, summary, workerId?, detail? }. Writes source='worker', action='worker_status', which UPSERTS one row per quest (edited in place each tick, created_at bumped so it stays the newest comment for liveness). Use every tick instead of appending a fresh status comment; a real state transition (objective done / stage change / escalation) is NOT a heartbeat — append those via their own action.",
  incrementQuestTally: "Atomically increment a quest's complexity tally columns (tool_call_count, total_tokens). Call at the end of a worker burst or from weapon-call hooks. Input: { questId, toolCalls?:number, tokens?:number }. Uses Postgres atomic increment (col + N). Returns { ok, tool_call_count, total_tokens }.",
  shouldDistillLesson: "Check whether a quest passes the complexity gate for lesson distillation. Returns { shouldDistill, tool_call_count, total_tokens, reason }. The closing leaf calls this BEFORE investing in lesson capture.",
  distillQuestLesson: "Write a distilled lesson to quest_lessons. Input: { questId, description, gitRef?, smokeTestResults?, smokeTestPass? }. Returns { ok, lessonId }.",
};

// ── Cron action registry ──────────────────────────────────────────────────────
//
// The set of actions Cron:questTick runs each tick, in order. Exported so the
// tick (and any UI / status surface) can RENDER the registered actions rather
// than hard-coding them at the call site. Each entry is pure metadata; the tick
// maps id -> behavior. Add a row here to register a new tick action.
export const QUEST_TICK_ACTIONS = [
  {
    id: "reclaim-stale-leases",
    stage: "*",
    description:
      "Reclaim leases held > staleMinutes (default 20) with no progress — clear worker_id so a dead/hung worker never pins a quest in execute forever (unstuckQuests).",
  },
  {
    id: "dispatch-plan",
    stage: "plan",
    description:
      "Find plan-stage quests owned by this assignee that are not leased; lease via compare-and-set and deploy a PLANNING leaf claude agent that itemizes the big description into objectives + deliverables + questions (questmaster.planQuest + skill-book howTo), shrinking the description to global instructions only, then transitions the quest to execute. Shares the single-flight slot budget.",
  },
  {
    id: "kickback-on-user-comment",
    stage: "review|escalated",
    description:
      "Find review/escalated-stage quests owned by this assignee whose LATEST quest_comment is from the user; bounce stage → execute so a worker re-addresses the user's input. Spares manual stage reassignment. Writes NO comment (the stage change is audited in quest_stage_log; bouncing to execute removes it from this scan, so no re-bounce; the worker reads the user's actual directive).",
  },
  {
    id: "dispatch-execute",
    stage: "execute",
    description:
      "Find execute-stage quests owned by this assignee that have an active (pending/running) objective and no worker_id lease; lease via compare-and-set and deploy a leaf claude agent to work the quest (executeQuest procedure).",
  },
  {
    id: "dispatch-purrview",
    stage: "purrview",
    description:
      "Find purrview-stage quests owned by this assignee with no lease; lease via compare-and-set and deploy a REVIEW leaf claude agent to review the submission (confirmSubmission → runReviewLoop → approve/bounce). Shares the single-flight slot budget with execute dispatch.",
  },
];

// ── helpers ──────────────────────────────────────────────────────────────────

async function getServiceClient() {
  const mod = await import("@/libs/council/database");
  return mod.database.init("service");
}

/** Parse JSON from LLM text; tolerates ```json fences and surrounding prose. */
function parseJsonFromText(text) {
  if (typeof text !== "string") return null;
  let s = text.trim();
  const fence = /^```(?:json)?\s*([\s\S]*?)```/im.exec(s);
  if (fence) s = fence[1].trim();
  const firstBrace = s.indexOf("{");
  const firstBracket = s.indexOf("[");
  const start = firstBrace === -1 ? firstBracket : firstBracket === -1 ? firstBrace : Math.min(firstBrace, firstBracket);
  if (start === -1) return null;
  // Try balanced scan
  const open = s[start];
  const close = open === "{" ? "}" : "]";
  let depth = 0;
  for (let i = start; i < s.length; i++) {
    if (s[i] === open) depth++;
    else if (s[i] === close) {
      depth--;
      if (depth === 0) {
        try {
          return JSON.parse(s.slice(start, i + 1));
        } catch {
          return null;
        }
      }
    }
  }
  return null;
}

// ── evaluate ─────────────────────────────────────────────────────────────────

/**
 * Programmatic SMART-readiness gate. No LLM. The whole point per D4 is to
 * decide whether SMARTClarification needs to run at all — running SMARTClarification
 * on a complete context will over-clarify (LLM bias toward finding issues).
 *
 * Input shape (the `context` produced by summarizeRelevantContext):
 *   {
 *     summary: string,
 *     questId?: string | null,
 *     mentions: string[],
 *     workflowComplete?: boolean,   // set by summarizeRelevantContext: false when the
 *                                   // user framed the workflow as partial/non-terminal,
 *                                   // OR the objectives do not reach a terminal deliverable
 *                                   // that resolves the stated objective, OR the objectives
 *                                   // reach a terminal action but the product boundary
 *                                   // is undefined (deliverable shape — reusable
 *                                   // feature/modules vs one-off run — and/or the
 *                                   // terminal output/report/notification/done-condition
 *                                   // not user-stated).
 *     undefinedTail?: string,       // one-line description of what is missing when
 *                                   // workflowComplete is false — the missing tail, or
 *                                   // the undefined product-boundary element(s); ""
 *                                   // otherwise.
 *     verificationOwnership?: "defined" | "undefined",  // undefined when any objective's
 *                                   // verification names a check but not the OWNER
 *                                   // entity that performs it (Cat / GM CIC / cursor
 *                                   // worker / per-page vs batch). missing == undefined.
 *     adoptionPrecedence?: "defined" | "undefined" | "n/a",  // undefined when the
 *                                   // workflow adopts content from one substrate into
 *                                   // another AND no precedence-default rule for
 *                                   // mid-execution ambiguities is named. "n/a" when
 *                                   // no cross-substrate adoption exists.
 *     finaleSurfacing?: "defined" | "undefined",  // undefined when the workflow
 *                                   // produces multiple deliverable artifacts AND no
 *                                   // end-point CIC-tab spec is given. missing == undefined.
 *     objectives?: Array<{
 *       n?: number,
 *       name: string,
 *       input?: string,
 *       method?: string,
 *       output?: Array<{ asset: string, format?: string, display?: string|null,
 *                        verification?: string, verification_example?: any }>,
 *       viewResults?: "Show"|"Checkpoint"|"No",
 *       outputKey?: string|null,
 *       notes?: string
 *     }>,
 *     deliverables?: Array<{ name: string, kind: "text"|"html"|"image"|"video"|"document", ref?: string|null }>
 *   }
 *
 * Returns {ok:true, smartJson} when (a) the workflow is complete/terminal AND
 * product-boundary-defined (context.workflowComplete !== false) AND (b) every
 * objective has input + method + non-empty output[] (each output entry carrying at
 * least asset + format) AND (c) every [IMPORTANT DETAIL]-bearing objective has a
 * NON-EMPTY verification (user-stated OR agent-composed — a missing
 * [USER-STATED] prefix is NOT, by itself, a gap; an EMPTY verification on such
 * an objective still is). Per-objective field presence is NECESSARY-BUT-NOT-SUFFICIENT:
 * a well-formed but truncated/partial front-half — OR an end-to-end workflow
 * that reaches a terminal action but whose product boundary (deliverable shape +
 * terminal report/notification/done-condition) the user has not defined — fails
 * on the completeness gate even when each listed objective is fully populated. Else
 * {ok:false, gaps:[{objectiveN?, missingField, note}]}.
 */
export function evaluate({ context } = {}) {
  const gaps = [];

  if (!context || typeof context !== "object") {
    gaps.push({ objectiveN: null, missingField: "context", note: "no context object provided" });
    const out = { ok: false, gaps };
    logWeaponCall({ weapon: "questmaster", fn: "evaluate", input: { context: !!context }, output: out, status: "fail" });
    return out;
  }

  const objectives = Array.isArray(context.objectives) ? context.objectives : [];

  if (objectives.length === 0) {
    gaps.push({ objectiveN: null, missingField: "objectives", note: "no objectives defined" });
    const out = { ok: false, gaps };
    logWeaponCall({ weapon: "questmaster", fn: "evaluate", input: { objectiveCount: 0 }, output: out, status: "fail" });
    return out;
  }

  // Completeness / terminality / product-boundary gate (programmatic, no LLM).
  // Per-objective field presence is NECESSARY-BUT-NOT-SUFFICIENT: a well-formed but
  // truncated front-half (each objective fully populated, but the workflow never
  // reaches a terminal deliverable resolving the user's objective) — OR an
  // end-to-end workflow that reaches a terminal ACTION but whose product
  // boundary is undefined (the user has not stated the deliverable shape —
  // reusable feature/modules vs one-off run — and/or the terminal
  // output/report/notification/done-condition) — must NOT pass to upsertQuest.
  // summarizeRelevantContext judges BOTH completeness and product-boundary
  // definition and emits the single explicit signal context.workflowComplete
  // (false when partial/non-terminal, when objectives don't reach a terminal
  // deliverable, OR when an end-to-end workflow's product boundary is
  // undefined). This firewall keys on that one LLM-emitted signal the same way
  // it keys on [IMPORTANT DETAIL]/[USER-STATED] — it stays no-LLM here, one
  // gate, no logic change vs the field's original semantics; summarize merely
  // broadened WHEN it sets the field. A tool/API/schema unknown INSIDE an
  // otherwise end-to-end, product-boundary-defined workflow is an
  // implementation checkpoint and leaves workflowComplete === true, so this
  // gate does not over-fire on those. Only an explicit `false` fires it.
  if (context.workflowComplete === false) {
    const tail =
      typeof context.undefinedTail === "string" && context.undefinedTail.trim() !== ""
        ? context.undefinedTail.trim()
        : "the workflow is partial / non-terminal, or its product boundary is undefined — no terminal deliverable resolves the stated objective and/or the deliverable shape + terminal report are not user-stated";
    gaps.push({
      objectiveN: null,
      missingField: "workflowComplete",
      note: `workflow not ready for quest creation: ${tail}. The listed objectives may each be well-formed and may even run to a terminal action, but the workflow either does not run end-to-end to a deliverable that resolves the user's objective OR its product boundary (deliverable shape — reusable feature/modules vs one-off run — and the terminal output/report/notification/done-condition) is undefined — route to SMARTClarification Tier 2 (partial plan table + targeted tail / product-boundary questions), do not create a quest.`,
    });
  }

  // Verification-owner / adoption-precedence / finale-surfacing gaps (locked
  // 2026-05-20 after a real over-acceptance defect — SE-cursor-hand-off-test
  // Step 2). The prior gate accepted any non-empty verification string and
  // never asked WHO performs the check, never noticed cross-substrate adoption
  // ambiguities, and never noticed when a multi-deliverable workflow lacked an
  // end-point CIC-tab spec. summarizeRelevantContext now emits three explicit
  // context-level signals that mirror the workflowComplete pattern:
  //   - verificationOwnership: "defined" | "undefined" — undefined when any objective's
  //     verification clause names a check (curl, OCR, screenshot, lint) but does
  //     NOT name the OWNER entity that performs it (Cat at purrview / GM CIC /
  //     cursor worker / per-page or batch).
  //   - adoptionPrecedence: "defined" | "undefined" | "n/a" — undefined when the
  //     workflow ADOPTS content/data from one substrate into another (X→Y mapping,
  //     CJGEO→Magento adoption, source→destination hydration) and the
  //     precedence/default rule for unspecified mid-execution ambiguities (diagram
  //     fill, anchor injection, deep-dive link injection, branding-grep) is NOT
  //     named. "n/a" when no cross-substrate adoption exists in the workflow.
  //   - finaleSurfacing: "defined" | "undefined" — undefined when the workflow
  //     has MULTIPLE deliverable artifacts (multiple live pages, multiple files,
  //     a quest page + N satellite artifacts) and no end-point CIC-tab spec
  //     names which tabs the GM opens at finish (quest page only? + live mockup?
  //     + cursor agent FINISHED page?). "defined" when the workflow produces a
  //     single artifact OR the tab list is explicit.
  // Each undefined signal becomes a gap class; SMARTClarification Tier-2 surfaces
  // one targeted question per gap. Same no-LLM contract as workflowComplete.
  // A signal that is missing entirely from context is treated as "undefined"
  // (the howTo MUST emit it) — same fail-loud discipline as workflowComplete.
  if (context.verificationOwnership === "undefined" || context.verificationOwnership === undefined) {
    gaps.push({
      objectiveN: null,
      missingField: "verificationOwnership",
      note: "one or more objectives name a verification check (OCR, screenshot, curl, lint) but do NOT name the OWNER entity that performs it (Cat at purrview / GM CIC / cursor worker / per-page vs batch). A verification without a named owner is not actionable — Tier-2 must ask which entity owns each unowned check.",
    });
  }
  // adoptionPrecedence: tri-state ("defined" | "undefined" | "n/a"). Per the
  // howTo, the LLM MUST emit one of the three on every turn with objectives; a
  // missing field == "undefined" (same fail-loud discipline as workflowComplete
  // / verificationOwnership / finaleSurfacing). The carve-out is "n/a" — pure
  // infra / pure data-transform workflows that do NO cross-substrate content
  // adoption — which passes. Only an explicit "defined" or "n/a" passes.
  if (context.adoptionPrecedence !== "defined" && context.adoptionPrecedence !== "n/a") {
    gaps.push({
      objectiveN: null,
      missingField: "adoptionPrecedence",
      note: "the workflow adopts content/data from one substrate into another (source→destination hydration / X→Y mapping) and the PRECEDENCE rule for unspecified mid-execution ambiguities (diagram fill, anchor injection, deep-dive link injection, branding-grep, etc.) is NOT named. Without a precedence default (e.g. 'default to BryanBlog precedent' vs 'invent a destination-specific path'), the executing agent re-litigates every ambiguity. Tier-2 must ask which precedent rule applies. (Set context.adoptionPrecedence='n/a' explicitly if the workflow does no cross-substrate content adoption.)",
    });
  }
  if (context.finaleSurfacing === "undefined" || context.finaleSurfacing === undefined) {
    gaps.push({
      objectiveN: null,
      missingField: "finaleSurfacing",
      note: "the workflow produces multiple deliverable artifacts (multiple live pages / multiple files / a quest page + N satellites) and the end-point CIC-tab surfacing rule is NOT defined — which tabs does the GM open at finish (quest page only? + one representative live artifact? + a dispatched-agent's session page?). Tier-2 must ask for the explicit end-point tab list.",
    });
  }

  const REQUIRED_STRING_FIELDS = ["name", "input", "method"];
  for (let i = 0; i < objectives.length; i++) {
    const s = objectives[i];
    const ref = s && typeof s.n === "number" ? `objective ${s.n}` : `objective #${i + 1}`;
    for (const field of REQUIRED_STRING_FIELDS) {
      const v = s?.[field];
      if (!v || (typeof v === "string" && v.trim() === "")) {
        gaps.push({ objectiveN: s?.n ?? null, missingField: field, note: `${ref}: ${field} missing or empty` });
      }
    }
    // An objective whose method carries a user-flagged [IMPORTANT DETAIL] must have a
    // NON-EMPTY verification — but it does NOT have to be user-stated. Relaxed
    // 2026-05-15 after a real over-gating defect: requiring a [USER-STATED]
    // prefix on every [IMPORTANT DETAIL] objective turned every constraint whose
    // check the user did not spell out into a planning gap, producing a wall of
    // "how do you verify X?" escalations and blocking quest creation even when
    // the workflow + product boundary were fully defined. By design,
    // summarizeRelevantContext now COMPOSES a concrete baseline verification for
    // each [IMPORTANT DETAIL] objective from the constraint itself (and records any
    // genuine tool/API/credential unknown as an inline impl-checkpoint note in
    // the objective, NOT as a clarification question). So: on an [IMPORTANT DETAIL]
    // objective, an EMPTY verification is still a gap (the agent was expected to
    // compose one); a NON-EMPTY verification — user-stated ([USER-STATED]) OR
    // agent-composed — is NOT a gap. Missing [USER-STATED] is no longer, by
    // itself, a planning gap when workflowComplete is true (the agent is
    // expected to have composed the verification; an unresolved tool/API is at
    // most an implementation checkpoint, never a Tier-2 escalation). The
    // workflowComplete===false product-boundary/completeness gate above still
    // blocks; genuine product-decision gaps still block; user-stated
    // verifications are still accepted verbatim. Stays no-LLM here.
    const methodHasImportantDetail =
      typeof s?.method === "string" && s.method.includes("[IMPORTANT DETAIL]");
    // output must be a non-empty array, and each entry must have asset + format.
    if (!Array.isArray(s?.output) || s.output.length === 0) {
      gaps.push({ objectiveN: s?.n ?? null, missingField: "output", note: `${ref}: output[] missing or empty` });
    } else {
      for (let j = 0; j < s.output.length; j++) {
        const o = s.output[j];
        if (!o || typeof o !== "object") {
          gaps.push({ objectiveN: s?.n ?? null, missingField: `output[${j}]`, note: `${ref}: output[${j}] is not an object` });
          continue;
        }
        if (!o.asset || (typeof o.asset === "string" && o.asset.trim() === "")) {
          gaps.push({ objectiveN: s?.n ?? null, missingField: `output[${j}].asset`, note: `${ref}: output[${j}].asset missing` });
        }
        if (!o.format || (typeof o.format === "string" && o.format.trim() === "")) {
          gaps.push({ objectiveN: s?.n ?? null, missingField: `output[${j}].format`, note: `${ref}: output[${j}].format missing` });
        }
        if (methodHasImportantDetail) {
          const ver = typeof o.verification === "string" ? o.verification.trim() : "";
          // Only an EMPTY verification on an [IMPORTANT DETAIL] objective is a gap —
          // the agent was expected to compose a concrete baseline verification
          // from the constraint. A non-empty verification (user-stated
          // [USER-STATED] OR agent-composed) passes: missing [USER-STATED] is
          // NOT, by itself, a planning gap. (workflowComplete===false still
          // blocks above; genuine product-decision gaps still block.)
          if (ver === "") {
            gaps.push({
              objectiveN: s?.n ?? null,
              missingField: `output[${j}].verification`,
              note: `${ref}: method carries a user-flagged [IMPORTANT DETAIL] but its verification is blank — the agent was expected to compose a concrete baseline verification derived from the constraint (or copy a [USER-STATED] one verbatim). An empty verification on an [IMPORTANT DETAIL] objective is the gap; a non-empty agent-composed or [USER-STATED] verification is NOT.`,
            });
          }
        }
      }
    }
  }

  if (gaps.length > 0) {
    const out = { ok: false, gaps };
    logWeaponCall({ weapon: "questmaster", fn: "evaluate", input: { objectiveCount: objectives.length }, output: { ok: false, gapCount: gaps.length }, status: "fail" });
    return out;
  }

  // Normalize objective shape; preserve n if provided, else assign 1-based.
  const normalizedObjectives = objectives.map((s, i) => ({
    n: typeof s.n === "number" ? s.n : i + 1,
    name: s.name,
    status: s.status || "pending",
    input: s.input,
    method: s.method,
    output: s.output,
    viewResults: s.viewResults || "No",
    outputKey: s.outputKey ?? null,
    notes: s.notes || "",
  }));

  const deliverables = Array.isArray(context.deliverables) ? context.deliverables : [];

  const smartJson = {
    title: context.summary?.slice(0, 80) || "Untitled SMART plan",
    summary: context.summary || "",
    mentions: context.mentions || [],
    objectives: normalizedObjectives,
    deliverables: deliverables.map((d) => ({
      name: d.name,
      kind: d.kind,
      ref: d.ref ?? null,
    })),
  };
  const out = { ok: true, smartJson };
  logWeaponCall({ weapon: "questmaster", fn: "evaluate", input: { objectiveCount: objectives.length, deliverableCount: deliverables.length }, output: { ok: true, objectiveCount: objectives.length }, status: "ok" });
  return out;
}

// ── SMARTItemClarification ───────────────────────────────────────────────────

/**
 * Deep eval for ONE objective. Wraps the deliverableTacticalEvaluation skill book
 * howTo as the LLM prompt. Returns either {ok:true} or {ok:false, questions[]}.
 *
 * @param {{context:object, objective:object, measurementCriteria?:string}} opts
 */
export async function SMARTItemClarification({ context, objective, measurementCriteria } = {}) {
  const promptTemplate = questmasterSkillBook?.toc?.deliverableTacticalEvaluation?.howTo;
  if (!promptTemplate) {
    const err = { ok: false, questions: ["[substrate error: deliverableTacticalEvaluation howTo missing]"] };
    logWeaponCall({ weapon: "questmaster", fn: "SMARTItemClarification", input: { hasContext: !!context }, output: err, status: "fail" });
    return err;
  }

  // Compose a short verification summary from objective.output[*].verification (if present).
  const verificationSummary = Array.isArray(objective?.output)
    ? objective.output
        .map((o) => (o && typeof o.verification === "string" ? `[${o.asset}] ${o.verification}` : null))
        .filter(Boolean)
        .join("\n")
    : "";

  const messages = [
    {
      role: "system",
      content:
        "You are a SMART objective tactical evaluator. Follow the procedure in the howTo exactly. Return ONLY valid JSON.",
    },
    {
      role: "user",
      content: [
        "## Skill book howTo (procedure):",
        promptTemplate,
        "",
        "## Inputs",
        `### Context summary:\n${context?.summary || "(no summary)"}`,
        `### Objective being evaluated:\n${JSON.stringify(objective, null, 2)}`,
        `### Verification criteria (from objective.output[*].verification):\n${verificationSummary || measurementCriteria || "(not specified)"}`,
        "",
        "## Output schema (return exactly one JSON object)",
        '{ "ok": true } when all three questions are answered; otherwise { "ok": false, "questions": ["...","..."] }',
      ].join("\n"),
    },
  ];

  let parsed = null;
  try {
    const res = await runGenericChat({ messages });
    parsed = parseJsonFromText(res.text);
  } catch (e) {
    const err = { ok: false, questions: [`[LLM error: ${e?.message || String(e)}]`] };
    logWeaponCall({ weapon: "questmaster", fn: "SMARTItemClarification", input: { objectiveN: objective?.n }, output: err, status: "fail" });
    return err;
  }

  if (!parsed) {
    const err = { ok: false, questions: ["[unparseable LLM response]"] };
    logWeaponCall({ weapon: "questmaster", fn: "SMARTItemClarification", input: { objectiveN: objective?.n }, output: err, status: "fail" });
    return err;
  }

  const out =
    parsed.ok === true
      ? { ok: true }
      : { ok: false, questions: Array.isArray(parsed.questions) ? parsed.questions : ["[malformed clarification response]"] };
  logWeaponCall({ weapon: "questmaster", fn: "SMARTItemClarification", input: { objectiveN: objective?.n }, output: { ok: out.ok, qCount: out.questions?.length || 0 }, status: "ok" });
  return out;
}

// ── getActiveQuestNames ──────────────────────────────────────────────────────

/**
 * Reads non-terminal quests (any stage NOT in closing/complete).
 * Returns { quests: [{id, title, stage}] }.
 */
export async function getActiveQuestNames({ ownerId } = {}) {
  const client = await getServiceClient();
  let q = client.from(publicTables.quests).select("id, title, stage").not("stage", "in", "(closing,complete)");
  if (ownerId) q = q.eq("owner_id", ownerId);
  const { data, error } = await q;
  const out = error ? { error: error.message, quests: [] } : { quests: data || [] };
  logWeaponCall({ weapon: "questmaster", fn: "getActiveQuestNames", input: { ownerId: !!ownerId }, output: { count: out.quests.length, error: !!out.error }, status: error ? "fail" : "ok" });
  return out;
}

// ── getQuestDetails ──────────────────────────────────────────────────────────

/**
 * Reads one quest + its objectives (ordered by n) + deliverables + quest_comments.
 * Returns { quest, objectives, deliverables, comments } or { error }.
 */
export async function getQuestDetails({ questId } = {}) {
  if (!questId) {
    const out = { error: "questId required" };
    logWeaponCall({ weapon: "questmaster", fn: "getQuestDetails", input: {}, output: out, status: "fail" });
    return out;
  }
  const client = await getServiceClient();
  const { data: quest, error: qErr } = await client
    .from(publicTables.quests)
    .select("id, title, description, stage, assignee, owner_id, worker_id, created_at")
    .eq("id", questId)
    .single();
  if (qErr) {
    const out = { error: qErr.message };
    logWeaponCall({ weapon: "questmaster", fn: "getQuestDetails", input: { questId }, output: { error: qErr.message }, status: "fail" });
    return out;
  }

  const { data: objectives, error: sErr } = await client
    .from("objectives")
    .select("id, n, name, status, progress, delta, notes, created_at, updated_at")
    .eq("quest_id", questId)
    .order("n", { ascending: true });

  const { data: deliverables, error: iErr } = await client
    .from("deliverables")
    .select("id, name, kind, ref, status, verify_screenshot_ref, created_at, updated_at")
    .eq("quest_id", questId);

  const { data: comments, error: cErr } = await client
    .from(publicTables.questComments)
    .select("id, summary, actor_name, source, action, verdict, resolved, created_at")
    .eq("quest_id", questId)
    .order("created_at", { ascending: false });

  // quest_assets — NON-FATAL: this submodule is additive, so a missing table /
  // schema-cache miss must NOT promote to the top-level `error` (that would
  // notFound() the whole quest page over an optional section). Degrade to [].
  const { data: assets, error: aErr } = await client
    .from(publicTables.questAssets)
    .select("id, label, ref, kind, source, actor_name, resolved, created_at, updated_at")
    .eq("quest_id", questId)
    .order("created_at", { ascending: true });

  // quest_questions — NON-FATAL (additive submodule; renders above objectives).
  const { data: questions, error: qqErr } = await client
    .from(publicTables.questQuestions)
    .select("id, qtype, direction, body, answer, answered, unblocks_objective_ids, actor_name, created_at, updated_at")
    .eq("quest_id", questId)
    .order("created_at", { ascending: true });

  // A sub-query that genuinely errors (PostgREST column drift, transient
  // schema-cache miss, RLS) must NOT masquerade as a successful empty array.
  // The old `objectives: sErr ? [] : objectives` shape silently zeroed
  // objectives/deliverables while stashing the message in
  // objectivesError/deliverablesError — fields no caller reads:
  // the quest page checks only `details.error` (so it rendered an empty page
  // with zero signal instead of notFound()-ing), and a raw-table read of the
  // same rows still succeeded — the exact "rows demonstrably exist but
  // getQuestDetails returns []" symptom that forced a leaf to bypass this
  // weapon. This is the same silent-swallow class as the body→summary fixes
  // (0e4a420 / b902d2a), which only patched the specific column name and left
  // the masking mechanism in place. questExecution.submit already treats
  // details.error / details.deliverablesError as a verification FAILURE to fix
  // (the page's own data source must load), and the SE locked rule mandates the
  // read path fail LOUD, not empty. So promote any sub-query error to the
  // top-level `error` every caller already checks (page → notFound,
  // searchQuestBySimilarName → detailErr, questExecution → details.error),
  // while keeping the per-field *Error diagnostics and the unchanged
  // {quest, objectives, deliverables, comments} shape. No stage gate exists or
  // is added — this returns real objectives/deliverables/comments regardless of
  // quest.stage.
  const subError = sErr?.message
    ? `objectives query failed: ${sErr.message}`
    : iErr?.message
      ? `deliverables query failed: ${iErr.message}`
      : cErr?.message
        ? `comments query failed: ${cErr.message}`
        : null;

  const out = {
    quest,
    objectives: sErr ? [] : objectives || [],
    objectivesError: sErr?.message,
    deliverables: iErr ? [] : deliverables || [],
    deliverablesError: iErr?.message,
    comments: cErr ? [] : comments || [],
    commentsError: cErr?.message,
    // assets is additive + non-fatal: keep its error in assetsError only, never
    // fold it into the top-level `error` that the page treats as notFound().
    assets: aErr ? [] : assets || [],
    assetsError: aErr?.message,
    questions: qqErr ? [] : questions || [],
    questionsError: qqErr?.message,
    ...(subError ? { error: subError } : {}),
  };
  logWeaponCall({
    weapon: "questmaster",
    fn: "getQuestDetails",
    input: { questId },
    output: {
      hasQuest: !!quest,
      objectiveCount: out.objectives.length,
      deliverableCount: out.deliverables.length,
      commentCount: out.comments.length,
      assetCount: out.assets.length,
      ...(subError ? { error: subError } : {}),
    },
    status: subError ? "fail" : "ok",
  });
  return out;
}

// ── summarizeQuest ───────────────────────────────────────────────────────────

/**
 * Rapid, concise quest digest so a worker can orient in ONE call instead of
 * raw-reading every objective / comment / question to figure out "where is this
 * quest and what do I do next." One deterministic pass over getQuestDetails (NO
 * LLM, no extra round-trips): counts objectives by status, names the next
 * actionable objective, flags OPEN questions + ANSWERED-but-not-digested ones,
 * deliverable fill ratio, the latest cold-legible note, and the lease holder.
 * Returns a compact `digest` string an agent reads in one shot, plus structured
 * fields for programmatic use. This is the orientation entry point — call it
 * first; only deep-read (getQuestDetails) the specific rows the digest flags.
 *
 * @param {{ questId:string, maxNoteLen?:number }} opts
 * @returns {Promise<
 *   | { ok:true, questId:string, title:string, stage:string, assignee:string|null, workerId:string|null, url:string,
 *       objectives:object, deliverables:object, questions:object, latestNote:object|null, digest:string }
 *   | { ok:false, error:string }
 * >}
 */
export async function summarizeQuest({ questId, maxNoteLen = 240 } = {}) {
  if (!questId) {
    const out = { ok: false, error: "questId required" };
    logWeaponCall({ weapon: "questmaster", fn: "summarizeQuest", input: {}, output: out, status: "fail" });
    return out;
  }
  const details = await getQuestDetails({ questId });
  if (details.error || !details.quest) {
    const out = { ok: false, error: details.error || "quest not found" };
    logWeaponCall({ weapon: "questmaster", fn: "summarizeQuest", input: { questId }, output: out, status: "fail" });
    return out;
  }
  const q = details.quest;
  const objectives = details.objectives || [];
  const deliverables = details.deliverables || [];
  const assets = details.assets || [];
  const questions = details.questions || [];
  const comments = details.comments || []; // desc by created_at

  const byStatus = (s) => objectives.filter((o) => o.status === s).length;
  const counts = {
    total: objectives.length,
    done: byStatus("done"),
    running: byStatus("running"),
    pending: byStatus("pending"),
    review: byStatus("review"),
  };
  // Next actionable objective: lowest-n not done (running first, else pending).
  const active = objectives.filter((o) => o.status !== "done");
  const next =
    active.find((o) => o.status === "running") || active.find((o) => o.status === "pending") || active[0] || null;

  const hasRef = (r) => !!(r && String(r).trim());
  const filled = deliverables.filter((d) => hasRef(d.ref)).length;
  // Brief index (point 4 refinement): names + has-details status only, NO bodies.
  // The leaf transcript showed workers re-querying deliverables/assets/storage
  // just to learn "what exists and is it filled" — this one-line index kills that
  // without bloating context. Bodies stay on-demand / read at completion.
  const cap = (arr, n) => (arr.length > n ? arr.slice(0, n).concat([`+${arr.length - n} more`]) : arr);
  const deliverableIndex = deliverables.map((d) => `${d.name}${d.kind ? `(${d.kind})` : ""} [${hasRef(d.ref) ? "filled" : "empty"}]`);
  // Deliverables are FIRST-CLASS executables, not derived from objectives — an
  // empty-ref deliverable is unfinished work even when every objective is 'done'
  // (objectives are the plan/scratchpad; deliverables are what the quest ships and
  // what purrview judges). Surface the unfilled ones LOUDLY so the worker produces
  // them instead of reading "all objectives done" and stopping.
  const unfilledDeliverables = deliverables.filter((d) => !hasRef(d.ref));
  const assetIndex = assets.map((a) => `${a.label || a.kind || "asset"} [${hasRef(a.ref) ? "ref" : "—"}]`);
  const openQuestions = questions.filter((x) => !x.answered);
  const answeredPending = questions.filter((x) => x.answered); // still present = not yet digested

  // Asset-checkpoint flag (point 2): a quick deterministic pass over the ACTIVE
  // objective names — the ones that "produce a durable artifact" should
  // save-forward as a quest_asset (ruleCheckpointAssets) so a later objective
  // can consume it. Heuristic (verb × artifact-noun), upgradeable to an LLM pass.
  const ASSET_VERB = /\b(generat|creat|produc|render|writ|build|assembl|draw|design|export|compil|synthesi|forge|render)/i;
  const ASSET_NOUN = /\b(image|pdf|file|dataset|asset|doc|chart|deck|slide|video|book|page|report|diagram|table|json|csv|markdown|graphic|figure|mockup|screenshot|artifact|illustration|audio)/i;
  const assetCheckpoints = objectives
    .filter((o) => o.status !== "done" && ASSET_VERB.test(o.name || "") && ASSET_NOUN.test(o.name || ""))
    .map((o) => ({ n: o.n, name: o.name }));

  const trunc = (s, n) => {
    const t = (s || "").toString().trim();
    return t.length > n ? t.slice(0, n) + "…" : t;
  };

  // ── Curated worker read-surface (replaces the old "latest note + compact flag").
  // The full thread is the durable, IMMUTABLE audit log (getQuestDetails returns
  // every row). The DIGEST is the worker's WORKING SET: surface ONLY comments a
  // worker must act on, so the slice it reads approaches 100% signal. We classify
  // on the existing source/action (no schema change) and DELIBERATELY HIDE system
  // state-machine telemetry (derail_kickback, lease, unstuck, *_digested) — that
  // is debugging history, not worker guidance. Nothing is deleted; noise is
  // filtered OUT of the read, never OUT of the record. Supersedes the phantom
  // `compactComments` flag, which named a non-existent, audit-destroying action.
  const isVerify = (c) => /verif|verdict|bounce/i.test(c.action || "");
  const isHandoff = (c) => c.source === "worker" || c.source === "chaperon" || /^c?leaf-/.test(c.source || "");
  const userDirectives = comments.filter((c) => c.source === "user" && !c.resolved); // unresolved user instructions — ALL
  // Open verification fails. Classify on the STRUCTURED verdict field (set by
  // recordVerifyPass + the questPurrview/questReview gates) — NOT prose. The old
  // path regex-sniffed "fail"/"bounce" out of the summary, which mislabeled a
  // comment that merely mentioned the words. The field is authoritative:
  // verdict='fail' is a fail, verdict='pass' never is. The prose regex survives
  // ONLY as a fallback for legacy rows written before the verdict column existed
  // (verdict === null).
  const openVerifyFails = comments.filter((c) => {
    if (c.resolved) return false;
    if (c.verdict === "fail") return true;
    if (c.verdict === "pass") return false;
    return isVerify(c) && /\bfail|bounce/i.test(`${c.action} ${c.summary}`); // legacy (no verdict) fallback
  });
  const latestHandoff = comments.find(isHandoff) || null; // newest worker/leaf/chaperon note (comments are desc by created_at)
  // Foundational origin: the EARLIEST chaperon hand-off (the quest's "what landed"
  // seed), which otherwise ages out of any latest-N window as the thread grows.
  const foundational = [...comments].reverse().find((c) => c.source === "chaperon" || c.action === "deliver") || null;
  const hiddenTelemetry = comments.filter((c) => c.source === "system").length;

  const lines = [];
  lines.push(`Quest "${q.title}" — stage=${q.stage}, assignee=${q.assignee || "—"}, worker=${q.worker_id || "none"}.`);
  if (q.description) lines.push(`Goal: ${trunc(q.description, 300)}`);
  lines.push(
    `Objectives: ${counts.done}/${counts.total} done` +
      (counts.running ? `, ${counts.running} running` : "") +
      `, ${counts.pending} pending.`,
  );
  if (next) {
    lines.push(`Next (n=${next.n}, ${next.status}): ${next.name}`);
  } else if (unfilledDeliverables.length) {
    lines.push(
      `Next: all objectives done, BUT ${unfilledDeliverables.length} deliverable(s) still EMPTY — PRODUCE them now (they are executables, not derived from objectives): ${cap(unfilledDeliverables.map((d) => `"${d.name}"${d.kind ? `(${d.kind})` : ""}`), 12).join(", ")}. The quest is NOT done until every deliverable has a ref — do not stop or escalate.`,
    );
  } else {
    lines.push("Next: none — all objectives done and all deliverables filled.");
  }
  // Surface the next objective's notes — this carries the plan-time "Gear:"
  // registry scan (planQuest), so the leaf needn't re-run gearUp or deep-read
  // the objective row just to learn which skill books/weapons to use.
  if (next && next.notes && String(next.notes).trim()) lines.push(`Next-objective notes: ${trunc(next.notes, 240)}`);
  if (assetCheckpoints.length)
    lines.push(
      `⚑ Save-forward (these active objectives likely produce a durable artifact — archive as a quest_asset): obj ${assetCheckpoints.map((o) => o.n).join(", ")}`,
    );
  lines.push(
    `Deliverables (${filled}/${deliverables.length} filled): ${deliverables.length ? cap(deliverableIndex, 12).join(", ") : "none"} — names+status only; bodies on demand.`,
  );
  if (unfilledDeliverables.length)
    lines.push(
      `⚠ ${unfilledDeliverables.length} DELIVERABLE(S) NOT PRODUCED (ref empty) — UNFINISHED WORK regardless of objective status: ${cap(unfilledDeliverables.map((d) => `"${d.name}"${d.kind ? `(${d.kind})` : ""}`), 12).join(", ")}. Produce each now: build the artifact → worker.shipItemArtifact (upsert the deliverables row's ref) → questPurrview.autoVisualVerify. Never stop or escalate with an empty deliverable.`,
    );
  if (assets.length) lines.push(`Assets (${assets.length}): ${cap(assetIndex, 12).join(", ")} — names only; fetch a body on demand.`);
  if (openQuestions.length)
    lines.push(
      `OPEN questions (${openQuestions.length}) — answer/escalate FIRST: ${openQuestions.map((x) => `"${trunc(x.body, 100)}"`).join(" | ")}`,
    );
  if (answeredPending.length)
    lines.push(
      `ANSWERED-not-digested (${answeredPending.length}) — digest into objectives then remove: ${answeredPending.map((x) => `"${trunc(x.body, 70)}" → "${trunc(x.answer, 70)}"`).join(" | ")}`,
    );
  if (userDirectives.length)
    lines.push(
      `USER directives — unresolved (${userDirectives.length}, MUST address): ${userDirectives
        .map((c) => `[${c.action}${c.actor_name ? ` by ${c.actor_name}` : ""}] ${trunc(c.summary, 140)}`)
        .join(" | ")}`,
    );
  if (openVerifyFails.length)
    lines.push(`OPEN verification fails (${openVerifyFails.length}): ${openVerifyFails.map((c) => trunc(c.summary, 120)).join(" | ")}`);
  if (latestHandoff)
    lines.push(
      `Latest hand-off (${latestHandoff.source}/${latestHandoff.action}${latestHandoff.actor_name ? ` by ${latestHandoff.actor_name}` : ""}): ${trunc(latestHandoff.summary, maxNoteLen)}`,
    );
  if (foundational && foundational.id !== latestHandoff?.id)
    lines.push(`Origin (${foundational.source}/${foundational.action}): ${trunc(foundational.summary, 160)}`);
  if (hiddenTelemetry)
    lines.push(`(${hiddenTelemetry} system/telemetry comment${hiddenTelemetry === 1 ? "" : "s"} hidden from read — in audit log only.)`);

  const out = {
    ok: true,
    questId,
    title: q.title,
    stage: q.stage,
    assignee: q.assignee || null,
    workerId: q.worker_id || null,
    url: questPath(questId),
    objectives: { ...counts, next: next ? { n: next.n, name: next.name, status: next.status } : null, assetCheckpoints },
    comments: {
      total: comments.length,
      userDirectivesOpen: userDirectives.length,
      openVerifyFails: openVerifyFails.length,
      hiddenTelemetry,
    },
    deliverables: {
      total: deliverables.length,
      filled,
      index: deliverables.map((d) => ({ name: d.name, kind: d.kind, filled: hasRef(d.ref) })),
    },
    assets: { total: assets.length, index: assets.map((a) => ({ label: a.label, kind: a.kind, hasRef: hasRef(a.ref) })) },
    questions: {
      open: openQuestions.map((x) => ({ id: x.id, body: x.body })),
      answeredPending: answeredPending.map((x) => ({ id: x.id, body: x.body, answer: x.answer })),
    },
    latestNote: latestHandoff
      ? { source: latestHandoff.source, action: latestHandoff.action, actorName: latestHandoff.actor_name, summary: latestHandoff.summary, createdAt: latestHandoff.created_at }
      : null,
    digest: lines.join("\n"),
  };
  logWeaponCall({
    weapon: "questmaster",
    fn: "summarizeQuest",
    input: { questId },
    output: { stage: q.stage, objectivesDone: counts.done, total: counts.total, openQuestions: openQuestions.length },
    status: "ok",
  });
  return out;
}

// ── recordVerifyPass ─────────────────────────────────────────────────────────

/**
 * Record ONE structured visual-verification comment for a whole purrview/review
 * pass, UPSERT'd (replace-in-place).
 *
 * The verify gate used to write one quest_comments row PER deliverable (1 roll-up
 * + N findings); a second racing leaf doubled it — the #2 source of comment bloat
 * (10 near-identical rows on quest 988c9792). This collapses a whole pass into a
 * SINGLE row: per-deliverable findings in detail.findings (machine-readable), a
 * compact roll-up in the summary (human-readable), and the overall pass/fail in
 * the top-level `verdict` column (what summarizeQuest.openVerifyFails classifies
 * on). A re-run by another leaf (or a re-verify after rework) REPLACES the row in
 * place — keyed on quest_id + source='verify' + action='visual_verify' + passType
 * — never appends, and never DELETES (the durable pass/fail audit lives on the
 * questPurrview/questReview lockphrase comments, which this does not touch).
 *
 * Anti-rubber-stamp preserved: every finding must carry a non-empty `depicts`
 * (what is actually on screen) and `readback` (a verbatim on-screen quote) — the
 * proof the verifier looked at real pixels. A finding missing either is rejected.
 *
 * @param {{
 *   questId: string,
 *   passType?: 'purrview'|'review',
 *   workerId?: string|null,
 *   findings: Array<{ deliverable: string, depicts: string, readback: string,
 *                     verdict: 'match'|'mismatch', probes?: string, mismatch?: string }>,
 *   note?: string,
 * }} args
 * @returns {Promise<{ ok:true, mode:'inserted'|'replaced', verdict:'pass'|'fail', deliverables:number, commentId:string|null } | { ok:false, error:string }>}
 */
export async function recordVerifyPass({ questId, passType = "purrview", workerId = null, findings, note = "" } = {}) {
  if (!questId || typeof questId !== "string") {
    const out = { ok: false, error: "questId required" };
    logWeaponCall({ weapon: "questmaster", fn: "recordVerifyPass", input: {}, output: out, status: "fail" });
    return out;
  }
  if (!Array.isArray(findings) || findings.length === 0) {
    const out = { ok: false, error: "findings required: a non-empty array of { deliverable, depicts, readback, verdict } (one per deliverable)." };
    logWeaponCall({ weapon: "questmaster", fn: "recordVerifyPass", input: { questId }, output: out, status: "fail" });
    return out;
  }
  if (passType !== "purrview" && passType !== "review") {
    return { ok: false, error: `passType must be 'purrview' or 'review' (got "${passType}").` };
  }

  // Normalize + validate. depicts + readback are MANDATORY (anti-rubber-stamp:
  // a finding with no observed content / no verbatim quote is not a real look).
  const norm = [];
  for (const f of findings) {
    if (!f || typeof f !== "object") continue;
    const deliverable = String(f.deliverable || "").trim();
    const depicts = String(f.depicts || "").trim();
    const readback = String(f.readback || "").trim();
    const verdict = f.verdict === "match" ? "match" : f.verdict === "mismatch" ? "mismatch" : null;
    if (!deliverable || !depicts || !readback || !verdict) {
      const out = {
        ok: false,
        error: `Each finding needs { deliverable, depicts (what is on screen), readback (a verbatim on-screen quote), verdict:'match'|'mismatch' }. Offending: ${JSON.stringify(f).slice(0, 200)}`,
      };
      logWeaponCall({ weapon: "questmaster", fn: "recordVerifyPass", input: { questId, passType }, output: out, status: "fail" });
      return out;
    }
    norm.push({
      deliverable,
      depicts,
      readback,
      verdict,
      probes: String(f.probes || "").trim() || null,
      mismatch: String(f.mismatch || "").trim() || null,
    });
  }
  if (!norm.length) return { ok: false, error: "no valid findings after normalization" };

  const overall = norm.some((f) => f.verdict === "mismatch") ? "fail" : "pass";
  const rollup = norm
    .map((f) => `${f.deliverable}: ${f.verdict === "match" ? "✓" : `✗ ${f.mismatch || "mismatch"}`}`)
    .join("; ");
  const summary = `[VISUAL_VERIFY:${overall}] ${passType} — ${norm.length} deliverable(s): ${rollup}${note ? ` | ${note}` : ""}${workerId ? ` (by ${workerId})` : ""}`.slice(0, 2000);
  const detail = { passType, workerId, overall_verdict: overall, findings: norm, note: note || null, generated_by: "recordVerifyPass" };

  // Write through the centralized chokepoint (libs/quest recordQuestComment),
  // which OWNS the verify-pass policy: 'visual_verify' is an UPSERT action, so a
  // second racing leaf — or a re-verify after rework — REPLACES this row in place
  // (keyed on quest_id+action) instead of appending a fresh findings set, and
  // never deletes. Routing here (vs a hand-rolled insert) keeps that dedup policy
  // in ONE place rather than re-implementing it per writer.
  const { recordQuestComment } = await import("@/libs/quest/index.js");
  const { data, error, upserted } = await recordQuestComment(questId, {
    source: "verify",
    action: "visual_verify",
    summary,
    detail,
    verdict: overall,
    actor_name: workerId || "verify-leaf",
  });
  if (error) {
    const out = { ok: false, error: error.message || String(error) };
    logWeaponCall({ weapon: "questmaster", fn: "recordVerifyPass", input: { questId, passType }, output: out, status: "fail" });
    return out;
  }

  const out = { ok: true, mode: upserted ? "replaced" : "inserted", verdict: overall, deliverables: norm.length, commentId: data?.id || null };
  logWeaponCall({ weapon: "questmaster", fn: "recordVerifyPass", input: { questId, passType, findings: norm.length }, output: { mode: out.mode, verdict: overall }, status: "ok" });
  return out;
}

// ── searchQuestBySimilarName ─────────────────────────────────────────────────

/**
 * Loads active quests and asks LLM to pick a match (or none). Double-verifies
 * a positive match by reading the matched quest's description.
 *
 * @param {{candidateName:string, smartPlan?:object, ownerId?:string}} opts
 */
export async function searchQuestBySimilarName({ candidateName, smartPlan, ownerId } = {}) {
  if (!candidateName || typeof candidateName !== "string") {
    const out = { match: null, reason: "no candidateName provided" };
    logWeaponCall({ weapon: "questmaster", fn: "searchQuestBySimilarName", input: { candidateName }, output: out, status: "fail" });
    return out;
  }

  const { quests, error } = await getActiveQuestNames({ ownerId });
  if (error) {
    const out = { match: null, reason: `getActiveQuestNames error: ${error}` };
    logWeaponCall({ weapon: "questmaster", fn: "searchQuestBySimilarName", input: { candidateName }, output: out, status: "fail" });
    return out;
  }
  if (!quests.length) {
    const out = { match: null, reason: "no active quests" };
    logWeaponCall({ weapon: "questmaster", fn: "searchQuestBySimilarName", input: { candidateName }, output: out, status: "ok" });
    return out;
  }

  const promptTemplate =
    questmasterSkillBook?.toc?.searchQuestBySimilarName?.howTo ||
    "Pick the active quest whose title most likely refers to the same objective as the candidate. Return JSON.";

  const messages = [
    { role: "system", content: "You are matching a candidate quest name against a list of active quests. Return ONLY valid JSON." },
    {
      role: "user",
      content: [
        "## Skill book howTo (procedure):",
        promptTemplate,
        "",
        "## Candidate name",
        candidateName,
        "",
        "## SMART plan summary (optional context)",
        smartPlan ? JSON.stringify(smartPlan).slice(0, 1500) : "(none)",
        "",
        "## Active quests",
        JSON.stringify(quests, null, 2),
        "",
        "## Output schema",
        '{ "match": "<questId or null>", "reason": "<one-line rationale>" }',
      ].join("\n"),
    },
  ];

  let parsed = null;
  try {
    const res = await runGenericChat({ messages });
    parsed = parseJsonFromText(res.text);
  } catch (e) {
    const out = { match: null, reason: `LLM error: ${e?.message || String(e)}` };
    logWeaponCall({ weapon: "questmaster", fn: "searchQuestBySimilarName", input: { candidateName, questCount: quests.length }, output: out, status: "fail" });
    return out;
  }

  if (!parsed || typeof parsed !== "object") {
    const out = { match: null, reason: "unparseable LLM response" };
    logWeaponCall({ weapon: "questmaster", fn: "searchQuestBySimilarName", input: { candidateName, questCount: quests.length }, output: out, status: "fail" });
    return out;
  }

  const rawMatch = typeof parsed.match === "string" ? parsed.match.trim() : null;
  if (!rawMatch || rawMatch.toLowerCase() === "null" || rawMatch === "") {
    const out = { match: null, reason: parsed.reason || "no match" };
    logWeaponCall({ weapon: "questmaster", fn: "searchQuestBySimilarName", input: { candidateName, questCount: quests.length }, output: { match: null }, status: "ok" });
    return out;
  }

  // Double-verify by reading the matched quest's description
  const { quest, error: detailErr } = await getQuestDetails({ questId: rawMatch });
  if (detailErr || !quest) {
    const out = { match: null, reason: "match returned by LLM not found in DB" };
    logWeaponCall({ weapon: "questmaster", fn: "searchQuestBySimilarName", input: { candidateName, llmMatch: rawMatch }, output: out, status: "fail" });
    return out;
  }

  const out = { match: rawMatch, reason: parsed.reason || "LLM matched", verifiedTitle: quest.title };
  logWeaponCall({ weapon: "questmaster", fn: "searchQuestBySimilarName", input: { candidateName, questCount: quests.length }, output: { match: rawMatch, verifiedTitle: quest.title }, status: "ok" });
  return out;
}

// ── createQuest ──────────────────────────────────────────────────────────────

/**
 * DB write under the new schema. Quests are BORN AT stage='plan' (2026-06-12):
 * the canonical creation input is ownerId + title + a BIG description (the
 * user's ask, summarized but not yet itemized) — NO objectives, NO
 * deliverables. Cron:questTick then deploys a planning agent (planQuest) that
 * itemizes the description into objectives + deliverables + questions and
 * transitions the quest to execute.
 *
 *   1. Insert into quests with stage='plan' and the assignee column populated
 *      (defaults to "Guildmaster" — the local-run executor that
 *      questTick scans for — when the caller omits it; a quest with
 *      no executor would never be picked up off the bus).
 *   2. (optional) Insert one row into `objectives` per `objectives[i]`
 *      (preserving the ordering via the `n` column) — for callers that arrive
 *      pre-planned.
 *   3. (optional) Insert one row into `deliverables` per `deliverables[i]`.
 *   4. If objectives were provided, attempt moveQuestToExecute immediately
 *      (planning is already done); the guard keeps it in 'plan' if deliverables
 *      or assignee are still missing, and the planning tick fills the gaps.
 *
 * @param {{
 *   ownerId: string,
 *   title: string,
 *   description: string,
 *   assignee?: string,
 *   objectives?: Array<{
 *     n?: number, name: string, status?: string,
 *     input?: string, method?: string,
 *     output: Array<{ asset:string, format:string, display?:string|null,
 *                     verification?:string, verification_example?:any }>,
 *     viewResults?: "Show"|"Checkpoint"|"No",
 *     outputKey?: string|null,
 *     notes?: string
 *   }>,
 *   deliverables?: Array<{ name:string, kind:"text"|"html"|"image"|"video"|"document", ref?:string|null }>
 * }} opts
 */
export async function createQuest({ ownerId, title, description, assignee, objectives, deliverables } = {}) {
  const hasObjectives = Array.isArray(objectives) && objectives.length > 0;
  // Objectives are now OPTIONAL: the canonical path is description-only creation
  // at stage='plan' (planQuest itemizes later). A quest with NEITHER a
  // description NOR objectives carries nothing for the planner to itemize.
  if (!ownerId || !title || (!hasObjectives && !(typeof description === "string" && description.trim()))) {
    const out = { error: "ownerId, title, and a description (or pre-planned objectives[]) are required" };
    logWeaponCall({
      weapon: "questmaster",
      fn: "createQuest",
      input: { hasOwner: !!ownerId, hasTitle: !!title, objectiveCount: objectives?.length || 0 },
      output: out,
      status: "fail",
    });
    return out;
  }

  const client = await getServiceClient();

  // Roster gate: createQuest writes the LIVE `assignee` column directly, so this
  // is the real phantom-injection point (a name with no adventurer row — e.g. a
  // mislabelled "Operator_SIO" — lands here, is never dispatched by any caravan
  // class, and is never rescued by assignOrphans → it stalls invisibly forever).
  // resolveAssignee resolves the name GLOBALLY (the correct dispatch scope); an
  // `unassigned` result is a phantom → reject before insert. No auto-provision
  // (minting a row from an arbitrary string would pollute the dispatchable pool).
  // The default "Main_Claude" always resolves (a seeded worker).
  const assigneeName = assignee || "Main_Claude";
  const resolvedAssignee = await resolveAssignee(assigneeName, client);
  if (resolvedAssignee.type !== "adventurer") {
    const out = { error: `No adventurer named "${assigneeName}" on your roster` };
    logWeaponCall({ weapon: "questmaster", fn: "createQuest", input: { title, assignee: assigneeName }, output: out, status: "fail" });
    return out;
  }

  // 1. Quest row — born at stage='plan'. The planning tick (questTick
  // dispatch-plan → planQuest) itemizes the description and transitions to
  // execute; pre-planned quests (objectives provided) attempt the transition
  // immediately below.
  const questPayload = {
    owner_id: ownerId,
    title,
    description: description || "",
    stage: "plan",
    assignee: resolvedAssignee.profile.name,
  };

  const { data: questRow, error: qErr } = await client
    .from(publicTables.quests)
    .insert(questPayload)
    .select("id")
    .single();

  if (qErr || !questRow?.id) {
    const out = { error: qErr?.message || "quest insert returned no id" };
    logWeaponCall({ weapon: "questmaster", fn: "createQuest", input: { title }, output: out, status: "fail" });
    return out;
  }

  const questId = questRow.id;

  // 2. Objectives rows (optional — empty on the canonical description-only path)
  const objectiveRows = (Array.isArray(objectives) ? objectives : [])
    .filter((s) => s && typeof s.name === "string" && s.name.trim())
    .map((s, i) => ({
      quest_id: questId,
      n: typeof s.n === "number" ? s.n : i + 1,
      name: s.name.trim(),
      status: s.status && ["pending", "done", "review"].includes(s.status) ? s.status : "pending",
      notes: typeof s.notes === "string" ? s.notes : "",
    }));

  let objectiveInsertError = null;
  if (objectiveRows.length > 0) {
    const { error: sErr } = await client.from("objectives").insert(objectiveRows);
    if (sErr) objectiveInsertError = sErr.message;
  }

  // 3. Deliverables rows (initial inputs only; downstream execution populates the rest)
  const deliverableRows = Array.isArray(deliverables)
    ? deliverables
        .filter(
          (d) =>
            d &&
            typeof d.name === "string" &&
            d.name.trim() &&
            typeof d.kind === "string" &&
            ["text", "html", "image", "video", "document"].includes(d.kind),
        )
        .map((d) => ({
          quest_id: questId,
          name: d.name.trim(),
          kind: d.kind,
          ref: typeof d.ref === "string" ? d.ref : null,
        }))
    : [];

  let deliverableInsertError = null;
  if (deliverableRows.length > 0) {
    const { error: iErr } = await client.from("deliverables").insert(deliverableRows);
    if (iErr) deliverableInsertError = iErr.message;
  }

  // 4. Pre-planned quests (objectives provided) skip the planning tick: attempt
  // the plan→execute transition now. The guard (>=1 objective, >=1 deliverable,
  // assignee) keeps the quest in 'plan' when something is still missing — the
  // planning tick then fills the gaps (planQuest is idempotent on existing
  // objectives and falls through to moveQuestToExecute).
  let transition = null;
  if (objectiveRows.length > 0 && !objectiveInsertError) {
    transition = await moveQuestToExecute({ questId });
  }

  const out = {
    questId,
    url: questPath(questId),
    stage: transition?.ok ? transition.stage : "plan",
    objectiveCount: objectiveRows.length,
    deliverableCount: deliverableRows.length,
    ...(transition ? { transition } : {}),
    ...(objectiveInsertError ? { objectivesError: objectiveInsertError } : {}),
    ...(deliverableInsertError ? { deliverablesError: deliverableInsertError } : {}),
  };
  logWeaponCall({
    weapon: "questmaster",
    fn: "createQuest",
    input: { title, objectiveCount: objectiveRows.length, deliverableCount: deliverableRows.length },
    output: { questId, objectiveCount: objectiveRows.length, deliverableCount: deliverableRows.length, objectiveInsertError, deliverableInsertError },
    status: objectiveInsertError || deliverableInsertError ? "fail" : "ok",
  });
  return out;
}

// ── planQuest ────────────────────────────────────────────────────────────────

// Matches an "AUTONOMY / PRE-APPROVED DECISIONS:" header line plus its contiguous
// body (every following non-empty line, up to the first blank line or EOF).
// Tolerates casing and the separator between the two words (/, &, "and", or none).
const AUTONOMY_BLOCK_RE = /^[^\n]*AUTONOMY[^\n]*PRE-?APPROVED[^\n]*\n(?:[^\n]+\n?)*/im;

/** The operator's grant-of-latitude block, extracted verbatim (or "" if absent). */
function extractAutonomyBlock(text) {
  if (!text || typeof text !== "string") return "";
  const m = text.match(AUTONOMY_BLOCK_RE);
  return m ? m[0].trim() : "";
}

/**
 * Distill a plan-stage quest's verbose description into:
 *   - a concise description (global instructions only),
 *   - objectives rows (the materialised plan),
 *   - deliverables rows (final artifacts interpreted from the description),
 *   - quest_questions rows (ambiguities the user must resolve).
 *
 * Anything moved into a sub-item is REMOVED from the description — the
 * done-condition is "description short AND non-overlapping with the sub-items".
 * EXCEPTION: an "AUTONOMY / PRE-APPROVED DECISIONS:" block is preserved verbatim
 * (never folded into objectives) so no later stage re-asks a delegated decision.
 *
 * Idempotent PER RESOURCE (re-plan safe): objectives/questions/deliverables are
 * written ONLY when their table is empty for this quest (never duplicated), but
 * the description-shrink ALWAYS runs — so re-planning a quest that already has
 * objectives keeps those canonical rows and merely strips the description's
 * overlap with them. On success, calls moveQuestToExecute.
 *
 * @param {{ questId: string }} opts
 */
export async function planQuest({ questId } = {}) {
  if (!questId) {
    const out = { ok: false, error: "questId is required" };
    logWeaponCall({ weapon: "questmaster", fn: "planQuest", input: { questId }, output: out, status: "fail" });
    return out;
  }
  const client = await getServiceClient();

  const { data: quest, error: qErr } = await client
    .from(publicTables.quests)
    .select("id, title, description, stage, assignee, deliverables")
    .eq("id", questId)
    .single();
  if (qErr || !quest) {
    const out = { ok: false, error: qErr?.message || "quest not found" };
    logWeaponCall({ weapon: "questmaster", fn: "planQuest", input: { questId }, output: out, status: "fail" });
    return out;
  }
  if (quest.stage !== "plan") {
    const out = { ok: false, error: `quest stage is '${quest.stage}', expected 'plan'` };
    logWeaponCall({ weapon: "questmaster", fn: "planQuest", input: { questId }, output: out, status: "fail" });
    return out;
  }

  // Per-resource idempotency (FIXED 2026-06-13). The old guard returned early
  // whenever ANY objective existed and jumped straight to moveQuestToExecute —
  // which ABANDONED the rest of the done-condition: the verbose description was
  // never shrunk (so its inline objective-dump kept overlapping the objectives
  // table) and missing deliverables were never interpreted (so the transition
  // silently failed the deliverables guard and the quest stalled in plan with a
  // redundant description forever). Planning's done-condition is "description
  // short AND non-overlapping with the sub-items" — that work must run EVERY
  // time, even on a re-plan. So we no longer early-return; instead each WRITE is
  // guarded by whether that resource already exists: objectives/questions are
  // inserted only when their table is empty for this quest (never duplicated),
  // while the description-shrink + deliverable-interpretation ALWAYS run.
  const { data: existingObjectives } = await client
    .from(publicTables.objectives)
    .select("id, n, name")
    .eq("quest_id", questId)
    .order("n", { ascending: true });
  const { data: existingQuestions } = await client
    .from(publicTables.questQuestions)
    .select("id")
    .eq("quest_id", questId);
  const hasObjectives = (existingObjectives || []).length > 0;
  const hasQuestions = (existingQuestions || []).length > 0;

  // Use LLM to decompose the description into objectives + questions + concise description
  const systemPrompt = `You are a quest planner. Given a quest title, description, and deliverables, produce a JSON object with:
1. "conciseDescription" — the description stripped of per-objective detail (keep only global instructions, links, and references). CRITICAL: preserve VERBATIM any "AUTONOMY / PRE-APPROVED DECISIONS:" block (its header line and every bullet under it) — these are the operator's grants of latitude and MUST survive the shrink so no later stage re-asks a decision the user already delegated. Never fold them into objectives, never reword them.
2. "objectives" — an array of { "name": "<verb-led, future-focused, one action>" } objects. Each objective should be a discrete, actionable work item. Order them by dependency (lowest n = do first).
3. "questions" — an array of { "body": "<question text>", "unblocks": [<objective indices, 0-based>] } for ONLY the ambiguities that are genuinely IRREVERSIBLE or business-damaging to guess wrong. Apply the sage question-gate test: PROCEED (pick a sensible default, no question) on anything reversible; ESCALATE (ask) only what cannot be cheaply undone. Before adding ANY question, DROP it if any of these match:
   (a) EXPLICIT DELEGATION — the description hands the decision to the agent in any form ("agent's discretion", "you decide", "up to you", "your call", "default is fine", or a stated RANGE like "5–10 chapters"). Do NOT ask; choose a value within the latitude given and proceed. (A stated range is permission to pick, never a question.)
   (b) DISCOVERABLE/INFERABLE — the answer is in the description or can be reasonably inferred from it.
   (c) REVERSIBLE / LOW-STAKES — the choice is cheap to redo: layout, page size/format, palette, character styling, spread-vs-single-page, count within a stated range, naming, ordering, formatting, tone within the stated audience. Do NOT ask; pick a defensible default and record it in the relevant objective's notes instead of asking.
   Default to making the call, not asking. A well-specified brief should usually yield ZERO questions; only a costly-and-irreversible-if-wrong decision (publishing target, legal/brand/IP constraint, real spend, an external commitment) earns one.
   STRUCTURE (required for every question that survives the filter — this is the questmaster skill book's "raiseUnblockingQuestionDiplomatically" standard): the "body" MUST be two explicit, labelled parts — (1) "WHAT YOU NEED TO DO:" the concrete decision/answer the user must give, with the exact options and the agent's RECOMMENDED default; and (2) "WHY I CAN'T DO THIS:" one sentence naming the SPECIFIC irreversibility or user-only knowledge that stops the agent from deciding (if you can't write a crisp, non-circular reason the USER is required, the question failed the filter — drop it). Lay the two parts out as two NUMBERED parts separated by a blank line (a double newline) for readability — "1. WHAT YOU NEED TO DO: ..." then an empty line then "2. WHY I CAN'T DO THIS: ...". A bare "Which X?" with no WHY, a vague "how should we proceed?" with no concrete ask, or the two parts run together without the blank-line gap, is malformed — do not emit it.
4. "deliverables" — an array of { "name": "<short noun phrase>", "kind": "<one of: text|html|image|video|document>", "ref": null } — the FINAL ARTIFACTS the quest must produce, interpreted from the description. The "kind" is HOW A HUMAN CONSUMES the artifact (not the file format): text (read it — prose/markdown/json/xml/code/a link), html (view a rendered web page/site), image (view a picture/screenshot/chart/diagram), video (watch it), document (flip through an embedded paged file — pdf/ppt/office). CRITICAL: pick kind by what the artifact IS. "a screenshot" / "a diagram" / "a chart" → image. "a PDF" / "a slide deck" → document. "a report" / "a summary" / "a JSON file" → text. "a landing page" / "a live page" → html. "a demo video" → video. Each distinct artifact is its own row (do not bundle). ref is always null at plan time (the worker fills it).

Rules:
- Objective names start with a verb, are future-focused, and describe one action each.
- Do NOT include objectives for things already done or for the deliverables themselves (deliverables track output; objectives track work).
- Return ONLY valid JSON, no markdown fences.`;

  // On a RE-PLAN the objectives table is already the canonical work list. Tell
  // the planner so it (a) does NOT invent a competing objective set and (b)
  // produces a conciseDescription that STRIPS every detail already captured by
  // those existing objectives — the whole point of the re-plan is to stop the
  // description from overlapping the sub-items.
  const existingObjectiveBlock = hasObjectives
    ? `\n\nRE-PLAN MODE. This quest ALREADY has the canonical objectives below — do NOT re-create or renumber them (your "objectives" array is ignored on this pass; leave it []). But you MUST still produce the OTHER fields fully:\n- "conciseDescription": rewrite the description to REMOVE every per-objective detail these objectives already cover, keeping only global instructions/links/references.\n- "deliverables": STILL REQUIRED — interpret the FINAL ARTIFACTS from the description's definition-of-done / goal (e.g. a "homepage renders at <url>" done-condition → an html deliverable; a "deploy a module/script" → its artifact). Do not return an empty deliverables array just because objectives exist.\n- "questions": only genuinely load-bearing ambiguities.\nExisting objectives:\n${(existingObjectives || []).map((o) => `  ${o.n}. ${o.name}`).join("\n")}`
    : "";

  const userPrompt = `Quest: ${quest.title}

Description:
${quest.description || "(empty)"}

Deliverables: ${JSON.stringify(quest.deliverables || [])}${existingObjectiveBlock}`;

  let plan;
  try {
    // runGenericChat takes { messages:[{role,content}] } and returns { text }.
    const llmResult = await runGenericChat({
      messages: [
        { role: "system", content: systemPrompt },
        { role: "user", content: userPrompt },
      ],
    });
    const raw = String(llmResult?.text || "")
      .trim()
      .replace(/^```(?:json)?\s*/i, "")
      .replace(/\s*```$/i, "")
      .trim();
    plan = JSON.parse(raw);
  } catch (err) {
    const out = { ok: false, error: `LLM planning failed: ${err.message}` };
    logWeaponCall({ weapon: "questmaster", fn: "planQuest", input: { questId }, output: out, status: "fail" });
    return out;
  }

  // Plan-time registry scan — the skill-book/weapon research is decided ONCE
  // here, not re-run by every leaf. gearUp is the deterministic token scorer
  // (no LLM, in-memory), so a per-objective call is free. The top matches land
  // as a "Gear:" line in the objective's notes; leaves read it from the digest
  // and only fall back to a live gearUp when the recorded gear doesn't fit.
  // Mutable later via patchObjective (notes is whitelisted).
  let gearFor = () => "";
  try {
    const { gearUp } = await import("@/libs/weapon/skillBooks/index.js");
    gearFor = (objectiveName) => {
      const { suggestions } = gearUp({ taskDescription: `${quest.title} — ${objectiveName}`, agent: "questmaster:planQuest" });
      // Score = raw token-hit count (scoreMatch); 1-2 hits is generic-word
      // noise, so require >= 3 per action. A weak scan records NOTHING — the
      // empty Gear line is the leaf's signal to run gearUp live instead.
      const top = (suggestions || [])
        .slice(0, 2)
        .flatMap((s) =>
          [...(s.actions || [])]
            .filter((a) => a.score >= 3)
            .sort((a, b) => b.score - a.score)
            .slice(0, 2)
            .map((a) => `${s.skillBook}.${a.action}`),
        );
      return top.length ? `Gear: ${top.join(", ")}` : "";
    };
  } catch { /* registry unavailable — leaves fall back to live gearUp */ }

  // Write objectives
  const objectiveRows = (plan.objectives || [])
    .filter((o) => o && typeof o.name === "string" && o.name.trim())
    .map((o, i) => ({
      quest_id: questId,
      n: i + 1,
      name: o.name.trim(),
      status: "pending",
      notes: gearFor(o.name.trim()),
    }));

  // Insert objectives ONLY when the table is empty — a re-plan over a quest
  // that already has objectives keeps the canonical rows (they may carry live
  // status/progress) and never duplicates them; the description-shrink below
  // still runs so the description stops overlapping those existing objectives.
  let objectivesWritten = 0;
  if (objectiveRows.length > 0 && !hasObjectives) {
    const { error: oErr } = await client.from(publicTables.objectives).insert(objectiveRows);
    if (oErr) {
      const out = { ok: false, error: `objectives insert failed: ${oErr.message}` };
      logWeaponCall({ weapon: "questmaster", fn: "planQuest", input: { questId }, output: out, status: "fail" });
      return out;
    }
    objectivesWritten = objectiveRows.length;
  }

  // Write questions
  const questionRows = (plan.questions || [])
    .filter((q) => q && typeof q.body === "string" && q.body.trim())
    .map((q) => ({
      quest_id: questId,
      qtype: "unblocking",
      direction: "to_user",
      body: q.body.trim(),
      answered: false,
      actor_name: "questmaster:planQuest",
    }));

  // Insert questions ONLY when none exist yet — re-planning must not pile up
  // duplicate unblocking questions on every pass.
  let questionsWritten = 0;
  if (questionRows.length > 0 && !hasQuestions) {
    for (const qr of questionRows) {
      await client.from(publicTables.questQuestions).insert(qr);
    }
    questionsWritten = questionRows.length;
  }

  // Materialize deliverables interpreted from the description — ONLY if the
  // deliverables table is empty for this quest. This closes the gap where a
  // prose description names artifacts ("a screenshot", "a 9-page PDF") but no
  // formal deliverable rows exist, so workers improvise malformed rows (wrong
  // kind/ref — e.g. a "screenshot" written as a link). If the user already
  // defined deliverables, leave them untouched (the user owns deliverables).
  let deliverableCount = 0;
  const { data: existingDeliverables } = await client
    .from("deliverables")
    .select("id")
    .eq("quest_id", questId)
    .limit(1);
  if (!existingDeliverables || existingDeliverables.length === 0) {
    const VALID_DELIVERABLE_KINDS = ["text", "html", "image", "video", "document"];
    const deliverableRows = (plan.deliverables || [])
      .filter((d) => d && typeof d.name === "string" && d.name.trim())
      .map((d) => ({
        quest_id: questId,
        name: d.name.trim(),
        kind: VALID_DELIVERABLE_KINDS.includes(d.kind) ? d.kind : "text",
        ref: typeof d.ref === "string" && d.ref.trim() ? d.ref.trim() : null,
        status: "active",
      }));
    if (deliverableRows.length > 0) {
      const { error: dErr } = await client.from("deliverables").insert(deliverableRows);
      if (!dErr) deliverableCount = deliverableRows.length;
    }
  }

  // Update description to concise version.
  if (plan.conciseDescription && typeof plan.conciseDescription === "string") {
    let finalDesc = plan.conciseDescription.trim();
    // Preserve the operator's grants of latitude VERBATIM through the shrink.
    // The planner is instructed to keep the AUTONOMY block (systemPrompt rule 1),
    // but enforce it deterministically too: a stripped grant lets a later stage
    // re-ask a decision the user already delegated (the known planQuest residual,
    // libs/skill_book/questmaster/engineering_decisions.md). Drop whatever
    // (possibly reworded) block the shrink emitted, then re-attach the ORIGINAL
    // verbatim — so the block is always present, exact, and never duplicated.
    const autonomy = extractAutonomyBlock(quest.description);
    if (autonomy) {
      finalDesc = finalDesc.replace(AUTONOMY_BLOCK_RE, "").trim();
      finalDesc = `${finalDesc}\n\n${autonomy}`.trim();
    }
    await client
      .from(publicTables.quests)
      .update({ description: finalDesc })
      .eq("id", questId);
  }

  // NO "plan_complete" comment. Comments are the strategic/future-bearing channel,
  // not an action log — and the plan IS its output (the objectives / questions /
  // deliverables rows + the condensed description), which a reader sees directly.
  // A "decomposed into N objectives" bookkeeping line carries no forward insight.

  // Transition. EXPECTED BEHAVIOR: planning that produced UNRESOLVED unblocking
  // questions must NOT enter execute on the first pass — the quest ESCALATES and
  // rests awaiting the user's answers (a quest may only sit in 'escalated' when
  // there is a concrete open question for the user; the caravan's derailed class
  // treats escalated-with-open-question as a valid resting state). Only a
  // question-free plan proceeds to execute. On a RE-PLAN after the user answered,
  // those questions are answered=true, so the open count is 0 and it proceeds.
  const { data: openUnblocking } = await client
    .from(publicTables.questQuestions)
    .select("id")
    .eq("quest_id", questId)
    .eq("qtype", "unblocking")
    .eq("answered", false);
  const transition =
    (openUnblocking || []).length > 0
      ? await moveQuestToEscalated({ questId })
      : await moveQuestToExecute({ questId });

  const out = {
    ok: true,
    objectiveCount: objectivesWritten,
    questionCount: questionsWritten,
    deliverableCount,
    replanned: hasObjectives,
    descriptionShrunk: !!(plan.conciseDescription && typeof plan.conciseDescription === "string"),
    transition,
  };
  logWeaponCall({ weapon: "questmaster", fn: "planQuest", input: { questId }, output: out, status: "ok" });
  return out;
}

// ── Stage-transition guards ──────────────────────────────────────────────────

/**
 * Guard: quest has >=1 objective, >=1 deliverable, and an assignee.
 * If all present -> stage='execute'. Else stay in plan with a comment.
 */
export async function moveQuestToExecute({ questId } = {}) {
  if (!questId) return { ok: false, error: "questId is required" };
  const client = await getServiceClient();

  const { data: quest } = await client
    .from(publicTables.quests)
    .select("id, stage, assignee")
    .eq("id", questId)
    .single();
  if (!quest) return { ok: false, error: "quest not found" };

  const { data: objectives } = await client
    .from(publicTables.objectives)
    .select("id")
    .eq("quest_id", questId);
  const { data: deliverables } = await client
    .from(publicTables.deliverables)
    .select("id")
    .eq("quest_id", questId);
  // A quest may NOT enter execute while unblocking questions are still open —
  // those are decisions awaiting the user, and executing past them produces the
  // wrong work. This is the authoritative stage invariant (defense-in-depth:
  // planQuest already routes such quests to escalated, but no caller may bypass
  // it here). Cleared questions (answered=true) do not block.
  const { data: openQuestions } = await client
    .from(publicTables.questQuestions)
    .select("id")
    .eq("quest_id", questId)
    .eq("qtype", "unblocking")
    .eq("answered", false);

  const missing = [];
  if (!objectives || objectives.length === 0) missing.push("objectives");
  if (!deliverables || deliverables.length === 0) missing.push("deliverables");
  if (!quest.assignee) missing.push("assignee");
  if (openQuestions && openQuestions.length > 0) missing.push(`${openQuestions.length} open unblocking question(s)`);

  if (missing.length > 0) {
    // Routed through the chokepoint: moveQuestToExecute is retried every tick on a
    // not-yet-ready quest, so dedup makes the identical "still blocked" comment a
    // no-op instead of churning the thread.
    const { recordQuestComment } = await import("@/libs/quest/index.js");
    await recordQuestComment(questId, {
      source: "questmaster",
      action: "transition_blocked",
      summary: `moveQuestToExecute blocked: missing ${missing.join(", ")}. Quest stays in '${quest.stage}'.`,
      actor_name: "questmaster:moveQuestToExecute",
    }, { client });
    const out = { ok: false, error: `missing: ${missing.join(", ")}`, stage: quest.stage };
    logWeaponCall({ weapon: "questmaster", fn: "moveQuestToExecute", input: { questId }, output: out, status: "fail" });
    return out;
  }

  const { error: upErr } = await client
    .from(publicTables.quests)
    .update({ stage: "execute" })
    .eq("id", questId);
  if (upErr) return { ok: false, error: upErr.message };

  const out = { ok: true, stage: "execute" };
  logWeaponCall({ weapon: "questmaster", fn: "moveQuestToExecute", input: { questId }, output: out, status: "ok" });
  return out;
}

/**
 * Guard: every objective status='done', every deliverable has a non-null ref.
 * Delegates to questExecution.submit for the full gate + lockphrase.
 */
export async function moveQuestToReview({ questId } = {}) {
  if (!questId) return { ok: false, error: "questId is required" };

  // Soft gate: warn if composite deliverables exist but no quest_assets (ruleCheckpointAssets)
  try {
    const client = await getServiceClient();
    const { data: deliverables } = await client
      .from(publicTables.deliverables)
      .select("kind")
      .eq("quest_id", questId);
    const compositeKinds = ["pdf", "html"];
    const hasComposite = (deliverables || []).some((d) => compositeKinds.includes(d.kind));
    if (hasComposite) {
      const { data: assets } = await client
        .from(publicTables.questAssets)
        .select("id")
        .eq("quest_id", questId)
        .limit(1);
      if (!assets || assets.length === 0) {
        // IDEMPOTENT: this advisory is informational and UNACTIONABLE here —
        // checkpointing happens DURING execution (when an artifact is produced),
        // not at submit time. moveQuestToReview is retried every tick on a quest
        // that can't yet advance, so the identical comment used to spam the log
        // (40+ byte-identical rows on quest ec0cde6c over 3 days). The manual
        // write-once guard is GONE: the comment chokepoint (recordQuestComment)
        // now owns dedup centrally — the constant summary makes a repeat a no-op.
        // One warning is the whole signal. (Real checkpoint enforcement lives at
        // execution time — summarizeQuest's assetCheckpoints flag + worker
        // ruleCheckpointAssets.)
        const { recordQuestComment } = await import("@/libs/quest/index.js");
        await recordQuestComment(questId, {
          source: "system",
          action: "checkpoint_warning",
          summary:
            "Soft gate: quest has composite deliverables (pdf/html) but zero quest_assets. Intermediate artifacts may be missing — future reassembly would require regeneration. Consider checkpointing artifacts per ruleCheckpointAssets.",
          detail: {},
        }, { client });
      }
    }
  } catch (e) {
    console.warn("[moveQuestToReview] checkpoint warning check failed:", e.message);
  }

  // Delegate to the submit gate which does the full verification
  const { submit } = await import("@/libs/weapon/questExecution/index.js");
  const result = await submit({ questId });
  logWeaponCall({ weapon: "questmaster", fn: "moveQuestToReview", input: { questId }, output: { ok: result.ok, stage: result.stage }, status: result.ok ? "ok" : "fail" });
  return result;
}

/**
 * Guard: every workable objective is blocked on an open user question.
 * If so -> stage='escalated'.
 */
export async function moveQuestToEscalated({ questId } = {}) {
  if (!questId) return { ok: false, error: "questId is required" };
  const client = await getServiceClient();

  const { data: quest } = await client
    .from(publicTables.quests)
    .select("id, stage")
    .eq("id", questId)
    .single();
  if (!quest) return { ok: false, error: "quest not found" };

  // Get active (non-done) objectives
  const { data: objectives } = await client
    .from(publicTables.objectives)
    .select("id, n, name, status")
    .eq("quest_id", questId)
    .order("n", { ascending: true });
  const active = (objectives || []).filter((o) => o.status !== "done");
  if (active.length === 0) {
    return { ok: false, error: "no active objectives — quest may be ready for review, not escalation" };
  }

  // Get open (unanswered) questions
  const { data: questions } = await client
    .from(publicTables.questQuestions)
    .select("id, unblocks_objective_ids")
    .eq("quest_id", questId)
    .eq("answered", false);
  if (!questions || questions.length === 0) {
    return { ok: false, error: "no open questions — nothing to escalate on" };
  }

  // Check: every active objective must be blocked by at least one open question
  // (A question blocks an objective if unblocks_objective_ids includes that objective's id,
  //  or if unblocks_objective_ids is empty — meaning it blocks the whole quest.)
  const globalBlockers = questions.filter(
    (q) => !Array.isArray(q.unblocks_objective_ids) || q.unblocks_objective_ids.length === 0,
  );
  if (globalBlockers.length > 0) {
    // Global blocker = all objectives are blocked
  } else {
    const blockedIds = new Set(questions.flatMap((q) => q.unblocks_objective_ids || []));
    const unblocked = active.filter((o) => !blockedIds.has(o.id));
    if (unblocked.length > 0) {
      return {
        ok: false,
        error: `${unblocked.length} active objective(s) are not blocked by any question: ${unblocked.map((o) => `n${o.n}`).join(", ")}`,
      };
    }
  }

  const { error: upErr } = await client
    .from(publicTables.quests)
    .update({ stage: "escalated" })
    .eq("id", questId);
  if (upErr) return { ok: false, error: upErr.message };

  const { recordQuestComment } = await import("@/libs/quest/index.js");
  await recordQuestComment(questId, {
    source: "questmaster",
    action: "escalated",
    summary: `moveQuestToEscalated: ${active.length} active objective(s) blocked by ${questions.length} open question(s). Quest escalated to user.`,
    actor_name: "questmaster:moveQuestToEscalated",
  }, { client });

  const out = { ok: true, stage: "escalated" };
  logWeaponCall({ weapon: "questmaster", fn: "moveQuestToEscalated", input: { questId }, output: out, status: "ok" });
  return out;
}

// ── dispatchRoledAgent ───────────────────────────────────────────────────────

/**
 * Compose role-aware substrate priming and dispatch a sub-agent. The dispatched
 * agent receives a prompt prefixed with the contents of BOTH `CLAUDE.md` (cross-role
 * fundamentals) AND `CLAUDE-<role>.md` (role-specific rules, when the file exists),
 * followed by the caller's task prompt.
 *
 * Role-file resolution:
 *   - role === "GM" → CLAUDE-GM.md
 *   - role === "DungeonMaster" → CLAUDE-DungeonMaster.md (loaded only if file present)
 *   - role === "SA" or "MA" → no role file expected; skill book howTo is the procedure.
 *   - When CLAUDE-<role>.md does not exist, priming includes only CLAUDE.md with a
 *     note that no role file was found — this is graceful, not an error, because
 *     not every role needs its own file (the Role Index in CLAUDE.md is canonical).
 *
 * Hard rejects:
 *   - role missing / non-string / empty → { ok:false, error }
 *   - role contains path-separator characters or "../" → { ok:false, error } (path injection guard)
 *   - prompt missing / empty → { ok:false, error }
 *   - When CLAUDE.md is missing entirely → { ok:false, error } (cross-role fundamentals are not optional)
 *
 * @param {{
 *   role: string,
 *   prompt: string,
 *   repoRoot?: string,   // defaults to process.cwd() — main checkout root
 *   handshake?: string,  // optional handshake string the dispatch convention requires
 * }} opts
 * @returns {Promise<{ ok:boolean, output?:string, rawOutput?:string, roleFileLoaded?:boolean, error?:string }>}
 */
export async function dispatchRoledAgent({ role, prompt, repoRoot, handshake } = {}) {
  if (!role || typeof role !== "string" || !role.trim()) {
    const out = { ok: false, error: "role is required (non-empty string)" };
    logWeaponCall({ weapon: "questmaster", fn: "dispatchRoledAgent", input: { hasRole: !!role }, output: out, status: "fail" });
    return out;
  }
  const trimmedRole = role.trim();
  if (/[\\/]|\.\./.test(trimmedRole)) {
    const out = { ok: false, error: "role contains illegal characters (no path separators or '..' allowed)" };
    logWeaponCall({ weapon: "questmaster", fn: "dispatchRoledAgent", input: { role: trimmedRole }, output: out, status: "fail" });
    return out;
  }
  if (!prompt || typeof prompt !== "string" || !prompt.trim()) {
    const out = { ok: false, error: "prompt is required (non-empty string)" };
    logWeaponCall({ weapon: "questmaster", fn: "dispatchRoledAgent", input: { role: trimmedRole, hasPrompt: !!prompt }, output: out, status: "fail" });
    return out;
  }

  const root = repoRoot && typeof repoRoot === "string" ? repoRoot : process.cwd();
  const baseClaudeMd = join(root, "CLAUDE.md");
  const roleClaudeMd = join(root, `CLAUDE-${trimmedRole}.md`);

  if (!existsSync(baseClaudeMd)) {
    const out = { ok: false, error: `CLAUDE.md not found at ${baseClaudeMd}` };
    logWeaponCall({ weapon: "questmaster", fn: "dispatchRoledAgent", input: { role: trimmedRole }, output: out, status: "fail" });
    return out;
  }

  let baseContent = "";
  try {
    baseContent = readFileSync(baseClaudeMd, "utf8");
  } catch (e) {
    const out = { ok: false, error: `failed to read CLAUDE.md: ${e?.message || String(e)}` };
    logWeaponCall({ weapon: "questmaster", fn: "dispatchRoledAgent", input: { role: trimmedRole }, output: out, status: "fail" });
    return out;
  }

  let roleContent = "";
  let roleFileLoaded = false;
  if (existsSync(roleClaudeMd)) {
    try {
      roleContent = readFileSync(roleClaudeMd, "utf8");
      roleFileLoaded = true;
    } catch (e) {
      const out = { ok: false, error: `CLAUDE-${trimmedRole}.md exists but could not be read: ${e?.message || String(e)}` };
      logWeaponCall({ weapon: "questmaster", fn: "dispatchRoledAgent", input: { role: trimmedRole }, output: out, status: "fail" });
      return out;
    }
  }

  const priming = [
    "## Substrate priming — read before acting",
    "",
    "You are being dispatched with role `" + trimmedRole + "`. The following two documents are your operating substrate. Treat their content as authoritative; the task prompt below applies WITHIN their constraints.",
    "",
    "### CLAUDE.md (cross-role fundamentals)",
    "",
    baseContent.trim(),
    "",
    roleFileLoaded
      ? `### CLAUDE-${trimmedRole}.md (role-specific)\n\n${roleContent.trim()}`
      : `### Role file CLAUDE-${trimmedRole}.md\n\n(none present — the Role Index in CLAUDE.md indicates this role's procedure is owned elsewhere, e.g., a skill book howTo. Look up your role's owner there.)`,
    "",
    "---",
    "",
    handshake ? `## Handshake\n\n${handshake}\n\n---\n` : "",
    "## Task",
    "",
    prompt.trim(),
  ].filter(Boolean).join("\n");

  let res;
  try {
    res = await claudecliInvoke(priming);
  } catch (e) {
    const out = { ok: false, error: `claudecli.invoke threw: ${e?.message || String(e)}`, roleFileLoaded };
    logWeaponCall({ weapon: "questmaster", fn: "dispatchRoledAgent", input: { role: trimmedRole, roleFileLoaded }, output: out, status: "fail" });
    return out;
  }

  if (!res || !res.ok) {
    const out = { ok: false, error: res?.error || "claudecli returned not-ok", rawOutput: res?.rawOutput || "", roleFileLoaded };
    logWeaponCall({ weapon: "questmaster", fn: "dispatchRoledAgent", input: { role: trimmedRole, roleFileLoaded }, output: { ok: false, error: out.error }, status: "fail" });
    return out;
  }

  const out = { ok: true, output: res.html, rawOutput: res.rawOutput, roleFileLoaded };
  logWeaponCall({ weapon: "questmaster", fn: "dispatchRoledAgent", input: { role: trimmedRole, roleFileLoaded, promptLen: prompt.length }, output: { ok: true, outputLen: (res.rawOutput || "").length }, status: "ok" });
  return out;
}

// ── tickWorkerDispatch ───────────────────────────────────────────────────────

/**
 * One worker-dispatcher tick (programmatic; no LLM). Scans active quests
 * assigned to `Guildmaster` (the local-run executor) at stage='execute',
 * walks them newest-first, and on the first quest with work either resumes
 * its already-running objective or loads its lowest-n pending objective
 * (flipping that objective pending -> running). Single worker, no parallel fan-out.
 *
 * @param {{ assignee?: string }} [opts] assignee defaults to 'Guildmaster'
 * @returns {Promise<
 *   | { ok:true, questId:string, url:string, title:string, objective:{ id:string, n:number, name:string }, resumed?:boolean }
 *   | { ok:false, reason:string }
 * >}
 */
export async function tickWorkerDispatch({ assignee = "Main_Claude" } = {}) {
  const client = await getServiceClient();

  const { data: quests, error: qErr } = await client
    .from(publicTables.quests)
    .select("id, title, stage, assignee, created_at")
    .eq("assignee", assignee)
    .eq("stage", "execute")
    .order("created_at", { ascending: false });

  if (qErr) {
    const out = { ok: false, reason: `quests query failed: ${qErr.message}` };
    logWeaponCall({ weapon: "questmaster", fn: "tickWorkerDispatch", input: { assignee }, output: out, status: "fail" });
    return out;
  }
  if (!quests || quests.length === 0) {
    const out = { ok: false, reason: `no execute-stage quests assigned to ${assignee}` };
    logWeaponCall({ weapon: "questmaster", fn: "tickWorkerDispatch", input: { assignee }, output: out, status: "ok" });
    return out;
  }

  for (const quest of quests) {
    const { data: objectives, error: sErr } = await client
      .from("objectives")
      .select("id, n, name, status")
      .eq("quest_id", quest.id)
      // Actionable = not-done. Include the non-canonical synonyms `todo`/`active`
      // a worker might improvise, so a stray status word never hides a quest from
      // the dispatcher (the canonical set is pending/running/done/review). A blocked
      // objective stays OPEN (active) — the block lives in an unblocking question +
      // escalated stage, NOT a status — so it remains actionable here.
      .in("status", ["pending", "running", "todo", "active"])
      .order("n", { ascending: true });
    if (sErr) {
      const out = { ok: false, reason: `objectives query failed: ${sErr.message}` };
      logWeaponCall({ weapon: "questmaster", fn: "tickWorkerDispatch", input: { questId: quest.id }, output: out, status: "fail" });
      return out;
    }
    if (!objectives || objectives.length === 0) continue; // nothing to do on this quest

    const running = objectives.find((s) => s.status === "running");
    if (running) {
      const out = {
        ok: true,
        questId: quest.id,
        url: questPath(quest.id),
        title: quest.title,
        objective: { id: running.id, n: running.n, name: running.name },
        resumed: true,
      };
      logWeaponCall({ weapon: "questmaster", fn: "tickWorkerDispatch", input: { assignee }, output: { questId: quest.id, objectiveN: running.n, resumed: true }, status: "ok" });
      return out;
    }

    const next = objectives[0]; // lowest-n pending
    const { error: uErr } = await client.from("objectives").update({ status: "running" }).eq("id", next.id);
    if (uErr) {
      const out = { ok: false, reason: `failed to flip objective ${next.n} to running: ${uErr.message}` };
      logWeaponCall({ weapon: "questmaster", fn: "tickWorkerDispatch", input: { questId: quest.id, objectiveN: next.n }, output: out, status: "fail" });
      return out;
    }
    const out = {
      ok: true,
      questId: quest.id,
      url: questPath(quest.id),
      title: quest.title,
      objective: { id: next.id, n: next.n, name: next.name },
    };
    logWeaponCall({ weapon: "questmaster", fn: "tickWorkerDispatch", input: { assignee }, output: { questId: quest.id, objectiveN: next.n }, status: "ok" });
    return out;
  }

  const out = { ok: false, reason: `no quest assigned to ${assignee} has a pending/running objective` };
  logWeaponCall({ weapon: "questmaster", fn: "tickWorkerDispatch", input: { assignee }, output: out, status: "ok" });
  return out;
}

// ── toggleQuestActiveExecution (worker_id lease) ───────────────────────────────

/**
 * Lease or release a quest's single execution slot via a compare-and-set on
 * `quests.worker_id`. This is the duplicate-worker guard from
 * docs/quest-lifecycle.md §2: only one worker may hold a quest at a time.
 *
 * Lease  (release=false): set worker_id from NULL -> workerId atomically
 *   (the `.is("worker_id", null)` predicate makes the UPDATE a CAS — if another
 *   tick already leased it, 0 rows match and we report leased:false).
 * Release (release=true): clear worker_id + worker_claimed_at. Scoped to the
 *   holder when workerId is given (won't steal another worker's lease).
 *
 * @param {{ questId:string, workerId:string, release?:boolean }} opts
 * @returns {Promise<{ ok:boolean, leased?:boolean, released?:boolean, heldBy?:string|null, error?:string }>}
 */
export async function toggleQuestActiveExecution({ questId, workerId, release = false } = {}) {
  if (!questId || (!release && !workerId)) {
    const out = { ok: false, error: "questId (and workerId when leasing) are required" };
    logWeaponCall({ weapon: "questmaster", fn: "toggleQuestActiveExecution", input: { questId, release }, output: out, status: "fail" });
    return out;
  }
  const client = await getServiceClient();

  if (release) {
    let q = client.from(publicTables.quests).update({ worker_id: null, worker_claimed_at: null }).eq("id", questId);
    if (workerId) q = q.eq("worker_id", workerId);
    const { data, error } = await q.select("id");
    if (error) {
      const out = { ok: false, error: error.message };
      logWeaponCall({ weapon: "questmaster", fn: "toggleQuestActiveExecution", input: { questId, release: true }, output: out, status: "fail" });
      return out;
    }
    const released = (data?.length || 0) > 0;
    const out = { ok: true, released };
    logWeaponCall({ weapon: "questmaster", fn: "toggleQuestActiveExecution", input: { questId, workerId, release: true }, output: out, status: "ok" });
    return out;
  }

  // Lease = compare-and-set worker_id NULL -> workerId.
  const { data, error } = await client
    .from(publicTables.quests)
    .update({ worker_id: workerId, worker_claimed_at: new Date().toISOString() })
    .eq("id", questId)
    .is("worker_id", null)
    .select("id");
  if (error) {
    const out = { ok: false, error: error.message };
    logWeaponCall({ weapon: "questmaster", fn: "toggleQuestActiveExecution", input: { questId, workerId }, output: out, status: "fail" });
    return out;
  }
  const leased = (data?.length || 0) === 1;
  let heldBy = null;
  if (!leased) {
    const { data: row } = await client.from(publicTables.quests).select("worker_id").eq("id", questId).maybeSingle();
    heldBy = row?.worker_id || null;
  }
  const out = { ok: true, leased, heldBy };
  logWeaponCall({ weapon: "questmaster", fn: "toggleQuestActiveExecution", input: { questId, workerId }, output: out, status: "ok" });
  return out;
}

// ── unstuckQuests (stale-lease reclaim) ────────────────────────────────────────

/**
 * Reclaim leases that have gone stale. A worker that died or hung leaves
 * `worker_id` set; without this, the quest would sit in execute forever
 * (docs/quest-lifecycle.md §2 unstuckQuest). We treat a lease as stale when
 * `worker_claimed_at` is older than staleMinutes. For each stale lease: clear
 * worker_id, write a `source='system', action='unstuck'` incident comment, so
 * the next tick re-dispatches the quest.
 *
 * Liveness mechanism (operator left this open): leaf workers are headless
 * `claude --print` subprocesses with no queryable session, so lease-age IS the
 * liveness signal — a still-working leaf bumps the quest's child rows (keeping
 * the lease fresh would require touching worker_claimed_at, which the worker
 * does on each burst); a dead one goes stale and is reclaimed here.
 *
 * @param {{ assignee?:string, staleMinutes?:number }} opts
 * @returns {Promise<{ ok:boolean, reclaimed:Array<{questId:string,workerId:string,ageMin:number}>, error?:string }>}
 */
export async function unstuckQuests({ assignee = "Main_Claude", staleMinutes = 20 } = {}) {
  const client = await getServiceClient();
  const cutoffMs = Date.now() - staleMinutes * 60_000;

  // Candidate = every currently-leased quest. Staleness is decided by LAST
  // PROGRESS, not lease-age: a leaf legitimately working > staleMinutes keeps
  // bumping the quest + its child rows (comments, objectives), so we must read
  // the most-recent of {quest.updated_at, latest comment, latest objective}
  // and only reclaim when NOTHING moved in the window (docs/quest-lifecycle.md
  // §2 — "the quest and all its child rows hasn't been modified in > 20 min").
  // Using worker_claimed_at alone would falsely reclaim a slow-but-live worker
  // and double-dispatch it.
  const { data: leased, error } = await client
    .from(publicTables.quests)
    .select("id, title, worker_id, worker_claimed_at, updated_at")
    .eq("assignee", assignee)
    .not("worker_id", "is", null);

  if (error) {
    const out = { ok: false, reclaimed: [], error: error.message };
    logWeaponCall({ weapon: "questmaster", fn: "unstuckQuests", input: { assignee, staleMinutes }, output: out, status: "fail" });
    return out;
  }

  const reclaimed = [];
  for (const q of leased || []) {
    // Last progress = max(quest.updated_at, worker_claimed_at, latest comment, latest objective update).
    const times = [q.updated_at, q.worker_claimed_at].filter(Boolean).map((t) => new Date(t).getTime());
    const { data: lastC } = await client
      .from(publicTables.questComments)
      .select("created_at")
      .eq("quest_id", q.id)
      .order("created_at", { ascending: false })
      .limit(1)
      .maybeSingle();
    if (lastC?.created_at) times.push(new Date(lastC.created_at).getTime());
    const { data: lastO } = await client
      .from("objectives")
      .select("updated_at")
      .eq("quest_id", q.id)
      .order("updated_at", { ascending: false })
      .limit(1)
      .maybeSingle();
    if (lastO?.updated_at) times.push(new Date(lastO.updated_at).getTime());

    const lastProgressMs = times.length ? Math.max(...times) : 0;
    if (lastProgressMs >= cutoffMs) continue; // still making progress — leave the lease

    const ageMin = Math.round((Date.now() - lastProgressMs) / 60_000);
    const prior = q.worker_id;
    const { error: uErr } = await client
      .from(publicTables.quests)
      .update({ worker_id: null, worker_claimed_at: null })
      .eq("id", q.id)
      .eq("worker_id", prior);
    if (uErr) continue;
    // NO "unstuck" comment. Reclaiming a dead worker's lease is pure bookkeeping
    // (the next worker gains no forward insight from it) — it's already recorded
    // in the lease clear, the console log, and cron_log. Comments stay strategic.
    reclaimed.push({ questId: q.id, workerId: prior, ageMin });
  }

  if (reclaimed.length > 0) {
    console.log(`[questTick] unstuck — reclaimed ${reclaimed.length} stale lease(s):`, JSON.stringify(reclaimed));
  }
  const out = { ok: true, reclaimed };
  logWeaponCall({ weapon: "questmaster", fn: "unstuckQuests", input: { assignee, staleMinutes }, output: { reclaimed: reclaimed.length }, status: "ok" });
  return out;
}

// ── derailment rescue note (rescue lives in libs/caravan) ─────────────────────
//
// docs/quest-lifecycle.md §7. The caravan derailed class (libs/caravan) is the
// LIVE rescue surface: it deterministically kicks a derailed quest back to its
// right stage with a coaching comment, and after MAX_RESCUE_ATTEMPTS kickbacks
// for the same condition ESCALATES it (stage='escalated' + reason comment +
// unblocking question) rather than looping. There is NO separate `parked`
// terminal — the only resting states are `escalated` and `complete` (retired
// 2026-06-14; the earlier detectDerailedQuests + parkQuest were both removed as
// the caravan is the single rescue surface).

// ── questTick (the cron stage-router) ──────────────────────────────────────────

/** Build the REVIEW-leaf prompt for a purrview-stage quest. The leaf runs the
 *  Questmaster review procedure against the script-locked gate weapons. */
function buildLeafReviewPrompt({ questId, title, url, workerId, digest = null }) {
  return [
    `You are a GuildOS REVIEW leaf deployed by Cron:questTick to review ONE quest at the purrview stage.`,
    ``,
    `Quest: "${title}"`,
    `Quest id: ${questId}`,
    `Quest URL: ${url}`,
    `Your worker_id (already leased to you by the cron): ${workerId}`,
    ``,
    ...(digest
      ? [
          `=== ORIENTATION DIGEST (pre-fetched by the dispatcher at deploy time — read this INSTEAD of calling summarizeQuest) ===`,
          digest,
          `=== END DIGEST ===`,
          ``,
        ]
      : []),
    `THE CANONICAL CLI — all skill-book/weapon access goes through it (resolves @/ imports; do NOT probe import paths, hand-roll node -e loaders, or hunt for alias-loader files):`,
    `  • Run an action:        node --no-warnings libs/adventurer/run-action.mjs <skillBookId> <action> '<inputJson>'`,
    `  • Read ONE procedure:   node --no-warnings libs/adventurer/run-action.mjs --howto <skillBookId> <action>   (e.g. --howto questmaster reviewSubmission; NEVER read/grep a whole skill-book file to find a howTo)`,
    ``,
    `Run the purrview review procedure (docs/quest-lifecycle.md + questmaster skill book):`,
    `1. sage.preflight (the harness injects it). The procedures you need are runReviewLoop and (fallback) reviewSubmission — fetch each via --howto above; do not read the questmaster skill-book file end-to-end.`,
    `2. ORIENT: ${digest ? `the digest above IS questmaster.summarizeQuest's output — stage, deliverables (names + filled), latest note. Do not re-call it unless you need a refresh after mutating state.` : `questmaster.summarizeQuest({questId:"${questId}"}) — get stage, deliverables (names + filled), and the latest note in one call.`}`,
    `3. questmaster.runReviewLoop({questId:"${questId}"}) — the script-locked path: confirmSubmission (verifies the submit lockphrase) → judge each deliverable → approve (all pass) or bounce (any fail). Read its returned action:`,
    `   • action='approved' → it moved the quest to review. Done.`,
    `   • action='bounced'  → it moved the quest back to execute with per-deliverable verdicts. Done.`,
    `   • action='manual_required' (non-image deliverables — text/document/html) → fall back to the manual questmaster.reviewSubmission howTo: confirmSubmission, then judge each deliverable against its claim (claude-multimodal-read for pdf/image content, read for text/html), then questPurrview.approve (all pass) or questPurrview.bounce (any fail) — every verdict MUST carry a valid judge_origin.`,
    `3b. VISUAL VERIFICATION (auto-captured for you). The cron ran questPurrview.autoVisualVerify BEFORE you launched: it headlessly rendered each VISUAL deliverable (pdf.js for documents, a full-page browser shot for html/image/video), uploaded the PNG to GuildOS_Bucket, and set verify_screenshot_ref. So every visual deliverable should already carry a real render screenshot — you do NOT need a CIC/browser channel. Your job for those: confirm each screenshot looks like a genuine render of the deliverable (the approve gate independently vision-judges it via openai_images.judge — blank/404/skeleton screenshots are rejected), then approve. If a visual deliverable still lacks verify_screenshot_ref, run questPurrview.autoVisualVerify({questId:"${questId}"}) yourself to fill it. NEVER fake a link.`,
    `3c. TEXT DELIVERABLES ARE NOT SCREENSHOTTED. A text/json/markdown/csv deliverable is verified by READING its ref content — NEVER by rendering it to a PNG and vision-judging it (a headless render of raw text lands on a blank/login page and the image judge false-fails it: the "72 bounced as a login screen" bug). autoVisualVerify SKIPS text kinds on purpose; do NOT screenshot them or set an image verify_screenshot_ref for them. Read the content, compare it to what the quest asks for, and post the verdict with judge_origin='claude-multimodal-read' — then set verify_screenshot_ref to a \`PASS — <why>\` / \`FAIL — <why>\` verdict. The approve/bounce gate REJECTS any text-deliverable verdict that carries an openai_images.judge origin.`,
    `4. NEVER write quests.stage directly and NEVER open a deliverable before confirmSubmission passes. If confirmSubmission fails (no submit lockphrase) → bounce or escalate; the quest is untrusted.`,
    `5. Write ONE structured closing comment (Done: <verdict + what you reviewed> · Next: <what the worker must fix, if bounced> · Blocked: <any blocker or "none">). Every comment must be INSIGHT-BEARING + ACTIONABLE — it tells the NEXT step what to do; never post a pure status/log line. If a PRIOR review-leaf already left the SAME closing/blocked comment for the SAME unresolved blocker (a repeat attempt), EDIT that comment in place (PATCH — bump its attempt count) instead of inserting a new near-identical one. Do NOT pile up.`,
    `6. ALWAYS release your lease at the end (questmaster.toggleQuestActiveExecution with release=true, workerId="${workerId}"). unstuckQuests reclaims it after 20min if you die first.`,
    ``,
    `Do real review against the real deliverables. Verify visually where applicable. Do not rubber-stamp.`,
  ].join("\n");
}

/**
 * THE QUESTIONS-GATE prompt (operator directive 2026-06-13). When a plan/execute
 * quest's question inbox is not fully cleared (`questionGate.blocked`), the
 * dispatch pattern hands the worker THIS prompt INSTEAD of the plan/execute one.
 * The worker's ONLY job is to clear the inbox — answer user questions, digest the
 * user's answers, escalate anything still awaiting the user — then STOP. No
 * planning, no execution, no advancing. The NEXT worker does the real work once
 * the gate is clear. This is the hard gate: the work instructions are not even
 * present in the prompt, so the worker cannot proceed past the questions.
 *
 * @param {{ questId:string, title:string, url:string, workerId:string, stageLabel:string, digest?:string, questionGate:object }} a
 */
function buildQuestionGatePrompt({ questId, title, url, workerId, stageLabel, digest = null, questionGate }) {
  const g = questionGate || {};
  const list = (arr, fmt) => (arr || []).map((q, i) => `     ${i + 1}. ${fmt(q)}`).join("\n");
  return [
    `You are a GuildOS QUESTIONS-GATE leaf deployed by the caravan/cron to clear ONE quest's question inbox.`,
    `This quest is at the ${stageLabel} stage, but it has OPEN questions, so it is HARD-GATED: no ${stageLabel} work happens until every question is addressed.`,
    ``,
    `Quest: "${title}"`,
    `Quest id: ${questId}`,
    `Quest URL: ${url}`,
    `Your worker_id (already leased to you): ${workerId}`,
    ``,
    ...(digest ? [`=== ORIENTATION DIGEST (pre-fetched) ===`, digest, `=== END DIGEST ===`, ``] : []),
    `=== THE OPEN QUESTION INBOX (your ONLY job this tick) ===`,
    `USER questions to ANSWER (qtype='user', not yet answered) — ${(g.userToAnswer || []).length}:`,
    (g.userToAnswer || []).length ? list(g.userToAnswer, (q) => `[${q.id}] ${q.body}`) : `     (none)`,
    `Your UNBLOCKING questions the user has ANSWERED — DIGEST then remove (qtype='unblocking', answered=true) — ${(g.answersToDigest || []).length}:`,
    (g.answersToDigest || []).length ? list(g.answersToDigest, (q) => `[${q.id}] Q: ${q.body}  →  A: ${q.answer}`) : `     (none)`,
    `Your UNBLOCKING questions STILL AWAITING the user (qtype='unblocking', answered=false) — ${(g.awaitingUser || []).length}:`,
    (g.awaitingUser || []).length ? list(g.awaitingUser, (q) => `[${q.id}] ${q.body}`) : `     (none)`,
    `=== END INBOX ===`,
    ``,
    `THE CANONICAL CLI — all skill-book/weapon access goes through it (resolves @/ imports):`,
    `  • Run an action:        node --no-warnings libs/adventurer/run-action.mjs <skillBookId> <action> '<inputJson>'`,
    `  • Read ONE procedure:   node --no-warnings libs/adventurer/run-action.mjs --howto <skillBookId> <action>   (e.g. --howto questmaster questionDigestion)`,
    ``,
    `Run the questions-gate procedure — DO ONLY THIS, then stop:`,
    `1. sage.preflight (the harness injects it).`,
    `2. ANSWER every open USER question: write your reply into the question's 'answer' field via questmaster.updateQuestQuestion (NEVER delete a user question — it is the user's to remove after reading your answer; the DB guard rejects a worker delete anyway).`,
    `3. DIGEST every ANSWERED unblocking question per the questmaster questionDigestion howTo: migrate the user's decision into the objectives/quest/description, RUN any change it implies, then REMOVE that unblocking question. Read --howto questmaster questionDigestion first.`,
    `4. If ANY unblocking question is STILL AWAITING the user after steps 2-3 (the user has not answered it), the quest is blocked on the user BY DEFINITION — you cannot clear it. Leave those questions in place (never answer your own) and MOVE THE QUEST TO ESCALATED (questmaster.moveQuestToEscalated). This is MANDATORY, not a judgment call: it is what stops the quest from being re-dispatched every tick to no effect. The gate cannot be "fully cleared" while an awaiting-user question exists, so the only resting state is escalated.`,
    `5. HARD STOP. Do NOT plan, do NOT itemize, do NOT execute any objective, do NOT produce any deliverable, do NOT advance toward ${stageLabel === "plan" ? "execute" : "review"} on account of the work. Clearing the inbox IS the whole job. The NEXT worker picks up the actual ${stageLabel} work once no open questions remain.`,
    `6. Write ONE structured closing comment (Done: <which questions answered/digested> · Next: "${stageLabel} work — gate is now clear" or "still awaiting user on <id(s)>" · Blocked: <the awaiting-user question(s), or "none">).`,
    `7. ALWAYS release your lease at the end (questmaster.toggleQuestActiveExecution with release=true, workerId="${workerId}"). unstuckQuests reclaims it after the stale window if you die first.`,
  ].join("\n");
}

/** Build the PLANNING-leaf prompt for a plan-stage quest. The planning agent's
 *  one job is ITEMIZING the big description: migrate its work items into
 *  objectives, its final artifacts into deliverables, its ambiguities into
 *  quest_questions — until the description is short and non-overlapping with
 *  the sub-items. Best practice lives in the questmaster skill book (planQuest
 *  howTo); the scripted itemizer is the questmaster.planQuest weapon action.
 *
 *  HARD QUESTIONS GATE: if `questionGate.blocked`, the worker is handed the
 *  questions-only prompt instead — no planning until the inbox is cleared. */
export function buildLeafPlanPrompt({ questId, title, url, workerId, digest = null, questionGate = null }) {
  if (questionGate?.blocked) {
    return buildQuestionGatePrompt({ questId, title, url, workerId, stageLabel: "plan", digest, questionGate });
  }
  return [
    `You are a GuildOS PLANNING leaf deployed by Cron:questTick to plan ONE quest at the plan stage.`,
    ``,
    `Quest: "${title}"`,
    `Quest id: ${questId}`,
    `Quest URL: ${url}`,
    `Your worker_id (already leased to you by the cron): ${workerId}`,
    ``,
    `THE CANONICAL CLI — all skill-book/weapon access goes through it (resolves @/ imports; do NOT probe import paths, hand-roll node -e loaders, or hunt for alias-loader files):`,
    `  • Run an action:        node --no-warnings libs/adventurer/run-action.mjs <skillBookId> <action> '<inputJson>'`,
    `  • Read ONE procedure:   node --no-warnings libs/adventurer/run-action.mjs --howto <skillBookId> <action>   (e.g. --howto questmaster planQuest; NEVER read/grep a whole skill-book file to find a howTo)`,
    ``,
    `Run the planning procedure:`,
    `1. sage.preflight (the harness injects it). Read the best-practice procedure: --howto questmaster planQuest. That howTo OWNS how itemizing is done — follow it.`,
    `2. Run the scripted itemizer: questmaster.planQuest({questId:"${questId}"}). It distills the big description into objectives + quest_questions + deliverables and shrinks the description, then attempts moveQuestToExecute.`,
    `3. VERIFY the done-condition before you stop (getQuestDetails): the description is SHORT (global instructions, links, references only) and NON-OVERLAPPING with the sub-items; every work item from the original description exists as a verb-led objective (ruleObjectiveClarity); every final artifact is a deliverables row with a content-typed kind; every load-bearing ambiguity is an open quest_question. If anything is missing or still duplicated in the description, fix it surgically (patchObjective / updateQuest description / insert the missing row) — do not re-run the whole itemizer over good rows.`,
    `4. If planQuest's transition reports the quest moved to execute, you are done. If it stayed in 'plan' (missing objectives/deliverables/assignee), fill the named gap and call questmaster.moveQuestToExecute({questId:"${questId}"}) yourself. If the description is too vague to itemize at all, write ONE open quest_question to the user naming what you need (the quest stays in plan; the tick re-plans after the user answers).`,
    `5. NEVER write quests.stage directly — moveQuestToExecute is the only transition. Do NOT start executing objectives; planning leaves the work for the execute leaf.`,
    `6. ALWAYS release your lease at the end (questmaster.toggleQuestActiveExecution with release=true, workerId="${workerId}"). unstuckQuests reclaims it after 20min if you die first.`,
  ].join("\n");
}

/** Build the leaf-worker prompt for a quest. The leaf is a headless
 *  `claude --print` session run from the repo root; it inherits the full
 *  GuildOS substrate (CLAUDE.md preflight, skill books, weapons). Its job is
 *  the executeQuest procedure in docs/quest-lifecycle.md §2.
 *
 *  HARD QUESTIONS GATE: if `questionGate.blocked`, the worker is handed the
 *  questions-only prompt instead — no execution until the inbox is cleared. */
export function buildLeafExecutePrompt({ questId, title, url, workerId, digest = null, browser = false, questionGate = null }) {
  if (questionGate?.blocked) {
    return buildQuestionGatePrompt({ questId, title, url, workerId, stageLabel: "execute", digest, questionGate });
  }
  return [
    `You are a GuildOS worker leaf deployed by Cron:questTick to push ONE quest forward.`,
    ``,
    `Quest: "${title}"`,
    `Quest id: ${questId}`,
    `Quest URL: ${url}`,
    `Your worker_id (already leased to you by the cron): ${workerId}`,
    ``,
    ...(digest
      ? [
          `=== ORIENTATION DIGEST (pre-fetched by the dispatcher at deploy time — read this INSTEAD of calling summarizeQuest) ===`,
          digest,
          `=== END DIGEST ===`,
          ``,
        ]
      : []),
    `THE CANONICAL CLI — all skill-book/weapon access goes through it (resolves @/ imports; do NOT probe import paths, hand-roll node -e loaders, or hunt for alias-loader files):`,
    `  • Run an action:        node --no-warnings libs/adventurer/run-action.mjs <skillBookId> <action> '<inputJson>'`,
    `  • Read ONE procedure:   node --no-warnings libs/adventurer/run-action.mjs --howto <skillBookId> <action>   (NEVER read/grep a whole skill-book file to find a howTo)`,
    `  • Enumerate registry:   node --no-warnings libs/adventurer/run-action.mjs --list`,
    `Quest reads/mutations go through questmaster actions over this CLI — getQuestDetails (full objective/comment/question bodies in ONE call), patchObjective, updateQuestQuestion, resolveQuestComment, moveQuestToReview. NEVER orient or mutate via raw curl against Supabase REST (table-name guessing wastes trips; e.g. the table is "objectives", not "quest_objectives").`,
    `  • LARGE PAYLOADS — NEVER inline big content on the command line. A whole shell command line caps at ~32 KB (Windows CreateProcess); a big HTML/text/base64 deliverable inlined as '<inputJson>' blows that limit and the mutation fails before it runs. For ANY large payload (rule of thumb: > ~20 KB, e.g. a rendered HTML page), WRITE it to a scratch file first (e.g. under the OS temp dir) and pass the PATH: node --no-warnings libs/adventurer/run-action.mjs <sb> <action> --input-file "<scratchPath>". Ship deliverable artifacts (worker.shipItemArtifact / quest_assets) by UPLOAD, and reference the resulting ref/URL — never paste the content into an objective note or comment.`,
    ``,
    ...(browser
      ? [
          `AUTHENTICATED BROWSER AVAILABLE: you have a headless Chrome carrying the operator's logged-in profile via the Playwright MCP (tools named mcp__playwright__*). For ANY deliverable that is a URL / live page / rendered artifact, VISUALLY VERIFY it: mcp__playwright__browser_navigate to the URL, then browser_take_screenshot (and browser_snapshot for DOM). This is real visual verification — use it INSTEAD of WebFetch/curl, which only see raw HTML. The profile is logged in, so auth-walled pages (Vercel previews, dashboards) load directly.`,
          ``,
        ]
      : []),
    `Run the executeQuest procedure from docs/quest-lifecycle.md §2 exactly:`,
    `1. Open the session with sage.preflight (the harness injects it). The right skill books/weapons were researched at PLAN time: each objective's notes may open with "Gear: <book.action, …>" (the digest surfaces it for the next objective) — use those first. Only if no Gear line exists or it doesn't fit: run ONE registry scan via node libs/weapon/skillBooks/cli.mjs "<task>" — do NOT also spawn Explore/general-purpose sub-agents or read skill-book files one by one for the same question (one scan, one answer). Obey ruleDoNotImprovise: NEVER write scratch scripts — use skill books + weapons only (forge via blacksmith if missing).`,
    `2. ORIENT FAST: ${digest ? `the digest above IS questmaster.summarizeQuest's output — do not re-call it unless you mutate quest state and need a refresh.` : `call questmaster.summarizeQuest({questId:"${questId}"}) — one concise digest gives you the stage, the next actionable objective, every OPEN and answered-not-digested question, and the deliverable fill ratio.`} Do NOT raw-read every objective/comment just to learn the status. Then deep-read ONLY what the digest flags: the OPEN/answered questions it lists and any unresolved user comments (getQuestDetails for the full bodies). The user steers you through comments and questions — never start work without handling the ones the digest surfaces.`,
    `3. Clear the inbox FIRST — handle the user's questions and comments BEFORE you touch any objective:`,
    `   a. USER questions (quest_questions where qtype='user') — ANSWER, NEVER delete. Write your reply into the question's 'answer' field (PATCH /api/quest/questions / updateQuestQuestion). A user question is the USER's to remove after they read your answer; you have NO authority to delete or resolve it (the DB guard rejects a worker delete of a user question anyway). Answer EVERY open user question before you execute any objective.`,
    `   b. UNBLOCKING questions (qtype='unblocking') are YOUR questions to the user. If answered=true, DIGEST per questionDigestion's 4 steps — NEVER just delete: (1) EXTRACT the decisions the answer establishes; (2) MAP+APPLY each — global/affects-many → edit the DESCRIPTION, affects ONE objective → write it into THAT objective, introduces NEW work → ADD new objective(s), establishes/changes a needed artifact → the DELIVERABLES/quest_assets; (3) VERIFY each main point is now actually present in the quest (re-read to confirm), capturing anything missing; (4) only THEN remove the question — ONLY via questmaster.digestAndRemoveQuestion({questId, questionId, migrationNote:"<where each decision landed>"}). That is the SOLE sanctioned removal path: it REFUSES without a migrationNote and archives the Q+A first. NEVER raw-delete quest_questions (client.from('quest_questions').delete()) and NEVER call DELETE /api/quest/questions — both bypass the digestion guard and risk destroying the user's decision (the 5d29b94f DATA-LOSS defect). If answered=false it is waiting on the user — leave it, and leave its objective OPEN with the linked unblocking question (the open question marks it blocked — there is no 'attention' status); never answer your own unblocking question.`,
    `   c. USER comments (quest_comments where source='user' and resolved=false) — follow the directive (per commentDigestion), then mark it resolved via questmaster.resolveQuestComment; never delete it (comments are the durable audit log). Record what you changed in a NEW comment.`,
    `4. DELIVERABLES ARE THE SOURCE OF TRUTH — objectives are only the plan, NOT a 1:1 proxy for deliverable completeness. First work the lowest-n active objective that has no open blocking question: produce its share of the deliverables, verify each, ship via worker.shipItemArtifact (upload + upsert the deliverables row) and/or a quest_assets pointer, and update the objective's Progress / Δ / Status. THEN — independent of objective status — inspect EVERY deliverable's ref + visual-verify directly (do NOT assume "objective done" ⇒ "deliverable shipped"): for each deliverable that still has no ref (or a missing/failed verify), PRODUCE it now — even if all objectives already read 'done', an unfilled/unverified deliverable IS unfinished work and IS your job this tick (build the artifact, ship via worker.shipItemArtifact, then questPurrview.autoVisualVerify it). You are NOT done until every deliverable has a non-null ref AND a passing visual-verify; an all-objectives-done quest with an empty deliverable is NOT complete — do not stop, do not escalate, produce the deliverable.`,
    `   • SAVE FORWARD (ruleCheckpointAssets — primary): if a result you produce will be CONSUMED by a LATER objective (an image a PDF-assembly step needs, a data file a later step reads, any artifact that took a non-trivial tool call / >30s to reproduce), save it as a quest_asset NOW — upload to GuildOS_Bucket and insert { quest_id, label, ref, kind, source:"worker", actor_name:"${workerId}" } — before you move on. Do this proactively; it is the main path.`,
    `   • RECOVER BACKWARD (fallback for when SAVE FORWARD was missed): if the objective you are working NEEDS an upstream artifact that has NO quest_asset (a prior objective produced it but didn't archive it), do NOT silently regenerate it and do NOT fabricate. Instead REACTIVATE the prerequisite objective — set its status back to 'active', add a note on it that it MUST archive its result as a quest_asset this time, and leave a comment recording the gap. The next tick re-runs that objective (now archiving the asset); your objective then consumes it. Only proceed once the asset exists.`,
    `5. Before stopping, run sage.askSage. If every objective is done and every deliverable ref is filled → questmaster.moveQuestToReview. If an objective genuinely cannot proceed without the user, RAISE the question via the questmaster skill book's raiseUnblockingQuestionDiplomatically procedure — run sage.askSage FIRST: a reversible/discoverable/defensible-default "block" is NOT a real block (decide it, record it in the objective's notes, keep going); ONLY a genuinely irreversible-if-wrong or user-only-knowledge block earns a written question, and its body MUST state (1) WHAT the user needs to do and (2) WHY you can't do it. Leave that objective OPEN with its linked unblocking question (no status change — the open question + escalated stage ARE the block); once every workable objective is so blocked → moveQuestToEscalated. ALWAYS write exactly ONE structured closing summary comment — this is the durable hand-off the next worker and the user read, so make it bear insight, not narration. Use this exact shape (omit a line only if truly empty):\n      Done: <what landed THIS burst — concrete results/artifacts, not "worked on X">\n      Next: <the single most important thing the next worker should do, with enough specifics to start cold>\n      Blocked: <open question / missing input / decision needed, or "none">\n      Learned: <any gotcha or dead-end worth not repeating, or omit>\n   A summary with no actionable Next is a failed summary — say what to do, not just what happened.`,
    `6. ALWAYS release your lease at the end by clearing the quest's worker_id (questmaster.toggleQuestActiveExecution with release=true, workerId="${workerId}"). The cron's unstuckQuests will reclaim it after 20min if you die first.`,
    ``,
    `Do real work against the real substrate. Verify visually where applicable. Do not improvise deliverables.`,
  ].join("\n");
}

/** Build the leaf-worker prompt for the PURRVIEW visual-verify pass (caravan
 *  `purrview` class). The leaf opens the quest's rendered results in the USER's
 *  channel (Claude-in-Chrome), screenshots them, and reviews for obvious
 *  low-level mistakes before advancing the quest to review. Per the operator SE
 *  spec 2026-06-12, refined 2026-06-13.
 *
 *  BLINDED BY DESIGN: this prompt embeds ONLY the quest description + the
 *  deliverable list (the caravan's VERIFY payload). It deliberately does NOT
 *  carry — and forbids fetching — the objectives, quest_assets, or comments,
 *  because those spell out what each deliverable is supposed to contain, and a
 *  verifier that reads them will parrot that spec as if it had seen it on screen
 *  (the failure observed 2026-06-13). With no crib sheet, the leaf can only
 *  report what it ACTUALLY sees and judge it against the description. */
export function buildLeafPurrviewPrompt({ questId, title, url, workerId, digest = null, description = "", deliverables = [] }) {
  const delLines = (deliverables || []).length
    ? deliverables.map((d, i) => `  ${i + 1}. ${d.name}${d.kind ? ` [${d.kind}]` : ""}${d.status ? ` (status: ${d.status})` : ""}${d.ref ? ` — view: ${d.ref}` : ""}`)
    : [`  (no deliverable rows were handed to you — treat the quest as derailed and bounce to execute)`];
  return [
    `You are a GuildOS PURRVIEW (visual-verify) leaf deployed by the caravan to verify ONE quest's deliverables through the user's own channel.`,
    ``,
    `Quest: "${title}"`,
    `Quest id: ${questId}`,
    `Quest URL: ${url}`,
    `Your worker_id (already leased to you by the caravan): ${workerId}`,
    ``,
    `=== QUEST DESCRIPTION (the ONLY ground truth you are given) ===`,
    description || "(no description on this quest)",
    `=== END QUEST DESCRIPTION ===`,
    ``,
    `=== DELIVERABLES TO VERIFY ===`,
    ...delLines,
    `=== END DELIVERABLES ===`,
    ``,
    ...(digest ? [`=== VERIFY-MODE NOTE ===`, digest, `=== END NOTE ===`, ``] : []),
    `BLINDED VERIFICATION — READ THIS FIRST. You have been given the quest description and the deliverable list ON PURPOSE, and NOTHING ELSE. You do NOT get the objectives, the quest assets, or the comments, because they describe in detail what each deliverable is supposed to contain — and if you could read them you would be tempted to repeat that description as if you had seen it. DO NOT fetch them: do NOT call questmaster.getQuestDetails / summarizeQuest, do NOT query the objectives / quest_assets / quest_comments tables, do NOT read any planning notes. Your ONLY inputs are (a) the quest description above and (b) what you SEE when you open each deliverable. Judge whether what you see matches the description. If you ever find yourself writing a detail you did not actually observe on screen, stop — that is the exact failure this gate exists to catch.`,
    ``,
    `THE CANONICAL CLI — all skill-book/weapon access goes through it (resolves @/ imports):`,
    `  • Run an action:        node --no-warnings libs/adventurer/run-action.mjs <skillBookId> <action> '<inputJson>'`,
    `  • Read ONE procedure:   node --no-warnings libs/adventurer/run-action.mjs --howto <skillBookId> <action>`,
    `Use it ONLY for the verify MUTATIONS below (attach screenshot, move stage, release lease) — not to re-read the quest.`,
    ``,
    `SCOPE — verify ONLY deliverables with status 'active'. Skip any deliverable marked non-active (e.g. done/archived/superseded): it is out of scope, needs no finding and no verification value.`,
    ``,
    `BRANCH BY TYPE — the verification METHOD depends on the deliverable's Type (the [kind] tag above). Two branches:`,
    `  • VISUAL kinds (html · image · document · video): verify by SCREENSHOT — you must open the rendered result and look at the pixels.`,
    `  • TEXT kind: verify by READING the text content — no screenshot. A simple text answer (a number, a paragraph, an explanation) is checked by reading it and stating whether it passes, not by photographing it.`,
    ``,
    `Run the purrview procedure:`,
    `1. sage.preflight (the harness injects it). For VISUAL deliverables the authoritative browser path is the browsercontrol skill book — CIC (Claude-in-Chrome, mcp__Claude_in_Chrome__*) is the ONLY local browser path; never use synthetic-canvas / html2canvas / OS-level screenshots.`,
    `2. For VISUAL deliverables: open the quest details page in CIC and click "view results" (the deliverables carousel), on localhost (dev server, port 3002) — quest URL: ${url}. (Each deliverable above also has a direct "view:" link.) For TEXT deliverables: open the deliverable's "view:" ref (or its View Results slide) and read the actual content.`,
    `3. Gather the evidence per deliverable:`,
    `   • VISUAL: CIC-screenshot the rendered result. Capture the FULL height, or up to 1500px / 3 page-scrolls stitched into ONE image. Use real computer scroll (not JS scrollIntoView) for scroll-driven content.`,
    `   • TEXT: read the whole content. There is nothing to screenshot — the text itself is the evidence.`,
    `4. ACTUALLY LOOK / ACTUALLY READ — evidence alone is NOT acceptance (this gate exists because agents have rubber-stamped without looking). For EACH active deliverable form a structured FINDING:`,
    `   • deliverable: the deliverable name.`,
    `   • depicts: one line of what is actually on screen / in the text.`,
    `   • probes: answer the checks implied by the QUEST DESCRIPTION above (e.g. a Chinese 5-chapter picture book: body text Chinese? 5 chapters? each chapter has an image? / a text answer: is the value correct? is the reasoning sound?). Answer each yes/no with the observed value — derived from the description, NOT from any spec you fetched.`,
    `   • readback: QUOTE verbatim a specific piece of the rendered/read content (a chapter title, the actual answer text). Proof-of-looking — must be real content, and it is audited.`,
    `   • verdict: 'match' (matches the description) or 'mismatch' (with what is wrong/missing).`,
    `4b. Record ALL findings in ONE call — questmaster.recordVerifyPass({questId:"${questId}", passType:"purrview", workerId:"${workerId}", findings:[ {deliverable, depicts, probes, readback, verdict, mismatch?}, … ]}). ONE structured verdict-bearing quest_comment for the whole pass, UPSERTed — do NOT hand-write one comment per deliverable. It rejects any finding lacking depicts or a verbatim readback.`,
    `5. Record the "Purrview Verification" value on each deliverable (deliverables.verify_screenshot_ref) — the METHOD differs by Type:`,
    `   • VISUAL: UPLOAD the screenshot to Supabase Storage (GuildOS_Bucket, path: quest-assets/<questId>/verify/<deliverableId>.png) via the supabase_storage weapon, then set verify_screenshot_ref to the PUBLIC URL (https://*.supabase.co/storage/*). NEVER a local file path.`,
    `   • TEXT: set verify_screenshot_ref to a short verdict string beginning with PASS or FAIL — e.g. "PASS — answer is 72, correct and clearly stated" or "FAIL — asked for 8×9 but the text says 63". This IS the text deliverable's Purrview Verification; there is no screenshot. (questmaster mutation, NOT raw curl.)`,
    `6. Decide: if any active deliverable mismatches the description (or its text verdict is FAIL) → bounce back to execute (questmaster.moveQuestToExecute) with a comment naming exactly what is wrong; if every active deliverable matches AND has a payload ref AND a Purrview Verification value (screenshot URL for visual, PASS verdict for text) AND a recorded finding → questmaster.moveQuestToReview. Do NOT advance any deliverable whose finding is missing or generic.`,
    `7. ALWAYS release your lease at the end (questmaster.toggleQuestActiveExecution release=true, workerId="${workerId}"). The caravan rescue reclaims it after the stale window if you die first.`,
  ].join("\n");
}

/** Build the leaf-worker prompt for the CLOSING stage (caravan `closing` class).
 *  The archive leaf decides what of a finished quest must be ARCHIVED before the
 *  quest is closed, performs the auto-approved archive (post deliverables + a
 *  completion report to the source Asana task), and escalates any archiving that
 *  needs a user decision (email, applying discovered EDs/insights to a
 *  repo/substrate, any non-Asana target, an unresolved finale comment). It does
 *  NOT flip the stage itself — questmaster.archiveQuestTakeaways owns the
 *  close-or-escalate transition. */
export function buildLeafArchivePrompt({ questId, title, url, workerId, digest = null, description = "", deliverables = [] }) {
  const delLines = (deliverables || []).length
    ? deliverables.map((d, i) => `  ${i + 1}. ${d.name}${d.kind ? ` [${d.kind}]` : ""}${d.ref ? ` — ${d.ref}` : " — (no ref)"}`)
    : [`  (no deliverable rows on this quest)`];
  return [
    `You are a GuildOS CLOSING (archive-takeaways) leaf deployed by the caravan to wrap up ONE quest that has reached the closing stage.`,
    ``,
    `Quest: "${title}"`,
    `Quest id: ${questId}`,
    `Quest URL: ${url}`,
    `Your worker_id (already leased to you by the caravan): ${workerId}`,
    ``,
    ...(digest ? [`=== ORIENTATION DIGEST ===`, digest, `=== END DIGEST ===`, ``] : []),
    `=== QUEST DESCRIPTION (its SOURCE linkage is here — e.g. "SOURCE ASANA TASK: ... gid <n>") ===`,
    description || "(no description)",
    `=== END DESCRIPTION ===`,
    ``,
    `=== DELIVERABLES PRODUCED ===`,
    ...delLines,
    `=== END DELIVERABLES ===`,
    ``,
    `THE CANONICAL CLI — all skill-book/weapon access goes through it (resolves @/ imports):`,
    `  • Run an action:  node --no-warnings libs/adventurer/run-action.mjs <skillBookId> <action> '<inputJson>'`,
    `  • Read a howTo:   node --no-warnings libs/adventurer/run-action.mjs --howto <skillBookId> <action>`,
    ``,
    `THE CLOSING JOB — archive the takeaways, then close (or escalate). In order:`,
    `1. sage.preflight (the harness injects it). Orient from the digest + description + deliverables above; deep-read only what you must (questmaster.getQuestDetails for objective NOTES + comments) to learn the real takeaways and spot any insight/ED worth keeping.`,
    `2. COMPOSE a concise completion REPORT (plain text): what was delivered, the key outcome(s), and where each deliverable lives (name + link). Write it for the human who owns the source task, not as narration.`,
    `3. CLASSIFY the archiving this quest needs:`,
    `   • AUTO-APPROVED (the engine does it for you): post the report + deliverables as a comment to the SOURCE ASANA TASK named in the description. This is the ONLY auto-approved, completion-worthy archive today, and it is the common case.`,
    `   • NEEDS A USER DECISION (you must escalate — do NOT do it yourself): emailing the results to someone; applying any discovered ED / insight / method to a repo or substrate (code, skill book, docs); any archiving target other than the source Asana task. Collect each as a one-line "pendingDecision" string saying WHAT needs deciding and WHY it needs the user. (Unresolved action='closing' finale comments are folded in automatically.)`,
    `4. CALL THE ENGINE — it performs the auto Asana archive, then closes or escalates:`,
    `   node --no-warnings libs/adventurer/run-action.mjs questmaster archiveQuestTakeaways '{"questId":"${questId}","report":"<your report>","pendingDecisions":[<zero or more strings>]}'`,
    `   • pendingDecisions EMPTY → the engine posts to Asana and CLOSES the quest (closing→complete).`,
    `   • pendingDecisions NON-EMPTY → the engine still posts the Asana archive when it can, then raises each as an unblocking question and sets the quest to ESCALATED. Never close a quest that has pending decisions.`,
    `5. LESSON DISTILLATION (self-improvement loop):`,
    `   5a. Check the complexity gate: node --no-warnings libs/adventurer/run-action.mjs questmaster shouldDistillLesson '{"questId":"${questId}"}'`,
    `   5b. If shouldDistill=true, examine the quest's execution history (objectives, comments, EDs) for genuinely REUSABLE lessons — counter-intuitive decisions, non-obvious patterns, or context-dependent insights that a future agent would otherwise get wrong. Apply the same selectivity bar as non-native EDs: if a competent agent would reach this conclusion on its own, it is NOT a lesson.`,
    `   5b-i. DIRECT each candidate by the run-evaluation objective function (libs/quest/engineering_decisions.md → Deliverable 0b). A lesson is justified ONLY if it moves the corpus along a NAMED axis: ↑reuse (generalize to the CLASS of task, description-matchable — not this one quest), ↑effectiveness (a real correction/technique/verification pattern), or ↓always-on footprint. State, in the lesson description, WHICH axis it improves and how. If you cannot name one, it is not a lesson — skip it.`,
    `   5b-ii. HARD GATE (reject even a useful lesson that fails this): the skill-book/ED index loads on every future leaf, so anything that bloats always-on context or destabilizes the skill index is cache-hostile and REJECTED. Keep any new skill-book description ≤60 chars, class-level, no per-quest/PR/error-string names. Prefer PATCHING an existing skill book (blacksmith.patchSkillBook) over creating a new one.`,
    `   5b-iii. DO NOT capture (these harden into self-imposed refusals — negative-value skills): environment/setup failures (missing creds, uninstalled deps, 'command not found'), negative tool-claims ('X is broken', 'can't use Y'), transient errors that resolved before close, and one-off task narratives. If a tool failed on setup state, capture the FIX (the command/config), never 'this doesn't work'.`,
    `   5c. For each qualifying lesson that involves a CODE CHANGE (skill book update, ED edit, weapon tweak):`,
    `       i.  Create a branch: git checkout -b lesson/${questId.slice(0,8)}/<short-slug> origin/main`,
    `       ii. Make the change, commit it.`,
    `       iii. Push: git push -u origin lesson/${questId.slice(0,8)}/<short-slug>`,
    `       iv. Record the lesson WITH the git_ref (branch name): node --no-warnings libs/adventurer/run-action.mjs questmaster distillQuestLesson '{"questId":"${questId}","description":"<what + how to smoke-test>","gitRef":"lesson/${questId.slice(0,8)}/<short-slug>"}'`,
    `       v.  Return to the working branch afterward: git checkout -`,
    `   5d. For lessons that are PURE INSIGHT (no code change needed), omit gitRef.`,
    `   5e. Most quests yield ZERO lessons. That is normal. Do not force a lesson where none exists.`,
    `6. Do NOT write quests.stage yourself and do NOT invent new archiving channels — archiveQuestTakeaways owns the transition; anything beyond the Asana archive is the user's call (escalate it).`,
    `7. ALWAYS release your lease at the end (questmaster.toggleQuestActiveExecution release=true, workerId="${workerId}"). The caravan rescue reclaims it after the stale window if you die first.`,
  ].join("\n");
}

/**
 * Cron:questTick — the deterministic stage-router (docs/quest-lifecycle.md §2).
 *
 * One tick:
 *   1. RENDER the QUEST_TICK_ACTIONS registry (so the run is self-describing).
 *   2. reclaim-stale-leases  → unstuckQuests (clear leases > staleMinutes old).
 *   2b. dispatch-plan        → for each owned plan-stage quest with no lease:
 *      lease, then deploy a PLANNING leaf (buildLeafPlanPrompt) that itemizes
 *      the description into objectives/deliverables/questions per the
 *      questmaster planQuest howTo and transitions the quest to execute.
 *   3. dispatch-execute      → for each owned execute-stage quest with an active
 *      objective and no lease: lease via toggleQuestActiveExecution, then deploy
 *      a leaf claude agent (claudecli, fire-and-forget) to work it. Capped at
 *      maxDispatch leaves per tick. The lease makes re-ticks idempotent.
 *   4. scan-purrview         → report purrview-stage quests (review path owns dispatch).
 *
 * Kill-switch: GUILDOS_DISABLE_QUESTTICK=1 renders the registry + scan but
 * dispatches nothing (safe observe mode). `dispatch:false` does the same per-call.
 *
 * Concurrency: leaf workers run `claude --print` from the SHARED main checkout,
 * so two leaves editing the same tree would collide on git/files. `maxConcurrent`
 * (default 1) caps the number of simultaneously-leased quests — a single-flight
 * guard on the shared checkout. True parallelism needs per-worker worktree
 * isolation (deferred — see the tracking quest). `preferQuestId` moves one quest
 * to the front of the dispatch order (used for the b55db5d6 dogfood).
 *
 * @param {{ assignee?:string, dispatch?:boolean, maxConcurrent?:number, staleMinutes?:number, preferQuestId?:string }} opts
 * @returns {Promise<{ ok:boolean, ts:string, assignee:string, registry:object[], reclaimed:object[], dispatched:object[], skipped:object[], purrview:object[] }>}
 */
export async function questTick({ assignee = "Main_Claude", dispatch = true, maxConcurrent = 1, staleMinutes = 20, preferQuestId = null, forceQuestId = null } = {}) {
  const ts = new Date().toISOString();
  const client = await getServiceClient();
  const killed = process.env.GUILDOS_DISABLE_QUESTTICK === "1";
  const willDispatch = dispatch && !killed;

  // Resolve the assignee's worker RUNTIME once and route every leaf this tick
  // through it — this is what makes the manual "Trigger tick" honor the quest's
  // GUILD by assignee (GuildOS = claude workers, Hermes = codex-runtime workers).
  // Uniform per tick because every dispatch select below is .eq("assignee",
  // assignee), so the runtime is the same for all quests this call touches.
  //   - claude workers keep the claudecli launcher — it is the ONLY path that
  //     threads --mcp-config for [BROWSER] leaves (invokeAdventurerLeaf has no
  //     mcpConfig token), so browser-enabled GuildOS quests are unaffected.
  //   - every other worker_type (hermes/codex/…) routes through
  //     invokeAdventurerLeaf, which renders the adventurer's own system_prompt
  //     start-command template — the exact same vendor-agnostic launcher the
  //     caravan cron already uses. Fail-open to claude if the row can't be read.
  let workerType = "claude";
  try {
    const { data: adv } = await client
      .from(publicTables.adventurers)
      .select("worker_type")
      .eq("name", assignee)
      .maybeSingle();
    if (adv?.worker_type) workerType = adv.worker_type;
  } catch { /* fail-open to claude */ }
  const dispatchLeaf = (prompt, { logPath, mcpConfigPath = null } = {}) =>
    workerType === "claude"
      ? claudecliInvoke(prompt, { logPath, streamJson: true, mcpConfigPath })
      : invokeAdventurerLeaf(prompt, assignee, { client, logPath, streamJson: true });

  // 1. Render the registry — the tick is self-describing.
  const registry = QUEST_TICK_ACTIONS.map((a) => ({ id: a.id, stage: a.stage, description: a.description }));
  console.log(`[questTick] ${ts} assignee=${assignee} dispatch=${willDispatch}${killed ? " (GUILDOS_DISABLE_QUESTTICK=1)" : ""} — registry:`);
  for (const a of registry) console.log(`[questTick]   • ${a.id} [${a.stage}] — ${a.description}`);

  // 2. reclaim-stale-leases
  const { reclaimed } = await unstuckQuests({ assignee, staleMinutes });

  // 2a. plan-stale guard (SIO #7) — escalate any plan-stage quest owned by this
  //     assignee that has 0 objectives and is older than the threshold, BEFORE
  //     dispatch-plan below re-attempts it. dispatch-plan already re-runs planQuest
  //     every tick (continuous re-plan), so this is the terminal for when planning
  //     keeps failing: rather than let a never-itemized quest rot silently at plan,
  //     surface it to the user with an explicit reason. Runs before 2b so an
  //     escalated quest leaves the plan stage and 2b won't re-lease it (were it
  //     after 2b, the re-lease would hide the quest from this guard's unleased
  //     filter every tick). Never throws — a guard failure must not abort the tick.
  let planStale = { escalated: [] };
  try { planStale = await escalateStalePlanQuests({ client, assignee, dispatch: willDispatch }); }
  catch (e) { planStale = { escalated: [], error: e?.message || String(e) }; }

  // 2b. dispatch-plan — deploy a PLANNING leaf to each plan-stage quest owned by
  //     assignee with no lease. The leaf runs the questmaster planQuest howTo
  //     (itemize the description → objectives/deliverables/questions, then
  //     moveQuestToExecute) and verifies the done-condition before stopping.
  //     Lease-before-dispatch keeps re-ticks idempotent, same as execute/purrview.
  //     NOTE: planning slots are budgeted BEFORE execute/purrview below — a
  //     quest that can't even be worked yet (no objectives) is the most
  //     upstream blockage, so it drains first.
  //     Orphan adoption removed — the standalone assignOrphans() in
  //     libs/caravan/index.js is the authoritative safety net (runs at the front
  //     of the rescue flow, before any dispatcher). questTick only selects quests
  //     already assigned to its own assignee.
  const { data: planQuests } = await client
    .from(publicTables.quests)
    .select("id, title, worker_id, assignee")
    .eq("assignee", assignee)
    .eq("stage", "plan")
    .order("created_at", { ascending: true });
  const planned = [];
  let planSlots = maxConcurrent - (planQuests || []).filter((q) => q.worker_id).length;
  for (const pq of (planQuests || [])) {
    if (pq.worker_id) {
      planned.push({ questId: pq.id, title: pq.title, dispatched: false, reason: `already leased by "${pq.worker_id}"` });
      continue;
    }
    const isForced = !!forceQuestId && pq.id === forceQuestId;
    if (planSlots <= 0 && !isForced) {
      planned.push({ questId: pq.id, title: pq.title, dispatched: false, reason: `single-flight: ${maxConcurrent} leaf already active` });
      continue;
    }
    if (!willDispatch) {
      planSlots -= 1;
      planned.push({ questId: pq.id, title: pq.title, dispatched: false, reason: "observe-mode (would deploy planning leaf)" });
      continue;
    }
    const planWorkerId = `plan-${pq.id.slice(0, 8)}-${Date.now().toString(36)}`;
    const lease = await toggleQuestActiveExecution({ questId: pq.id, workerId: planWorkerId });
    if (!lease.ok || !lease.leased) {
      planned.push({ questId: pq.id, title: pq.title, dispatched: false, reason: `lease lost (heldBy=${lease.heldBy || "?"})` });
      continue;
    }
    planSlots -= 1;
    const planUrl = questPath(pq.id);
    const planLogPath = join(process.cwd(), ".worker-logs", `${planWorkerId}.log`);
    // HARD QUESTIONS GATE — if the plan quest has open questions, the leaf gets a
    // questions-only prompt (answer/digest/escalate, then stop); no planning until
    // the inbox is cleared (operator directive 2026-06-13).
    const { data: planQ } = await client
      .from(publicTables.questQuestions)
      .select("id, qtype, body, answer, answered")
      .eq("quest_id", pq.id);
    const planGate = computeQuestionGate(planQ || []);
    const planPrompt = buildLeafPlanPrompt({ questId: pq.id, title: pq.title, url: planUrl, workerId: planWorkerId, questionGate: planGate });
    dispatchLeaf(planPrompt, { logPath: planLogPath })
      .then((res) => console.log(`[questTick] planning leaf ${planWorkerId} finished quest ${pq.id} (ok=${res?.ok})`))
      .catch((err) => console.error(`[questTick] planning leaf ${planWorkerId} threw on quest ${pq.id}: ${err?.message || err}`))
      .finally(async () => {
        try { await toggleQuestActiveExecution({ questId: pq.id, workerId: planWorkerId, release: true }); } catch { /* unstuck reclaims */ }
      });
    planned.push({ questId: pq.id, title: pq.title, dispatched: true, workerId: planWorkerId });
    console.log(`[questTick] dispatched PLANNING leaf ${planWorkerId} → plan quest "${pq.title}" (${pq.id})`);
  }

  // 2c. kickback-on-user-comment — a review/escalated quest whose LATEST comment
  //     is from the user is a signal to re-open it: bounce stage → execute so the
  //     dispatch-execute step below picks it up THIS tick. Spares the operator
  //     from manually reassigning the stage. Idempotent: we write a system
  //     'user_kickback' comment, which becomes the new latest comment, so the
  //     next tick won't re-bounce (latest is no longer the user's).
  const kickedBack = [];
  const { data: openStageQuests } = await client
    .from(publicTables.quests)
    .select("id, title, stage")
    .eq("assignee", assignee)
    .in("stage", ["review", "escalated"]);
  for (const oq of openStageQuests || []) {
    const { data: lastComment } = await client
      .from(publicTables.questComments)
      .select("source, created_at")
      .eq("quest_id", oq.id)
      .order("created_at", { ascending: false })
      .limit(1)
      .maybeSingle();
    if (!lastComment || lastComment.source !== "user") continue;
    if (!willDispatch) {
      kickedBack.push({ questId: oq.id, title: oq.title, from: oq.stage, wouldKickback: true });
      continue;
    }
    const { error: upErr } = await client.from(publicTables.quests).update({ stage: "execute" }).eq("id", oq.id);
    if (upErr) {
      kickedBack.push({ questId: oq.id, title: oq.title, from: oq.stage, ok: false, error: upErr.message });
      continue;
    }
    // NO quest_comment for the bounce. Comments are the strategic/future-bearing
    // channel, not an action log — and one isn't needed here: the stage change is
    // audited in quest_stage_log, bouncing to execute removes the quest from this
    // kickback scan (so no re-bounce), and the worker reads the USER's actual
    // comment (the real directive) via summarizeQuest rather than a system note.
    kickedBack.push({ questId: oq.id, title: oq.title, from: oq.stage, ok: true });
    console.log(`[questTick] kickback "${oq.title}" (${oq.id}) ${oq.stage} → execute (latest comment from user)`);
  }

  // 3. dispatch-execute — find execute-stage quests owned by assignee with no lease.
  const { data: execQuests, error: qErr } = await client
    .from(publicTables.quests)
    .select("id, title, description, worker_id, created_at")
    .eq("assignee", assignee)
    .eq("stage", "execute")
    .order("created_at", { ascending: true });
  if (qErr) {
    const out = { ok: false, ts, assignee, registry, reclaimed, dispatched: [], skipped: [], purrview: [], error: qErr.message };
    logWeaponCall({ weapon: "questmaster", fn: "questTick", input: { assignee, dispatch: willDispatch }, output: { error: qErr.message }, status: "fail" });
    return out;
  }

  // Order quests: preferQuestId first (dogfood/targeted), else oldest-first (fair).
  let queue = execQuests || [];
  if (preferQuestId) {
    queue = [...queue].sort((a, b) => (a.id === preferQuestId ? -1 : b.id === preferQuestId ? 1 : 0));
  }

  // Single-flight: count quests already leased; only (maxConcurrent - leased)
  // new leaves may launch this tick (shared-checkout collision guard).
  // Plan-stage leases (pre-existing + the planning leaves just dispatched in
  // 2b) share the same budget — every leaf runs from the shared checkout.
  const planLeased =
    (planQuests || []).filter((q) => q.worker_id).length +
    planned.filter((p) => p.dispatched).length;
  const alreadyLeased = queue.filter((q) => q.worker_id).length + planLeased;
  let slots = Math.max(0, maxConcurrent - alreadyLeased);

  const dispatched = [];
  const skipped = [];
  const advanced = []; // execute quests with all objectives done → moveQuestToReview
  for (const quest of queue) {
    if (quest.worker_id) {
      skipped.push({ questId: quest.id, reason: `already leased by "${quest.worker_id}"` });
      continue;
    }
    // A manual/targeted trigger (forceQuestId) bypasses the single-flight budget
    // for THAT quest only — the operator deliberately deployed it. Its own lease
    // is still respected (checked above), so there is no double-deploy.
    const isForced = !!forceQuestId && quest.id === forceQuestId;
    // The single-flight slot gate is applied BELOW, only to leaf DISPATCH (a heavy
    // claude --print on the shared checkout). It must NOT gate the cheap
    // moveQuestToReview advancement of done quests (a deterministic DB call) —
    // otherwise a single busy leaf strands every completed quest in execute.
    // Worker-actionable work present? An active objective, OR an inbox item the
    // worker must handle: an open user question to answer, an answered unblocking
    // question to digest, or an unresolved user comment/directive to follow.
    // (A quest with a fresh question but no active objective still gets a worker.)
    const { data: objs } = await client
      .from("objectives")
      .select("id, n, name, status")
      .eq("quest_id", quest.id)
      // Tolerate non-canonical synonyms (todo/active) so a stray status word never
      // hides an execute-stage quest from dispatch. Canonical: pending/running.
      .in("status", ["pending", "running", "todo", "active"])
      .order("n", { ascending: true });
    const firstObjective = objs && objs[0];
    let workReason = firstObjective ? `objective ${firstObjective.n} — ${firstObjective.name}` : null;
    if (!workReason) {
      const { data: aq } = await client
        .from(publicTables.questQuestions)
        .select("id")
        .eq("quest_id", quest.id)
        .or("and(qtype.eq.user,answered.eq.false),and(qtype.eq.unblocking,answered.eq.true)")
        .limit(1);
      if (aq && aq.length) workReason = "an actionable question (answer/digest)";
    }
    if (!workReason) {
      const { data: uc } = await client
        .from(publicTables.questComments)
        .select("id")
        .eq("quest_id", quest.id)
        .eq("source", "user")
        .eq("resolved", false)
        .limit(1);
      if (uc && uc.length) workReason = "an unresolved user comment (directive)";
    }
    if (!workReason) {
      // No active objective and no inbox. If the quest HAS objectives and they
      // are ALL done, it is a COMPLETED execute quest stranded here — ADVANCE it
      // (moveQuestToReview runs the submit gate: all-done + deliverables filled →
      // purrview; else it stays in execute and leaves a helpful comment). This is
      // the path that un-strands "all objectives done" execute quests, which the
      // dispatcher used to skip forever as "no work".
      const { data: allObjs } = await client
        .from("objectives")
        .select("status")
        .eq("quest_id", quest.id);
      const total = (allObjs || []).length;
      const doneCount = (allObjs || []).filter((o) => o.status === "done").length;
      if (total > 0 && doneCount === total) {
        if (!willDispatch) {
          advanced.push({ questId: quest.id, title: quest.title, wouldAdvance: true });
        } else {
          try {
            const adv = await moveQuestToReview({ questId: quest.id });
            advanced.push({ questId: quest.id, title: quest.title, ok: adv.ok, stage: adv.stage, reason: adv.ok ? null : adv.report?.msg || adv.error });
            console.log(`[questTick] advance "${quest.title}" (${quest.id}) → ${adv.ok ? adv.stage : "stayed (gate fail)"}`);
          } catch (err) {
            advanced.push({ questId: quest.id, title: quest.title, ok: false, reason: err?.message });
          }
        }
        continue;
      }
      skipped.push({ questId: quest.id, reason: total === 0 ? "no objectives (nothing to work or advance)" : "no active objective and no pending inbox (question/comment)" });
      continue;
    }

    // Single-flight slot gate — applies ONLY to leaf dispatch (heavy, shared
    // checkout), not to the advancement path above.
    if (slots <= 0 && !isForced) {
      skipped.push({ questId: quest.id, reason: `single-flight: ${maxConcurrent} leaf already active (shared checkout)` });
      continue;
    }

    const workerId = `leaf-${quest.id.slice(0, 8)}-${Date.now().toString(36)}`;
    const url = questPath(quest.id);

    if (!willDispatch) {
      slots -= 1;
      skipped.push({ questId: quest.id, reason: "observe-mode (no dispatch)", wouldDispatch: { workerId, work: workReason, forced: isForced } });
      continue;
    }

    // Lease before dispatch (compare-and-set) so two ticks never double-deploy.
    const lease = await toggleQuestActiveExecution({ questId: quest.id, workerId });
    if (!lease.ok || !lease.leased) {
      skipped.push({ questId: quest.id, reason: `lease lost (heldBy=${lease.heldBy || "?"})` });
      continue;
    }
    slots -= 1;

    // Deploy the leaf — fire-and-forget; release the lease when it returns.
    // Tee the leaf's live transcript into .worker-logs/<workerId>.log so the
    // quest UI's worker-log button can show what the leaf is doing.
    const logPath = join(process.cwd(), ".worker-logs", `${workerId}.log`);
    // Pre-fetch the orientation digest and inline it into the leaf prompt — the
    // dispatcher already holds a DB client, so this converts the leaf's first
    // 1-2 agentic round-trips (summarizeQuest + status reads) into zero. The
    // leaf launches immediately after, so staleness is negligible; it can still
    // re-call summarizeQuest mid-burst for a refresh.
    let digest = null;
    try {
      const sum = await summarizeQuest({ questId: quest.id });
      if (sum.ok) digest = sum.digest;
    } catch { /* leaf falls back to calling summarizeQuest itself */ }

    // Browser-enable: a quest tagged [BROWSER] in its title/description gets an
    // AUTHENTICATED headless Playwright browser (a per-leaf copy of the captured
    // master Chrome profile) injected via --mcp-config. SELF-ARMING with zero
    // operator action: ensureMasterFromChrome() opportunistically copies the
    // operator's LIVE Chrome login (cookie included) whenever Chrome isn't holding
    // Cookies under its exclusive lock — i.e. the next time it's closed. It gates
    // on the actual session COOKIE (hasMasterCookie), not mere profile presence,
    // so a [BROWSER] leaf is never dispatched against a cookieless profile that
    // would just bounce to /signin. Stays a NO-OP (unauth, unchanged dispatch)
    // until that first cookie capture succeeds.
    const wantsBrowser = /\[BROWSER\]/i.test(`${quest.title || ""} ${quest.description || ""}`);
    let mcpConfigPath = null;
    if (wantsBrowser && leafBrowser.ensureMasterFromChrome()) {
      const prep = leafBrowser.prepareLeafBrowser({ workerId });
      if (prep.ok) mcpConfigPath = prep.mcpConfigPath;
      else console.warn(`[questTick] leaf ${workerId} browser prep failed: ${prep.reason}`);
    }

    // HARD QUESTIONS GATE — if the quest has open questions, the leaf gets a
    // questions-only prompt (answer/digest/escalate, then stop); no execution
    // until the inbox is cleared (operator directive 2026-06-13).
    const { data: execQ } = await client
      .from(publicTables.questQuestions)
      .select("id, qtype, body, answer, answered")
      .eq("quest_id", quest.id);
    const execGate = computeQuestionGate(execQ || []);
    const prompt = buildLeafExecutePrompt({ questId: quest.id, title: quest.title, url, workerId, digest, browser: !!mcpConfigPath, questionGate: execGate });
    dispatchLeaf(prompt, { logPath, mcpConfigPath })
      .then((res) => {
        console.log(`[questTick] leaf ${workerId} finished quest ${quest.id} (ok=${res?.ok})`);
      })
      .catch((err) => {
        console.error(`[questTick] leaf ${workerId} threw on quest ${quest.id}: ${err?.message || err}`);
      })
      .finally(async () => {
        if (mcpConfigPath) { try { leafBrowser.cleanupLeafBrowser({ workerId }); } catch { /* best-effort */ } }
        try { await toggleQuestActiveExecution({ questId: quest.id, workerId, release: true }); } catch { /* unstuck will reclaim */ }
      });

    // NO quest_comment for the dispatch event. A dispatch is a START marker — it
    // has zero "what was done / what next" insight, so writing it into the
    // quest_comments thread just clutters the channel humans + the next worker
    // read for actionable intel. The dispatch is already fully recorded where it
    // belongs (auditable, non-cluttering): cron_log.log.questTick.dispatched
    // (below), the worker_id lease (worker_claimed_at), and .worker-logs/<id>.log.
    // The INSIGHT comment is the leaf's structured closing summary (Done/Next/
    // Blocked) at end-of-burst — that is what carries forward what happened.
    dispatched.push({
      questId: quest.id,
      title: quest.title,
      url,
      workerId,
      work: workReason,
      objectiveN: firstObjective?.n ?? null,
      forced: isForced,
      runtime: workerType,
    });
    console.log(`[questTick] dispatched leaf ${workerId} → quest "${quest.title}" (${quest.id})`);
  }

  // 4. dispatch-purrview — deploy a REVIEW leaf to each owned purrview quest with
  //    no lease, within the shared single-flight slot budget. The review leaf runs
  //    confirmSubmission → runReviewLoop → approve/bounce.
  const { data: purrviewRows } = await client
    .from(publicTables.quests)
    .select("id, title, worker_id")
    .eq("assignee", assignee)
    .eq("stage", "purrview")
    .order("created_at", { ascending: true });
  const purrview = [];
  for (const pq of purrviewRows || []) {
    if (pq.worker_id) {
      purrview.push({ questId: pq.id, title: pq.title, leased: true, dispatched: false, reason: `already leased by "${pq.worker_id}"` });
      continue;
    }
    const isForced = !!forceQuestId && pq.id === forceQuestId;
    if (slots <= 0 && !isForced) {
      purrview.push({ questId: pq.id, title: pq.title, leased: false, dispatched: false, reason: `single-flight: ${maxConcurrent} leaf already active` });
      continue;
    }
    if (!willDispatch) {
      slots -= 1;
      purrview.push({ questId: pq.id, title: pq.title, leased: false, dispatched: false, reason: "observe-mode (would deploy review leaf)" });
      continue;
    }
    const workerId = `review-${pq.id.slice(0, 8)}-${Date.now().toString(36)}`;
    const lease = await toggleQuestActiveExecution({ questId: pq.id, workerId });
    if (!lease.ok || !lease.leased) {
      purrview.push({ questId: pq.id, title: pq.title, leased: false, dispatched: false, reason: `lease lost (heldBy=${lease.heldBy || "?"})` });
      continue;
    }
    slots -= 1;
    // Headless visual verification (no CIC needed): render + capture each
    // unverified deliverable's screenshot and set verify_screenshot_ref, BEFORE
    // the review leaf runs — so the leaf can satisfy the approve gate's visual
    // check autonomously instead of escalating (headless leaves have no browser).
    try {
      const cap = await autoVisualVerify({ questId: pq.id });
      if (cap?.captured?.length) console.log(`[questTick] auto-captured ${cap.captured.length} verify screenshot(s) for purrview quest ${pq.id}`);
      if (cap?.errors?.length) console.warn(`[questTick] autoVisualVerify errors for ${pq.id}:`, JSON.stringify(cap.errors));
    } catch (e) {
      console.error(`[questTick] autoVisualVerify threw for ${pq.id}: ${e?.message || e}`);
    }
    const url = questPath(pq.id);
    const logPath = join(process.cwd(), ".worker-logs", `${workerId}.log`);
    let reviewDigest = null;
    try {
      const sum = await summarizeQuest({ questId: pq.id });
      if (sum.ok) reviewDigest = sum.digest;
    } catch { /* leaf falls back to calling summarizeQuest itself */ }
    const prompt = buildLeafReviewPrompt({ questId: pq.id, title: pq.title, url, workerId, digest: reviewDigest });
    dispatchLeaf(prompt, { logPath })
      .then((res) => console.log(`[questTick] review leaf ${workerId} finished quest ${pq.id} (ok=${res?.ok})`))
      .catch((err) => console.error(`[questTick] review leaf ${workerId} threw on quest ${pq.id}: ${err?.message || err}`))
      .finally(async () => {
        try { await toggleQuestActiveExecution({ questId: pq.id, workerId, release: true }); } catch { /* unstuck reclaims */ }
      });
    purrview.push({ questId: pq.id, title: pq.title, leased: true, dispatched: true, workerId });
    console.log(`[questTick] dispatched REVIEW leaf ${workerId} → purrview quest "${pq.title}" (${pq.id})`);
  }

  const out = { ok: true, ts, assignee, registry, reclaimed, planStale: planStale.escalated, planned, kickedBack, dispatched, advanced, skipped, purrview };
  console.log(`[questTick] done — reclaimed=${reclaimed.length} planStale=${planStale.escalated.length} planned=${planned.length} kickedBack=${kickedBack.length} dispatched=${dispatched.length} advanced=${advanced.length} skipped=${skipped.length} purrview=${purrview.length}`);
  logWeaponCall({ weapon: "questmaster", fn: "questTick", input: { assignee, dispatch: willDispatch, maxConcurrent }, output: { reclaimed: reclaimed.length, planned: planned.length, dispatched: dispatched.length, skipped: skipped.length }, status: "ok" });
  return out;
}

// ── questUpdate (comment-first half) ──────────────────────────────────────────

/**
 * Comment-first audit write for a mid-flight quest change. Writes the user's
 * request to quest_comments BEFORE any mutation is rendered, then returns the
 * new comment id + the quest's current objectives so the dedicated update agent
 * can classify + target one attribute. The mutation itself goes through
 * `patchObjective` (or a description/assignee/deliverables write); the request
 * comment is closed via `resolveQuestComment` only AFTER the change lands.
 *
 * @param {{ questId:string, requestSummary:string, actorName?:string }} opts
 * @returns {Promise<{ ok:true, commentId:string, objectives:Array<object> } | { ok:false, error:string }>}
 */
export async function questUpdate({ questId, requestSummary, actorName = "user" } = {}) {
  if (!questId || !requestSummary || !String(requestSummary).trim()) {
    const out = { ok: false, error: "questId and a non-empty requestSummary are required" };
    logWeaponCall({ weapon: "questmaster", fn: "questUpdate", input: { hasQuest: !!questId }, output: out, status: "fail" });
    return out;
  }
  const client = await getServiceClient();

  const { recordQuestComment } = await import("@/libs/quest/index.js");
  const { data: comment, error: cErr } = await recordQuestComment(questId, {
    summary: String(requestSummary).trim(),
    actor_name: actorName,
    source: "user",
    action: "update_request",
    resolved: false,
  }, { client });

  if (cErr || !comment?.id) {
    const out = { ok: false, error: cErr?.message || "comment insert returned no id" };
    logWeaponCall({ weapon: "questmaster", fn: "questUpdate", input: { questId }, output: out, status: "fail" });
    return out;
  }

  const { data: objectives, error: sErr } = await client
    .from("objectives")
    .select("id, n, name, status, progress, delta, notes")
    .eq("quest_id", questId)
    .order("n", { ascending: true });

  const out = { ok: true, commentId: comment.id, objectives: objectives || [] };
  logWeaponCall({
    weapon: "questmaster",
    fn: "questUpdate",
    input: { questId },
    output: { commentId: comment.id, objectiveCount: (objectives || []).length, objectivesError: sErr?.message || null },
    status: "ok",
  });
  return out;
}

// ── patchObjective (render half) ──────────────────────────────────────────────

const PATCHABLE_OBJECTIVE_FIELDS = new Set(["name", "progress", "delta", "notes", "status"]);

/**
 * Write the required execute-burst handoff comment without misclassifying it as
 * a user refinement. The summary must already be structured by the caller.
 *
 * SIO #2: action='worker_summary' is a WORKER_HEARTBEAT action, so the chokepoint
 * (recordQuestComment) now UPSERTS this row per quest (edited in place, created_at
 * bumped) instead of appending a fresh burst row every tick. commentId is returned
 * on both the first insert and later in-place updates. For the canonical per-tick
 * status line prefer writeWorkerStatus (action='worker_status').
 *
 * @param {{ questId:string, summary:string, workerId?:string }} opts
 * @returns {Promise<{ ok:true, commentId:string } | { ok:false, error:string }>}
 */
export async function writeWorkerSummaryComment({ questId, summary, workerId = "worker" } = {}) {
  if (!questId || !summary || !String(summary).trim()) {
    const out = { ok: false, error: "questId and a non-empty summary are required" };
    logWeaponCall({ weapon: "questmaster", fn: "writeWorkerSummaryComment", input: { hasQuest: !!questId }, output: out, status: "fail" });
    return out;
  }

  const client = await getServiceClient();
  const { recordQuestComment } = await import("@/libs/quest/index.js");
  const { data: comment, error } = await recordQuestComment(questId, {
    summary: String(summary).trim(),
    actor_name: workerId,
    source: "worker",
    action: "worker_summary",
    resolved: false,
  }, { client });

  if (error || !comment?.id) {
    const out = { ok: false, error: error?.message || "comment insert returned no id" };
    logWeaponCall({ weapon: "questmaster", fn: "writeWorkerSummaryComment", input: { questId, workerId }, output: out, status: "fail" });
    return out;
  }

  const out = { ok: true, commentId: comment.id };
  logWeaponCall({ weapon: "questmaster", fn: "writeWorkerSummaryComment", input: { questId, workerId }, output: out, status: "ok" });
  return out;
}

/**
 * Write/refresh a worker's STATUS HEARTBEAT (SIO #2) — the canonical per-tick
 * "here is where I am now" line. UPSERTS one row per quest (action='worker_status',
 * edited in place each tick, created_at bumped so it stays the newest comment for
 * liveness) instead of appending a fresh status row every burst — so a long-running
 * worker leaves ONE status row, not hundreds (a0ac1be3 had 76, 5214107b 1000+).
 * A real state transition (objective done / stage change / escalation) is NOT a
 * heartbeat — record those via their own append action.
 *
 * @param {{ questId:string, summary:string, workerId?:string, detail?:Record<string,unknown> }} opts
 * @returns {Promise<{ ok:true, commentId:string, heartbeat:boolean } | { ok:false, error:string }>}
 */
export async function writeWorkerStatus({ questId, summary, workerId = "worker", detail = {} } = {}) {
  if (!questId || !summary || !String(summary).trim()) {
    const out = { ok: false, error: "questId and a non-empty summary are required" };
    logWeaponCall({ weapon: "questmaster", fn: "writeWorkerStatus", input: { hasQuest: !!questId }, output: out, status: "fail" });
    return out;
  }

  const client = await getServiceClient();
  const { recordQuestComment } = await import("@/libs/quest/index.js");
  const { data: comment, error, heartbeat } = await recordQuestComment(questId, {
    summary: String(summary).trim(),
    actor_name: workerId,
    source: "worker",
    action: "worker_status",
    detail: detail && typeof detail === "object" && !Array.isArray(detail) ? detail : {},
    resolved: false,
  }, { client });

  if (error || !comment?.id) {
    const out = { ok: false, error: error?.message || "status write returned no id" };
    logWeaponCall({ weapon: "questmaster", fn: "writeWorkerStatus", input: { questId, workerId }, output: out, status: "fail" });
    return out;
  }

  const out = { ok: true, commentId: comment.id, heartbeat: !!heartbeat };
  logWeaponCall({ weapon: "questmaster", fn: "writeWorkerStatus", input: { questId, workerId }, output: { mode: heartbeat ? "upserted" : "inserted" }, status: "ok" });
  return out;
}

/**
 * Render a scoped mutation on ONE objectives row. Only whitelisted fields; one
 * call = one targeted change (comment-first-then-render: the request comment must
 * already exist via `questUpdate`).
 *
 * @param {{ objectiveId:string, patch:Record<string,unknown> }} opts
 * @returns {Promise<{ ok:true, objectiveId:string, fields:string[] } | { ok:false, error:string }>}
 */
export async function patchObjective({ objectiveId, patch } = {}) {
  if (!objectiveId || !patch || typeof patch !== "object" || Array.isArray(patch)) {
    const out = { ok: false, error: "objectiveId and a patch object are required" };
    logWeaponCall({ weapon: "questmaster", fn: "patchObjective", input: { hasObjective: !!objectiveId }, output: out, status: "fail" });
    return out;
  }
  const fields = Object.keys(patch).filter((k) => PATCHABLE_OBJECTIVE_FIELDS.has(k));
  if (fields.length === 0) {
    const out = { ok: false, error: `no patchable fields in patch (allowed: ${[...PATCHABLE_OBJECTIVE_FIELDS].join(", ")})` };
    logWeaponCall({ weapon: "questmaster", fn: "patchObjective", input: { objectiveId }, output: out, status: "fail" });
    return out;
  }
  if ("status" in patch && !["pending", "running", "active", "done", "review"].includes(patch.status)) {
    const out = { ok: false, error: `invalid status '${patch.status}'` };
    logWeaponCall({ weapon: "questmaster", fn: "patchObjective", input: { objectiveId }, output: out, status: "fail" });
    return out;
  }
  const update = {};
  for (const f of fields) update[f] = patch[f];

  const client = await getServiceClient();
  const { error } = await client.from("objectives").update(update).eq("id", objectiveId);
  if (error) {
    const out = { ok: false, error: error.message };
    logWeaponCall({ weapon: "questmaster", fn: "patchObjective", input: { objectiveId, fields }, output: out, status: "fail" });
    return out;
  }
  const out = { ok: true, objectiveId, fields };
  logWeaponCall({ weapon: "questmaster", fn: "patchObjective", input: { objectiveId, fields }, output: out, status: "ok" });
  return out;
}

// ── resolveQuestComment ───────────────────────────────────────────────────────

/**
 * Close an update_request (or other) quest_comment by id. Call ONLY after the
 * requested change has actually landed + been verified.
 *
 * @param {{ commentId:string }} opts
 * @returns {Promise<{ ok:true, commentId:string } | { ok:false, error:string }>}
 */
export async function resolveQuestComment({ commentId } = {}) {
  if (!commentId) {
    const out = { ok: false, error: "commentId required" };
    logWeaponCall({ weapon: "questmaster", fn: "resolveQuestComment", input: {}, output: out, status: "fail" });
    return out;
  }
  const client = await getServiceClient();
  const { error } = await client
    .from(publicTables.questComments)
    .update({ resolved: true })
    .eq("id", commentId);
  if (error) {
    const out = { ok: false, error: error.message };
    logWeaponCall({ weapon: "questmaster", fn: "resolveQuestComment", input: { commentId }, output: out, status: "fail" });
    return out;
  }
  const out = { ok: true, commentId };
  logWeaponCall({ weapon: "questmaster", fn: "resolveQuestComment", input: { commentId }, output: out, status: "ok" });
  return out;
}

// ── addQuestComment ───────────────────────────────────────────────────────────

/**
 * Append one quest_comments row via the guarded `recordQuestComment` chokepoint.
 * The CLI bridge for the per-tick stop-summary that ruleAcceptableStopState
 * requires — without it, workers were resorting to ad-hoc .worker-tmp driver
 * scripts (a scratch-guard violation) and the stop-summary channel went dark
 * for 18+ consecutive ticks on this very quest.
 *
 * @param {{ questId:string, summary:string, action?:string, source?:string, actorName?:string|null, detail?:object, resolved?:boolean }} opts
 * @returns {Promise<{ ok:true, commentId?:string } | { ok:false, error:string }>}
 */
export async function addQuestComment({ questId, summary, action, source, actorName, detail, resolved } = {}) {
  if (!questId || typeof summary !== "string" || !summary.trim()) {
    const out = { ok: false, error: "questId and non-empty summary required" };
    logWeaponCall({ weapon: "questmaster", fn: "addQuestComment", input: { questId }, output: out, status: "fail" });
    return out;
  }
  const { recordQuestComment } = await import("@/libs/quest/index.js");
  const client = await getServiceClient();
  const payload = {
    source: source || "worker",
    action: action || "close_summary",
    summary,
    detail: detail && typeof detail === "object" && !Array.isArray(detail) ? detail : {},
    actor_name: actorName ?? null,
    resolved: typeof resolved === "boolean" ? resolved : true,
  };
  const res = await recordQuestComment(questId, payload, { client });
  if (res?.error) {
    const out = { ok: false, error: res.error.message || String(res.error) };
    logWeaponCall({ weapon: "questmaster", fn: "addQuestComment", input: { questId, action: payload.action }, output: out, status: "fail" });
    return out;
  }
  const commentId = res?.data?.id || null;
  const out = { ok: true, commentId, redirected: !!res?.redirected, upserted: !!res?.upserted, deduped: !!res?.deduped };
  logWeaponCall({ weapon: "questmaster", fn: "addQuestComment", input: { questId, action: payload.action }, output: out, status: "ok" });
  return out;
}

// ── raiseUnblockingQuestion ───────────────────────────────────────────────────

/**
 * CLI bridge that inserts ONE quest_questions row (qtype='unblocking',
 * direction='to_user'). Mirrors addQuestComment: the gap it closes is that
 * workers had no weapon action to raise a blocking question and were forced to
 * hand-roll scripts (a scratch-guard violation) or go dark.
 *
 * After inserting the question, the caller MUST call moveQuestToEscalated to
 * move the quest to escalated stage (the question alone does not do that).
 *
 * @param {{ questId:string, body:string, unblocks_objective_ids?:string[], actorName?:string }} opts
 * @returns {Promise<{ ok:true, questionId:string } | { ok:false, error:string }>}
 */
export async function raiseUnblockingQuestion({ questId, body, unblocks_objective_ids, actorName } = {}) {
  if (!questId || typeof body !== "string" || !body.trim()) {
    const out = { ok: false, error: "questId and non-empty body required" };
    logWeaponCall({ weapon: "questmaster", fn: "raiseUnblockingQuestion", input: { questId }, output: out, status: "fail" });
    return out;
  }
  const client = await getServiceClient();
  const { insertQuestQuestion } = await import("@/libs/council/database/serverQuest.js");
  const { data: row, error } = await insertQuestQuestion(
    {
      questId,
      qtype: "unblocking",
      body: body.trim(),
      answer: null,
      unblocks_objective_ids: Array.isArray(unblocks_objective_ids) ? unblocks_objective_ids : [],
      actor_name: actorName ?? "questmaster:raiseUnblockingQuestion",
    },
    { client },
  );
  if (error) {
    const out = { ok: false, error: error.message || String(error) };
    logWeaponCall({ weapon: "questmaster", fn: "raiseUnblockingQuestion", input: { questId }, output: out, status: "fail" });
    return out;
  }
  const questionId = row?.id || null;
  const out = { ok: true, questionId };
  logWeaponCall({ weapon: "questmaster", fn: "raiseUnblockingQuestion", input: { questId }, output: out, status: "ok" });
  return out;
}

// ── patchQuestQuestion ────────────────────────────────────────────────────────

/**
 * CLI bridge: patch allowed fields on a quest_questions row.
 * Supports body, answer, answered, unblocks_objective_ids.
 *
 * @param {{ questId:string, questionId:string, patch:object }} opts
 * @returns {Promise<{ ok:true, question:object } | { ok:false, error:string }>}
 */
export async function patchQuestQuestion({ questId, questionId, patch } = {}) {
  if (!questId || !questionId || !patch || typeof patch !== "object") {
    const out = { ok: false, error: "questId, questionId, and patch object required" };
    logWeaponCall({ weapon: "questmaster", fn: "patchQuestQuestion", input: { questId, questionId }, output: out, status: "fail" });
    return out;
  }
  const client = await getServiceClient();
  const { updateQuestQuestionById } = await import("@/libs/council/database/serverQuest.js");
  const { data: row, error } = await updateQuestQuestionById({ questId, questionId, patch }, { client });
  if (error) {
    const out = { ok: false, error: error.message || String(error) };
    logWeaponCall({ weapon: "questmaster", fn: "patchQuestQuestion", input: { questId, questionId }, output: out, status: "fail" });
    return out;
  }
  const out = { ok: true, question: row };
  logWeaponCall({ weapon: "questmaster", fn: "patchQuestQuestion", input: { questId, questionId }, output: out, status: "ok" });
  return out;
}

// ── digestAndRemoveQuestion ───────────────────────────────────────────────────

/**
 * The ONLY sanctioned way for a worker to remove an answered unblocking question.
 * Routes through the guarded deleteQuestQuestionById helper (service client),
 * which archives the Q+A+migrationNote into quest_comments BEFORE deleting and
 * REFUSES when migrationNote is empty. Workers must call this — NEVER a raw
 * `client.from('quest_questions').delete()` (the path that caused the 5d29b94f
 * data-loss) and NEVER the auth-gated DELETE /api/quest/questions route.
 *
 * Precondition: migrate the answer's decisions into the quest's durable state
 * (description / objective / deliverable) FIRST, then pass migrationNote naming
 * where each decision landed.
 *
 * @param {{ questId:string, questionId:string, migrationNote:string }} opts
 * @returns {Promise<{ ok:true, deletedId:string, archived:true } | { ok:false, error:string }>}
 */
export async function digestAndRemoveQuestion({ questId, questionId, migrationNote } = {}) {
  if (!questId || !questionId) {
    const out = { ok: false, error: "questId and questionId are required" };
    logWeaponCall({ weapon: "questmaster", fn: "digestAndRemoveQuestion", input: { questId, questionId }, output: out, status: "fail" });
    return out;
  }
  const note = typeof migrationNote === "string" ? migrationNote.trim() : "";
  if (!note) {
    const out = {
      ok: false,
      error:
        "migrationNote is required. Migrate the answer's decisions into the quest (description / objective / deliverable) FIRST, then call this naming where each decision landed. Removing without migrating loses the user's decision.",
    };
    logWeaponCall({ weapon: "questmaster", fn: "digestAndRemoveQuestion", input: { questId, questionId }, output: out, status: "fail" });
    return out;
  }
  const client = await getServiceClient();
  const { deleteQuestQuestionById } = await import("@/libs/council/database/serverQuest.js");
  // userInitiated omitted (=false) → the digest guard applies: archive-then-delete.
  const { data, error } = await deleteQuestQuestionById({ questId, questionId, migrationNote: note }, { client });
  if (error) {
    const out = { ok: false, error: error.message };
    logWeaponCall({ weapon: "questmaster", fn: "digestAndRemoveQuestion", input: { questId, questionId }, output: out, status: "fail" });
    return out;
  }
  if (!data || !data.length) {
    const out = { ok: false, error: "question not found (already removed?)" };
    logWeaponCall({ weapon: "questmaster", fn: "digestAndRemoveQuestion", input: { questId, questionId }, output: out, status: "fail" });
    return out;
  }
  const out = { ok: true, deletedId: questionId, archived: true };
  logWeaponCall({ weapon: "questmaster", fn: "digestAndRemoveQuestion", input: { questId, questionId }, output: out, status: "ok" });
  return out;
}

// ── closeQuest ────────────────────────────────────────────────────────────────

/**
 * Terminal stage transition closing -> complete. Hard-gated:
 *   - quest must be at stage 'closing';
 *   - EVERY action='closing' finale comment must be resolved=true (refuses
 *     otherwise — a pending save-to-skill-book proposal must be answered first).
 * On pass: archives a managerial-summary quest_comment, writes stage='complete',
 * and SELECT-back-verifies the write.
 *
 * @param {{ questId:string, summary?:string }} opts
 * @returns {Promise<{ ok:true, stage:"complete" } | { ok:false, reason:string }>}
 */
export async function closeQuest({ questId, summary } = {}) {
  if (!questId) {
    const out = { ok: false, reason: "questId required" };
    logWeaponCall({ weapon: "questmaster", fn: "closeQuest", input: {}, output: out, status: "fail" });
    return out;
  }
  const client = await getServiceClient();

  const { data: quest, error: qErr } = await client
    .from(publicTables.quests)
    .select("id, title, stage")
    .eq("id", questId)
    .single();
  if (qErr || !quest) {
    const out = { ok: false, reason: qErr?.message || "quest not found" };
    logWeaponCall({ weapon: "questmaster", fn: "closeQuest", input: { questId }, output: out, status: "fail" });
    return out;
  }
  if (quest.stage !== "closing") {
    const out = { ok: false, reason: `quest stage is "${quest.stage}", expected "closing" — only closing->complete is valid here` };
    logWeaponCall({ weapon: "questmaster", fn: "closeQuest", input: { questId, stage: quest.stage }, output: out, status: "fail" });
    return out;
  }

  // Hard gate: no unresolved closing-stage finale comment.
  const { data: openClosing, error: cErr } = await client
    .from(publicTables.questComments)
    .select("id")
    .eq("quest_id", questId)
    .eq("action", "closing")
    .eq("resolved", false);
  if (cErr) {
    const out = { ok: false, reason: `closing-comment check failed: ${cErr.message}` };
    logWeaponCall({ weapon: "questmaster", fn: "closeQuest", input: { questId }, output: out, status: "fail" });
    return out;
  }
  if (openClosing && openClosing.length > 0) {
    const out = { ok: false, reason: `${openClosing.length} unresolved action='closing' finale comment(s) — resolve the save-to-skill-book proposal before closing` };
    logWeaponCall({ weapon: "questmaster", fn: "closeQuest", input: { questId, openClosing: openClosing.length }, output: out, status: "fail" });
    return out;
  }

  // Hard gate: no pending (un-reviewed) quest_lessons. A quest with distilled
  // lessons cannot close until every lesson has a terminal Council decision
  // (approved and merged-to-staging, or rejected). Push alone is not enough.
  const { data: pendingLessons, error: lErr } = await client
    .from(publicTables.questLessons)
    .select("id")
    .eq("quest_id", questId)
    .eq("status", "pending");
  if (lErr) {
    const out = { ok: false, reason: `quest_lessons check failed: ${lErr.message}` };
    logWeaponCall({ weapon: "questmaster", fn: "closeQuest", input: { questId }, output: out, status: "fail" });
    return out;
  }
  if (pendingLessons && pendingLessons.length > 0) {
    const out = { ok: false, reason: `${pendingLessons.length} pending quest_lessons — Council must approve or reject each before the quest can close` };
    logWeaponCall({ weapon: "questmaster", fn: "closeQuest", input: { questId, pendingLessons: pendingLessons.length }, output: out, status: "fail" });
    return out;
  }

  // Archive a managerial summary comment.
  // quest_comments has no `body` column — the text column is `summary` (see the
  // questUpdate insert above and the table schema). Capture + surface the error
  // so a failed archive isn't silently swallowed before the stage transition.
  const { recordQuestComment } = await import("@/libs/quest/index.js");
  const { error: archiveErr } = await recordQuestComment(questId, {
    summary: String(summary || `Quest "${quest.title}" closed: deliverables verified through purrview + final gate; finale proposals resolved.`).trim(),
    actor_name: "guildmaster",
    source: "guildmaster",
    action: "closure_summary",
    resolved: true,
  }, { client });
  if (archiveErr) {
    const out = { ok: false, reason: `managerial-summary archive failed: ${archiveErr.message}` };
    logWeaponCall({ weapon: "questmaster", fn: "closeQuest", input: { questId }, output: out, status: "fail" });
    return out;
  }

  const { error: uErr } = await client
    .from(publicTables.quests)
    .update({ stage: "complete" })
    .eq("id", questId);
  if (uErr) {
    const out = { ok: false, reason: `stage write failed: ${uErr.message}` };
    logWeaponCall({ weapon: "questmaster", fn: "closeQuest", input: { questId }, output: out, status: "fail" });
    return out;
  }

  const { data: verify } = await client
    .from(publicTables.quests)
    .select("stage")
    .eq("id", questId)
    .single();
  if (verify?.stage !== "complete") {
    const out = { ok: false, reason: `stage write did not persist; reads back as "${verify?.stage}"` };
    logWeaponCall({ weapon: "questmaster", fn: "closeQuest", input: { questId }, output: out, status: "fail" });
    return out;
  }

  const out = { ok: true, stage: "complete" };
  logWeaponCall({ weapon: "questmaster", fn: "closeQuest", input: { questId }, output: out, status: "ok" });
  return out;
}

// ── archiveQuestTakeaways (closing-stage archiving engine) ────────────────────

/** Parse the SOURCE task linkage embedded in a quest description (the Asana task
 *  this quest was created from). Prefers an explicit "SOURCE ASANA TASK … gid
 *  <n>", then a bare "Asana <n>", then any "gid <n>". Returns { gid } or null.
 *  The project id in "(…[CJ] backlogs (1205…))" carries no "gid" prefix, so the
 *  gid-anchored patterns pick the TASK, not the project. */
function resolveAsanaSourceTask(description = "") {
  const text = String(description || "");
  const m =
    text.match(/SOURCE ASANA TASK[\s\S]*?gid\s+(\d{8,})/i) ||
    text.match(/\bAsana(?:\s+task)?\s+(\d{8,})/i) ||
    text.match(/\bgid\s+(\d{8,})/i);
  return m ? { gid: m[1] } : null;
}

/**
 * Closing-stage archiving engine — the action the caravan `closing` archive-leaf
 * runs. For a quest at stage='closing': post the deliverables + a completion
 * report to the linked source Asana task (when one is linked) and AUTO-ARCHIVE
 * via closeQuest (closing→complete). SIO #11: routine closes are automatic —
 * escalation is reserved for a GENUINE decision fork ONLY (see closingArchive.js
 * / the ED): a caller-supplied `pendingDecisions` entry, or a FAILED external
 * side-effect (the Asana post threw). No linked target, a missing report, and
 * unresolved action='closing' self-improvement finale comments are ROUTINE —
 * they carry a safe default (complete internally / auto-compose the report /
 * leave the offers open and noted) and never block the close.
 *
 * @param {{ questId:string, report?:string, deliverableRefs?:Array<{name:string,kind?:string,ref?:string}>, pendingDecisions?:string[] }} opts
 * @returns {Promise<{ok:boolean, action?:"closed"|"escalated", archivedToAsana?:boolean, asanaTaskGid?:string|null, questions?:string[], reason?:string}>}
 */
export async function archiveQuestTakeaways({ questId, report = "", deliverableRefs = null, pendingDecisions = [] } = {}) {
  if (!questId) {
    const out = { ok: false, reason: "questId required" };
    logWeaponCall({ weapon: "questmaster", fn: "archiveQuestTakeaways", input: {}, output: out, status: "fail" });
    return out;
  }
  const client = await getServiceClient();

  const { data: quest, error: qErr } = await client
    .from(publicTables.quests)
    .select("id, title, description, stage")
    .eq("id", questId)
    .single();
  if (qErr || !quest) {
    const out = { ok: false, reason: qErr?.message || "quest not found" };
    logWeaponCall({ weapon: "questmaster", fn: "archiveQuestTakeaways", input: { questId }, output: out, status: "fail" });
    return out;
  }
  if (quest.stage !== "closing") {
    const out = { ok: false, reason: `quest stage is "${quest.stage}", expected "closing"` };
    logWeaponCall({ weapon: "questmaster", fn: "archiveQuestTakeaways", input: { questId, stage: quest.stage }, output: out, status: "fail" });
    return out;
  }

  // Deliverables for the archive comment (caller may pass them; else read).
  let dels = Array.isArray(deliverableRefs) ? deliverableRefs : null;
  if (!dels) {
    const { data } = await client
      .from(publicTables.deliverables)
      .select("name, kind, ref")
      .eq("quest_id", questId);
    dels = (data || []).map((d) => ({ name: d.name, kind: d.kind, ref: d.ref }));
  }

  // Unresolved action='closing' finale comments (uncommitted EDs/insights from
  // the closing-stage scanForUncommittedMethods pass). SIO #11: these are
  // OPTIONAL self-improvement offers — they do NOT gate closure. Count them so
  // the closure summary can note they remain open; never escalate on them.
  const { data: openClosing } = await client
    .from(publicTables.questComments)
    .select("id, summary")
    .eq("quest_id", questId)
    .eq("action", "closing")
    .eq("resolved", false);
  const openMethodOfferCount = (openClosing || []).length;

  // ── AUTO-APPROVED archive: post report + deliverables to the source Asana task
  //    WHEN one is linked. No linked target is ROUTINE (an internal quest) — not
  //    a decision. A missing report is auto-composed from the deliverables.
  const source = resolveAsanaSourceTask(quest.description);
  const delBlock = dels.length
    ? dels.map((d) => `• ${d.name}${d.kind ? ` [${d.kind}]` : ""}${d.ref ? `: ${d.ref}` : ""}`).join("\n")
    : "(no deliverable artifacts recorded)";
  const effectiveReport = String(report || "").trim() || "Quest completed — see deliverables below.";
  let archivedToAsana = false;
  let asanaPostFailed = false;
  let asanaFailReason = "";
  if (source?.gid) {
    const text =
      `✅ GuildOS quest complete — "${quest.title}"\n\n` +
      `${effectiveReport}\n\n` +
      `Deliverables:\n${delBlock}\n\n` +
      `(Archived automatically from GuildOS quest ${questId}.)`;
    try {
      // userId fallback (PIGEON_POST_OWNER_ID) lets the Asana token resolve from
      // the owner profile's env_vars when it isn't in the leaf's process.env.
      await asana.writeComment({ task_id: source.gid, text }, process.env.PIGEON_POST_OWNER_ID);
      archivedToAsana = true;
      const { recordQuestComment } = await import("@/libs/quest/index.js");
      await recordQuestComment(questId, {
        source: "guildmaster",
        actor_name: "archivist",
        action: "archived_to_source",
        summary: `Posted completion report + ${dels.length} deliverable link(s) to source Asana task ${source.gid}.`,
        resolved: true,
        detail: { target: "asana", asanaTaskGid: source.gid, deliverables: dels.length },
      }, { client });
    } catch (e) {
      // A genuine external side-effect FAILURE (fork #2) — not "no target".
      asanaPostFailed = true;
      asanaFailReason = String(e?.message || e).slice(0, 160);
    }
  }
  // No archive target at all → nothing to archive; skip silently and close. A
  // missing external target is NOT a user decision — closing is review/evolve,
  // not forced archiving (see engineering_decisions.md 2026-07-03).

  // ── SIO #11 decision fork: escalate ONLY on a genuine, conservatively-defined
  //    decision (caller pendingDecisions, or a FAILED external side-effect).
  //    Everything routine — no target, open self-improvement offers, a missing
  //    report — carries a safe default and auto-archives.
  const { classifyClosingArchive } = await import("./closingArchive.js");
  const { escalate, decisions } = classifyClosingArchive({
    pendingDecisions,
    asanaPostFailed,
    asanaFailReason,
    openMethodOfferCount,
  });

  if (escalate) {
    for (const body of decisions) {
      await client.from(publicTables.questQuestions).insert({
        quest_id: questId,
        qtype: "unblocking",
        direction: "to_user",
        body: `[archiving] ${body}`,
        answered: false,
        actor_name: "archivist",
      });
    }
    await client
      .from(publicTables.quests)
      .update({ stage: "escalated", worker_id: null, worker_claimed_at: null })
      .eq("id", questId)
      .eq("stage", "closing");
    const { recordQuestComment } = await import("@/libs/quest/index.js");
    await recordQuestComment(questId, {
      source: "guildmaster",
      actor_name: "archivist",
      action: "archive_escalated",
      summary: `Archiving needs a user decision (${decisions.length} item(s)) → escalated.${archivedToAsana ? " The auto-approved Asana archive was still posted." : ""}`,
      resolved: false,
      detail: { archivedToAsana, pending: decisions.length },
    }, { client });
    const out = { ok: true, action: "escalated", archivedToAsana, asanaTaskGid: source?.gid || null, questions: decisions };
    logWeaponCall({ weapon: "questmaster", fn: "archiveQuestTakeaways", input: { questId }, output: { action: "escalated", pending: decisions.length }, status: "ok" });
    return out;
  }

  // Routine auto-archive: close internally unless a real archive decision failed above.
  // Open self-improvement offers remain visible but do not block closure.
  const archiveNote = source?.gid
    ? `Takeaways archived to source Asana task ${source.gid}.`
    : "No external archive target - completed internally.";
  const offersNote = openMethodOfferCount > 0
    ? ` ${openMethodOfferCount} self-improvement offer(s) remain open for separate action.`
    : "";
  const closed = await closeQuest({ questId, summary: `${archiveNote}${offersNote} ${String(report || "").trim().slice(0, 240)}`.trim() });
  const out = closed.ok
    ? { ok: true, action: "closed", archivedToAsana, asanaTaskGid: source?.gid || null, openMethodOffers: openMethodOfferCount, stage: closed.stage }
    : { ok: false, reason: `close failed: ${closed.reason}`, archivedToAsana, asanaTaskGid: source?.gid || null };
  logWeaponCall({ weapon: "questmaster", fn: "archiveQuestTakeaways", input: { questId }, output: out, status: out.ok ? "ok" : "fail" });
  return out;
}

// ── auditLeafLogs ─────────────────────────────────────────────────────────────
const WASTE_PATTERNS = [
  { name: "getQuestDetails-miss", re: /getQuestDetails/ },
  { name: "import-probing", re: /node -e.*require\(|node -e.*import\(|npx tsx|node --import|node --loader|alias-loader|_alias-hook|cat >.*_tmp/ },
  { name: "action-discovery", re: /--list/ },
  { name: "skill-book-src-read", re: /libs\/skill_book\/.*\/index\.(js|mjs)/ },
  { name: "gearUp-routing", re: /cli\.mjs.*"/ },
  { name: "find-registry", re: /find.*skill_book|find.*weapon/ },
];

export function auditLeafLogs({ logDir = ".worker-logs", filterPrefix = "leaf-" } = {}) {
  const dir = join(process.cwd(), logDir);
  if (!existsSync(dir)) return { ok: false, reason: `${dir} not found` };

  const files = readdirSync(dir).filter(f => f.startsWith(filterPrefix) && f.endsWith(".log"));
  const results = [];

  for (const file of files) {
    const lines = readFileSync(join(dir, file), "utf8").split("\n");
    const toolCalls = [];

    for (const line of lines) {
      if (!line.trim() || line.startsWith("#")) continue;
      try {
        const entry = JSON.parse(line);
        if (entry.type === "assistant" && entry.message?.content) {
          for (const block of entry.message.content) {
            if (block.type === "tool_use") {
              const cmd = block.input?.command || block.input?.pattern || block.input?.file_path || "";
              let waste = null;
              for (const { name, re } of WASTE_PATTERNS) {
                if (re.test(cmd)) { waste = name; break; }
              }
              toolCalls.push({ tool: block.name, cmd: cmd.slice(0, 120), waste });
            }
          }
        }
      } catch (_) { /* skip non-JSON lines */ }
    }

    const totalWaste = toolCalls.filter(tc => tc.waste).length;
    const wasteBreakdown = {};
    for (const tc of toolCalls) {
      if (tc.waste) wasteBreakdown[tc.waste] = (wasteBreakdown[tc.waste] || 0) + 1;
    }
    const firstRealIdx = toolCalls.findIndex(tc => !tc.waste);

    results.push({
      file,
      totalCalls: toolCalls.length,
      totalWaste,
      wasteRate: toolCalls.length ? `${Math.round(100 * totalWaste / toolCalls.length)}%` : "n/a",
      wasteBeforeFirstReal: firstRealIdx === -1 ? toolCalls.length : firstRealIdx,
      wasteBreakdown,
      first10: toolCalls.slice(0, 10).map((tc, i) => `${i + 1}. [${tc.waste || "WORK"}] ${tc.tool}: ${tc.cmd}`),
    });
  }

  const out = { ok: true, leafCount: results.length, leaves: results };
  logWeaponCall({ weapon: "questmaster", fn: "auditLeafLogs", input: { logDir, filterPrefix }, output: { leafCount: results.length }, status: "ok" });
  return out;
}

// ── checkpointQuestAsset ──────────────────────────────────────────────────────
// assetInlineVsFileStorage (LOCKED 2026-06-12): short text lives INLINE in the
// row's ref — no upload — while large/binary artifacts get a durable file.
const ASSET_INLINE_TEXT_MAX = 1000;

export async function checkpointQuestAsset({
  questId, label, content, filename, kind,
  contentType = "application/octet-stream", source = "worker", actorName = null,
} = {}) {
  if (!questId || !label || content == null || content === "") {
    const out = { ok: false, reason: "questId, label, and content are required" };
    logWeaponCall({ weapon: "questmaster", fn: "checkpointQuestAsset", input: { questId, label }, output: out, status: "fail" });
    return out;
  }

  // INLINE path: text under the threshold lives directly in `ref` — no storage
  // round-trip, no filename needed. Kind defaults to the read-it "text" kind.
  const inline = typeof content === "string" && content.length < ASSET_INLINE_TEXT_MAX;

  try {
    const { insertQuestAsset } = await import("@/libs/council/database/serverQuest.js");
    // Worker checkpoints run outside a Next request (cron / CLI / leaf harness).
    // scripts/_alias-hook.mjs stubs next/headers so resolveServerClient's
    // try/catch fallback never fires — it picks the publishable-key (RLS-bound)
    // SSR client and the row insert fails RLS. Inject the service client so
    // the write reaches the table regardless of how the caller is invoked.
    const serviceClient = await getServiceClient();

    if (inline) {
      const { data: asset, error } = await insertQuestAsset({
        questId, label, ref: content, kind: kind || "text", source, actor_name: actorName,
      }, { client: serviceClient });
      if (error) {
        const out = { ok: false, reason: `asset row insert failed: ${error.message}` };
        logWeaponCall({ weapon: "questmaster", fn: "checkpointQuestAsset", input: { questId, label, inline: true }, output: out, status: "fail" });
        return out;
      }
      const out = { ok: true, inline: true, assetId: asset?.id };
      logWeaponCall({ weapon: "questmaster", fn: "checkpointQuestAsset", input: { questId, label, inline: true }, output: out, status: "ok" });
      return out;
    }

    // FILE path: large text or any binary → upload to GuildOS_Bucket. filename
    // is required only here (the storage path needs it).
    if (!filename) {
      const out = { ok: false, reason: "filename is required for file (non-inline) checkpoints — text under 1000 chars stores inline without one" };
      logWeaponCall({ weapon: "questmaster", fn: "checkpointQuestAsset", input: { questId, label }, output: out, status: "fail" });
      return out;
    }
    const storage = await import("@/libs/weapon/supabase_storage/index.js");
    const storagePath = storage.buildPath({ channel: "quest-assets", questId, filename });
    await storage.writeFile({ path: storagePath, content, contentType });
    const { url: publicUrl } = storage.readPublicUrl({ path: storagePath });

    const { data: asset, error } = await insertQuestAsset({
      questId, label, ref: publicUrl, kind: kind || "document", source, actor_name: actorName,
    }, { client: serviceClient });
    if (error) {
      const out = { ok: false, reason: `asset row insert failed: ${error.message}`, publicUrl };
      logWeaponCall({ weapon: "questmaster", fn: "checkpointQuestAsset", input: { questId, label }, output: out, status: "fail" });
      return out;
    }

    const out = { ok: true, publicUrl, assetId: asset?.id, storagePath };
    logWeaponCall({ weapon: "questmaster", fn: "checkpointQuestAsset", input: { questId, label }, output: out, status: "ok" });
    return out;
  } catch (err) {
    const out = { ok: false, reason: err.message };
    logWeaponCall({ weapon: "questmaster", fn: "checkpointQuestAsset", input: { questId, label }, output: out, status: "fail" });
    return out;
  }
}

export async function shipDeliverableArtifact({
  questId,
  deliverableName,
  kind = "image",
  filename,
  contentBase64,
  content,
  contentType = "application/octet-stream",
  workerId = "worker",
  rationale = "",
  checkpointAsset = true,
} = {}) {
  const name = String(deliverableName || "").trim();
  const safeFilename = String(filename || "").trim();
  if (!questId || !name || !safeFilename) {
    const out = { ok: false, error: "questId, deliverableName, and filename are required" };
    logWeaponCall({ weapon: "questmaster", fn: "shipDeliverableArtifact", input: { questId, hasName: !!name }, output: out, status: "fail" });
    return out;
  }
  if (!contentBase64 && (content == null || content === "")) {
    const out = { ok: false, error: "contentBase64 or content is required" };
    logWeaponCall({ weapon: "questmaster", fn: "shipDeliverableArtifact", input: { questId, deliverableName: name }, output: out, status: "fail" });
    return out;
  }

  try {
    const payload = contentBase64
      ? Buffer.from(String(contentBase64).replace(/^data:[^,]+,/, ""), "base64")
      : String(content);
    const storage = await import("@/libs/weapon/supabase_storage/index.js");
    const storagePath = storage.buildPath({ channel: "quest-assets", questId, filename: safeFilename });
    await storage.writeFile({ path: storagePath, content: payload, contentType, upsert: true });
    const { url: publicUrl } = storage.readPublicUrl({ path: storagePath });

    const { writeDeliverable, recordQuestComment } = await import("@/libs/quest/index.js");
    const { data: deliverable, error: dErr } = await writeDeliverable({
      questId,
      name,
      kind,
      ref: publicUrl,
      verify_screenshot_ref: null,
    });
    if (dErr) {
      const out = { ok: false, error: dErr.message, publicUrl, storagePath };
      logWeaponCall({ weapon: "questmaster", fn: "shipDeliverableArtifact", input: { questId, deliverableName: name }, output: out, status: "fail" });
      return out;
    }

    let assetId = null;
    if (checkpointAsset) {
      const { insertQuestAsset } = await import("@/libs/council/database/serverQuest.js");
      const { data: asset, error: aErr } = await insertQuestAsset({
        questId,
        label: name,
        ref: publicUrl,
        kind,
        source: "worker",
        actor_name: workerId,
      });
      if (aErr) throw new Error(`asset row insert failed: ${aErr.message}`);
      assetId = asset?.id || null;
    }

    let commentId = null;
    const summary = String(rationale || "").trim();
    if (summary) {
      const { data: comment, error: cErr } = await recordQuestComment(questId, {
        source: "worker",
        action: "asset_rationale",
        summary: `${name}: ${summary}`,
        actor_name: workerId,
        detail: { asset_name: name, ref: publicUrl },
      });
      if (cErr) throw new Error(`asset rationale comment failed: ${cErr.message}`);
      commentId = comment?.id || null;
    }

    const out = { ok: true, deliverable, publicUrl, storagePath, assetId, commentId };
    logWeaponCall({ weapon: "questmaster", fn: "shipDeliverableArtifact", input: { questId, deliverableName: name }, output: { publicUrl, assetId, commentId }, status: "ok" });
    return out;
  } catch (err) {
    const out = { ok: false, error: err.message || String(err) };
    logWeaponCall({ weapon: "questmaster", fn: "shipDeliverableArtifact", input: { questId, deliverableName: name }, output: out, status: "fail" });
    return out;
  }
}

/**
 * Atomically increment a quest's complexity tally columns.
 * Uses Postgres `col + N` via the Supabase rpc or raw SQL to ensure atomicity.
 *
 * @param {{ questId:string, toolCalls?:number, tokens?:number }} opts
 * @returns {Promise<{ ok:true, tool_call_count:number, total_tokens:number } | { ok:false, error:string }>}
 */
export async function incrementQuestTally({ questId, toolCalls = 0, tokens = 0 } = {}) {
  if (!questId) {
    const out = { ok: false, error: "questId is required" };
    logWeaponCall({ weapon: "questmaster", fn: "incrementQuestTally", input: { questId }, output: out, status: "fail" });
    return out;
  }
  if (toolCalls === 0 && tokens === 0) {
    const out = { ok: false, error: "at least one of toolCalls or tokens must be > 0" };
    logWeaponCall({ weapon: "questmaster", fn: "incrementQuestTally", input: { questId }, output: out, status: "fail" });
    return out;
  }

  const client = await getServiceClient();

  // Supabase JS doesn't support atomic col+N increments natively.
  // Use rpc to run a raw SQL UPDATE ... SET col = col + $1 RETURNING.
  const { data, error } = await client.rpc("increment_quest_tally", {
    p_quest_id: questId,
    p_tool_calls: toolCalls,
    p_tokens: tokens,
  });

  if (error) {
    const out = { ok: false, error: error.message };
    logWeaponCall({ weapon: "questmaster", fn: "incrementQuestTally", input: { questId, toolCalls, tokens }, output: out, status: "fail" });
    return out;
  }

  const row = Array.isArray(data) ? data[0] : data;
  const out = { ok: true, tool_call_count: row?.tool_call_count ?? 0, total_tokens: row?.total_tokens ?? 0 };
  logWeaponCall({ weapon: "questmaster", fn: "incrementQuestTally", input: { questId, toolCalls, tokens }, output: out, status: "ok" });
  return out;
}

// ── Complexity gate ─────────────────────────────────────────────────────────

/** Minimum thresholds for a quest to justify lesson distillation at closing. */
const COMPLEXITY_GATE = { minToolCalls: 10, minTokens: 5000 };

/**
 * Check whether a quest passes the complexity gate for lesson distillation.
 *
 * @param {{ questId:string }} opts
 * @returns {Promise<{ shouldDistill:boolean, tool_call_count:number, total_tokens:number, reason:string }>}
 */
export async function shouldDistillLesson({ questId } = {}) {
  if (!questId) {
    const out = { shouldDistill: false, tool_call_count: 0, total_tokens: 0, reason: "questId is required" };
    logWeaponCall({ weapon: "questmaster", fn: "shouldDistillLesson", input: {}, output: out, status: "fail" });
    return out;
  }

  const client = await getServiceClient();
  const { data: quest, error } = await client
    .from(publicTables.quests)
    .select("tool_call_count, total_tokens")
    .eq("id", questId)
    .single();

  if (error || !quest) {
    const out = { shouldDistill: false, tool_call_count: 0, total_tokens: 0, reason: error?.message || "quest not found" };
    logWeaponCall({ weapon: "questmaster", fn: "shouldDistillLesson", input: { questId }, output: out, status: "fail" });
    return out;
  }

  const tc = quest.tool_call_count ?? 0;
  const tt = quest.total_tokens ?? 0;
  const passes = tc >= COMPLEXITY_GATE.minToolCalls || tt >= COMPLEXITY_GATE.minTokens;
  const reason = passes
    ? `passes gate: ${tc} tool calls (min ${COMPLEXITY_GATE.minToolCalls}), ${tt} tokens (min ${COMPLEXITY_GATE.minTokens})`
    : `below gate: ${tc} tool calls (min ${COMPLEXITY_GATE.minToolCalls}), ${tt} tokens (min ${COMPLEXITY_GATE.minTokens})`;

  const out = { shouldDistill: passes, tool_call_count: tc, total_tokens: Number(tt), reason };
  logWeaponCall({ weapon: "questmaster", fn: "shouldDistillLesson", input: { questId }, output: out, status: "ok" });
  return out;
}

/**
 * Write a distilled lesson to quest_lessons.
 *
 * @param {{ questId:string, description:string, gitRef?:string, smokeTestResults?:string, smokeTestPass?:boolean }} opts
 * @returns {Promise<{ ok:true, lessonId:string } | { ok:false, error:string }>}
 */
export async function distillQuestLesson({ questId, description, gitRef, smokeTestResults, smokeTestPass } = {}) {
  if (!questId || !description || !String(description).trim()) {
    const out = { ok: false, error: "questId and a non-empty description are required" };
    logWeaponCall({ weapon: "questmaster", fn: "distillQuestLesson", input: { hasQuest: !!questId }, output: out, status: "fail" });
    return out;
  }

  const client = await getServiceClient();
  const row = {
    quest_id: questId,
    description: String(description).trim(),
    ...(gitRef ? { git_ref: gitRef } : {}),
    ...(smokeTestResults ? { smoke_test_results: smokeTestResults } : {}),
    ...(typeof smokeTestPass === "boolean" ? { smoke_test_pass: smokeTestPass } : {}),
  };

  const { data: lesson, error } = await client
    .from(publicTables.questLessons)
    .insert(row)
    .select("id")
    .single();

  if (error || !lesson?.id) {
    const out = { ok: false, error: error?.message || "insert returned no id" };
    logWeaponCall({ weapon: "questmaster", fn: "distillQuestLesson", input: { questId }, output: out, status: "fail" });
    return out;
  }

  const out = { ok: true, lessonId: lesson.id };
  logWeaponCall({ weapon: "questmaster", fn: "distillQuestLesson", input: { questId }, output: out, status: "ok" });
  return out;
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/questmaster/planStaleGuard.js -->
```js
/**
 * @file Plan-stage stale guard (SIO #7).
 *
 * A quest created but never itemized by planQuest sits at `stage='plan'` with 0
 * objectives forever, rotting silently — nothing advances it and the derail
 * ladder considers "plan + no objectives" the CORRECT shape (so the caravan
 * rescuer never touches it). Evidence: 7fd07621 "Migrate and Smoke test skills to
 * Hermes", 0 objectives, stuck in plan.
 *
 * questTick's dispatch-plan step already re-runs planQuest EVERY tick (it deploys
 * a planning leaf to each unleased plan quest), so re-planning happens
 * continuously on its own. This guard is the TERMINAL for when that keeps failing:
 * a plan quest with 0 objectives older than the threshold escalates with an
 * explicit reason, surfacing it to the user instead of letting it rot.
 *
 * Lives in its own module (not inlined in the 3.5k-line questmaster index.js) so
 * it can be unit-tested without dragging in that file's heavy import graph
 * (claudecli / leafBrowser / asana / questPurrview) — the same why-split as
 * loadquests.js / logs.js in the caravan weapon.
 */

import { publicTables } from "@/libs/council/publicTables";

/** Plan-stage quests with 0 objectives older than this (minutes) escalate. The
 *  cron ticks every 5 min and dispatch-plan re-attempts planQuest each tick, so
 *  30 min ≈ 6 planning attempts before we declare planning stuck. Env-tunable so
 *  a slower/faster box can retune without a code edit. */
export const PLAN_STALE_MINUTES = Math.max(1, Number(process.env.GUILDOS_PLAN_STALE_MINUTES) || 30);

/**
 * Escalate plan-stage quests that were never itemized (SIO #7). For each
 * plan-stage, UNLEASED quest owned by `assignee` that is older than
 * `thresholdMinutes` and has ZERO objectives (and is not already resting on an
 * open unblocking question), flip stage→escalated (CAS), mint an unblocking
 * question, and write a queryable `escalation_reason` comment (same tag as the
 * caravan escalation sites — SIO #4 — so `WHERE action='escalation_reason'`
 * returns a uniform reason across both subsystems).
 *
 * MUST run BEFORE questTick's dispatch-plan step: an escalated quest leaves the
 * plan stage, so dispatch-plan won't re-lease it — whereas if this ran AFTER
 * dispatch-plan, that step would re-lease the stale quest every tick and this
 * guard's unleased filter would never see it (it would perpetually skip).
 *
 * Skips: leased quests (a planning leaf is mid-flight — don't interrupt it) and
 * quests with an open unblocking question (already surfaced to the user).
 *
 * @param {object} opts
 * @param {object} opts.client              Supabase service client
 * @param {string} [opts.assignee]          owner filter (questTick's assignee)
 * @param {number} [opts.thresholdMinutes]  age gate; default PLAN_STALE_MINUTES
 * @param {number} [opts.nowMs]             clock injection point for tests
 * @param {boolean} [opts.dispatch=true]    false = observe (report, no writes)
 * @returns {Promise<{escalated:Array<{questId:string,title:string,ageMin:number,wouldEscalate?:boolean}>, error?:string}>}
 */
export async function escalateStalePlanQuests({
  client,
  assignee = "Main_Claude",
  thresholdMinutes = PLAN_STALE_MINUTES,
  nowMs = Date.now(),
  dispatch = true,
} = {}) {
  const escalated = [];
  const { data: planQuests } = await client
    .from(publicTables.quests)
    .select("id, title, stage, created_at, worker_id")
    .eq("assignee", assignee)
    .eq("stage", "plan")
    .is("worker_id", null)
    .order("created_at", { ascending: true });
  if (!planQuests?.length) return { escalated };

  const cutoffMs = nowMs - thresholdMinutes * 60_000;
  for (const q of planQuests) {
    const createdMs = q.created_at ? new Date(q.created_at).getTime() : 0;
    // Too young (or unknown age) — give planning more ticks before declaring stuck.
    if (!createdMs || createdMs > cutoffMs) continue;

    // Itemized already? A single objective row means planning succeeded → healthy.
    const { data: objs } = await client
      .from(publicTables.objectives)
      .select("id")
      .eq("quest_id", q.id)
      .limit(1);
    if (objs && objs.length) continue;

    // Already resting on an open unblocking question → surfaced to the user
    // already (planning correctly raised a blocker); escalating again is redundant.
    const { data: openQ } = await client
      .from(publicTables.questQuestions)
      .select("id")
      .eq("quest_id", q.id)
      .eq("qtype", "unblocking")
      .eq("answered", false)
      .limit(1);
    if (openQ && openQ.length) continue;

    const ageMin = Math.round((nowMs - createdMs) / 60_000);
    if (!dispatch) {
      escalated.push({ questId: q.id, title: q.title, ageMin, wouldEscalate: true });
      continue;
    }

    // CAS escalate (→escalated is ungated by the enforce_stage_gate DB trigger).
    const { data: moved } = await client
      .from(publicTables.quests)
      .update({ stage: "escalated" })
      .eq("id", q.id)
      .eq("stage", "plan")
      .is("worker_id", null)
      .select("id");
    if ((moved?.length || 0) !== 1) continue; // lost the race (leased/advanced); leave it

    // Mint the unblocking question so gate 1 keeps it resting at escalated (an
    // escalated quest with no open question would immediately re-derail).
    await client.from(publicTables.questQuestions).insert({
      quest_id: q.id,
      qtype: "unblocking",
      direction: "to_user",
      body:
        `1. WHAT YOU NEED TO DO: Add enough detail for this quest to be broken into objectives — or confirm it should be dropped.\n\n` +
        `2. WHY I CAN'T DO THIS: This quest has sat at the plan stage for ${ageMin} min with ZERO objectives — automatic planning ran every ` +
        `tick but never produced a workable breakdown (usually the description is too vague or empty to itemize). Rather than let it rot ` +
        `silently, auto-rescue escalated it for your input.`,
      answered: false,
      actor_name: "rescuer",
    });

    // Queryable escalation reason — SAME action tag as the caravan escalation
    // sites (SIO #4), so a reviewer can answer "why did this escalate?" for a
    // plan-stale quest with the identical one-shot query.
    const { recordQuestComment } = await import("@/libs/quest/index.js");
    await recordQuestComment(
      q.id,
      {
        source: "system",
        action: "escalation_reason",
        summary: `Escalated from plan (no objectives): automatic planning produced 0 objectives after ${ageMin} min — description likely too vague to itemize.`,
        detail: {
          reason: "plan-stage quest never itemized (0 objectives past threshold)",
          ageMin,
          thresholdMinutes,
          from: "plan",
          escalation: "plan_stale",
          allObjectivesDone: false,
          objectives: { total: 0, done: 0, active: 0 },
        },
      },
      { client },
    );

    escalated.push({ questId: q.id, title: q.title, ageMin });
    console.log(`[questTick] plan-stale guard — escalated "${q.title}" (${q.id}) after ${ageMin} min at plan with 0 objectives`);
  }

  if (escalated.length) console.log(`[questTick] plan-stale guard — escalated ${escalated.length} quest(s):`, JSON.stringify(escalated));
  return { escalated };
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/questmaster/planStaleGuard.test.js -->
```js
import { describe, it, expect, vi, beforeEach } from "vitest";

// ── mocks ────────────────────────────────────────────────────────────────────

vi.mock("@/libs/council/publicTables", () => ({
  publicTables: {
    quests: "quests",
    questComments: "quest_comments",
    objectives: "objectives",
    questQuestions: "quest_questions",
  },
}));

// The escalation_reason comment routes through the quest_comments chokepoint;
// mock it to a single insert on the injected client (isolate from libs/quest deps).
vi.mock("@/libs/quest/index.js", () => ({
  recordQuestComment: vi.fn(async (questId, payload, { client } = {}) => {
    await client.from("quest_comments").insert({ quest_id: questId, ...payload });
    return { data: { id: "c-mock" }, error: null };
  }),
}));

import { escalateStalePlanQuests } from "./planStaleGuard.js";

// ── helpers ──────────────────────────────────────────────────────────────────

function buildChain(resolveData = [], resolveError = null) {
  const calls = { updates: [], inserts: [] };
  const chain = {
    select: () => chain,
    not: () => chain,
    is: () => chain,
    or: () => chain,
    eq: () => chain,
    in: () => chain,
    order: () => chain,
    limit: () => chain,
    like: () => chain,
    neq: () => chain,
    insert(row) { calls.inserts.push(row); return chain; },
    update(row) { calls.updates.push(row); return chain; },
    then(resolve) { return resolve({ data: resolveData, error: resolveError }); },
  };
  return { chain, calls };
}

const NOW = Date.parse("2026-07-03T12:00:00Z");
const OLD = "2026-07-03T11:00:00Z";   // 60 min old (> default 30 min threshold)
const YOUNG = "2026-07-03T11:55:00Z"; // 5 min old (< threshold)

// ── tests ────────────────────────────────────────────────────────────────────

describe("escalateStalePlanQuests (SIO #7)", () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it("escalates a plan quest with 0 objectives older than the threshold, minting a question + escalation_reason", async () => {
    // .from() sequence: 1 plan list · 2 objectives probe · 3 openQ probe ·
    // 4 CAS escalate · 5 mint question · 6 escalation_reason comment.
    const planList = buildChain([{ id: "q1", title: "Migrate skills to Hermes", stage: "plan", created_at: OLD, worker_id: null }]);
    const objChain = buildChain([]); // 0 objectives
    const openQChain = buildChain([]); // no open unblocking question
    const movedChain = buildChain([{ id: "q1" }]); // CAS escalate wins
    const mintQChain = buildChain();
    const reasonChain = buildChain();

    const db = { from: vi.fn() };
    let n = 0;
    db.from.mockImplementation(() => {
      n++;
      if (n === 1) return planList.chain;
      if (n === 2) return objChain.chain;
      if (n === 3) return openQChain.chain;
      if (n === 4) return movedChain.chain;
      if (n === 5) return mintQChain.chain;
      return reasonChain.chain;
    });

    const result = await escalateStalePlanQuests({ client: db, nowMs: NOW });

    expect(result.escalated).toHaveLength(1);
    expect(result.escalated[0]).toMatchObject({ questId: "q1", ageMin: 60 });
    // Flipped stage → escalated, minted an unblocking question, wrote reason tag.
    expect(movedChain.calls.updates[0].stage).toBe("escalated");
    expect(mintQChain.calls.inserts[0]).toMatchObject({ qtype: "unblocking", answered: false, direction: "to_user" });
    const reason = reasonChain.calls.inserts[0];
    expect(reason.action).toBe("escalation_reason");
    expect(reason.detail).toMatchObject({ escalation: "plan_stale", from: "plan", allObjectivesDone: false });
    expect(reason.detail.objectives).toMatchObject({ total: 0, done: 0, active: 0 });
  });

  it("skips a young plan quest (planning still deserves more ticks)", async () => {
    const planList = buildChain([{ id: "q2", title: "Fresh", stage: "plan", created_at: YOUNG, worker_id: null }]);
    const db = { from: vi.fn(() => planList.chain) };

    const result = await escalateStalePlanQuests({ client: db, nowMs: NOW });

    expect(result.escalated).toHaveLength(0);
    expect(planList.calls.updates).toHaveLength(0); // never escalated
  });

  it("skips a plan quest that already has objectives", async () => {
    const planList = buildChain([{ id: "q3", title: "Itemized", stage: "plan", created_at: OLD, worker_id: null }]);
    const objChain = buildChain([{ id: "o1" }]); // has an objective → healthy
    const db = { from: vi.fn() };
    let n = 0;
    db.from.mockImplementation(() => { n++; return n === 1 ? planList.chain : objChain.chain; });

    const result = await escalateStalePlanQuests({ client: db, nowMs: NOW });
    expect(result.escalated).toHaveLength(0);
  });

  it("skips a plan quest already resting on an open unblocking question", async () => {
    const planList = buildChain([{ id: "q4", title: "Blocked", stage: "plan", created_at: OLD, worker_id: null }]);
    const objChain = buildChain([]);
    const openQChain = buildChain([{ id: "oq1" }]); // already surfaced to user
    const db = { from: vi.fn() };
    let n = 0;
    db.from.mockImplementation(() => {
      n++;
      if (n === 1) return planList.chain;
      if (n === 2) return objChain.chain;
      return openQChain.chain;
    });

    const result = await escalateStalePlanQuests({ client: db, nowMs: NOW });
    expect(result.escalated).toHaveLength(0);
  });

  it("observe-mode (dispatch:false) reports wouldEscalate without writing", async () => {
    const planList = buildChain([{ id: "q5", title: "Stale", stage: "plan", created_at: OLD, worker_id: null }]);
    const objChain = buildChain([]);
    const openQChain = buildChain([]);
    const db = { from: vi.fn() };
    let n = 0;
    db.from.mockImplementation(() => {
      n++;
      if (n === 1) return planList.chain;
      if (n === 2) return objChain.chain;
      return openQChain.chain;
    });

    const result = await escalateStalePlanQuests({ client: db, nowMs: NOW, dispatch: false });
    expect(result.escalated).toHaveLength(1);
    expect(result.escalated[0]).toMatchObject({ questId: "q5", wouldEscalate: true });
    // No CAS update, no inserts anywhere.
    expect(planList.calls.updates).toHaveLength(0);
    expect(objChain.calls.inserts).toHaveLength(0);
    expect(openQChain.calls.inserts).toHaveLength(0);
  });
});
```
<!-- /GUILDOS-FILE -->
