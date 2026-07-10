# blacksmith

<!-- GUILDOS-SKILL name="blacksmith" -->

<!-- GUILDOS-DESC -->
Forge new weapons and skill books. Reads a task description, decides whether a new weapon/skill book is needed, produces and reviews a blueprint, then forges the weapon file via the claudecli weapon and verifies it with a browser smoke test. Also patches existing skill book data sections and holds the forging discipline + tool/ACI design rules every new weapon must follow.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/blacksmith/SKILL.md -->
```markdown
---
name: blacksmith
description: >-
  Forge new weapons and skill books. Reads a task description, decides whether a
  new weapon/skill book is needed, produces and reviews a blueprint, then forges
  the weapon file via the claudecli weapon and verifies it with a browser smoke
  test. Also patches existing skill book data sections and holds the forging
  discipline + tool/ACI design rules every new weapon must follow.
---

# Blacksmith

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/blacksmith/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto blacksmith <action>` (or read the source directly).

## Actions

- **forgeWeapon** — Use the claudecli weapon to write the content in libs (skill-book howTo in libs/skill_book/<name>/index.js and/or executable libs/weapon/<name>), then regenerate the .claude/skills index (buildIndex.mjs) and verify via a browser smoke test.
- **patchSkillBook** — Incrementally update an existing skill's howTo prose in its libs source (libs/skill_book/<name>/index.js) with a verified addition, then regenerate the .claude/skills index. Use this instead of editing files directly.
- **refForgeDiscipline** — Reference: rules and conventions for forging new skills + weapons (purpose-not-tactic naming, directory-driven .claude/skills discovery — no hub edit, six-verb naming, TOC requirements, line-of-responsibility split). Read this before writeBlueprint or forgeWeapon.
- **refToolDesign** — Reference (LOCKED 2026-06-12): the tool/ACI design rules every new weapon must follow — workflow-shaped (not raw-API-mirror), token-lean outputs (pagination/limit/filter with default caps), namespaced names, high-signal natural-language returns over opaque IDs, and steering error messages. Adopted from Anthropic's tool-design + context-engineering guidance. Read alongside refForgeDiscipline when forging.
- **reviewBlueprint** — Inspect the blueprint in quest inventory. Escalate to the user if credentials or setup are missing; otherwise mark ready to forge.
- **writeBlueprint** — Read the task description and decide whether a new weapon is needed. If yes, produce a blueprint JSON; if no, return skip.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/blacksmith/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/skill_book/blacksmith`

Rationale layer (per CLAUDE.md §6). The WHY behind the forging disciplines — the
load-bearing reasoning that the `refForgeDiscipline` / `refToolDesign` howTos state
as rules but do not justify. howTo says *what to do*; this says *why it breaks
without it*. Append; supersede in place when a decision changes.

---

## 2026-07-05 — forge writes CONTENT to `libs/`, then regenerates the `.claude/skills` INDEX; discovery is directory-driven (SUPERSEDES the 2026-06-13 hub-registration ED below)

**Decision.** Forging a skill is now: (1) write the content in `libs/` —
skill-book howTo in `libs/skill_book/<name>/index.js`, executable code in
`libs/weapon/<name>`; (2) for a NEW skill, hand-write only the frontmatter
`description` in `.claude/skills/<name>/SKILL.md`; (3) run
`node libs/weapon/skillBooks/buildIndex.mjs` to (re)generate the thin discovery
index. Discovery (`gearUp`/preflight/ownership) scans `.claude/skills` directly
(`scanClaudeSkillsFromDisk`), so the skill is routable the moment the index is
regenerated — **no hub edit**. Do NOT hand-author per-action `.md` content in
`.claude/skills` (buildIndex deletes it; content lives once in `libs/`).

**Rejected alternative.** (a) The prior model (2026-06-13, below): wire every new
book into the static `SKILL_BOOKS` hub in `libs/skill_book/index.js` for routing.
(b) Author the howTo prose directly in `.claude/skills/<name>/*.md` (verbatim
copies — the initial 2026-07-04 build; drifts from libs).

**Why.** The whole-registry move to `.claude/skills` (see
`libs/weapon/skillBooks/engineering_decisions.md`, 2026-07-05) makes discovery a
filesystem scan of the generated index tree, so the static hub is no longer the
routing gate. Keeping content single-sourced in `libs/` and the index generated
kills the dual-maintenance drift AND lets one repo serve many runtimes (a Cursor
target is just another buildIndex output).

**Consequence if violated.** Wiring a book into the hub "for discovery" now does
nothing for routing (wasted, misleading edit). Hand-writing howTo into
`.claude/skills` gets silently deleted on the next buildIndex run. Forgetting to
run buildIndex after editing libs leaves the index stale — the skill's actions
won't reflect the new content in gearUp.

**NOTE — supersedes the 2026-06-13 hub-registration ED immediately below:** that
ED's "static hub is the routing gate; disk presence is not enough" is no longer
true. `gearUp`/preflight read `.claude/skills`, not `SKILL_BOOKS`. The hub is
retained only for `scanSMARTPlans` + JS-action EXECUTION (`getSkillBook`), not
discovery. Kept below for history.

---

## 2026-06-13 — [SUPERSEDED 2026-07-05, see above] a new skill book MUST be registered in `libs/skill_book/index.js`; disk presence is NOT enough

**Decision.** Adding a skill book requires editing `libs/skill_book/index.js`
(imports + the `SKILL_BOOKS` hash + `ADVENTURER_REGISTRY`), even though
`dumpOwnershipMap` is generated by a live filesystem scan that needs no edits.

**Rejected alternative.** "If it's on disk under `libs/skill_book/<name>/`, it's
discoverable" — treat the directory scan as the single source.

**Why.** The two surfaces are deliberately split: the **disk scan** drives the
ownership map (always fresh, zero-maintenance); the **static hub** is what
`gearUp`/preflight actually read for routing. The static hub is the gating point —
a book on disk does not become routable until a human wires it in, which keeps
deployment and discovery decoupled.

**Consequence if violated.** A well-named book sits on disk but never surfaces in
`gearUp`/preflight (observed 2026-06-13): the agent can't find it and mazes.

_(Superseded 2026-07-05: the rejected alternative — directory scan as the single
source — is now exactly how discovery works, via `.claude/skills`.)_

---

## 2026-06-13 — the weapon↔skill-book reference is one-way: skill book → weapon, never reverse

**Decision.** A **skill book action** decides *which weapon to call, when, with
what situational adjustments* (procedure/orchestration); a **weapon action** does
*what it does + I/O shape + always-on plumbing* (substrate: auth, retries,
pagination). References flow skill book → weapon only.

**Rejected alternative.** Symmetry — either layer may call the other, or blend
orchestration into weapons "for convenience."

**Why.** Direction is what keeps each tree coherent. Let weapons call skill books
and weapons become bespoke orchestrators; let skill books hold plumbing and they
become unreadable infrastructure. Both then drift and the substrate/procedure
split (the thing that makes `gearUp` discovery meaningful) collapses.

**Consequence if violated.** Weapons bloat with orchestration logic; skill books
turn into infra; the registry's two-tier discovery degrades.

---

## 2026-06-13 — six-verb weapon naming; banned synonyms

**Decision.** Weapon functions use only `read · write · delete · search ·
transform · normalize`. Banned: `get/fetch/load/list/find/create/update`.

**Rejected alternative.** Any descriptive English verb per author's taste.

**Why.** Past ~40 weapons, synonym sprawl (`getTask` vs `readTask` vs `fetchTask`
vs `listTasks`) is the dominant wrong-tool-selection failure. A closed verb set
collapses the search space so an agent reaches for the one correct name.

**Consequence if violated (observed 2026-06-13).** Agents confuse near-synonym
actions, pick the wrong weapon, and maze.

---

## 2026-06-05 — name a skill book for its PURPOSE, not its tactic

**Decision.** A skill book is named for what it accomplishes
(`slideDeckAndVideoGeneration`), never for its implementation tactic
(`htmlStudio` — the banned exemplar). The tactic lives in the title/description.

**Rejected alternative.** Name by tech stack / implementation.

**Why.** `gearUp` scores token overlap against the *purpose* an agent searches with
("generate a slide deck"), not the tactic. A tactic name simply never matches the
query.

**Consequence if violated.** Purpose searches miss the book; agents rescan or
re-forge a duplicate.

---

## 2026-06-12 — every new weapon follows the five agent-centric tool-design rules (LOCKED; adopted from Anthropic guidance)

**Decision.** New weapons must be (1) workflow-shaped, not raw-API mirrors;
(2) token-lean (pagination/limit/filter with sane defaults); (3) namespaced +
six-verb named; (4) high-signal natural-language returns, not opaque IDs;
(5) steering error messages. Sourced from Anthropic's "Writing tools for agents" +
"Effective context engineering."

**Rejected alternative.** Mirror the underlying vendor API one-to-one.

**Why.** A raw-API mirror dumps unbounded rows (context blowout), returns bare keys
(ID hallucination), and clones overlapping vendor endpoints with confusing names
(tool confusion). Agent-centric shaping is the override that prevents all three.

**Consequence if violated.** Weapons return full schemas, agents ask "what's the ID
for…?", list calls blow the context window, and tool selection degrades.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/blacksmith/index.js -->
```js
/**
 * Blacksmith skill book — pure text registry.
 * Each action value is a natural-language prompt describing how the adventurer
 * should perform the action. The adventurer uses weapons (claudecli, database)
 * inline to execute. No JS logic here.
 */

