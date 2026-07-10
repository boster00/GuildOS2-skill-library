# DungeonMaster

<!-- GUILDOS-SKILL name="DungeonMaster" -->

<!-- GUILDOS-DESC -->
Author Supervised Evolution rubrics. Encodes the locked 3-section layout, one-sentence summary template, end-point-from-user-perspective rule, ownership notation ([TYPE:identifier->method]), level-of-detail rule, and [IMPORTANT DETAIL] labeling convention. Use to write a new SE rubric for a stated goal — producing a markdown doc that follows the locked 3-section layout (summary+end-point → steps storyboard → behavior-logic encyclopedia). Read refLayoutContract before authorSERubric.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/DungeonMaster/SKILL.md -->
```markdown
---
name: DungeonMaster
description: >-
  Author Supervised Evolution rubrics. Encodes the locked 3-section layout,
  one-sentence summary template, end-point-from-user-perspective rule, ownership
  notation ([TYPE:identifier->method]), level-of-detail rule, and
  [IMPORTANT DETAIL] labeling convention. Use to write a new SE rubric for a
  stated goal — producing a markdown doc that follows the locked 3-section
  layout (summary+end-point → steps storyboard → behavior-logic encyclopedia).
  Read refLayoutContract before authorSERubric.
---

# Dungeon Master

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/DungeonMaster/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto DungeonMaster <action>` (or read the source directly).

## Actions

- **authorSERubric** — Write a new SE rubric for a stated goal. Produces a markdown doc following the locked 3-section layout.
- **refLayoutContract** — Reference: the locked 3-section layout, summary template, end-point rule, ownership notation, level-of-detail rule, and [IMPORTANT DETAIL] convention every authored rubric must follow. Read before authorSERubric.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/DungeonMaster/index.js -->
```js
/**
 * DungeonMaster skill book — authors Supervised Evolution rubrics.
 *
 * Pure text registry. DungeonMaster owns the WRITING of rubrics (style, layout,
 * tone). The SE execution method itself lives in `supervisedEvolution.run`
 * and is the same across every rubric.
 *
 * Locked layout: summary+end-point → steps storyboard → behavior-logic encyclopedia.
 * Summary is one sentence per template. End point describes what the user observes
 * at the END of THIS rubric's run, from the user's perspective. Section 2 step
 * structure has only two components: the user's message and the expected agent
 * behaviors (WBS-numbered with inline [TYPE:identifier->method] tags). Behaviors ARE
 * the score criteria.
 *
 * Behaviors are written at the minimum level of detail the SE judging agent needs to
 * compare actual vs expected. Must-have specifics from the user's instructions are
 * flagged inline with [IMPORTANT DETAIL] and mirrored in a Section 1 cross-reference list.
 */

export const skillBook = {
  id: "DungeonMaster",
  title: "Dungeon Master",
  description:
    "Author Supervised Evolution rubrics. Encodes the locked 3-section layout, one-sentence summary template, end-point-from-user-perspective rule, ownership notation ([TYPE:identifier->method]), level-of-detail rule, and [IMPORTANT DETAIL] labeling convention.",

  toc: {
    authorSERubric:
      "Write a new SE rubric for a stated goal. Produces a markdown doc following the locked 3-section layout.",
    refLayoutContract:
      "Reference: the locked 3-section layout, summary template, end-point rule, ownership notation, level-of-detail rule, and [IMPORTANT DETAIL] convention every authored rubric must follow. Read before authorSERubric.",
  },

  authorSERubric: `
You are authoring a Supervised Evolution rubric.

