# questmaster

<!-- GUILDOS-SKILL name="questmaster" -->

<!-- GUILDOS-DESC -->
Plan user requests into quest-shaped JSON and pick roster adventurers by capability. Owns the quest lifecycle procedures a Questmaster runs: the pre-creation clarify gate, the plan-stage itemizing pass (description → objectives + deliverables + questions), worker stop-state / comment / question digestion rules, the SMART clarification cascade, the review → closing → complete approval arc, and assignee matching. Use when creating, planning, refining, reviewing, or closing a quest, or when authoring objectives, unblocking questions, or checkpoint assets.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/questmaster/SKILL.md -->
```markdown
---
name: questmaster
description: >-
  Plan user requests into quest-shaped JSON and pick roster adventurers by
  capability. Owns the quest lifecycle procedures a Questmaster runs: the
  pre-creation clarify gate, the plan-stage itemizing pass (description →
  objectives + deliverables + questions), worker stop-state / comment /
  question digestion rules, the SMART clarification cascade, the review →
  closing → complete approval arc, and assignee matching. Use when creating,
  planning, refining, reviewing, or closing a quest, or when authoring
  objectives, unblocking questions, or checkpoint assets.
---

# Questmaster

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/questmaster/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto questmaster <action>` (or read the source directly).

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
- **raiseUnblockingQuestionDiplomatically** — When you need to raise an unblocking question to the user — which bumps the quest to stage `escalated` — use THIS action to author it right. Every such question MUST: (1) state exactly WHAT the user needs to do/answer; (2) state WHY this cannot be done by the agent; and (3) first PASS sage.askSage before it is written into the quest. Most candidate questions are manufactured stops (deferred design defaults / self-answerable facts) and must be self-resolved, never raised. LOCKED 2026-06-15.
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
- **runReviewApproval** — Handle a user review-stage approval ('looks good, approve it') on a quest at stage='review'. Run the final gate (confirmApproval → real final read → questReview.pass), and on FINAL_GATE_PASS auto-advance review→closing (the deploy objective already landed), then trigger scanForUncommittedMethods.
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
\`verify_screenshot_ref\` MUST ALWAYS be a **public Supabase Storage URL** (https://*.supabase.co/storage/*) — no exceptions, not even for the Guildmaster. Upload to \`GuildOS_Bucket\` (path convention: \`quest-assets/<questId>/verify/<deliverableId>.png\`) via the \`supabase_storage\` weapon, then set \`verify_screenshot_ref\` to the public URL. A local file path, \`file://\` URL, or any non-Supabase URL is NEVER valid for \`verify_screenshot_ref\`. The golden path is \`questPurrview.autoVisualVerify\` (headless, always uploads); manual CIC screenshots must be uploaded the same way before setting the field.

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
      description:
        "When you need to raise an unblocking question to the user — which bumps the quest to stage `escalated` — use THIS action to author it right. Every such question MUST: (1) state exactly WHAT the user needs to do/answer; (2) state WHY this cannot be done by the agent; and (3) first PASS sage.askSage before it is written into the quest. Most candidate questions are manufactured stops (deferred design defaults / self-answerable facts) and must be self-resolved, never raised. LOCKED 2026-06-15.",
      howTo: `
