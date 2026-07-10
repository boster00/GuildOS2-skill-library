# combobulator

<!-- GUILDOS-SKILL name="combobulator" -->

<!-- GUILDOS-DESC -->
Refactor cumbersome / sprawling / repetitive code for readability and modularity. Triggers on: make dry, dry it up, DRY, dry refactor, de-duplicate, remove duplication, improve readability, improve modularity, make it modular, clean up / tidy this function, extract helpers, thin orchestrator, one line per action, simplify a big function, reduce complexity, this function does too much / is too long. Core recipe: reduce a big function to a one-line-per-step ORCHESTRATOR that delegates to named HELPER functions, keeping behavior identical. DRY is MANUAL-TRIGGER only (never auto-DRY).
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/combobulator/SKILL.md -->
```markdown
---
name: combobulator
description: >-
  Refactor cumbersome / sprawling / repetitive code for readability and
  modularity. Triggers on: make dry, dry it up, DRY, dry refactor, de-duplicate,
  remove duplication, improve readability, improve modularity, make it modular,
  clean up / tidy this function, extract helpers, thin orchestrator, one line per
  action, simplify a big function, reduce complexity, this function does too much /
  is too long. Core recipe: reduce a big function to a one-line-per-step
  ORCHESTRATOR that delegates to named HELPER functions, keeping behavior
  identical. DRY is MANUAL-TRIGGER only (never auto-DRY).
---

# Combobulator — readability & modularity refactors

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/combobulator/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto combobulator <action>` (or read the source directly).

## Actions

- **dryItUp** — DRY it up — de-duplicate repeated logic into a shared helper or lookup table. When to DRY
- **orchestrate** — The orchestrator + helper-functions pattern — the core recipe. Reduce a cumbersome,
- **readability** — Readability & modularity checklist applied after (or instead of) restructuring: descriptive
- **refDiscipline** — Reference: trigger phrases, the golden behavior-identical rule, and the parity-verification
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/combobulator/index.js -->
```js
/**
 * Combobulator skill book — code readability & modularity refactors.
 *
 * The opposite of discombobulate: take a cumbersome, sprawling, or repetitive
 * function and sort it out into a clean orchestrator + named helpers. This is the
 * skill to reach for when the ask is "make this DRY", "dry it up", "improve
 * readability", "make it more modular", "clean up this function", "extract helpers",
 * or "this function is too long / does too much".
 *
 * PURE TEXT registry (discipline + recipe, no executable actions). The worked
 * reference example is the applyMutations refactor in GuildOS2 libs/quest/write.js
 * (2026-07-06): a ~145-line function with an inline op-loop reduced to a 7-line
 * orchestrator over guardBatch/assertOwner/readStage/applyOp/stampActivity/
 * reconcileStage helpers, behavior verified identical.
 *
 * GOLDEN RULE: a readability refactor MUST NOT change behavior. Re-run the same
 * inputs before and after and confirm identical outputs (the parity check is the
 * proof, not "it looks right").
 */

export const skillBook = {
  id: "combobulator",
  title: "Combobulator — readability & modularity refactors",
  description:
    "Refactor cumbersome / sprawling / repetitive code for readability and modularity. " +
    "Triggers on: make dry, dry it up, DRY, dry refactor, de-duplicate, remove duplication, " +
    "improve readability, improve modularity, make it modular, clean up / tidy this function, " +
    "extract helpers, thin orchestrator, one line per action, simplify a big function, reduce " +
    "complexity, this function does too much / is too long. Core recipe: reduce a big function " +
    "to a one-line-per-step ORCHESTRATOR that delegates to named HELPER functions, keeping " +
    "behavior identical. DRY is MANUAL-TRIGGER only (never auto-DRY).",

  toc: {
    orchestrate:
      "The orchestrator + helper-functions pattern — the core recipe. Reduce a cumbersome, " +
      "sprawling function to a thin orchestrator that reads as ONE LINE PER STEP, each line " +
      "delegating to a named helper that holds the actual logic. How to find the steps, name " +
      "the helpers, thread per-iteration state via a ctx object, and keep behavior identical. " +
      "Use for: improve readability, make modular, 'this function is too long / does too much', " +
      "extract helpers, thin the orchestrator, one line per action.",
    dryItUp:
      "DRY it up — de-duplicate repeated logic into a shared helper or lookup table. When to DRY " +
      "(MANUAL-TRIGGER ONLY — only on an explicit 'dry it up' / 'make dry' / 'de-duplicate' ask; " +
      "never auto-DRY as a side effect) and when NOT to (incidental duplication that will diverge). " +
      "Use for: make dry, dry refactor, remove duplication, factor out repeated code.",
    readability:
      "Readability & modularity checklist applied after (or instead of) restructuring: descriptive " +
      "names over comments, right altitude, small single-purpose helpers, thread state via a ctx " +
      "object not globals, match surrounding style, and record only NON-NATIVE rationale as an ED. " +
      "Use for: improve readability, make code clearer, clean up, tidy.",
    refDiscipline:
      "Reference: trigger phrases, the golden behavior-identical rule, and the parity-verification " +
      "step every combobulator refactor must pass before it is claimed done. Read first.",
  },

  orchestrate: `
