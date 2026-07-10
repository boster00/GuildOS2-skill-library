# guildmaster

<!-- GUILDOS-SKILL name="guildmaster" -->

<!-- GUILDOS-DESC -->
MA triage: resolve ambiguous phrases, classify situation, assemble SA prompt, dispatch. MA orchestration — interpret each group of user messages into intents, track active objectives in a table, classify each into a fulfillment type (quest or direct_action), assemble and dispatch SA prompts, and evaluate output against closing criteria. Also runs the evolution loops (EL full-run scoring, SE step-checkpoint hardening) that validate substrate changes via fresh-agent dispatch. Use when acting as the MA orchestrator over user requests or running an EL/SE evolution run.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/guildmaster/SKILL.md -->
```markdown
---
name: guildmaster
description: >-
  MA triage: resolve ambiguous phrases, classify situation, assemble SA prompt,
  dispatch. MA orchestration — interpret each group of user messages into
  intents, track active objectives in a table, classify each into a fulfillment
  type (quest or direct_action), assemble and dispatch SA prompts, and evaluate
  output against closing criteria. Also runs the evolution loops (EL full-run
  scoring, SE step-checkpoint hardening) that validate substrate changes via
  fresh-agent dispatch. Use when acting as the MA orchestrator over user
  requests or running an EL/SE evolution run.
---

# Guildmaster

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/guildmaster/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto guildmaster <action>` (or read the source directly).

## Actions

- **active_objectives** — AO (active objectives) — a state in table format that tracks the current active objectives, closing criteria, fulfillment plan, status, progress, delta, questions/instructions.
- **dispatch** — Dispatch a subagent for each Running objective. Calls triage.dispatch() which validates fulfillment_plan in code and assembles the SA prompt — hard error if plan is missing.
- **evolutionLoop** — EL — iterative improvement loop. ALWAYS launches a fresh GM (or fresh SA/MA for component tests) via Agent dispatch — never validates substrate in the current orchestrator thread. Scores against the 42-item rubric or a component subset, applies prompt-level fixes per tightening mode, iterates up to N times.
- **fulfillment_types** — Decision rule: quest or direct_action. MA classifies only — behavior lives in the type's skill book.
- **packageDirect** — Execution rules for a direct-action subagent — covers discussion, one-shot tasks, search, and analysis.
- **supervisedEvolution** — SE — step-checkpoint sibling of EL. Dispatches a fresh agent per step (per the premade script), halts at the first behavioral deviation, applies ONE substrate fix, re-dispatches the same step, and only advances when that step is compliant. Same expected-behavior script format as EL; differs in granularity (per-step) and cadence (halt-on-deviation vs full-run scoring).
- **triage** — the first step for every group of user messages. Interprets these messages into multiple intents, classifies each one into a predefined situation, assembles SA prompt, dispatches to SA, evaluates output.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/guildmaster/index.js -->
```js
/**
 * Guildmaster skill book — MA orchestration.
 * Add actions back manually as needed.
 */
export const skillBook = {
  id: "guildmaster",
  title: "Guildmaster",
  description: "MA triage: resolve ambiguous phrases, classify situation, assemble SA prompt, dispatch.",
  steps: [],
  toc: {
    triage: {
      description: "the first step for every group of user messages. Interprets these messages into multiple intents, classifies each one into a predefined situation, assembles SA prompt, dispatches to SA, evaluates output.",
      howTo: `
      Before start, ensure these states are extracted from context: [active_objectives]
      Format of [active_objectives]: if not in context, read this skill book->toc->active_objectives for instructions on how to populate it.
      1. clarify all user terms. Identify ambiguous phrases, organize into array, and use weapon "triage" action "sloganLookup" to get definitions. For the phrases remaining ambiguous after this lookup, ask the user to clarify.
      2. interpret the user messages into objectives with clear closing criteria. Update the new objectives into the [active_objectives] state.
      3. output format: the response to each group of user message is a full print of the active_objectives table, plus text format of any non-objective specific responses.

      fulfillment_types: if missing from context, read this skill book -> toc -> fulfillment_types for instructions on how to fulfill different types of objectives 