Raising an unblocking question stalls the quest until a human acts and bumps it to \`escalated\` — EXPENSIVE. "Diplomatically" = respect the user's time: raise a question ONLY when it genuinely needs them, and when you do, make it answerable in one read. Follow these steps IN ORDER before any \`quest_questions\` row is written.

**STEP 1 — PASS sage.askSage FIRST. The question is written ONLY if it survives this gate.**
Run the candidate through the sage question-gate (the same gate the chat uses — see the \`sage\` skill book \`askSage\`): would a competent agent, given this task and NO answer, be UNABLE to proceed correctly?
  - **PROCEED** (reversible / a sensible default exists / discoverable / you can defensibly decide) → DO NOT raise a question. Decide it, record the decision in the objective's \`notes\` (or a comment), keep working. This is the COMMON case.
  - **ESCALATE** (only) when the block is genuinely IRREVERSIBLE / business-damaging if guessed wrong (real spend, going-live/publishing, legal/brand/IP, an external commitment, a hard-to-undo change) OR needs USER-ONLY KNOWLEDGE (a preference / private fact / intent not derivable from the repo, tools, memory, or a defensible default).
Only an ESCALATE verdict earns a written question. KILL the candidate (resolve it yourself, do not ask) if ANY hold:
  - DESIGN DEFAULT — "which approach / structure / corpus / format / first target?" with a reasonable default. Pick it; note it.
  - SELF-ANSWERABLE FACT — "is X available? what's the shape of Y? does Z exist?" Go look (grep/read/gearUp); never ask what you can settle by inspection.
  - RATIFICATION — you already proposed the answer inside the question. If you can propose it, you can decide it.
  - DELEGATED — the brief hands you the call ("your discretion", a stated range).
If you cannot name a crisp, NON-circular reason the USER specifically is required, there is no block — there is nothing to raise.

**STEP 2 — compose the body with the TWO MANDATORY components (only after Step 1 returns ESCALATE).**
The \`body\` MUST contain, explicitly and labelled:
  1. **WHAT YOU NEED TO DO:** the concrete decision/action/answer, actionable COLD — the exact options with the agent's RECOMMENDED default, or the exact command/setting/file the user must change. No vague "how should we proceed?" menu.
  2. **WHY I CAN'T DO THIS:** one sentence naming the SPECIFIC reason the agent cannot resolve it — the irreversibility, or the user-only knowledge. If you cannot write this crisply, the question failed Step 1 — go back and resolve it yourself.
LAYOUT (mandatory for readability): write the two components as two NUMBERED parts separated by a blank line (a double newline) — "1. WHAT YOU NEED TO DO: ..." then an empty line then "2. WHY I CAN'T DO THIS: ...". A body missing either component, or with the two parts run together without the blank-line gap, is malformed and must not be written.

**STEP 3 — write it + link the objective.** Create the \`unblocking\` \`quest_questions\` row (\`unblocks_objective_ids\` = the blocked objective id(s)) — **leave the objective OPEN** (do not mark it done) — and let \`moveQuestToEscalated\` bump the quest to \`escalated\`. The open unblocking question **plus** the \`escalated\` stage ARE the block — there is **no separate objective "blocked" status** (the \`attention\` status was retired 2026-06-15; a single source of truth). A blocked objective is simply an open objective that has a linked open unblocking question.

This is sage.askSage applied to the QUEST substrate, not just the chat. Used wherever unblocking questions originate: \`planQuest\` (planning) and worker bursts (\`workScopePerTick\` #1, \`ruleAcceptableStopState\` #2).
`,
    },
    ruleAcceptableStopState: {
      description:
        "The criteria a worker burst MUST reach before it may stop. Every objective either done, or open-and-blocked-by-a-linked-unblocking-question; every block externalized as an unblocking question; exactly one stop-summary comment (edit the prior failed-stop comment, don't pile up). LOCKED 2026-06-12; objective `attention` status retired 2026-06-15.",
      howTo: `
A worker burst may STOP only when the quest is in an **acceptable stop state**. All three criteria must hold:

1. **Every objective is either \`done\` or a genuinely-blocked OPEN objective** — none left \`pending\`/\`active\` and unworked. You either finished it, or it is blocked (left OPEN, NOT marked done) and carries a linked unblocking question. There is no \`attention\` status — a blocked objective is just an open objective with a linked open unblocking question.
2. **Every BLOCKED objective is mapped to a \`quest_questions\` row of type \`unblocking\` RAISED VIA \`raiseUnblockingQuestionDiplomatically\`** — a blocked objective MUST have a corresponding unblocking question (put its id in the question's \`unblocks_objective_ids\`), and that question MUST have been authored through \`raiseUnblockingQuestionDiplomatically\` (passed sage.askSage as a real agent-can't-do-this block, and its body states WHAT-you-need-to-do + WHY-I-can't). An OPEN objective whose only "block" fails that gate is NOT a real block — resolve it yourself and finish the objective. An open, unfinished objective with NO conforming unblocking question is NOT an acceptable stop (it is workable — keep going).
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
1. Pick the lowest-n workable objective (open — \`pending\`/\`active\` — with no open unblocking question) and work it to \`done\`. If you think it is blocked, do NOT just create a question — run \`raiseUnblockingQuestionDiplomatically\`: its Step 1 (sage.askSage) decides whether this is a real block or a reversible default/self-answerable fact you must resolve yourself (record in \`notes\` and finish the objective). ONLY when that gate returns ESCALATE do you author the \`unblocking\` question per its Step 2 (WHAT-you-need-to-do + WHY-I-can't), **leave the objective OPEN**, and link its id (\`unblocks_objective_ids\`) — the open question, not a status, marks it blocked.
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
        "Handle a user review-stage approval ('looks good, approve it') on a quest at stage='review'. Run the final gate (confirmApproval → real final read → questReview.pass), and on FINAL_GATE_PASS auto-advance review→closing (the deploy objective already landed), then trigger scanForUncommittedMethods.",
      howTo: `The user approved a quest that is at \`stage='review'\` (it got there via \`questPurrview.approve\`). This is the FINAL human gate. You orchestrate the \`questReview\` weapon — never write \`quests.stage\` directly.

## Arc

1. **confirmApproval** — call \`questReview.confirmApproval({questId})\` to verify the quest legitimately reached \`review\` via \`questPurrview.approve\` (the APPROVE lockphrase is present). No lockphrase → it did not pass purrview; do not final-gate it.
2. **Real final read** — genuinely re-verify the deployed/assembled result one last time (CIC / multimodal read of the actual artifact), not a rubber-stamp. The user said "approve" but the gate still earns it.
3. **questReview.pass** — call \`questReview.pass(...)\`; it writes the \`FINAL_GATE_PASS\` lockphrase (\`"this quest has cleared final verification"\`). **Under the new 5-stage model, with the deploy/last objective already landed, this owner auto-advances \`stage='review' → 'closing'\`.** (The legacy weapon kept stage at \`review\` for a separate GM-desk step; that desk step is gone in the 5-stage model — the auto-advance to \`closing\` on FINAL_GATE_PASS is the scoped weapon adaptation. Do NOT emulate the advance with a direct \`quests.stage\` write — the lockphrase + gated transition must be weapon-written.)
4. **Trigger \`questmaster.scanForUncommittedMethods\`** at \`closing\` (see that action) so any in-execution method developed during the quest is offered for skill-book capture.
5. Report via \`worker.reportMutation\`: confirm the quest moved to \`closing\` and that the save-to-skill-book proposal (if any) is waiting as a quest comment.

## Rules

- Lockphrase + stage transition are weapon-written; never a direct \`quests.stage\` write.
- The final read must be real even though the user already said approve — the gate's job is independent verification.
- Only fire \`scanForUncommittedMethods\` AFTER the quest is at \`closing\` (it writes a closing-stage comment).
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
- \`localhost:3002\` unreachable → agent forgot to start dev server
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

3. **Is the verification of this objective clear?** Each entry in \`objective.output[]\` should carry a \`verification\` field. The objective is "clear" only if every output entry has a concrete check (CIC screenshot vs. expectation, file lint, API round-trip, schema validation, etc.) **that the USER stated**. "Looks right" or silence is NOT clear. **If the objective's \`method\` carries a user-flagged \`[IMPORTANT DETAIL]\`, the verification of that non-negotiable MUST be user-stated — an agent-supplied "obvious" check does NOT make it clear; treat it as unresolved and ask.** Consistent with the \`evaluate\` firewall and the \`summarizeRelevantContext\` provenance rule: on an \`[IMPORTANT DETAIL]\`-bearing objective, a \`verification\` is resolved ONLY if it carries the literal token \`[USER-STATED]\` (set by summarizeRelevantContext from an explicit user statement). A non-empty \`verification\` WITHOUT \`[USER-STATED]\` on such an objective is an agent-synthesized check — treat it as UNRESOLVED exactly as if it were blank, and ask the user how the flagged non-negotiable is verified. If unclear → ask which verification gate applies to which output asset.

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

<!-- GUILDOS-FILE: libs/weapon/asana/index.js -->
```js
/**
 * Asana weapon — unified Asana REST API connector.
 *
 * Auth: ASANA_ACCESS_TOKEN from profiles.env_vars or process.env.
 */

export const toc = {
  searchWorkspaces: "Search visible Asana workspaces for the authenticated user.",
  searchProjects: "Search Asana projects by name or workspace.",
  readProject: "Read a single Asana project by id.",
  writeProject: "Update an Asana project's name and/or notes (description).",
  searchTasks: "Search Asana tasks across projects with filters.",
  readTask: "Read full details of one Asana task.",
  writeTask: "Create or update an Asana task (upsert).",
  deleteTask: "Delete an Asana task.",
  readSections: "Read sections of an Asana project.",
  readComments: "Read stories/comments on an Asana task.",
  writeComment: "Post a comment on an Asana task.",
  editComment: "Edit the text of an existing comment story (only the author can edit).",
  deleteComment: "Delete a comment story by its GID (only the author can delete).",
};
import { database } from "@/libs/council/database";

const API_BASE = "https://app.asana.com/api/1.0";

// ---------------------------------------------------------------------------
// Auth
// ---------------------------------------------------------------------------

async function getToken(userId) {
  if (process.env.ASANA_ACCESS_TOKEN) return process.env.ASANA_ACCESS_TOKEN;
  if (!userId) return null;
  const db = await database.init("service");
  const { data } = await db
    .from("profiles")
    .select("env_vars")
    .eq("id", userId)
    .single();
  return data?.env_vars?.ASANA_ACCESS_TOKEN ?? null;
}

async function asanaFetch(path, opts = {}, userId) {
  const token = await getToken(userId);
  if (!token) throw new Error("Missing ASANA_ACCESS_TOKEN");
  const url = path.startsWith("http") ? path : `${API_BASE}${path}`;
  const res = await fetch(url, {
    ...opts,
    headers: {
      Authorization: `Bearer ${token}`,
      "Content-Type": "application/json",
      ...opts.headers,
    },
  });
  if (!res.ok) {
    const body = await res.text();
    throw new Error(`Asana API ${res.status}: ${body.slice(0, 500)}`);
  }
  const json = await res.json();
  return json.data !== undefined ? json : json;
}

// ---------------------------------------------------------------------------
// Workspaces
// ---------------------------------------------------------------------------

/**
 * List all workspaces the token has access to.
 * @param {object} [input]
 * @param {string} [userId]
 */
export async function searchWorkspaces(input = {}, userId) {
  const res = await asanaFetch("/workspaces?limit=100", {}, userId);
  return { workspaces: res.data || [] };
}

// ---------------------------------------------------------------------------
// Projects
// ---------------------------------------------------------------------------

/**
 * Search projects by name or list all in a workspace.
 * @param {{ workspace_id?: string, query?: string, limit?: number }} input
 * @param {string} [userId]
 */
export async function searchProjects({ workspace_id, query, limit = 100 } = {}, userId) {
  if (workspace_id) {
    const res = await asanaFetch(
      `/projects?workspace=${workspace_id}&opt_fields=name,archived,color&limit=${limit}`,
      {},
      userId,
    );
    let projects = res.data || [];
    if (query) {
      const q = query.toLowerCase();
      projects = projects.filter((p) => p.name.toLowerCase().includes(q));
    }
    return { projects };
  }
  // Search all workspaces
  const { workspaces } = await searchWorkspaces({}, userId);
  const all = [];
  for (const ws of workspaces) {
    const res = await asanaFetch(
      `/projects?workspace=${ws.gid}&opt_fields=name,archived,color&limit=${limit}`,
      {},
      userId,
    );
    let projects = res.data || [];
    if (query) {
      const q = query.toLowerCase();
      projects = projects.filter((p) => p.name.toLowerCase().includes(q));
    }
    all.push(...projects);
  }
  return { projects: all };
}

/**
 * Read a single project by GID.
 * @param {{ project_id: string }} input
 * @param {string} [userId]
 */
export async function readProject({ project_id } = {}, userId) {
  if (!project_id) throw new Error("project_id is required");
  const res = await asanaFetch(
    `/projects/${project_id}?opt_fields=name,notes,archived,color,created_at,modified_at,owner.name,members.name`,
    {},
    userId,
  );
  return res.data || res;
}

/**
 * Update an Asana project (name and/or notes/description).
 * @param {{ project_id: string, name?: string, notes?: string }} input
 * @param {string} [userId]
 */
export async function writeProject({ project_id, name, notes } = {}, userId) {
  if (!project_id) throw new Error("project_id is required");
  const body = {};
  if (name !== undefined) body.name = name;
  if (notes !== undefined) body.notes = notes;
  if (!Object.keys(body).length) throw new Error("Nothing to update — provide name and/or notes");
  const res = await asanaFetch(
    `/projects/${project_id}`,
    { method: "PUT", body: JSON.stringify({ data: body }) },
    userId,
  );
  return res.data || res;
}

// ---------------------------------------------------------------------------
// Tasks
// ---------------------------------------------------------------------------

/**
 * Search tasks in a project or by assignee.
 * @param {{ project_id?: string, assignee?: string, workspace_id?: string, completed?: boolean, limit?: number }} input
 * @param {string} [userId]
 */
export async function searchTasks({ project_id, assignee, workspace_id, completed, limit = 100 } = {}, userId) {
  const optFields = "name,notes,completed,assignee.name,memberships.section.name,tags.name,due_on,created_at,modified_at";

  if (project_id) {
    const params = new URLSearchParams({ opt_fields: optFields, limit: String(limit) });
    if (completed === false) params.set("completed_since", "now");
    const allTasks = [];
    let offset = null;
    do {
      if (offset) params.set("offset", offset);
      const res = await asanaFetch(`/projects/${project_id}/tasks?${params}`, {}, userId);
      if (res.data) allTasks.push(...res.data);
      offset = res.next_page?.offset || null;
    } while (offset && allTasks.length < limit);
    return { tasks: allTasks.slice(0, limit) };
  }

  if (assignee && workspace_id) {
    const params = new URLSearchParams({
      assignee,
      workspace: workspace_id,
      opt_fields: optFields,
      limit: String(limit),
    });
    if (completed === false) params.set("completed_since", "now");
    const res = await asanaFetch(`/tasks?${params}`, {}, userId);
    return { tasks: res.data || [] };
  }

  throw new Error("Either project_id or both assignee + workspace_id are required.");
}

/**
 * Read a single task by GID.
 * @param {{ task_id: string }} input
 * @param {string} [userId]
 */
export async function readTask({ task_id } = {}, userId) {
  if (!task_id) throw new Error("task_id is required");
  const res = await asanaFetch(
    `/tasks/${task_id}?opt_fields=name,notes,completed,assignee.name,memberships.section.name,tags.name,due_on,created_at,modified_at,parent.name,custom_fields`,
    {},
    userId,
  );
  return res.data || res;
}

/**
 * Create or update a task.
 * @param {{ task_id?: string, project_id?: string, name?: string, notes?: string, assignee?: string, due_on?: string, completed?: boolean }} input
 * @param {string} [userId]
 */
export async function writeTask({ task_id, project_id, name, notes, assignee, due_on, completed } = {}, userId) {
  const body = {};
  if (name != null) body.name = name;
  if (notes != null) body.notes = notes;
  if (assignee != null) body.assignee = assignee;
  if (due_on != null) body.due_on = due_on;
  if (completed != null) body.completed = completed;

  if (task_id) {
    // Update existing task
    const res = await asanaFetch(
      `/tasks/${task_id}`,
      { method: "PUT", body: JSON.stringify({ data: body }) },
      userId,
    );
    return res.data || res;
  }

  // Create new task
  if (!project_id && !body.projects) {
    throw new Error("project_id is required when creating a new task");
  }
  if (project_id) body.projects = [project_id];
  const res = await asanaFetch(
    "/tasks",
    { method: "POST", body: JSON.stringify({ data: body }) },
    userId,
  );
  return res.data || res;
}

/**
 * Delete a task.
 * @param {{ task_id: string }} input
 * @param {string} [userId]
 */
export async function deleteTask({ task_id } = {}, userId) {
  if (!task_id) throw new Error("task_id is required");
  await asanaFetch(`/tasks/${task_id}`, { method: "DELETE" }, userId);
  return { ok: true, deleted: task_id };
}

// ---------------------------------------------------------------------------
// Sections
// ---------------------------------------------------------------------------

/**
 * Read sections in a project.
 * @param {{ project_id: string }} input
 * @param {string} [userId]
 */
export async function readSections({ project_id } = {}, userId) {
  if (!project_id) throw new Error("project_id is required");
  const res = await asanaFetch(`/projects/${project_id}/sections?opt_fields=name`, {}, userId);
  return { sections: res.data || [] };
}

// ---------------------------------------------------------------------------
// Comments / Stories
// ---------------------------------------------------------------------------

/**
 * Read comments on a task.
 * @param {{ task_id: string, limit?: number }} input
 * @param {string} [userId]
 */
export async function readComments({ task_id, limit = 20 } = {}, userId) {
  if (!task_id) throw new Error("task_id is required");
  const res = await asanaFetch(
    `/tasks/${task_id}/stories?opt_fields=created_by.name,text,created_at,resource_subtype&limit=${limit}`,
    {},
    userId,
  );
  const comments = (res.data || [])
    .filter((s) => s.resource_subtype === "comment_added")
    .map((s) => ({
      author: s.created_by?.name || "unknown",
      text: (s.text || "").slice(0, 1000),
      created_at: s.created_at,
    }));
  return { comments };
}

/**
 * Write a comment on a task.
 * @param {{ task_id: string, text: string }} input
 * @param {string} [userId]
 */
export async function writeComment({ task_id, text } = {}, userId) {
  if (!task_id) throw new Error("task_id is required");
  if (!text) throw new Error("text is required");
  const res = await asanaFetch(
    `/tasks/${task_id}/stories`,
    { method: "POST", body: JSON.stringify({ data: { text } }) },
    userId,
  );
  return res.data || res;
}

/**
 * Edit the text of an existing comment story. Only the original author can edit.
 * @param {{ story_id: string, text: string }} input
 * @param {string} [userId]
 */
export async function editComment({ story_id, text } = {}, userId) {
  if (!story_id) throw new Error("story_id is required");
  if (text == null) throw new Error("text is required");
  const res = await asanaFetch(
    `/stories/${story_id}`,
    { method: "PUT", body: JSON.stringify({ data: { text } }) },
    userId,
  );
  return res.data || res;
}

/**
 * Delete a comment story by its GID. Only the original author can delete.
 * @param {{ story_id: string }} input
 * @param {string} [userId]
 */
export async function deleteComment({ story_id } = {}, userId) {
  if (!story_id) throw new Error("story_id is required");
  await asanaFetch(`/stories/${story_id}`, { method: "DELETE" }, userId);
  return { ok: true, deleted: story_id };
}

// ---------------------------------------------------------------------------
// Credential check
// ---------------------------------------------------------------------------

export async function checkCredentials(userId) {
  try {
    const token = await getToken(userId);
    if (token) {
      return { ok: true, msg: "ASANA_ACCESS_TOKEN is set" };
    }
    return {
      ok: false,
      msg: "Missing ASANA_ACCESS_TOKEN — add it to profile env_vars or process.env.",
    };
  } catch (e) {
    return { ok: false, msg: `Error: ${e.message}` };
  }
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/claudecli/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/weapon/claudecli`

Rationale layer (per CLAUDE.md §6). The WHY behind the leaf-invocation subprocess.
Append; supersede in place when a decision changes.

---

## 2026-06-13 — the full dispatch prompt is persisted into the worker log, delimited

**Decision.** In the streaming path (`logPath` set), `invoke()` writes the full
`taskPrompt` into the log immediately after the start marker, wrapped in
`# ===PROMPT_BEGIN===` / `# ===PROMPT_END===`. The quest-detail worker-log viewer
(`formatWorkerLog`) renders that block whole, untruncated.

**Rejected alternative.** Rely on `claude --print --output-format stream-json` to
echo the input prompt back (it does NOT), or just remove the UI's clip.

**Why.** stream-json does not echo the input prompt, and the prompt temp file is
deleted after the run — so without an explicit write there is **no record of what
the leaf was actually told**, which the operator needs to audit dispatches. A
delimited block is parser-safe (the JSON-per-line `formatWorkerLog` skips
non-JSON lines, so an undelimited multi-line prompt would be silently dropped).

**Consequence if violated.** Worker logs lose the prompt entirely (the symptom:
the operator could only see a 600-char clip of it, sourced inconsistently from
stream-json events that don't reliably contain it).

---

## 2026-06-13 — leaf invocations run `node --no-warnings` (don't flip package.json type)

**Decision.** Leaf-prompt invocations of `run-action.mjs` use `node --no-warnings`
to suppress `MODULE_TYPELESS_PACKAGE_JSON`.

**Rejected alternative.** Add `"type":"module"` to `package.json`.

**Why.** This is a mixed CommonJS/ESM repo (Next app); flipping `package.json` to
`module` risks breaking `require`-based `.js` files. Suppressing the cosmetic
warning at the call site is the low-risk fix.

**Consequence if violated.** Either warning noise floods every worker
transcript/tool-result, or a global `type` flip breaks CJS modules.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/claudecli/index.js -->
```js
/**
 * claudeCLI weapon — invokes `claude --print --dangerously-skip-permissions` and returns
 * stdout. Uses exec() with a temp file for the prompt to avoid shell escaping issues
 * and environment quirks in the Next.js dev server.
 */

export const toc = {
  invoke: "Invoke Claude Code CLI as a subprocess with a prompt; returns stdout / raw output.",
};
import { exec, spawn } from "child_process";
import { writeFileSync, unlinkSync, mkdirSync, createWriteStream } from "fs";
import { join, dirname } from "path";
import { tmpdir } from "os";
import { randomUUID } from "crypto";

const TIMEOUT_MS = 30 * 60 * 1000; // 30 minutes — real Tier-2/quest-creation GM cascades exceed 5m

/**
 * Invoke `claude --print`. Two modes:
 *  - Default (no opts): buffered exec, returns final stdout. Used by GM cascades.
 *  - Streaming (opts.logPath set): spawn + tee stdout/stderr LIVE into a log file
 *    so a UI can tail a leaf worker's transcript. opts.streamJson adds
 *    `--verbose --output-format stream-json` for a per-event (tool-call) log.
 * @param {string} taskPrompt
 * @param {{ logPath?: string, streamJson?: boolean, mcpConfigPath?: string }} [opts]
 *   mcpConfigPath: path to an --mcp-config JSON (e.g. leafBrowser.prepareLeafBrowser's
 *   output) injecting extra MCP servers for THIS invocation only — used to hand a
 *   leaf an authenticated Playwright browser. Omitted ⇒ leaf inherits only the
 *   config-declared servers, exactly as before.
 * @returns {Promise<{ ok: boolean, html: string, rawOutput: string, error?: string }>}
 */
export async function invoke(taskPrompt, opts = {}) {
  // Support object-style input from run-action.mjs (fn(input) calling convention):
  //   claudeCLI invoke '{"taskPrompt":"...","mcpConfigPath":"..."}'
  if (taskPrompt && typeof taskPrompt === "object" && !Array.isArray(taskPrompt)) {
    opts = { ...opts, ...taskPrompt };
    taskPrompt = taskPrompt.taskPrompt;
  }
  if (!taskPrompt || typeof taskPrompt !== "string" || !taskPrompt.trim()) {
    return { ok: false, html: "", rawOutput: "", error: "taskPrompt is required" };
  }
  const { logPath = null, streamJson = false, mcpConfigPath = null } = opts || {};

  // Write prompt to a temp file so we don't have to shell-escape it
  const tmpFile = join(tmpdir(), `claudecli-${randomUUID()}.txt`);
  try {
    writeFileSync(tmpFile, taskPrompt.trim(), "utf8");
  } catch (e) {
    return { ok: false, html: buildErrorHtml(`Failed to write temp file: ${e.message}`, ""), rawOutput: "", error: e.message };
  }

  // Use `type` (Windows) or `cat` (Unix) to pipe the prompt file into claude --print
  const cat = process.platform === "win32" ? "type" : "cat";
  const fmt = streamJson ? " --verbose --output-format stream-json" : "";
  // Per-invocation MCP injection (e.g. an authenticated Playwright browser for a
  // browser-tagged leaf). --dangerously-skip-permissions already auto-allows the
  // injected tools, so no --allowedTools allowlist is needed (and adding one would
  // wrongly RESTRICT the leaf to only those tools).
  const mcp = mcpConfigPath ? ` --mcp-config "${mcpConfigPath}"` : "";
  const cmd = `${cat} "${tmpFile}" | claude --print --dangerously-skip-permissions${fmt}${mcp}`;

  // Strip ANTHROPIC_API_KEY so claude CLI uses OAuth login (Max subscription) instead of API credits
  const env = { ...process.env };
  delete env.ANTHROPIC_API_KEY;

  // ── Streaming path — tee a leaf worker's live transcript into logPath ──
  if (logPath) {
    try { mkdirSync(dirname(logPath), { recursive: true }); } catch { /* ignore */ }
    let ws;
    try { ws = createWriteStream(logPath, { flags: "w" }); } catch { ws = null; }
    if (ws) ws.write(`# worker log — started ${new Date().toISOString()}\n`);
    // Persist the FULL dispatch prompt at the top of the log. `claude --print
    // --output-format stream-json` does NOT echo the input prompt back, and the
    // prompt temp file is deleted after the run — so without this the operator
    // has no record of what the leaf was actually told. Delimited so the log
    // viewer (formatWorkerLog) can render it whole, untruncated.
    if (ws) ws.write(`# ===PROMPT_BEGIN===\n${taskPrompt.trim()}\n# ===PROMPT_END===\n`);
    return new Promise((resolve) => {
      let out = "";
      const child = spawn(cmd, { cwd: process.cwd(), env, shell: true, windowsHide: true });
      // Decode as UTF-8 at the stream level (not per-chunk) so multibyte chars
      // (em-dash, →, β, emoji) split across a pipe-buffer boundary aren't mangled
      // into U+FFFD �. Node's StringDecoder buffers partial sequences across chunks.
      child.stdout?.setEncoding("utf8");
      child.stderr?.setEncoding("utf8");
      const killer = setTimeout(() => { try { child.kill(); } catch { /* ignore */ } }, TIMEOUT_MS);
      child.stdout?.on("data", (d) => { const s = d.toString(); out += s; if (ws) ws.write(s); });
      child.stderr?.on("data", (d) => { const s = d.toString(); if (ws) ws.write(s); });
      child.on("error", (e) => {
        clearTimeout(killer);
        if (ws) { try { ws.end(`\n# spawn error: ${e.message}\n`); } catch { /* ignore */ } }
        try { unlinkSync(tmpFile); } catch { /* ignore */ }
        resolve({ ok: false, html: "", rawOutput: out, error: e.message });
      });
      child.on("close", (code) => {
        clearTimeout(killer);
        if (ws) { try { ws.end(`\n# worker log — exited code=${code} at ${new Date().toISOString()}\n`); } catch { /* ignore */ } }
        try { unlinkSync(tmpFile); } catch { /* ignore */ }
        resolve({ ok: code === 0, html: out.trim(), rawOutput: out });
      });
    });
  }

  // ── Default path — buffered exec with retry on transient failure ──
  // A non-zero `claude --print` exit under concurrency (many simultaneous leaf
  // judges) shows up as "Command failed" and was bouncing quests on a flake.
  // Retry transient (non-timeout) failures with linear backoff; the tmpFile is
  // kept until all attempts finish. A real 30-min timeout is NOT retried.
  const MAX_ATTEMPTS = 3;
  const runOnce = () =>
    new Promise((res) => {
      exec(cmd, { cwd: process.cwd(), timeout: TIMEOUT_MS, maxBuffer: 10 * 1024 * 1024, env }, (err, stdout, stderr) => {
        res({ err, stdout: stdout || "", stderr: stderr || "" });
      });
    });

  let last;
  for (let attempt = 1; attempt <= MAX_ATTEMPTS; attempt++) {
    last = await runOnce();
    if (process.env.NODE_ENV === "development" && last.stderr) {
      process.stderr.write(`[claudecli:stderr] ${last.stderr}`);
    }
    if (!last.err) {
      try { unlinkSync(tmpFile); } catch { /* ignore */ }
      return { ok: true, html: last.stdout.trim(), rawOutput: last.stdout };
    }
    const isTimeout = last.err.killed || last.err.code === "ETIMEDOUT";
    if (isTimeout) break; // a genuine 30-min timeout is not transient — don't retry
    if (attempt < MAX_ATTEMPTS) {
      console.warn(`[claudecli] attempt ${attempt}/${MAX_ATTEMPTS} failed (${last.err.message?.split("\n")[0]}); retrying after backoff`);
      await new Promise((r) => setTimeout(r, 1500 * attempt));
    }
  }
  try { unlinkSync(tmpFile); } catch { /* ignore */ }
  const isTimeout = last.err.killed || last.err.code === "ETIMEDOUT";
  const msg = isTimeout
    ? "Claude CLI timed out after 30 minutes."
    : `Claude CLI error after ${MAX_ATTEMPTS} attempts: ${last.err.message}`;
  return { ok: false, html: buildErrorHtml(msg, last.stdout), rawOutput: last.stdout, error: isTimeout ? "timeout" : last.err.message };
}

/** @param {string} msg @param {string} log */
function buildErrorHtml(msg, log) {
  const escaped = (s) => s.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;");
  return `<!DOCTYPE html><html><head><meta charset="utf-8"><title>claudeCLI Error</title>
<style>body{font-family:monospace;padding:2rem;background:#1a1a1a;color:#f87171}
pre{white-space:pre-wrap;word-break:break-all;color:#d1d5db;background:#111;padding:1rem;border-radius:6px}</style>
</head><body><h2>claudeCLI Error</h2><p>${escaped(msg)}</p>${log ? `<pre>${escaped(log)}</pre>` : ""}</body></html>`;
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/leafBrowser/cli.mjs -->
```js
#!/usr/bin/env node
/**
 * leafBrowser CLI — operator helpers for the leaf authenticated-browser profile.
 *
 *   node libs/weapon/leafBrowser/cli.mjs capture   # launch headed Chrome on the master profile to log in
 *   node libs/weapon/leafBrowser/cli.mjs status    # show master/chrome detection
 *
 * `capture` is the ONE manual step: Chrome opens on the master profile, you log
 * into the sites leaves must reach, then close it. Leaves get copies thereafter.
 */
import { spawn } from "child_process";
import { mkdirSync } from "fs";
import { captureMasterProfileCmd, hasMasterProfile, hasMasterCookie, captureMasterFromChrome, openInteractiveLogin, exportStorageState, findChrome } from "./index.js";

const cmd = process.argv[2] || "status";

// Headed, persistent interactive login (for sites that reject copied cookies,
// e.g. Gmail). Auto-seeds the account from the master profile's storageState so
// the page recognizes the operator (Gmail shows their email) BEFORE they log in.
// Run DETACHED — it stays open until the operator closes the window.
//   node libs/weapon/leafBrowser/cli.mjs login [url]
if (cmd === "login") {
  const url = process.argv[3] || "https://mail.google.com/";
  let seedStatePath = null;
  if (hasMasterProfile()) {
    const exp = await exportStorageState({ fromProfile: "master", outPath: ".browser-profiles/auth-seed.json" });
    if (exp.ok) { seedStatePath = exp.path; console.log(`Seeded account from master (${exp.cookieCount} cookies) → ${exp.path}`); }
    else console.log(`Seed export skipped: ${exp.reason}`);
  }
  const r = await openInteractiveLogin({ url, seedStatePath, resetProfile: true });
  console.log(JSON.stringify(r));
  process.exit(r.ok ? 0 : 1);
}

if (cmd === "status") {
  console.log(JSON.stringify({
    hasMasterProfile: hasMasterProfile(),
    hasMasterCookie: hasMasterCookie(),
    chrome: findChrome(),
    capture: captureMasterProfileCmd(),
  }, null, 2));
  process.exit(0);
}

// Non-interactive capture from the operator's LIVE Chrome profile (zero login
// step). Grabs the Cookies DB only when Chrome isn't holding it under its
// exclusive lock — close Chrome first if `hasMasterCookie` stays false.
if (cmd === "capture-from-chrome") {
  const r = captureMasterFromChrome();
  console.log(JSON.stringify(r, null, 2));
  console.log(r.ok ? "ARMED (session cookie captured)." : "NOT armed — Cookies still locked. Close Chrome once and re-run, or it self-arms on the next questTick after Chrome closes.");
  process.exit(r.ok ? 0 : 1);
}

if (cmd === "capture") {
  const info = captureMasterProfileCmd();
  if (!info.chrome) {
    console.error("Chrome not found — install Chrome or set its path. Detected: null");
    process.exit(1);
  }
  mkdirSync(info.masterDir, { recursive: true });
  console.log(`Launching Chrome on master profile:\n  ${info.masterDir}\nLog into the sites leaves need, then CLOSE Chrome to finish.`);
  const child = spawn(info.chrome, [`--user-data-dir=${info.masterDir}`], { stdio: "ignore", detached: false });
  child.on("close", (code) => {
    console.log(`Chrome closed (code=${code}). Master profile ready: ${hasMasterProfile()}`);
    process.exit(0);
  });
  child.on("error", (e) => { console.error("Failed to launch Chrome:", e.message); process.exit(1); });
} else {
  console.error(`Unknown command "${cmd}". Use: capture | status`);
  process.exit(1);
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/leafBrowser/engineering_decisions.md -->
```markdown
# leafBrowser — Engineering Decisions

## 2026-06-15 · leafBrowser is the SINGLE browser substrate; in-process screenshots route through it

**Decision.** leafBrowser owns ALL Playwright/Chromium launching in the repo, via
two surfaces over one auth model:
- `prepareLeafBrowser` — MCP injection for headless `claude --print` LEAVES
  (interactive page-driving by an LLM).
- `launchContext` / `screenshotUrl` — in-process render-to-PNG (no LLM), the
  primitive `questPurrview.autoVisualVerify`, `cicVerify`, and `poster_render`
  now delegate to. `launchContext` is the only `chromium.launch*` call site.

**Rejected alternative.** Leaving each weapon with its own
`await import("playwright"); chromium.launch({headless:true})` (the state before
this date: questPurrview.renderDeliverableScreenshot, cicVerify.screenshotFromUrl,
poster_render.renderPoster each launched independently), and letting the
"planned substrate" (leafBrowser) stay MCP-only.

**Why.** leafBrowser was built with ONLY the MCP-leaf surface, gated on a
captured master profile — so it was dormant (no profile → never called) while the
job that actually recurs is the *unauthenticated, in-process* "URL → verify PNG".
With no in-process surface on leafBrowser, three weapons grew parallel chromium
launchers and, under pressure, leaves hand-rolled `node -e require('playwright')`
and fell back to **html2canvas** (the operator's locked not-real-verification
path). Four+ launchers + a synthetic-canvas escape hatch = exactly the substrate
sprawl a single owner prevents (drifting auth policy, chromium-resolution, and
fail-closed semantics; the browsercontrol skill book even still claimed Chrome was
launched in only one place). Centralizing also makes the auth-profile model
reachable from the in-process path (`authed:true` → persistent context over a
profile copy), so the captured-profile work finally pays off for screenshots, not
just MCP leaves.

**Consequence if violated.** Re-introducing a local `chromium.launch` /
`puppeteer` / `html2canvas` in any weapon forks the substrate again: verify
screenshots stop sharing the fail-closed-on-blank guard and the ref→URL
resolution (the bug that made `proving-grounds/x/index.html` refs un-navigable and
silently empty), and synthetic-canvas "verifications" creep back in. New screenshot
needs MUST call `leafBrowser.screenshotUrl` (or `launchContext` if they drive the
page themselves).

**Scope note.** `CICPolice` (drives a *headed* browser to police CIC) and
`scripts/auth-capture.mjs` (manual storageState export for cloud Cursor agents)
are deliberately NOT folded in — different jobs, not the verify-screenshot
substrate. They are the only remaining direct Playwright launches and are headed /
operator-manual, not headless screenshot producers.

**Proven.** 2026-06-15 — `screenshotUrl({ ref:'proving-grounds/about-us-rewrite/index.html' })`
returned a 304 KB full-page PNG (real render, visually confirmed), including the
dev-server→`file://` fallback for static public refs.

## 2026-06-16 · Master capture = copy the operator's LIVE Chrome profile, non-interactively + self-arming

**Decision.** `captureMasterFromChrome` populates the master by copying the
operator's EXISTING logged-in Chrome profile (auth-only allowlist: `Local State` +
`Network/Cookies` + `Local Storage/leveldb` + `Preferences`) rather than the
interactive headed re-login (`captureMasterProfileCmd` / `cli.mjs capture`).
Dispatch gates on `hasMasterCookie()` (the session Cookies DB) — NOT
`hasMasterProfile()` — and `questTick` calls `ensureMasterFromChrome()` so the
pipeline self-arms with zero operator action.

**Rejected alternatives.** (1) The interactive capture (operator opens headed
Chrome, logs into every site, closes) — rejected because the operator is ALREADY
logged in in their main Chrome and explicitly wanted no manual step. (2) Gating on
`hasMasterProfile()` — rejected: after a partial copy, `Local State` exists so it
returns true while the profile has NO cookie, so a `[BROWSER]` leaf would dispatch
"authenticated" and silently bounce to `/signin`. (3) Force-closing / taskkilling
the operator's Chrome to free the cookie lock — rejected (memory
feedback_browser_control: never taskkill the operator's Chrome).

**Why / the load-bearing constraint.** Chrome's Network service holds
`Default/Network/Cookies` under an EXCLUSIVE OS lock while running — verified three
ways 2026-06-16: `CopyFile` → EBUSY, libuv read-handle → EBUSY, .NET
`FileShare.ReadWrite` → sharing violation; this box is non-admin (no VSS/`esentutl`)
and Chrome 149 refuses `--remote-debugging-port` on the default profile dir (no CDP
fallback). So the session cookie CANNOT be extracted while Chrome is live. The
copy is therefore (a) lock-tolerant (`lockTolerantCopy`: libuv read-handle bypasses
CopyFile's share-mode failure for files Chrome doesn't hold exclusively), (b)
NON-destructive (no `rmSync` wipe — a re-run that can't grab the live-locked cookie
must preserve a previously captured one), and (c) opportunistic (re-attempts only
the cookie each tick; succeeds the moment Chrome is next closed). Everything except
the cookie copies fine while Chrome runs; the cookie self-captures on the next
natural Chrome close — zero operator action.

**Consequence if violated.** Re-adding `rmSync(MASTER_DIR)` to the capture wipes a
good cookie whenever the next opportunistic run hits a live lock (the pipeline
would arm then disarm itself every cron tick). Gating dispatch back on
`hasMasterProfile()` dispatches `[BROWSER]` leaves against a cookieless profile
that just screenshots the sign-in wall (the exact deadlock on quest 988c9792).

**PROVEN end-to-end 2026-06-16.** Operator closed Chrome → `captureMasterFromChrome`
grabbed the now-unlocked `Network/Cookies` → a headless Playwright leaf rendered the
**authenticated** GuildOS `/caravan` (no `/signin` redirect; full dashboard —
`public/proving-grounds/caravan-authed.png`). So ABE cookie decryption works in a
copied profile launched by Playwright with system Chrome on the same user+machine,
and the cron-leaf auth-screenshot deadlock (quest 988c9792) is cleared.

**Google-account caveat (Gmail will NOT work via copied cookies).** Same run, target
`mail.google.com`: Google recognized the account from the copied session ("Hi Sijie,
<REDACTED-EMAIL>") but stopped at "Verifying it's you — complete sign-in using your
passkey" (`public/proving-grounds/gmail-inbox-delivered.png`), even headed + with
anti-automation flags (`--disable-blink-features=AutomationControlled`, no
`--enable-automation`, `navigator.webdriver` spoofed). High-value Google accounts
require a passkey/2FA to authorize a session on a *new browser instance*
(device-bound-session-style hardening), which a profile/cookie copy cannot satisfy
(the passkey is hardware/TPM-bound, non-exportable) and which an agent must not
complete. **Consequence:** do not waste cycles trying to make Gmail (or other
passkey-gated Google surfaces) work through `leafBrowser` **cookie-COPY** reuse — it
is a Google-side wall, not a pipeline defect. The pipeline IS correct for normal
cookie/Supabase-session sites (proven on `/caravan`).

## 2026-06-16 · HAZARD — copying a rotating-refresh-token session can LOG OUT the source browser

**Observed.** After a leaf rendered the authenticated `/caravan` via the cookie-COPY
master (real, screenshotted), within ~the hour BOTH the copy AND the operator's MAIN
Chrome were logged out (`/caravan` → `/signin`). The operator noticed their own
localhost:3002 was signed out.

**Why.** GuildOS auth is Supabase, which **rotates the refresh token** on use. The
copied cookie carries the SAME refresh token as the source. When the leaf's copy
exercised the session and a refresh fired, Supabase rotated the token and invalidated
the prior one — which the operator's main browser still held → the operator got
signed out as collateral. (Plain ~1h access-token expiry can produce the same symptom;
rotation triggered by the copy's use is the likely accelerant.)

**Consequence / guidance.** Cookie-copy auth is **best-effort and can disturb the
source login** for any site with rotating sessions (Supabase, many OAuth sessions).
Do NOT present a cookie-copy render as a durable/shared logged-in state — it is a
point-in-time snapshot that can desync. For durable automation auth, prefer the
**dedicated-interactive-profile** model (log in ONCE in `.browser-profiles/interactive`
and have leaves reuse that profile DIRECTLY) — the session lives only there, so
refreshing/rotating it never touches the operator's main browser. Reserve cookie-copy
for read-mostly, short-lived, non-rotating targets.

## 2026-06-18 · PROVEN — Gmail works headlessly via native interactive login + same-profile reuse

The Gmail path below is now VERIFIED end-to-end, and it resolves the earlier
"Gmail is walled" caveat (which applies ONLY to cookie-COPY):
- **Clean login avoids the passkey wall.** A FRESH interactive login (empty profile,
  no seeded/copied cookies) advances `<REDACTED-EMAIL>` straight to the PASSWORD
  step — NOT the "complete sign-in using your passkey" challenge. The passkey wall
  only fires when Google sees a copied/seeded session it must re-verify on a "new
  device." So do NOT seed the profile when the goal is a real login; seed only to
  show the account chooser.
- **Same-profile headless reuse renders the inbox.** After the operator logged in
  (password) in the headed window on `.browser-profiles/interactive` and closed it
  gracefully, a HEADLESS Playwright launch on that SAME profile dir loaded
  `mail.google.com/mail/u/0/#inbox` with the live inbox (Inbox 368, Compose, labels)
  — `public/proving-grounds/gmail-inbox-delivered.png`. No copy → no device-binding
  re-challenge. This is the durable Gmail-in-a-leaf path; quest 6b5133b8 = complete.

## 2026-06-18 · Leaf-MCP injection: `cmd /c npx` on Windows + `directProfileDir` (PROVEN)

**Decision.** `prepareLeafBrowser`'s mcp-config launches the Playwright MCP server as
`{ command:"cmd", args:["/c","npx",...] }` on win32 (not `{ command:"npx", args:[...] }`),
and accepts `directProfileDir` (use a profile IN PLACE, no copy) + `outputDir`.

**Why — the bug this fixes.** An MCP host (`claude --print --mcp-config`) spawns the
server command DIRECTLY, without a shell. On Windows `npx` is `npx.cmd`; a bare
`"npx"` is not found, the server never connects, and `claude --print` proceeds with
**no `mcp__playwright__*` tools** — silently (no fatal error). First leaf run hit
exactly this: the leaf reported "no Playwright MCP tools connected." Wrapping via
`cmd /c npx` resolves it. **Rejected alternative:** `command:"npx"` (worked on
macOS/Linux, silently broke every Windows leaf — and GuildOS runs on Windows).

**`directProfileDir`.** The default per-leaf COPY of master is wrong for Google
(copy → passkey re-challenge) and for rotating sessions (logs the operator out).
`directProfileDir` points `--user-data-dir` at the operator's native-login profile
(`.browser-profiles/interactive`) in place — single-flight, never copied, never
cleaned up by `cleanupLeafBrowser` (which only removes `LEAF_ROOT/<workerId>`).

**Consequence if violated.** Reverting to `command:"npx"` silently disables browser
for EVERY Windows leaf (the deadlock returns, with no error to diagnose). Copying the
interactive profile instead of using it directly re-triggers the Google passkey wall.

**PROVEN 2026-06-18.** A real `claude --print` leaf with this config navigated to
Gmail and screenshotted the operator's authenticated inbox (<REDACTED-EMAIL>, real
senders) → `public/proving-grounds/leaf-gmail-inbox.png`. This is the end-to-end
leaf-MCP proof the earlier "not yet proven" note (browsercontrol) was waiting on.

## 2026-06-16 · HAZARD — never FORCE-KILL an interactive-login Chrome window

Force-killing the Playwright login window (`Stop-Process -Force` / `taskkill /F`)
skips Chrome's graceful shutdown, which is what flushes the just-completed session
to the profile on disk → the operator's login is LOST (observed: a completed
`<REDACTED-EMAIL>` login vanished after a force-kill, profile showed "Signed
out" next launch). Close via `context.close()` (graceful) or let the operator close
the window. Only force-kill when you are about to WIPE that profile anyway.

Also: Google's account chooser offers no inline "Remove an account" while one of the
listed accounts holds an active session — so to drop a specific account (e.g. a
passkey-walled personal account, keeping a password-login work account) the reliable
move is to WIPE the dedicated profile (`.browser-profiles/interactive`; `master` is
separate and untouched) and have the operator log into only the wanted account. A
passkey-gated Google account cannot be automated headlessly at all; a password-login
(non-passkey) account can.

**The Gmail path (added 2026-06-16): native interactive login, reused directly.**
`openInteractiveLogin()` opens a HEADED persistent Playwright browser on a DEDICATED
profile (`.browser-profiles/interactive`) where the operator logs in BY HAND. Because
the session is MINTED in that browser+profile (not copied in), the device binding is
native → Google does not re-challenge, and it persists for days. **Load-bearing
rule: reuse that profile dir DIRECTLY — NEVER copy it** (a copy re-triggers the exact
passkey wall above; that is the whole reason the cookie-copy master fails for Gmail).
So Google/Gmail leaf targets must launch on `.browser-profiles/interactive` itself,
single-flight (Chrome locks a user-data-dir), NOT via the per-leaf copy path used for
the cookie-copy master. Do not "unify" interactive into the copy flow — it breaks it.
(The Gmail MCP remains a fine alternative when only email *content* is needed, no
browser screenshot.)

## 2026-06-16 · Cloudflare-protected targets (bosterbio): HEADED + ONE fresh context per page

**Discovery (measured this date, verifying the EK1695 / Biotium escalated quests).**
For `www.bosterbio.com` (Cloudflare "Just a moment…" JS challenge):
- `screenshotUrl`/`launchContext` with `headless:true` is **403'd every time** — both
  `isolated` AND `authed` (the bundled-Chromium headless fingerprint is the tell;
  even the operator's real profile + `cf_clearance` cookie does not save it).
- `headless:false` (headed, system Chrome via `executablePath`) **clears the
  challenge** — but only for the **FIRST navigation in a given context**. Cloudflare
  adaptively re-challenges the 2nd+ nav in the same session (observed: page A 200,
  page B/C 403 in one context). So the reliable pattern is **one fresh
  `launchContext` per page** (first-hit-per-context passes), with a poll-until-cleared
  wait (title ≠ "just a moment" AND bodyLen>~400, up to ~25–30 s) before screenshot.

**Rejected alternative.** Batching N bosterbio pages through one headless (or one
shared headed) context — produces a pile of "Just a moment…" challenge screenshots
that *look* like real captures (fail-closed catches blanks, but a 265-char challenge
page is not blank), i.e. silent false verification.

**Consequence if violated.** A verify/QA run that reuses a context or stays headless
will catalog Cloudflare interstitials as "verified" deliverables. Multi-step flows on
bosterbio (add-to-cart → cart) inherently need ≥2 navs → the 2nd gets challenged, so
**cart-submit cannot be auto-driven this way** (verify cart via the cart API, or a
single manual add). Pace + headed + fresh-context-per-page, or don't claim it.

## 2026-06-16 · The master profile authenticates PROD + bosterbio, NOT localhost — verify against prod URLs

**Discovery.** `captureMasterFromChrome` copies the operator's LIVE Chrome cookies, so
the master holds sessions for what they actually browse: `guild-os-ten.vercel.app`
(prod GuildOS), `<REDACTED-HOST>` (prod boster_nexus), `bosterbio.com`,
`cjgeoai.com`, Google. It does **NOT** hold `localhost:3002` / `localhost:3001`
cookies (the operator signs into local dev via CIC / a different context, not their
main Chrome). Verified 2026-06-16: `authed` render of `localhost:3002/caravan` →
302 `/signin`, but `guild-os-ten.vercel.app/caravan` → 200 authed (full dashboard);
same for `<REDACTED-HOST>` `/ooa-dashboard` + `/purchase-orders/open-items`.

**Consequence / guidance.** To visually verify an auth-gated app feature with
leafBrowser, target its **production URL**, not localhost — the cookie is there.
This means a feature must be **DEPLOYED to prod** to be leaf-verifiable; a local-only
build (uncommitted worktree changes, or a dev-only page like the OOA user-drill-down
and the Open-PO per-item status field, both confirmed ABSENT on prod 2026-06-16)
**cannot** be verified this way. The fix for a local-only target is either (a) deploy
it to prod then verify, or (b) capture a localhost login into the master (operator
logs into the local dev server in their main Chrome, then `ensureMasterFromChrome`)
AND have that dev server running. Do not assume `authed:true` reaches localhost.
## 2026-07-03 - Isolated verification launches prefer system Chrome; purrview disables lazy-scroll

**Decision.** `launchContext` uses the operator's system Chrome for isolated
headless screenshots when it can find it, and `questPurrview.autoVisualVerify`
calls `screenshotUrl` with `waitUntil:"domcontentloaded"` and `scrollLazyLoad:false`.

**Rejected alternative.** Keep isolated screenshots on Playwright's bundled
Chromium and let purrview use the screenshot primitive's default
`networkidle`+lazy-scroll behavior.

**Why.** On the Hermes visual-gate acceptance quest, bundled/default rendering
stalled the real `autoVisualVerify` path: the deliverable was a valid public
Supabase HTML page, but the purrview capture never returned. A bounded system
Chrome render of the same URL produced a real PNG in about two seconds. Purrview's
job is to capture enough real pixels for the independent vision judge, not to
crawl an arbitrary page forever.

**Consequence if violated.** Hermes-produced deliverables can reach `purrview`
with valid refs and still never get a `verify_screenshot_ref`, so the review gate
appears broken even though the worker output is correct.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/leafBrowser/index.js -->
```js
/**
 * leafBrowser weapon — THE single browser substrate for GuildOS automation.
 *
 * It exposes TWO surfaces over ONE shared profile/auth model. Both share
 * findChrome() + the isolated-vs-profile-copy decision; nothing else in the repo
 * should launch Chromium/Playwright directly (see engineering_decisions.md
 * 2026-06-15 — consolidation).
 *
 *  SURFACE 1 — MCP injection for LEAVES (prepareLeafBrowser): hand a headless
 *  `claude --print` leaf a live `mcp__playwright__*` browser via
 *  `--mcp-config`. Used when an LLM leaf must DRIVE a page interactively.
 *
 *  SURFACE 2 — in-process render-to-PNG (launchContext / screenshotUrl):
 *  deterministic "navigate a URL → PNG buffer" with no LLM in the loop. This is
 *  what visual-verification (questPurrview.autoVisualVerify), poster rendering
 *  (poster_render), and headless QA (cicVerify) actually need. Before the
 *  2026-06-15 consolidation each of those weapons hand-rolled its own
 *  `chromium.launch`; they now all delegate here.
 *
 * Why this weapon owns both: CIC (Claude-in-Chrome) rides Chrome Native
 * Messaging, binds to ONE interactive session, and is undeclarable in config, so
 * a headless leaf can never inherit it (memory reference_cic_not_mcp_leaf_browser).
 * Playwright is the only headless-capable path; centralizing it here keeps a
 * single auth model and a single chromium-resolution policy.
 *
 * Profile model (full-profile-copy, chosen 2026-06-13): Chrome locks a
 * user-data-dir, so each authenticated leaf gets its OWN copy of the master
 * profile. The copy is selective (auth-bearing files only, cache excluded) to
 * stay small. The master profile is created once, interactively, by the user
 * (captureMaster / cli.mjs capture) — only the user can perform the logins.
 * AUTHENTICATED mode is a no-op until that capture runs; ISOLATED mode (public /
 * localhost / unauthenticated targets — the common case) needs no profile and
 * works immediately.
 */

import { existsSync, mkdirSync, rmSync, cpSync, readFileSync, writeFileSync, statSync } from "fs";
import { join, dirname } from "path";
import { execSync } from "child_process";
import { logWeaponCall } from "../weaponLog/index.js";

export const toc = {
  screenshotUrl:
    "THE single in-process render-to-PNG primitive. Navigate a URL or public-relative ref (proving-grounds/…), scroll to trigger lazy-loads, full-page screenshot → PNG Buffer (or base64). Handles PDFs via pdf.js (multi-page), element shots via `selector`, and fails CLOSED on a blank/errored page. Isolated (no auth) by default; pass authed:true to use the operator's profile copy. Every screenshot job (verify, poster, QA) goes through here — do NOT call chromium.launch elsewhere.",
  launchContext:
    "Low-level: launch ONE Playwright browser context (the only chromium-launch site in the repo) with the isolated-vs-profile-copy auth decision applied. Returns { ok, context, browser, mode, close }. Use when you need to drive the page yourself (e.g. poster_render injecting data); prefer screenshotUrl for plain screenshots.",
  prepareLeafBrowser:
    "Emit an --mcp-config JSON declaring @playwright/mcp with the auth/profile decision applied, for injecting an authenticated browser into a `claude --print` leaf. Default: per-leaf COPY of the master profile. directProfileDir → use that profile IN PLACE, no copy (REQUIRED for Google/Gmail — copying re-triggers the passkey wall; single-flight). outputDir → where the leaf's screenshots land (--output-dir). Returns { ok, mcpConfigPath, profileDir, mode } or { ok:false, reason }.",
  cleanupLeafBrowser: "Remove a leaf's profile copy + mcp-config temp file after the leaf finishes.",
  hasMasterProfile: "True if a captured master profile DIR exists (Default/ or Local State present). NB: not sufficient for auth — use hasMasterCookie for the real dispatch gate.",
  hasMasterCookie: "True only when the master holds a session Cookies DB — the real 'auth armed' gate. A profile without it cannot pass an SSR auth gate (e.g. /caravan). Dispatch gates on this.",
  ensureMasterFromChrome: "Opportunistic self-arming gate: returns true once a cookie is present; otherwise attempts captureMasterFromChrome (succeeds the next moment Chrome isn't holding Cookies under its exclusive lock). Call this from dispatch so the pipeline arms itself with ZERO operator action.",
  findChrome: "Locate the Chrome executable on this machine (registry App Paths → known per-user/Program Files paths).",
  captureMasterProfileCmd: "Return the headed-Chrome command the USER runs once to create + log into the master profile.",
  openInteractiveLogin:
    "Open a HEADED, persistent, anti-automation Playwright browser on a DEDICATED profile (.browser-profiles/<name>, default 'interactive') and navigate to a URL so the OPERATOR can log in BY HAND (passkey/2FA included). Optional seedStatePath (an exportStorageState JSON) pre-seeds the account so the target RECOGNIZES the operator before they touch it (Gmail shows their email in the account chooser, not a blank sign-in); resetProfile wipes a stale prior session first. Because the login completes IN this profile, the device-bound session is native and PERSISTS for days — the path for sites that reject copied cookies (Gmail/Google). Stays open until the operator closes the window. Run detached via `cli.mjs login [url]`. Reuse the SAME profile directly (no copy) to keep the session.",
  exportStorageState:
    "Export a profile's cookies+localStorage as a Playwright storageState JSON — the stable, plaintext, lock-free auth artifact a leaf imports via newContext({storageState}). Reads WITHOUT navigating (no token-refresh/rotation side effect). From the master profile it yields a JSON whose Google cookies make Gmail recognize the operator's account. Use to seed openInteractiveLogin or to hand leaves a portable auth file.",
  captureMasterFromChrome:
    "NON-INTERACTIVE master capture: copy the operator's LIVE Chrome login state (cookies + Local State key + localStorage) into the master slot — zero manual step when they are already logged in. Copies an auth-only allowlist (the full Default profile is ~1 GB) and tolerates per-file locks while Chrome runs. Prefer this over captureMasterProfileCmd when the operator wants no operation. Returns { ok, masterDir, copied, skipped }.",
  navigateAndGetText:
    "Navigate a URL with a Playwright browser (supports directProfileDir for the interactive Google profile) and return the page's innerText. Use for reading Google Drive folder listings, Google Docs, or any auth-walled page where you need TEXT not a screenshot. Pass directProfileDir to use the native Google login profile. Returns { ok, text, url, mode }.",
};

// Repo-root-relative locations (gitignored — see .gitignore .browser-profiles/).
const PROFILES_ROOT = join(process.cwd(), ".browser-profiles");
const MASTER_DIR = join(PROFILES_ROOT, "master");
const LEAF_ROOT = join(PROFILES_ROOT, "leaves");
const MCP_TMP_ROOT = join(PROFILES_ROOT, "mcp-config");

// Cache / non-auth subdirs excluded from the per-leaf copy so a copy is small
// and fast. Auth lives in Cookies / Login Data / Network/ / Web Data /
// Preferences + top-level Local State, all of which ARE copied.
const EXCLUDE_DIR_NAMES = new Set([
  "Cache",
  "Code Cache",
  "GPUCache",
  "GrShaderCache",
  "ShaderCache",
  "DawnCache",
  "DawnGraphiteCache",
  "DawnWebGPUCache",
  "Service Worker",
  "Crashpad",
  "component_crx_cache",
  "extensions_crx_cache",
  "Download Service",
  "optimization_guide_model_store",
  "segmentation_platform",
]);

// Auth-only allowlist for NON-INTERACTIVE capture from the operator's LIVE Chrome
// profile (captureMasterFromChrome). Only the files that actually carry a
// logged-in session — the full Default profile is ~1 GB, so copying it whole (and
// then again per leaf) is wasteful. {p} = the profile sub-dir ("Default").
//   - Local State            → top-level os_crypt key that decrypts Cookies.
//   - Network/Cookies(+wal)   → cookie-based sessions (GuildOS /caravan = Supabase SSR cookie).
//   - Cookies (legacy)        → pre-Network-dir Chrome location, if present.
//   - Preferences             → avoids Chrome's "profile error" repair prompt on launch.
//   - Local Storage/leveldb   → localStorage-based sessions (Supabase browser client, etc.).
const AUTH_CAPTURE_PATHS = [
  "Local State",
  "{p}/Network/Cookies",
  "{p}/Network/Cookies-wal",
  "{p}/Network/Cookies-journal",
  "{p}/Cookies",
  "{p}/Cookies-journal",
  "{p}/Preferences",
  "{p}/Local Storage/leveldb",
];

// Lock / singleton files that must NOT travel into a copy (a stale lock confuses
// the Chrome that later opens the copy; leveldb's LOCK can't be copied while held).
const COPY_SKIP_FILES = new Set(["LOCK", "SingletonLock", "SingletonCookie", "SingletonSocket", "lockfile"]);

/** Copy one path, tolerating Chrome's live write-lock on SQLite files. Windows
 *  CopyFile (used by cpSync/copyFileSync) requests only FILE_SHARE_READ and fails
 *  EBUSY/EPERM against a file Chrome holds open for write (e.g. Network/Cookies).
 *  libuv's read-handle (readFileSync, 'r') shares read+write+delete, so it can read
 *  the same bytes. Dirs (leveldb) go through cpSync; single files fall back to a
 *  byte read→write when CopyFile is blocked. Returns true on success. */
function lockTolerantCopy(src, dest) {
  mkdirSync(dirname(dest), { recursive: true });
  if (statSync(src).isDirectory()) {
    cpSync(src, dest, {
      recursive: true, force: true, errorOnExist: false,
      filter: (s) => !COPY_SKIP_FILES.has(s.split(/[\\/]/).pop()),
    });
    return true;
  }
  try {
    cpSync(src, dest, { force: true, errorOnExist: false });
    return true;
  } catch {
    writeFileSync(dest, readFileSync(src)); // libuv 'r' handle bypasses the write-lock
    return true;
  }
}

export function hasMasterProfile() {
  return existsSync(join(MASTER_DIR, "Default")) || existsSync(join(MASTER_DIR, "Local State"));
}

/** TRUE only when the master holds a session COOKIE — the real "auth is armed"
 *  gate. A profile with Local State + localStorage but NO Cookies DB cannot pass
 *  a server-side (SSR) auth gate like GuildOS /caravan, so dispatch must gate on
 *  THIS, not hasMasterProfile(). (Verified 2026-06-16: cookie-less master → /caravan
 *  302→/signin.) */
export function hasMasterCookie() {
  return (
    existsSync(join(MASTER_DIR, "Default", "Network", "Cookies")) ||
    existsSync(join(MASTER_DIR, "Default", "Cookies"))
  );
}

/** Opportunistic self-arming gate for browser dispatch. Returns true once a
 *  session cookie is present. If it is NOT yet present, attempts a
 *  captureMasterFromChrome() — which succeeds the moment the operator's Chrome
 *  isn't holding Cookies under its exclusive lock (i.e. next time Chrome is
 *  closed). Cheap when already armed (cookie present → no copy). This is what
 *  makes the pipeline require ZERO operator action: it captures the live login
 *  whenever Chrome is next down, with no interactive capture step. */
export function ensureMasterFromChrome() {
  if (hasMasterCookie()) return true;
  try { captureMasterFromChrome(); } catch { /* locked / no chrome — stays unarmed, retry next tick */ }
  return hasMasterCookie();
}

/** Resolve the live Chrome "User Data" dir from the located chrome.exe
 *  (.../Application/chrome.exe → .../User Data), with a LOCALAPPDATA fallback. */
export function chromeUserDataDir() {
  const exe = findChrome();
  if (exe) {
    const ud = join(dirname(dirname(exe)), "User Data"); // Application → Chrome → User Data
    if (existsSync(ud)) return ud;
  }
  const fb = process.env.LOCALAPPDATA && join(process.env.LOCALAPPDATA, "Google/Chrome/User Data");
  return fb && existsSync(fb) ? fb : null;
}

/**
 * Capture the master profile NON-INTERACTIVELY by copying the operator's LIVE
 * Chrome login state into MASTER_DIR. Zero-operation alternative to the headed
 * captureMasterProfileCmd login — correct when the operator is already logged into
 * the target sites in their main Chrome and wants no manual step. Copies only the
 * auth allowlist (AUTH_CAPTURE_PATHS) and tolerates per-file locks (Chrome may be
 * running), so a single locked file degrades to a skip, not a thrown capture.
 * @returns {{ ok:boolean, masterDir:string, copied:string[], skipped:string[], reason?:string }}
 */
export function captureMasterFromChrome({ profile = "Default", userDataDir = null } = {}) {
  const ud = userDataDir || chromeUserDataDir();
  if (!ud) return { ok: false, masterDir: MASTER_DIR, copied: [], skipped: [], reason: "chrome_user_data_not_found" };
  // NON-DESTRUCTIVE (no rmSync wipe): a re-run that can't grab the live-locked
  // Cookies must PRESERVE a previously captured cookie, not destroy it. Per-file
  // skip-if-exists keeps repeated opportunistic calls cheap — only the Cookies
  // file is re-attempted every time (it's the one under Chrome's exclusive lock,
  // so we keep retrying until a Chrome-closed moment lets it through).
  mkdirSync(MASTER_DIR, { recursive: true });
  const copied = [];
  const skipped = [];
  for (const tmpl of AUTH_CAPTURE_PATHS) {
    const rel = tmpl.replace("{p}", profile);
    const isCookie = /Cookies/.test(rel);
    const src = join(ud, rel);
    const dest = join(MASTER_DIR, rel);
    if (existsSync(dest) && !isCookie) { skipped.push(`${rel} (have)`); continue; }
    if (!existsSync(src)) { skipped.push(`${rel} (absent)`); continue; }
    try {
      lockTolerantCopy(src, dest);
      copied.push(rel);
    } catch (e) {
      skipped.push(`${rel} (${e.code || e.message})`);
    }
  }
  const ok = hasMasterCookie();
  logWeaponCall({
    weapon: "leafBrowser", fn: "captureMasterFromChrome",
    input: { userDataDir: ud, profile },
    output: { copied: copied.length, skipped: skipped.length, ok },
    status: ok ? "ok" : "fail",
  });
  return { ok, masterDir: MASTER_DIR, copied, skipped };
}

/** Locate chrome.exe: registry App Paths first (authoritative on Windows), then known paths. */
export function findChrome() {
  if (process.platform === "win32") {
    for (const hive of ["HKCU", "HKLM"]) {
      try {
        const out = execSync(
          `reg query "${hive}\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\App Paths\\chrome.exe" /ve`,
          { encoding: "utf8", stdio: ["ignore", "pipe", "ignore"] },
        );
        const m = out.match(/REG_SZ\s+(.+chrome\.exe)/i);
        if (m && existsSync(m[1].trim())) return m[1].trim();
      } catch { /* try next */ }
    }
  }
  const candidates = [
    process.env.LOCALAPPDATA && join(process.env.LOCALAPPDATA, "Google/Chrome/Application/chrome.exe"),
    "C:/Program Files/Google/Chrome/Application/chrome.exe",
    "C:/Program Files (x86)/Google/Chrome/Application/chrome.exe",
    "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
    "/usr/bin/google-chrome",
  ].filter(Boolean);
  for (const c of candidates) if (existsSync(c)) return c;
  return null;
}

export function captureMasterProfileCmd() {
  const chrome = findChrome();
  return {
    masterDir: MASTER_DIR,
    chrome,
    instructions:
      "Run this ONCE (headed). A fresh Chrome opens on the master profile — log into every site leaves must reach, then close Chrome. Re-run anytime to refresh logins.",
    command: chrome ? `"${chrome}" --user-data-dir="${MASTER_DIR}"` : null,
    cli: "node libs/weapon/leafBrowser/cli.mjs capture",
  };
}

/** Selective recursive copy of the master profile into destDir, skipping cache dirs. */
function copyProfile(srcDir, destDir) {
  mkdirSync(destDir, { recursive: true });
  cpSync(srcDir, destDir, {
    recursive: true,
    force: true,
    errorOnExist: false,
    filter: (src) => {
      const base = src.split(/[\\/]/).pop();
      return !EXCLUDE_DIR_NAMES.has(base);
    },
  });
}

/**
 * Prepare a per-leaf authenticated browser. Copies master→leaf profile and
 * writes an --mcp-config JSON for @playwright/mcp pointed at the copy.
 * @param {{ workerId:string, headless?:boolean }} opts
 * @returns {{ ok:true, mcpConfigPath:string, profileDir:string, serverName:string }
 *          | { ok:false, reason:string }}
 */
export function prepareLeafBrowser({ workerId, headless = true, allowIsolated = false, directProfileDir = null, outputDir = null } = {}) {
  if (!workerId) return fail("workerId required");
  const chrome = findChrome();
  if (!chrome) return fail("chrome_not_found");

  // Profile mode selection:
  //  - directProfileDir given → DIRECT: use that profile IN PLACE (no copy). Needed
  //    for Google/Gmail, where copying re-triggers the device-binding passkey wall
  //    (ED 2026-06-18) — the operator's native interactive login must be reused as-is.
  //    Single-flight (Chrome locks a user-data-dir); never cleaned up by cleanupLeafBrowser.
  //  - master profile present  → AUTHENTICATED: per-leaf copy via --user-data-dir
  //    (carries the operator's logins; needed for auth-walled targets).
  //  - no master, allowIsolated → ISOLATED: ephemeral in-memory profile (--isolated).
  //  - no master, !allowIsolated → refuse (caller must capture a profile first).
  const useDirect = !!(directProfileDir && existsSync(directProfileDir));
  const authed = useDirect || hasMasterProfile();
  if (!authed && !allowIsolated) return fail("no_master_profile");

  const profileDir = useDirect ? directProfileDir : (authed ? join(LEAF_ROOT, workerId) : null);
  const mcpConfigPath = join(MCP_TMP_ROOT, `${workerId}.json`);
  try {
    if (authed && !useDirect) {
      rmSync(profileDir, { recursive: true, force: true });
      copyProfile(MASTER_DIR, profileDir);
    }
    mkdirSync(dirname(mcpConfigPath), { recursive: true });
    const args = [
      "-y",
      "@playwright/mcp@latest",
      ...(headless ? ["--headless"] : []),
      ...(authed ? ["--user-data-dir", profileDir] : ["--isolated"]),
      "--executable-path",
      chrome,
    ];
    if (outputDir) { mkdirSync(outputDir, { recursive: true }); args.push("--output-dir", outputDir); }
    // Windows spawn gotcha: an MCP host spawns the server command directly (no
    // shell), and `npx` on Windows is `npx.cmd` — a bare "npx" fails to launch, the
    // server never connects, and the leaf silently sees NO mcp__playwright__* tools
    // (observed 2026-06-18). Wrap via `cmd /c` on win32 so npx resolves.
    const server = process.platform === "win32"
      ? { command: "cmd", args: ["/c", "npx", ...args] }
      : { command: "npx", args };
    const config = { mcpServers: { playwright: server } };
    writeFileSync(mcpConfigPath, JSON.stringify(config, null, 2));
    const mode = useDirect ? "direct-profile" : authed ? "authenticated" : "isolated";
    const out = { ok: true, mcpConfigPath, profileDir, serverName: "playwright", mode };
    logWeaponCall({ weapon: "leafBrowser", fn: "prepareLeafBrowser", input: { workerId, mode }, output: { profileDir }, status: "ok" });
    return out;
  } catch (e) {
    if (profileDir && !useDirect) { try { rmSync(profileDir, { recursive: true, force: true }); } catch { /* ignore */ } }
    return fail(`prepare_failed: ${e.message}`);
  }
}

export function cleanupLeafBrowser({ workerId } = {}) {
  if (!workerId) return { ok: false, reason: "workerId required" };
  try {
    rmSync(join(LEAF_ROOT, workerId), { recursive: true, force: true });
    rmSync(join(MCP_TMP_ROOT, `${workerId}.json`), { force: true });
    return { ok: true };
  } catch (e) {
    return { ok: false, reason: e.message };
  }
}

function fail(reason) {
  logWeaponCall({ weapon: "leafBrowser", fn: "prepareLeafBrowser", input: {}, output: { reason }, status: "fail" });
  return { ok: false, reason };
}

/**
 * Open a HEADED, persistent Playwright browser on a DEDICATED profile and
 * navigate to `url` so the OPERATOR logs in BY HAND (passkey/2FA). The cookie-copy
 * master fails for Google because the session was minted in a DIFFERENT browser
 * (passkey challenge on a "new" device); logging in HERE binds the session to THIS
 * profile natively, so it PERSISTS across restarts for days. Anti-automation flags
 * (no --enable-automation, navigator.webdriver spoofed, real system Chrome) so
 * Google does not flag the sign-in. Stays open until the operator closes the
 * window — everything is written to the profile dir live, so the login survives.
 * Reuse the SAME profileDir DIRECTLY (never a copy) to keep the session valid.
 * @returns {Promise<{ ok:true, profileDir:string } | { ok:false, reason:string }>}
 */
export async function openInteractiveLogin({ url = "https://mail.google.com/", profileName = "interactive", seedStatePath = null, resetProfile = false } = {}) {
  let chromium;
  try { ({ chromium } = await import("playwright")); }
  catch (e) { return { ok: false, reason: `playwright_unavailable: ${e.message}` }; }
  const chrome = findChrome();
  const profileDir = join(PROFILES_ROOT, profileName);
  // Reset wipes a stale/blank prior session so the seed starts clean (the empty
  // interactive profile otherwise shows a BARE sign-in with no account).
  if (resetProfile) rmSync(profileDir, { recursive: true, force: true });
  mkdirSync(profileDir, { recursive: true });
  let context;
  try {
    context = await chromium.launchPersistentContext(profileDir, {
      headless: false,
      ...(chrome ? { executablePath: chrome } : {}),
      ignoreDefaultArgs: ["--enable-automation"],
      args: ["--disable-blink-features=AutomationControlled", "--no-first-run", "--no-default-browser-check", "--start-maximized"],
      viewport: null,
    });
  } catch (e) {
    return { ok: false, reason: `launch_failed: ${e.message}` };
  }
  // Seed the account from a storageState JSON (exportStorageState output) so the
  // target RECOGNIZES the operator before they touch it — Gmail shows the account
  // chooser with their email instead of a blank sign-in. Cookies write into the
  // persistent profile, so the eventual login they complete is durable here.
  let seeded = 0;
  if (seedStatePath && existsSync(seedStatePath)) {
    try {
      const state = JSON.parse(readFileSync(seedStatePath, "utf8"));
      if (Array.isArray(state.cookies) && state.cookies.length) {
        await context.addCookies(state.cookies);
        seeded = state.cookies.length;
      }
    } catch { /* seeding is best-effort */ }
  }
  await context.addInitScript(() => {
    Object.defineProperty(navigator, "webdriver", { get: () => undefined });
  });
  const page = context.pages()[0] || (await context.newPage());
  await page.goto(url, { waitUntil: "domcontentloaded", timeout: 60000 }).catch(() => {});
  logWeaponCall({ weapon: "leafBrowser", fn: "openInteractiveLogin", input: { url, profileName, seeded }, output: { profileDir }, status: "ok" });
  console.log(`[leafBrowser] interactive browser OPEN on ${profileDir} at ${url} (seeded ${seeded} cookies) — log in by hand; the session persists in that profile. Close the window when done.`);
  await new Promise((resolve) => context.on("close", resolve));
  return { ok: true, profileDir, seeded };
}

/**
 * Export a profile's cookies + localStorage as a Playwright storageState JSON —
 * the stable, plaintext, lock-free auth artifact a leaf imports via
 * `newContext({ storageState })`. Reads from a persistent context WITHOUT
 * navigating (so it never triggers a token refresh/rotation that could disturb
 * the source session). The master profile's copied cookies recognize the
 * operator's Google account, so this JSON makes Gmail show the account chooser.
 * @returns {Promise<{ ok:true, path:string, cookieCount:number } | { ok:false, reason:string }>}
 */
export async function exportStorageState({ fromProfile = "master", outPath = null } = {}) {
  let chromium;
  try { ({ chromium } = await import("playwright")); }
  catch (e) { return { ok: false, reason: `playwright_unavailable: ${e.message}` }; }
  const profileDir = join(PROFILES_ROOT, fromProfile);
  if (!existsSync(profileDir)) return { ok: false, reason: `profile_not_found: ${fromProfile}` };
  const dest = outPath || join(PROFILES_ROOT, `${fromProfile}-state.json`);
  const chrome = findChrome();
  let context;
  try {
    context = await chromium.launchPersistentContext(profileDir, {
      headless: true, ...(chrome ? { executablePath: chrome } : {}),
      ignoreDefaultArgs: ["--enable-automation"],
    });
  } catch (e) { return { ok: false, reason: `launch_failed: ${e.message}` }; }
  let state;
  try { state = await context.storageState(); }
  finally { await context.close(); }
  writeFileSync(dest, JSON.stringify(state));
  logWeaponCall({ weapon: "leafBrowser", fn: "exportStorageState", input: { fromProfile }, output: { dest, cookieCount: state.cookies.length }, status: "ok" });
  return { ok: true, path: dest, cookieCount: state.cookies.length };
}

// ───────────────────────────────────────────────────────────────────────────
// SURFACE 2 — in-process render-to-PNG (the consolidated screenshot substrate).
// Everything that used to do `await import("playwright"); chromium.launch(...)`
// in its own weapon (questPurrview, cicVerify, poster_render, hand-rolled leaf
// fallbacks) now routes through launchContext / screenshotUrl. See ED 2026-06-15.
// ───────────────────────────────────────────────────────────────────────────

/** Resolve a deliverable ref / target to a navigable URL.
 *  - absolute http(s)/file/data → unchanged.
 *  - public-relative ("proving-grounds/x/index.html", "/poster-templates/…") →
 *    resolved against baseUrl (the GuildOS dev server, default :3002).
 *  Returns { url, fileFallback } where fileFallback is a file:// URL into
 *  ./public for static assets when the dev server is down. */
export function resolveTargetUrl(ref, { baseUrl = "http://localhost:3002" } = {}) {
  const raw = String(ref || "").trim();
  if (!raw) return { url: null, fileFallback: null };
  if (/^(https?|file|data):/i.test(raw)) return { url: raw, fileFallback: null };
  const rel = raw.replace(/^\/+/, "");
  const url = `${baseUrl.replace(/\/+$/, "")}/${rel}`;
  const fileFallback = "file:///" + join(process.cwd(), "public", rel).replace(/\\/g, "/");
  return { url, fileFallback };
}

/** Ensure a per-leaf authenticated profile copy exists; returns its dir. */
function ensureAuthProfile(workerId) {
  const profileDir = join(LEAF_ROOT, workerId || "screenshot");
  rmSync(profileDir, { recursive: true, force: true });
  copyProfile(MASTER_DIR, profileDir);
  return profileDir;
}

/**
 * The ONLY place in the repo that launches Chromium. Returns a ready browser
 * context with the auth decision applied.
 *   - authed:true + a captured master → persistent context over a profile copy
 *     (carries the operator's logins), driving SYSTEM Chrome via executablePath.
 *   - otherwise → ephemeral bundled-Chromium context (no login). Correct for
 *     public / localhost / unauthenticated targets (the common case).
 * @returns {Promise<{ ok:true, context, browser, mode:"authenticated"|"isolated", close:()=>Promise<void> }
 *          | { ok:false, reason:string }>}
 */
export async function launchContext({
  authed = false,
  workerId = null,
  headless = true,
  viewport = { width: 1280, height: 1600 },
  deviceScaleFactor = 1,
  directProfileDir = null,
} = {}) {
  let chromium;
  try { ({ chromium } = await import("playwright")); }
  catch (e) { return { ok: false, reason: `playwright_unavailable: ${e.message}` }; }

  // directProfileDir: use a native login profile IN PLACE (no copy) — required for
  // Google/Gmail which reject copied sessions with a passkey wall (ED 2026-06-18).
  if (directProfileDir && existsSync(directProfileDir)) {
    const exe = findChrome();
    try {
      const context = await chromium.launchPersistentContext(directProfileDir, {
        headless,
        viewport,
        deviceScaleFactor,
        ...(exe ? { executablePath: exe } : {}),
      });
      return {
        ok: true, context, browser: context.browser(), mode: "direct-profile",
        close: async () => { try { await context.close(); } catch { /* ignore */ } },
      };
    } catch (e) {
      return { ok: false, reason: `launch_failed_direct: ${e.message}` };
    }
  }

  const useAuth = authed && hasMasterProfile();
  try {
    if (useAuth) {
      const exe = findChrome();
      const profileDir = ensureAuthProfile(workerId);
      const context = await chromium.launchPersistentContext(profileDir, {
        headless,
        viewport,
        deviceScaleFactor,
        ...(exe ? { executablePath: exe } : {}),
      });
      return {
        ok: true, context, browser: context.browser(), mode: "authenticated",
        close: async () => {
          try { await context.close(); } catch { /* ignore */ }
          try { rmSync(profileDir, { recursive: true, force: true }); } catch { /* ignore */ }
        },
      };
    }
    const exe = findChrome();
    const browser = await chromium.launch({
      headless,
      ...(exe ? { executablePath: exe } : {}),
    });
    const context = await browser.newContext({ viewport, deviceScaleFactor });
    return {
      ok: true, context, browser, mode: "isolated",
      close: async () => { try { await browser.close(); } catch { /* ignore */ } },
    };
  } catch (e) {
    return { ok: false, reason: `launch_failed: ${e.message}` };
  }
}

/**
 * Render a target (URL or public-relative ref) to a full-page PNG. THE single
 * screenshot primitive — absorbs the logic the divergent weapons each had:
 * pdf.js multi-page rendering, lazy-load scrolling, fail-closed-on-blank,
 * optional element (`selector`) shots, and dev-server→file:// fallback.
 *
 * @param {{
 *   ref?:string, url?:string, kind?:string,
 *   fullPage?:boolean, selector?:string,
 *   viewport?:{width:number,height:number}, deviceScaleFactor?:number,
 *   waitUntil?:"load"|"domcontentloaded"|"networkidle",
 *   scrollLazyLoad?:boolean, failClosedOnBlank?:boolean,
 *   baseUrl?:string, timeoutMs?:number,
 *   authed?:boolean, workerId?:string, encoding?:"buffer"|"base64",
 *   beforeShot?:string,   // stringified fn body run via page.evaluate before the shot
 * }} opts
 * @returns {Promise<{ ok:true, buffer?:Buffer, base64?:string, mode:string, resolvedUrl:string }
 *          | { ok:false, error:string }>}
 */
export async function screenshotUrl(opts = {}) {
  const {
    ref, url, kind,
    fullPage = true, selector = null,
    viewport = { width: 1280, height: 1600 }, deviceScaleFactor = 1,
    waitUntil = "networkidle",
    scrollLazyLoad = true, failClosedOnBlank = true,
    baseUrl = "http://localhost:3002", timeoutMs = 30000,
    authed = false, workerId = null, encoding = "buffer",
    directProfileDir = null,
  } = opts;

  const target = url || ref;
  if (!target) return { ok: false, error: "ref_or_url_required" };
  const { url: primary, fileFallback } = url ? { url, fileFallback: null } : resolveTargetUrl(ref, { baseUrl });

  const launched = await launchContext({ authed, workerId, viewport, deviceScaleFactor, directProfileDir });
  if (!launched.ok) return { ok: false, error: launched.reason };
  const { context, mode, close } = launched;

  try {
    const page = await context.newPage();
    const isPdf = kind === "document" || /\.pdf(\?|#|$)/i.test(primary || "");

    let buffer;
    if (isPdf) {
      buffer = await renderPdf(page, primary, timeoutMs);
    } else {
      let resolvedUrl = primary;
      let resp = await page.goto(primary, { waitUntil, timeout: timeoutMs }).catch(() => null);
      let contentLen = await pageTextLen(page);
      // dev-server down? fall back to file:// for static public assets.
      if ((!resp || !resp.ok()) && contentLen === 0 && fileFallback) {
        resolvedUrl = fileFallback;
        resp = await page.goto(fileFallback, { waitUntil: "load", timeout: timeoutMs }).catch(() => null);
        contentLen = await pageTextLen(page);
      }
      if (scrollLazyLoad) await autoScroll(page);
      contentLen = await pageTextLen(page);
      if (failClosedOnBlank && (!resp || !resp.ok()) && contentLen === 0) {
        throw new Error(`navigation to ${resolvedUrl} failed (status=${resp ? resp.status() : "no-response"}) and page rendered empty`);
      }
      if (selector) {
        const el = await page.$(selector);
        if (!el) throw new Error(`selector not found: ${selector}`);
        buffer = await el.screenshot({ type: "png" });
      } else {
        buffer = await page.screenshot({ fullPage, type: "png" });
      }
    }

    const out = { ok: true, mode, resolvedUrl: primary };
    if (encoding === "base64") out.base64 = buffer.toString("base64"); else out.buffer = buffer;
    logWeaponCall({ weapon: "leafBrowser", fn: "screenshotUrl", input: { target: primary, mode, authed: mode === "authenticated" }, output: { bytes: buffer.length }, status: "ok" });
    return out;
  } catch (e) {
    logWeaponCall({ weapon: "leafBrowser", fn: "screenshotUrl", input: { target: primary }, output: { error: e.message }, status: "fail" });
    return { ok: false, error: (e.message || String(e)).split("\n")[0] };
  } finally {
    await close();
  }
}

async function pageTextLen(page) {
  return page.evaluate(() => (document.body ? document.body.innerText.trim().length : 0)).catch(() => 0);
}

async function autoScroll(page) {
  await page
    .evaluate(
      () =>
        new Promise((res) => {
          let y = 0;
          const t = setInterval(() => {
            window.scrollBy(0, 500);
            y += 500;
            if (y >= document.body.scrollHeight) { clearInterval(t); res(); }
          }, 120);
        }),
    )
    .catch(() => {});
  await page.waitForTimeout(800);
  await page.evaluate(() => window.scrollTo(0, 0)).catch(() => {});
}

/** pdf.js renders pages 1..N (capped) stacked into one canvas column, then we
 *  screenshot the stack. Headless Chromium downloads PDFs instead of viewing,
 *  so we render them ourselves. */
async function renderPdf(page, ref, timeoutMs) {
  const ver = "3.11.174";
  await page.setContent(
    `<!DOCTYPE html><html><head><script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/${ver}/pdf.min.js"></script></head><body style="margin:0;background:#fff"><div id="pages"></div></body></html>`,
    { waitUntil: "load", timeout: timeoutMs },
  );
  const meta = await page.evaluate(async (pdfUrl) => {
    const L = window.pdfjsLib;
    if (!L) return { ok: false };
    L.GlobalWorkerOptions.workerSrc = "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js";
    const pdf = await L.getDocument(pdfUrl).promise;
    const MAX_PAGES = 15;
    const n = Math.min(pdf.numPages, MAX_PAGES);
    const container = document.getElementById("pages");
    for (let i = 1; i <= n; i++) {
      const p = await pdf.getPage(i);
      const vp = p.getViewport({ scale: 1.5 });
      const c = document.createElement("canvas");
      c.width = vp.width; c.height = vp.height; c.style.display = "block";
      container.appendChild(c);
      await p.render({ canvasContext: c.getContext("2d"), viewport: vp }).promise;
    }
    return { ok: true, total: pdf.numPages, rendered: n };
  }, ref);
  if (!meta?.ok || !meta.rendered) throw new Error("pdf.js failed to render the document");
  const el = await page.$("#pages");
  return el.screenshot();
}

/**
 * Navigate to a URL using a direct Chrome profile and return the page text content.
 * Designed for reading Google Drive / Google Docs / Sheets content using the
 * interactive profile (which holds the operator's Google session).
 *
 * @param {{ url:string, directProfileDir?:string, authed?:boolean, workerId?:string,
 *           timeoutMs?:number, waitUntil?:string, selector?:string }} opts
 * @returns {Promise<{ ok:true, text:string, url:string, mode:string }
 *          | { ok:false, error:string }>}
 */
export async function navigateAndGetText({
  url,
  directProfileDir = null,
  authed = false,
  workerId = null,
  timeoutMs = 30000,
  waitUntil = "networkidle",
  selector = null,
} = {}) {
  if (!url) return { ok: false, error: "url required" };
  const launched = await launchContext({ authed, workerId, directProfileDir });
  if (!launched.ok) return { ok: false, error: launched.reason };
  const { context, mode, close } = launched;
  try {
    const page = await context.newPage();
    await page.goto(url, { waitUntil, timeout: timeoutMs }).catch(() => null);
    await autoScroll(page);
    let text;
    if (selector) {
      const el = await page.$(selector);
      text = el ? await el.innerText() : await page.evaluate(() => document.body?.innerText || "");
    } else {
      text = await page.evaluate(() => document.body?.innerText || "");
    }
    return { ok: true, text: text.trim(), url, mode };
  } catch (e) {
    return { ok: false, error: `navigate_failed: ${e.message}` };
  } finally {
    try { await close(); } catch { /* ignore */ }
  }
}

/**
 * Navigate to a URL and evaluate a JS expression on the page, returning the result.
 * Use for extracting structured data (links, folder IDs, file listings) from pages.
 *
 * @param {{ url:string, jsExpr:string, directProfileDir?:string, authed?:boolean,
 *           workerId?:string, timeoutMs?:number, waitUntil?:string }} opts
 * @returns {Promise<{ ok:true, result:any, url:string, mode:string }
 *          | { ok:false, error:string }>}
 */
export async function navigateAndEvalJs({
  url,
  jsExpr,
  directProfileDir = null,
  authed = false,
  workerId = null,
  timeoutMs = 30000,
  waitUntil = "networkidle",
} = {}) {
  if (!url) return { ok: false, error: "url required" };
  if (!jsExpr) return { ok: false, error: "jsExpr required" };
  const launched = await launchContext({ authed, workerId, directProfileDir });
  if (!launched.ok) return { ok: false, error: launched.reason };
  const { context, mode, close } = launched;
  try {
    const page = await context.newPage();
    await page.goto(url, { waitUntil, timeout: timeoutMs }).catch(() => null);
    await autoScroll(page);
    const result = await page.evaluate((expr) => {
      try { return eval(expr); } catch (e) { return { _evalError: e.message }; }
    }, jsExpr);
    return { ok: true, result, url, mode };
  } catch (e) {
    return { ok: false, error: `navigate_eval_failed: ${e.message}` };
  } finally {
    try { await close(); } catch { /* ignore */ }
  }
}

/** Read a prepared mcp-config (debug/inspection helper). */
export function readMcpConfig(workerId) {
  const p = join(MCP_TMP_ROOT, `${workerId}.json`);
  return existsSync(p) ? JSON.parse(readFileSync(p, "utf8")) : null;
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/openai_images/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/weapon/openai_images`

Rationale layer (per CLAUDE.md §6). Append; supersede in place when a decision
changes.

---

## 2026-06-09 — never pass `response_format` to the images endpoints

**Decision.** `generate`/`edit` never send `response_format`. `b64_json` is always
returned regardless.

**Rejected alternative.** Pass `response_format: "b64_json"` for explicitness (the
intuitive, "correct-looking" call).

**Why.** OpenAI's image endpoints 400 with `unknown_parameter` when
`response_format` is supplied — the parameter isn't in the current schema, yet
b64_json is the only format and is always returned. Sending it for "clarity" breaks
every call.

**Consequence if violated.** All image calls fail with `unknown_parameter`; the
caller suspects their request shape when it's a service-side quirk.

---

## 2026-06-09 — the vision judge uses `gpt-4o`; `gpt-4o-mini` is banned for verification (commit 5f9b4c0)

**Decision.** `judge` runs on full `gpt-4o`. `gpt-4o-mini` is explicitly banned for
the verification path.

**Rejected alternative.** Use `gpt-4o-mini` to cut verification cost.

**Why.** Mini measured a ~49% false-alarm rate on dense UI screenshots (the actual
use case) — insufficient visual reasoning. Verification is a gate; a cheap judge
that hallucinates failures (or passes) defeats its purpose. The cost/accuracy
trade-off was settled at 5f9b4c0 so it isn't re-litigated.

**Consequence if violated.** The visual gate accepts wrong images (or false-alarms
on good ones), undermining the verification system that the rest of GuildOS leans
on.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/openai_images/index.js -->
```js
/**
 * OpenAI Images weapon — image generation and editing.
 *
 * Model: configured via OPENAI_IMAGE_MODEL env var, default "gpt-image-2".
 * To swap models cluster-wide (e.g. when a newer model ships), update the env
 * var in deployment + every locally-running .env.local — no code change.
 *
 * ─── CALLER DISCIPLINE (LOCKED 2026-05-22) ────────────────────────────────
 * Callers MUST be API routes (app/api/**) or skill-book actions
 * (libs/skill_book/**). One-off `.mjs` scripts in `scripts/` are NOT a
 * supported mechanism — they bypass the env-driven model source of truth in
 * practice (people hardcode the model name) and bypass the API-route audit
 * surface. The legacy `scripts/_generate-marketing-infographics.mjs` and
 * `scripts/_design_smoke_v4_gpt_image_2.mjs` are NOT precedent for new
 * one-offs; they're being deprecated. If the dev server is broken, surface
 * that — don't fall back to a script.
 *
 * Credential: OPENAI_API_KEY from profiles.env_vars, fallback to process.env.
 *
 * ─── API surface ──────────────────────────────────────────────────────────
 * Generate: POST https://api.openai.com/v1/images/generations
 *   body (JSON): { model, prompt, size, quality, n }
 *   response:    { data: [{ b64_json }], usage }
 *
 * Edit: POST https://api.openai.com/v1/images/edits  (multipart/form-data)
 *   fields: model, prompt, size, quality, n, image[] (Blob | Buffer)
 *   response: { data: [{ b64_json }], usage }
 *
 * ─── Verified behavior (smoke-tested 2026-04-23) ──────────────────────────
 * - DO NOT pass response_format — causes unknown_parameter error; b64_json is
 *   always returned regardless.
 * - Supported sizes: "1024x1024" | "1024x1536" | "1536x1024" | "auto"
 * - Supported quality: "low" | "medium" | "high"
 * - gpt-image-2 requires org verification on the OpenAI platform before the
 *   model unlocks for API calls.
 */

import { database } from "@/libs/council/database";

export const toc = {
  generate: {
    description:
      "Generate an image — infographic, chart, diagram, poster, illustration, logo, icon, banner, slide background, artwork, or photo — from a text prompt using gpt-image-2 (\"ChatGPT image\"). THE way to create any picture / visual / graphic from words. Returns array of b64_json strings. Via run-action, pass a long/multiline prompt with --input-file <json> and write the PNG(s) with --out <path> (n>1 → -1,-2… suffix).",
    // Binary-output contract honored by run-action's `--out`: the result IS an
    // array of base64 PNG payloads ("from":"$"), each decoded and written to a
    // .png file. This is the metadata the adapter cli.mjs used to carry.
    output: { binary: true, encoding: "base64", contentType: "image/png", ext: "png", from: "$" },
  },
  edit: {
    description:
      "Edit, restyle, redraw, composite, inpaint, or merge existing images (change style, swap / add / remove elements, combine multiple pictures) using gpt-image-2 edits endpoint. Accepts image Buffers + prompt. Returns array of b64_json strings. Via run-action, write the PNG(s) with --out <path>.",
    output: { binary: true, encoding: "base64", contentType: "image/png", ext: "png", from: "$" },
  },
  judge:
    "Contextless vision check: ask gpt-4o (full) whether a given image matches a written claim. Optional baselineUrl adds a second image (the AI-approved design baseline) — pass requires both claim satisfaction AND ≥90% visual similarity (or all key elements present). Returns { verdict, confidence, reasoning, model, baseline_compared }. No chat history, no narrative drift. gpt-4o-mini was retired 2026-04-26 (49% false-alarm rate, see CLAUDE.md WWCD bans).",
};

/**
 * API base URL. Defaults to OpenAI direct. Override with OPENAI_IMAGE_BASE_URL
 * (or OPENAI_BASE_URL) to route through a supported-region OpenAI-compatible
 * proxy — needed where OpenAI geo-blocks the images endpoint by request origin
 * ("Country, region, or territory not supported"). Value should be the full
 * base incl. version segment, e.g. "https://my-proxy.example.com/v1".
 */
function apiBase() {
  return (
    process.env.OPENAI_IMAGE_BASE_URL ||
    process.env.OPENAI_BASE_URL ||
    "https://api.openai.com/v1"
  );
}

/**
 * Read the active image-generation model. Single source of truth — every
 * caller (weapon + scripts) goes through this so a model bump is one env
 * change, not a code-wide grep-and-replace.
 */
export function imageModel() {
  return process.env.OPENAI_IMAGE_MODEL || "gpt-image-2";
}

async function loadKey(userId) {
  if (userId) {
    try {
      const db = await database.init("service");
      const { data } = await db
        .from("profiles")
        .select("env_vars")
        .eq("id", userId)
        .single();
      const k = data?.env_vars?.OPENAI_API_KEY;
      if (k) return k;
    } catch {
      /* fall through */
    }
  }
  return process.env.OPENAI_API_KEY || null;
}

/**
 * Generate images from a text prompt.
 *
 * @param {{ prompt: string, size?: string, quality?: string, n?: number, background?: "transparent"|"opaque"|"auto" }} input
 *   background — gpt-image-2 alpha control. "transparent" returns a true alpha PNG
 *   (use for stickers/badges/logos that overlay other imagery). Omit for default.
 * @param {string} [userId]
 * @returns {Promise<string[]>} Array of base64-encoded PNG strings (one per image)
 */
export async function generate(
  { prompt, size = "1024x1024", quality = "medium", n = 1, background } = {},
  userId
) {
  const key = await loadKey(userId);
  if (!key) throw new Error("OPENAI_API_KEY not configured");

  const body = { model: imageModel(), prompt, size, quality, n };
  if (background) body.background = background;

  const res = await fetch(`${apiBase()}/images/generations`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${key}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(body),
  });

  const json = await res.json();
  if (json.error) throw new Error(`OpenAI Images error: ${json.error.message}`);
  return json.data.map((d) => d.b64_json);
}

/**
 * Edit / restyle one or more images.
 *
 * @param {{ images: (Buffer|Blob)[], prompt: string, size?: string, quality?: string, n?: number }} input
 *   images — array of image Buffers (PNG preferred). Max 10 inputs.
 * @param {string} [userId]
 * @returns {Promise<string[]>} Array of base64-encoded PNG strings
 */
export async function edit(
  { images, prompt, size = "1024x1024", quality = "medium", n = 1 } = {},
  userId
) {
  const key = await loadKey(userId);
  if (!key) throw new Error("OPENAI_API_KEY not configured");
  if (!images?.length) throw new Error("At least one image is required for edit");

  const form = new FormData();
  form.append("model", imageModel());
  form.append("prompt", prompt);
  form.append("size", size);
  form.append("quality", quality);
  form.append("n", String(n));

  for (const img of images) {
    const blob =
      img instanceof Blob ? img : new Blob([img], { type: "image/png" });
    form.append("image[]", blob, "image.png");
  }

  const res = await fetch(`${apiBase()}/images/edits`, {
    method: "POST",
    headers: { Authorization: `Bearer ${key}` },
    body: form,
  });

  const json = await res.json();
  if (json.error) throw new Error(`OpenAI Images edit error: ${json.error.message}`);
  return json.data.map((d) => d.b64_json);
}

/**
 * Contextless image-vs-claim judge. Each call is its own zero-history request to
 * the chat-completions endpoint with a vision message. Used as a "second pair of
 * eyes" gate — the judging model has no narrative about the work, no pressure to
 * defend prior output, no risk of overanchoring on the agent's own description.
 * Strict JSON output for downstream gating.
 *
 * @param {{ imageUrl: string, claim: string, baselineUrl?: string, model?: string }} input
 *   imageUrl    — public URL of the ACTUAL image being verified (the worker's
 *                 deliverable). OpenAI server must be able to fetch it.
 *   claim       — what the image is supposed to show. THE canonical claim
 *                 (typically items.expectation verbatim).
 *   baselineUrl — OPTIONAL. The AI-approved design baseline image from the
 *                 design stage. When present, the judge ALSO checks that the
 *                 actual is ≥90% similar to the baseline OR contains all key
 *                 elements (layout, components, hierarchy). Both checks must
 *                 pass for a 'match' verdict. When absent, behavior is the
 *                 same as before — text-only claim check.
 *   model       — defaults to "gpt-4o" (full). gpt-4o-mini is BANNED for
 *                 verification (CLAUDE.md WWCD bans, 5f9b4c0): 49% false-alarm
 *                 rate on dense screenshots.
 * @param {string} [userId]
 * @returns {Promise<{ verdict: 'match'|'mismatch'|'inconclusive', confidence: number, reasoning: string, model: string, baseline_compared: boolean }>}
 */
export async function judge({ imageUrl, claim, baselineUrl, model = "gpt-4o" } = {}, userId) {
  if (typeof imageUrl !== "string" || !imageUrl.trim()) {
    throw new Error("judge: imageUrl is required");
  }
  if (typeof claim !== "string" || !claim.trim()) {
    throw new Error("judge: claim is required");
  }
  const hasBaseline = typeof baselineUrl === "string" && !!baselineUrl.trim();

  const key = await loadKey(userId);
  if (!key) throw new Error("OPENAI_API_KEY not configured");

  const systemPromptBase = [
    "You are a strict image verification judge.",
    "Your only job is to decide whether the deliverable image satisfies the claim.",
    "Be conservative: if you cannot confirm from the image(s) alone, return 'mismatch' or 'inconclusive'.",
    "Common failure modes to flag as mismatch:",
    "  - stock photo / generic image instead of the claimed subject",
    "  - blank / empty / loading state instead of populated content",
    "  - error page, 404, login screen, dev overlay instead of the claimed page",
    "  - placeholder text or skeleton loaders never resolved",
    "  - URL bar / page title clearly inconsistent with the claim",
  ];
  const systemPromptBaselineExtra = [
    "You will be given TWO images: the ACTUAL deliverable AND a BASELINE design that was AI-approved at the design stage.",
    "Pass requires BOTH:",
    "  (a) the actual image satisfies the claim text, AND",
    "  (b) the actual image is ≥90% visually similar to the baseline OR contains all key elements (layout, components, hierarchy, primary content blocks).",
    "Cosmetic differences (minor spacing, font weight) are tolerated. Missing whole sections, wrong layout, or different page entirely → mismatch.",
  ];
  const systemPromptOutput = [
    "Respond with VALID JSON ONLY, no markdown fences, no preamble:",
    '{ "verdict": "match" | "mismatch" | "inconclusive", "confidence": 0.0-1.0, "reasoning": "<one sentence>" }',
  ];
  const systemPrompt = (hasBaseline
    ? [...systemPromptBase, ...systemPromptBaselineExtra, ...systemPromptOutput]
    : [...systemPromptBase, ...systemPromptOutput]
  ).join("\n");

  const userMessage = hasBaseline
    ? [
        {
          type: "text",
          text: `Claim: ${claim}\n\nFirst image: ACTUAL deliverable.\nSecond image: BASELINE design (AI-approved reference).\nDoes the actual satisfy the claim AND match the baseline?`,
        },
        { type: "image_url", image_url: { url: imageUrl } },
        { type: "image_url", image_url: { url: baselineUrl } },
      ]
    : [
        { type: "text", text: `Claim: ${claim}\n\nDoes the image match this claim?` },
        { type: "image_url", image_url: { url: imageUrl } },
      ];

  const res = await fetch(`${apiBase()}/chat/completions`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${key}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model,
      messages: [
        { role: "system", content: systemPrompt },
        { role: "user", content: userMessage },
      ],
      temperature: 0,
      max_tokens: 200,
      response_format: { type: "json_object" },
    }),
  });

  const json = await res.json();
  if (json.error) throw new Error(`OpenAI judge error: ${json.error.message}`);
  const content = json?.choices?.[0]?.message?.content;
  if (typeof content !== "string") {
    throw new Error("OpenAI judge returned no content");
  }

  let parsed;
  try {
    parsed = JSON.parse(content);
  } catch {
    throw new Error(`OpenAI judge returned non-JSON content: ${content.slice(0, 200)}`);
  }

  const verdict =
    parsed.verdict === "match" || parsed.verdict === "mismatch" || parsed.verdict === "inconclusive"
      ? parsed.verdict
      : "inconclusive";
  const confidence =
    typeof parsed.confidence === "number" && parsed.confidence >= 0 && parsed.confidence <= 1
      ? parsed.confidence
      : 0;
  const reasoning = typeof parsed.reasoning === "string" ? parsed.reasoning : "";

  return { verdict, confidence, reasoning, model, baseline_compared: hasBaseline };
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/questExecution/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/weapon/questExecution` (and the gate weapons)

Rationale layer (per CLAUDE.md §6). The WHY behind the stage-transition gates
shared by `questExecution`, `questPurrview`, and `questReview` — not API docs, not
procedure. Append; supersede in place when a decision changes.

---

## 2026-06-13 — every stage transition is a lockphrase-bearing comment, written weapon-atomically with the stage write

**Decision.** A stage transition writes `quests.stage` AND inserts a
`quest_comments` row carrying a gate-specific **lockphrase**
(`SUBMIT_LOCKPHRASE` at `questExecution/index.js:40`, `APPROVE_LOCKPHRASE` at
`questPurrview/index.js:96`, `FINAL_GATE_PASS_LOCKPHRASE` at
`questReview/index.js:56`) together. The next consumer (`confirmSubmission`,
`confirmApproval`) **greps for the prior gate's lockphrase** before proceeding; it
never trusts a bare `.stage` value.

**Rejected alternative.** A simple stage-column check (`quest.stage === 'execute'`)
with no orthogonal audit comment.

**Why.** A bare stage write is invisible to auditing and forgeable by raw SQL or a
rogue call. The lockphrase comment IS the proof the gate ran: it makes every
transition tamper-evident, and couples the audit trail to the stage so a partial
transition (stage moved, comment missing) is detected and rejected by the next
gate rather than sailing through.

**Consequence if violated.** Rogue/partial stage writes bypass gates entirely — a
quest reaches review with unfinished objectives and the review gates become
theater (the leak class the operator repeatedly flags).

---

## 2026-06-13 — each gate stamps a `GATE_VERSION` into its audit comment

**Decision.** Every gate exports a `GATE_VERSION` constant and writes it to the
audit comment's `detail.gate_version` (`questExecution:34`, `questPurrview:86`,
`questReview:49`, `questDesign:28`). Quest comments are immutable; the version lets
a future reader know which gate logic a past approval passed.

**Rejected alternative.** Omit versioning (no forward tracking of gate changes).

**Why.** When a gate's criteria change, old quests carry comments from the old
gate. Without a version stamp, a past approval is ambiguous — did it clear the
current gate or a weaker predecessor? — making targeted re-validation impossible.

**Consequence if violated.** Gate-logic evolution becomes invisible in the trail;
re-validation after a gate change is guesswork.

---

## 2026-06-13 — visual verification is the reviewer's responsibility, NOT the worker's

**Decision.** The worker's `submit()` gate requires only a non-null deliverable
`ref` ("I made it exists"). The `verify_screenshot_ref` requirement lives
downstream at `questPurrview.approve()` ("it's real and satisfies the spec"), with
`autoVisualVerify()` capturing the screenshot for headless quests.

**Rejected alternative.** Symmetric gates (worker also blocks on a screenshot), or
the worker captures the screenshot itself.

**Why.** A headless worker leaf has no CIC / user-channel browser to capture a live
screenshot — only the CIC-capable reviewer (or local Claude at review) can. Putting
the screenshot gate on the worker would either block every headless quest or push
the worker to fake it; deferring it to purrview keeps the worker autonomous while
preserving a real visual gate.

**Consequence if violated.** Either headless workers stall on an unobtainable
screenshot, or unverified deliverables flow into review as "trusted" and the visual
gate goes toothless.

---

## 2026-06-13 — open unblocking questions at submit route to `escalated`, not `purrview`

**Decision.** If `submit()` finds any open unblocking question
(`qtype='unblocking', answered=false`), it routes the quest to `escalated` (with an
explanatory comment) instead of advancing to `purrview`.

**Rejected alternative.** Always `submit() → purrview` (gate only on objectives +
deliverables) and let the review path notice the questions later.

**Why.** A quest with unanswered user-facing questions is not "ready for review" —
it's waiting on the user. Parking it in purrview misrepresents its state;
escalating signals the user that input is needed now. (Companion to the
`questmaster` planner-side escalate ED — same principle at the submit boundary.)

**Consequence if violated.** Quests sit in purrview awaiting answers, breaking the
contract that purrview = "worker done, please review."
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/questExecution/index.js -->
```js
/**
 * questExecution weapon — worker-side quest stage transitions.
 *
 * Owns the WORKER's view: how an adventurer ships a quest from execute → purrview.
 * The agent CANNOT advance a quest by writing `stage` directly. The only
 * supported path is `submit({ questId })`, which refuses to move the quest unless
 * every gate passes.
 *
 * New steps/items model (post 20260515120000_steps_items_refactor):
 *   - readiness is the `steps` + `items` state, NOT the retired
 *     items.expectation / per-item item_comments / tier-check columns.
 *   - A quest is submittable when every step up to AND including the Checkpoint
 *     (the step whose view_results='Checkpoint') is status='done', and every
 *     asset those steps name in output[].asset exists as an `items` row with a
 *     non-null `ref`.
 *
 * Sister weapon: `questPurrview` owns the QUESTMASTER's view (confirmSubmission
 * for the read-side handoff check, approve/bounce for the write-side transitions
 * out of purrview).
 *
 * Auth: uses the service-role DB facade (`database.init("service")`).
 * No external API. Reads `steps`, `items`, `quests`; writes `quests`, `quest_comments`.
 */