Goal: a reader (or future agent) understands the function's SHAPE in one glance —
the orchestrator lists the steps; each helper holds one step's logic.

## The pattern
1. **List the phases.** Read the cumbersome function and name its distinct phases in
   order (guard inputs → check preconditions → snapshot state → do the work loop →
   post-process → return). Each phase becomes one line in the orchestrator.
2. **Extract one helper per phase.** Move each phase's body into a named function.
   Name it for what it DOES (\`guardBatch\`, \`assertOwner\`, \`applyOp\`, \`stampActivity\`,
   \`reconcileStage\`) — a reader should predict the body from the name.
3. **Make the orchestrator one line per action.** The top function becomes a linear
   list of \`const x = await helper(...)\` / \`for (...) results.push(await stepFn(...))\`.
   No inline logic beyond wiring and short-circuit returns.
4. **Thread per-iteration state via a \`ctx\` object**, not module globals or long
   parameter lists. A helper that must advance shared state (e.g. \`ctx.priorStage\`)
   mutates the ctx it was handed, so the sequence stays legible.
5. **Short-circuit with guard helpers that return an error RESPONSE or null.** The
   orchestrator reads \`const rejected = guardBatch(...); if (rejected) return rejected;\`
   — the guard owns the "what's wrong" message; the orchestrator owns only the flow.
6. **Loop bodies get their own dispatcher.** A per-item \`for\` loop calls ONE helper
   (\`applyOp\`) that dispatches by type to sub-helpers (\`applyInsert\`/\`applyDelete\`/…),
   each returning a result row and never throwing.

## Worked reference (canonical)
GuildOS2 \`libs/quest/write.js\` \`applyMutations\` (2026-07-06): a ~145-line function
with a giant inline op-loop became:
\`\`\`js
export async function applyMutations(questId, ops, { client, ownerId } = {}) {
  const rejected = guardBatch(questId, ops, client);           if (rejected) return rejected;
  const denied = await assertOwner(client, questId, ownerId);  if (denied) return denied;
  const ctx = { client, questId, priorStage: await readStage(client, questId) };
  const results = [];
  for (const op of ops) results.push(await applyOp(ctx, op));   // one op → one result row
  await stampActivity(client, questId, results);
  const reconciledStage = await reconcileStage(client, questId, results);
  return { ok: results.every((r) => r.ok), results, ...(reconciledStage ? { reconciledStage } : {}) };
}
\`\`\`
Every phase (guard, own, snapshot, apply-each, stamp, self-heal) is one line; the
logic lives in the helpers below it. \`ctx.priorStage\` carries the state the explicit
stage-transition log advances. The exported signature + return shape are UNCHANGED.

## Guardrails
- **Behavior identical.** Do not "fix" logic while restructuring — a combobulator pass
  is a pure move. If you spot a real bug, call it out separately; don't fold it in.
- **Keep the public surface.** Same exported name, params, and return shape. Helpers
  are module-private (not exported) unless already shared.
- **Verify parity** (see refDiscipline) before claiming done.
`.trim(),

  dryItUp: `
DRY = Don't Repeat Yourself: collapse genuinely-duplicated logic into one shared
helper or a lookup table. But DRY is a scalpel, not a reflex.

## MANUAL-TRIGGER ONLY
Only DRY when the user EXPLICITLY asks ("dry it up", "make dry", "de-duplicate",
"factor this out"). Do NOT auto-DRY as a side effect of another change — over-eager
extraction couples code that was only incidentally similar and will later diverge.

## When DRY is right
- The SAME logic appears 3+ times and is CONCEPTUALLY one thing (it should change in
  lockstep). Extract to a helper; give it a name that states the concept.
- A repeated \`switch\`/if-ladder over a fixed set → a lookup TABLE keyed by the
  discriminant (mirrors the \`TABLES\` map + \`pickFields\` in the write.js reference).

## When to leave it
- The blocks LOOK alike but answer to different reasons — they'll drift. Duplication is
  cheaper than the wrong abstraction.
- Extracting would need a pile of flags/params to cover the variants. That's a sign the
  cases are different; keep them separate.

## How
1. Name the shared concept precisely.
2. Extract the common core; pass the varying bits as params (few, well-named).
3. Replace every call site; confirm each still reads clearly locally.
4. Verify parity (refDiscipline).
`.trim(),

  readability: `
Applied after restructuring (or on its own for a lighter cleanup). Readability comes
from NAMES + SHAPE, not from more comments.

## Checklist
- **Descriptive names over comments.** A helper/variable named for its job removes the
  comment that would have explained it. (Global rule: readability = descriptive names +
  ED rationale, NOT verbose comments.)
- **Right altitude.** The orchestrator talks in phases; helpers talk in operations; the
  DB/IO calls sit at the bottom. Don't mix altitudes in one function.
- **Small, single-purpose helpers.** One reason to exist each. If a helper needs "and"
  to describe it, split it.
- **Thread state via a \`ctx\` object**, not module globals or 6-arg signatures.
- **Match the surrounding code.** Mirror the file's existing idiom, comment density, and
  naming — a refactor should look native, not imported.
- **Comment only the non-obvious.** Keep the load-bearing "why" (the surprising default,
  the ordering constraint); delete narration of what the code plainly says.
- **Record NON-NATIVE rationale as an ED**, not inline prose — a decision a future agent
  would otherwise revert goes in the module's engineering_decisions.md.

## Anti-patterns
- Renaming for its own sake in a way that fights the file's conventions.
- Extracting a "helper" used exactly once that only adds a jump with no clarity gain
  (inline it unless it names a meaningful phase in an orchestrator).
- Leaving stale comments that described the OLD shape.
`.trim(),

  refDiscipline: `
## Trigger phrases (route here)
"make dry", "dry it up", "DRY", "dry refactor", "de-duplicate", "remove duplication",
"improve readability", "make it more readable / clearer", "improve modularity", "make
it modular", "clean up / tidy this function", "extract helpers", "thin orchestrator",
"one line per action", "this function is too long / cumbersome / does too much",
"simplify this function", "reduce complexity".

## The golden rule — behavior MUST stay identical
A combobulator refactor is a PURE MOVE. Same inputs → same outputs, same side effects,
same public signature + return shape. Do not smuggle in logic fixes; if you find a real
bug, surface it separately.

## Parity verification (required before claiming done)
1. Keep the exported name, params, and return shape unchanged.
2. Syntax-check the file.
3. Re-run the SAME real inputs you can exercise (a live endpoint hit, a unit call, a
   test) against the refactored code and confirm the outputs match the pre-refactor
   behavior — the parity result is the proof, not "it reads better".
4. Only then report done, pointing at the file + the parity evidence.
`.trim(),
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->