`,
    },

    active_objectives: {
      description: "AO (active objectives) — a state in table format that tracks the current active objectives, closing criteria, fulfillment plan, status, progress, delta, questions/instructions.",
      howTo: `
      ## Closing criteria ownership
      MA proposes closing criteria for every new objective — do not leave it blank or wait for the user to write it. Make a specific, verifiable guess based on what the user said. User confirms or corrects before the objective moves to Running. If the user doesn't respond to the closing criteria, treat it as confirmed and proceed.

      ## Closing criteria format — checklist only
      Closing criteria must be a checklist. Each item is one finishable, independently verifiable thing.
      Format per item: ✅ <description> when done, ⬜ <description> when pending.
      Rules:
      - No compound criteria in one item. If two things must both be true, write two items.
      - No prose paragraphs. Each item must be checkable in isolation.
      - At least one item, no upper limit.
      - Update tick/box status each time the AO is reported — items flip ⬜ → ✅ as they complete.
      Example (good):
        ✅ gmail skill book triageInbox step 1 sets maxResults to 500 (file confirmed)
        ⬜ Running triage returns more than 100 emails in a single pass (user verifies on next run)
      Example (bad): "triageInbox returns all emails and the skill book is updated and stars are applied correctly"

      ## Lifecycle stages
      1. Clarifying: closing criteria proposed but not yet confirmed, OR fulfillment_plan not yet defined. Must contain the MA's proposed closing criteria for user to confirm. To move to Running: user confirms closing criteria AND fulfillment_plan is defined with at least a type and relevant skill books.
      2. Running: objective is actively being worked on, must contain an ETA in progress, use best estimate. Details to include: subagent status — pending, dispatched, processing response.
      3. Blocked: must contain a question or instruction for the user to resolve the blocking issue.
      4. Review: objective has been completed successfully. Must contain instructions for the user to review and confirm closure or give feedback. If the results are visible in a URL, must contain it so user can easily open and review.
      Note: closed objectives should be removed from the table. User is responsible for verbally confirming closure, so MA can safely remove it from the table. Rank active objectives by status with clarifying at the bottom, then blocked, then review, then running on top. This is because the way the msg outputs in LLMs, user sees the bottom part more clearly, so we want to make sure the most urgent objectives are at the bottom.
      Example: actual output should be in table format, but here is a text representation of how to populate the table:
      [
        {
          "objective": "Write a blog post about MA",
          "closing_criteria": "a 500 word blog post that covers the main features and benefits of MA, with an engaging title and a call to action at the end, with self evaluated score of 9/10 or higher, to be shown on the demo page URL = xxxx, opened in CIC",
          "fulfillment_plan": { "type": "direct_action" },
          "status": "running",
          "progress": "200 words written, outline completed, introduction and main features sections drafted",
          "delta": "100 words written since last update, outline completed since last update, introduction and main features sections drafted since last update",
          "questions_instructions": "Please continue writing the blog post. Focus on the benefits of MA and make sure to include an engaging title. Let me know if you need any specific information or if you want me to review what you've written so far."
        }
      ]`,
    },

    dispatch: {
      description: "Dispatch a subagent for each Running objective. Calls triage.dispatch() which validates fulfillment_plan in code and assembles the SA prompt — hard error if plan is missing.",
      howTo: `
For each objective in [active_objectives] with status = "running":

## Step 1 — Assemble and fire

  const result = await triage.dispatch({ objective })
  if (result.error) → move objective to "clarifying", set questions_instructions to result.error. Stop.
  if (result.warnings.length) → surface warnings to user.
  Agent({ description: objective.objective, prompt: result.saPrompt })

One Agent call per Running objective. Do not batch.

