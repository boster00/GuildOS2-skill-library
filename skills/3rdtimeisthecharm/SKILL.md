# 3rdtimeisthecharm

<!-- GUILDOS-SKILL name="3rdtimeisthecharm" -->

<!-- GUILDOS-DESC -->
3rdtimeisthecharm, aka 3TITC. Iteratively IMPROVE / REFINE / TIGHTEN / POLISH a PROMPT (or any artifact) by getting a SECOND OPINION from a ChatGPT session (driven via Claude-in-Chrome) on the current state plus a user-given improvement DIRECTION, then JUDGING that feedback with your own repo/doctrine context and adopting ONLY the sensible points, up to 3 rounds, until a passing grade (default 9.0/10). Triggers on: improve this prompt, refine / tighten / sharpen wording, make a prompt or copy SHORTER / more CONCISE / more impactful, optimize a prompt, prompt optimization, review and revise, score and improve, get a second opinion on a prompt, run it by ChatGPT, iterate a prompt with ChatGPT, refine-until-good. Core discipline: the outside model is CONTEXT-BLIND, so its feedback is INPUT not orders — you accept only what survives your judgement and reject cosmetic churn; if the state isn't stably visible, build a surface to extract it first; loop stops when the score clears the (arbitrary but guarded) passing bar. Always use ChatGPT's smartest model. REQUIREMENTS GATE: before looping, confirm BOTH the subject (which prompt/artifact to improve) and the direction (what 'better' means); if either is missing or vague, clarify with the user first — never guess the subject or invent a direction.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/3rdtimeisthecharm/SKILL.md -->
```markdown
---
name: 3rdtimeisthecharm
description: >-
  3rdtimeisthecharm, aka 3TITC. Iteratively IMPROVE / REFINE / TIGHTEN / POLISH a PROMPT (or any artifact) by getting
  a SECOND OPINION from a ChatGPT session (driven via Claude-in-Chrome) on the current
  state plus a user-given improvement DIRECTION, then JUDGING that feedback with your own
  repo/doctrine context and adopting ONLY the sensible points, up to 3 rounds, until a
  passing grade (default 9.0/10). Triggers on: improve this prompt, refine / tighten /
  sharpen wording, make a prompt or copy SHORTER / more CONCISE / more impactful, optimize
  a prompt, prompt optimization, review and revise, score and improve, get a second opinion
  on a prompt, run it by ChatGPT, iterate a prompt with ChatGPT, refine-until-good. Core
  discipline: the outside model is CONTEXT-BLIND, so its feedback is INPUT not orders — you
  accept only what survives your judgement and reject cosmetic churn; if the state isn't
  stably visible, build a surface to extract it first; loop stops when the score clears the
  (arbitrary but guarded) passing bar. Always use ChatGPT's smartest model. REQUIREMENTS GATE:
  before looping, confirm BOTH the subject (which prompt/artifact to improve) and the direction
  (what 'better' means); if either is missing or vague, clarify with the user first — never
  guess the subject or invent a direction.
---

# 3rd Time Is The Charm (3TITC) — iterative improve-via-ChatGPT loop

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/3rdtimeisthecharm/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto 3rdtimeisthecharm <action>` (or read the source directly).

## Actions

