# whittler

<!-- GUILDOS-SKILL name="whittler" -->

<!-- GUILDOS-DESC -->
Curb over-engineering AT CODE-WRITE TIME: write the minimum working code, nothing speculative. Triggers on: don't over-engineer, keep it minimal / simple, smallest change / diff, YAGNI, do we even need this, avoid over-abstraction, no premature abstraction, don't add a dependency for this, simplest thing that works, minimal implementation, be lazy, ponytail. Core recipe: climb THE LADDER (need it? → reuse? → stdlib? → native? → installed dep? → one-liner? → minimal code) and STOP at the first rung that holds — while never simplifying past the no-simplify boundaries (validation, error handling, security, accessibility, requested features). Sibling of combobulator (which cleans up code that already exists); whittler stops you building too much in the first place. DRY is NOT whittler's job — that stays manual-trigger in combobulator.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/whittler/SKILL.md -->
```markdown
---
name: whittler
description: >-
  Curb over-engineering AT CODE-WRITE TIME: write the minimum working code, nothing
  speculative. Triggers on: don't over-engineer, keep it minimal / simple, smallest
  change / diff, YAGNI, do we even need this, avoid over-abstraction, no premature
  abstraction, don't add a dependency for this, simplest thing that works, minimal
  implementation, be lazy, ponytail. Core recipe: climb THE LADDER (need it? → reuse?
  → stdlib? → native? → installed dep? → one-liner? → minimal code) and STOP at the
  first rung that holds — while never simplifying past the no-simplify boundaries
  (validation, error handling, security, accessibility, requested features). Sibling
  of combobulator (which cleans up code that already exists); whittler stops you
  building too much in the first place. DRY is NOT whittler's job — that stays
  manual-trigger in combobulator.
---

# Whittler — write-time minimalism gate

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/whittler/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto whittler <action>` (or read the source directly).

## Actions

