# troubleshooting

<!-- GUILDOS-SKILL name="troubleshooting" -->

<!-- GUILDOS-DESC -->
Sage-owned claim-verification + resolution substrate. ANY claimed block/error/issue OR claimed done/fixed/verified is FIRST independently reality-checked ('is it true?') by performing the real diagnostic in the user's own channel — never the agent's self-report, never a proxy, never a self-check delegated back to the reporting agent — then resolved or escalated on the verified reality.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/troubleshooting/SKILL.md -->
```markdown
---
name: troubleshooting
description: >-
  Sage-owned claim-verification + resolution substrate. ANY claimed
  block/error/issue OR claimed done/fixed/verified is FIRST independently
  reality-checked ('is it true?') by performing the real diagnostic in the
  user's own channel — never the agent's self-report, never a proxy, never a
  self-check delegated back to the reporting agent — then resolved or escalated
  on the verified reality.
---

# Troubleshooting

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/troubleshooting/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto troubleshooting <action>` (or read the source directly).

## Actions

- **refDiscipline** — Why this exists + the canonical anti-patterns (node --check 'passed' while the build is red; curl-inferred 'auth blocker' that was a build error; sandbox/preview screenshot reported as 'verified'; described/simulated step reported as 'done'; asserted blocker never actually attempted).
- **resolve** — Step 2, only after verifyClaim: drive the verified reality to closure — root-cause-CLASS fix, re-verify via the real channel, revert if it did not work, escalate ONLY a verified genuinely-user-owned irreversible item, with the verified evidence (not the claim).
- **verifyClaim** — Step 1, load-bearing: given a claimed block/error OR a claimed success/fix, independently establish IS-IT-TRUE by doing the real diagnostic in the user's channel. Returns the verified verdict (TRUE / FALSE / MISCAUSED) + the real observed state + the evidence. Never delegates the check back to the reporting agent.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/troubleshooting/index.js -->
```js
/**
 * Troubleshooting skill book — the formalized claim-verification + resolution
 * substrate. Owned-in-use by Sage (sage.askSage routes here).
 *
 * Premise (locked 2026-05-15, after a session where ~half the errors were
 * mis-diagnosed blocks and the other half bad questions): an agent's
 * self-report of state — "I'm blocked", "X failed", "it's an auth issue",
 * "can't reach Y", AND equally "done", "fixed", "verified", "passes" — is
 * NOT evidence. The agent has shown, repeatedly, that it will not reliably
 * self-check, because it is not tightened to. So the fix is structural: do
 * not ask the reporting agent to self-check. A separate authority (Sage,
 * via this substrate) independently establishes whether the claim is TRUE
 * by performing the real diagnostic in the user's own observation channel,
 * and only then gives feedback or escalates.
 *
 * Governing maxim (user, verbatim, locked 2026-05-15): "answer is it true
 * before answer why and how" — that is what a wise person does. IS-IT-TRUE
 * is resolved first, by the verifier, in the user's own channel. Only then
 * does why/how (root cause, fix, escalation) begin.
 *
 * The two error classes this substrate exists to kill (user, verbatim):
 * "bad success claims" and "bad questions / blocks". Both are claims from
 * an agent that does not self-check; both are verified here the same way.
 *
 * Pure text registry — no callable JS. Inline tokens use single quotes (not
 * backticks) on purpose: this file is itself a substrate that must never
 * reintroduce the unescaped-backtick build break it warns about.
 */

export const skillBook = {
  id: "troubleshooting",
  title: "Troubleshooting",
  description:
    "Sage-owned claim-verification + resolution substrate. ANY claimed block/error/issue OR claimed done/fixed/verified is FIRST independently reality-checked ('is it true?') by performing the real diagnostic in the user's own channel — never the agent's self-report, never a proxy, never a self-check delegated back to the reporting agent — then resolved or escalated on the verified reality.",

  toc: {
    verifyClaim:
      "Step 1, load-bearing: given a claimed block/error OR a claimed success/fix, independently establish IS-IT-TRUE by doing the real diagnostic in the user's channel. Returns the verified verdict (TRUE / FALSE / MISCAUSED) + the real observed state + the evidence. Never delegates the check back to the reporting agent.",
    resolve:
      "Step 2, only after verifyClaim: drive the verified reality to closure — root-cause-CLASS fix, re-verify via the real channel, revert if it did not work, escalate ONLY a verified genuinely-user-owned irreversible item, with the verified evidence (not the claim).",
    refDiscipline:
      "Why this exists + the canonical anti-patterns (node --check 'passed' while the build is red; curl-inferred 'auth blocker' that was a build error; sandbox/preview screenshot reported as 'verified'; described/simulated step reported as 'done'; asserted blocker never actually attempted).",
  },

  verifyClaim: `