import { database } from "@/libs/council/database";
import { publicTables } from "@/libs/council/publicTables";
import { getQuestDetails } from "@/libs/weapon/questmaster/index.js";

export const toc = {
  submit:
    "Move a quest from execute to purrview. Hard-gated on the new steps/items model: every step up to and including the Checkpoint is status='done', and the items those steps produce exist with a non-null ref. On success, leaves the lockphrase comment that questPurrview.confirmSubmission checks for.",
};

export const GATE_VERSION = 2;
/**
 * Stable phrase embedded in the success comment.
 * `questPurrview.confirmSubmission` greps for it verbatim.
 * Do not change without coordinating with every consumer.
 */
export const SUBMIT_LOCKPHRASE = "this quest now meets the criteria for purrview";

const CRITERIA_CHECKED = [
  "questId is a non-empty string",
  "quest exists",
  "quest.stage === 'execute'",
  "objectives rows exist for the quest",
  "every objective is status='done'",
  "every deliverable has a non-null ref",
  "stage write to purrview verified by SELECT-back",
];

/**
 * Submit a quest for Questmaster review.
 *
 * On full pass: writes `stage = "purrview"`, posts a `submit_for_purrview` audit
 * comment containing the lockphrase + full criteria list, returns
 * `{ ok: true, stage: "purrview", ... }`.
 *
 * On any failure: stage stays at `execute`, returns
 * `{ ok: false, failed: [<gate-id>], report: { msg, fix, ...details } }`.
 *
 * @param {{ questId: string }} args
 */