- **intensity** — Intensity levels — Lite and Full only (ponytail's Ultra is intentionally dropped).
- **noSimplifyBoundaries** — The hard 'never be lazy here' list — where minimalism is FORBIDDEN. Never simplify
- **refDiscipline** — Reference: trigger phrases, the golden rule, and the LOCKED GuildOS adaptations
- **theLadder** — THE LADDER — the write-time decision hierarchy. Before writing new code, climb
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/whittler/engineering_decisions.md -->
```markdown
# Engineering decisions — whittler

## 2026-07-07 · Adopt ponytail as a re-scoped skill, not a verbatim clone
**Decision.** whittler adopts only the non-conflicting core of the viral `ponytail`
skill (DietrichGebert/ponytail): the Ladder, the no-simplify boundaries, and
deletion-over-addition. Three ponytail behaviours are deliberately dropped/re-scoped.
**Rejected alternative.** Cloning ponytail wholesale (its auto-DRY at Full/Ultra, its
"Ultra" one-liner/no-file/no-abstraction extremism, and its `ponytail:` inline shortcut
comments).
**Why.** Each dropped part collides with a LOCKED GuildOS rule: (1) DRY is manual-trigger
only and owned by `combobulator` — auto-DRY is explicitly rejected here; (2) "Ultra"
fewest-files/no-abstraction fights the forge-don't-inline mandate and the structured
skill-book/weapon registry; (3) GuildOS records rationale as dated EDs, not verbose code
comments (readability = descriptive names + EDs). whittler is write-time (don't build too
much); combobulator is refactor-time (thin what exists) — they must not overlap on DRY.
**Consequence if violated.** A future agent "completing" whittler by adding Ultra or
auto-DRY would re-introduce over-eager extraction and inline-scripting that sage.preflight
and the manual-DRY lock exist to prevent — the exact drift this split was built to stop.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/whittler/index.js -->
```js
/**
 * Whittler skill book — write-time minimalism gate (anti-over-engineering).
 *
 * Adopted 2026-07-07 from the viral `ponytail` skill (DietrichGebert/ponytail):
 * the "laziest senior dev" discipline — "the best code is the code never written".
 * Whittler is the GENERATION-time sibling of `combobulator`: combobulator cleans up
 * code that already exists (refactor-time); whittler stops you from building too much
 * in the first place (write-time). Reach for it when the ask is "don't over-engineer",
 * "keep it minimal", "smallest change", "do we even need this", "YAGNI".
 *
 * PURE TEXT registry (discipline, no executable actions).
 *
 * ONLY the non-conflicting core of ponytail is adopted. Three ponytail behaviours are
 * DELIBERATELY dropped/re-scoped because they collide with LOCKED GuildOS rules — this
 * is the whole reason whittler is a distinct skill and not a verbatim clone (see
 * refDiscipline for the rationale, recorded as this skill's engineering decision):
 *   1. ponytail's auto-DRY (Full/Ultra) is DROPPED — DRY stays MANUAL-TRIGGER only,
 *      owned by combobulator. Whittler never de-duplicates as a side effect.
 *   2. ponytail's "Ultra" intensity is DROPPED — its "fewest files, one-liners, no
 *      abstractions" extremism fights GuildOS's forge-don't-inline mandate.
 *   3. ponytail's `ponytail:` shortcut comments are RE-MAPPED to dated EDs in the
 *      module's engineering_decisions.md (GuildOS records rationale as EDs, not code
 *      comments — see the global ED rule + the readability=names+EDs feedback).
 *
 * HARD PRECEDENCE: whittler never overrides sage.preflight. "Reuse before build"
 * (Ladder rung 2) means reuse an existing SKILL BOOK / WEAPON first; a reusable
 * capability still gets FORGED (blacksmith) even though that is more than the minimal
 * diff. Minimalism applies to within-task business logic, not to the registry itself.
 */

export const skillBook = {
  id: "whittler",
  title: "Whittler — write-time minimalism gate",
  description:
    "Curb over-engineering AT CODE-WRITE TIME: write the minimum working code, nothing " +
    "speculative. Triggers on: don't over-engineer, keep it minimal / simple, smallest " +
    "change / diff, YAGNI, do we even need this, avoid over-abstraction, no premature " +
    "abstraction, don't add a dependency for this, simplest thing that works, minimal " +
    "implementation, be lazy, ponytail. Core recipe: climb THE LADDER (need it? → reuse? " +
    "→ stdlib? → native? → installed dep? → one-liner? → minimal code) and STOP at the " +
    "first rung that holds — while never simplifying past the no-simplify boundaries " +
    "(validation, error handling, security, accessibility, requested features). Sibling " +
    "of combobulator (which cleans up code that already exists); whittler stops you " +
    "building too much in the first place. DRY is NOT whittler's job — that stays " +
    "manual-trigger in combobulator.",

  toc: {
    theLadder:
      "THE LADDER — the write-time decision hierarchy. Before writing new code, climb " +
      "seven rungs in order and STOP at the first that holds: (1) does it need to exist " +
      "(YAGNI), (2) already in the codebase / a skill book or weapon, (3) stdlib, (4) " +
      "native platform feature, (5) an already-installed dependency, (6) a one-liner, " +
      "(7) otherwise the smallest working code. Use for: don't over-engineer, keep it " +
      "minimal, smallest change, YAGNI, avoid over-abstraction, do we need this.",
    noSimplifyBoundaries:
      "The hard 'never be lazy here' list — where minimalism is FORBIDDEN. Never simplify " +
      "away: input validation at trust boundaries, error handling that prevents data loss, " +
      "security, accessibility, or anything the user explicitly requested. Understand the " +
      "problem fully BEFORE shrinking the diff — a small diff that skips understanding is " +
      "the dangerous kind. Use for: 'is it safe to cut this', reviewing a minimal change.",
    intensity:
      "Intensity levels — Lite and Full only (ponytail's Ultra is intentionally dropped). " +
      "Lite: build as requested, name the lazier alternative in one line. Full (default): " +
      "enforce the ladder, prefer stdlib + native, minimal explanation. No Ultra — its " +
      "one-liner/no-abstraction extremism fights GuildOS's forge-don't-inline rule.",
    refDiscipline:
      "Reference: trigger phrases, the golden rule, and the LOCKED GuildOS adaptations " +
      "that make whittler NOT a verbatim ponytail clone (defer DRY to combobulator, no " +
      "Ultra, shortcuts recorded as EDs not code comments, never override sage.preflight / " +
      "forge-don't-inline). Read first.",
  },

  theLadder: `
Goal: write the least code that fully solves the ask. Before adding anything new,
climb the ladder and STOP at the first rung that holds.

## The Ladder (evaluate top-down, stop at the first that holds)
1. **Does it need to exist?** Skip speculative needs. The feature nobody asked for is
   the cheapest one to maintain — it doesn't exist. (YAGNI.)
2. **Already available?** Reuse an existing helper/util/type in the codebase — AND,
   the GuildOS extension: an existing **skill book or weapon** (this is sage.preflight's
   job; whittler just reinforces "scan before you build").
3. **Stdlib covers it?** Prefer the standard library over hand-rolled code.
4. **Native platform feature?** Use a native/platform capability over pulling in a lib.
5. **Already-installed dependency?** Use a dep that's already in the tree; do NOT add a
   new dependency for trivial code.
6. **One-line solution?** If it collapses to one clear line, write that line.
7. **Otherwise:** the minimum working code — and no more.

## Core rules
- **No unrequested abstractions.** No single-implementation interface, no factory for
  one product, no config option for a value that never changes. Add the abstraction the
  day the second caller actually arrives, not in anticipation of it.
- **Deletion over addition.** The best diff often removes lines. Boring beats clever.
- **Shortest working diff wins — but only AFTER you understand the problem.** A tiny
  diff that misreads the requirement is worse than a slightly larger correct one.
- **Fix the root cause, not every caller.** Guard a shared function once; don't patch
  each call site (that's symptom-whacking and it drifts).

## GuildOS precedence (do NOT let laziness break these)
- **Reusable capability → still forge it.** If the logic will be needed again / has a
  verb-noun shape, it becomes a weapon or skill book (blacksmith) EVEN THOUGH that is
  more than the minimal inline diff. Minimalism governs within-task business logic, not
  whether to register a capability. Never inline-script around the registry to "save a
  file" — that's the failure sage.preflight exists to stop.
- **Mark deliberate shortcuts as an ED, not a code comment.** When you consciously ship
  a simpler ceiling than the ideal, record it as a dated entry in the module's
  engineering_decisions.md (decision · rejected alternative · why · upgrade trigger) —
  GuildOS records rationale as EDs, not \`// ponytail:\`-style inline notes.
`.trim(),

  noSimplifyBoundaries: `
Minimalism is a scalpel with a NO-CUT list. Being lazy in these places isn't lean —
it's a defect. Never simplify away, no matter the intensity:

- **Input validation at trust boundaries.** Anything crossing an API edge, a form, a
  file/DB read, an external payload — validate it. "It'll usually be well-formed" is how
  data corruption ships.
- **Error handling that prevents data loss.** Do not swallow or skip the handling that
  keeps a partial write, a failed transaction, or a lost message from destroying state.
- **Security.** Authz/authn checks, secret handling, injection guards, escaping — full
  strength, always. (Mirrors the vendor-ingest / secret-redaction locks.)
- **Accessibility.** Semantic markup, labels, keyboard paths — not optional polish.
- **Explicitly requested features.** If the user asked for it, it is in scope. Whittler
  trims what nobody asked for; it never trims what somebody did.

## Understand before you shrink
Skipping understanding to ship a small diff is the dangerous kind of lazy. Read the
surrounding code, know why it's shaped that way, THEN find the minimal correct change.
The order is understand → minimize, never minimize → hope.
`.trim(),

  intensity: `
Two intensities. (ponytail's third level, "Ultra", is intentionally NOT adopted — its
"fewest files / one-liners / challenge every requirement" extremism collides with
GuildOS's forge-don't-inline mandate and its structured skill-book/weapon layout.)

- **Lite** — Build essentially as requested; in ONE line, name the lazier alternative
  you see so the user can opt into it. Use when the user wants the full thing but would
  like to know where it could be trimmed.
- **Full (default)** — Enforce the ladder. Prefer stdlib + native. Keep the explanation
  shorter than the code. This is the everyday write-time posture.

Whittler is ON by default for coding tasks and quietly biases toward the smaller correct
solution. It is advisory, not a hard gate — it never blocks a turn (that's sage's job).
`.trim(),

  refDiscipline: `
## Trigger phrases (route here)
"don't over-engineer", "keep it minimal / simple", "smallest change / diff", "YAGNI",
"do we even need this", "avoid over-abstraction", "no premature abstraction", "don't add
a dependency for that", "simplest thing that works", "minimal implementation", "be
lazy", "ponytail".

## The golden rule
The best code is the code never written. Every line you add is a line someone maintains
forever. Default to LESS — then stop at the no-simplify boundaries.

## LOCKED GuildOS adaptations (why whittler ≠ verbatim ponytail) — adopted 2026-07-07
Whittler adopts ponytail's Ladder + no-simplify boundaries + deletion-over-addition, and
drops/re-scopes the three parts that collide with locked GuildOS rules:
1. **DRY stays with combobulator, manual-trigger only.** ponytail auto-DRYs at Full/Ultra;
   GuildOS does NOT auto-DRY. Whittler is about not BUILDING excess; de-duplicating
   existing code is combobulator's job and only on an explicit "dry it up".
2. **No "Ultra".** Its one-liner/no-file/no-abstraction extremism fights forge-don't-inline
   and the structured registry. Lite + Full only.
3. **Shortcuts → EDs, not code comments.** A deliberate simplification is recorded as a
   dated engineering_decisions.md entry (with its upgrade trigger), not a \`ponytail:\`
   inline note — because GuildOS's readability rule is descriptive-names + EDs, not
   verbose comments.
4. **Never overrides sage.preflight / forge-don't-inline.** "Reuse before build" means
   reuse a skill book/weapon first; a reusable capability is still forged even when that
   exceeds the minimal diff.

## Relationship to combobulator
- **whittler** = write-time: don't build too much (this skill).
- **combobulator** = refactor-time: clean up / thin what's already built (its skill).
They are complementary and never overlap on DRY.
`.trim(),
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->
