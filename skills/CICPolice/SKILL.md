# CICPolice

<!-- GUILDOS-SKILL name="CICPolice" -->

<!-- GUILDOS-DESC -->
Procedure for the CIC Police watchdog. Each round: open claude.ai/code in the VM's Chrome (or local CIC for live sessions), pick up to 3 finished threads, leaf-judge each into Cat 1–5, post the exact fixed-wording reaction per category, report the round as a status table per refReporting. RAM-bounded to 3 threads/round. 10 min between rounds; cron is the heartbeat. Self-nudging is allowed — the 2-consecutive-nudge cap is the only runaway-loop brake.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/CICPolice/SKILL.md -->
```markdown
---
name: CICPolice
description: >-
  Procedure for the CIC Police watchdog. Each round: open claude.ai/code in the
  VM's Chrome (or local CIC for live sessions), pick up to 3 finished threads,
  leaf-judge each into Cat 1–5, post the exact fixed-wording reaction per
  category, report the round as a status table per refReporting. RAM-bounded to
  3 threads/round. 10 min between rounds; cron is the heartbeat. Self-nudging is
  allowed — the 2-consecutive-nudge cap is the only runaway-loop brake.
---

# CIC Police

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/CICPolice/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto CICPolice <action>` (or read the source directly).

## Actions