- **cicDriving** — DRIVE CHATGPT via Claude-in-Chrome to review and improve a prompt: pick the smartest
- **judgeDontObey** — JUDGE the ChatGPT feedback, don't obey it — when improving/refining a prompt via a
- **makeStateVisible** — MAKE THE PROMPT/ARTIFACT STATE VISIBLE before improving it — if the current prompt
- **refGuardrails** — GUARDRAILS for the improve-a-prompt-via-ChatGPT loop: passing grade 9.0 by default
- **requirementsGate** — REQUIREMENTS GATE — before looping, confirm the TWO inputs are clear: the SUBJECT
- **theLoop** — THE LOOP to improve / refine / tighten / polish a prompt (or any artifact): consult
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/3rdtimeisthecharm/index.js -->
```js
/**
 * 3rdtimeisthecharm — CIC-driven iterative improvement loop.
 *
 * Take ANY artifact you're trying to improve (a prompt, a lease wrapper, a piece of
 * copy, a schema, a design) plus a user-given IMPROVEMENT DIRECTION, show its CURRENT
 * STATE to a ChatGPT session (driven via Claude-in-Chrome), get scored feedback, JUDGE
 * that feedback with the repo/doctrine context you have and the model does NOT, adopt
 * only the sensible points, and repeat up to 3 rounds until a passing grade (default 9.0).
 *
 * WHY a second model at all: a fresh outside reviewer catches bloat, ambiguity, and
 * missing guards you've gone blind to — but it is CONTEXT-BLIND (it can't see your repo,
 * your locked EDs, what's load-bearing). So its feedback is INPUT, never a command: you
 * are the judge. This skill is the disciplined wrapper that keeps the outside opinion
 * useful without letting it drive.
 *
 * PURE-TEXT procedure skill (the ChatGPT driving is done by the agent via the
 * Claude-in-Chrome MCP tools; there is no executable action to run here).
 */

export const skillBook = {
  id: "3rdtimeisthecharm",
  title: "3rd Time Is The Charm (3TITC) — iterative improve-via-ChatGPT loop",
  description:
    "3rdtimeisthecharm, aka 3TITC. Iteratively improve a prompt or any artifact by consulting a ChatGPT session (via " +
    "Claude-in-Chrome) with the current state + a user-given improvement direction, then " +
    "JUDGING the feedback with your own context and adopting only the sensible points, up " +
    "to 3 rounds, until a passing grade (default 9.0/10). Reach for it on: improve/refine/" +
    "polish/tighten a prompt, make a prompt or copy shorter / more concise / sharper / more " +
    "impactful, get a second opinion on wording, iterate on an instruction/system prompt, " +
    "score-and-improve, prompt optimization, review-and-revise-until-good, 'run it by " +
    "ChatGPT', A/B a prompt against a reviewer. Core discipline: outside model is " +
    "context-BLIND, so its feedback is input not orders — you accept only what survives " +
    "your judgement; loop stops when score >= the (arbitrary but guarded) passing grade. " +
    "PRE-CHECK the two inputs FIRST — the SUBJECT (what artifact/prompt to improve) and the " +
    "DIRECTION (what 'better' means here); if either is missing or vague, clarify with the user " +
    "BEFORE looping (never guess the subject or invent a direction).",

  toc: {
    requirementsGate:
      "REQUIREMENTS GATE — before looping, confirm the TWO inputs are clear: the SUBJECT " +
      "(which specific prompt/artifact to improve) and the DIRECTION (what 'better' means). " +
      "If either is missing or vague, CLARIFY WITH THE USER first — never guess the subject or " +
      "invent a direction. Use for: which prompt to improve, what does 'better' mean, unclear " +
      "improvement goal, is the ask specific enough to start, before running the improve loop.",
    theLoop:
      "THE LOOP to improve / refine / tighten / polish a prompt (or any artifact): consult " +
      "ChatGPT for a second opinion with the current prompt + your improvement direction " +
      "(shorter, more concise, sharper, more impactful), score it /10, iterate up to 3 " +
      "rounds. Use for: improve this prompt, refine wording, make it more concise, optimize " +
      "a prompt, review and revise, get a second opinion, run it by ChatGPT, score and improve.",
    judgeDontObey:
      "JUDGE the ChatGPT feedback, don't obey it — when improving/refining a prompt via a " +
      "second opinion, the reviewer is context-blind, so accept only sensible load-bearing " +
      "changes and reject cosmetic churn or synonym swaps. Use for: improve/refine/tighten a " +
      "prompt, make it more concise, evaluate ChatGPT's prompt feedback, review and revise, " +
      "which suggestions to accept.",
    makeStateVisible:
      "MAKE THE PROMPT/ARTIFACT STATE VISIBLE before improving it — if the current prompt " +
      "isn't stably extractable, build a surface that dumps it so each refine round scores " +
      "the same real thing, not a paraphrase. Use for: improve a prompt produced by code, " +
      "iterate on a builder's output, refine / tighten / optimize a generated prompt, extract " +
      "the current state to review.",
    cicDriving:
      "DRIVE CHATGPT via Claude-in-Chrome to review and improve a prompt: pick the smartest " +
      "model, inject the prompt + improvement direction (base64+atob to dodge escaping), send, " +
      "read the score via screenshot. Use for: get a second opinion on a prompt, run a prompt " +
      "by ChatGPT, score-and-improve wording, iterate a prompt with ChatGPT, refine via CIC.",
    refGuardrails:
      "GUARDRAILS for the improve-a-prompt-via-ChatGPT loop: passing grade 9.0 by default " +
      "(arbitrary but guarded, overridable), max 3 rounds, model is a context-blind advisor, " +
      "stop early on pass, abandon regressions. Use for: how good is good enough, when to stop " +
      "iterating on a prompt, prompt-improvement passing bar, refine-until-good discipline.",
  },

  requirementsGate: `