## Step 2 — Evaluate SA output against closing criteria

Two evaluations happen at different points — both are required:

SA self-evaluation (during execution, SA's responsibility):
  SA checks its own output before reporting done. For quest type this is the purrview gate.
  For direct_action the SA self-checks before returning. MA cannot skip this expectation.

MA evaluation (after SA returns, MA's responsibility):
  MA checks the returned output against objective.closing_criteria.
  For quest: did the quest reach "review" stage? If yes, purrview already did the heavy lifting.
  For direct_action: does the output actually satisfy the closing criteria?

  - Criteria met → move to "review". Set questions_instructions to review URL or artifact.
  - Not met, retryable → re-dispatch (up to 3 retries).
  - Not met, blocked → move to "blocked". State what the SA could not resolve and what user action is needed.
`,
    },

    fulfillment_types: {
      description: "Decision rule: quest or direct_action. MA classifies only — behavior lives in the type's skill book.",
      howTo: `
MA's only job: decide quest or direct_action. Record in fulfillment_plan.type.

## The one decision: does this go through quest?

Go through quest if ANY of these are true:
  - Work updates a repo (code changes, file commits)
  - Deliverable requires visual review (webpage, graphic, UI, screenshot)
  - Scope is large enough that formal tracking + review gate is worth the overhead

Stay direct_action if ALL of these are true:
  - Easy to verify inline (a quick check, a search result, a written answer)
  - No repo changes
  - No visual deliverable

When unclear → ask user before proceeding.

Fulfillment type to skill book map: load these skill books as default for each type.
  direct_action  → skill book: localwork
  quest          → skill book: questmaster
  compliance_nudge → skill book: localwork (transitional — goal is to eliminate)
`,
    },

    evolutionLoop: {
      description: "EL — iterative improvement loop. ALWAYS launches a fresh GM (or fresh SA/MA for component tests) via Agent dispatch — never validates substrate in the current orchestrator thread. Scores against the 42-item rubric or a component subset, applies prompt-level fixes per tightening mode, iterates up to N times.",
      howTo: `
## Core method (locked 2026-05-11)

EL ALWAYS spawns a fresh GM (or a fresh component agent) via the Agent dispatch tool. EL does NOT run cascade validation in the current orchestrator thread — that contaminates context and bypasses the isolation the test is supposed to validate. The orchestrator's job in EL is: dispatch, observe, score, fix substrate, re-dispatch.

## When to use
When substrate changes need empirical validation before committing — new cascade rules, new instruction-file rules, new weapon integrations, hardening from a live-run failure.

## Substrate constraint (Claude Code Agent tool topology)

The Claude Code Agent tool does not recursively grant Agent tool access to spawned sub-agents. A dispatched sub-agent is a leaf — it can use Read/Write/Bash/etc. but cannot itself dispatch sub-sub-agents. This means:

- **Full-cascade EL** (GM → MA-runner → SA-execute → ...) cannot run inside a single Agent dispatch. The dispatched fresh-GM will hit a substrate gap when MA-runner tries to dispatch its own SAs, and per MA-runner load-bearing rule #4 will correctly return \`completed: false\`.
- **Component EL** (dispatch one SA or one MA with synthetic inputs, score its single-layer behavior) is the practical EL form available today.
- **Full-cascade validation** requires either (a) the user running the test in a fresh top-level Claude Code session — Path C / manual passthrough, or (b) a future substrate that supports recursive Agent dispatch.

Choose component EL when iterating on a specific component's rules (SA-AO-match, SA-SMART-scan, MA-SMART-advisor, MA-runner-step-0). Choose Path C when validating cascade flow end-to-end.

## Rubric

Use the 42-item rubric from \`proving-grounds/smoketest3\` or \`proving-grounds/smoketest5-migration-rail\` as the master rubric. For component EL, score only items relevant to that component (typically 5-10 items per component). Items irrelevant to the test scope are N/A, not FAIL.

A run is "converged" when:
- 0 FAIL on the items scoped for the test, OR
- All remaining FAIL items map to a known substrate constraint surfaced honestly (e.g., Agent tool topology — counts as design constraint, not discipline failure)

## Per-loop procedure

1. **Identify the top FAIL from the prior loop** (or, for loop 0, define the test scope).
2. **Apply a single substrate fix.** Prompt-level rule first per tightening mode; scripted enforcement only if the prompt rule demonstrably failed in a prior loop. ONE hypothesis per loop — don't batch.
3. **Dispatch fresh agent via Agent tool.** Hand it the handshake + structured inputs + return-format spec. Do NOT contaminate with explanations of what's being tested or why — role-identity conflicts cause catastrophic regressions (verified 2026-05-07).
4. **Score the returned trace** against the rubric subset. Note PASS/FAIL/PARTIAL/N/A per item.
5. **Keep substrate fix only if improved.** If score regressed on any previously-passing item, revert (git revert or manual undo). Commit only fixes that produce measurable improvement.
6. Repeat up to N loops. Default N=5 for component EL, N=10 for full cascade. Stop conditions:
   - 0 fails on scoped items
   - 3 loops with no improvement (architectural gap, escalate to user)
   - Fix requires user decision (escalate)

## Failure modes to avoid

- **Running EL in current thread.** The orchestrator's context biases the test agent. Always Agent-dispatch.
- **Telling the test agent it is being tested.** Causes role identity loss and execution of the task directly instead of orchestration (EL5 2026-05-07 lesson). Just give it the user message and let it behave naturally.
- **Batching multiple hypotheses per loop.** Score regressions hide which fix mattered. One change per loop.
- **CLI weapon invocation in tests.** \`node -e\` calls fail on \`@/\` alias / Next.js runtime — agent often fakes results instead of failing honestly. Either run in app context or test the weapon via its skill book action with the weapon's actual signature.
- **Substrate constraint counted as FAIL.** If the test surfaces a topology limit (e.g., recursive Agent dispatch), that's a design constraint, not a discipline failure. Mark DEFERRED and surface as a substrate-evolution question.

## Output

Per loop:
| # | Item | Expected | Actual | Verdict |
|---|---|---|---|---|
| (rubric items relevant to scope) | | | | PASS/FAIL/PARTIAL/N/A |
| Total | | | | X PASS / Y FAIL |

Summary across all loops:
| Loop | Test | Fix applied | Result |
|---|---|---|---|
| 0 | (baseline test scope) | — | X/Y fail |
| 1 | (top FAIL from loop 0) | (one-line fix) | X/Y fail (kept/reverted) |
| ... | | | |

## Durable artifacts

- Substrate fixes that improved scores: committed to main with EL-loop reference in commit message
- Substrate constraints that surfaced: documented in CLAUDE.md or relevant agent file (so they don't re-surface as bugs)
- This howTo: update with new findings after each EL run
`,
    },

    supervisedEvolution: {
      description: "SE — step-checkpoint sibling of EL. Dispatches a fresh agent per step (per the premade script), halts at the first behavioral deviation, applies ONE substrate fix, re-dispatches the same step, and only advances when that step is compliant. Same expected-behavior script format as EL; differs in granularity (per-step) and cadence (halt-on-deviation vs full-run scoring).",
      howTo: `
## Core method

SE consumes a premade script that enumerates the expected behavior at each step of a cascade. It walks the script linearly. For each step:

1. Dispatch a fresh agent (SA or MA) with the step's synthesized inputs.
2. Compare the returned action + output against the step's expectedBehavior.
3. PASS → record verdict, advance to next step.
4. FAIL → halt forward progress. Identify the substrate gap. Apply ONE prompt-level fix (per tightening mode). Re-dispatch the SAME step. Repeat until PASS or retry budget exhausted.

SE is a sibling of EL — same handshake-and-dispatch substrate, same rubric vocabulary, same fresh-agent isolation. The only structural differences are (a) one dispatch per step instead of one per full run, and (b) the loop iterates within a step until that step is compliant rather than iterating across full runs.

## When to use SE vs EL

- **SE**: targeted hardening of a known-suspicious step, or step-by-step verification of a multi-step protocol where you want a clean per-step pass before advancing. Best when the cascade has well-defined step boundaries and synthesizable per-step inputs.
- **EL**: discovery — you don't yet know which step is breaking, want full-trace scoring, or step behaviors are too coupled to validate in isolation.
- **Combined**: run EL first to surface the failing step, then run SE on that step to harden it without re-running the whole cascade per fix.

## Premade script — input contract

The script is provided to SE before it runs. It mirrors EL's rubric structure but expands each rubric item into a dispatchable step.

\`\`\`
[
  {
    stepId: "S1-SA-AO-match-update-intent",
    dispatch: { kind: "SA" | "MA", target: "SA-AO-match" },
    synthesizedInputs: {
      // the structured inputs the SA's input contract expects
      userMessage: "...",
      // plus any prior-step fixtures the agent would normally inherit
      aoFixture: "...path or inline..."
    },
    expectedBehavior: {
      action: "returns null (no UPDATE intent matched)",
      output: "{ matched: false, reason: '<expected reason shape>' }"
    },
    scoreCriteria: [
      "Returned object has 'matched' field set to false",
      "Did NOT dispatch sub-sub-agents",
      "Did NOT edit docs/active-objectives.md directly"
    ],
    fixHints: ["if matched=true on invocation phrase, check SA-AO-match.md rule about UPDATE-intent-only"]
  },
  ...
]
\`\`\`

Rules for a valid script:

- Each step is independently dispatchable — synthesizedInputs must be complete; SE never lets a step depend on the live output of a prior step (use committed fixtures instead).
- ExpectedBehavior is concrete enough that PASS/FAIL is a binary judgment, not a vibe call.
- ScoreCriteria are checkable from the returned trace — anything that would require running another cascade to verify belongs in a SA-verify call, not a SE step.
- FixHints are optional but encouraged — they preserve operator knowledge from prior failures and shorten the retry cycle.

## Substrate constraint (same as EL)

The Claude Code Agent tool does not grant recursive Agent dispatch. So each SE step is a single leaf-agent dispatch: one SA or one MA, run to completion, return. SE cannot pause a fresh GM mid-cascade and resume after a substrate edit. Multi-step GM cascade tests become N separate SE steps, each dispatched fresh with synthesized inputs that simulate "this is what the cascade state would look like at this point."

## Per-step procedure

1. **Read the next step from the script.** If none remain → run terminates with PASS.
2. **Dispatch fresh agent via Agent tool** with handshake + synthesizedInputs + return-format spec. Do NOT tell the dispatched agent that it's being tested or which step it is — same role-identity hygiene as EL (verified 2026-05-07 EL5).
3. **Score the returned trace** against the step's scoreCriteria. Verdict ∈ { PASS, FAIL, PARTIAL, DEFERRED }.
   - PASS → record ledger row, advance to next step **immediately**. Do NOT pause for user check-in. Do NOT ask "shall I proceed?" A passing step is forward motion, not a deviation.
   - PARTIAL → treat as FAIL for halting purposes; capture which criteria passed in the ledger.
   - DEFERRED → known substrate constraint surfaced honestly (e.g., recursive-dispatch topology). Record reason, advance.
   - FAIL → enter fix loop.

4. **Fix loop (only on FAIL) — autonomous, up to 10 attempts per step (locked 2026-05-13):**
   - Identify the substrate gap. Prompt-level rule first per tightening mode; scripted enforcement only after a prompt rule has demonstrably failed in a prior retry.
   - Apply ONE fix. Commit it (so the re-dispatched fresh agent sees the new substrate).
   - Re-dispatch SAME step with the SAME synthesizedInputs.
   - **Improvement gate:** compare new score vs prior best for this step. Keep the delta only if PASS-count strictly increased AND no previously-passing criterion regressed. Otherwise revert the change (\`git revert\` the substrate commit) and try a different hypothesis. Never accumulate non-improving changes — that's how substrate decays.
   - Re-score. PASS → advance. FAIL → next attempt with a new hypothesis.
   - **Retry budget = 10 attempts per step** (raised from 3 on 2026-05-13). Orchestrator iterates without pausing for user confirmation between attempts. SE surfaces real deviations via execution; it does not deliberate them with the user upfront.

5. **Halt-and-ask conditions — the ONLY moments user is involved (locked 2026-05-13):**
   - **10 attempts exhausted without convergence on a single step.** Architectural gap — surface as State 1 with full attempt-by-attempt diagnostic.
   - **Irreversible-decision required mid-loop.** Production schema migration that would destroy data, drop-table or force-push, mass external side effects (Asana mass-update, mass email send), changes to unrelated app flows. These genuinely need user approval before applying.
   - **Substrate constraint surfaced.** Topology limit (e.g., recursive Agent dispatch unavailable). Mark DEFERRED, surface as substrate-evolution question.

   **NOT halt conditions (do NOT involve user for any of these):**
   - "Next attempt is a deletion." Use archiving instead (see point 7).
   - "A step just PASSED." Advance silently to the next step; brief progress note only in the response.
   - "We have multiple fix hypotheses." Pick one, try it. If it doesn't improve, revert per improvement gate, try the next. No deliberation.
   - "I'm about to change the criteria." Try substrate fixes first — criteria-change only after 10 substrate-fix attempts fail, and that surfaces via the 10-fail escalation.

6. **Question hygiene — what reaches the user.**
   - Real deviations are observed via execution, not predicted via deliberation. Don't ask "should we try X or Y?" — try one, score, try the other if the first didn't improve.
   - Questions should describe a costly-to-reverse choice already on the table, not a brainstorm.
   - "Should I delete this file?" is NEVER a question (per point 7). "Should I drop this production column?" IS (irreversible without backup).
   - "Which approach do you prefer?" is NOT a question if both are cheap to try.

7. **Archive over delete — locked 2026-05-13.**
   - Any substrate cleanup that LOOKS like "delete and try again" → instead, move the artifact under \`_archive/<topic>/<date>/\` (or similar). Then the SE re-dispatch sees a clean substrate, but rollback is just a \`mv\` away.
   - Archiving is cheap to reverse → does NOT count as a costly-to-reverse decision → does NOT halt-and-ask. SE applies it autonomously.
   - This applies to instruction files, skill book actions/folders, weapon folders, docs that are being replaced. App code with live imports cannot be archived without breaking — those edits still need normal commits, but the deleted code blocks should be preserved alongside (e.g., commented-out next to the new code, or a sibling \`_archive/<filename>.preserved.js\` file) so the diff is reversible.

8. **Regression check after any kept fix:** if the fix touched an instruction file or weapon that earlier (already-passing) steps also exercised, re-dispatch those earlier steps to confirm no regression. The improvement gate in step 4 covers within-step regression; this point covers cross-step regression. If a previously-passing step now FAILs → revert the fix and try a different hypothesis.

9. **Advance only on PASS** (or DEFERRED). PARTIAL never advances on its own.

## Failure modes to avoid

- **Synthesized-input drift.** A hand-written synthesizedInputs may not match what a real prior step would produce, so the step passes in SE but fails in the live cascade. Mitigation: fixtures should be drawn from a recorded actual prior-step output (commit the recording), not improvised.
- **Treating PARTIAL as PASS.** PARTIAL means some criteria matched and some didn't — that's still a deviation. Halt and fix.
- **Batching multiple fixes per step.** Same as EL: one hypothesis per retry. Otherwise you can't tell which fix mattered, and regression checks lose meaning.
- **Telling the dispatched agent it is being tested.** Same role-identity hazard as EL. Just hand it the handshake + inputs and let it behave naturally.
- **Skipping regression check.** A fix that passes step K but quietly breaks step K-2 is worse than no fix. Always re-fire prior steps that touched the same substrate locus.
- **Confusing SE's per-step retry with EL's full-run retry.** SE retries one step until compliant; it does NOT re-run earlier passed steps each retry (except for the regression check after a committed fix). This is what makes SE cheaper than EL when the failing step is known.

## Output

Step-by-step ledger:

| # | Step | Dispatch | Expected | Actual | Verdict | Fix applied | Retries |
|---|---|---|---|---|---|---|---|
| 1 | S1-... | SA-AO-match | ... | ... | PASS | — | 0 |
| 2 | S2-... | SA-SMART-scan | ... | ... | FAIL→PASS | (one-line fix) | 1 |
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

## Durable artifacts

- Substrate fixes that passed both per-step and regression checks: committed to main with SE-step reference in commit message.
- Fixtures committed alongside the script (so future SE runs are reproducible).
- Substrate constraints surfaced as DEFERRED: documented in CLAUDE.md or the relevant agent file so they don't re-surface as bugs.
- This howTo: update with new findings after each SE run.

## Relationship to EL

- **EL converges by reducing FAIL count across full traces.** SE converges by reducing FAIL count one step at a time, never advancing past a non-compliant step.
- **EL's rubric is the master vocabulary.** SE steps reuse rubric items where applicable — the scoreCriteria of a step is typically a subset of the rubric items relevant to that step's locus.
- **EL has higher discovery power; SE has higher diagnostic precision.** Discovery → EL. Locked-down hardening → SE.
- **Both share fresh-agent isolation, one-hypothesis-per-loop, and the substrate-constraint-not-FAIL rule.**
`,
    },

    packageDirect: {
      description: "Execution rules for a direct-action subagent — covers discussion, one-shot tasks, search, and analysis.",
      howTo: `
A direct-action subagent executes a focused task and replies inline. No quest, no stage transitions, no items rows.

## Rules
1. Read the ## Task section at the bottom of this prompt — those are your closing criteria.
2. Execute and produce output directly in your reply.
3. Self-evaluate before replying: does every element of the closing criteria appear in the output?
4. End with one of:
   - "✅ Done — [one-line summary of what was produced]"
   - "⚠ Incomplete — [specific gap and why]"

## Scope
- Discussion / explanation / analysis → inline text
- Research / search → inline findings
- Simple writes (email drafts, docs) → inline content
- Read-only queries (DB, files) → inline results
- Config edits not requiring a quest → edit + confirm in reply

## What direct_action is NOT
- Repo commits (→ quest)
- Visual deliverables needing review (→ quest)
- Multi-step pipelines with formal tracking (→ quest)
`,
    },
  },
};

/**
 * callToArms — legacy automated trigger.
 * Fires when a quest moves from idea → execute. Logs the event.
 * No agent decision required.
 */
/**
 * respawnAdventurer — stub. Real implementation removed during SE refactor.
 * Returns {ok:false, reason} so the cron auto-respawn path no-ops without
 * blowing up at compile time. The cron checks the return value before logging.
 */
export async function respawnAdventurer() {
  return { ok: false, reason: "respawnAdventurer disabled — feature deprecated" };
}

export function callToArms(_userId, input = {}) {
  try {
    const entry = `[${new Date().toISOString()}] callToArms: ${JSON.stringify(input)}`;
    // Log to console only — no file I/O to avoid side effects in SSR
    if (process.env.NODE_ENV === "development") console.log(entry);
  } catch {
    // silent
  }
  return { ok: true };
}
```
<!-- /GUILDOS-FILE -->
