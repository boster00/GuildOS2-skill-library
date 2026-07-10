# justDoIt

<!-- GUILDOS-SKILL name="justDoIt" -->

<!-- GUILDOS-DESC -->
Freestyle fallback skill book. Triggers on user phrases: JDI, jedi, just do it, wing it. Defaults to CIC for browser, native tools for local. Enforces visual verification on every result. Grows over time to tighten freestyle behavior.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/justDoIt/SKILL.md -->
```markdown
---
name: justDoIt
description: >-
  Freestyle fallback skill book. Triggers on user phrases: JDI, jedi, just do it,
  wing it. Defaults to CIC for browser, native tools for local. Enforces visual
  verification on every result. Grows over time to tighten freestyle behavior.
---

# Just Do It (JDI)

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/justDoIt/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto justDoIt <action>` (or read the source directly).

## Actions

- **refDiscipline** — Reference: triggers, defaults, tab hygiene, and the visual verification mandate. Read this on first JDI invocation in a session.
- **reportResult** — Report a completed result so the user can review it in one click. If the result has a renderable URL or page, open it in CIC and leave the tab visible. If the result is a file, AO asset, comment, or data record without a URL, reply in chat with the exact location (path, AO id + slug, etc.).
- **verify** — Mandatory before claiming any work done. Dispatch SA-verify with the result URL + the success criterion. SA opens in CIC, screenshots, judges visually, returns pass/fail. On fail, surface to GM — never report done.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/justDoIt/index.js -->
```js
/**
 * Just Do It (JDI) skill book — pure text registry.
 *
 * Freestyle / native-mode fallback. When the user signals "wing it" by saying
 * any of: "JustDoIt", "JDI", "jedi", "just do it", "wing it", "use what you have" —
 * GM routes through this skill book. Defaults to CIC for browser work and native
 * tools (Read/Edit/Write/Bash) for local work.
 *
 * The book starts minimal — visual verify + reporting. More instructions
 * accumulate over time to tighten "wing it" behavior. Other skill books
 * (especially leaf actions doing browser work) may reference this book
 * instead of inlining CIC MCP tool calls.
 */

export const skillBook = {
  id: "justDoIt",
  title: "Just Do It (JDI)",
  description: "Freestyle fallback skill book. Triggers on user phrases: JDI, jedi, just do it, wing it. Defaults to CIC for browser, native tools for local. Enforces visual verification on every result. Grows over time to tighten freestyle behavior.",

  toc: {
    verify: "Mandatory before claiming any work done. Dispatch SA-verify with the result URL + the success criterion. SA opens in CIC, screenshots, judges visually, returns pass/fail. On fail, surface to GM — never report done.",
    reportResult: "Report a completed result so the user can review it in one click. If the result has a renderable URL or page, open it in CIC and leave the tab visible. If the result is a file, AO asset, comment, or data record without a URL, reply in chat with the exact location (path, AO id + slug, etc.).",
    refDiscipline: "Reference: triggers, defaults, tab hygiene, and the visual verification mandate. Read this on first JDI invocation in a session.",
  },

  verify: `
Before claiming any work as done — including "the report is rendered," "the deliverable is live," "the PR is open," "the file is written" — you MUST dispatch SA-verify and pass its check.

## When this applies
- Any time you would otherwise write "done", "verified", "rendered", "ready" in a response to GM or user
- Any time a deliverable has a URL, file path, screenshot, or other renderable artifact
- Exception: pure-data outputs with no visual rendering (e.g. a JSON record returned to GM) — in those cases verification is reading the data back via the relevant weapon and confirming shape/content matches the claim

## Procedure
1. Identify what is being claimed and what success looks like (the "claim" string).
2. Dispatch SA-verify with \`{ resultUrl | resultPath, claim, agent }\`. SA opens the result in CIC, screenshots, judges visually against the claim, returns \`{ pass: boolean, reason: string }\`.
3. If pass → proceed to report.
4. If fail → DO NOT report done. Surface the failure to GM with SA-verify's reason; GM decides retry, redirect, or escalate.

## Anti-patterns
- Claiming "verified" because the HTTP status was 200 or the file write succeeded. Status code is not a visual verdict.
- Claiming "verified" because you wrote it yourself and the tool didn't error. Self-trust without inspection is the failure mode this rule prevents.
- Skipping SA-verify because the deliverable is "obviously correct". If it's obvious, the verify pass is cheap; run it.

The phrase "verified" / "rendered" / "done" / "ready" without an SA-verify pass attached is a discipline violation. GM corrects.
`.trim(),

  reportResult: `
Goal: the user should be able to reach the result in one click. No hunting through chat history or guessing where the artifact lives.

## Two cases

**Case A — result has a renderable URL or page** (proving-grounds page, GM Room, quest detail, Asana task, Notion doc, etc.):
1. Open the URL in CIC (\`mcp__Claude_in_Chrome__navigate\`).
2. Leave the tab visible — do NOT close it. The user will visit Chrome to inspect.
3. Reply in chat with a one-line pointer: "Opened in CIC at <url>."
4. Visually confirm via SA-verify before stating it's open (case A always pairs with verify).

**Case B — result has no URL** (a file edit, an AO asset update, a comment posted to a task, a data record inserted):
1. Reply in chat with the exact location, formatted so the user can click or copy-paste:
   - File: \`docs/path/to/file.md\` (relative to repo root)
   - AO asset: \`AO [<id>] → [<asset-slug>]\` and the line/section if relevant
   - Task comment: link to the task + the timestamp of the comment
   - DB record: table name + primary key
2. If the result is a multi-row data change, give the count + a representative example (e.g. "Inserted 13 rows. Example: \`{ id: 42, ... }\`").

## Rules
- Never report done without either Case A or Case B coverage.
- Never assume the user remembers from chat where things are. The pointer must stand alone.
- If both apply (file edit + visible page), do both — open the page in CIC AND state the file path.
`.trim(),

  refDiscipline: `
## Trigger phrases
GM routes through this skill book when the user says any of:
- "JustDoIt"
- "JDI"
- "jedi"
- "just do it"
- "wing it"
- "use what you have"
- "you know what to do"

These signal: the user has decided the task is doable with native capabilities (CIC + Read/Edit/Write/Bash) — no formal SMART plan, no Sage skill selection needed. GM proceeds directly to execution, applying the JDI rules below.

## Default tool routing
- **Browser work** → CIC (\`mcp__Claude_in_Chrome__*\`). Never \`mcp__computer-use__*\` unless the user explicitly named it.
- **Local file work** → Read / Edit / Write / Glob / Grep / Bash.
- **Dedicated MCP available** → use the MCP first (gmail, asana, etc.) per the MCP-first rule in CLAUDE.md.

## CIC tab hygiene
- Always start with \`tabs_context_mcp({ createIfEmpty: true })\` to get the current tab group.
- One tab group per objective.
- Close tabs you opened for one-off checks. Leave open only those needed for user review (case A of reportResult).

## The verify mandate
Every JDI completion ends with \`justDoIt.verify\` (or its inline equivalent for no-render cases). No exceptions. The verify pass is the price of skipping formal SMART planning.

## The report mandate
Every JDI completion ends with \`justDoIt.reportResult\`. The user must be able to reach the result in one click.

## When to expand this book
When you find yourself writing the same JDI sub-procedure twice (e.g. "open URL, screenshot, OCR, compare"), promote it to a new action in this book. The book is intentionally minimal at start; growth is driven by repeated patterns.
`.trim(),
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->