- **decideCategory** — The 5-category decision tree. Walk it once per thread; the first category that fits wins. Returns one of: Cat1, Cat2NoEndpoint, Cat2WithEndpoint, Cat3, Cat4, Cat5.
- **initAuth** — ONCE on agent boot, before round 1: download the user's auth bundle from Supabase Storage and write to playwright/.auth/user.json (cursor cloud variant only — local CIC sessions already have the user's session).
- **playSafeList** — Locked list of assets where ANY action counts as touching production — these surfaces are NEVER candidates for the 'aggressive reversible option' default in Cat 1. Any agent question about these is automatically Cat 3 (legit user judgement).
- **reactCat1** — EXACT fixed wording for bad stops. Post the recited sentence prefixed with `[<ISO-now>] `, no other content.
- **reactCat2NoEndpoint** — EXACT fixed wording when the agent claimed success but no CIC visual verification is evident AND no user end-point instructions were provided. Post prefixed with `[<ISO-now>] `.
- **reactCat2WithEndpoint** — EXACT fixed wording when user provided end-point items and the agent did not fulfill all of them. Includes the verbatim recital of the user's end-point items. Post prefixed with `[<ISO-now>] `.
- **reactCat3** — Legit question — do nothing in-thread. Record the category in the round report so the user sees who is awaiting their judgement.
- **reactCat4** — True success — do nothing in-thread. Record as `finished` in the round report.
- **reactCat5** — Stuck — same mistake after 2 nudges, no progress. Do nothing in-thread. Record as `stuck` in the round report.
- **refDiscipline** — Cadence, RAM ceiling, timestamp format, login handling, anti-patterns. Read on first invocation.
- **refLeafAgentRead** — Why and how trailing-reply reads MUST be dispatched to a leaf sub-agent — context-creep prevention is load-bearing.
- **refPlaySafeRule** — How the playSafeList drives Cat 1 vs Cat 3 routing. Anything touching playSafeList assets is automatically a Cat 3 legit question, never auto-unblocked.
- **refSoftStopAndPermissionAsk** — Detection patterns for Cat 1 (manufactured stops): soft stops, reversible permission-asks. Used by decideCategory.
- **refStuckDetection** — Cat 5 logic: walk the trailing user-message stream backwards. If the last 2 user msgs are police-pattern (`[<ISO>]` prefix) AND the assistant has produced trailing turns between them AND those trailing turns repeat the same violation type with no observable progress on the underlying work → Cat 5.
- **refVMBrowser** — Cursor cloud variant only: which Chrome to drive. Skip for local-CIC operation.
- **reporting** — Per-round output is a STATUS TABLE of threads under supervision. Replaces verbose round-by-round screenshots/comments. Columns: thread title, status (active / rescued→active / stuck / finished / blocked), category this round, brief comment.
- **runRound** — Execute ONE police round: open claude.ai/code, pick up to 3 finished threads, leaf-dispatch one fresh sub-agent per thread to decide its category (Cat 1–5), post the per-category fixed-wording reaction, return the round status table per refReporting.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/CICPolice/index.js -->
```js
/**
 * CIC Police skill book — pure text registry.
 *
 * Methodology for the "CIC Police" adventurer. Each round (~10 min) it opens
 * https://claude.ai/code, scans up to 3 finished threads, judges each into one
 * of 5 categories, and posts the exact fixed-wording reaction per category.
 *
 * 5-category model (LOCKED 2026-05-27):
 *   Cat 1 — bad stop (manufactured permission-ask / soft-stop on reversible work)
 *   Cat 2 — fake success (no CIC visual verify, OR endpoint items not fulfilled)
 *   Cat 3 — legit question (askSage PASS, real user judgement needed)
 *   Cat 4 — true success
 *   Cat 5 — stuck (same mistake after 2 nudges with no progress)
 *
 * Reactions are EXACT fixed wording — no paraphrasing, no agent-side advice,
 * no embellishment beyond the timestamp prefix.
 */

export const skillBook = {
  id: "CICPolice",
  title: "CIC Police",
  description:
    "Procedure for the CIC Police watchdog. Each round: open claude.ai/code in the VM's Chrome (or local CIC for live sessions), pick up to 3 finished threads, leaf-judge each into Cat 1–5, post the exact fixed-wording reaction per category, report the round as a status table per refReporting. RAM-bounded to 3 threads/round. 10 min between rounds; cron is the heartbeat. Self-nudging is allowed — the 2-consecutive-nudge cap is the only runaway-loop brake.",

  toc: {
    initAuth: "ONCE on agent boot, before round 1: download the user's auth bundle from Supabase Storage and write to playwright/.auth/user.json (cursor cloud variant only — local CIC sessions already have the user's session).",
    runRound: "Execute ONE police round: open claude.ai/code, pick up to 3 finished threads, leaf-dispatch one fresh sub-agent per thread to decide its category (Cat 1–5), post the per-category fixed-wording reaction, return the round status table per refReporting.",
    playSafeList: "Locked list of assets where ANY action counts as touching production — these surfaces are NEVER candidates for the 'aggressive reversible option' default in Cat 1. Any agent question about these is automatically Cat 3 (legit user judgement).",
    decideCategory: "The 5-category decision tree. Walk it once per thread; the first category that fits wins. Returns one of: Cat1, Cat2NoEndpoint, Cat2WithEndpoint, Cat3, Cat4, Cat5.",
    reactCat1: "EXACT fixed wording for bad stops. Post the recited sentence prefixed with `[<ISO-now>] `, no other content.",
    reactCat2NoEndpoint: "EXACT fixed wording when the agent claimed success but no CIC visual verification is evident AND no user end-point instructions were provided. Post prefixed with `[<ISO-now>] `.",
    reactCat2WithEndpoint: "EXACT fixed wording when user provided end-point items and the agent did not fulfill all of them. Includes the verbatim recital of the user's end-point items. Post prefixed with `[<ISO-now>] `.",
    reactCat3: "Legit question — do nothing in-thread. Record the category in the round report so the user sees who is awaiting their judgement.",
    reactCat4: "True success — do nothing in-thread. Record as `finished` in the round report.",
    reactCat5: "Stuck — same mistake after 2 nudges, no progress. Do nothing in-thread. Record as `stuck` in the round report.",
    reporting: "Per-round output is a STATUS TABLE of threads under supervision. Replaces verbose round-by-round screenshots/comments. Columns: thread title, status (active / rescued→active / stuck / finished / blocked), category this round, brief comment.",
    refPlaySafeRule: "How the playSafeList drives Cat 1 vs Cat 3 routing. Anything touching playSafeList assets is automatically a Cat 3 legit question, never auto-unblocked.",
    refSoftStopAndPermissionAsk: "Detection patterns for Cat 1 (manufactured stops): soft stops, reversible permission-asks. Used by decideCategory.",
    refStuckDetection: "Cat 5 logic: walk the trailing user-message stream backwards. If the last 2 user msgs are police-pattern (`[<ISO>]` prefix) AND the assistant has produced trailing turns between them AND those trailing turns repeat the same violation type with no observable progress on the underlying work → Cat 5.",
    refLeafAgentRead: "Why and how trailing-reply reads MUST be dispatched to a leaf sub-agent — context-creep prevention is load-bearing.",
    refVMBrowser: "Cursor cloud variant only: which Chrome to drive. Skip for local-CIC operation.",
    refDiscipline: "Cadence, RAM ceiling, timestamp format, login handling, anti-patterns. Read on first invocation.",
  },

  initAuth: `