Before ANY round, the loop needs TWO clear inputs. Check both; if either is missing or vague,
STOP and ask the user — this is the one place the skill defers to the user.

1. **Subject — WHAT to improve.** The specific artifact: this prompt, this lease wrapper, this
   paragraph, this schema. "Improve my prompts" is NOT a subject — which one, exactly? If the
   user gestured at a class of things, ask them to name the one to work on (or confirm you'll
   take the most recent / the one on screen).
2. **Direction — what BETTER means.** The optimization target: shorter, sharper, safer, more
   concise, more persuasive, fewer tokens, clearer to a junior. Without it the reviewer scores
   in a vacuum and you can't judge its feedback. If the user said "improve it" with no axis,
   ask what axis matters most here.

Why this gate exists: the loop OPTIMIZES toward the direction against the subject. Guess either
and you optimize the wrong thing, and you won't discover it until you've burned all 3 rounds
and a chunk of the user's ChatGPT quota. One clarifying question up front is far cheaper.

This is a genuine, load-bearing clarification — the exception the sage question gate allows
(ESCALATE), NOT a manufactured stop. Do not invent a subject or a direction to avoid asking.
Once both are clear (given, or confirmed), proceed to the loop.
`.trim(),

  theLoop: `
Goal: improve the ARTIFACT toward the user's DIRECTION using an outside reviewer, without
letting the context-blind reviewer drive.

## Requirements gate (BEFORE round 1 — see requirementsGate)
Confirm the loop's TWO inputs are clear FIRST:
- **Subject** — the specific artifact/prompt to improve (not "my prompts" — which one exactly).
- **Direction** — what "better" means here (shorter, sharper, safer, more concise, …).
If EITHER is missing or vague, ASK THE USER to pin it down before you start. Do NOT guess the
subject or invent a direction — the loop would optimize the wrong thing and burn all 3 rounds.
This is the ONE user-stop the skill allows; per the sage question gate it is a load-bearing
clarification (ESCALATE), not a manufactured stop.

## The loop (repeat up to 3 rounds; stop early once score >= passing grade)
0. **Make the state visible + stable.** If you can't already extract the artifact's current
   state as text, build a surface that dumps it (see makeStateVisible). Each round must read
   the SAME real state, not a paraphrase.
1. **Compose the eval message** with THREE parts:
   - the CURRENT STATE verbatim (the exact prompt/artifact);
   - the user's IMPROVEMENT DIRECTION (e.g. "shorter, more concise while impactful");
   - the CONTEXT THE MODEL LACKS — the constraints, locked decisions, and what is
     load-bearing that only you can see (this is what makes its feedback usable). Ask for a
     score /10, ONLY load-bearing changes, and "is it >= <grade>?".
2. **Consult ChatGPT** on its SMARTEST model, via CIC (see cicDriving).
3. **Read** the score and the enumerated points.
4. **JUDGE, don't obey** (see judgeDontObey). Accept only the sensible, load-bearing points;
   reject the rest and say why to yourself.
5. **Apply** the accepted points and **re-verify** the artifact (run its tests / re-render its
   state) so the next round reads the improved real thing.
6. **Stop** when the score >= the passing grade (default 9.0) OR after 3 rounds. If a change
   made the score REGRESS, abandon that change.

## Report
State the score trajectory (e.g. 8.6 -> 9.1), which points you adopted and which you rejected
(with the one-line reason), and where the improved artifact lives. Do NOT claim a pass you
didn't actually get from the reviewer.
`.trim(),

  judgeDontObey: `
The single rule that makes this skill safe: **the outside model's feedback is INPUT, not
orders.** ChatGPT cannot see your repo, your locked engineering_decisions, your registry, or
which line is load-bearing. You can. So you are the final judge every round.