export async function submit({ questId }) {
  // Gate 1 — questId
  if (typeof questId !== "string" || !questId.trim()) {
    return {
      ok: false,
      failed: ["questId"],
      report: {
        msg: "questId is required (non-empty string).",
        fix: "Pass { questId: '<uuid>' } as the argument.",
      },
    };
  }

  const db = await database.init("service");

  // Gates 2 + 3 — read the quest (also pulls assignee so we can stamp
  // actor_name on the submit comment).
  const { data: quest, error: qErr } = await db
    .from(publicTables.quests)
    .select("id, stage, title, assignee")
    .eq("id", questId)
    .single();
  if (qErr || !quest) {
    return {
      ok: false,
      failed: ["quest_exists"],
      report: {
        msg: qErr?.message || "Quest not found.",
        fix: "Verify the questId. Confirm the quest exists with: SELECT id, stage FROM quests WHERE id = '<uuid>'.",
        questId,
      },
    };
  }
  if (quest.stage !== "execute") {
    return {
      ok: false,
      failed: ["stage"],
      report: {
        msg: `Quest stage is "${quest.stage}", expected "execute". Submission only valid from execute.`,
        fix:
          quest.stage === "purrview" || quest.stage === "review"
            ? "Quest is already past execute — no resubmission needed. If you have new artifacts, post them as comments and ping the Questmaster."
            : "If the quest is in closing/complete, contact the Guildmaster — submission only opens from execute.",
        current_stage: quest.stage,
      },
    };
  }

  // Gate 3b — open unblocking questions BLOCK submission → route to escalated.
  // If the agent has unanswered questions TO the user (qtype='unblocking',
  // answered=false), the quest is waiting on the user — it is NOT ready for
  // purrview/review. A quest can never pass review with the agent's own
  // questions unanswered, so we transition it to 'escalated' (awaiting the user)
  // instead of advancing. (operator rule 2026-06-13: "with N unblocking
  // questions unanswered, it should not progress to review but to escalated".)
  const { data: openUnblocking } = await db
    .from(publicTables.questQuestions)
    .select("id, body")
    .eq("quest_id", questId)
    .eq("qtype", "unblocking")
    .eq("answered", false);
  if (openUnblocking && openUnblocking.length > 0) {
    await db.from(publicTables.quests).update({ stage: "escalated" }).eq("id", questId);
    const { recordQuestComment } = await import("@/libs/quest/index.js");
    await recordQuestComment(questId, {
      source: "questExecution",
      action: "escalated_open_questions",
      summary: `Routed to escalated, not purrview: ${openUnblocking.length} open unblocking question(s) await the user. A quest cannot pass review while the agent's own questions to the user are unanswered.`,
      actor_name: quest.assignee || null,
      detail: { open_unblocking_count: openUnblocking.length, question_ids: openUnblocking.map((q) => q.id) },
    }, { client: db });
    return {
      ok: true,
      stage: "escalated",
      escalated: true,
      open_unblocking: openUnblocking.length,
      report: {
        msg: `${openUnblocking.length} open unblocking question(s) await the user — quest routed to escalated instead of purrview.`,
        fix: "The user must answer the unblocking questions (quest_questions, qtype='unblocking'). Once answered, the worker digests them and re-submits.",
      },
    };
  }

  // Gate 4 — objectives rows exist
  const { data: steps, error: sErr } = await db
    .from(publicTables.objectives)
    .select("id, n, name, status")
    .eq("quest_id", questId)
    .order("n", { ascending: true });
  if (sErr) {
    return {
      ok: false,
      failed: ["steps_read"],
      report: {
        msg: sErr.message,
        fix: "DB read against `steps` failed. Check service-role auth and table name (libs/council/publicTables.js: steps).",
      },
    };
  }
  const stepList = steps || [];
  if (stepList.length === 0) {
    return {
      ok: false,
      failed: ["no_steps"],
      report: {
        msg: "Quest has no steps rows. The materialized SMART plan is empty — there's nothing to execute or submit.",
        fix: "Steps are written at quest creation by questmaster.createQuest. If they're missing, the quest was created incorrectly — escalate to the Guildmaster.",
      },
    };
  }

  // Gate 5 — every objective is status='done'
  const unfinished = stepList.filter((s) => s.status !== "done");
  if (unfinished.length > 0) {
    return {
      ok: false,
      failed: ["steps_not_done"],
      report: {
        msg: `${unfinished.length} step(s) up to the Checkpoint are not status='done'.`,
        fix: `Finish each step via worker.executeStep until its status flips to 'done'. Unfinished (non-done) steps: ${unfinished
          .map((s) => `n${s.n} "${s.name}" (${s.status})`)
          .join(", ")}.`,
        unfinished_steps: unfinished.map((s) => ({ n: s.n, name: s.name, status: s.status })),
      },
    };
  }

  // Gate 6 — every deliverable has a non-null ref (the produced artifact).
  // Deliverables are the canonical list of what the quest must produce; the
  // reviewer grades them, not objective output[] (dropped in BCS migration).
  const details = await getQuestDetails({ questId });
  if (details?.error) {
    return {
      ok: false,
      failed: ["quest_unreadable_via_ui_path"],
      report: {
        msg: `getQuestDetails failed: ${details.error}. The reviewer's page would not render.`,
        fix: "Fix getQuestDetails / the read path so the quest page can load.",
      },
    };
  }
  const uiDeliverables = Array.isArray(details?.deliverables) ? details.deliverables : [];
  const missingRef = uiDeliverables.filter(
    (d) => d.ref == null || (typeof d.ref === "string" && d.ref.trim() === ""),
  );
  if (missingRef.length > 0) {
    return {
      ok: false,
      failed: ["deliverables_missing_ref"],
      report: {
        msg: `${missingRef.length} deliverable(s) have no ref (produced artifact): ${missingRef.map((d) => d.name).join(", ")}.`,
        fix: "Ship each deliverable via worker.shipItemArtifact / libs/quest writeDeliverable. UPSERT on (quest_id, name) replaces in place.",
        missing: missingRef.map((d) => d.name),
      },
    };
  }

  // NOTE: VISUAL VERIFICATION is the PURRVIEW agent's responsibility, NOT the
  // worker's — a headless worker leaf has no CIC/user-channel browser to capture
  // a verify screenshot. The submit gate therefore requires only a produced `ref`
  // (above). The verify_screenshot_ref requirement lives at the purrview→review
  // gate (questPurrview.approve), where the CIC-capable reviewer sets it. See
  // docs/quest-lifecycle.md (visual-verification responsibility).

  // All gates pass — write lockphrase comment FIRST (the DB trigger
  // enforce_stage_gate requires it before the stage write succeeds).
  const summary = `Gate v${GATE_VERSION} passed. ${stepList.length} objective(s) done, ${uiDeliverables.length} deliverable(s) verified. ${SUBMIT_LOCKPHRASE.charAt(0).toUpperCase() + SUBMIT_LOCKPHRASE.slice(1)}.`;
  const { recordQuestComment } = await import("@/libs/quest/index.js");
  // submit_for_purrview is a PASS_THROUGH gate action (the chokepoint always
  // inserts it fresh — the DB enforce_stage_gate trigger requires the lockphrase
  // comment before the stage write, and auditStageBypass times its freshness).
  const { error: commentErr } = await recordQuestComment(questId, {
    source: "questExecution",
    action: "submit_for_purrview",
    summary,
    actor_name: quest.assignee || null,
    detail: {
      gate_version: GATE_VERSION,
      lockphrase: SUBMIT_LOCKPHRASE,
      criteria_checked: CRITERIA_CHECKED,
      steps_count: stepList.length,
      assets: uiDeliverables.map((d) => d.name),
    },
  }, { client: db });
  if (commentErr) {
    return {
      ok: false,
      failed: ["lockphrase_comment_write"],
      report: {
        msg: `Lockphrase comment failed to write: ${commentErr.message}. Stage NOT advanced (comment-before-stage order).`,
        fix: `Debug the quest_comments insert. The DB trigger enforce_stage_gate blocks stage→purrview without this comment.`,
      },
    };
  }

  // Advance stage (the trigger allows it because the lockphrase comment exists).
  const { error: upErr } = await db
    .from(publicTables.quests)
    .update({ stage: "purrview" })
    .eq("id", questId);
  if (upErr) {
    return {
      ok: false,
      failed: ["stage_write"],
      report: {
        msg: upErr.message,
        fix: "Quest stage write failed at the DB layer. Check service-role auth or the enforce_stage_gate trigger — this is a system-level failure, not an agent fix.",
      },
    };
  }
  const { data: verify } = await db
    .from(publicTables.quests)
    .select("stage")
    .eq("id", questId)
    .single();
  if (verify?.stage !== "purrview") {
    return {
      ok: false,
      failed: ["stage_write_verify"],
      report: {
        msg: `Stage write did not persist; reads back as "${verify?.stage}".`,
        fix: "Stage write was silently rejected (likely RLS or the enforce_stage_gate trigger). Check that the lockphrase comment exists.",
      },
    };
  }

  return {
    ok: true,
    stage: "purrview",
    steps_count: stepList.length,
    assets_count: uiDeliverables.length,
    lockphrase: SUBMIT_LOCKPHRASE,
    criteria_checked: CRITERIA_CHECKED,
  };
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/questPurrview/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/weapon/questPurrview`

Rationale layer (per CLAUDE.md §6). The WHY behind purrview's visual-verification
gate. Append; supersede in place when a decision changes.

---

## 2026-07-03 — a PASS milestone verdict (`approve`, `final_gate_pass`) is written `resolved=true`

**Decision.** `questPurrview.approve` (and `questReview.final_gate_pass`, same
class) now writes its `quest_comments` row with `resolved: true`. **Rejected
alternative:** the prior behavior — inherit the column default `resolved=false`
"for audit." **Why:** an approval is a *terminal* pass verdict, not an open
to-do. Multiple `resolved=false` scans classify a comment as pending FEEDBACK by
its **source prefix** alone — `registry.js` rung-8 digest
(`unresolvedDigestibleCommentsByQuest`) and `caravan/index.js`
`resolveGateForward.hasFeedback` (`src.startsWith("questpurrview")` /
`.startsWith("review")`). An unresolved PASS row therefore reads back as
"unaddressed feedback," which either mis-routes a done quest or is only masked by
the milestone short-circuit firing first. Writing PASS rows `resolved=true` keeps
them out of those scans entirely (the verdict-based guard in the rung-8 counter
stays as defense-in-depth for legacy rows). **Safe because:** the approve /
final_gate_pass UPSERT keys on `(quest_id, action)` — NOT on `resolved` — and
`milestoneByQuest` does not filter `resolved`, so the milestone floor
(approved → review, final_gate_passed → closing) is unaffected. **Consequence if
violated:** revert to `resolved=false` and a PASS verdict re-enters the feedback
scans, resurfacing the approve-treated-as-feedback class of bug the rung-8
`isCatFeedback` (verdict==='fail') guard was added to kill.

---

## 2026-07-04 — `approve` flips each passed deliverable `active → done`

**Decision.** On a passing gate, `questPurrview.approve` now writes
`deliverables.status = 'done'` for every active deliverable it just verified,
right before advancing the stage to `review`. **Rejected alternative:** leaving
the pass verdict to live ONLY on the `quest_comments` approve row and never
touching `deliverables.status` (the prior behavior). **Why:** the rest of purrview
was already written *as if* passed deliverables become `done` — both
`preflightStageTransition` and `autoVisualVerify` filter to `status==='active'`
with the comment "a non-active row (e.g. status 'done') is out of scope — it needs
no verdict and no verification link." But nothing in the pipeline ever set `done`,
so that filter was dead: a passed asset stayed `active` forever, got re-verified
from scratch on any re-entry to purrview, and showed perpetually active on the
board (observed on quest `c73b68a9` — infographic judged `match`, approved, still
`active`). Objectives get lifecycle transitions (`questmaster` sets
`running`/`done`); deliverables had none. **Consequence if violated:** the
active-only filter stays a no-op — passed deliverables re-enter every future
purrview verdict count, and the UI never shows a deliverable as finished.

---

## 2026-06-13 — the judge claim must carry the quest's acceptance criteria, not just the deliverable name

**Decision.** Both judge paths (`runReviewLoop` for images, `approve` Gate 3 for
every deliverable) build the vision-judge claim via `buildJudgeClaim()`, which
folds the quest **title + description** into the claim ("must satisfy these
requirements; judge mismatch on wrong style/language/missing-or-forbidden
elements"). `confirmSubmission` returns `title`/`description` so the claim source
is available.

**Rejected alternative.** Claim = `deliverable.name` (the post-migration default)
or a generic "is this a real, non-blank render".

**Why.** A contextless judge told only the name verifies that *an artifact
exists*, not that it *satisfies this brief*. A correctly-rendered but WRONG
deliverable (wrong art style, wrong language, violence present, missing the
required story mutation) sails through. The acceptance criteria must reach the
judge or the gate is theater.

**Consequence if violated.** Purrview "passes" wrong work — the exact leak the
operator flagged ("purrview has been rather leaky"). Reverting to a name-only
claim silently reopens it.

---

## 2026-06-13 — document deliverables are rendered in FULL (all pages), not page 1

**Decision.** `renderDeliverableScreenshot` loops every PDF page (capped 15)
stacked into one tall screenshot.

**Rejected alternative.** `pdf.getPage(1)` — render only page 1.

**Why.** The deliverable here is a multi-chapter book (`kind=document`). Verifying
it by page 1 means chapters 2..N are never seen by the gate — a blind spot that
passes a book with a fine cover and broken interior.

**Consequence if violated.** Multi-page documents are "verified" by their cover.

---

## 2026-06-13 — auto visual-verify fails CLOSED on render error

**Decision.** `renderDeliverableScreenshot` throws when navigation errors AND the
page rendered empty, instead of screenshotting and uploading a blank/partial page.

**Rejected alternative.** Swallow nav failures (`.catch(()=>{})`) and screenshot
whatever is there.

**Why.** A blank/error screenshot that gets uploaded can satisfy a weak gate; a
thrown error leaves `verify_screenshot_ref` unset, so `approve` cannot pass. Fail
closed, not open, on a verification surface.

**Consequence if violated.** A never-loaded deliverable self-certifies via an
empty screenshot.

---

## 2026-06-15 — buildJudgeClaim must distinguish per-deliverable roles in multi-deliverable quests

**Decision.** `buildJudgeClaim` now tells the vision judge that this image is ONE
specific deliverable (by name), and that a quest may produce multiple deliverables
with different roles. The judge evaluates content against the deliverable's stated
role within the quest description, not against the entire quest scope.

**Rejected alternative.** Feed the full quest description without per-deliverable
framing (the pre-fix default).

**Why.** Multi-deliverable quests produce artifacts with distinct purposes (e.g. a
proposal document + a customer follow-up email). The vision judge, given only the
full quest description, rejects supplementary deliverables because they "don't
match the quest requirements" — it compares an email against the full proposal
spec. This caused 100% false-negative on the email deliverable for quest
1613dd7d (Antibody Continuity Program) despite the email being correct and
complete for its role.

**Consequence if violated.** Supplementary deliverables in multi-deliverable quests
will always fail the vision gate, forcing manual workarounds or single-deliverable
quest designs.

---

## 2026-06-15 — approve/bounce write ONE structured verdict comment, not one row per asset

**Decision.** `approve` and `bounce` no longer call a `writeVerdictComments` helper
that inserted one `action='asset_verdict'` `quest_comments` row per deliverable.
The full per-asset verdict set rides in the SINGLE lockphrase comment's
`detail.perItemVerdicts` (machine-readable — what `countPriorBounces` reads) plus a
compact roll-up in its `summary`, and the row now carries the top-level `verdict`
column (`'pass'` for approve, `'fail'` for bounce). `writeVerdictComments` was
deleted. `countPriorBounces` keeps `'asset_verdict'` in its action filter so legacy
rows still read.

**Rejected alternative.** Keep emitting a separate per-asset verdict row alongside
the lockphrase comment (the pre-fix shape, and what an agent re-derives by default —
"one audit row per asset").

**Why.** The lockphrase comment ALREADY carried the full `perItemVerdicts` array, so
the per-asset rows were pure duplication — N extra rows per pass on top of the
lockphrase, the weapon-side half of the verify comment-bloat (#2 source; see the
questmaster `recordVerifyPass` ED). Collapsing also removes a latent double-count:
each prior bounce previously contributed twice to `countPriorBounces` (once via its
`asset_verdict` row, once via the bounce row's `perItemVerdicts`), so a prior bounce
now counts exactly ONCE and the `>= 2` chronic threshold finally means "3rd bounce"
as its comment claims. The lockphrase string MUST remain in `summary` — the DB
trigger `enforce_stage_gate` greps it before allowing the forward stage write.

**Consequence if violated.** Reintroduce `writeVerdictComments` and the per-pass row
count jumps back to N+1, re-flooding the thread; the chronic-bounce counter silently
double-counts again. Drop the lockphrase from `summary` and the stage transition is
blocked at the DB layer.

---

## 2026-07-03 — pre-submit render-sanity is ADVISE-MODE, not a gate (SIO #1)

**Decision.** `adviseRenderSanity` runs a set of CHEAP HTTP-fetch heuristics over
each deliverable ref (localhost/loopback ref, unreachable/HTTP-error, HTML served
with the wrong content-type or as an attachment, escaped-source body, blank body,
restrictive CSP) and ANNOTATES the thread with fixable-error notes — but it NEVER
changes a verdict. It is called from `confirmSubmission` in a swallowed try/catch,
so a slow fetch or a thrown error inside it can never block the submit gate. It
writes one **upserted** `advise_render_check` comment plus a `render_advise`
telemetry row (`would_have_bounced`) on the caravan log channel for hit-rate.

**Rejected alternative.** Make it a HARD gate that blocks submit when a heuristic
fires (the "obvious" way to stop bad deliverables reaching purrview). Also rejected:
do a full headless RASTER (screenshot + non-blank check) at submit time.

**Why.** The operator's de-risked mandate was advise-FIRST: a heuristic is a
proxy, and a false positive that *blocks* submit is strictly worse than the 2-3x
bounce it replaces — it would strand correct deliverables with no override. Advise
mode gets the fixable notes onto the thread with zero blast radius; the telemetry
counter lets the operator MEASURE the true would-have-caught rate before deciding
whether to promote advise→gate. Raster is deferred because the blank-render case is
already caught downstream by `autoVisualVerify` (fail-closed-on-blank) + the vision
judge — a cheap fetch adds the checks the current pipeline only discovers AFTER an
expensive render→judge→bounce (localhost refs, wrong content-type, escaped source),
which is exactly the friction in quests ac48ab69 / 7ec94b9d / 3f5860e4 / 4065db7e.
Render-shape checks (content-type/escaped/blank/CSP) are gated on `kind==='html'`
because a PNG served as image/png or a PDF as application/pdf is correct, not a
defect. The upsert (see `advise_render_check` in `libs/quest` `UPSERT_ACTIONS`)
keeps the advisory from itself becoming per-pass comment bloat.

**Consequence if violated.** Turn advise into a gate before the telemetry justifies
it and a false-positive heuristic hard-blocks a correct deliverable with no operator
visibility — reintroducing the very stall the SIO set out to remove. Emit the
advisory as an appended (non-upserted) comment and the render-advise annotation
re-floods the thread it was meant to keep clean.

## 2026-07-03 — Purrview Verification is polymorphic by deliverable Type; text kinds verify by reading, not by screenshot

**Decision.** The `verify_screenshot_ref` column is now the **"Purrview Verification"**
value and is polymorphic by the deliverable's `kind`:
- **VISUAL** kinds (html/image/document/video) → a Supabase Storage **screenshot URL**,
  vision-judged by `openai_images.judge` (Gate 3, unchanged).
- **TEXT** kind → a **`PASS …`/`FAIL …` text verdict**. The reviewer READS the content
  and states the outcome; `PASS` passes, anything else bounces. No screenshot, no
  vision-judge. The three writers that validated the field as a Supabase URL
  (`/api/quest/mutate`, `questMutator`, `libs/quest.writeDeliverable`) now also accept
  a `^(PASS|FAIL)\b` verdict. The `approve` Gate 3 branches on the value shape.
Purrview also now scopes to **ACTIVE** deliverables only (`status ?? 'active'`).

**Rejected alternatives.** (a) Force a screenshot for text too — a headless render of
"72" or a paragraph is verification theater and, when the render channel was
auth-gated, produced a *login-screen* screenshot that the judge (correctly) failed,
escalating dozens of finished text quests. (b) Add a separate `verify_text_verdict`
column — needless migration + a second nullable field the UI and every gate must
special-case; the field is already "the proof this deliverable is verified", so keep
it single and polymorphic. (c) Rename the DB column to `purrview_verification` — a
cross-tree rename for a label change; the user-facing concept is relabeled in the UI
while the column name stays for compatibility.

**Why.** A simple text answer is verified by reading it, not photographing it. Coupling
ALL verification to a screenshot made the whole gate hostage to the browser/auth
channel — the "cat disabled during purrview → image shows a login screen" false-fail
class that stranded finished quests in `escalated`.

**Consequence if violated.** Re-requiring a screenshot for text kinds (or re-tightening
any of the three validators back to URL-only) reopens the login-screen false-fail and
re-strands text quests; dropping the active-only scope makes purrview re-verify
archived/superseded rows.

---

### 2026-07-03 · Judge on deliverable KIND, not just the verification VALUE SHAPE (residual login-screen hole)

**Decision.** The purrview verify gate now branches on `deliverable.kind` (via
`isTextKind`), not only on the shape of `verify_screenshot_ref`. For a text-like kind
(text/json/markdown/csv): (a) the approve Gate 3 REFUSES to vision-judge a screenshot
value — it demands a `PASS/FAIL` read verdict; and (b) `preflightStageTransition`
(shared by approve+bounce) REJECTS any per-item verdict on a text deliverable that
carries an `openai_images.judge*` origin (`isImageJudgeOrigin`). The questmaster
review-leaf prompt (buildLeafReviewPrompt 3b/3c) was also corrected: it had falsely
claimed `autoVisualVerify` renders text to "a full-page browser shot" — the code
SKIPS text; the prompt led leaves to screenshot + image-judge text anyway.

**Rejected alternative.** Rely on the value-shape branch alone (the prior fix). That
only helps when a text deliverable's verification is ALREADY a `PASS/FAIL` string.
Nothing stopped a stray render PNG from being SET as a text deliverable's
`verify_screenshot_ref` (or a leaf computing an image-judge verdict directly and
calling bounce) — then `classifyPurrviewVerification` returns `screenshot` and the
image judge false-fails it. Quest fa1accaf ("compute 8×9", deliverable "72", kind
text) was approved 18:46 then re-worked and bounced 19:47 as "shows a login screen,
not 72", then escalated — the exact residual.

**Why.** The KIND is the source of truth for HOW a deliverable is verified; the
value shape is downstream and forgeable. Gating on shape lets a mis-typed value route
text through the pixel judge.

**Consequence if violated.** Reverting to value-shape-only judging (or dropping the
text_deliverable_image_judged guard) reopens the false login-screen bounce for text
deliverables that carry a screenshot value, re-stranding finished text quests in
execute→escalated after a spurious purrview kickback.

---

## 2026-07-08 — `bounce` targets `plan`, not `execute` (plan = digest)

**Decision.** `bounce` writes `stage='plan'` (was `execute`), and `BOUNCE_LOCKPHRASE`
became "this quest has been bounced back to plan". The caravan purrview VERIFY leaf
likewise routes a mismatch through the new `questmaster.moveQuestToPlan` (not
`moveQuestToExecute`).

**Rejected alternative.** Keep `bounce → execute` (the native/historical target). A
bounce "feels like" sending the worker back to redo production. Rejected per operator
directive 2026-07-08 (plan = digest): the fail feedback is a digestible input that must
be reconciled into the objectives/deliverables at the plan altitude BEFORE re-production.

**Why.** The lifecycle ladder now maps a quest carrying unresolved cat-fail feedback to
`plan` (see `libs/caravan/engineering_decisions.md` 2026-07-08). Writing `execute`
directly only created a one-tick disagreement that the derail rescuer immediately
corrected back to `plan` — an extra hop and a misleading "back to execute" audit trail.
Writing `plan` directly matches the ladder. The transition is DB-safe: the
`enforce_stage_gate` trigger gates only FORWARD moves (→purrview/review/closing); →plan
is ungated.

**Consequence if violated.** Re-target `execute` and every bounce incurs a redundant
derail-kickback hop (execute→plan) and the "bounced back to execute" wording contradicts
where the quest actually lands, confusing both the worker and the audit log.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/questPurrview/index.js -->
```js
/**
 * questPurrview weapon — Questmaster-side quest stage transitions.
 *
 * Owns the QUESTMASTER's view: how Cat verifies a worker handoff and decides
 * whether the quest goes forward (purrview → review) or back (purrview → plan).
 * A bounce lands at plan, not execute, because plan == digest (operator directive
 * 2026-07-08): the fail feedback is a digestible input the plan leaf reconciles
 * into the objectives/deliverables before the quest re-executes.
 * Sister weapon: `questExecution` owns the worker's `submit` path.
 *
 * New objectives/deliverables model (post 20260515120000_steps_items_refactor):
 *   - deliverables are pure assets (name, kind, ref) — there is no deliverables.expectation,
 *     deliverables.url, deliverables.caption, or per-deliverable tier-check column anymore.
 *   - the dropped item_comments table is gone; per-asset verdicts and the
 *     bounce-loop history live on quest_comments (with detail jsonb).
 *   - per-asset verdicts key on the asset NAME (deliverables.name), the same key the
 *     objectives' output[].asset uses.
 *
 * Three actions, all script-locked:
 *   - `confirmSubmission` — verify the worker's `submit` lockphrase before opening
 *                           any artifact. No phrase, no review.
 *   - `approve`           — purrview → review. Requires per-asset Cat verdicts, all
 *                           must be 'pass', each carrying judge_origin. Writes the
 *                           next-stage lockphrase.
 *   - `bounce`            — purrview → plan (plan = digest). Requires per-asset Cat verdicts with
 *                           ≥1 'fail' and a reason. Writes the bounce lockphrase.
 *
 * Auth: uses the service-role DB facade (`database.init("service")`).
 * No external API. Reads `deliverables`, `objectives`, `quests`; writes `quests`, `quest_comments`.
 */

import { database } from "@/libs/council/database";
import { publicTables } from "@/libs/council/publicTables";
import { SUBMIT_LOCKPHRASE } from "@/libs/weapon/questExecution";
import { judge } from "@/libs/weapon/openai_images/index.js";
import { writeFile, readPublicUrl } from "@/libs/weapon/supabase_storage/index.js";
import { screenshotUrl } from "@/libs/weapon/leafBrowser/index.js";
// NOTE: recordQuestComment (@/libs/quest) and recordQuestEventLog
// (@/libs/caravan/logs) are pulled via DYNAMIC import inside adviseRenderSanity,
// matching the codebase convention (questmaster.recordVerifyPass, the quest
// chokepoint's telemetry redirect) that avoids the weapon↔quest↔caravan cycle.

export const toc = {
  confirmSubmission:
    "Verify a quest reached purrview through questExecution.submit (lockphrase comment present). Run BEFORE opening any artifact — no comment, no review. Returns the produced deliverables (name/kind/ref) + the objective outputs to grade against. Also fires adviseRenderSanity (SIO #1, advisory only — never blocks).",
  adviseRenderSanity:
    "ADVISORY pre-submit render-sanity check (SIO #1, advise-mode — NEVER gates/blocks). Cheap HTTP-fetch heuristics over each deliverable ref that catch the common render failures BEFORE the expensive render→vision-judge→bounce cycle: localhost/unreachable refs, HTML served with the wrong content-type (renders as raw code), escaped-source pages, blank bodies, and page-level CSP. Writes ONE upserted 'advise_render_check' comment with fixable-error notes + a queryable 'render_advise' telemetry row (would_have_bounced) so the operator can measure hit-rate and later decide whether to promote advise→gate. Returns { ok, would_have_bounced, findings }.",
  approve:
    "Move a quest from purrview to review. Verifies ONLY ACTIVE deliverables. Hard-gated: submit lockphrase + per-asset verdicts (all 'pass', each carrying judge_origin) + EVERY active deliverable filled (ref present) + EVERY active deliverable carries a Purrview Verification (verify_screenshot_ref). The verification BRANCHES on deliverable Type: VISUAL kinds (html/image/document/video) need a Supabase screenshot URL ACTUALLY vision-judged (openai_images.judge) as a real render — blank/404/login-screen/skeleton/unrelated shots are rejected; TEXT kind needs a `PASS …`/`FAIL …` verdict (reviewer read the content) — PASS passes, FAIL bounces, no screenshot required. Writes per-asset verdict comments + the review-handoff lockphrase.",
  bounce:
    "Move a quest from purrview back to plan (plan = digest: the fail feedback is re-planned into the objectives/deliverables before re-execution). Hard-gated: requires submit lockphrase + per-asset verdicts with ≥1 'fail' + a reason + each verdict carries judge_origin. Writes per-asset verdict comments + the bounce lockphrase.",
  autoVisualVerify:
    "Headless visual verification — fills verify_screenshot_ref for deliverables that lack it WITHOUT a CIC/browser session. Renders each deliverable's ref to a PNG (pdf.js for documents, Playwright navigate+full-page for html/image/text), uploads it to GuildOS_Bucket, and sets verify_screenshot_ref. This is what lets the headless cron review leaf satisfy the approve gate's visual-verification check autonomously (no human CIC needed). Returns { captured, skipped, errors }.",
};

/** Vision-judge origins accepted by the approve/bounce gates. Cat MUST run a
 *  judge before posting verdicts — the layer-attribution finding (5f9b4c0)
 *  showed Cat's standalone visual judgment has 0/2 catch rate. */
const VALID_JUDGE_ORIGINS = new Set([
  "openai_images.judge",
  "openai_images.judge:gpt-4o",
  "claude-multimodal-read", // local-Claude direct vision Read (T3.5)
]);

/**
 * Build the vision-judge claim for one deliverable. The claim is the ONLY thing
 * the contextless judge knows about what "correct" means — so it must carry the
 * quest's acceptance criteria, not just the deliverable name. Feeding the quest
 * description in is what turns the gate from "an artifact exists" into "the
 * artifact satisfies THIS brief" (closes the content-blind leak). Capped so a
 * huge description doesn't blow the vision prompt.
 */
export function buildJudgeClaim({ deliverableName, kind, questTitle, questDescription }) {
  const brief = String(questDescription || "").trim().slice(0, 1800);
  const lines = [
    `This image is the rendered deliverable "${deliverableName}"${kind ? ` (type: ${kind})` : ""}${questTitle ? ` for the quest "${questTitle}"` : ""}.`,
    `IMPORTANT: This is ONE specific deliverable of the quest, identified by its name "${deliverableName}". The quest may produce MULTIPLE deliverables with different roles. Judge this image ONLY against what the quest description says about THIS deliverable's role — do NOT reject it because it doesn't cover the entire quest scope.`,
    `It must show real, substantive rendered content — NOT a blank/empty page, NOT a 404/error/login screen, NOT an unresolved loading skeleton, NOT obviously unrelated content.`,
  ];
  if (brief) {
    lines.push(
      `AND it must actually satisfy its role within the quest's requirements below. Judge 'mismatch' ONLY if the content violates or omits a hard requirement SPECIFIC TO THIS DELIVERABLE's role (e.g. wrong art style, wrong language, missing/forbidden story elements, age-inappropriate content). A supplementary deliverable (like a follow-up email) that correctly serves its stated purpose is a 'match' even if it doesn't contain the main deliverable's content:`,
      brief,
    );
  }
  return lines.join("\n");
}

/**
 * Classify a "Purrview Verification" value (the polymorphic verify_screenshot_ref)
 * by its shape — the single source of truth the approve Gate 3 + the UI branch on:
 *   - "screenshot"  : a http(s) URL — a VISUAL kind's rendered proof (vision-judged).
 *   - "text_pass"   : a `PASS …` verdict — a TEXT kind read + accepted by the reviewer.
 *   - "text_fail"   : a `FAIL …` verdict — a TEXT kind read + rejected.
 *   - "empty"       : null/blank — not verified yet.
 *   - "invalid"     : a non-empty value that is neither a URL nor a PASS/FAIL verdict.
 * @param {string|null|undefined} value
 * @returns {"screenshot"|"text_pass"|"text_fail"|"empty"|"invalid"}
 */
export function classifyPurrviewVerification(value) {
  const s = typeof value === "string" ? value.trim() : "";
  if (!s) return "empty";
  if (/^PASS\b/i.test(s)) return "text_pass";
  if (/^FAIL\b/i.test(s)) return "text_fail";
  if (/^https?:\/\//i.test(s)) return "screenshot";
  return "invalid";
}

export function isValidJudgeOrigin(origin) {
  if (typeof origin !== "string" || !origin.trim()) return false;
  if (VALID_JUDGE_ORIGINS.has(origin)) return true;
  // Tolerate openai_images.judge:<model> for any model the caller picks.
  if (origin.startsWith("openai_images.judge:")) return true;
  return false;
}

/** Is this a NON-VISUAL deliverable kind — verified by READING the content, never
 *  by rendering it to a PNG and vision-judging it? A headless render of raw text
 *  lands on a blank/login page, which the image judge false-fails. These kinds
 *  must carry a read-based verdict (classifyPurrviewVerification → text_pass/fail,
 *  judge_origin='claude-multimodal-read'), NOT an image judge. (ED 2026-07-03.) */
export function isTextKind(kind) {
  const k = String(kind || "text").toLowerCase();
  return k === "text" || k === "json" || k === "markdown" || k === "md" || k === "csv";
}

/** True for judge_origins produced by the IMAGE judge (vision over a screenshot).
 *  A text-kind deliverable must never carry one — see isTextKind. */
export function isImageJudgeOrigin(origin) {
  return typeof origin === "string" && origin.startsWith("openai_images.judge");
}

export const GATE_VERSION = 2;

/** Default actor name for Cat-side comments. Override via the caller's input
 *  if multi-questmaster setups exist down the road. */
const QUESTMASTER_ACTOR_DEFAULT = "Cat";

/** Worker → Questmaster handoff lockphrase (set by questExecution.submit). */
// (imported above as SUBMIT_LOCKPHRASE)

/** Questmaster → user/GM-desk handoff lockphrase (set by approve). */
export const APPROVE_LOCKPHRASE = "this quest now meets the criteria for review";

/** Questmaster → worker handback lockphrase (set by bounce). */
export const BOUNCE_LOCKPHRASE = "this quest has been bounced back to plan";

const APPROVE_CRITERIA_CHECKED = [
  "questId is a non-empty string",
  "quest exists",
  "quest.stage === 'purrview'",
  "questExecution.submit lockphrase comment present (worker handoff verified)",
  "perItemVerdicts is a non-empty array",
  "perItemVerdicts.length === produced-deliverables count and every asset name matched",
  "every verdict === 'pass'",
  "every verdict carries a recognized judge_origin (openai_images.judge[:model] or claude-multimodal-read)",
  "every deliverable is filled (ref present)",
  "every deliverable carries a verify_screenshot_ref",
  "each verify_screenshot_ref vision-judged 'match' via openai_images.judge (not blank/404/skeleton/unrelated) — actual visual verification, not presence",
  "ONE structured verdict comment written carrying all per-asset verdicts in detail + top-level verdict='pass' (source='questPurrview', action='approve')",
  "every passed active deliverable flipped status active → done",
  "stage write to review verified by SELECT-back",
];

const BOUNCE_CRITERIA_CHECKED = [
  "questId is a non-empty string",
  "quest exists",
  "quest.stage === 'purrview'",
  "questExecution.submit lockphrase comment present (worker handoff verified)",
  "perItemVerdicts is a non-empty array",
  "perItemVerdicts.length === produced-deliverables count and every asset name matched",
  "≥1 verdict === 'fail'",
  "every verdict carries a recognized judge_origin",
  "reason is a non-empty string",
  "ONE structured verdict comment written carrying all per-asset verdicts in detail + top-level verdict='fail' (source='questPurrview', action='bounce')",
  "stage write back to plan verified by SELECT-back (plan = digest: the bounce feedback is re-planned before re-execution)",
];

// ---------------------------------------------------------------------------
// confirmSubmission — read-side gate. Questmaster runs this BEFORE reviewing.
// ---------------------------------------------------------------------------

/**
 * Verify the quest reached `purrview` through `questExecution.submit`
 * (not a bare stage write). Looks for the SUBMIT_LOCKPHRASE comment.
 *
 * Questmaster contract: run this BEFORE opening any artifact. If it returns
 * ok:false, the quest didn't pass through the gate — bounce it back to
 * execute or escalate, do NOT start review.
 *
 * @param {{ questId: string }} args
 */
export async function confirmSubmission({ questId }) {
  if (typeof questId !== "string" || !questId.trim()) {
    return { ok: false, reason: "questId_required", msg: "questId is required (non-empty string)." };
  }

  const db = await database.init("service");

  const { data: quest, error: qErr } = await db
    .from(publicTables.quests)
    .select("id, stage, title, description")
    .eq("id", questId)
    .single();
  if (qErr || !quest) {
    return { ok: false, reason: "quest_not_found", msg: qErr?.message || "Quest not found." };
  }
  if (quest.stage !== "purrview") {
    return {
      ok: false,
      reason: "wrong_stage",
      msg: `Quest stage is "${quest.stage}", expected "purrview". Quest is not awaiting Questmaster review.`,
      current_stage: quest.stage,
    };
  }

  const { data: gateComments, error: cErr } = await db
    .from(publicTables.questComments)
    .select("id, source, action, summary, detail, created_at")
    .eq("quest_id", questId)
    .eq("source", "questExecution")
    .eq("action", "submit_for_purrview")
    .order("created_at", { ascending: false })
    .limit(1);
  if (cErr) {
    return { ok: false, reason: "comments_read_failed", msg: cErr.message };
  }
  const gate = gateComments?.[0];
  if (!gate) {
    return {
      ok: false,
      reason: "no_gate_comment",
      msg: "No questExecution.submit_for_purrview comment found. Quest reached purrview without passing through the gate. Bounce back to execute and require submit().",
    };
  }

  const summary = String(gate.summary || "");
  const detailLock =
    gate.detail && typeof gate.detail === "object" ? String(gate.detail.lockphrase || "") : "";
  const summaryHasPhrase = summary.toLowerCase().includes(SUBMIT_LOCKPHRASE.toLowerCase());
  const detailHasPhrase = detailLock.toLowerCase() === SUBMIT_LOCKPHRASE.toLowerCase();
  if (!summaryHasPhrase && !detailHasPhrase) {
    return {
      ok: false,
      reason: "lockphrase_missing",
      msg: `Gate comment exists but lockphrase "${SUBMIT_LOCKPHRASE}" is absent from both summary and detail.lockphrase. Treating as untrusted.`,
      gate_comment_id: gate.id,
    };
  }

  // Under the new model the produced assets ARE the deliverables. Return the
  // deliverables rows (name/kind/ref) Cat grades, plus the objectives' output
  // specs so Cat can judge each asset against its declared verification.
  // Per-asset verdicts go back through approve/bounce keyed on deliverables.name.
  const { data: deliverables } = await db
    .from(publicTables.deliverables)
    .select("id, name, kind, ref")
    .eq("quest_id", questId);
  const { data: objectives } = await db
    .from(publicTables.objectives)
    .select("n, name, status")
    .eq("quest_id", questId)
    .order("n", { ascending: true });

  // SIO #1 — pre-submit render-sanity, ADVISE MODE. Fire the cheap heuristic
  // pass now (the deliverables are already loaded) so the fixable-error notes
  // are on the thread BEFORE Cat opens the expensive render→judge cycle. This is
  // ADVISORY ONLY: it never changes confirmSubmission's verdict, so a failure or
  // slow fetch inside it can never block the gate. Fully swallowed.
  let renderAdvice = null;
  try {
    renderAdvice = await adviseRenderSanity({ questId, deliverables, db });
  } catch {
    /* advisory — never blocks submit */
  }

  return {
    ok: true,
    msg: "Submission gate verified. Safe to begin review.",
    render_advice: renderAdvice
      ? { would_have_bounced: renderAdvice.would_have_bounced, flagged: renderAdvice.findings?.length || 0 }
      : null,
    gate_version: gate.detail?.gate_version ?? null,
    // title + description are the acceptance-criteria SOURCE the reviewer feeds
    // into the vision judge — a judge told only the deliverable name verifies
    // "an artifact exists", not "it satisfies THIS quest's brief" (the leak).
    title: quest.title || "",
    description: quest.description || "",
    deliverables_count: deliverables?.length ?? 0,
    deliverables: deliverables || [],
    objectives: objectives || [],
    submitted_at: gate.created_at,
  };
}

// ---------------------------------------------------------------------------
// approve — purrview → review with per-asset Cat verdicts (all pass)
// ---------------------------------------------------------------------------

/**
 * @param {{ questId: string, perItemVerdicts: Array<{item_key: string, verdict: 'pass'|'fail', text: string, judge_origin: string, judge_rationale?: string, judge_model?: string}>, summary?: string, actor_name?: string }} args
 *   `item_key` is the asset NAME (deliverables.name) under the new model.
 */
export async function approve({ questId, perItemVerdicts, summary: approvalSummary, actor_name = QUESTMASTER_ACTOR_DEFAULT }) {
  const pre = await preflightStageTransition({ questId, perItemVerdicts, requiredStage: "purrview" });
  if (!pre.ok) return pre.failure;
  const { db, deliverables, normalizedVerdicts, description: questDescription, title: questTitle } = pre;

  // Approve-specific gate: every verdict === 'pass'
  const fails = normalizedVerdicts.filter((v) => v.verdict !== "pass");
  if (fails.length > 0) {
    return {
      ok: false,
      failed: ["verdict_not_all_pass"],
      report: {
        msg: `${fails.length} verdict(s) are not 'pass'. Approve requires every asset to pass.`,
        fix: "Use bounce({ questId, perItemVerdicts, reason }) to send the quest back to execute with the failure feedback.",
        failing_item_keys: fails.map((v) => v.item_key),
      },
    };
  }

  // ── Gate 1: every deliverable must be FILLED (produced) — a ref is present. ──
  // The deliverable itself must exist before there is anything to visually verify.
  const missingRef = deliverables.filter(
    (d) => d.ref == null || (typeof d.ref === "string" && d.ref.trim() === ""),
  );
  if (missingRef.length > 0) {
    return {
      ok: false,
      failed: ["deliverables_not_filled"],
      report: {
        msg: `${missingRef.length} deliverable(s) are not filled (no ref / not produced): ${missingRef.map((d) => d.name).join(", ")}. Every deliverable must be produced before review.`,
        fix: "Produce each deliverable and set deliverables.ref (the worker bounces back to execute if it isn't done). Then re-run approve.",
        missing: missingRef.map((d) => d.name),
      },
    };
  }

  // ── Gate 2: every deliverable must carry a Purrview Verification value
  //    (verify_screenshot_ref) — EITHER a screenshot (visual kinds) OR a PASS/FAIL
  //    text verdict (text kind). Presence precondition for the actual check below. ──
  const missingVerify = deliverables.filter(
    (d) => d.verify_screenshot_ref == null || (typeof d.verify_screenshot_ref === "string" && d.verify_screenshot_ref.trim() === ""),
  );
  if (missingVerify.length > 0) {
    return {
      ok: false,
      failed: ["deliverables_missing_visual_verification"],
      report: {
        msg: `${missingVerify.length} deliverable(s) have no Purrview Verification (verify_screenshot_ref): ${missingVerify.map((d) => d.name).join(", ")}. As the purrview reviewer you must verify each ACTIVE deliverable and record the result before approving.`,
        fix: "For a VISUAL deliverable (html/image/document/video): open the rendered result, capture a screenshot, upload it to Supabase Storage, and set verify_screenshot_ref to the public URL. For a TEXT deliverable: read the content, and set verify_screenshot_ref to a `PASS — <why>` or `FAIL — <why>` verdict. Then re-run approve.",
        missing: missingVerify.map((d) => d.name),
      },
    };
  }

  // ── Gate 3: ACTUALLY verify — don't just trust a value is present. The check
  //    BRANCHES on the deliverable's verification TYPE (the value shape):
  //      • screenshot (Supabase URL)  → vision-judge it (openai_images.judge). A
  //        blank / 404-or-error / login-screen / loading-skeleton / unrelated shot
  //        must NOT pass — the judge looks at the real pixels.
  //      • text verdict (`PASS …`/`FAIL …`) → the reviewer read the text content and
  //        stated the outcome. `PASS` passes; `FAIL` (or anything not-PASS) fails.
  //    This is what lets a simple TEXT deliverable pass without a screenshot (the
  //    login-screen false-fail class), while keeping the pixel gate for renders. ──
  const visualFails = [];
  for (const d of deliverables) {
    const val = String(d.verify_screenshot_ref).trim();
    const vkind = classifyPurrviewVerification(val);
    if (vkind === "text_pass") continue; // reviewer read the text and accepted it
    if (vkind === "text_fail") {
      visualFails.push({ name: d.name, reason: `The reviewing agent read the text deliverable "${d.name}" and it did NOT pass. Verdict: ${val}. Fix the content and re-verify before reporting done.` });
      continue;
    }
    // A TEXT deliverable is verified by READING its content — never by vision-judging
    // a screenshot. If a text deliverable carries a screenshot value here (a stray
    // render PNG), it must NOT be sent to the image judge: a headless render of raw
    // text lands on a login screen / blank page and the judge false-fails it (the
    // exact "cat approved then bounced 72 as a login screen" class). Demand a PASS/FAIL
    // verdict instead. Gate on d.kind, NOT on the value shape. (See ED 2026-07-03.)
    if (isTextKind(d.kind) && vkind === "screenshot") {
      visualFails.push({ name: d.name, reason: `The text deliverable "${d.name}" was verified with a screenshot, but text is checked by READING it — a picture of text gets misread (e.g. as a login screen). Read the content, confirm it is correct, and record a \`PASS — <why>\` verdict (not an image link). Then re-submit.` });
      continue;
    }
    if (vkind !== "screenshot") {
      visualFails.push({ name: d.name, reason: `The deliverable "${d.name}" has no valid Purrview Verification. For a visual deliverable, take a screenshot/PNG of the finished result, upload it to Supabase Storage, and save that public image URL. For a text deliverable, record a \`PASS — <why>\` verdict. Then re-submit.` });
      continue;
    }
    try {
      const v = await judge({
        imageUrl: val,
        claim: buildJudgeClaim({
          deliverableName: d.name,
          kind: d.kind,
          questTitle,
          questDescription,
        }),
      });
      if (v?.verdict !== "match") {
        visualFails.push({ name: d.name, reason: `The reviewing agent looked at the uploaded screenshot for "${d.name}" and it does not show the deliverable finished/correct. What the reviewer saw: ${v?.reasoning || "no detail given"}. Fix the deliverable, take a fresh screenshot of the corrected result, re-upload it to Supabase, and update the link before reporting done.` });
      }
    } catch (e) {
      // Could not run the vision check → cannot confirm → do NOT pass (conservative).
      visualFails.push({ name: d.name, reason: `The reviewing agent could not open the screenshot for "${d.name}" to check it (the image link may be broken or unreachable): ${e.message}. Make sure the screenshot is a real, public Supabase image URL, then re-submit.` });
    }
  }
  if (visualFails.length > 0) {
    return {
      ok: false,
      failed: ["visual_verification_failed"],
      report: {
        msg: `${visualFails.length} deliverable(s) FAILED actual visual verification — the screenshot was vision-judged, not just checked for presence: ${visualFails.map((f) => f.name).join(", ")}.`,
        fix: "Re-capture a real CIC screenshot showing the rendered deliverable (scroll to trigger lazy-loaded content, capture the full page), set verify_screenshot_ref, and re-run approve. If the deliverable itself is wrong, bounce the quest to execute.",
        failures: visualFails,
      },
    };
  }

  // ONE structured comment per pass — was N per-asset rows + this lockphrase row
  // (the #2 comment-bloat source). The per-asset verdicts live in
  // detail.perItemVerdicts (machine-readable — what countPriorBounces reads) plus
  // a compact roll-up in the summary (human-readable); the overall pass/fail is
  // the top-level `verdict` column the digest (summarizeQuest.openVerifyFails)
  // classifies on. The lockphrase MUST stay in the summary — the DB trigger
  // enforce_stage_gate greps it before allowing the stage write to review.
  const verdictRollup = normalizedVerdicts.map((v) => `${v.item_key}:${v.verdict}`).join(", ");
  const lockSummary = `Approve gate v${GATE_VERSION} passed. ${normalizedVerdicts.length}/${deliverables.length} assets pass [${verdictRollup}]. ${capitalize(APPROVE_LOCKPHRASE)}.${approvalSummary ? ` ${actor_name} note: ${approvalSummary}` : ""}`;
  // `approve` is an UPSERT action: a re-run replaces the prior approve row in
  // place (keyed on quest_id+action), and the first call inserts so the
  // enforce_stage_gate trigger sees the lockphrase comment before the
  // purrview→review stage write. verdict='pass' is threaded through the chokepoint.
  const { recordQuestComment } = await import("@/libs/quest/index.js");
  const { error: commentErr } = await recordQuestComment(questId, {
    source: "questPurrview",
    action: "approve",
    summary: lockSummary,
    actor_name,
    verdict: "pass",
    // An approval is a TERMINAL pass verdict, not an open to-do → resolved=true.
    // Keeps it out of every `resolved=false` feedback scan (registry rung-8 digest,
    // caravan resolveGateForward.hasFeedback) that would otherwise mis-read a PASS
    // as pending feedback by its source prefix. Milestone detection (milestoneByQuest)
    // does NOT filter on resolved, so review-routing is unaffected. (ED 2026-07-03.)
    resolved: true,
    detail: {
      gate_version: GATE_VERSION,
      lockphrase: APPROVE_LOCKPHRASE,
      criteria_checked: APPROVE_CRITERIA_CHECKED,
      items_count: deliverables.length,
      perItemVerdicts: normalizedVerdicts,
      questmaster_summary: approvalSummary || null,
    },
  }, { client: db });
  if (commentErr) {
    return {
      ok: false,
      failed: ["lockphrase_comment_write"],
      report: {
        msg: `Lockphrase comment failed to write: ${commentErr.message}. Stage NOT advanced (comment-before-stage order).`,
        fix: `Debug the quest_comments insert. The DB trigger enforce_stage_gate blocks stage→review without this comment.`,
      },
    };
  }

  // Every active deliverable just PASSED the gate (all verdicts 'pass' + real
  // visual verification) — flip it active → done. This is what makes the
  // "purrview only (re)verifies ACTIVE deliverables" filter (preflightStageTransition,
  // autoVisualVerify) actually mean something: a passed asset is out of scope on any
  // later re-entry to purrview, and the board stops showing it perpetually active.
  // Without this the pass verdict lived ONLY on a quest_comment and the deliverable
  // row stayed 'active' forever. (ED 2026-07-04.)
  const passedIds = deliverables.map((d) => d.id);
  const { error: doneErr } = await db
    .from(publicTables.deliverables)
    .update({ status: "done" })
    .in("id", passedIds);
  if (doneErr) {
    return {
      ok: false,
      failed: ["deliverables_mark_done"],
      report: {
        msg: `Approve verdicts passed and the lockphrase comment was written, but flipping the deliverable(s) to status='done' failed: ${doneErr.message}. Stage NOT advanced.`,
        fix: "Retry approve (idempotent) once the deliverables.status write is unblocked (likely RLS/trigger).",
      },
    };
  }

  // Advance stage (the trigger allows it because the lockphrase comment exists).
  const stageRes = await writeStage(db, questId, "review");
  if (!stageRes.ok) return stageRes.failure;

  return {
    ok: true,
    stage: "review",
    items_count: deliverables.length,
    deliverables_marked_done: passedIds.length,
    lockphrase: APPROVE_LOCKPHRASE,
    criteria_checked: APPROVE_CRITERIA_CHECKED,
  };
}

// ---------------------------------------------------------------------------
// bounce — purrview → plan with per-asset Cat verdicts (≥1 fail) + reason
// (plan = digest: the fail feedback is re-planned/reconciled before re-execution)
// ---------------------------------------------------------------------------

/**
 * @param {{ questId: string, perItemVerdicts: Array<{item_key: string, verdict: 'pass'|'fail', text: string, judge_origin: string}>, reason: string }} args
 */
export async function bounce({ questId, perItemVerdicts, reason, actor_name = QUESTMASTER_ACTOR_DEFAULT }) {
  if (typeof reason !== "string" || !reason.trim()) {
    return {
      ok: false,
      failed: ["reason_required"],
      report: {
        msg: "bounce requires a non-empty `reason` string.",
        fix: "Pass { reason: '<issue-FIRST, plain, actionable feedback the worker can act on without decoding jargon. Lead with WHAT is wrong, then WHAT TO DO. Do NOT use internal field names or codes (no \"lacks verify_screenshot_ref\", no \"404\"). Example: \"The deliverables are not uploaded to Supabase, so the reviewing agent cannot open them to check them. Upload each finished deliverable to Supabase and save its link before reporting done.\">' }.",
      },
    };
  }

  const pre = await preflightStageTransition({ questId, perItemVerdicts, requiredStage: "purrview" });
  if (!pre.ok) return pre.failure;
  const { db, deliverables, normalizedVerdicts } = pre;

  // Bounce-specific gate: at least one verdict === 'fail'
  const fails = normalizedVerdicts.filter((v) => v.verdict === "fail");
  if (fails.length === 0) {
    return {
      ok: false,
      failed: ["no_failures"],
      report: {
        msg: "bounce requires at least one verdict === 'fail'. All verdicts are pass — use approve() instead.",
        fix: "If the quest passes, call approve({ questId, perItemVerdicts }).",
      },
    };
  }

  // Bounce-loop detection: count prior BOUNCE verdict comments per failing
  // asset name across the quest's comment history (purrview + review bounces).
  // Doesn't block — surfaces the count so the worker / Guildmaster can decide
  // whether to escalate instead of running yet another retry.
  const failingNames = fails.map((v) => v.item_key);
  const priorBounces = await countPriorBounces(db, questId, failingNames);

  // Move stage back to plan with SELECT-back verify (plan = digest: the bounce
  // feedback comment is a digestible input the plan leaf reconciles before re-execute)
  const stageRes = await writeStage(db, questId, "plan");
  if (!stageRes.ok) return stageRes.failure;

  // ONE structured comment per pass (was N per-asset rows + this lockphrase row).
  // The full per-asset verdicts ride in detail.perItemVerdicts (countPriorBounces
  // reads them) and the failing names roll up in the summary; the top-level
  // `verdict` is what the digest classifies the open fail on.
  const chronicKeys = failingNames.filter((k) => (priorBounces.get(k) || 0) >= 2); // 2 prior + this one = 3rd bounce
  const lockSummary = `⚠️ NOT DONE — ${fails.length} deliverable(s) did not pass review and must be fixed before this can be reported done: ${failingNames.join(", ")}.\n\n${reason}\n\n${capitalize(BOUNCE_LOCKPHRASE)}.${chronicKeys.length > 0 ? ` These have now failed review 3+ times: ${chronicKeys.join(", ")} — stop resubmitting and escalate to the operator with exactly what is blocking you and the options for fixing it.` : ""}`;
  // `bounce` is PASS_THROUGH (always insert): countPriorBounces tallies these
  // rows' detail.perItemVerdicts, so deduping/upserting would undercount chronic
  // bounces. verdict='fail' is threaded through the chokepoint.
  const { recordQuestComment } = await import("@/libs/quest/index.js");
  const { error: commentErr } = await recordQuestComment(questId, {
    source: "questPurrview",
    action: "bounce",
    summary: lockSummary,
    actor_name,
    verdict: "fail",
    detail: {
      gate_version: GATE_VERSION,
      lockphrase: BOUNCE_LOCKPHRASE,
      criteria_checked: BOUNCE_CRITERIA_CHECKED,
      reason,
      failing_item_keys: failingNames,
      perItemVerdicts: normalizedVerdicts,
    },
  }, { client: db });
  if (commentErr) {
    return {
      ok: false,
      failed: ["lockphrase_comment_write"],
      report: {
        msg: `Stage moved to plan but the bounce audit comment failed to write: ${commentErr.message}.`,
        fix: `Manually insert a quest_comments row with the phrase "${BOUNCE_LOCKPHRASE}" + reason so the worker knows what to fix.`,
        partial: true,
        stage: "plan",
      },
    };
  }

  return {
    ok: true,
    stage: "plan",
    items_count: deliverables.length,
    failing_count: fails.length,
    failing_item_keys: failingNames,
    chronic_bounce_keys: chronicKeys, // asset names that have now bounced 3+ times — caller should consider escalating
    prior_bounce_counts: Object.fromEntries(failingNames.map((k) => [k, priorBounces.get(k) || 0])),
    lockphrase: BOUNCE_LOCKPHRASE,
    criteria_checked: BOUNCE_CRITERIA_CHECKED,
  };
}

/**
 * Count prior bounce verdict comments per asset name (across both Cat and
 * Guildmaster bounces). Reads quest_comments.detail.perItemVerdicts. As of
 * 2026-06-15 each bounce writes ONE comment whose detail.perItemVerdicts carries
 * the full verdict set, so a prior bounce of asset X now counts exactly ONCE
 * (before, the separate per-asset 'asset_verdict' row double-counted it). The
 * legacy 'asset_verdict' action stays in the filter so historical rows still read.
 */
async function countPriorBounces(db, questId, assetNames) {
  if (!assetNames.length) return new Map();
  const { data } = await db
    .from(publicTables.questComments)
    .select("action, detail, summary")
    .eq("quest_id", questId)
    .in("action", ["bounce", "final_gate_bounce", "asset_verdict"]);
  const counts = new Map();
  for (const c of data || []) {
    const verdicts =
      c.detail && typeof c.detail === "object" && Array.isArray(c.detail.perItemVerdicts)
        ? c.detail.perItemVerdicts
        : [];
    for (const v of verdicts) {
      if (v && v.verdict === "fail" && assetNames.includes(v.item_key)) {
        counts.set(v.item_key, (counts.get(v.item_key) || 0) + 1);
      }
    }
  }
  return counts;
}

// ---------------------------------------------------------------------------
// shared preflight — gates 1–6 are common to approve and bounce
// ---------------------------------------------------------------------------

/**
 * Returns either { ok: true, db, deliverables, normalizedVerdicts } or { ok: false, failure: <skill-shaped error> }.
 */
async function preflightStageTransition({ questId, perItemVerdicts, requiredStage }) {
  if (typeof questId !== "string" || !questId.trim()) {
    return {
      ok: false,
      failure: {
        ok: false,
        failed: ["questId"],
        report: { msg: "questId is required (non-empty string).", fix: "Pass { questId: '<uuid>' }." },
      },
    };
  }
  if (!Array.isArray(perItemVerdicts) || perItemVerdicts.length === 0) {
    return {
      ok: false,
      failure: {
        ok: false,
        failed: ["perItemVerdicts_required"],
        report: {
          msg: "perItemVerdicts is required: a non-empty array of {item_key, verdict, text, judge_origin}. item_key is the asset name (deliverables.name).",
          fix: "After running confirmSubmission, judge each produced asset and pass the verdicts here.",
        },
      },
    };
  }

  const db = await database.init("service");

  // Worker handoff must be verified — same logic as confirmSubmission.
  const confirm = await confirmSubmission({ questId });
  if (!confirm.ok) {
    return {
      ok: false,
      failure: {
        ok: false,
        failed: ["confirmSubmission_failed"],
        report: {
          msg: `Worker handoff not verified: ${confirm.msg}`,
          fix: "Cannot approve/bounce a quest that didn't reach purrview through questExecution.submit. Bounce back to execute or escalate.",
          confirmSubmissionReason: confirm.reason,
        },
      },
    };
  }
  // confirmSubmission also enforces stage === 'purrview'; redundant guard for clarity:
  if (requiredStage !== "purrview") {
    return {
      ok: false,
      failure: {
        ok: false,
        failed: ["unsupported_stage"],
        report: { msg: `preflightStageTransition only supports requiredStage='purrview'.` },
      },
    };
  }

  const { data: deliverables, error: iErr } = await db
    .from(publicTables.deliverables)
    .select("id, name, kind, ref, status, verify_screenshot_ref")
    .eq("quest_id", questId);
  if (iErr) {
    return {
      ok: false,
      failure: {
        ok: false,
        failed: ["deliverables_read"],
        report: { msg: iErr.message, fix: "DB read against `deliverables` failed." },
      },
    };
  }
  // Purrview only (re)verifies ACTIVE deliverables. A non-active row (e.g. status
  // 'done'/archived/superseded) is out of scope — it needs no verdict and no
  // verification link. `status` defaults to 'active' when unset.
  const deliverableList = (deliverables || []).filter((d) => (d.status ?? "active") === "active");

  // Normalize verdicts and require 1:1 match against produced deliverables by name.
  // judge_origin is REQUIRED — Cat (T2 Composer 2.0) alone misses too much,
  // so every verdict must carry the verdict from a vision judge. judge_rationale
  // is optional but recommended; it gets written into the per-asset comment text
  // for audit. judge_model is optional metadata.
  const normalizedVerdicts = perItemVerdicts
    .map((v) => {
      if (!v || typeof v !== "object") return null;
      const item_key = typeof v.item_key === "string" ? v.item_key.trim() : "";
      const verdict = v.verdict === "pass" ? "pass" : v.verdict === "fail" ? "fail" : null;
      const text = typeof v.text === "string" ? v.text.trim() : "";
      const judge_origin = typeof v.judge_origin === "string" ? v.judge_origin.trim() : "";
      const judge_rationale =
        typeof v.judge_rationale === "string" ? v.judge_rationale.trim() : "";
      const judge_model = typeof v.judge_model === "string" ? v.judge_model.trim() : "";
      if (!item_key || !verdict || !text) return null;
      return { item_key, verdict, text, judge_origin, judge_rationale, judge_model };
    })
    .filter(Boolean);

  if (normalizedVerdicts.length !== perItemVerdicts.length) {
    return {
      ok: false,
      failure: {
        ok: false,
        failed: ["verdict_shape"],
        report: {
          msg: "Each verdict must be { item_key: string (asset name), verdict: 'pass'|'fail', text: string-non-empty, judge_origin: string-non-empty }. Optional: judge_rationale, judge_model.",
          fix: "Fix the malformed entries and retry.",
        },
      },
    };
  }

  // A TEXT-kind deliverable must be verified by READING it, so its verdict cannot
  // carry an IMAGE judge_origin. This is the structural backstop for the "cat
  // approved, then bounced 72 as a login screen" class: a review leaf that renders
  // a text deliverable to a PNG and vision-judges it produces a false 'fail'. Reject
  // the verdict here so no such bounce (or approve) can ever land — the leaf must
  // re-verify by reading (judge_origin='claude-multimodal-read'). (ED 2026-07-03.)
  const kindByName = new Map(deliverableList.map((d) => [d.name, d.kind]));
  const textImageJudged = normalizedVerdicts.filter(
    (v) => isTextKind(kindByName.get(v.item_key)) && isImageJudgeOrigin(v.judge_origin),
  );
  if (textImageJudged.length > 0) {
    return {
      ok: false,
      failure: {
        ok: false,
        failed: ["text_deliverable_image_judged"],
        report: {
          msg: `${textImageJudged.length} text deliverable(s) were vision-judged from a screenshot: ${textImageJudged.map((v) => v.item_key).join(", ")}. Text is checked by READING its content — a picture of text gets misread (e.g. as a login screen), which is a false verdict.`,
          fix: "For each text deliverable, READ its ref content, compare it to what the quest asks for, and post the verdict with judge_origin='claude-multimodal-read' (not openai_images.judge). Do NOT render text to a PNG or set an image verify_screenshot_ref for it.",
          item_keys: textImageJudged.map((v) => v.item_key),
        },
      },
    };
  }

  const missingJudge = normalizedVerdicts.filter((v) => !isValidJudgeOrigin(v.judge_origin));
  if (missingJudge.length > 0) {
    return {
      ok: false,
      failure: {
        ok: false,
        failed: ["judge_origin_missing_or_invalid"],
        report: {
          msg: `${missingJudge.length} verdict(s) missing or have invalid judge_origin. Cat must invoke openai_images.judge per asset BEFORE posting verdicts (or use claude-multimodal-read for the T3.5 final gate). Cat alone has 0/2 catch rate on visual verification — the judge is mandatory, not optional. See CLAUDE.md WWCD layer-attribution bans.`,
          fix: "For each asset, call `judge({ imageUrl: item.ref, claim: <the step output[].verification for this asset> })` from `@/libs/weapon/openai_images`. Map the returned verdict 'match'→'pass', 'mismatch'/'inconclusive'→'fail'. Then pass the result as `{ ..., judge_origin: 'openai_images.judge:gpt-4o', judge_rationale: returned.reasoning, judge_model: returned.model }`.",
          missing_item_keys: missingJudge.map((v) => v.item_key),
          accepted_origins: [...VALID_JUDGE_ORIGINS, "openai_images.judge:<any-model>"],
        },
      },
    };
  }

  if (normalizedVerdicts.length !== deliverableList.length) {
    const verdictKeys = normalizedVerdicts.map((v) => v.item_key);
    const itemKeys = deliverableList.map((i) => i.name);
    return {
      ok: false,
      failure: {
        ok: false,
        failed: ["verdict_count"],
        report: {
          msg: `Verdict count (${normalizedVerdicts.length}) !== produced-deliverables count (${deliverableList.length}). Cat must judge every produced asset.`,
          fix: "Provide one verdict per deliverables.name. Don't skip any.",
          missing_keys: itemKeys.filter((k) => !verdictKeys.includes(k)),
          unexpected_keys: verdictKeys.filter((k) => !itemKeys.includes(k)),
        },
      },
    };
  }

  const itemNameSet = new Set(deliverableList.map((i) => i.name));
  const unmatched = normalizedVerdicts.filter((v) => !itemNameSet.has(v.item_key));
  if (unmatched.length > 0) {
    return {
      ok: false,
      failure: {
        ok: false,
        failed: ["verdict_key_mismatch"],
        report: {
          msg: `${unmatched.length} verdict(s) reference asset names that don't exist on this quest.`,
          fix: "Use the exact asset names returned by confirmSubmission.deliverables[].name.",
          unmatched_keys: unmatched.map((v) => v.item_key),
        },
      },
    };
  }

  return { ok: true, db, deliverables: deliverableList, normalizedVerdicts, description: confirm.description || "", title: confirm.title || "" };
}

// (writeVerdictComments removed 2026-06-15 — approve/bounce now fold all per-asset
//  verdicts into the SINGLE lockphrase comment's detail.perItemVerdicts + top-level
//  `verdict`, instead of one quest_comments row per asset. See engineering_decisions.md.)

async function writeStage(db, questId, nextStage) {
  const { error: upErr } = await db.from(publicTables.quests).update({ stage: nextStage }).eq("id", questId);
  if (upErr) {
    return {
      ok: false,
      failure: {
        ok: false,
        failed: ["stage_write"],
        report: { msg: upErr.message, fix: "Stage write failed at the DB layer. Contact Guildmaster." },
      },
    };
  }
  const { data: verify } = await db
    .from(publicTables.quests)
    .select("stage")
    .eq("id", questId)
    .single();
  if (verify?.stage !== nextStage) {
    return {
      ok: false,
      failure: {
        ok: false,
        failed: ["stage_write_verify"],
        report: { msg: `Stage write did not persist; reads back as "${verify?.stage}".`, fix: "Likely RLS or trigger. Contact Guildmaster." },
      },
    };
  }
  return { ok: true };
}

function capitalize(s) {
  return s.charAt(0).toUpperCase() + s.slice(1);
}

// ---------------------------------------------------------------------------
// autoVisualVerify — headless visual verification (no CIC/browser session).
// Renders each unverified deliverable to a PNG, uploads it to GuildOS_Bucket,
// and sets verify_screenshot_ref — so the headless cron review leaf can satisfy
// the approve gate's visual-verification check autonomously.
// ---------------------------------------------------------------------------

/** Render a deliverable's ref to a PNG Buffer, headlessly. Thin delegate to the
 *  single browser substrate (leafBrowser.screenshotUrl) — which owns the pdf.js
 *  multi-page render, lazy-load scrolling, fail-closed-on-blank, and ref→URL
 *  resolution. Do NOT re-introduce a local chromium.launch here (see leafBrowser
 *  engineering_decisions.md 2026-06-15 — one substrate). */
async function renderDeliverableScreenshot({ ref, kind }) {
  const res = await screenshotUrl({
    ref,
    kind,
    waitUntil: "domcontentloaded",
    scrollLazyLoad: false,
    timeoutMs: 15000,
  });
  if (!res.ok) throw new Error(res.error);
  return res.buffer;
}

/**
 * Fill verify_screenshot_ref for every deliverable that has a ref but no
 * screenshot, by rendering it headlessly + uploading the PNG. No CIC needed.
 * @param {{ questId: string }} args
 * @returns {Promise<{ ok: boolean, captured: Array, skipped: Array, errors: Array }>}
 */
export async function autoVisualVerify({ questId }) {
  if (typeof questId !== "string" || !questId.trim()) {
    return { ok: false, captured: [], skipped: [], errors: [{ reason: "questId_required" }] };
  }
  const db = await database.init("service");
  const { data: deliverables, error } = await db
    .from(publicTables.deliverables)
    .select("id, name, kind, ref, status, verify_screenshot_ref")
    .eq("quest_id", questId);
  if (error) return { ok: false, captured: [], skipped: [], errors: [{ reason: error.message }] };

  const captured = [], skipped = [], errors = [];
  for (const d of deliverables || []) {
    // Purrview only (re)verifies ACTIVE deliverables (status defaults to 'active').
    if ((d.status ?? "active") !== "active") { skipped.push({ name: d.name, reason: `status is '${d.status}', not active` }); continue; }
    const hasShot = d.verify_screenshot_ref && String(d.verify_screenshot_ref).trim();
    const hasRef = d.ref && String(d.ref).trim();
    if (hasShot) { skipped.push({ name: d.name, reason: "already has verify_screenshot_ref" }); continue; }
    // TEXT deliverables are verified by READING the content (a PASS/FAIL verdict),
    // not by a screenshot — the headless renderer can't judge text. Leave those to
    // the purrview leaf's text-verify branch.
    if (String(d.kind || "text") === "text") { skipped.push({ name: d.name, reason: "text kind — verified by reading content (purrview leaf writes a PASS/FAIL verdict), not a headless screenshot" }); continue; }
    if (!hasRef) { errors.push({ name: d.name, reason: "no ref to render (deliverable not produced)" }); continue; }
    try {
      const buf = await renderDeliverableScreenshot({ ref: d.ref, kind: d.kind });
      const path = `quest-assets/${questId}/verify/${d.id}.png`;
      await writeFile({ path, content: buf, contentType: "image/png", upsert: true });
      const { url } = readPublicUrl({ path });
      const { error: upErr } = await db.from(publicTables.deliverables).update({ verify_screenshot_ref: url }).eq("id", d.id);
      if (upErr) { errors.push({ name: d.name, reason: "db set failed: " + upErr.message }); continue; }
      captured.push({ name: d.name, verify_screenshot_ref: url });
    } catch (e) {
      errors.push({ name: d.name, reason: "render/upload failed: " + (e.message || String(e)).split("\n")[0] });
    }
  }
  return { ok: errors.length === 0, captured, skipped, errors };
}

// ---------------------------------------------------------------------------
// adviseRenderSanity — SIO #1 pre-submit render-sanity, ADVISE MODE.
//
// Five quests (ac48ab69, 7ec94b9d, 3f5860e4, 4c53dd27, 4065db7e) each burned
// 2-3 render→vision-judge→bounce cycles on failures a cheap HTTP fetch of the
// deliverable ref would have caught up front: localhost refs unreachable from
// the cloud reviewer, HTML served with the wrong content-type (renders as raw
// code), escaped-source pages, blank bodies, restrictive CSP. This runs those
// cheap checks at submit time and ANNOTATES the thread with fixable-error notes
// + a queryable hit-rate counter — but it NEVER blocks (advise, not gate). If
// the operator sees a high would-have-caught rate in the telemetry, advise can
// later be promoted to a hard gate.
// ---------------------------------------------------------------------------

/**
 * PURE heuristic evaluator — given a deliverable + the head of its HTTP response,
 * return the fixable render issues. No I/O, so it is unit-testable in isolation.
 * The render-shape checks (content-type / escaped-source / blank / CSP) apply
 * only to `kind === 'html'` — a PNG served as image/png or a PDF as
 * application/pdf is correct, not a defect.
 *
 * @returns {Array<{ code:string, issue:string, fix:string }>}
 */
export function evaluateRenderSanity({ name, kind, ref, fetchOk, status, contentType = "", contentDisposition = "", csp = "", bodyHead = "" }) {
  const findings = [];
  const k = String(kind || "").toLowerCase();
  const r = String(ref || "");
  const ct = String(contentType || "").toLowerCase();
  const cd = String(contentDisposition || "").toLowerCase();
  const head = String(bodyHead || "");

  // 1. localhost / loopback ref — unreachable from the reviewer's cloud browser.
  if (/^\s*https?:\/\/(localhost|127\.0\.0\.1|0\.0\.0\.0|\[::1\]|::1)([:/]|$)/i.test(r)) {
    findings.push({
      code: "localhost_ref",
      issue: `"${name}" is hosted at a localhost/loopback URL — the reviewer's cloud browser can't reach it, so the render will be blank.`,
      fix: "Upload the asset to Supabase storage (or public/proving-grounds) and point ref at the public https URL, then re-capture the verify screenshot.",
    });
  }

  // 2. unreachable — fetch failed, or an error status.
  if (fetchOk === false) {
    findings.push({
      code: "unreachable",
      issue: `"${name}" could not be fetched from its ref (network error or timeout).`,
      fix: "Confirm the ref URL loads in a browser; host it somewhere publicly reachable and update ref.",
    });
  } else if (typeof status === "number" && status >= 400) {
    findings.push({
      code: "http_error",
      issue: `"${name}" returns HTTP ${status} — the ref does not resolve to content.`,
      fix: "Re-upload the asset and set ref to a URL that returns HTTP 200.",
    });
  }

  // The rest concern how an HTML page will RENDER — skip images/docs/video and
  // any ref we could not read a body from.
  const gotBody = fetchOk !== false && (typeof status !== "number" || status < 400);
  if (k === "html" && gotBody) {
    // 3a. served as a download → the browser saves the file, renders no page.
    if (/attachment/i.test(cd)) {
      findings.push({
        code: "content_disposition_attachment",
        issue: `"${name}" is served with Content-Disposition: attachment, so the browser downloads the file instead of rendering it (the reviewer sees no page).`,
        fix: "Serve the HTML inline — e.g. Supabase storage writeFile with contentType 'text/html' and no attachment disposition.",
      });
    } else if (ct && !/^text\/html\b/.test(ct)) {
      // 3b. wrong content-type → Chrome paints raw source, not a page.
      findings.push({
        code: "wrong_content_type",
        issue: `"${name}" is an html deliverable but is served as "${ct}" — a browser shows it as raw source code, not a rendered page (the "HTML rendered as code" bounce).`,
        fix: "Re-upload with content-type text/html (supabase_storage.writeFile contentType:'text/html').",
      });
    }

    // 4. escaped HTML source — literal &lt;…&gt; entities paint as text even under text/html.
    const escaped = (head.match(/&lt;/g) || []).length;
    if (escaped >= 3) {
      findings.push({
        code: "escaped_source",
        issue: `"${name}" contains escaped HTML entities (&lt;…&gt;) in its body — it will display as literal source text, not a rendered page.`,
        fix: "Store the raw, unescaped HTML so the markup is real tags rather than escaped text.",
      });
    }

    // 5. blank body — nothing to render.
    const textLen = head.replace(/<[^>]*>/g, "").replace(/\s+/g, " ").trim().length;
    if (head.trim().length === 0 || textLen < 8) {
      findings.push({
        code: "blank_body",
        issue: `"${name}" has an essentially empty body — the page will render blank.`,
        fix: "Confirm the deliverable file actually contains the built content before submitting.",
      });
    }

    // 6. restrictive CSP (response header or <meta http-equiv>) — advisory.
    const metaCsp = /http-equiv=["']?content-security-policy/i.test(head);
    if ((csp && /(?:default-src|script-src|style-src|img-src)\s+'none'/i.test(csp)) || metaCsp) {
      findings.push({
        code: "csp_restrictive",
        issue: `"${name}" declares a Content-Security-Policy that may block Chrome from rendering inline styles/scripts/images (the "CSP blocking Chrome" bounce).`,
        fix: "Relax the CSP for the rendered deliverable, or inline assets as data: URLs so nothing is blocked.",
      });
    }
  }

  return findings;
}

/** Fetch just the head of a ref (Range 0-8KB, browser-ish UA) to read status +
 *  headers (+ body for html). Best-effort: any failure returns { fetchOk:false }.
 *  For non-html kinds the body is not needed, so the stream is cancelled. */
async function fetchRefHead(url, timeoutMs = 8000, wantBody = true) {
  const ctrl = new AbortController();
  const t = setTimeout(() => ctrl.abort(), timeoutMs);
  try {
    const resp = await fetch(url, {
      method: "GET",
      redirect: "follow",
      signal: ctrl.signal,
      headers: {
        "User-Agent": "Mozilla/5.0 (GuildOS render-advise)",
        Range: "bytes=0-8191",
        Accept: "text/html,application/xhtml+xml,*/*",
      },
    });
    const out = {
      fetchOk: true,
      status: resp.status,
      contentType: resp.headers.get("content-type") || "",
      contentDisposition: resp.headers.get("content-disposition") || "",
      csp: resp.headers.get("content-security-policy") || "",
      bodyHead: "",
    };
    if (wantBody) {
      try {
        const buf = await resp.arrayBuffer();
        out.bodyHead = Buffer.from(buf).subarray(0, 8192).toString("utf8");
      } catch { /* body read best-effort */ }
    } else {
      try { await resp.body?.cancel(); } catch { /* ignore */ }
    }
    return out;
  } catch (e) {
    return { fetchOk: false, status: 0, error: e?.message || String(e) };
  } finally {
    clearTimeout(t);
  }
}

/**
 * Run the ADVISORY render-sanity pass over a quest's deliverables. NEVER blocks.
 * Writes one upserted 'advise_render_check' comment (fixable-error notes) and a
 * queryable 'render_advise' telemetry row (would_have_bounced) for hit-rate.
 *
 * @param {{ questId:string, deliverables?:Array, db?:object }} args
 *   deliverables/db are optional — passed in by confirmSubmission to avoid a
 *   re-query; omitted when a worker calls this standalone pre-submit.
 * @returns {Promise<{ ok:boolean, would_have_bounced:boolean, checked?:number, findings:Array, error?:string }>}
 */
export async function adviseRenderSanity({ questId, deliverables: passedDeliverables = null, db: passedDb = null } = {}) {
  if (typeof questId !== "string" || !questId.trim()) {
    return { ok: false, would_have_bounced: false, findings: [], error: "questId_required" };
  }
  const db = passedDb || (await database.init("service"));
  let deliverables = passedDeliverables;
  if (!Array.isArray(deliverables)) {
    const { data, error } = await db
      .from(publicTables.deliverables)
      .select("id, name, kind, ref")
      .eq("quest_id", questId);
    if (error) return { ok: false, would_have_bounced: false, findings: [], error: error.message };
    deliverables = data || [];
  }

  const MAX = 12; // advisory — keep the submit-time fetch budget bounded
  const toCheck = deliverables.slice(0, MAX);
  const findings = [];
  await Promise.all(
    toCheck.map(async (d) => {
      const ref = d.ref && String(d.ref).trim();
      const kind = String(d.kind || "").toLowerCase();
      if (!ref) {
        findings.push({ name: d.name, code: "not_produced", issue: `"${d.name}" has no ref yet (not produced).`, fix: "Produce the asset, upload it, and set ref before submitting." });
        return;
      }
      // Only http(s) refs are fetchable here; storage-path refs are resolved by
      // the renderer downstream, so a cheap fetch can't reach them — skip.
      if (!/^https?:\/\//i.test(ref)) return;
      const probe = await fetchRefHead(ref, 8000, kind === "html");
      const fs = evaluateRenderSanity({
        name: d.name, kind, ref,
        fetchOk: probe.fetchOk, status: probe.status,
        contentType: probe.contentType, contentDisposition: probe.contentDisposition,
        csp: probe.csp, bodyHead: probe.bodyHead,
      });
      for (const f of fs) findings.push({ name: d.name, ...f });
    }),
  );

  const wouldHaveBounced = findings.length > 0;
  const summary = wouldHaveBounced
    ? `[RENDER-ADVISE] ${findings.length} fixable render issue(s) across ${toCheck.length} deliverable(s) — advisory, submit NOT blocked: ${findings.map((f) => `${f.name}: ${f.code}`).join("; ")}`.slice(0, 2000)
    : `[RENDER-ADVISE] render-sanity clean across ${toCheck.length} deliverable(s).`;

  // (a) Visible ADVISORY annotation — one UPSERTED comment (never appended per
  //     pass; see UPSERT_ACTIONS in libs/quest). Dynamic import breaks the cycle.
  try {
    const { recordQuestComment } = await import("@/libs/quest/index.js");
    await recordQuestComment(questId, {
      source: "questPurrview",
      action: "advise_render_check",
      summary,
      verdict: wouldHaveBounced ? "warn" : "pass",
      detail: { mode: "advise", would_have_bounced: wouldHaveBounced, checked: toCheck.length, flagged: findings.length, findings, generated_by: "adviseRenderSanity" },
      actor_name: "render-advise",
    }, { client: db });
  } catch { /* advisory — best-effort, never blocks */ }

  // (b) Queryable HIT-RATE counter — telemetry channel (prunable, out of the
  //     comment thread). Emitted on EVERY run so the operator can compute the
  //     would-have-caught rate and decide whether to promote advise→gate.
  try {
    const { recordQuestEventLog } = await import("@/libs/caravan/logs.js");
    await recordQuestEventLog(questId, {
      action: "render_advise",
      summary,
      detail: { would_have_bounced: wouldHaveBounced, checked: toCheck.length, flagged: findings.length, codes: findings.map((f) => f.code) },
    }, { client: db });
  } catch { /* telemetry — best-effort */ }

  return { ok: true, would_have_bounced: wouldHaveBounced, checked: toCheck.length, findings };
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/questPurrview/index.test.js -->
```js
import { describe, it, expect, vi } from "vitest";

// Mock the DB-touching deps so the import resolves without a live Supabase
// client. We're testing the pure validators here, not the gate flow.
vi.mock("@/libs/council/database", () => ({
  database: { init: vi.fn() },
}));

vi.mock("@/libs/council/publicTables", () => ({
  publicTables: { quests: "quests", items: "items", steps: "steps", questComments: "quest_comments" },
}));

vi.mock("@/libs/weapon/questExecution", () => ({
  SUBMIT_LOCKPHRASE: "this quest now meets the criteria for purrview",
}));

import { isValidJudgeOrigin, APPROVE_LOCKPHRASE, BOUNCE_LOCKPHRASE, evaluateRenderSanity, classifyPurrviewVerification, isTextKind, isImageJudgeOrigin } from "./index.js";

describe("isTextKind / isImageJudgeOrigin — text deliverables are read, never vision-judged (72-as-login-screen guard)", () => {
  it("text-like kinds are non-visual (read-verified)", () => {
    for (const k of ["text", "json", "markdown", "md", "csv", "TEXT", "JSON"]) expect(isTextKind(k)).toBe(true);
    expect(isTextKind(null)).toBe(true); // defaults to text
    expect(isTextKind(undefined)).toBe(true);
  });
  it("visual kinds are NOT text-kind (screenshot-verified)", () => {
    for (const k of ["html", "image", "document", "pdf", "video"]) expect(isTextKind(k)).toBe(false);
  });
  it("openai_images.judge origins are image-judge origins", () => {
    expect(isImageJudgeOrigin("openai_images.judge")).toBe(true);
    expect(isImageJudgeOrigin("openai_images.judge:gpt-4o")).toBe(true);
    expect(isImageJudgeOrigin("claude-multimodal-read")).toBe(false);
    expect(isImageJudgeOrigin("")).toBe(false);
    expect(isImageJudgeOrigin(null)).toBe(false);
  });
});

describe("classifyPurrviewVerification — polymorphic Purrview Verification value (text vs screenshot)", () => {
  it("classifies a Supabase screenshot URL as 'screenshot'", () => {
    expect(classifyPurrviewVerification("https://x.supabase.co/storage/v1/object/public/b/verify.png")).toBe("screenshot");
    expect(classifyPurrviewVerification("http://any.host/shot.png")).toBe("screenshot");
  });

  it("classifies a PASS text verdict as 'text_pass' (accepted without a screenshot)", () => {
    expect(classifyPurrviewVerification("PASS — answer is 72, correct")).toBe("text_pass");
    expect(classifyPurrviewVerification("pass: reads correctly")).toBe("text_pass");
  });

  it("classifies a FAIL text verdict as 'text_fail' (bounces)", () => {
    expect(classifyPurrviewVerification("FAIL — asked for 8×9 but says 63")).toBe("text_fail");
  });

  it("treats null/blank as 'empty' (not verified yet)", () => {
    expect(classifyPurrviewVerification("")).toBe("empty");
    expect(classifyPurrviewVerification("   ")).toBe("empty");
    expect(classifyPurrviewVerification(null)).toBe("empty");
    expect(classifyPurrviewVerification(undefined)).toBe("empty");
  });

  it("rejects a non-URL, non-verdict value as 'invalid' (no laundering a bare path/prose)", () => {
    expect(classifyPurrviewVerification("C:/tmp/shot.png")).toBe("invalid");
    expect(classifyPurrviewVerification("looks good to me")).toBe("invalid");
  });
});

describe("isValidJudgeOrigin — Cat per-item judge mandate", () => {
  it("accepts the canonical openai_images.judge tag", () => {
    expect(isValidJudgeOrigin("openai_images.judge")).toBe(true);
    expect(isValidJudgeOrigin("openai_images.judge:gpt-4o")).toBe(true);
  });

  it("accepts openai_images.judge:<any-model> for forward-compat", () => {
    expect(isValidJudgeOrigin("openai_images.judge:gpt-4o-2024-11-20")).toBe(true);
    expect(isValidJudgeOrigin("openai_images.judge:o4-mini")).toBe(true);
  });

  it("accepts the local-Claude direct multimodal Read origin (T3.5)", () => {
    expect(isValidJudgeOrigin("claude-multimodal-read")).toBe(true);
  });

  it("rejects empty / whitespace / non-string", () => {
    expect(isValidJudgeOrigin("")).toBe(false);
    expect(isValidJudgeOrigin("   ")).toBe(false);
    expect(isValidJudgeOrigin(null)).toBe(false);
    expect(isValidJudgeOrigin(undefined)).toBe(false);
    expect(isValidJudgeOrigin(42)).toBe(false);
  });

  it("rejects unrecognized origins (no laundering through made-up labels)", () => {
    expect(isValidJudgeOrigin("cat")).toBe(false);
    expect(isValidJudgeOrigin("composer")).toBe(false);
    expect(isValidJudgeOrigin("manual-review")).toBe(false);
    expect(isValidJudgeOrigin("eyeball")).toBe(false);
    // Even a near-miss prefix should fail unless it's the openai_images.judge family.
    expect(isValidJudgeOrigin("openai-judge")).toBe(false);
  });
});

describe("evaluateRenderSanity — SIO #1 advise-mode heuristics (pure)", () => {
  const ok = { name: "page", kind: "html", ref: "https://x.supabase.co/storage/v1/object/public/b/p.html", fetchOk: true, status: 200, contentType: "text/html; charset=utf-8", bodyHead: "<!DOCTYPE html><body><h1>Real content here that renders</h1></body>" };

  const codes = (args) => evaluateRenderSanity(args).map((f) => f.code);

  it("passes a clean, correctly-served html page (no findings)", () => {
    expect(evaluateRenderSanity(ok)).toEqual([]);
  });

  it("flags a localhost/loopback ref (unreachable from the cloud reviewer)", () => {
    expect(codes({ ...ok, ref: "http://localhost:3002/proving-grounds/x/index.html" })).toContain("localhost_ref");
    expect(codes({ ...ok, ref: "http://<REDACTED-IP>:3001/x.html" })).toContain("localhost_ref");
  });

  it("flags an unreachable ref and an HTTP error status", () => {
    expect(codes({ ...ok, fetchOk: false })).toContain("unreachable");
    expect(codes({ ...ok, status: 404 })).toContain("http_error");
  });

  it("flags html served with the wrong content-type (renders as raw code)", () => {
    expect(codes({ ...ok, contentType: "text/plain" })).toContain("wrong_content_type");
    expect(codes({ ...ok, contentType: "application/octet-stream" })).toContain("wrong_content_type");
  });

  it("flags a Content-Disposition: attachment (download, not render)", () => {
    expect(codes({ ...ok, contentDisposition: "attachment; filename=p.html" })).toContain("content_disposition_attachment");
  });

  it("flags escaped HTML source that would paint as literal text", () => {
    expect(codes({ ...ok, bodyHead: "&lt;!doctype&gt; &lt;div&gt;hi&lt;/div&gt; &lt;span&gt;x&lt;/span&gt;" })).toContain("escaped_source");
  });

  it("flags a blank body", () => {
    expect(codes({ ...ok, bodyHead: "   " })).toContain("blank_body");
    expect(codes({ ...ok, bodyHead: "<html><body></body></html>" })).toContain("blank_body");
  });

  it("flags a restrictive CSP (header or meta) — advisory", () => {
    expect(codes({ ...ok, csp: "default-src 'none'" })).toContain("csp_restrictive");
    expect(codes({ ...ok, bodyHead: "<meta http-equiv=\"Content-Security-Policy\" content=\"default-src 'none'\"><h1>hi there content</h1>" })).toContain("csp_restrictive");
  });

  it("does NOT apply render-shape checks to non-html kinds (a PNG as image/png is correct)", () => {
    expect(evaluateRenderSanity({ ...ok, kind: "image", contentType: "image/png", bodyHead: "" })).toEqual([]);
    expect(evaluateRenderSanity({ ...ok, kind: "document", contentType: "application/pdf", bodyHead: "" })).toEqual([]);
  });

  it("still flags localhost/unreachable for non-html kinds (reachability is kind-agnostic)", () => {
    expect(codes({ ...ok, kind: "image", ref: "http://localhost:3002/x.png", contentType: "image/png" })).toContain("localhost_ref");
    expect(codes({ ...ok, kind: "image", fetchOk: false, contentType: "image/png" })).toContain("unreachable");
  });
});

describe("Lockphrase exports — must stay verbatim", () => {
  it("exports the approve lockphrase verbatim", () => {
    expect(APPROVE_LOCKPHRASE).toBe("this quest now meets the criteria for review");
  });

  it("exports the bounce lockphrase verbatim", () => {
    expect(BOUNCE_LOCKPHRASE).toBe("this quest has been bounced back to plan");
  });
});
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

## 2026-07-09 — Lease prompts APPOINT skills / howTos; repeating procedure is fetched-and-adopted on demand, never inlined, never ad-hoc

**Decision.** A `buildLeaf*Prompt` states the stage OUTCOME and APPOINTS the capability
that produces it — either by naming the skill-book howTo to pull
(`run-action.mjs --howto <skillBookId> <action>`) or by telling the leaf to use a named
skill — rather than inlining the reusable procedure. Any procedure that repeats across
quests/stages lives in the skill book's `toc.<action>.howTo` (or a skill/weapon), retrieved
lazily. The fixed shape: *use skill/howTo X to do Y; if you don't have X, get it — pull+adopt
it via `skillExchange`, and if it exists nowhere, FORGE it via `blacksmith`; escalate only as
last resort.* The terminal chain for a missing capability is
local skill/howTo → `skillExchange` pull+adopt → `blacksmith` forge → escalate —
**never ad-hoc/hand-rolled** (the scratch-guard + stop-gate hooks block ad-hoc scratch).

**Rejected alternative.** Baking the full procedure (browser-capture steps, write-back op
shapes, submission-body JSON) straight into `buildLeaf*Prompt`. Also rejected: an "if the
skill is missing, just do it ad-hoc" fallback clause — it contradicts the positive law that
skills + weapons are the only methods beyond native tool calls, and the hooks refuse it.

**Why.** This is the authoring-side mirror of GuildOS2 `libs/quest` "Lease prompts appoint
skills" (2026-07-09) — same principle, GuildOS's pointer-and-retrieve model
(`--howto`/skill-book howTo) instead of GuildOS2's `HOWTOS` special-situations. Procedure is
the part of a prompt that only grows; inlined, it buries the stage-routing seam and duplicates
across the `buildLeaf*` family, so a tuning edit has to be made in N prompts instead of one
howTo. Appointing by outcome lets a heterogeneous worker resolve the capability it lacks; the
forge-not-ad-hoc terminal keeps every method inside the registry the hooks enforce.

**Consequence if violated.** Re-inlining regrows wall-of-text prompts and duplicates procedure
across the leaf builders (drift between copies). An ad-hoc-fallback clause trains leaves to
fight their own scratch-guards. Naming a skill the worker can't resolve, with no
pull/adopt/forge path, dead-ends the leaf.

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

---

## 2026-07-08 — Hermes leaf's codex app-server needs `%LOCALAPPDATA%/OpenAI/Codex/bin` on PATH (the caller must prepend it)

**Decision.** The Hermes leaf runs `--provider openai-codex --model gpt-5-codex`
over the `codex_app_server` runtime (hermes config
`%LOCALAPPDATA%/hermes/config.yaml`: `model.openai_runtime: codex_app_server`).
This ONLY works because `callHermes` (`hermes-leaf.mjs`) prepends
`%LOCALAPPDATA%/OpenAI/Codex/bin` to the child's PATH so hermes' `codex
app-server` subprocess resolves the REAL `codex.exe` installed there.

**Rejected alternative.** Assume `codex` is globally on PATH, or "fix" a spawn
failure by switching to `--model gpt-5.5` over the `auto` (chat_completions)
runtime.

**Why.** Hermes spawns the bare name `codex`, and Python `subprocess` resolves
only `.exe`, never an npm `.cmd`/`.ps1` shim — so if PATH carries only the npm
shim (or nothing) the turn dies `[WinError 2] cannot find the file specified`.
The official Codex installer's `codex.exe` DOES live at
`%LOCALAPPDATA%/OpenAI/Codex/bin`; with that dir on PATH, gpt-5-codex answers
correctly (verified: returns "72"). The chat_completions fallback (`gpt-5.5` +
`auto`) also works but is NOT the canonical path — GuildOS2's guild client
(`hermes/guildos2/guild.py`) pins gpt-5-codex + app-server, so the machine config
and this leaf must match it. Credential: the `openai-codex` device_code OAuth in
`hermes auth` (the Claude subscription path is billing-blocked — `--provider
anthropic` returns 400 "out of extra usage").

**Consequence if violated.** Drop the PATH prepend (or flip the config to `auto`
without changing the model) and the leaf fails silently — WinError 2, or
"no final response" for gpt-5-codex under chat_completions — the quest never
advances and the caravan re-dispatches it every tick.

## 2026-07-08 — The hermes-type adventurer's `system_prompt` points at `hermes-leaf.mjs`, not at hermes directly

**Decision.** The `Main_Hermes_Codex` adventurer (worker_type=`hermes`,
guilds=`[main,hermes]`) has `system_prompt` = `node --no-warnings
libs/weapon/questmaster/hermes-leaf.mjs {{PROMPT_FILE}}`. The caravan's
`invokeAdventurerLeaf` renders that template and the `.mjs` wrapper does all the
GuildOS DB ceremony (getQuestDetails → ship deliverables → patchObjective →
moveToReview), handing only the reasoning to `hermes -z`.

**Rejected alternative.** Point the system_prompt straight at `uv run … hermes
-z {{PROMPT}}` (or the pre-existing `npx @openai/codex exec` command the row
carried) and let the agent itself run the `run-action.mjs` quest mutations.

**Why.** The standard execute-leaf prompt tells the agent to shell
`node run-action.mjs` for every mutation; a Codex/Hermes agent narrates those
instead of executing them (or the nested node dies in-sandbox), so the quest
reads "worked on" but never actually advances. `hermes-leaf.mjs` runs the
deterministic reads/writes OUTSIDE the agent and only sends the creative slice
in. Proven end-to-end via the caravan on 2026-07-08 (quest d12faba6: leased
`cleaf-…`, shipped deliverable "72", objective done, moved to review).

**Consequence if violated.** Repoint the row at raw hermes/codex and the leaf
stops writing back — objectives/deliverables stay untouched and the quest
loops in `execute` forever.
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
import { tmpdir, homedir } from "node:os";
import { randomUUID } from "node:crypto";

// Redacted from source — override with HERMES_PROJECT in .env.local; defaults to
// <home>/hermes so the path carries no hardcoded username.
export const HERMES_PROJECT = process.env.HERMES_PROJECT || join(homedir(), "hermes");

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
import { computeQuestionGate } from "@/libs/caravan/loadquests.js";
import { escalateStalePlanQuests } from "./planStaleGuard.js";

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
  moveQuestToPlan: "Send a quest back to plan for digestion (plan = digest). Use when a downstream stage finds the quest needs re-planning — a purrview/verify bounce, or unresolved feedback/answers that must be reconciled into the objectives/deliverables before further execution. Writes a reason comment then the (DB-ungated) stage=plan write, SELECT-back verified. Input: { questId, reason }. reason is REQUIRED (it is the feedback the plan/digest pass acts on).",
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
 * Send a quest back to plan — the sanctioned "digest" bounce (plan = digest,
 * operator directive 2026-07-08). Use this when a downstream stage (purrview
 * verify, execute) discovers the quest needs re-planning: a deliverable failed
 * verification, or unresolved feedback/answers must be reconciled into the
 * objectives/deliverables before any further execution. The transition to plan is
 * DB-ungated (only forward gates carry lockphrases), so this is a plain,
 * verified stage write plus a reason comment. The plan leaf then runs the digest
 * prompt and re-itemizes. `reason` is required — it is the feedback the digest pass
 * acts on.
 */
export async function moveQuestToPlan({ questId, reason, actorName = "questmaster:moveQuestToPlan" } = {}) {
  if (!questId) return { ok: false, error: "questId is required" };
  if (typeof reason !== "string" || !reason.trim()) {
    return { ok: false, error: "reason is required — it is the feedback the plan/digest pass reconciles" };
  }
  const client = await getServiceClient();

  const { data: quest } = await client
    .from(publicTables.quests)
    .select("id, stage")
    .eq("id", questId)
    .single();
  if (!quest) return { ok: false, error: "quest not found" };

  // Comment first (durable feedback the digest pass reads), then the stage write.
  const { recordQuestComment } = await import("@/libs/quest/index.js");
  await recordQuestComment(questId, {
    source: "questmaster",
    action: "bounce_to_plan",
    summary: `Bounced ${quest.stage} → plan for digestion: ${reason}`,
    actor_name: actorName,
    verdict: "fail",
  }, { client });

  const { error: upErr } = await client
    .from(publicTables.quests)
    .update({ stage: "plan" })
    .eq("id", questId);
  if (upErr) return { ok: false, error: upErr.message };

  // SELECT-back verify the write landed.
  const { data: after } = await client
    .from(publicTables.quests)
    .select("stage")
    .eq("id", questId)
    .single();
  if (after?.stage !== "plan") {
    const out = { ok: false, error: `stage write did not persist; reads back as "${after?.stage}"`, stage: after?.stage };
    logWeaponCall({ weapon: "questmaster", fn: "moveQuestToPlan", input: { questId }, output: out, status: "fail" });
    return out;
  }

  const out = { ok: true, stage: "plan" };
  logWeaponCall({ weapon: "questmaster", fn: "moveQuestToPlan", input: { questId }, output: out, status: "ok" });
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
    ...(digest
      ? [`=== ORIENTATION DIGEST (pre-fetched — read this INSTEAD of re-calling summarizeQuest) ===`, digest, `=== END DIGEST ===`, ``]
      : []),
    `PLAN = DIGEST. The plan stage is NOT only fresh itemization — it is the RECONCILIATION stage. A quest is at plan for one of two reasons, and this leaf handles BOTH:`,
    `  • FRESH PLAN — the quest was just created and its verbose description is not yet itemized into objectives/deliverables/questions.`,
    `  • RE-PLAN / DIGEST — the quest already has objectives/deliverables but came BACK to plan because it carries un-digested inputs: a purrview/review BOUNCE (a cat-fail / verify-fail feedback comment), an unresolved USER comment, or an ANSWERED unblocking question whose decisions haven't been migrated. These MUST be reconciled into the durable quest state before it can move forward again.`,
    ``,
    `THE CANONICAL CLI — all skill-book/weapon access goes through it (resolves @/ imports; do NOT probe import paths, hand-roll node -e loaders, or hunt for alias-loader files):`,
    `  • Run an action:        node --no-warnings libs/adventurer/run-action.mjs <skillBookId> <action> '<inputJson>'`,
    `  • Read ONE procedure:   node --no-warnings libs/adventurer/run-action.mjs --howto <skillBookId> <action>   (e.g. --howto questmaster planQuest / --howto questmaster questionDigestion / --howto questmaster commentDigestion; NEVER read/grep a whole skill-book file to find a howTo)`,
    ``,
    `Run the plan/digest procedure:`,
    `1. sage.preflight (the harness injects it). ORIENT from the digest above (or questmaster.getQuestDetails for full bodies) — is this a FRESH quest (no objectives/deliverables) or a RE-PLAN (already itemized, came back with feedback)? Note every unresolved user/cat-fail comment and every answered-not-digested unblocking question.`,
    `2. DIGEST THE INPUTS FIRST — reconcile every un-digested input into the durable quest state (this is why plan = digest; do it BEFORE any itemizing/moveQuestToExecute):`,
    `   a. BOUNCE / cat-fail / verify-fail feedback comments — read the reason. Turn each failure into concrete rework: re-activate (status='active') or ADD the objective(s) that will fix it, and mark the failed deliverable(s) for re-production (the execute leaf reproduces any deliverable whose verify failed). Then mark the feedback comment resolved (questmaster.resolveQuestComment) — NEVER delete it (comments are the durable audit log).`,
    `   b. USER comments (source='user', resolved=false) — follow the directive (commentDigestion): reconcile it into the description/objectives/deliverables, then resolveQuestComment. Never delete.`,
    `   c. ANSWERED unblocking questions — DIGEST per questionDigestion's 4 steps (EXTRACT decisions → MAP+APPLY into description/objectives/deliverables → VERIFY present → remove ONLY via questmaster.digestAndRemoveQuestion with a migrationNote). NEVER raw-delete a question.`,
    `3. ITEMIZE / RE-ITEMIZE:`,
    `   • FRESH quest (no objectives/deliverables yet): read --howto questmaster planQuest, then run the scripted itemizer questmaster.planQuest({questId:"${questId}"}) — it distills the big description into objectives + quest_questions + deliverables, shrinks the description, and attempts moveQuestToExecute.`,
    `   • RE-PLAN (already itemized): do NOT re-run the whole itemizer over good rows. Surgically apply only the changes step 2's digestion implies (patchObjective, insert/modify deliverables, updateQuest description).`,
    `4. VERIFY the done-condition before you stop (getQuestDetails): the description is SHORT (global instructions, links, references only) and NON-OVERLAPPING with the sub-items; every work item exists as a verb-led objective (ruleObjectiveClarity); every final artifact is a deliverables row with a content-typed kind; every load-bearing ambiguity is an open quest_question; AND the inbox is CLEAR — no unresolved user/cat-fail comments and no answered-not-digested questions remain (if any remain, the ladder will just kick the quest back to plan).`,
    `5. TRANSITION: call questmaster.moveQuestToExecute({questId:"${questId}"}). If it stays in 'plan' (missing objectives/deliverables/assignee, or an OPEN unblocking question), fill the named gap. If the description is too vague to itemize at all, write ONE open quest_question to the user naming what you need (the quest stays in plan; the tick re-plans after the user answers). NEVER write quests.stage directly — moveQuestToExecute is the only forward transition. Do NOT start executing objectives; planning/digesting leaves the production for the execute leaf.`,
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
    : [`  (no deliverable rows were handed to you — treat the quest as derailed and bounce to plan)`];
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
    `6. Decide: if any active deliverable mismatches the description (or its text verdict is FAIL) → bounce back to plan for digestion (questmaster.moveQuestToPlan with reason naming exactly what is wrong — plan = digest, so the fail feedback is re-planned into the objectives/deliverables before re-execution); if every active deliverable matches AND has a payload ref AND a Purrview Verification value (screenshot URL for visual, PASS verdict for text) AND a recorded finding → questmaster.moveQuestToReview. Do NOT advance any deliverable whose finding is missing or generic.`,
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

<!-- GUILDOS-FILE: libs/weapon/skillBooks/buildIndex.mjs -->
```js
/**
 * skillBooks weapon — skill-index generator (Option B, 2026-07-05).
 *
 * Rebuilds each `.claude/skills/<name>/SKILL.md` as a THIN INDEX that points back
 * to the single content source in `libs/` — instead of the verbatim per-action
 * `.md` copies (which duplicate the skill-book prose and drift). The split:
 *   - INDEX  (.claude/skills/<name>/SKILL.md): frontmatter name+description
 *     (discovery — preserved as-is) + `## Actions` list + a pointer to libs +
 *     `## Weapons` pointers. Portable per runtime.
 *   - CONTENT (libs/skill_book/<name>/index.js `howTo` prose, libs/weapon/<name>):
 *     the single source of truth, read on-demand when a skill fires.
 *
 * Idempotent: reads the CURRENT SKILL.md frontmatter (to preserve the hand-tuned
 * discovery description) + the folded-weapon list (from the existing `weapons/`
 * dir), pulls the Actions list fresh from libs via tocScan, rewrites SKILL.md,
 * and deletes the now-redundant per-action `.md` + `weapons/` copies.
 *
 * Skips folders with no libs source (the harness-native meta skills:
 * question-gate, skill-book-router, success-gate) — those are authored directly.
 *
 * Usage:  node libs/weapon/skillBooks/buildIndex.mjs [--dry]
 */

import { readdirSync, readFileSync, writeFileSync, statSync, rmSync } from "node:fs";
import { join, resolve } from "node:path";
import { fileURLToPath } from "node:url";
import { scanSkillBookTocsFromDisk, scanWeaponTocsFromDisk } from "./tocScan.js";

const __dirname = fileURLToPath(new URL(".", import.meta.url));
const ROOT = resolve(__dirname, "..", "..", "..");
const SKILLS = join(ROOT, ".claude", "skills");
const SB = join(ROOT, "libs", "skill_book");
const WP = join(ROOT, "libs", "weapon");
const DRY = process.argv.includes("--dry");

// action→desc grouped by owner, from the libs content source (single source of truth).
const sbRows = scanSkillBookTocsFromDisk();
const wpRows = scanWeaponTocsFromDisk();
const sbToc = groupBy(sbRows, "skillBook");
const wpToc = groupBy(wpRows, "weapon");

function groupBy(rows, key) {
  const m = {};
  for (const r of rows) (m[r[key]] ||= []).push({ action: r.action, description: r.description });
  return m;
}
function pick(src, field) {
  const m = src.match(new RegExp(`${field}\\s*:\\s*(["'\`])((?:\\\\.|(?!\\1).)*)\\1`, "m"));
  return m ? m[2].replace(/\\(.)/g, "$1") : null;
}
function frontmatter(src) {
  const fm = src.match(/^---\s*\n[\s\S]*?\n---\s*(\n|$)/);
  return fm ? fm[0].replace(/\s*$/, "\n") : null;
}
function exists(p) { try { statSync(p); return true; } catch { return false; } }

let built = 0, skipped = 0, deleted = 0;
const report = [];

for (const name of readdirSync(SKILLS)) {
  const dir = join(SKILLS, name);
  if (!exists(dir) || !statSync(dir).isDirectory()) continue;
  const skillMd = join(dir, "SKILL.md");
  if (!exists(skillMd)) continue;

  // Resolve the libs content source + action list.
  let sourceKind, sourcePath, title, actions, runHint;
  if (exists(join(SB, name, "index.js"))) {
    sourceKind = "skill_book";
    sourcePath = `libs/skill_book/${name}/index.js`;
    const s = readFileSync(join(SB, name, "index.js"), "utf8");
    title = pick(s, "title") || name;
    actions = sbToc[name] || [];
    runHint = `node libs/adventurer/run-action.mjs --howto ${name} <action>`;
  } else if (exists(join(WP, name, "index.js"))) {
    sourceKind = "weapon";
    sourcePath = `libs/weapon/${name}/index.js`;
    const s = readFileSync(join(WP, name, "index.js"), "utf8");
    title = pick(s, "title") || name;
    actions = wpToc[name] || [];
    runHint = `node libs/adventurer/run-action.mjs ${name} <action> '<inputJson>'`;
  } else {
    skipped++;                          // no libs source → native meta skill, leave alone
    continue;
  }

  // Preserve the hand-tuned discovery frontmatter; fall back to a minimal one.
  const cur = readFileSync(skillMd, "utf8");
  const fm = frontmatter(cur) || `---\nname: ${name}\ndescription: ${title}\n---\n`;

  // Folded weapons = existing weapons/<W>.md basenames (preserve the fold).
  const weaponsDir = join(dir, "weapons");
  const folded = exists(weaponsDir)
    ? readdirSync(weaponsDir).filter(f => f.endsWith(".md")).map(f => f.replace(/\.md$/, ""))
    : [];

  const lines = [];
  lines.push(fm.trimEnd(), "");
  lines.push(`# ${title}`, "");
  lines.push(
    `> **Index only.** This skill is a discovery pointer. The full procedure for each`,
    `> action lives in the single content source \`${sourcePath}\`. Load it on demand:`,
    `> \`${runHint}\` (or read the source directly).`,
    ""
  );
  if (actions.length) {
    lines.push("## Actions", "");
    for (const a of actions) {
      const d = (a.description || "").replace(/\n+/g, " ").trim();
      lines.push(`- **${a.action}**${d ? ` — ${d}` : ""}`);
    }
    lines.push("");
  }
  if (folded.length) {
    lines.push("## Weapons", "");
    lines.push(`Folded-in executable weapons (code + toc in \`libs/weapon/\`; invoke via run-action):`, "");
    for (const w of folded) {
      const wt = (wpToc[w] || []).slice(0, 3).map(x => x.action).join(", ");
      lines.push(`- **${w}** — \`libs/weapon/${w}/\`${wt ? ` (e.g. ${wt})` : ""}`);
    }
    lines.push("");
  }
  const out = lines.join("\n").replace(/\n{3,}/g, "\n\n");

  if (DRY) {
    report.push(`[dry] ${name}: ${actions.length} actions, ${folded.length} weapons`);
    built++;
    continue;
  }

  writeFileSync(skillMd, out, "utf8");
  // Delete the now-redundant content copies: every .md except SKILL.md, and weapons/.
  for (const f of readdirSync(dir)) {
    const p = join(dir, f);
    if (f === "SKILL.md") continue;
    if (f === "weapons" && exists(p) && statSync(p).isDirectory()) { rmSync(p, { recursive: true, force: true }); deleted++; continue; }
    if (f.endsWith(".md")) { rmSync(p, { force: true }); deleted++; }
  }
  built++;
  report.push(`${name}: index rebuilt (${actions.length} actions, ${folded.length} weapons)`);
}

console.log(report.join("\n"));
console.log(`\n${DRY ? "[DRY] " : ""}Rebuilt ${built} skill indexes · skipped ${skipped} (no libs source) · removed ${deleted} redundant copies`);
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/skillBooks/cli-ownership.mjs -->
```js
/**
 * skillBooks weapon — ownership-map CLI entrypoint.
 *
 * Source of truth = the frontmatter `description` + `## Actions`/`## Weapons`
 * bullets in each `.claude/skills/<name>/SKILL.md` (refactor 2026-07-04).
 * Discovery = directory scan (no hand-maintained hub-file lookup).
 *
 * Usage:
 *   node libs/weapon/skillBooks/cli-ownership.mjs            → markdown to stdout
 *   node libs/weapon/skillBooks/cli-ownership.mjs --json     → JSON to stdout
 *   node libs/weapon/skillBooks/cli-ownership.mjs --audit    → list entries with thin descriptions
 *
 * Runtime (`dumpOwnershipMap`) + CLI share `tocScan.scanClaudeSkillsFromDisk`
 * so their outputs cannot drift.
 */

import { scanClaudeSkillsFromDisk } from "./tocScan.js";

const args = process.argv.slice(2);
const wantJson = args.includes("--json");
const wantAudit = args.includes("--audit");

const rows = scanClaudeSkillsFromDisk();

if (wantAudit) {
  const thin = rows.filter(r => !r.description || r.description.length <= 30);
  const empty = rows.filter(r => !r.description);
  console.log(`Total TOC entries: ${rows.length}`);
  console.log(`Skill books represented: ${new Set(rows.map(r => r.skillBook)).size}`);
  console.log(`Thin descriptions (≤30 chars): ${thin.length}`);
  console.log(`Empty descriptions: ${empty.length}`);
  console.log("");
  if (thin.length) {
    console.log("--- Thin / empty entries (fix in source skill book) ---");
    for (const r of thin.slice(0, 60)) {
      console.log(`  ${r.skillBook}.${r.action} → ${JSON.stringify(r.description || "")}`);
    }
    if (thin.length > 60) console.log(`  ... and ${thin.length - 60} more`);
  }
  process.exit(0);
}

if (wantJson) {
  console.log(JSON.stringify({ rows, generatedAt: new Date().toISOString() }, null, 2));
  process.exit(0);
}

const lines = [
  "# Ownership map (generated from .claude/skills SKILL.md)",
  "",
  `> Auto-generated by \`libs/weapon/skillBooks/cli-ownership.mjs\` via directory-scan. Source of truth = the frontmatter \`description\` + \`## Actions\`/\`## Weapons\` bullets in each \`.claude/skills/<name>/SKILL.md\`. Discovery = filesystem walk, no hub-file maintenance required. To fix an entry, edit its description in that SKILL.md.`,
  "",
  `Generated: ${new Date().toISOString()} · ${rows.length} entries across ${new Set(rows.map(r => r.skillBook)).size} skills`,
  "",
  "| Skill | Action | Owns / describes |",
  "|---|---|---|",
];
for (const r of rows) {
  const desc = (r.description || "").replace(/\|/g, "\\|").replace(/\n/g, " ").trim() || "_(no description)_";
  lines.push(`| \`${r.skillBook}\` | \`${r.action}\` | ${desc} |`);
}
console.log(lines.join("\n"));
process.exit(0);
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/skillBooks/cli.mjs -->
```js
/**
 * skillBooks weapon — CLI entrypoint.
 *
 * The skillBooks weapon is a function module (`gearUp`), normally
 * called from app/agent context. The skill-book-router skill needs to invoke
 * it as a process. Naive `node -e` fails on the `@/` path alias and the
 * registry's Next-style extensionless imports (documented failure mode in
 * guildmaster.evolutionLoop). That resolution bootstrap is always-on plumbing
 * hidden from the agent, so per blacksmith.refForgeDiscipline it lives WITH
 * the weapon, not as a satellite script. The skill calls this; this calls the
 * weapon. One-way reference, weapon stays the source of truth.
 *
 * Usage: node libs/weapon/skillBooks/cli.mjs "<pending action description>"
 */

import { register } from "node:module";
import { pathToFileURL } from "node:url";
import { resolve } from "node:path";

const projectRoot = resolve(import.meta.dirname, "..", "..", "..");
register("data:text/javascript," + encodeURIComponent(`
  import { pathToFileURL } from "node:url";
  import { existsSync } from "node:fs";
  const root = ${JSON.stringify(projectRoot.replace(/\\/g, "/"))};
  // Next.js bundler tolerates extensionless file AND directory imports
  // (e.g. "@/libs/council/profileEnvVars" or "@/libs/weapon/zoho").
  // Plain Node ESM does not — mimic the standard resolution candidates.
  const CANDIDATES = [".js", ".mjs", ".cjs", "/index.js", "/index.mjs"];
  function toPath(url) {
    try { return decodeURIComponent(new URL(url).pathname).replace(/^\\/([A-Za-z]:)/, "$1"); }
    catch { return null; }
  }
  export async function resolve(specifier, context, next) {
    let spec = specifier;
    if (spec.startsWith("@/")) spec = pathToFileURL(root + "/" + spec.slice(2)).href;
    try {
      return await next(spec, context);
    } catch (err) {
      if (!err || (err.code !== "ERR_UNSUPPORTED_DIR_IMPORT" && err.code !== "ERR_MODULE_NOT_FOUND")) throw err;
      const base = (err.url || spec).replace(/\\/$/, "");
      const basePath = toPath(base);
      if (basePath) {
        for (const ext of CANDIDATES) {
          if (existsSync(basePath + ext)) return await next(base + ext, context);
        }
      }
      throw err;
    }
  }
`), pathToFileURL(projectRoot + "/"));

const rawArgs = process.argv.slice(2);
const withContext = rawArgs.includes("--with-context");
const taskDescription = rawArgs.filter(a => a !== "--with-context").join(" ").trim();

if (!taskDescription) {
  console.error('Usage: node libs/weapon/skillBooks/cli.mjs "<pending action description>" [--with-context]');
  process.exit(2);
}

const { gearUp } = await import("@/libs/weapon/skillBooks/index.js");

const gearResult = gearUp({
  taskDescription,
  withContext,
  agent: "skill-book-router",
});
// gearUp returns a Promise when withContext=true, plain object otherwise
const { suggestions = [], error, contextSnippets } = withContext ? await gearResult : gearResult;

if (error) {
  console.error(`scan error: ${error}`);
  process.exit(1);
}

if (suggestions.length === 0) {
  console.log("NO_MATCH — no skill book scored against this task. Proceed normally.");
  process.exit(0);
}

const top = suggestions.slice(0, 5);
console.log(`MATCHES (${top.length} of ${suggestions.length} scored — .claude/skills) for: ${taskDescription}\n`);
for (const s of top) {
  console.log(`# ${s.skillBook}  [SKILL]  (totalScore=${s.totalScore})`);
  for (const a of s.actions.sort((x, y) => y.score - x.score).slice(0, 4)) {
    console.log(`  - ${a.action} [${a.score}]: ${a.toc.slice(0, 160)}`);
  }
  console.log("");
}
const t0 = top[0];
console.log(
  `NEXT: read .claude/skills/${t0.skillBook}/SKILL.md, then open the matching action file ` +
    `.claude/skills/${t0.skillBook}/<action>.md for its howTo. If the action needs runtime ` +
    `logic, its executable weapon is invocable via ` +
    `node --no-warnings libs/adventurer/run-action.mjs ${t0.skillBook} <action> '<inputJson>'. ` +
    `If none of the above fits the pending action, proceed normally.`,
);

// When --with-context, append recall snippets as supplemental context
if (contextSnippets && contextSnippets.length > 0) {
  console.log(`\n── CONTEXT SNIPPETS (from docs + memory) ──\n`);
  for (const s of contextSnippets) {
    console.log(`[${s.sourceId}] ${s.docPath} § ${s.section}  (score=${s.score.toFixed(2)}, ~${s.tokenEstimate}tok)`);
    const lines = s.text.split("\n").slice(0, 3);
    for (const l of lines) console.log(`  ${l}`);
    if (s.text.split("\n").length > 3) console.log(`  …`);
    console.log("");
  }
}

process.exit(0);
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/skillBooks/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/weapon/skillBooks`

Rationale layer (per CLAUDE.md §6). The WHY behind this module's design — not API
docs, not procedure. Append; supersede in place when a decision changes.

---

## 2026-06-13 — `gearUp` scans skill books AND weapons (directory-driven)

**Decision.** `gearUp` ranks weapon `toc` actions alongside skill-book actions,
each suggestion tagged `kind: "skill_book" | "weapon"`. Weapon TOCs are read
from disk via `scanWeaponTocsFromDisk()` (`tocScan.js`), mirroring the existing
skill-book scan — directory-driven, so a new weapon folder is picked up with zero
edits here.

**Rejected alternative.** Scanning only skill books (the original), or maintaining
a hand-written weapon hub to scan.

**Why.** Capabilities live in BOTH trees, but discovery only covered one half —
so weapon actions (e.g. `openai_images.generate`) were invisible to the registry
scan, and agents that needed them had to already know the id/action or go
guessing. Disk-driven scan avoids a dual-maintained hub (same architecture call
as the skill-book scanner).

**Consequence if violated.** Half the capability surface is undiscoverable via the
preflight scan → agents maze on weapon-backed tasks (the 2026-06-13 image-gen
maze; full rationale in `libs/adventurer/engineering_decisions.md`).

---

## 2026-07-05 — `.claude/skills` is a GENERATED thin INDEX, not a content copy (supersedes the copy approach in the 2026-07-04 entry below)

**Decision.** Each `.claude/skills/<name>/SKILL.md` is a thin INDEX: preserved
frontmatter (name + hand-tuned discovery `description`) + a `## Actions` list
generated from the libs `toc` + a pointer to the libs content source +
`## Weapons` pointers. The per-action `<action>.md` content files (verbatim howTo
copies) and the `weapons/<W>.md` docs were DELETED. Content lives ONLY in
`libs/skill_book/<name>/index.js` (howTo prose) and `libs/weapon/<name>`
(executable). `libs/weapon/skillBooks/buildIndex.mjs` regenerates every index from
libs, preserving frontmatter and the folded-weapon list; blacksmith's forge +
patchSkillBook now write to libs then run buildIndex.

**Rejected alternative.** (a) Keep the verbatim per-action `.md` copies in
`.claude/skills` (what the initial 2026-07-04 build did). (b) Copy the skills into
each runtime (e.g. Cursor equivalents) per guild.

**Why.** The verbatim copies duplicated the skill-book prose in two places →
guaranteed drift, dual maintenance. The index-and-point model keeps ONE content
source (libs) and makes the discovery layer a generated, disposable artifact.
Bonus: porting to another runtime/guild (Cursor `.mdc`, `AGENTS.md`) becomes a
second `buildIndex` output target over the SAME repo — port the thin index, not
the content. Footprint dropped 593 → 83 md files (~502 copies removed) with
discovery (gearUp/ownership: 698 rows/83 skills) unchanged.

**Consequence if violated.** If a future agent hand-writes howTo prose back into
`.claude/skills/<name>/*.md`, it re-forks the content and buildIndex will DELETE it
on the next run (silent data loss of the hand-edit). Edit howTo in `libs/`, then
regenerate. The frontmatter `description` is the ONE field authored in
`.claude/skills` (buildIndex preserves it) — it's the per-runtime discovery index.

---

## 2026-07-04 — `.claude/skills` is the canonical discovery registry; libs/ keeps the executable code

**Decision.** Discovery/routing now scans the `.claude/skills/<name>/SKILL.md`
tree, not the JS `toc` blocks. `scanClaudeSkillsFromDisk` (`tocScan.js`) parses
each SKILL.md's frontmatter `description` + `## Actions`/`## Weapons`/`## Functions`
bullets into the same `{skillBook, action, description}` rows the rest of the
system already consumes; `gearUp`, `dumpOwnershipMap`, `cli-ownership.mjs`, and the
preflight hook's `listSkillBooks` all read this one source. The former
`libs/skill_book/*/index.js` `toc` scan and the separate `libs/weapon/*` `toc`
scan (`scanSkillBookTocsFromDisk` / `scanWeaponTocsFromDisk`) are retained but no
longer wired into discovery. Every skill book + weapon was mirrored into
`.claude/skills` (folded weapons live under their owner skill's `weapons/`;
standalone weapons are their own skill folder).

**Rejected alternative.** (a) Keep discovery on the JS `toc` blocks and treat
`.claude/skills` as docs-only. (b) Go the whole way — delete `libs/skill_book`
prose and have adventurer EXECUTION parse the `.md` files too.

**Why.** The operator wants GuildOS capabilities to BE native Claude Code skills
(auto-loaded by the harness from `.claude/skills`), so the SKILL.md tree must be
the source of truth — one registry, not two that drift. But executable actions
(`libs/skill_book/<name>` `adventurerActions` loaded by `getSkillBook`; every
weapon's integration code invoked by `run-action`) are CODE, not markdown — they
cannot move into `.md`. So the split is deliberate: **markdown = discovery +
howTo prose; libs/ = executable code.** A folded weapon has a discovery doc under
its owner skill AND its real code in `libs/weapon`.

**Consequence if violated.** If a future agent "simplifies" by pointing discovery
back at the JS `toc` blocks, new skills authored only as `.claude/skills` folders
(the blacksmith's new forge target) become invisible to `gearUp`/preflight. If it
instead deletes `libs/skill_book`/`libs/weapon` to "finish the migration,"
adventurer/quest execution and every weapon call break — the `.md` files are not
executable. Keep both trees; keep discovery on `.claude/skills`.

**Follow-on (2026-07-05) — the preflight hook must NOT re-dump the registry.**
Once skills live in `.claude/skills`, the Claude Code harness ALREADY lists every
skill's name+description in the system prompt (native progressive disclosure —
free; the body loads only when the Skill tool invokes it). So
`.claude/hooks/preflight.mjs` was trimmed to STOP injecting the full per-skill
description dump (it had ballooned to ~5k tokens/turn — ~4,979 for the registry
block alone). It now injects only the gate text + a pointer to the native list +
the `gearUp` CLI + weapon IDs (~1.2k tok/turn total, down from ~5.8k). **Do NOT
re-add `bookLines = books.map(b => \`- ${b.id}: ${b.description}\`)` to the hook
output** — that re-duplicates the free native list and is pure per-turn waste. The
`listSkillBooks()` count is still read (cheap, for the header); only the dump is
gone. Realizing "skill-picking is free" was the whole point of moving to
`.claude/skills` — the hook dump silently defeated it.

---

## (standing) — TOC is the two-tier discovery mechanism; keep `description` thin

**Decision.** A `toc` entry's `description` is the **strategic** layer `gearUp`
scans across the whole registry in one pass; the `howTo` is the **tactical**
layer loaded only for the chosen action. Keep descriptions short and
intent-bearing (verb + noun + when-to-use), not procedure.

**Rejected alternative.** Fat descriptions that inline the how, or rationale; a
parallel discovery doc.

**Why.** Cheap reliable discovery = O(all descriptions) once + O(one howTo) after.
Bloating the description inflates every scan and blurs the strategic/tactical
split; putting rationale here (vs `engineering_decisions.md`) makes the scan
noisier without helping discovery.

**Consequence if violated.** `gearUp` ranking degrades (noise tokens), and the
two-tier efficiency that makes the preflight scan affordable erodes.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/skillBooks/index.js -->
```js
/**
 * skillBooks weapon — discovery and capability layer over the skill book registry.
 *
 * Used by GM-dispatched SAs to:
 *   - scanSMARTPlans({ userMessage }) → which [SMART]-prefixed actions match a user message
 *   - gearUp({ taskDescription }) → "gear up at the town stalls before the quest"; rank skill books for a task
 *
 * Both are noise-absorbing functions: they read all TOCs internally and surface
 * only the matches with their TOC descriptions. The full howTos are never returned.
 */

import { SKILL_BOOKS } from "@/libs/skill_book/index.js";
import { scanClaudeSkillsFromDisk } from "./tocScan.js";
import { logWeaponCall } from "../weaponLog/index.js";

const SMART_PREFIX = "[SMART]";

// ── helpers ──────────────────────────────────────────────────────────────────

function tocLine(entry) {
  if (typeof entry === "string") return entry;
  if (typeof entry?.description === "string") return entry.description;
  return "";
}

function tokenize(text) {
  return text
    .toLowerCase()
    .replace(/[^\p{L}\p{N}\s]/gu, " ")
    .split(/\s+/)
    .filter(t => t.length >= 3);
}

function scoreMatch(userTokens, tocText) {
  const haystack = tocText.toLowerCase();
  let hits = 0;
  for (const tok of userTokens) {
    if (haystack.includes(tok)) hits++;
  }
  return hits;
}

// ── scanSMARTPlans ───────────────────────────────────────────────────────────

/**
 * Scan every [SMART]-prefixed TOC entry across all skill books. Score matches
 * against the user message by simple token overlap. Return ranked candidates.
 *
 * @param {{ userMessage: string, agent?: string }} opts
 * @returns {{ matches: Array<{ skillBook, action, toc, score }> }}
 */
export function scanSMARTPlans({ userMessage, agent = "GM" }) {
  if (!userMessage || typeof userMessage !== "string") {
    const result = { matches: [], error: "userMessage required" };
    logWeaponCall({ weapon: "skillBooks", fn: "scanSMARTPlans", agent, input: { messageLength: 0 }, output: result, status: "error" });
    return result;
  }

  const userTokens = tokenize(userMessage);
  const candidates = [];

  for (const [bookId, book] of Object.entries(SKILL_BOOKS)) {
    const toc = book?.toc;
    if (!toc) continue;
    for (const [actionKey, entry] of Object.entries(toc)) {
      const line = tocLine(entry);
      if (!line.trim().startsWith(SMART_PREFIX)) continue;
      const score = scoreMatch(userTokens, line);
      candidates.push({ skillBook: bookId, action: actionKey, toc: line, score });
    }
  }

  // Sort by score desc, drop zeros
  const matches = candidates
    .filter(c => c.score > 0)
    .sort((a, b) => b.score - a.score);

  const result = { matches, totalScanned: candidates.length };
  logWeaponCall({
    weapon: "skillBooks",
    fn: "scanSMARTPlans",
    agent,
    input: { messageLength: userMessage.length, userTokens: userTokens.length },
    output: { matchCount: matches.length, totalScanned: candidates.length },
    status: "ok",
  });
  return result;
}

// ── gearUp ───────────────────────────────────────────────────────────────────

/**
 * Gear up the adventurer for the quest. Reads every skill book's TOC and ranks
 * the most relevant skill books for a task description by token overlap —
 * the equivalent of stopping at the town square's stalls before heading out
 * the gates: which books, which weapons, which supplies for this run?
 *
 * @param {{ taskDescription: string, withContext?: boolean, contextBudget?: number, agent?: string }} opts
 * @returns {{ suggestions: Array<{ skillBook, actions: Array<{action, toc, score}>, totalScore }>, contextSnippets?: Array }}
 */
export function gearUp({ taskDescription, withContext = false, contextBudget = 2000, agent = "GM" }) {
  if (!taskDescription || typeof taskDescription !== "string") {
    return { suggestions: [], error: "taskDescription required" };
  }

  const tokens = tokenize(taskDescription);
  const perBook = {};

  // Canonical discovery source (refactor 2026-07-04): the `.claude/skills/<name>/`
  // tree. Each folder's SKILL.md frontmatter description + `## Actions` / `## Weapons`
  // / `## Functions` bullets are scanned as the toc. This is the single scan that
  // replaces BOTH the old SKILL_BOOKS-hub loop and the separate libs/weapon disk
  // scan — folded weapons already live under their owning skill's `## Weapons`, and
  // standalone weapons are their own skill folder, so one scan covers everything.
  // Executable JS still lives in libs/; a matched skill is READ (SKILL.md +
  // <action>.md), and if it has runtime logic the leaf invokes the paired weapon.
  for (const r of scanClaudeSkillsFromDisk()) {
    const score = scoreMatch(tokens, r.description || "");
    if (score <= 0) continue;
    const key = r.skillBook;
    if (!perBook[key]) perBook[key] = { skillBook: key, kind: "skill", actions: [], totalScore: 0 };
    // The `(skill)` synthetic row carries the frontmatter description — surface it
    // as the skill's own line but don't list it as an invokable action.
    if (r.action !== "(skill)") perBook[key].actions.push({ action: r.action, toc: r.description, score });
    perBook[key].totalScore += score;
  }

  const suggestions = Object.values(perBook).sort((a, b) => b.totalScore - a.totalScore);

  logWeaponCall({
    weapon: "skillBooks",
    fn: "gearUp",
    agent,
    input: { taskLength: taskDescription.length, tokens: tokens.length, withContext },
    output: { skillCount: suggestions.length },
    status: "ok",
  });

  // Optional: enrich with recall context snippets from docs + memory corpora.
  // Skips the 'registry' corpus (gearUp already covers TOC scoring) to avoid
  // duplicating information; pulls from docs/ and memory/ for supplemental context.
  // When withContext=true, returns a Promise; otherwise returns a plain object
  // (backward-compatible with all existing sync callers).
  if (withContext) {
    return import("../recall/index.js").then(({ recall }) => {
      let contextSnippets;
      try {
        const { snippets } = recall({
          query: taskDescription,
          corpora: ["docs", "memory"],
          topK: 5,
          tokenBudget: contextBudget,
          agent,
        });
        contextSnippets = snippets;
      } catch { contextSnippets = []; }
      return { suggestions, contextSnippets };
    }).catch(() => ({ suggestions, contextSnippets: [] }));
  }

  return { suggestions };
}

// ── dumpOwnershipMap ─────────────────────────────────────────────────────────

/**
 * Dump every TOC entry across every skill book as a flat ownership map. The
 * single source of truth for "which concept lives in which skill book action"
 * is the description text of the TOC entries themselves — this action just
 * concatenates them and emits a markdown table (or JSON for programmatic use).
 *
 * Replaces the hand-maintained CLAUDE.md "Ownership map" table (removed in
 * commit df4b5b5). When a description is too thin to stand in for ownership
 * info, fix the description in the source skill book — do NOT add a parallel
 * ownership doc.
 *
 * @param {{ format?: 'markdown'|'json', agent?: string }} opts
 * @returns {{ rows, markdown, format }} — rows always present; markdown only when format='markdown'
 */
export function dumpOwnershipMap({ format = "markdown", agent = "GM" } = {}) {
  // Directory-driven scan of the canonical `.claude/skills/<name>/SKILL.md` tree
  // (refactor 2026-07-04). Adding a new skill folder = it shows up here
  // automatically. Rows come back in the same {skillBook, action, description}
  // shape the markdown renderer already expects.
  const rows = scanClaudeSkillsFromDisk();

  let markdown;
  if (format === "markdown") {
    const lines = [
      "# Ownership map (generated from .claude/skills SKILL.md)",
      "",
      `> Auto-generated by \`libs/weapon/skillBooks.dumpOwnershipMap\`. Source of truth = the frontmatter \`description\` + \`## Actions\`/\`## Weapons\` bullets in each \`.claude/skills/<name>/SKILL.md\`. To fix an entry, edit its description in that SKILL.md — do NOT edit this map.`,
      "",
      "| Skill book | Action | Owns / describes |",
      "|---|---|---|",
    ];
    for (const r of rows) {
      const desc = (r.description || "").replace(/\|/g, "\\|").replace(/\n/g, " ").trim() || "_(no description)_";
      lines.push(`| \`${r.skillBook}\` | \`${r.action}\` | ${desc} |`);
    }
    markdown = lines.join("\n");
  }

  const result = { rows, format };
  if (markdown) result.markdown = markdown;

  logWeaponCall({
    weapon: "skillBooks",
    fn: "dumpOwnershipMap",
    agent,
    input: { format },
    output: { rowCount: rows.length, bookCount: new Set(rows.map(r => r.skillBook)).size },
    status: "ok",
  });

  return result;
}

export const toc = {
  scanSMARTPlans: "Scan all skill book TOCs for [SMART]-prefixed actions matching a user message. Returns ranked matches. Input: {userMessage}. Output: {matches: [{skillBook, action, toc, score}]}.",
  gearUp: "Gear up the adventurer for a quest — rank the most relevant skill books (and their actions) for a given task description by token-overlap against every TOC entry. The town-square step before any run: which books / weapons / supplies for this job? Input: {taskDescription}. Output: {suggestions: [{skillBook, actions, totalScore}]}.",
  dumpOwnershipMap: "Generate the ownership map by reading every TOC across every skill book. Replaces the hand-maintained CLAUDE.md table. Input: {format?: 'markdown'|'json'}. Output: {rows, markdown?, format}. To fix an ownership entry, edit its description in the source skill book — never edit a parallel doc.",
  buildSkillIndex: "Regenerate the .claude/skills/<name>/SKILL.md discovery indexes from the libs content source (index-and-point model). Preserves each skill's frontmatter + folded-weapon list, rebuilds the Actions list from the libs toc, and deletes the redundant per-action content copies. Run after editing any skill-book/weapon content. CLI: node libs/weapon/skillBooks/buildIndex.mjs [--dry].",
};
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/skillBooks/tocScan.js -->
```js
/**
 * Skill book TOC scanner — directory-driven, NOT registry-driven.
 *
 * Reads `libs/skill_book/<name>/index.js` directly from disk and extracts each
 * skill book's top-level `toc: { ... }` (or `export const toc = { ... }`) block
 * via brace-balanced regex parsing.
 *
 * Why directory-driven (per user architecture call 2026-05-28):
 *   The hand-maintained `libs/skill_book/index.js` hub (which imports each
 *   skill book individually) requires an edit every time a new skill book is
 *   added. That's dual-maintenance with the per-skill-book TOC descriptions
 *   and "just bad architecture all around" — the user opposes it.
 *
 *   Auto-discovery via fs.readdirSync removes that maintenance burden. Adding
 *   a new skill book = create the folder + `index.js` with a `toc` block; the
 *   scanner picks it up on the next call. No hub-file edit needed.
 *
 * Used by:
 *   - `dumpOwnershipMap` in this weapon's `index.js`
 *   - `cli-ownership.mjs` CLI (bare-node invocation)
 *
 * Both code paths share this function so the runtime + CLI outputs cannot
 * drift.
 */

import { readdirSync, readFileSync, statSync } from "node:fs";
import { join, resolve } from "node:path";
import { fileURLToPath } from "node:url";

// Walk from this file (libs/weapon/skillBooks/tocScan.js) up three levels to
// reach the project root, then into libs/skill_book.
const __dirname = fileURLToPath(new URL(".", import.meta.url));
const PROJECT_ROOT = resolve(__dirname, "..", "..", "..");
export const SKILL_BOOKS_DIR = join(PROJECT_ROOT, "libs", "skill_book");
export const WEAPONS_DIR = join(PROJECT_ROOT, "libs", "weapon");
// Canonical registry (refactor 2026-07-04): the `.claude/skills/<name>/SKILL.md`
// tree is the source of truth for capability DISCOVERY. Each skill book + weapon
// was mirrored here (TOC → SKILL.md frontmatter + `## Actions`; folded weapons →
// `## Weapons`). Executable JS actions still live in libs/skill_book|libs/weapon
// and are loaded by getSkillBook for adventurer/quest EXECUTION — but gearUp /
// ownership / preflight discovery now scan THIS tree, not the JS `toc` blocks.
export const CLAUDE_SKILLS_DIR = join(PROJECT_ROOT, ".claude", "skills");

/**
 * @returns {Array<{name: string, path: string}>} per-folder index.js paths
 */
export function listSkillBookFiles(dir = SKILL_BOOKS_DIR) {
  const out = [];
  let entries;
  try { entries = readdirSync(dir); } catch { return out; }
  for (const name of entries) {
    const p = join(dir, name);
    let s;
    try { s = statSync(p); } catch { continue; }
    if (!s.isDirectory()) continue;
    const idx = join(p, "index.js");
    try { statSync(idx); out.push({ name, path: idx }); } catch {}
  }
  return out;
}

// Walk forward from `openIdx` (pointing at '{') to its matching '}',
// respecting strings, template literals, and comments.
function extractBalancedBlock(src, openIdx) {
  let depth = 0;
  let i = openIdx;
  const n = src.length;
  while (i < n) {
    const ch = src[i];
    if (ch === "/" && src[i + 1] === "/") {
      const nl = src.indexOf("\n", i);
      i = nl < 0 ? n : nl + 1;
      continue;
    }
    if (ch === "/" && src[i + 1] === "*") {
      const end = src.indexOf("*/", i + 2);
      i = end < 0 ? n : end + 2;
      continue;
    }
    if (ch === "'" || ch === '"' || ch === "`") {
      const quote = ch;
      i++;
      while (i < n) {
        if (src[i] === "\\") { i += 2; continue; }
        if (src[i] === quote) { i++; break; }
        if (quote === "`" && src[i] === "$" && src[i + 1] === "{") {
          let d = 1; i += 2;
          while (i < n && d > 0) {
            if (src[i] === "{") d++;
            else if (src[i] === "}") d--;
            i++;
          }
          continue;
        }
        i++;
      }
      continue;
    }
    if (ch === "{") depth++;
    else if (ch === "}") {
      depth--;
      if (depth === 0) return src.slice(openIdx, i + 1);
    }
    i++;
  }
  return null;
}

function parseTocBlock(inner, out) {
  let i = 0;
  const n = inner.length;
  while (i < n) {
    while (i < n) {
      const ch = inner[i];
      if (/\s/.test(ch) || ch === ",") { i++; continue; }
      if (ch === "/" && inner[i + 1] === "/") { const nl = inner.indexOf("\n", i); i = nl < 0 ? n : nl + 1; continue; }
      if (ch === "/" && inner[i + 1] === "*") { const e = inner.indexOf("*/", i + 2); i = e < 0 ? n : e + 2; continue; }
      break;
    }
    if (i >= n) break;
    let action = "";
    if (inner[i] === "'" || inner[i] === '"') {
      const q = inner[i]; i++;
      while (i < n && inner[i] !== q) { if (inner[i] === "\\") i++; action += inner[i] || ""; i++; }
      i++;
    } else {
      while (i < n && /[A-Za-z0-9_$]/.test(inner[i])) { action += inner[i]; i++; }
    }
    while (i < n && /\s/.test(inner[i])) i++;
    if (inner[i] !== ":") {
      while (i < n && inner[i] !== ",") i++;
      continue;
    }
    i++;
    while (i < n && /\s/.test(inner[i])) i++;
    let description = "";
    if (inner[i] === "'" || inner[i] === '"' || inner[i] === "`") {
      const q = inner[i]; i++;
      while (i < n) {
        if (inner[i] === "\\") { description += inner[i + 1] || ""; i += 2; continue; }
        if (inner[i] === q) { i++; break; }
        description += inner[i]; i++;
      }
    } else if (inner[i] === "{") {
      const block = extractBalancedBlock(inner, i);
      if (block) {
        const dm = block.match(/description\s*:\s*(["'`])((?:\\.|(?!\1).)*)\1/);
        if (dm) description = dm[2].replace(/\\(.)/g, "$1");
        i += block.length;
      }
    }
    if (action) out.push({ action, description: description.trim() });
    while (i < n && inner[i] !== ",") {
      if (inner[i] === "{") {
        const b = extractBalancedBlock(inner, i);
        if (b) { i += b.length; continue; }
      }
      i++;
    }
    if (inner[i] === ",") i++;
  }
}

function extractTocEntries(source) {
  const out = [];
  const re = /\btoc\s*[:=]\s*\{/g;
  let m;
  while ((m = re.exec(source)) !== null) {
    const start = m.index + m[0].length - 1;
    const block = extractBalancedBlock(source, start);
    if (!block) continue;
    parseTocBlock(block.slice(1, -1), out);
    break; // only first toc block per file
  }
  return out;
}

/**
 * Scan every `libs/skill_book/<name>/index.js` and emit ownership rows.
 * Directory-driven — adding a new skill book folder requires zero edits
 * anywhere else.
 *
 * @returns {Array<{skillBook: string, action: string, description: string}>}
 */
export function scanSkillBookTocsFromDisk() {
  const files = listSkillBookFiles();
  const rows = [];
  for (const f of files) {
    let src;
    try { src = readFileSync(f.path, "utf8"); }
    catch { continue; }
    const entries = extractTocEntries(src);
    for (const e of entries) {
      rows.push({ skillBook: f.name, action: e.action, description: e.description });
    }
  }
  rows.sort((a, b) => (a.skillBook + a.action).localeCompare(b.skillBook + b.action));
  return rows;
}

/**
 * Scan every `libs/weapon/<name>/index.js` and emit its `toc` entries — the
 * weapon-side mirror of scanSkillBookTocsFromDisk. Weapons expose their actions
 * via the same `toc` convention (action → description string), so gearUp can
 * rank weapon actions alongside skill-book actions. Directory-driven: a new
 * weapon folder is picked up with zero edits here.
 *
 * @returns {Array<{weapon: string, action: string, description: string}>}
 */
export function scanWeaponTocsFromDisk() {
  const files = listSkillBookFiles(WEAPONS_DIR);
  const rows = [];
  for (const f of files) {
    let src;
    try { src = readFileSync(f.path, "utf8"); }
    catch { continue; }
    for (const e of extractTocEntries(src)) {
      rows.push({ weapon: f.name, action: e.action, description: e.description });
    }
  }
  rows.sort((a, b) => (a.weapon + a.action).localeCompare(b.weapon + b.action));
  return rows;
}

// ── .claude/skills scanner (canonical discovery source, refactor 2026-07-04) ──

/**
 * List every `.claude/skills/<name>/SKILL.md`.
 * @returns {Array<{name: string, path: string, dir: string}>}
 */
export function listClaudeSkillFiles(dir = CLAUDE_SKILLS_DIR) {
  const out = [];
  let entries;
  try { entries = readdirSync(dir); } catch { return out; }
  for (const name of entries) {
    const p = join(dir, name);
    let s;
    try { s = statSync(p); } catch { continue; }
    if (!s.isDirectory()) continue;
    const md = join(p, "SKILL.md");
    try { statSync(md); out.push({ name, path: md, dir: p }); } catch {}
  }
  return out;
}

/**
 * Parse a SKILL.md into { name, description, actions: [{action, description}] }.
 * - Frontmatter `description:` (plain or `>-`/`|` block) → the skill-level line.
 * - Bullets under `## Actions`, `## Weapons`, `## Functions` → per-action lines.
 *   Recognises `- [key](file) — desc`, `- **key** — desc`, and `- key — desc`
 *   (em-dash `—` or ` - ` hyphen separator).
 */
export function parseSkillMd(src, skillName) {
  const out = { name: skillName, description: "", actions: [] };

  // Frontmatter block between the first two `---` fences.
  const fm = src.match(/^---\s*\n([\s\S]*?)\n---\s*(\n|$)/);
  if (fm) {
    const block = fm[1];
    // description: value  |  description: >-\n   folded lines...
    const dm = block.match(/^description:\s*(>-|>|\|-|\|)?\s*(.*)$/m);
    if (dm) {
      if (dm[1]) {
        // Folded/literal scalar: gather subsequent more-indented lines.
        const after = block.slice(block.indexOf(dm[0]) + dm[0].length).split("\n");
        const parts = [];
        for (const ln of after) {
          if (/^\S/.test(ln)) break;          // dedent → next key, stop
          parts.push(ln.trim());
        }
        out.description = parts.join(" ").trim();
      } else {
        out.description = dm[2].replace(/^["']|["']$/g, "").trim();
      }
    }
  }

  // Body bullets. Only inside relevant `##` sections.
  const body = fm ? src.slice(fm.index + fm[0].length) : src;
  const lines = body.split("\n");
  let inSection = false;
  for (const raw of lines) {
    const line = raw.replace(/\r$/, "");
    const h = line.match(/^##\s+(.+?)\s*$/);
    if (h) {
      inSection = /^(actions|weapons|functions)\b/i.test(h[1].trim());
      continue;
    }
    if (!inSection) continue;
    const b = line.match(/^\s*[-*]\s+(.*)$/);
    if (!b) continue;
    let rest = b[1];
    // Extract the label: [key](link) | **key** | `key` | key
    let key = "";
    let mk;
    if ((mk = rest.match(/^\[([^\]]+)\]\([^)]*\)/))) key = mk[1];
    else if ((mk = rest.match(/^\*\*([^*]+)\*\*/))) key = mk[1];
    else if ((mk = rest.match(/^`([^`]+)`/))) key = mk[1];
    else if ((mk = rest.match(/^([A-Za-z0-9_$.\-]+)/))) key = mk[1];
    // Description = everything after the first em-dash or " - " separator.
    let desc = "";
    const sep = rest.search(/\s(?:—|–|-)\s/);
    if (sep >= 0) desc = rest.slice(sep).replace(/^\s(?:—|–|-)\s/, "").trim();
    key = key.trim();
    if (key) out.actions.push({ action: key, description: desc || rest.trim() });
  }
  return out;
}

/**
 * Scan every `.claude/skills/<name>/SKILL.md` and emit ownership rows in the
 * SAME shape as scanSkillBookTocsFromDisk (`{skillBook, action, description}`),
 * so gearUp / dumpOwnershipMap consume it with no shape change. Emits one
 * skill-level row (`action: "(skill)"`, the frontmatter description) plus one
 * row per `## Actions`/`## Weapons`/`## Functions` bullet.
 *
 * @returns {Array<{skillBook: string, action: string, description: string}>}
 */
export function scanClaudeSkillsFromDisk() {
  const files = listClaudeSkillFiles();
  const rows = [];
  for (const f of files) {
    let src;
    try { src = readFileSync(f.path, "utf8"); }
    catch { continue; }
    const parsed = parseSkillMd(src, f.name);
    if (parsed.description) {
      rows.push({ skillBook: f.name, action: "(skill)", description: parsed.description });
    }
    for (const a of parsed.actions) {
      rows.push({ skillBook: f.name, action: a.action, description: a.description });
    }
  }
  rows.sort((a, b) => (a.skillBook + a.action).localeCompare(b.skillBook + b.action));
  return rows;
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/supabase_storage/index.js -->
```js
/**
 * Supabase Storage weapon — unified upload/download with path conventions.
 *
 * Auth: Uses SUPABASE_SECRETE_KEY via database service role.
 *
 * Storage convention:
 * - Bucket: GuildOS_Bucket (public)
 * - Path: {channel}/{questId}/{filename} — channel defaults to "general";
 *         cloud agent artifacts use channel="cursor_cloud"
 * - Retention: 30 days (enforced via bucket lifecycle policy or manual cleanup)
 */

export const toc = {
  buildPath: "Build a canonical storage path: {channel}/{questId}/{filename}.",
  writeFile: "Upload a file buffer to Supabase Storage.",
  readPublicUrl: "Get the public URL for a stored file.",
  readFile: "Download a file from Supabase Storage.",
  searchFiles: "Search files in a storage path.",
  deleteFiles: "Delete files from Supabase Storage.",
};
import { database } from "@/libs/council/database";

const DEFAULT_BUCKET = "GuildOS_Bucket";

// ---------------------------------------------------------------------------
// Helpers
// ---------------------------------------------------------------------------

async function getClient() {
  return database.init("service");
}

/**
 * Build a standardized storage path.
 * Convention: {channel}/{questId}/{filename}
 * @param {{ channel?: string, questId?: string, filename: string }} input
 */
export function buildPath({ channel = "general", questId, filename }) {
  if (!filename) throw new Error("filename is required");
  const parts = [channel];
  if (questId) parts.push(questId);
  parts.push(filename);
  return parts.join("/");
}

// ---------------------------------------------------------------------------
// Upload
// ---------------------------------------------------------------------------

/**
 * Upload a file to Supabase Storage.
 * @param {{ path: string, content: Buffer|Blob|string, contentType?: string, bucket?: string, upsert?: boolean }} input
 */
export async function writeFile({ path, content, contentType = "application/octet-stream", bucket = DEFAULT_BUCKET, upsert = true } = {}) {
  if (!path) throw new Error("path is required");
  if (!content) throw new Error("content is required");
  const db = await getClient();
  const { data, error } = await db.storage
    .from(bucket)
    .upload(path, content, { contentType, upsert });
  if (error) throw new Error(`Storage upload failed: ${error.message}`);
  return { path: data?.path || path, fullPath: data?.fullPath || path };
}

/**
 * Get the public URL for a stored file.
 * @param {{ path: string, bucket?: string }} input
 */
export function readPublicUrl({ path, bucket = DEFAULT_BUCKET } = {}) {
  const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL;
  if (!supabaseUrl) throw new Error("NEXT_PUBLIC_SUPABASE_URL not set");
  return {
    url: `${supabaseUrl}/storage/v1/object/public/${bucket}/${path}`,
  };
}

// ---------------------------------------------------------------------------
// Download
// ---------------------------------------------------------------------------

/**
 * Download a file from Supabase Storage.
 * @param {{ path: string, bucket?: string }} input
 */
export async function readFile({ path, bucket = DEFAULT_BUCKET } = {}) {
  if (!path) throw new Error("path is required");
  const db = await getClient();
  const { data, error } = await db.storage
    .from(bucket)
    .download(path);
  if (error) throw new Error(`Storage download failed: ${error.message}`);
  return { blob: data, path };
}

// ---------------------------------------------------------------------------
// List
// ---------------------------------------------------------------------------

/**
 * List files in a storage path.
 * @param {{ path?: string, bucket?: string, limit?: number }} input
 */
export async function searchFiles({ path = "", bucket = DEFAULT_BUCKET, limit = 100 } = {}) {
  const db = await getClient();
  const { data, error } = await db.storage
    .from(bucket)
    .list(path, { limit, sortBy: { column: "created_at", order: "desc" } });
  if (error) throw new Error(`Storage list failed: ${error.message}`);
  return { files: data || [] };
}

// ---------------------------------------------------------------------------
// Delete
// ---------------------------------------------------------------------------

/**
 * Delete files from Supabase Storage.
 * @param {{ paths: string[], bucket?: string }} input
 */
export async function deleteFiles({ paths, bucket = DEFAULT_BUCKET } = {}) {
  if (!Array.isArray(paths) || paths.length === 0) throw new Error("paths array is required");
  const db = await getClient();
  const { error } = await db.storage
    .from(bucket)
    .remove(paths);
  if (error) throw new Error(`Storage delete failed: ${error.message}`);
  return { ok: true, deleted: paths.length };
}

// ---------------------------------------------------------------------------
// Credential check
// ---------------------------------------------------------------------------

export async function checkCredentials() {
  try {
    const url = process.env.NEXT_PUBLIC_SUPABASE_URL;
    const key = process.env.SUPABASE_SECRETE_KEY;
    if (!url) return { ok: false, msg: "Missing NEXT_PUBLIC_SUPABASE_URL" };
    if (!key) return { ok: false, msg: "Missing SUPABASE_SECRETE_KEY" };
    return { ok: true, msg: "Supabase Storage credentials present" };
  } catch (e) {
    return { ok: false, msg: `Error: ${e.message}` };
  }
}
```
<!-- /GUILDOS-FILE -->