SE rubrics describe WHAT to test (the real-world endpoint the run lands the user at) and the user-→-agent storyboard that gets there. The *generic* execution loop (halt-on-deviation, fix loop, improvement gate, regression pass, finale prompt) lives in \`supervisedEvolution.run\` and is constant across rubrics — do not re-spell THAT. But **lifecycle mechanics that are themselves the behavior under test in a given step ARE spelled out as expected behaviors** (dispatcher routing, step status transitions, purrview/review/closing gates, the 10-try fallback condition, CIC verification) — see resolution 3 below. Omitting a tested mechanic because it is "substrate" is the failure, not the fix.

## Canonical-example precedence — the Hypoxia rubric is truth (locked 2026-05-15)

The hand-tuned \`docs/smoke-tests/SE-rubric-hypoxia-test.md\` (+ \`-part-2.md\`) is the ground-truth exemplar. **When these abstract authoring rules conflict with that example, the example wins.** These instructions define the skeleton; the example defines the real acceptable density, lifecycle coverage, and grounding style. Write and SCORE SE rubrics like the Hypoxia example, not like a minimal abstract contract. Ten resolved instruction-vs-example conflicts (example = truth) — apply on author AND on \`supervisedEvolution.run\` scoring:

1. **Section 2 steps are three-part**, not two: (a) mock user message → (b) expected behaviors → (c) \`**Agent response:**\`. The agent response is a required third component, not optional.
2. **Ownership tags \`[TYPE:id->method]\` are required on expected-behavior bullets ONLY** — NOT on embedded canonical-quest / quest-data blocks. \`input\`/\`method\`/\`output\`/\`verification\`/\`viewResults\` lines inside a pinned grounding-truth block are correctly untagged.
3. **Spell out lifecycle mechanics when they are the behavior under test** (dispatcher behavior, status transitions, purrview/review/closing gates, 10-try fallback, CIC verification, step-execution mechanics). Do not omit them merely because they are substrate mechanics.
4. **Agent-reply ownership is phase-sensitive**: early/planning steps use \`[MD:CLAUDE->triageCascade]\`; post-quest execution/update steps use \`[MD:CLAUDE-GM->triageCascade]\`. (The reply is still authored by the GM as conversation partner — the tag reflects which instruction set drives that phase.)
5. **Summary format** = a \`### Summary\` heading + one plain, template-like sentence. Literal bracket placeholders / bolding are not required; the wording is template-like, not a fill-in form.
6. **Section 1 "Important details" covers ALL load-bearing non-negotiables introduced anywhere in the scenario**, including mid-execution refinements (e.g. case-insensitive match rule, null-on-no-match behavior, an SSH/SQL retrieval method co-developed later) — not only the user's opening must-haves.
7. **The quest-creation step embeds the canonical quest as creation-time first-version truth**: every objective \`status='pending'\`, all \`notes\` empty, only pre-existing input deliverables present. It is NOT the mid-execution UI snapshot (which carries statuses/notes/extra deliverables). Source from the locked mockup, projected to first-version.
8. **Grounding-truth blocks nested inside expected behaviors are allowed** — they are pinned expected results, not a banned "separate criteria block." Behaviors remain the score criteria; the pinned block is the expected value those behaviors must produce.
9. **A deliberate \`[co-develop-with-user]\` anchor is a valid planned mid-quest halt** — in addition to the narrow \`supervisedEvolution.run\` halts (10-attempt / irreversible / unbuildable / finale). A rubric may script this halt as expected behavior.
10. **Section 3 entries may include concrete pinned canonical examples** (e.g. the hypoxia step examples under \`[SB:worker->executeStep]\`), not generic-only behavior. Generic-only is not strict enough when the example pins concretes.

## Scoring fidelity — granular clusters, semantic invariants, inline severity (locked 2026-05-18, user-directed)

These three rules refine HOW a rubric is made scorable. They do not relax any rule above; they make "behaviors ARE the criteria" diagnosable and non-brittle. Apply on author AND on \`supervisedEvolution.run\` scoring.

1. **Split a multi-assertion result step into separately-judged clusters.** When one step (above all the quest-creation step) bundles many independently-failing assertions — naming, description shape, item set, step count/order, per-step content, \`[IMPORTANT DETAIL]\` placement, status/notes cleanliness, finale reply — author each as its OWN WBS behavior (or a labelled \`[Cluster N — name]\` sub-behavior), each a standalone binary criterion, so a failure names exactly which assertion broke. A single monolithic "the quest is created correctly" behavior is mis-authored: it is undiagnosable under the fix loop.

2. **The embedded canonical block is a semantic-invariant judge reference, not a verbatim target.** The grounding-truth block is scored by SEMANTIC INVARIANTS, never by word-for-word match. Head the block with: (a) the explicit list of HARD constraints that must hold exactly (every \`[IMPORTANT DETAIL]\` in its owning step, any user-locked exact value / order / scope-out, exactly the stated creation-time items, strategic-only description, no invented architecture); and (b) the statement that word-for-word divergence from the canonical prose is NOT a deviation as long as each step's stated invariant and the hard constraints hold. Give every canonical step an explicit \`Invariant:\` line. This is the contract-coherent form of "do not require near-verbatim reproduction."

3. **Failure-class severity is annotated inline on the behavior/cluster — never as a separate section.** A behavior or cluster MAY carry an inline \`Severity — Hard fail: … · Minor fail: … · Pass: …\` annotation as judge guidance. This is the sanctioned realization of a "scoring rubric table": the severity-tiered granular behaviors ARE the single scoring surface. It does NOT reintroduce the banned construct — a SEPARATE, parallel "Score criteria" section or scoring table remains forbidden (see "Stays out"); the ban is on a second parallel list that drifts from the behaviors, NOT on per-behavior severity co-located with the one behavior list.

Read \`refLayoutContract\` before drafting. Produce exactly three sections, in this order.

The three sections form an authority hierarchy — **strategy (§1) → tactical (§2) → mechanism (§3)**. When sections conflict, the lower-numbered wins: Section 2 must achieve Section 1's end point and important details; Section 3 must produce Section 2's observable beats. While authoring, sanity-check that each Section 2 step actually lands the Section 1 strategy and that each Section 3 entry actually produces its Section 2 bullets. See \`refLayoutContract\` → "Section precedence" for the full rule.

---

## Section 1 — Summary + end point

### Summary

One sentence, following this template verbatim with the two slots filled in:

> This Supervised Evolution script describes what should happen for **[scope]** of **[quest]**.

Where:
- **[scope]** names the slice of the journey this rubric exercises — e.g. "planning the quest", "executing the SMART plan", "verifying the live page", "the save-and-deliver flow".
- **[quest]** names the overall objective being worked on — e.g. "updating the Boster Hypoxia pathway page", "adding the email-digest feature".

No scene-setter paragraphs. No restatement of the SE method. No exposition. One sentence.

### End point

Describe what the user observes at the end of THIS rubric's run, from the user's perspective. The user doesn't actively verify — the agent does the surfacing.

**Default end-point mode (locked): CIC shows the evidence.** The canonical framing is "the user returns to find X already open in their browser — the agent has pulled it up via CIC." Apply this default to every rubric. Only switch surfaces if the user's source instructions explicitly name a different one (inbox, dashboard, file in editor, Slack message, CLI output, etc.) — do not invent a non-CIC surface the user didn't ask for.

For a planning rubric the end point is typically a quest details page the agent has pulled up via CIC — describe the quest title, the (strategic-only) description, the objectives table rows (with #, input, method, output, view-results), the deliverables table (initial assets), and any agent-response prompt the user receives (e.g. SE's finale prompt about committing the SMART plan to a skill book).

For an execution rubric the end point is typically a live page / file / DB row the agent has updated, with CIC pre-navigated to it so the user can confirm at a glance — describe what's visible and how to spot it.

Include enough detail that a human can verify the observation in CIC: artifact names, key copy, key field values, the set of sub-artifacts (deliverables, rows, components) and what makes each one identifiable. If a quest is the artifact and it has multiple deliverables, list the deliverables by key with one-line descriptions.

If the SE finale prompt (per \`supervisedEvolution.run\`) is expected to fire for this run (planning rubric + SMART plan produced), mention it as part of what the user sees at the end.

Singular by default; only describe a second end point if the run genuinely lands the user at two distinct user-visible artifacts.

### Important details (optional cross-reference list)

If the user's source instructions for this rubric named any must-have specifics — exact copy, specific tool to use, specific value, named identifier, mandatory constraint — end Section 1 with a numbered list. Each entry **starts with the location** (where the detail appears in Section 2), then a colon, then the one-line description:

\`\`\`
### Important details

1. Step <N> — <step title>, behavior <wbs id>: <one-line description of the detail>.
2. Step <N> — <step title>, behavior <wbs id>: <one-line description>.
\`\`\`

Every list entry must correspond to a behavior bullet in Section 2 that is itself marked \`[IMPORTANT DETAIL]\` (see Section 2 rules). The list and the in-step labels are paired — if a detail appears on one but not the other, fix the missing side.

Omit this subsection entirely if the user's instructions had no must-have specifics. Do not invent details to fill it.

---

## Section 2 — Steps

The storyboard. Each step is one user interaction with EXACTLY TWO components: the mock user message, then the expected agent behaviors. The behaviors ARE the score criteria — there is no separate criteria block.

Per-step entry structure:

\`\`\`
### Step N — [short title]

> [mock user message verbatim, blockquoted; flows naturally from prior step's Agent response]

**Expected agent behaviors:**

1. [TYPE:identifier->method] one-line description of behavior 1.
2. [TYPE:identifier->method] one-line description of behavior 2 — when this behavior flags a user-emphasized must-have into quest data, the description names what's getting flagged. The \`[IMPORTANT DETAIL]\` marker itself lives in the quest's objective.method content (visible in Section 3), NOT in this bullet.
   2.1. [TYPE:identifier->method] one-line description of a nested behavior (when behavior 2 dispatches a sub-action).
   2.2. [TYPE:identifier->method] one-line description of another nested behavior.
3. [TYPE:identifier->method] one-line description of behavior 3.

**Agent response:** "<verbatim quoted user-visible reply from the GM>"
\`\`\`

Rules:

- **WBS numbering.** Top-level integers (1, 2, 3, ...). Nest with dot notation (2.1, 2.2; 2.1.1 if you ever need it but cap at three levels). Nesting reflects ownership chain — when behavior X dispatches a sub-action, the sub-action becomes X.1.
- **Inline ownership tag.** Every bullet begins with \`[TYPE:identifier->method]\` (see "Ownership notation" below), then a brief description of the observable behavior.
- **Level of detail.** A behavior bullet contains only what the SE judging agent needs to compare actual vs expected. No exposition, no rationale, no "because X" tails, no extra context. If you find yourself writing "the agent classifies as action because the verb is strong," strip the "because" clause — the verb-classification rule lives in the Section 3 paragraph for that owner, not in the bullet.
- **\`[IMPORTANT DETAIL]\` belongs to quest data, not the rubric.** When the user's mock message uses natural-language emphasis (e.g. "it is important that the promo copy reads exactly X", "must do Y"), the agent's expected behavior is to FLAG that emphasis in the corresponding objective's \`method\` content by embedding \`[IMPORTANT DETAIL]\` inline next to the must-have clause. The Section 2 bullet describes the flagging behavior; it does NOT carry \`[IMPORTANT DETAIL]\` inline itself. The actual \`[IMPORTANT DETAIL]\` marker appears in the quest's objective.method data (visible in Section 3's expected method content for that objective). Section 1's "Important details" cross-reference list points to the Section 2 behavior that performs the flagging. Pattern: every important detail is mentioned twice — once by user NL emphasis in the mock message, once by agent behavior flagging it into objective.method.
- **One bullet per unique behavior.** If two distinct behaviors share an owner, that's still two bullets — different methods, different invocations, or different observable beats of the same invocation.
- **Each bullet is a binary criterion.** A step passes when every numbered behavior is observable in the trace + the user-visible reply. No separate score-criteria section.
- **Pin the result, not just the action (grounding truth — load-bearing).** A behavior must assert WHAT the agent does AND the specific RESULT it must produce. The result is the grounding truth — the right answer the SE loop nudges the substrate toward. An agent that performs the action but arrives at a *different* result has deviated, exactly as if it skipped the action. "agent extracts the deliverables" is not enough; "agent extracts these N deliverables with these exact input/method/output values" is the criterion. This is most load-bearing for **result-producing steps whose output later steps depend on** — above all the step that creates the quest: if the produced \`quests.objectives\` plan is wrong, every downstream step tests against a broken foundation, so the wrong plan is a hard deviation even if every action "ran". **Judge-reference, not agent-input (anti-overanchor):** the embedded grounding truth is the SE judge's scoring yardstick ONLY — it is NEVER fed to the agent under test. The dispatched agent gets only the user message + its real production substrate; it must reproduce a similar-enough result from its OWN skill books/weapons/prompts. If it can't, the substrate is deficient and gets fixed via the SE loop — you do not hand the agent the answer. A rubric whose grounding truth leaks into the agent's prompt tests nothing.
- **Result-producing steps embed the expected canonical output.** The quest-creation step must contain the FULL first-version quest content — every objective's \`input\`/\`method\`/\`output\`, the \`deliverables\`, the strategic \`description\`, and every \`[IMPORTANT DETAIL]\` placement — as the grounding truth, not "a quest with N objectives is created". Source that canonical content from the locked reference (the committed quest-UI mockup) so the rubric and the mockup never drift. Other result-producing steps (worker produces an asset, page assembled, page deployed) likewise pin the expected concrete output (shape + key values), not "an asset is produced". Put the expected-output block inside the step it belongs to (or its Section 3 entry), clearly labeled, so the SE judge compares produced-vs-canonical.
- **Cover all five workflow phases concretely.** Every rubric must exercise: (1) clarifying with the user — the specific questions asked; (2) creating the quest — the canonical plan embedded; (3) doing the quest — the worker's concrete actions and their expected outputs; (4) escalating + resolving issues with the user — the specific deviation surfaced and the correct resolution; (5) reporting to the user — the specific outcome stated. Each phase is pinned to its right answer, specific enough that the agent handles incidental gaps natively but cannot drift on the load-bearing conclusions.
- **Native-now / chunking-candidate note.** When a result-producing behavior is *native* — the agent does it inline, no dedicated SB/WP action owns its decomposition — leave it native by default and add an explicit note in that owner's Section 3 entry: it is native-for-now and a chunking candidate; if SE runs show it underperforming against the grounding truth, the SE agent may chunk the context into more specific skill-book + weapon actions (the exact decomposition is the SE agent's call during the fix loop); escalate to the user only per \`supervisedEvolution.run\`'s standard halt-and-ask — no behavior-specific special-casing. Do not pre-emptively invent owned actions for a native behavior; the note is the hook, the SE loop decides if/when to act on it.
- **Agent response (explicit label).** The agent's user-visible reply is NOT a numbered behavior bullet — it is a separately labeled trailing entry after the behaviors list: \`**Agent response:** "<verbatim quoted reply>"\`. The response generator is always the GM (the conversation partner); other agents (workers, MAs, SAs) never speak directly to the user — they surface to GM, who replies. Include the response after the behaviors list with a blank line above.
- **Conversational flow.** The mock user message in step N+1 must be a natural response to the Agent response from step N. Read the agent's reply, then write the user's next message as a direct continuation — not as an orthogonal new prompt. If the user's next message can't be connected to the prior agent reply, the rubric is jumping flows; split into a sub-thread or rework the prior step's reply.
- **Describe, do not reason.** "the agent classifies the message as action" is fine; "the agent reasons through cascade rules to classify as action" is not — reasoning belongs in Section 3.

---

## Section 3 — Behavior logic

The encyclopedia. One entry per \`[TYPE:identifier->method]\` referenced anywhere in Section 2. Each entry is the substrate-truth that SE iterations nudge toward.

Per-entry shape:

\`\`\`
### [TYPE:identifier->method]

[One paragraph in plain prose. What this owner exists for; what triggers it; what it consumes; what it emits. No JSON, no schema bullets — paragraph form. This paragraph is the load-bearing truth; if substrate disagrees with it, substrate is what SE corrects.]

**Input schema:** [shape of inputs the owner consumes]

[For agent (LLM-bearing) owners only]
**Prompt template:**
- **Slug:** [short identifier, e.g. \`classify-message-v1\`]
- **Transmutation:** [2–4 sentences in prose describing what the prompt asks the model to do — no full prompt text, no example interpolations]
- **Output schema:** [shape the agent must return]

[For non-agent owners (weapons without LLM, DB columns, .md sections)]
**Source:** [file path / section anchor / table.column]
\`\`\`

Order entries so owners with no upstream dependencies come first; downstream owners follow. Group by kind if it improves scannability (MD entries, then SB entries, then WP entries, then DB entries).

---

## Ownership notation

Every owner reference — inline in Section 2 and as a heading in Section 3 — uses the format \`[TYPE:identifier->method]\`. No spaces around the colon or arrow.

| TYPE | Meaning | Identifier | Method |
|---|---|---|---|
| WP | Weapon (\`libs/weapon/<identifier>/index.js\`) | weapon name, e.g. \`questmaster\` | exported function name, e.g. \`createQuest\` |
| SB | Skill book (\`libs/skill_book/<identifier>/index.js\`) | skill book id, e.g. \`DungeonMaster\` | action key in the skill book's \`toc\` |
| MD | Markdown file (repo root or docs) | filename without \`.md\`, e.g. \`CLAUDE-GM\` | section anchor or rule name |
| DB | Database (\`<identifier>\` = table) | table name, e.g. \`deliverables\` | column name or row pattern |

Examples:

- \`[WP:questmaster->createQuest]\`
- \`[SB:DungeonMaster->authorSERubric]\`
- \`[MD:CLAUDE-GM->triageCascade]\`
- \`[MD:CLAUDE->workOnMain]\`
- \`[DB:objectives->output]\`
- \`[DB:deliverables->kind]\`
- \`[DB:quests->stage]\`

The same tag appears as an inline label in Section 2 behaviors and as the heading of the matching Section 3 entry. SE iterations find every owner referenced in steps by scanning for \`[TYPE:\` brackets.

### Owner vs subject (load-bearing)

The tag names the **owner** — the entity *responsible for instructing* the behavior. It is NOT the thing being acted on. Test: ask "who is *telling* this to happen?" — that is the owner; the thing it happens *to* is the subject and goes in the bullet's prose, never as the leading tag.

- **A DB table is a SUBJECT, not an owner, for framework behaviors** — status flips, scans, dispatch, stage transitions, comment-resolve, note-writes, row inserts. The owner is the weapon or skill-book action that *instructs* the change.
  - Wrong: \`[DB:quests->objectives] objective 4's status flips to attention\`.
  - Right: \`[SB:questmaster->reportAttentionItems] surfaces objective 4, whose status is \`attention\` in quests.objectives\`.

- **The one exception — \`[DB:quests->objectives]\` IS the owner when the behavior is the worker doing the actual concrete job** (fetch the input asset, run OCR, scrape pages, render HTML, SSH-deploy). The objective row's own input/method/output content is literally the instruction driving the worker — the worker has no other source of truth for the concrete work, so the objective content owns it.
  - Right: \`[DB:quests->objectives] objective 1's content instructs the worker's concrete job: fetch the pathway image, OCR-extract gene coords, write the genes JSON\`.

- **Concrete work must be described.** A rubric that only narrates lifecycle meta-beats (status flips, anchor halts, checkpoint surfacing) and never says what the worker physically does is incomplete. Every quest objective that does real work gets a \`[DB:quests->objectives]\`-owned beat naming the concrete actions (the actual fetch / transform / produce), placed at the SE-step where that quest objective executes. The \`[SB:worker->executeStep]\` Section 3 entry carries the generic read-instruction → fetch-input → perform-method → write-output+flip-status → self-verify cycle, with the quest's concrete objectives as the canonical example.

---

## Stays out of every authored rubric

- A multi-sentence Summary or scene-setter — Summary is one sentence per template.
- A SEPARATE / parallel "Score criteria" section or scoring table — behaviors ARE the criteria (per-behavior inline \`Severity —\` annotations are allowed and ARE the sanctioned failure-class form per "Scoring fidelity" above; the ban is on a second parallel list, not on co-located severity).
- An "Owner:" header line per step — the inline \`[TYPE:identifier->method]\` labels on each behavior bullet do that job.
- Exposition or rationale in behavior bullets — strip "because" clauses; the substrate truth lives in Section 3.
- An orchestration graph / parallel-serial group structure / any \`quests.action_steps\`-style field — that column was dropped. Orchestration is implicit in objective ordering: objectives are a flat ordered list, each with clear input + clear output, so execution renders event-bus-stably and the worker dispatcher walks them in order. Never author a \`{groups:[...]}\` shape or "parallel vs serial" narrative.
- SE execution mechanics (halt-on-deviation, fix loop, improvement gate, regression pass, finale prompt) — \`supervisedEvolution.run\` owns these.
- Locked-everywhere substrate facts (the \`objectives.output\` jsonb shape and its \`{asset, format, display?, verification?, verification_example?}\` element schema, the \`deliverables.kind\` enum \`html|image|markdown|json|link\`, initial stage = \`execute\` under the new 5-stage flow, the \`<<SE_TRACE>>\` envelope, the work-on-main rule) — already documented in the substrate.
- Per-criterion fix hints — the SE orchestrator deduces fixes from the failed trace + the Section 3 behavior paragraphs.
- Run ledger — separate file or commit history; rubrics are evergreen and re-runnable.
- Verbatim historical reference HTML comments / migration-moment commentary.
- A "build inventory" section that mirrors the skill book registry — agents read the registry directly.
- An "Important details" list with no corresponding \`[IMPORTANT DETAIL]\` bullets in Section 2 — and vice versa.

---

## Inputs the author must gather before writing

- What does the user observe at the end of THIS rubric's run? (The end point, from the user's perspective.)
- What is the [scope] / [quest] phrasing for the one-sentence summary?
- The set of behavior owners (\`[TYPE:identifier->method]\` tags) involved in producing the end point.
- The mock user-message arc — one user-side message per step.
- Existing substrate the rubric leans on, so it doesn't get re-described.
- The user-specified must-haves (if any) that should be flagged as \`[IMPORTANT DETAIL]\`.

## Output

A single markdown document. Save at the path the caller provides (typically \`docs/smoke-tests/<test-name>.md\`). Use markdown headings to demarcate the three sections; do not invent a fourth.

---

## Worked example (illustrative — fictional domain, do not copy content)

A fragment from a made-up "tea-clock daily reminder" feature. Names and copy are throwaway; the form is what to copy.

### Example Section 1 fragment

> ### Summary
>
> This Supervised Evolution script describes what should happen for the save-and-deliver flow of the tea-clock daily reminder.
>
> ### End point
>
> When the SE script is done, the user returns to find the settings page already open in their browser — the agent has pulled it up via CIC — showing the saved preferred reminder time (e.g. \`09:00\`) configured and persisting across reloads. At the scheduled time, a notification arrives on the user's device with the expected copy.
>
> ### Important details
>
> 1. Step 1 — User sets a preferred reminder time, behavior 2: the parsed time must be stored in the user's local timezone, not UTC.

### Example Section 2 — one step entry

> ### Step 1 — User sets a preferred reminder time
>
> > set my tea reminder to 9am please. it is important that the time is interpreted as my local timezone, not UTC.
>
> **Expected agent behaviors:**
>
> 1. [MD:CLAUDE-GM->triageCascade] the agent classifies the message as \`action\`.
> 2. [SB:teaclock->savePreferredTime] the agent persists \`09:00\` in the user's local timezone, flagging the local-timezone requirement as \`[IMPORTANT DETAIL]\` in the save objective's method.
>    2.1. [WP:preferences->writeRow] writes the row keyed by the calling user.
>
> **Agent response:** "Saved your tea reminder for 09:00 local time. You'll get a notification at 9 AM tomorrow."

### Example Section 3 entries

> ### [MD:CLAUDE-GM->triageCascade]
>
> The GM triage cascade routes every inbound user message to either a native discussion reply or the action cascade. Action-classification triggers on strong action verbs and on follow-ups to a pending action; everything else is discussion. Sub-classification of action lives downstream, not in triage.
>
> **Source:** \`CLAUDE-GM.md\` under the heading "GM triage cascade → Step 1 — Classify."

> ### [SB:teaclock->savePreferredTime]
>
> Reads a natural-language time expression from the user's message, normalizes to canonical \`HH:MM\` in the user's local timezone, and writes the value to the preferences row. Returns the parsed time along with confirmation copy.
>
> **Input schema:** \`{ userMessage: string, userId: string }\`
>
> **Prompt template:**
> - **Slug:** \`save-tea-time-v1\`
> - **Transmutation:** the prompt asks the model to parse a time expression from the user's natural-language message, validate it's a valid local time, and emit the canonical \`HH:MM\` form along with a short confirmation copy.
> - **Output schema:** \`{ time: "HH:MM", confirmCopy: string } | { error: string }\`

> ### [WP:preferences->writeRow]
>
> Writes a single column update to the user's preferences row. Idempotent at the column level. Returns the post-write row for the caller to echo.
>
> **Input schema:** \`{ userId: string, column: string, value: any }\`
>
> **Source:** \`libs/weapon/preferences/index.js\`. No LLM. Output: \`{ ok: true, row } | { ok: false, error }\`.

> ### [SB:teaclock->confirmReply]
>
> Composes the user-facing confirmation reply after a successful save. The reply mentions the saved time and the eta of the next expected reminder.
>
> **Input schema:** \`{ savedTime: "HH:MM", userTimezone: string }\`
>
> **Prompt template:**
> - **Slug:** \`tea-confirm-reply-v1\`
> - **Transmutation:** the prompt asks the model to compose a short confirmation in second person, mentioning both the saved time and the next-reminder eta.
> - **Output schema:** \`{ reply: string }\`

### What the example deliberately leaves out

No "build inventory." No "Score criteria" block. No per-criterion fix hints. No restatement of the SE loop. No locked-substrate facts. No "Owner:" line — the inline \`[TYPE:identifier->method]\` tags on each behavior bullet do that job. The Summary is one sentence; the end point describes what the user sees, not the agent's internal trace.
`.trim(),

  refLayoutContract: `
Reference: the locked 3-section layout, summary template, end-point rule, ownership notation, level-of-detail rule, and \`[IMPORTANT DETAIL]\` convention every DungeonMaster-authored SE rubric follows. Read before \`authorSERubric\`.

## Three sections, in this order

A rubric is read by humans (who skim Section 1, deep-read Section 2 when scoring) and by leaf-agents (who dispatch from Section 2 and verify substrate against Section 3). The split lets SE iterations rewrite Section 3 (the substrate-truth) without disturbing Sections 1–2 (the immovable target).

### Section 1 — Summary + end point

**Summary** is one sentence using the template:

> This Supervised Evolution script describes what should happen for **[scope]** of **[quest]**.

\`[scope]\` = the slice this rubric exercises (planning, execution, verification). \`[quest]\` = the overall objective. No scene-setter, no method recap.

**End point** describes what the user observes at the end of THIS rubric's run, from the user's perspective. The user doesn't actively verify — the agent surfaces the artifact and the user returns to find it already visible. **Default mode (locked): CIC shows the evidence** ("the user returns to find X already open in their browser — the agent has pulled it up via CIC"). Only switch surfaces if the user's instructions explicitly name a different one. Include enough detail that a human can verify the observation: artifact names, key copy, deliverables by key with one-line descriptions, the SE finale prompt if applicable.

**Important details** (optional cross-reference list) appears only when the user named must-have specifics. Each entry starts with the location, then a colon, then the description. Format per entry:

\`\`\`
N. Step <N> — <step title>, behavior <wbs id>: <one-line description>.
\`\`\`

Omit if no must-haves. Do not invent items.

### Section 2 — Steps

The storyboard. Each step has only two components: the mock user message, then the expected agent behaviors. The behaviors ARE the score criteria — a step passes when every numbered behavior is observable in the trace + reply.

Per step:

1. Mock user message — verbatim blockquote.
2. \`**Expected agent behaviors:**\` followed by a WBS-numbered list. Every bullet starts inline with the \`[TYPE:identifier->method]\` ownership tag, then a brief description. Nest sub-behaviors with dot notation (2.1, 2.2). When a behavior flags a user-emphasized must-have into quest data, the bullet describes that flagging — the actual \`[IMPORTANT DETAIL]\` marker lives in the quest's objective.method content (Section 3), not in the rubric bullet.
3. \`**Agent response:**\` — a labeled trailing entry after the behaviors list with the verbatim quoted user-visible reply (always from the GM, since GM is the conversation partner).
4. The next step's user message flows conversationally from this step's Agent response.

No "Owner:" header. No separate "Score criteria" section.

**Pin the result, not just the action (grounding truth).** Each behavior asserts the action AND the specific result it must produce. The result is the right answer the SE loop nudges toward; action-without-correct-result is a deviation, same as no action. The quest-creation step embeds the FULL first-version quest content (every objective's input/method/output, deliverables, strategic description, \`[IMPORTANT DETAIL]\` placements), sourced from the locked quest-UI mockup so the two never drift — because if the produced \`quests.objectives\` plan is wrong, every downstream step tests against a broken foundation. Other result-producing steps pin their expected concrete output. Every rubric concretely covers all five phases: clarify · create · do · escalate-and-resolve · report — each pinned to its right answer.

### Section 3 — Behavior logic

The encyclopedia of every \`[TYPE:identifier->method]\` referenced in Section 2. Each entry leads with one paragraph of plain prose (the substrate-truth), then schema/template details.

Required for every entry:

- Heading is the \`[TYPE:identifier->method]\` tag.
- One paragraph of plain prose first, before any schema or bullets.
- Input schema.
- For agent (LLM-bearing) owners: prompt template described as slug + 2–4 sentence transmutation NL + output schema.
- For non-agent owners: a \`**Source:**\` line pointing to file path / table.column / section anchor.

## Ownership notation

\`[TYPE:identifier->method]\` is the universal format for every owner reference. Used inline in Section 2 bullets AND as headings in Section 3.

| TYPE | Meaning | Identifier | Method |
|---|---|---|---|
| WP | Weapon | weapon name | exported function |
| SB | Skill book | skill book id | action key in toc |
| MD | Markdown file | filename without \`.md\` | section anchor / rule name |
| DB | Database | table name | column name or row pattern |

No spaces around the colon or arrow.

### Owner vs subject (load-bearing)

The tag names the **owner** — the entity *responsible for instructing* the behavior, NOT the thing acted on. Test: "who is telling this to happen?" = owner; the thing it happens to = subject (goes in prose, never as the leading tag).

- **DB tables are SUBJECTS, not owners, for framework behaviors** (status flips, scans, dispatch, stage transitions, comment-resolve, note-writes, row inserts). Owner = the weapon/skill-book action that instructs the change; the table is named in prose. Wrong: \`[DB:quests->objectives] objective 4 flips to attention\`. Right: \`[SB:questmaster->reportAttentionItems] surfaces objective 4, status \`attention\` in quests.objectives\`.
- **Exception: \`[DB:quests->objectives]\` IS the owner when the behavior is the worker doing the actual concrete job** — the objective's input/method/output content is literally the instruction; the worker has no other source of truth, so the objective content owns the concrete work.
- **Concrete work must be described.** A rubric that only narrates lifecycle meta-beats and never says what the worker physically does (fetch / OCR / scrape / render / deploy) is incomplete. Each real-work quest objective gets a \`[DB:quests->objectives]\`-owned concrete beat at the SE-step where it executes; the generic worker cycle lives in the \`[SB:worker->executeStep]\` Section 3 entry.

## Implementation-validation + user-channel verification (locked 2026-05-15)

A rubric must make every result-producing step testable for TWO failure classes, not one: (a) the agent's decision is wrong, and (b) the decision is right but the implementation / tool path fails (SSH won't connect, creds invalid/absent, deploy silently no-ops, migration blocked, API 500s, write doesn't persist). Authoring only "the agent decides X" leaves class (b) untestable — and class (b) is at least as common as (a). See \`supervisedEvolution.run\` "Real execution + CIC visual verification" rules 1b/1c.

For every result-producing step (anything that touches a tool, DB, external system, SSH, deploy, file, or UI), the authored grounding truth + that step's \`output[].verification\` MUST specify BOTH:

1. **The real implementation that must actually succeed, plus its concrete success signal** — name the tool path and the observable proof it *really worked*, not that it was attempted. Not "antibodies retrieved via SSH/SQL" but "the SSH session actually connected and the SQL returned ≥1 row for a known gene"; not "quest moved to review" but "the stage write persisted AND a fresh read returns \`review\` under the real credentials"; not "page deployed" but "the deployed URL serves 200 AND renders the expected element".
2. **The on-screen evidence the user would see** — what is visibly rendered, where, and the identifying copy / field / element a human confirms by looking. This is the ONLY acceptable verification framing. A step whose \`verification\` is a code/return-value proxy ("the weapon returns ok", "the function succeeds", "the row reads complete", "tests pass") is mis-authored — rewrite it as the on-screen observation the user would actually make. Per the alignment theorem (\`supervisedEvolution.run\` 1c/1d): non-visual verification systematically deviates from user-observed truth and is misaligned by construction. "On-screen, user's way" means the user's REAL channel — their connected browser via CIC, in their actual environment — NOT an agent-side proxy (\`Claude_Preview\`/preview_screenshot, headless/sandbox render, curl). Author the verification as what the user sees in their own browser; never author one an agent-side render could satisfy while the user's channel shows otherwise.

The End point (Section 1) already mandates the user-observed-on-CIC framing; this rule extends the same standard to EVERY result-producing step's per-step verification and adds the implementation-success-signal requirement. Do not author a result-producing step that cannot be failed for a tool/implementation failure, and never author a verification that a proxy could satisfy while the screen shows something wrong.

## Section precedence (authority hierarchy, locked)

The three sections describe the same quest at three altitudes — **strategy → tactical → mechanism**.

- **Section 1 — strategy.** What the user wants at the end. Most authoritative.
- **Section 2 — tactical.** The user→agent storyboard that lands the strategy.
- **Section 3 — mechanism.** The substrate truth behind each behavior. Most malleable.

When sections conflict, the lower-numbered section wins:

- If a Section 2 step doesn't achieve the Section 1 end point or an important detail → change Section 2 to fit Section 1.
- If a Section 3 behavior description can't produce a Section 2 observable beat → change Section 3 to fit Section 2.

SE iterations typically rewrite Section 3 (mechanism) and occasionally Section 2 (tactical). Section 1 changes only on explicit user direction; for an SE run it is the immovable target.

## Level of detail (locked)

Every behavior bullet contains only what the SE judging agent needs to compare actual vs expected. Strip rationale, exposition, and "because" tails — the substrate truth lives in the matching Section 3 paragraph. If a bullet reads "the agent does X because Y," the "because Y" clause must move to Section 3.

This rule is what keeps SE iterations cheap: behaviors are the immovable target, Section 3 is what SE rewrites. The more rationale leaks into Section 2, the more SE has to re-edit Section 2 each iteration.

## End point (user perspective, not artifact list)

The end point describes what the user OBSERVES at the end of the run — not an abstract list of artifacts produced, and not the user actively verifying. The agent does the surfacing; the user returns to find the artifact already visible.

**Default mode (locked): CIC shows the evidence.** The canonical framing is "the user returns to find X already pulled up in their browser — the agent has used CIC to open it." Apply this default to every rubric. Non-CIC surfaces (inbox, dashboard, file in editor, Slack message, CLI output, etc.) are allowed ONLY when the user's source instructions explicitly name them — do not invent them.

If the run is a planning rubric, the end point typically describes the user returning to find the just-created quest details page already open in CIC, with what's visible on the page (title, strategic-only description, objectives table rows with their input/method/output/verification, initial deliverables table, agent-response prompt). If the run is an execution rubric, the end point typically describes the user returning to find the changed live artifact already on screen.

The end point is allowed to be 3–8 sentences when the artifact has substantial structure (a quest with several deliverables, a page with multiple components). For simpler artifacts (a single setting that flipped, a single notification delivered) a one- or two-sentence end point is enough.

If \`supervisedEvolution.run\`'s finale prompt is expected to fire (planning rubric + SMART plan produced), mention it as one of the things the user sees at the end.

## \`[IMPORTANT DETAIL]\` convention

The \`[IMPORTANT DETAIL]\` marker belongs to the QUEST DATA, not to SE rubric bullets. When the user's mock message uses natural-language emphasis (e.g. "it is important that...", "must do...", "make sure..."), the agent's expected behavior is to flag that detail in the corresponding step's \`method\` content by embedding \`[IMPORTANT DETAIL]\` inline next to the must-have clause. The Section 2 bullet describes the flagging behavior — but the bullet itself does NOT carry \`[IMPORTANT DETAIL]\`. Example:

\`\`\`
Section 2 — Step X:
> user: "It's important that the promo copy is exactly 'Mix-and-match 5+ antibodies...'."

**Expected agent behaviors:**

2. [SB:questmaster->summarizeRelevantContext] the agent flags the promo-copy verbatim requirement as [IMPORTANT DETAIL] in the antibody-form objective's method.

Section 3 — corresponding step entry:
**Expected method content:**
"...Render promo banner. [IMPORTANT DETAIL] The banner reads verbatim: 'Mix-and-match 5+ antibodies from this panel, enjoy 50% off — quote emailed to you.'"
\`\`\`

Every important detail is mentioned **twice**: once by user NL in the mock message, once by agent behavior that flags it into objective.method. Section 1's "Important details" cross-reference list points to the Section 2 behavior that does the flagging.

Do not flag a detail \`[IMPORTANT DETAIL]\` unless the user actually emphasized it. Self-attested importance doesn't count.

## Agent response (explicit label)

Each step's Expected agent behaviors list ends with a separately labeled \`**Agent response:**\` line containing the verbatim quoted user-visible reply. The Agent response is NOT a numbered bullet — it's a labeled trailing entry. The response generator is always the GM (the conversation partner). Other agents (workers, MAs, SAs) never speak directly to the user; they surface to the GM, who composes the reply.

## Conversational flow

The mock user message in step N+1 must be a natural response to the Agent response from step N. The rubric reads as an actual chat. If step N+1's user message can't be connected to step N's Agent response, the steps are jumping flows — rework the prior reply or split the rubric into sub-threads.

## Why this layout

- **One-sentence summary, no exposition.** Cuts skim time to seconds. The template enforces that both the scope and quest are named.
- **End point from user perspective.** A rubric whose end point reads as "the agent emitted the correct trace" reads like an academic test plan. Anchoring on user observation forces the rubric to land at a human-checkable outcome.
- **WBS behaviors = criteria.** Holding two parallel lists (behaviors and criteria) forces them to mirror each other, and the second list drifts as the first changes. One list, scored as written, has no drift surface.
- **Inline ownership tags.** A reader can scan a step and immediately see which substrate produces each beat without scrolling back to a header.
- **NL paragraph first in Section 3.** SE iterates by nudging substrate (the paragraph) until the steps land. If the entry were schema-first, the entry would read as a contract and SE iterations would have nothing softer to nudge.
- **Minimum level of detail in bullets.** Less surface area for SE to disturb during iteration. Rationale lives once in Section 3, not duplicated across every step.
- **\`[IMPORTANT DETAIL]\` flags + Section 1 list.** User can audit must-haves in one place (Section 1 list) and confirm each is present in Section 2 by scanning for the matching flag.

## Excluded from every rubric

- A multi-sentence Summary or scene-setter — Summary is one sentence per template.
- A SEPARATE / parallel "Score criteria" section or scoring table — behaviors are the criteria (per-behavior inline \`Severity —\` annotations are allowed and are the sanctioned failure-class form; the ban is on a second parallel list, not on co-located severity).
- An "Owner:" header line per step — inline tags do that job.
- Rationale or exposition in behavior bullets — strip "because" clauses; the truth lives in Section 3.
- SE execution mechanics — owned by \`supervisedEvolution.run\`.
- Locked-everywhere substrate (the \`objectives.output\` jsonb shape, the \`deliverables.kind\` enum, initial stage = \`execute\` under the new 5-stage flow, trace envelope, work-on-main rule).
- Per-criterion fix hints.
- Run ledger.
- Historical verbatim references / migration-moment HTML comments.
- Build-inventory tables that mirror the skill book registry.
- Unpaired \`[IMPORTANT DETAIL]\` flags or unpaired Section 1 important-details entries.

---

## Worked example (illustrative — different fictional domain, do not copy content)

A fragment from a made-up "snack-shelf inventory" feature so the reader sees the pattern in a second instance.

### Example Section 1 fragment

> ### Summary
>
> This Supervised Evolution script describes what should happen for the one-tick restock flow of the snack-shelf inventory.
>
> ### End point
>
> When the SE script is done, the user returns to find the inventory page already open in their browser — the agent has pulled it up via CIC — showing shelf A3's new on-hand quantity reflecting the just-logged restock. No manual refresh was required. The stale-cache audit log shows an invalidation entry for the matching shelf key.
>
> ### Important details
>
> 1. Step 1 — Supplier posts a restock, behavior 2.2: the cache invalidation key must be the specific shelf id, not a wildcard prefix.

### Example Section 2 — one step entry

> ### Step 1 — Supplier posts a restock
>
> > supplier sent me a fresh case of cocoa — log the restock for shelf A3. important: invalidate the cache for shelf A3 specifically, not a wildcard prefix.
>
> **Expected agent behaviors:**
>
> 1. [MD:CLAUDE-GM->triageCascade] the agent classifies the message as \`action\`.
> 2. [SB:supplierPortal->submitRestock] the agent records the restock event tied to shelf A3, flagging the shelf-specific-invalidation requirement as \`[IMPORTANT DETAIL]\` in the invalidate objective's method.
>    2.1. [WP:inventory->writeEvent] appends the event row and updates the shelf's on-hand quantity atomically.
>    2.2. [WP:cache->invalidateByKey] invalidates the cache entry for shelf A3.
>
> **Agent response:** "Logged the cocoa restock on shelf A3. New on-hand quantity is 24. Cache for shelf:A3 invalidated."

### Example Section 3 entries

> ### [WP:inventory->writeEvent]
>
> Appends a single inventory-change event (restock, sale, adjust, audit) to the events log and updates the on-hand quantity row keyed by shelf. Atomic — either both writes commit or neither does. Emits a cache-invalidation hint with the affected shelf key on success.
>
> **Input schema:** \`{ shelfId: string, eventKind: "restock" | "sale" | "adjust" | "audit", quantityDelta: number, source: string }\`
>
> **Source:** \`libs/weapon/inventory/index.js\`. No LLM. Output: \`{ ok: true, eventId } | { ok: false, error }\`.

> ### [WP:cache->invalidateByKey]
>
> Drops cache entries matching a key prefix and writes an audit entry recording the invalidation reason and the caller. Supports both exact-key and prefix invalidations; the audit entry distinguishes the two.
>
> **Input schema:** \`{ keyPrefix: string, reason: string, mode?: "exact" | "prefix" }\`
>
> **Source:** \`libs/weapon/cache/index.js\`. No LLM.

### What the example deliberately leaves out

No restatement of the SE method. No "Score criteria" block. No "Owner:" header. No mock cache-infra description. No rationale in bullet 2 saying "because cache misses cause stale renders." The Summary is one sentence; the End point describes the user's observation, not the agent's internal trace.
`.trim(),
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->