Per suggestion, decide accept/reject WITH YOUR CONTEXT:
- **Accept** only a load-bearing change that moves toward the stated direction and doesn't
  break a constraint the model can't see.
- **Reject** cosmetic churn, synonym/domain-term swaps (keep the ecosystem's own noun),
  points the artifact ALREADY covers, and anything that would regress a locked decision.
- **Watch for context-blind misses**: the model may flag a "risk" that's actually a fixture
  artifact, or push a rename that collides with your doctrine (e.g. hardcoding a skill NAME
  when your rule is appoint-by-capability). These get rejected with a reason.
- **Abandon regressions**: if adopting a point drops the score, revert it.

Never paste the model's rewrite in wholesale. Adopt the DELTA you judged good; re-derive the
final wording yourself so it stays consistent with the surrounding artifact.
`.trim(),

  makeStateVisible: `
You cannot iterate on what you can't stably see. Before round 1, ensure the artifact's
CURRENT STATE is extractable as the same real text every round.

- **If it's already text** (a prompt string, a file, an API response) — extract it verbatim.
- **If it's produced by code** (like a prompt BUILDER, a composed lease, a rendered view) —
  call the real producer and capture its output; don't hand-type or paraphrase it.
- **If it's only visible in a UI** and not dumpable — BUILD a visible surface first: a button
  that prints the state as JSON/text, a debug route, a small pure function. This is a
  one-time investment that makes every round comparable.

Why this matters: feeding the reviewer a paraphrase makes it optimize a thing that doesn't
exist, and two rounds of paraphrase aren't comparable, so "did it improve?" becomes unanswerable.
Extract the SAME real state each round.
`.trim(),

  cicDriving: `
Concrete mechanics for one ChatGPT round via the Claude-in-Chrome MCP (load the browser tools
first; a persistent session tab can be reused across rounds).

1. **Use the smartest model.** Select ChatGPT's top reasoning model before sending (the user
   wants the best reviewer, always).
2. **Inject without escaping pain.** The message often contains backticks, quotes, and unicode
   that break naive string injection. Build the message, base64-encode it (server-side / in a
   node one-liner), then in-page: \`const msg = decodeURIComponent(escape(atob("<b64>")))\`,
   focus \`#prompt-textarea\` (or \`div[contenteditable=true]\`), and
   \`document.execCommand('insertText', false, msg)\`, then dispatch an \`input\` event.
3. **Send.** Click \`button[data-testid="send-button"]\`; if it's momentarily disabled after
   insert, click it again on the next call.
4. **Wait for completion.** Poll: response is done when there's no
   \`button[data-testid="stop-button"]\` AND the assistant-message count went up.
5. **Read via screenshot, not innerText.** ChatGPT renders scores/tables/code in ways that
   \`innerText\`/\`textContent\` truncate or the CIC text-extractor blocks (URLs trip a
   cookie/query-string filter). Scroll the thread and screenshot each section to read the
   score and the enumerated points.
6. **Compose the message** so the reviewer can only help: state + direction + the context it
   lacks + "score /10, only load-bearing changes, is it >= <grade>?".
`.trim(),

  refGuardrails: `
## Guardrails
- **Passing grade = 9.0 by default.** Arbitrary, and openly so — but a guard is better than
  none. Override per task when the user sets a different bar. 9.0 means "an outside reviewer,
  given the real constraints, would ship it."
- **Max 3 rounds.** Diminishing returns past that; if it isn't passing by round 3, the issue
  is usually the artifact's shape (raise that with the user), not more wording passes.
- **The model is a context-blind advisor.** It never has commit authority over your artifact.
- **Stop early on pass; abandon regressions.** The moment a round scores >= the grade, stop.
  If a change lowers the score, revert it.

## When to reach for this
Any "improve / refine / tighten / shorten / sharpen this prompt (or copy, or instruction, or
schema)", "get a second opinion", "run it by ChatGPT", "score and improve", "prompt
optimization", "iterate until good". Name of the skill is the whole idea: by the 3rd pass,
judged (not obeyed), it's usually sharp.

## Relationship to the sage gates
This skill produces an artifact; the sage success gate still applies to any claim that it's
"done / passing" — cite the REAL score the reviewer gave, verified through the actual channel,
never a self-attested pass.
`.trim(),
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->
