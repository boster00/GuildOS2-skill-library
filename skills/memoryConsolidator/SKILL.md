# memoryConsolidator

<!-- GUILDOS-SKILL name="memoryConsolidator" -->

<!-- GUILDOS-DESC -->
Pointer to the built-in `anthropic-skills:consolidate-memory` harness skill + GuildOS deltas (memory-dir path, feedback_* two-token clustering, ≤50-line MEMORY.md cap, askSage-ESCALATE-on-delete). Run when MEMORY.md exceeds ~40 entries or filename clusters appear. Do NOT reimplement the generic reflect/merge/prune procedure — invoke the harness skill and layer the GuildOS conventions.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/memoryConsolidator/SKILL.md -->
```markdown
---
name: memoryConsolidator
description: >-
  Pointer to the built-in `anthropic-skills:consolidate-memory` harness skill +
  GuildOS deltas (memory-dir path, feedback_* two-token clustering, ≤50-line
  MEMORY.md cap, askSage-ESCALATE-on-delete). Run when MEMORY.md exceeds ~40
  entries or filename clusters appear. Do NOT reimplement the generic
  reflect/merge/prune procedure — invoke the harness skill and layer the GuildOS
  conventions.
---

# Memory Consolidator

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/memoryConsolidator/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto memoryConsolidator <action>` (or read the source directly).

## Actions

- **run** — Invoke the built-in `anthropic-skills:consolidate-memory` harness skill, then apply the four GuildOS deltas: this workspace's memory-dir path, the feedback_* two-token clustering convention, the ≤50-line MEMORY.md cap, and the askSage-ESCALATE-on-delete safety tie-in.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/memoryConsolidator/index.js -->
```js
/**
 * memoryConsolidator skill book — thin pointer to the built-in
 * `anthropic-skills:consolidate-memory` harness skill, plus the GuildOS-specific
 * deltas that skill does not know about.
 *
 * The generic "reflect over memory files, merge duplicates, fix stale facts,
 * prune the index" procedure is now BUILT IN to the harness. Do NOT reimplement
 * it here — invoke the harness skill and layer the GuildOS conventions below.
 */

export const skillBook = {
  id: "memoryConsolidator",
  title: "Memory Consolidator",
  description:
    "Pointer to the built-in `anthropic-skills:consolidate-memory` harness skill + GuildOS deltas (memory-dir path, feedback_* two-token clustering, ≤50-line MEMORY.md cap, askSage-ESCALATE-on-delete). Run when MEMORY.md exceeds ~40 entries or filename clusters appear.",
  toc: {
    run: "Invoke the built-in `anthropic-skills:consolidate-memory` harness skill, then apply the four GuildOS deltas: this workspace's memory-dir path, the feedback_* two-token clustering convention, the ≤50-line MEMORY.md cap, and the askSage-ESCALATE-on-delete safety tie-in.",
  },
  run: `
The generic memory-consolidation procedure is BUILT IN. Do NOT reimplement it.

## Step 1 — Invoke the harness skill

Run the **\`anthropic-skills:consolidate-memory\`** skill (Skill tool). It does the
reflective pass: read memory files, merge duplicates, fix stale facts, prune the
index. That is the engine; everything below is GuildOS-specific overlay.

## Step 2 — Apply GuildOS deltas

These are the only things the built-in skill does not know about this workspace:

1. **Memory-dir path.** GuildOS auto-memory lives at
   \`C:/Users/xsj70/.claude/projects/C--Users-xsj70-GuildOS/memory/\`
   with the index at \`MEMORY.md\` in that dir. Point the harness skill there.

2. **Filename-token clustering convention.** Group files by the first two
   underscore-separated tokens after the type prefix
   (e.g. \`feedback_browser_control.md\` + \`feedback_browser_control_local.md\`
   cluster on \`browser_control\`). For each cluster ≥ 2 files: a \`(LOCKED ...)\`
   marker is canonical; otherwise the most recently updated wins. Contradictory
   entries (one says X, another NOT X) are a SUPERSEDED chain — keep the newest
   and add a "supersedes [[old-name]]" note rather than blind-merging.

3. **≤50-line MEMORY.md cap.** After rewrite, the index must be ≤ 50 lines, one
   line per surviving file, sorted by load-bearing priority (LOCKED rules first,
   then the operating strategy, then references). A memory that just restates a
   file path, function name, or git fact is delete-eligible — the source is
   authoritative.

4. **askSage-ESCALATE-on-delete safety tie-in.** Memory loss is irreversible.
   Do NOT delete memory files automatically — PROPOSE merges/deletions and route
   any not-yet-user-approved merge/delete through sage.askSage → ESCALATE. The
   built-in skill's reflective pass produces the proposal; the GuildOS gate
   decides whether it ships.
`.trim(),
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->