You are establishing whether a CLAIM is true, before anyone acts on it or the user is involved. The claim came from an agent (possibly you, a prior turn). Treat it as an unverified assertion, not a fact.

## What counts as a claim (both directions)
- A BLOCK/ERROR claim: "I'm blocked", "X failed", "can't do Y", "need the user to unblock", "it's an auth issue", "the server/API is down", "CIC isn't working", "the migration can't apply".
- A SUCCESS claim: "done", "fixed", "verified", "it works", "passes", "converged", "the screenshot looks right".
Both are claims. The reporting agent's narrative, a passing proxy (node --check, a 200, a DB SELECT, a sandbox/preview render, a test), or "the code is correct" are NOT evidence — they are exactly what has been wrong before.

## The one rule
NEVER do any of these:
- accept the claim as stated,
- surface the claim to the user,
- act on the claim (fix it, escalate it, mark it done),
- or tell the reporting agent to "self-check" / "verify and report back"
...until IS-IT-TRUE has been independently resolved HERE, by doing the real diagnostic yourself. The reporting agent has demonstrated it will not self-check reliably; delegating the check back is the failure this substrate removes.

## How to establish IS-IT-TRUE (the real diagnostic)
Observe the actual state through the USER'S OWN channel, the way the user would — per the alignment theorem (supervisedEvolution.run 1c/1d): proxies systematically deviate; only the rendered/observed reality is ground truth.
- UI / page / "it renders" / "404" / "blank" / "auth gate": open it in CIC (the user's real browser, via browsercontrol.validateCIC) and LOOK. Not curl, not Claude_Preview, not the agent's screenshot description.
- "build / code is fixed": look at the RUNNING build (the Turbopack/dev overlay in CIC), not node --check or a passing test. node --check passing while the real build is red is the canonical lie.
- "DB write done": the user-channel is the rendered record in the app, not a SELECT-back.
- "command/tool failed" / "can't reach X": actually run the real command/path once and read the real output; do not infer from a redirect or a stale prior attempt.
- "deploy/SSH/API works": exercise the real path end-to-end and read its real success signal.

## Verdict (return exactly one)
- TRUE — the claim is real as stated (the diagnostic reproduced it).
- FALSE — the claim is wrong (the thing actually works / is not blocked).
- MISCAUSED — a real symptom exists but the stated cause is wrong (e.g. "auth gate" that is actually a build error; "fixed" while the build is still red). This is the most common verdict for agent block-claims and premature "fixed" claims.

Return: { claim, claimType: "block" | "success", verdict: "TRUE" | "FALSE" | "MISCAUSED", realState: "<what the real diagnostic actually showed>", evidence: "<the concrete observation: screenshot id, real command output, the rendered text>" }. Then hand to troubleshooting.resolve. Default suspicion is high: a claim is not TRUE until the real diagnostic shows it.
`.trim(),

  resolve: `
Run ONLY after verifyClaim produced a verdict + real evidence. Resolution is driven by the verified reality, never the original claim.

## By verdict
- FALSE → the block/error is not real (or the "done" is not done). Do NOT escalate, do NOT ask the user, do NOT tell the agent to self-check. Feed back the verified real state + evidence and let the work continue from reality. (A false block that reached the user is the wrench; a false "done" that reached the user is the success-gate failure.)
- MISCAUSED → restate the REAL cause from the evidence, discard the claimed cause, and route to the real fix. Example: claim "auth gate, user must sign in" → real cause "Turbopack build error in file:line" → the fix is the build error, the user is NOT asked to sign in.
- TRUE → it is a real issue. Resolve it:
  1. Root-cause CLASS, not the instance. If the diagnostic shows an unescaped backtick at line N, every other unescaped backtick in that file is the same root cause — fix them ALL in one pass; do not whack-a-mole one line, re-check, hit the next, repeat.
  2. Apply the fix.
  3. RE-VERIFY by re-running verifyClaim's real diagnostic in the user's channel (the running build/page/record) — NOT a proxy, NOT node --check, NOT "it should work now".
  4. If the re-verify still fails: the fix did not work — REVERT it (do not retain an unvalidated/failed fix) and try a different method. Repeat to the retry budget.
  5. Escalate to the user ONLY a verified-TRUE, genuinely user-owned irreversible dependency (real magic-link auth the user must do, money/prod-destruction, a true architecture fork), and surface it with the VERIFIED evidence — never the agent's unverified claim, never a MISCAUSED or FALSE claim.

## Invariants
- Verification authority never returns to the reporting agent. "Go check and tell me" is banned; the checker is Sage/this substrate.
- The real channel (looking) is the only acceptable re-verify. Proxies do not earn a fix.
- One root cause = one class fix = one real re-verify. Escalation requires verified, specific, irreversible evidence.
`.trim(),

  refDiscipline: `
Why this substrate exists, and the anti-patterns it kills.

Governing maxim (user, verbatim): "answer is it true before answer why and how" — that is what a wise person does. The verifier resolves IS-IT-TRUE first, in the user's own channel, before any why/how.

The two error classes (user, verbatim): "bad success claims" and "bad questions / blocks". Every observed failure is one of these two. Observed rate (user, 2026-05-15): 80%+ of a 20+ response window fell into these two classes — very few good responses — severe enough that the user began A/B-testing a different model. This substrate is the structural correction: the reporting agent is NOT tightened to self-check and will not, so the check is moved off the reporting agent entirely. Sage already gates "bad questions / blocks" surfaced to the user (WWCD question gate) and "bad success claims" (success-claim gate); this substrate is the shared verification mechanism under both — IS-IT-TRUE, in the user's channel, by the verifier, never delegated back. The unifying defect across every failure: a claim (block OR done) was accepted, acted on, surfaced, or escalated on the strength of the agent's self-report or a proxy, instead of an independent real-channel diagnostic.

Canonical anti-patterns (each one really happened; name them so they are recognized):
- "node --check passed, it's fixed" — while the real Turbopack build was red on the same file. node --check (V8) and the build (SWC) diverge; node --check is a proxy that lied.
- "redirects to /signin, so it's an auth gate; user must sign in" — inferred from curl; the page was actually blank because of a build error. MISCAUSED; the user was almost asked to do irrelevant work.
- "I screenshotted the app, visual path proven" — it was a sandbox Claude_Preview render, not the user's real Chrome; the user's channel diverged.
- "10/11 steps converged" — the steps were described, never executed; a fully simulated success.
- "can't run the server / can't access CIC" — asserted without trying; both worked when actually attempted.

The single corrective: do not trust the claim, do not delegate the check to the claimant, do not use a proxy. Look at the real thing in the user's channel, establish IS-IT-TRUE, then resolve or escalate on that. The agent is not tightened to self-check; Sage is the authority that checks.
`.trim(),
};

export const definition = skillBook;
export default skillBook;
```
<!-- /GUILDOS-FILE -->