export const skillBook = {
  id: "blacksmith",
  title: "Blacksmith",
  description: "Forge new weapons and skill books.",

  toc: {
    writeBlueprint: "Read the task description and decide whether a new weapon is needed. If yes, produce a blueprint JSON; if no, return skip.",
    reviewBlueprint: "Inspect the blueprint in quest inventory. Escalate to the user if credentials or setup are missing; otherwise mark ready to forge.",
    forgeWeapon: "Use the claudecli weapon to write the content in libs (skill-book howTo in libs/skill_book/<name>/index.js and/or executable libs/weapon/<name>), then regenerate the .claude/skills index (buildIndex.mjs) and verify via a browser smoke test.",
    patchSkillBook: "Incrementally update an existing skill's howTo prose in its libs source (libs/skill_book/<name>/index.js) with a verified addition, then regenerate the .claude/skills index. Use this instead of editing files directly.",
    refForgeDiscipline: "Reference: rules and conventions for forging new skills + weapons (purpose-not-tactic naming, directory-driven .claude/skills discovery — no hub edit, six-verb naming, TOC requirements, line-of-responsibility split). Read this before writeBlueprint or forgeWeapon.",
    refToolDesign: "Reference (LOCKED 2026-06-12): the tool/ACI design rules every new weapon must follow — workflow-shaped (not raw-API-mirror), token-lean outputs (pagination/limit/filter with default caps), namespaced names, high-signal natural-language returns over opaque IDs, and steering error messages. Adopted from Anthropic's tool-design + context-engineering guidance. Read alongside refForgeDiscipline when forging.",
  },

  writeBlueprint: `
You are planning whether a new weapon is needed to complete a task.

Inputs you must gather:
- The quest description (from quest inventory or quest.description).

Decide:
- "forge" — an external system or local tool must be called and no existing weapon in libs/weapon/ covers it.
- "skip"  — an existing weapon already covers the need, or the quest is pure in-repo code work.

If "forge", output JSON with this exact shape:
{
  "action": "forge",
  "weapon_name": "<lowercase, no spaces>",
  "external_system": "<e.g. Stripe, Zoho Books, local filesystem>",
  "auth_type": "env_var | oauth | none",
  "credentials_needed": "<env var names or 'none'>",
  "requires_user_setup": true | false,
  "user_setup_reason": "<why the user must act, if any>",
  "setup_steps": ["Step 1: ...", "Step 2: ..."],
  "actions_to_implement": ["actionName1", "actionName2"],
  "msg": "<one-sentence summary>"
}

If "skip", output:
{ "action": "skip", "msg": "<why no weapon is needed>" }

Save the blueprint to quest inventory under key "blueprint".
`.trim(),

  reviewBlueprint: `
You are reviewing the blueprint produced by the writeBlueprint action. Read it from quest inventory under key "blueprint".

If blueprint.action === "skip", pass through with decision "skip".

Otherwise, escalate (decision "escalate") if ANY of:
- blueprint.requires_user_setup === true
- blueprint.weapon_name or blueprint.external_system is empty
- blueprint.actions_to_implement is empty

When escalating, post a quest comment of this form:

  Weapon blueprint review — user input needed:
  1. <reason>
  2. <reason>

  Weapon: <weapon_name>
  External system: <external_system>

  Please add the following in the Formulary (/council-hall/formulary):
    • Key name: <credentials_needed key>
    • Description: <credentials_needed>

  Setup steps:
    1. <setup_steps[0]>
    2. <setup_steps[1]>

  Once saved, reply 'done' to this comment to resume.

Otherwise return decision "ready" with the blueprint unchanged.
`.trim(),

  forgeWeapon: `
Use the claudecli weapon (import { invoke } from "@/libs/weapon/claudecli") to forge the weapon.
Read weapon_spec from input or derive it from quest inventory key "blueprint":
  { name, description, codeGoal, actions[], targetSkillBook? }

Send this prompt to claudecli.invoke():

---
You are writing production-quality JavaScript for a Next.js 15 project called GuildOS.
The project root is the current working directory. Use ES module syntax (import/export). No TypeScript.

## Task
<codeGoal>

## Project conventions (index-and-point model, refactor 2026-07-05)
- **CONTENT lives in \`libs/\` (single source of truth):** skill-book howTo prose in \`libs/skill_book/<name>/index.js\` (a \`toc\` object + one howTo string per action; JS-backed actions add functions), and executable weapon code in \`libs/weapon/<name>/index.js\` (with its \`toc\` export). This is where you WRITE.
- **DISCOVERY INDEX is \`.claude/skills/<name>/SKILL.md\` — GENERATED, not hand-written content.** It is a thin index: YAML frontmatter (\`name:\`, \`description: >-\` written PURPOSE-first for discovery) + \`## Actions\` (one bullet per toc action) + a pointer to the libs source + \`## Weapons\` pointers. Regenerate it with \`node libs/weapon/skillBooks/buildIndex.mjs\` after editing libs — do NOT hand-author per-action \`.md\` content files (the generator deletes them; the howTo lives ONLY in libs).
- Name a skill for its PURPOSE (e.g. \`slideDeckAndVideoGeneration\`), NOT its tactic (e.g. \`htmlStudio\`) — agents search by purpose; the tactic lives in the title/details.
- **Forge order:** (1) write the content in \`libs/skill_book/<name>/index.js\` (and/or the executable \`libs/weapon/<name>/index.js\`); (2) for a NEW skill, hand-write the frontmatter \`description\` in its \`.claude/skills/<name>/SKILL.md\` (the one hand-tuned discovery field — buildIndex preserves it); (3) run \`buildIndex.mjs\` to (re)generate the index. Discovery (gearUp/preflight/ownership) then picks it up with zero hub edits.
- Follow the patterns in existing files exactly (see \`.claude/skills/gmail/SKILL.md\` for the generated index shape and \`libs/skill_book/gmail/index.js\` for the content shape).

## Weapon TOC (required for every weapon)
Every weapon MUST export a \`toc\` constant at the top of its \`index.js\`. It maps each agent-facing exported function to a one-line description. Used by agents to discover capabilities at boot without reading the whole file.

\`\`\`javascript
export const toc = {
  searchTasks: "Search Asana tasks across projects with filters.",
  readTask: "Read full details of a single Asana task by id.",
  writeTask: "Create or update an Asana task (upsert semantics).",
};
\`\`\`

Rules:
- **Include:** only agent-facing exports — functions an adventurer might call for a quest.
- **Exclude:** internal plumbing (token acquisition, auth refresh, URL builders, path helpers). Per discipline rule 11, these are invisible to agents.
- **Shape:** \`{ functionName: "one-line description" }\`. Description starts with a verb, only add detail that disambiguates alternatives.
- **Location:** top of \`index.js\`, immediately after the module JSDoc. Exported constant so registry scripts can read it.
- **Naming:** function names follow the six-verb rule (read/write/delete/search/transform/normalize). Banned synonyms: get/fetch/load/list/find/create/update.

If the weapon wraps multiple resources that share a shape, parameterize one function (e.g. \`search({ module, query })\`) rather than adding per-entity variants.

## Discipline
Follow the "Skill book + weapon discipline" section in CLAUDE.md — especially the six-verb action naming rule and the line-of-responsibility split between weapons and skill books.

## Spec
Name: <weaponName>
Target skill book (optional): libs/skill_book/<targetSkillBook>/index.js
Actions: <actions joined by comma>

## Rules
- Do NOT create helper functions for logic used only once — inline it.
- Do NOT create files beyond what is needed.
- If modifying an existing file, keep changes minimal — add only, do not rewrite.
- Match the code style of the file you are editing.

## Output format
Your ENTIRE output must be a single valid HTML document. No markdown. No prose outside HTML tags.
- Dark theme: background #1a1a1a, text #d1d5db, headings #a78bfa, code blocks #111
- Start with <!DOCTYPE html>
- Summary → per-file heading with full path → complete file content in <pre><code> → Verification section at the end
---

Save the returned HTML to quest inventory under key "forge_report".

Then verify the forge by invoking claudecli.invoke() a second time with Chrome Extension MCP tools enabled:
- Navigate to {SITE_URL}/town-square/forge and confirm the weapon appears
- Take a screenshot, check for obvious errors
- Return JSON: { "verified": boolean, "reason": "<one sentence>", "screenshotId": "<id or null>" }

Save verification result to quest inventory under key "forge_verification".
`.trim(),

  refForgeDiscipline: `
Reference rules for forging new weapons and skill book actions. Read this before invoking writeBlueprint or forgeWeapon.

## Naming
- Skill book action names start with a verb (read/write/search/transform/normalize/delete + domain-specific verbs like escalate/comment/triage/dispatch) OR with \`ref\` (reference content loaded by multiple actions).
- Weapon function names follow the six-verb rule. Banned synonyms: get/fetch/load/list/find/create/update.
- Prefer parameterized multipurpose actions (\`search({ module, query })\`) over per-entity siblings when shapes match.

## Skill naming & discovery (index-and-point model, refactor 2026-07-05)
- **Name a skill for its PURPOSE, not its tactic.** The folder name is what agents search by, and they usually know the purpose ("make a slide deck and a video") but only sometimes the tactic. Use a purpose name like \`slideDeckAndVideoGeneration\`, NOT a tactic name like \`htmlStudio\`. The tactic belongs in the title parenthetical and the toc/howTo details. Front-load purpose keywords in the SKILL.md frontmatter \`description\` and in every toc description — \`gearUp\` ranks by plain-token overlap against those.
- **Two layers, one content source.** CONTENT (howTo prose, executable code) lives ONCE in \`libs/\`. The \`.claude/skills/<name>/SKILL.md\` INDEX is a generated pointer to it — thin frontmatter + \`## Actions\` list + libs pointer. Never duplicate howTo prose into \`.claude/skills\`; regenerate the index with \`node libs/weapon/skillBooks/buildIndex.mjs\` instead. This is what lets one repo serve many runtimes (Claude Code native skills today; a Cursor/AGENTS.md target is just another buildIndex output).
- **Discovery is directory-driven — regenerate the index, no hub edit.** \`gearUp\`, the preflight router, AND the ownership map all scan \`.claude/skills/\` via \`scanClaudeSkillsFromDisk\`. After you write content in libs and run buildIndex, the skill is instantly routable. The old static \`SKILL_BOOKS\` hub in \`libs/skill_book/index.js\` is NO LONGER the routing source — do not add entries there for discovery.
  - (Adventurer/quest EXECUTION of JS-backed actions still loads code from \`libs/skill_book/<name>/index.js\` via \`getSkillBook\`; keep that file for books that expose executable \`adventurerActions\`.)
- **Verify discovery after creating:** run \`node libs/weapon/skillBooks/cli.mjs "<the skill's purpose in plain words>"\` and confirm the new skill ranks at/near the top. If it does not appear, its SKILL.md description / libs toc lack purpose keywords.

## TOC requirements
- Every weapon exports a top-level \`toc\` constant mapping each agent-facing function to a one-line description.
- Skill books have a \`toc\` object on their definition.
- TOC descriptions start with a verb and disambiguate alternatives.
- \`[SMART]\` prefix only for plans that pass the qualification checklist in \`SMARTProjectManagement.refSMARTQualification\`.

## Line-of-responsibility split
- Skill book action = which weapon to call, when, with what situational adjustments.
- Weapon action = what it does, input/output shape, always-on plumbing (auth refresh, retries, pagination — hidden from the agent).
- One-way reference: skill books → weapons. Never the reverse.

## Forging order
1. \`writeBlueprint\` → produce a blueprint JSON saved to quest inventory.
2. \`reviewBlueprint\` → check for missing credentials or setup; escalate to user if needed.
3. \`forgeWeapon\` → write the weapon file via claudecli, verify with a browser smoke test.

## For SMART plans specifically
Authoring or revising a [SMART]-prefixed skill book action follows \`SMARTProjectManagement.refSMARTFormat\` (the canonical six-section structure) and \`SMARTProjectManagement.refSMARTQualification\` (the four-item qualification). Blacksmith only patches skill books via \`patchSkillBook\`; the SMART format definitions live in the SMARTProjectManagement skill book.
`.trim(),

  refToolDesign: `
Tool / ACI (agent-computer interface) design rules every new weapon MUST follow. LOCKED 2026-06-12, adopted from Anthropic's "Writing tools for agents" + "Effective context engineering" guidance (scribe best-practice adoption). Read alongside refForgeDiscipline when forging a weapon.

The agent's tools are its hands — design them with the same care as a human UI. Five rules:

1. **Workflow-shaped, not raw-API-mirror.** A weapon function should map to a task the agent actually performs, consolidating multi-step operations into one call — \`searchContacts({ query })\` that returns the rows the agent needs, NOT a thin \`listContacts\` + \`getContact\` the agent has to chain. Fewer, higher-level tools beat a 1:1 mirror of the underlying API: they cut context churn and the agent's intermediate-result juggling.

2. **Token-lean outputs.** Tool output competes for the finite context budget. Every list/search return MUST support pagination/limit/filter and carry a sensible DEFAULT cap (e.g. \`limit = 20\`) so an unbounded call can't dump thousands of rows into context. Prefer a "concise" response shape; offer a "detail" flag only when needed. Truncate long fields with a marker rather than returning them whole.

3. **Namespaced, six-verb names.** Keep the existing six-verb rule (refForgeDiscipline) AND namespace related functions under a common prefix so the agent disambiguates overlapping tools (\`pim*\`, \`quest*\`). Clear names are the cheapest way to stop wrong-tool selection.

4. **High-signal, natural-language returns.** Return interpretable values — names, labels, human-readable statuses — not opaque IDs the agent then has to resolve. Opaque identifiers waste reasoning and invite hallucination. If an ID is unavoidable, return it ALONGSIDE its human-readable name.

5. **Steering error messages.** On failure, return an actionable message that tells the agent how to fix it (the correct argument, the missing field, the alternative path) with a correct-format example — never a bare code or stack. Agents self-correct from good error text; they loop on opaque ones.

**Audit hook:** when reviewing or refactoring an existing weapon, check it against these five. The dominant fixes are (2) add a default limit to an unbounded list return and (4) swap an opaque-ID return for a name+ID pair. Use \`patchSkillBook\`-style discipline: verified change, preserve structure.
`.trim(),


  patchSkillBook: `
You are updating an existing skill book in place. The MA proposing this change has already vetted the substance — your job is to apply the edit correctly, preserve file structure, and enforce verification rules.

## Input payload

\`\`\`json
{
  "skillBook": "<id, e.g. 'dailies'>",
  "action": "<action key, e.g. 'triageEmailStep1'>",
  "section": "<human-readable section heading the addition belongs under>",
  "proposedAddition": "<the exact text to insert>",
  "evidence": { "<verification fields per the rules below>" }
}
\`\`\`

## Verification rules

Before editing, verify the proposal against the rules for its section type:

1. **Pattern lists (Patterns to Ignore / similar)** — evidence must show \`occurrenceCount >= 2\` in the past 6 months. Run \`gmail.searchInbox\` (or equivalent) to confirm. Reject if single-occurrence or one-off.

2. **Highlight / positive-signal patterns** — the proposed pattern must be decidable from sender category + subject alone, with no body read required and no contact-specific names or emails. Reject anything that violates this.

3. **Cross-cutting rules** — escalate to GM rather than patching unilaterally. Cross-cutting changes belong in the CLAUDE.md Insights buffer for next \`dailies.curateInsightsBuffer\` pass.

## Edit procedure

1. Read the target howTo in its libs source \`libs/skill_book/<skillBook>/index.js\` — the action's howTo string (the single content source; index-and-point model 2026-07-05). Locate the section by its heading. If the addition is a whole new action, add a new \`toc\` key + howTo string (and a function if it needs runtime logic).
2. Confirm the proposed addition is not a duplicate (substring match against existing entries).
3. Insert under the matching heading, preserving alphabetical or chronological ordering already in use.
4. Preserve file structure: the template literal's trailing \`.trim()\`, comment blocks, surrounding whitespace.
5. Smoke test: \`node --check libs/skill_book/<skillBook>/index.js\`, then regenerate the discovery index: \`node libs/weapon/skillBooks/buildIndex.mjs\`.
6. Return a confirmation object:

\`\`\`json
{
  "applied": true,
  "skillBook": "...",
  "section": "...",
  "additionPreview": "<first 80 chars>",
  "verification": { "evidenceCount": N, "duplicate": false, "parseOk": true }
}
\`\`\`

If rejected, return:

\`\`\`json
{ "applied": false, "reason": "<which rule failed and how>" }
\`\`\`

## Discipline

- Never edit beyond the proposed section. If the file is malformed in another place, surface it as a separate issue.
- Never invent new section headings without GM approval — only patch under existing ones.
- One proposal per SA dispatch. Batch proposals are not supported — each section update is its own SA call so each can pass/fail independently.
`.trim(),

};

export default skillBook;
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