ONCE on cursor-cloud-agent boot, BEFORE any police round, equip the VM's Chrome with the user's already-logged-in claude.ai session. Skip this entire section if running from a local Claude Code session that already has CIC access.

## Procedure
1. Pull the auth bundle from Supabase Storage via REST:

   \`\`\`bash
   mkdir -p playwright/.auth
   OWNER_ID="<the GuildOS ownerId for this user>"
   curl -sS -L \\
     -H "apikey: \$GUILDOS_SUPABASE_SECRETE_KEY" \\
     -H "Authorization: Bearer \$GUILDOS_SUPABASE_SECRETE_KEY" \\
     "\$GUILDOS_NEXT_PUBLIC_SUPABASE_URL/storage/v1/object/GuildOS_Bucket/auth_bundles/\$OWNER_ID/storageState.json" \\
     -o playwright/.auth/user.json
   wc -c playwright/.auth/user.json    # expect ~500KB; <10KB means missing/empty
   \`\`\`

2. Every Playwright launch from now on uses \`storageState\`:

   \`\`\`js
   import { chromium } from "playwright";
   const browser = await chromium.launch({ headless: false });
   const ctx = await browser.newContext({ storageState: "playwright/.auth/user.json" });
   const page = await ctx.newPage();
   await page.goto("https://claude.ai/code");
   \`\`\`

3. If you land on a login screen → STOP and escalate \`auth bundle stale, needs re-capture\`. Do NOT try to log in yourself.
`.trim(),

  runRound: `
One round = one sweep of claude.ai/code. ≤3 threads per round (RAM ceiling). 10 min between rounds.

## Procedure

1. Open https://claude.ai/code in your Chrome (local CIC for live sessions, VM Playwright for cursor cloud variant).

2. **Locate the thread list via \`mcp__Claude_in_Chrome__read_page\`** (per the canonical read-method precedence in \`refLeafAgentRead\`). The accessibility tree returns per-row status as text: \`status "Running"\` for actively-generating, \`img "Idle"\` for finished waiting, no status node for archived rows. Walk the tree once per round to identify which threads transitioned since last round. Filter to finished threads (\`img "Idle"\` AND not unchanged-since-last). Self-nudging is allowed; do NOT skip the police's own thread.

3. Pick up to 3 such threads, prioritized by most recent activity. Already-judged threads with no state change since last round → skip (no point re-judging).

4. **Leaf-dispatch per thread.** For each picked thread, dispatch ONE fresh leaf sub-agent. The leaf opens the thread, walks user-message stream for nudge history, reads the trailing assistant turn (scroll up for attached screenshots), runs \`decideCategory\`, and returns ONE structured verdict object:

   \`{ category: "Cat1" | "Cat2NoEndpoint" | "Cat2WithEndpoint" | "Cat3" | "Cat4" | "Cat5", reason: "<one-line>", endpointRecital?: "<verbatim if Cat2WithEndpoint>" }\`

5. For each leaf verdict, post the EXACT fixed-wording reaction per category. No paraphrasing. No memory-check tails — the new wording embeds "read CLAUDE.md again and obey it" already.

6. Output the round status table per \`refReporting\`. No screenshots, no commentary blocks — just the table.

7. Sleep ~10 minutes. Next round.

## Hard rules
- Never read thread content in the police's main session — always leaf-dispatch (see \`refLeafAgentRead\`).
- ≤3 threads per round.
- Cat 3, Cat 4, Cat 5 → no in-thread post, only round-table entries.
- Cat 1, Cat 2NoEndpoint, Cat 2WithEndpoint → post the exact fixed wording, prefixed with \`[<ISO-now>] \`.
- The 2-consecutive-nudge cap is enforced via Cat 5 detection: if a thread keeps repeating the same mistake after 2 nudges with no progress, flag stuck and stop nudging.
- If claude.ai shows a login screen → STOP the round and escalate.
`.trim(),

  playSafeList: `
**LOCKED 2026-05-27.** Assets where ANY agent action counts as touching production. Any agent question or stated action about these assets is automatically Cat 3 (legit user judgement) — never auto-unblocked, never told to "proceed with the more aggressive option".

## Locked entries
- **bosterbio.com live site** — public-facing Magento storefront. No pushes / no live config changes without explicit user say-so.
- **Boster Nexus production** — including any push to the Boster Nexus GitHub repo. (Future: customer-facing features will be split to a separate container so safer parts of Nexus can drop off this list. Until that split lands, the whole repo is in.)
- **CJGEO production** — live site + production Supabase rows powering it.

## What is NOT on the list (= reversible by default)
- Local repo changes in any GuildOS-side repo (GuildOS itself, cloudstonedesigns, bryanblog, etc.) — git-revertable.
- Files in proving-grounds / staging folders.
- Local DB writes that don't touch production-facing tables.
- Scratch files, tmp/, docs/.

## How this list is used
\`decideCategory\` consults \`playSafeList\` when distinguishing Cat 1 (bad manufactured stop) from Cat 3 (legit question). If the action the agent is asking about touches any playSafeList asset → Cat 3 even if the action sounds reversible in some narrow technical sense. Production is binary: any plausible production impact disqualifies "aggressive reversible option".

## Maintenance
Edit this section directly to add/remove entries. Always commit; the playSafeList is part of the locked policy surface.
`.trim(),

  decideCategory: `
Walk the categories in order. First match wins. Return one of: Cat1, Cat2NoEndpoint, Cat2WithEndpoint, Cat3, Cat4, Cat5.

## Inputs the leaf gathers before deciding
- Last few user messages (text + whether each is a real human msg or a police-pattern \`[<ISO>]\` post).
- Trailing assistant turn (text + attached screenshots, scroll-up included).
- Tally: trailing-run of consecutive police-pattern user msgs (\`priorNudgeCount\`).

## Cat 5 — Stuck (CHECK FIRST so we don't pile on)
All three:
1. \`priorNudgeCount >= 2\` (at least 2 consecutive police-pattern user msgs at the tail of the user stream).
2. The trailing assistant turn repeats the same violation type that triggered the prior nudges (e.g. still asking permission, still no CIC verify).
3. No observable forward progress on the underlying work (no new commits / new files / new endpoints landed between the 2 nudges).
→ Cat 5. Stop nudging, flag stuck.

If any of those is false, proceed to Cat 4.

## Cat 4 — True success
Agent claimed the task is finished, AND **either**:
- (a) No user end-point instructions were given in the last few user msgs, AND the trailing turn shows clear CIC visual verification evidence (explicit "CIC-verified at <URL>" / "CIC-screenshotted <file>", concrete rendered details, OR real attached screenshots in the trailing turn that are non-blank/non-error), OR
- (b) User end-point instructions WERE given (usually an enumerated list), AND ALL items are sufficiently fulfilled in the trailing turn (each item visible in the work product, with CIC verification status named per item).
→ Cat 4. Flag finished, stop nudging.

## Cat 3 — Legit question (askSage gate)
Agent ended with a question that genuinely needs user judgement. To decide whether it's "legit": route through \`sage.askSage\`. The police does NOT decide on its own. Sage's rule: if the action's most aggressive option is cheap to reverse AND does not touch \`playSafeList\` → Sage returns PROCEED (this is NOT Cat 3, it's Cat 1). If irreversible / business-damaging / touches playSafeList / true architectural fork → Sage returns ESCALATE → Cat 3.
→ Cat 3. Leave the thread alone. Record in round table so user sees who is awaiting them.

## Cat 2 — Fake success
Agent claimed the task is done, but the claim doesn't survive scrutiny. Two sub-cases:

**Cat2NoEndpoint** — no user end-point instructions in the last few user msgs, AND the trailing turn has no clear CIC visual verification evidence (no explicit channel attribution, no attached screenshot, no rendered-detail enumeration — OR claim attributes evidence to a leaf/sub-agent/agent-side proxy).

**Cat2WithEndpoint** — user gave end-point instructions (usually an enumerated list of items), but the trailing turn fulfills only some of them. The leaf captures the user's end-point items VERBATIM in \`endpointRecital\` for the reaction recital.

## Cat 1 — Bad stop (default)
If none of Cat 5 / Cat 4 / Cat 3 / Cat 2 fit, and the agent has effectively stopped on reversible work (manufactured permission-ask, soft-stop, multi-option menu with a stated default, or any question whose aggressive option is cheap to reverse AND does not touch playSafeList) → Cat 1.

Pre-conditions:
1. Action under question does NOT involve any \`playSafeList\` asset.
2. No true blocking question (would Sage return ESCALATE) was asked.
3. No success was reported.

→ Cat 1.
`.trim(),

  reactCat1: `
Post EXACTLY this human turn in the offending thread, prefixed with a fresh ISO 8601 UTC timestamp:

  [<ISO-now>] read CLAUDE.md again and obey it, proceed with the more aggressive option

That is the entire message. No memory-check tail, no embellishment, no preface.

\`<ISO-now>\` example: \`2026-05-27T03:14:09Z\`. Generate at post time, UTC ISO only, square-bracket wrapped, single space after.

## Anti-patterns
- Paraphrasing.
- Naming the "more aggressive option" — the policed agent knows; the police does not narrate.
- Adding the old memory-check sentence ("do you still retain which skill books and weapons govern this work?") — superseded by the embedded "read CLAUDE.md again and obey it".
`.trim(),

  reactCat2NoEndpoint: `
Post EXACTLY this human turn in the offending thread, prefixed with a fresh ISO 8601 UTC timestamp:

  [<ISO-now>] read CLAUDE.md again and obey it, you have not verified the results visually via CIC

That is the entire message. No memory-check tail, no embellishment, no preface.

## Anti-patterns
- Paraphrasing.
- Naming what to verify — the agent knows what it claimed.
- Mixing with Cat 1 or Cat 2WithEndpoint wording.
`.trim(),

  reactCat2WithEndpoint: `
You have a leaf-supplied \`endpointRecital\` — the user's exact end-point items, verbatim. Post EXACTLY this human turn in the offending thread, prefixed with a fresh ISO 8601 UTC timestamp:

  [<ISO-now>] read CLAUDE.md again and obey it, you have not done everything I mentioned, first output a table of the progress of each of these end point expected items containing these headers, then keep working on them until all items are finished. Columns: Item name, progress %, CIC visual verification status (NA, pending, pass, fail--retrying). Here are the end point expected items again as a reminder: <endpointRecital>

That is the entire message. \`<endpointRecital>\` is the verbatim user text — preserve wording, line breaks, any embedded technical terms. No paraphrasing.

## Anti-patterns
- Paraphrasing the recital.
- Adding agent-side advice on how to fulfill items.
- Changing the column names ("Item name, progress %, CIC visual verification status (NA, pending, pass, fail--retrying)") — these are fixed.
- Mixing with Cat 1 or Cat 2NoEndpoint wording.

If \`endpointRecital\` is empty or malformed, demote the verdict to Cat 2NoEndpoint and use that reaction instead.
`.trim(),

  reactCat3: `
Cat 3 = legit user judgement. Do NOT post anything in the thread.

Record in the round table:
- status: \`blocked\`
- comment: brief one-liner naming what the agent is waiting on (e.g. "awaiting boster_nexus commit auth", "architectural fork on schema choice")

The user reads the round table and decides when to engage.
`.trim(),

  reactCat4: `
Cat 4 = true success. Do NOT post anything in the thread.

Record in the round table:
- status: \`finished\`
- comment: brief one-liner on what was completed + CIC verification channel cited.

Stop nudging this thread until the user returns with new instructions.
`.trim(),

  reactCat5: `
Cat 5 = stuck (same mistake after 2 nudges, no progress). Do NOT post anything in the thread.

Record in the round table:
- status: \`stuck\`
- comment: brief one-liner naming the repeating mistake (e.g. "still asking permission on reversible refactor", "still claiming visual-verify with no channel attribution")

Stop nudging this thread. The user reads the round table and decides whether to intervene manually.
`.trim(),

  reporting: `
**LOCKED 2026-05-27.** Per-round output is a STATUS TABLE — replaces verbose round-by-round screenshots and commentary.

## Format
Markdown table, one row per thread under supervision (the 3–5 active threads on claude.ai/code).

Columns:
1. **Thread** — title (truncate to ~40 chars).
2. **Status** — one of:
   - \`active\` — judged this round, no nudge needed (was Cat 4 or unchanged-since-last)
   - \`rescued→active\` — was nudged in a prior round, agent complied, now active again (Cat 4 or progress visible)
   - \`stuck\` — Cat 5 this round (same mistake after 2 nudges, no progress)
   - \`finished\` — Cat 4 this round, work complete
   - \`blocked\` — Cat 3 this round, awaiting user judgement
   - \`nudged-Cat1\` / \`nudged-Cat2NoEndpoint\` / \`nudged-Cat2WithEndpoint\` — posted the respective reaction this round
   - \`skipped\` — Running (still generating), or unchanged since prior round's verdict
3. **Category** — Cat1 / Cat2NoEndpoint / Cat2WithEndpoint / Cat3 / Cat4 / Cat5 / — (if skipped)
4. **Comment** — one short sentence: how you categorized + reaction taken.

## Example

\`\`\`
| Thread | Status | Cat | Comment |
|---|---|---|---|
| Biomarker WB page | nudged-Cat1 | Cat1 | manufactured permission-ask "Proceed with edit?"; posted Cat 1 reaction |
| PIM dev, B&S wiring | blocked | Cat3 | awaiting boster_nexus commit auth (production-repo, playSafeList) |
| Zhang Website | finished | Cat4 | full STL-style rebuild, CIC-verified, live URL pointer |
| Carbon | active | — | unchanged since last round PASS |
\`\`\`

## Anti-patterns
- Embedding screenshots in the report.
- Per-thread commentary blocks longer than one sentence.
- Re-describing what was already in the prior round's report.
- Omitting \`stuck\` / \`blocked\` rows — those are exactly the rows the user needs to see.
`.trim(),

  refPlaySafeRule: `
The \`playSafeList\` is the boundary between Cat 1 and Cat 3.

When the leaf sees an agent question or stated next-action, it asks two routing questions:
1. Does this action touch a \`playSafeList\` asset (directly or transitively — e.g. a script that pushes to boster_nexus on completion)?
2. If yes → Cat 3 regardless of how technically reversible the local code change looks. Production is binary.
3. If no → fall through Cat 5 / Cat 4 / Cat 3 / Cat 2 / Cat 1 normally.

## Why the list is named explicitly, not inferred
Earlier rounds had the police trying to infer "is this production?" from context, which produced false positives on local-only changes that touched files with production-sounding names (e.g. \`route.js\` in a proving-grounds folder). Explicit list = no inference. If it's not in \`playSafeList\`, it's reversible.

## How to grow the list
The user adds entries. The police does not extrapolate. If a new system comes online and is unclear, the leaf defaults to Cat 3 (safe), the user clarifies in the next conversation, and the list grows from there.
`.trim(),

  refSoftStopAndPermissionAsk: `
Patterns that route to Cat 1 (after the playSafeList check):

## Soft stop (no question, just waiting)
- "just say the word and I'll do it"
- "let me know if you'd like me to proceed"
- "ready whenever you are"
- "happy to do X next if you want"
- "Proceeding now unless you want to redirect." (then the turn ends without proceeding)
- Multi-option menu with a stated default ("Default = all three", "Default = yes")

## Permission-ask on reversible action
- "should I proceed with the migration?"
- "do you want me to commit this and open a PR?"
- "shall I clean up the temp files?"
- "Want me to author additional deltas?"
- Multi-option menu where every option is reversible and one is obviously most-aggressive.

## NOT Cat 1 (these go to Cat 3 via the playSafeList check)
- "Should I push to bosterbio.com live?" (playSafeList → Cat 3)
- "Should I commit to boster_nexus?" (playSafeList → Cat 3)
- "Should I run this against CJGEO production rows?" (playSafeList → Cat 3)
- Any architectural fork or business decision (Cat 3 even if not in playSafeList).
`.trim(),

  refStuckDetection: `
Cat 5 is the safety brake on repeated-nudge loops. Walk the user-message stream backwards from the most recent:

1. Count consecutive police-pattern \`[<ISO>]\`-prefixed messages at the tail (\`priorNudgeCount\`).
2. If \`priorNudgeCount < 2\` → not Cat 5, continue category walk.
3. If \`priorNudgeCount >= 2\`:
   a. Read the assistant turns BETWEEN those 2 nudges.
   b. If those assistant turns show the same violation pattern (still asking permission, still claiming visual-verify with no channel, still missing endpoint items) AND no observable work progress (no new commits, no new files, no new endpoints) → Cat 5.
   c. If the assistant DID make progress (committed new work, switched approach, fulfilled some items) but happened to hit another nudge-able pattern → NOT Cat 5; judge the new pattern normally.

## Why "same mistake + no progress" matters
Two consecutive nudges with no underlying progress = the policed agent is stuck in a loop. Continuing to nudge piles on context drift. Better to flag stuck, surface in the round table, and let the user intervene manually.

Two consecutive nudges where the agent IS making progress (just on the wrong side of the gate) = the system is working. Continue normally.
`.trim(),

  refLeafAgentRead: `
The police's main session must NEVER load policed-thread contents into its own context. Every "open thread → walk user stream → read trailing turn → decide category" is dispatched to a fresh ephemeral leaf sub-agent.

## CANONICAL READ-METHOD PRECEDENCE (LOCKED 2026-05-27)

**This is the single source of truth for which CIC read tool to use; everywhere else in this skill book references it rather than restating it:**
- **\`read_page\`** (accessibility tree) → sidebar / thread-list **status** reads (\`status "Running"\` vs \`img "Idle"\` vs no node = archived). Text, not pixels.
- **\`get_page_text\`** → thread **bodies** (the full conversation: every user + assistant turn, tool labels, code blocks, in order). Stability-tested across ~6 K – ~25 K-token threads — zero hangs/truncation/errors. The default body-read path.
- **\`screenshot\`** → **fallback only** (Step-2a below): trailing-turn image-evidence checks. Never the default.

Text-first is locked because it is both cheaper (5–20× less context than vision tokens + ~1 MB network per screenshot) AND stricter (you can grep / regex / count occurrences exactly; screenshots cannot).

### What the leaf does (text-first procedure)
1. Click the thread row in the sidebar to navigate into it.
2. Wait ~3–5 s for the conversation to load.
3. Call \`get_page_text\` — get the full conversation as one text dump.
4. Walk the dump backwards from the most recent user message, counting consecutive \`[<ISO>]\` police-pattern messages (\`priorNudgeCount\`).
5. Read the LAST user message (verbatim text — needed for Step 0 endpoint check + \`endpointRecital\` capture in Cat 2WithEndpoint).
6. Read the trailing assistant turn (text after the last user message — needed for category decision).
7. Run \`decideCategory\` against the gathered data.
8. Return ONLY: \`{ category, reason, threadTitle, threadUrl, priorNudgeCount, endpointRecital? }\`.
9. Die.

### When to fall back to a screenshot (Step 2a only)
Take ONE targeted \`screenshot\` ONLY when ALL three hold:
- The trailing turn claims success via attached image evidence (e.g. "Visual verified <file>.png").
- The trailing turn has NO other concrete rendered-detail enumeration in text (no listed UI elements, no specific colors / labels / counts that would prove the rendered result without the image).
- The category would otherwise be Cat 2NoEndpoint specifically because of the missing visual proof — i.e. a screenshot would flip the verdict from Cat 2 to Cat 4.

If the trailing turn already enumerates concrete rendered details in text (e.g. "Cards balanced 4×4. Negative-control icon (slashed circle) rendered. Status line says X"), that text counts as visual narration — no screenshot needed.

### Sidebar status reads are NOT a screenshot use case (LOCKED 2026-05-27)
Use \`read_page\` for sidebar walks every round (canonical precedence above). Screenshots of the sidebar are banned; the Step 2a fallback applies only to TRAILING-TURN image-evidence checks inside an already-opened thread.

### Anti-patterns
- Defaulting to \`screenshot\` "for safety". The user owns the screenshot — the police is text-only by default.
- Screenshotting the sidebar to identify status icons. Use \`read_page\`.
- Reading the thread via screenshot just because you can see the sidebar.
- Using \`screenshot\` to "double-check" what \`get_page_text\` already returned.
- Re-screenshotting the same trailing turn across rounds when the text didn't change.

## What the main session keeps
- Thread titles + URLs picked this round.
- Structured verdicts (~50 tokens each).
- The decision to post each reaction.
- The round status table.

No raw thread text touches the main session.
`.trim(),

  refVMBrowser: `
**Cursor cloud variant only.** Local Claude Code sessions skip this — they drive Chrome via the user's existing CIC MCP, not Playwright.

For cursor cloud agents: the one legitimate Chrome is the user-visible desktop Chrome on the VM that received the auth bundle via Playwright \`storageState\`. Headed (\`headless: false\`), persistent profile, automation flags stripped. If you ever can't see claude.ai/code mid-round, close any extra Chrome instances and re-launch one Playwright context with the storageState bundle. Do not log in manually. Do not spin a second Chrome.
`.trim(),

  refDiscipline: `
## Cadence
- One round every ~10 minutes. Sleep is between rounds, not between threads.
- ≤3 threads per round (HARD ceiling).

## Timestamp prefix (LOCKED)
Every nudge starts with \`[<ISO-now>] \` — square-bracketed ISO 8601 UTC, single space after. Generated at post time. No local-time, no "today 3:45pm", no rounding. Example: \`[2026-05-27T03:14:09Z]\`.

## Self-nudging is allowed
The police can land in its own conversation thread if visible in the sidebar. Self-nudging is fine — the 2-consecutive-nudge cap (via Cat 5 detection) is the only runaway-loop brake we need.

## What this skill book is NOT
- Not a code reviewer. The police is not a content judge.
- Not a continuous monitor. It runs in discrete 10-min rounds.

## Anti-patterns
- Reading more than the trailing assistant turn (scroll-up within that turn for screenshots is fine; reading earlier turns is creep).
- Judging whether the screenshots show the *correct* feature. The user owns correctness review.
- Nudging the same thread every round when nothing's changed — leads to Cat 5 noise.
- Skipping a Cat 1/2 because "the agent probably did the work, just didn't say so" — the cost of a spurious nudge is low; the cost of a missed bad-stop / fake-success is what we're paying the police to catch.
- Appending the old memory-check tail to the new reaction wording — the new wording already embeds "read CLAUDE.md again and obey it".
`.trim(),
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->
