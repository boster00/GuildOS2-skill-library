# housekeeping

<!-- GUILDOS-SKILL name="housekeeping" -->

<!-- GUILDOS-DESC -->
Contracts every adventurer follows + the operational lifecycle: init, writeQuest, escalate, comment, submit for review. Shared by ALL adventurers — merges the former codex book (contracts, named protocols, tier-column ownership, spawn contract, honest escalation) with the quest operational actions (design → html_markup → execute → purrview → review stages, deliverable self-checks, agent session health). Use for any core quest-lifecycle or adventurer-contract operation.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/housekeeping/SKILL.md -->
```markdown
---
name: housekeeping
description: >-
  Contracts every adventurer follows + the operational lifecycle: init,
  writeQuest, escalate, comment, submit for review. Shared by ALL adventurers —
  merges the former codex book (contracts, named protocols, tier-column
  ownership, spawn contract, honest escalation) with the quest operational
  actions (design → html_markup → execute → purrview → review stages, deliverable
  self-checks, agent session health). Use for any core quest-lifecycle or
  adventurer-contract operation.
---

# Local Work — Core Contracts & Operations

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/housekeeping/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto housekeeping <action>` (or read the source directly).

## Actions

- **clarifyQuest** — Clarify which quest the user means when their instructions are ambiguous.
- **cleanupProvingGround** — Manual-prompt-only cleanup of proving-ground subpages. Triggered by the user; never auto-fires.
- **comment** — Post a comment to a quest.
- **escalate** — Move a quest to escalated stage when truly blocked. Structured: writes a comment with detail.reason + detail.unblock_path so verifyQuestComplete can recognize escalated as a valid end state.
- **initAgent** — Initialize agent session — verify GuildOS access (provisioned by cursor.writeAgent's setup block), read instructions, check for work.
- **invokeNamedProtocol** — When the user says `questReport` / BCS / CBS / CSB (any 3-letter B/C/S permutation), WWCD, DSY, YUNOQR / YUNOBCS, or SW, look up the spec verbatim before responding.
- **mountAuthBundle** — Cursor agent: pull the user's exported browser auth bundle from Supabase Storage and write it to local disk so headless Playwright can mount it via `storageState`. Call ONCE at agent init for any task that needs authenticated UI access.
- **noBypassSpawnContract** — Spawning a cursor agent is only valid via cursor.writeAgent — direct POSTs to the Cursor API skip the GuildOS credentials block.
- **presentPlan** — Present a strategic plan with a separate, structured deliverables list. WBS lives in the strategy section as narrative; itemized artifacts live in `deliverables`.
- **readContractsFromMain** — Always read CLAUDE.md from the main branch on session start (worktree drift is the documented cause of past contract failures).
- **respectTierColumnOwnership** — Each of the 5 review-tier columns on items is owned by exactly one tier; never write to a column you don't own.
- **runDesignStage** — Walk a quest through the design stage: for each items row, generate an AI baseline image from the expectation text, vision-judge it, retry up to 3x on fail. When all approved, advance to html_markup via questDesign.submit.
- **runHtmlMarkupStage** — Translate each AI-approved design baseline into a static-HTML proving-ground page under app/(town)/proving-ground/<questId>/<item_key>/page.js. When all items have html_markup_url, call questHtmlMarkup.submit; user reviews and approves on GM Desk.
- **searchQuests** — Search your assigned quests (default: not complete).
- **seekHelp** — Contact the Questmaster (Cat) for approval or assistance.
- **setNewAgent** — Replace a broken agent session with a fresh one.
- **submitForPurrview** — Submit quest for Questmaster review. Hard-gated by code via questExecution.submit — no agent shortcut around it.
- **summarizeComments** — Compress old comments to prevent flooding. Keep latest 4, summarize the rest.
- **verifyAdventurerSession** — Pre-dispatch session-health probe. Calls cursor.syncSessionStatus to reconcile the adventurers DB row against the upstream Cursor API; returns whether the session can safely accept a followup. Required before any writeFollowup or writeAgent dispatch.
- **verifyDeliverable** — Pre-flight self-honesty checks for every item. Advisory — the questExecution.submit gate enforces a subset; the rest is on you.
- **verifyQuestComplete** — Programmatic check that a quest is in a valid user-ready end state. Required before any 'done' claim.
- **writeExpectationInLockedStyle** — items.expectation is the canonical single-sentence ground-truth claim — what the deliverable shows. Anchored, specific, no padding.
- **writeQuest** — Create a quest in execute stage with structured deliverables (not WBS-in-description).
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/housekeeping/index.js -->
```js
/**
 * Housekeeping skill book — shared by ALL adventurers.
 * Core contracts + operational actions for quest management.
 *
 * Merged 2026-04-27: the former `codex` book (contracts + named protocols +
 * tier ownership + locked expectation language + spawn contract + honest
 * escalation) is now part of housekeeping. Workers, questmasters, and the
 * Guildmaster all carry these rules — there's no value in splitting them
 * across two globally-loaded books.
 */

export const skillBook = {
  id: "localwork",
  title: "Local Work — Core Contracts & Operations",
  description: "Contracts every adventurer follows + the operational lifecycle: init, writeQuest, escalate, comment, submit for review.",
  steps: [],
  toc: {
    // ── Contracts (formerly codex) ────────────────────────────────────────
    readContractsFromMain: {
      description: "Always read CLAUDE.md from the main branch on session start (worktree drift is the documented cause of past contract failures).",
      howTo: `On every session start: run \`git -C /workspace show main:CLAUDE.md\` (or \`git show main:CLAUDE.md\` if /workspace is GuildOS) and treat THAT as authoritative — not the auto-loaded file. If the auto-loaded copy differs from main, main wins without exception.

Why: a session can be rooted in a stale worktree. Auto-loaded CLAUDE.md is a courtesy, not source of truth. Worktree drift caused the 2026-04-26 BCS / WWCD failures and the description-as-status incident. Don't repeat them.

Refresh ritual: also re-run this on long sessions if contracts get fuzzy. Cheap to repeat.`,
    },

    invokeNamedProtocol: {
      description: "When the user says `questReport` / BCS / CBS / CSB (any 3-letter B/C/S permutation), WWCD, DSY, YUNOQR / YUNOBCS, or SW, look up the spec verbatim before responding.",
      howTo: `Named interaction protocols are domain terms with defined contracts. They don't follow training-data shapes. The specs no longer live in CLAUDE.md — read them from the skill books named below:

- **questReport / BCS / CBS / CSB / BSC / SBC / SCB** (the word "questReport" or any legacy 3-letter B/C/S permutation) → read \`sage.questReport\` howTo. The managerial-decision report (renamed from "BCS / Briefing CJ Style" and re-homed into the sage skill book 2026-06-09): five components — Health line, Item|Progress|Δ|Status table, question list, closing line with \`100%\`, User actions line. Also auto-required by \`sage.askSage\` on execution/research multi-item turns.
- **YUNOQR / YUNOBCS** (why you no Quest Report) → also \`sage.questReport\` (the YUNOQR/YUNOBCS shortphrase lives at the bottom of that action).
- **WWCD, DSY, SW** → read \`guildmaster.complianceNudge\` howTo. Combined handler for these protocol violations.

Do NOT look in CLAUDE.md for these specs — those sections have been migrated into the skill books. Do not free-style from training data.`,
    },

    writeExpectationInLockedStyle: {
      description: "items.expectation is the canonical single-sentence ground-truth claim — what the deliverable shows. Anchored, specific, no padding.",
      howTo: `**Rule (relaxed 2026-05-04 with the design-pipeline migration):** expectation is a single-sentence ground-truth claim that anchors what the deliverable shows. The locked \`"In the screenshot we should see…"\` prefix is no longer mandatory — write whatever sentence is canonical. The design image must achieve user-acceptable similarity to it; the purrview judge reads it as the claim; CJ can edit it inline on the GM Desk carousel ("edit" button next to the expectation).

**Style guidance (still recommended for clarity):**
- Screenshots: \`"In the screenshot, we should see <subject/UI element> showing <state or content> with these details: <specific, numbered facts>."\`
- Docs (.md, .json, .txt): \`"In the document, we should see <subject> covering <scope> with these details: <facts>."\`
- Other: \`"In the artifact, we should see <subject> demonstrating <property> with these details: <facts>."\`

**Anchored vs vague:**
- Bad: "image shows HTTP 200." (no anchor — what page? where?)
- Good: "In the screenshot, we should see the /api/track fire-response panel showing HTTP 200 with these details: response body is {inserted:8, rejected:[], errors:[]}."

The expectation is the literal claim handed to \`openai_images.judge\` (with baselineUrl=design_url at purrview time). Specific and anchored beats clever.

Full guidance: \`presentPlan\` in this same book.`,
    },

    respectTierColumnOwnership: {
      description: "Each of the 5 review-tier columns on items is owned by exactly one tier; never write to a column you don't own.",
      howTo: `5 review-tier columns on \`items\`:
- \`self_check\`     — T0, owned by the worker at submit time (worker self-claim).
- \`openai_check\`   — T1, owned by the OpenAI judge (\`openai_images.judge\` weapon).
- \`purrview_check\` — T2, owned by Cat (questPurrview.approve / .bounce).
- \`claude_check\`   — T3.5, owned by the Guildmaster's local Claude direct multimodal review (questReview.pass / .bounce).
- \`user_feedback\`  — T4, owned by the user via the GM-desk Feedback button.

A tier may overwrite its OWN column (e.g. T1 re-judging is fine). It must NEVER overwrite another tier's column. The 2026-04-26 calibration laundering happened because a downstream layer rewrote upstream verdicts; that's structurally banned.`,
    },

    noBypassSpawnContract: {
      description: "Spawning a cursor agent is only valid via cursor.writeAgent — direct POSTs to the Cursor API skip the GuildOS credentials block.",
      howTo: `Every cursor agent spawn must go through \`cursor.writeAgent\`. The weapon prepends a setup block to the spawn prompt that provisions GUILDOS_NEXT_PUBLIC_SUPABASE_URL + GUILDOS_SUPABASE_SECRETE_KEY into the agent's ~/.guildos.env.

Without that block, agents end up with their project's Supabase but not GuildOS — the documented 2026-04-26 ptglab failure mode. The agent does the work, can't post items rows, can't call submitForPurrview, and either escalates honestly or fakes artifacts.

If you're tempted to do a direct POST to https://api.cursor.com/v0/agents — stop, use cursor.writeAgent. The credentials block is the whole point.`,
    },

    // ── Operations ────────────────────────────────────────────────────────
    initAgent: {
      description: "Initialize agent session — verify GuildOS access (provisioned by cursor.writeAgent's setup block), read instructions, check for work.",
      howTo: [
        "**On session start or re-init:**",
        "",
        "**Step 0: Verify GuildOS credentials are loaded**",
        "Every agent spawned via `cursor.writeAgent` (the only supported spawn path) gets a setup block at the top of its prompt that creates `~/.guildos.env` and sources it. By the time you run initAgent, that block should already have executed.",
        "Verify: `echo \"$GUILDOS_SUPABASE_SECRETE_KEY\" | head -c 12` should print a non-empty prefix. `echo \"$GUILDOS_NEXT_PUBLIC_SUPABASE_URL\"` should print `https://<REDACTED-PROJECT-REF>.supabase.co`. The basic-named SUPABASE_SECRETE_KEY and NEXT_PUBLIC_SUPABASE_URL are also exported as fallbacks.",
        "If GUILDOS_-prefixed vars are empty: the writeAgent setup block was skipped or your shell lost the source. Run `source ~/.guildos.env` once more. If that still fails, your spawn was created without the credentials block — escalate immediately with `housekeeping.escalate` (`detail.reason='spawn missing GuildOS credentials block'`, `detail.unblock_path='Guildmaster respawns this agent via cursor.writeAgent (the only supported path)'`). Do NOT try to work around this gap by uploading to your project's Supabase — that's the documented 2026-04-26 ptglab failure mode.",
        "",
        "**Step 1: Confirm /workspace = GuildOS**",
        "All agents are spawned with **GuildOS as the main repo** (locked 2026-04-27 for consistent agent behavior). Cursor checks GuildOS out into `/workspace` automatically. Verify: `cd /workspace && git remote -v | head -1` should show `boster00/GuildOS`. Pull latest: `cd /workspace && git pull origin main && npm install`. This gives you: global instructions (CLAUDE.md), skill books, weapons, env-cred templates — without any extra clone.",
        "If `/workspace` is NOT GuildOS, your spawn used the wrong repository — escalate (`detail.reason='spawn repo != GuildOS'`, `detail.unblock_path='Guildmaster respawns via cursor.writeAgent with repository=github.com/boster00/GuildOS'`).",
        "",
        "**Step 2: Clone your project repo (if you're a project-worker class)**",
        "If your role works on a specific project (CJGEO Dev → cjgeo, BosterBio Website Dev → bosterbio.com2026, Nexus Armor Dev → boster_nexus, etc.), clone that repo as a sibling so you have both:",
        "```bash",
        "cd ~ && git clone https://github.com/boster00/<your-project-repo>.git",
        "cd ~/<repo> && git pull origin main && npm install",
        "```",
        "Your `adventurers.system_prompt` names which project repo you own (read it next in step 3). Questmasters and pure-orchestration roles don't need a project clone — GuildOS is enough.",
        "",
        "**Step 3: Persist creds into GuildOS .env.local (already at /workspace)**",
        "The repo's `.env.local` is gitignored and missing on fresh clones. Mirror your shell vars into it so node scripts that read `--env-file=.env.local` work:",
        "```bash",
        "cat > /workspace/.env.local <<EOF",
        "GUILDOS_NEXT_PUBLIC_SUPABASE_URL=$GUILDOS_NEXT_PUBLIC_SUPABASE_URL",
        "GUILDOS_SUPABASE_SECRETE_KEY=$GUILDOS_SUPABASE_SECRETE_KEY",
        "NEXT_PUBLIC_SUPABASE_URL=$NEXT_PUBLIC_SUPABASE_URL",
        "SUPABASE_SECRETE_KEY=$SUPABASE_SECRETE_KEY",
        "EOF",
        "```",
        "Do the same in your project repo's `.env.local` so its node scripts can also reach GuildOS for quest bookkeeping.",
        "Quest operations (read/write quests, items, quest_comments) use the GuildOS Supabase. Storage uploads to `GuildOS_Bucket` use the same project.",
        "",
        "**Step 3: Read instructions and load your skill books (index first, content on demand)**",
        "1. Read ~/guildos/CLAUDE.md. Pay attention to the 'Read, don't improvise' and 'Index first, content on demand' rules — they apply to everything below.",
        "2. Query your adventurer profile from Supabase (adventurers table by your ID) to get system_prompt and skill_books.",
        "3. Read your system_prompt in full — project identity and conventions.",
        "4. Load skill book tocs only (NOT full howTos). Use this one-liner for each book (housekeeping first, then each id in your `skill_books` array):",
        "   ```",
        "   cd ~/guildos && node -e \"import('./libs/skill_book/<NAME>/index.js').then(m=>{const sb=m.skillBook||m.definition;const t=sb?.toc||{};console.log(JSON.stringify(Object.fromEntries(Object.entries(t).map(([k,v])=>[k,v?.description||(typeof v==='string'?v.split('\\\\n')[0].slice(0,120):'')])),null,2));})\"",
        "   ```",
        "   This prints a compact { actionName: description } map. Keep that in working context. Do NOT read the full file.",
        "5. When you're about to execute a specific action, open ONLY that action's `howTo` (e.g. `grep -A 40 '<actionName>:' libs/skill_book/<NAME>/index.js`). Follow the instruction verbatim. Discard the howTo from working context after the action completes.",
        "6. Your assigned books are the default working set, not a hard limit. When stuck with no matching action, BEFORE escalating: scan the registry at libs/skill_book/index.js, extract the toc of any book whose id looks relevant (same one-liner above), and try. Only escalate after that fallback is exhausted.",
        "7. **Never perform a non-trivial action natively if an instruction exists.** If in doubt, grep the skill books for the action name before freestyling.",
        "",
        "**Step 4: Check for work**",
        "Use searchQuests to find quests assigned to you.",
        "Work on the highest priority quest first (high > medium > low).",
        "",
        "**When to re-init:** When told to refresh context, switching projects, or after environment errors.",
      ].join("\n"),
    },
    setNewAgent: {
      description: "Replace a broken agent session with a fresh one.",
      howTo: `
**When:** initAgent fails due to stale repo, broken environment, or unrecoverable errors.

**Steps:**
1. Before giving up, \`git push\` any uncommitted work to preserve progress.
2. Check which branch you were on: \`git branch --show-current\`. Note this branch name.
3. Tell the user: "My environment is broken. Please create a new Cursor cloud agent at cursor.com for repo [repo name], branch main. Then give me the new agent ID. My work-in-progress is on branch [branch name]."
4. Once the user provides the new agent ID, update the adventurer record:
   \`\`\`javascript
   await db.from('adventurers').update({
     session_id: '<new-agent-id>',
     session_status: 'idle'
   }).eq('id', '<adventurer-id>');
   \`\`\`
5. Send initAgent to the NEW agent session (via Cursor API writeFollowup).
6. Tell the new agent to inherit the old branch: "After init, run: git fetch origin && git checkout [old-branch-name] to pick up where the previous session left off."
7. Verify the new agent responds and passes the environment check.
8. Once confirmed running smoothly, archive the old agent:
   \`\`\`javascript
   // Update the outpost record if one exists
   await db.from('outposts').update({ status: 'archived' }).eq('session_id', '<old-agent-id>');
   \`\`\`

**The user's manual step:** Create the agent at cursor.com. Everything else is automated.
`,
    },
    comment: {
      description: "Post a comment to a quest.",
      howTo: `
Insert a row into quest_comments with: quest_id, source (adventurer), action (milestone/escalate/response), summary.

Only comment for significant events: major milestone completion, escalation, responding to feedback. Not routine progress.
`,
    },
    escalate: {
      description: "Move a quest to escalated stage when truly blocked. Structured: writes a comment with detail.reason + detail.unblock_path so verifyQuestComplete can recognize escalated as a valid end state.",
      howTo: [
        "Escalate only when truly blocked — you cannot proceed at all.",
        "If a workaround exists (use placeholder image, skip non-critical step, etc.), note the issue in a regular comment and keep working. Reserve formal escalation for situations where you cannot make any progress.",
        "",
        "**Structured escalation contract.** `escalated` is a valid user-ready end state IFF the escalation comment carries structured `detail.reason` (what specifically failed) and `detail.unblock_path` (what the user must do to unblock). `housekeeping.verifyQuestComplete` refuses Mode A on escalated quests that lack these fields. Vague escalations are abandonment, not handoff.",
        "",
        "Escalation target: Guildmaster (higher-privilege actor with local machine access).",
        "",
        "Steps:",
        "1. Insert a comment with the structured payload:",
        "   ```javascript",
        "   import { recordQuestComment } from '@/libs/quest';",
        "   await recordQuestComment(questId, {",
        "     source: 'housekeeping',",
        "     action: 'escalate',",
        "     actor_name: '<your-adventurer-name>',",
        "     summary: '<one-sentence headline of what blocks you>',",
        "     detail: {",
        "       reason: '<specific, exact error or blocker — paste the error message verbatim if there is one>',",
        "       unblock_path: '<what the user (or Guildmaster) must do to unblock — specific: a credential to add, a file to drop, a decision to make>',",
        "       options: ['<choice 1 — concrete>', '<choice 2 — concrete>', '<choice 3 — your recommendation, marked>'], // OPTIONAL: include ONLY when the unblock is a DECISION among alternatives (not a single missing input)",
        "       attempted: ['<what you tried>', '<and why it failed>'],",
        "     },",
        "   });",
        "   ```",
        "2. Update the quest stage to `escalated` via the standard quest update path.",
        "3. The Guildmaster will resolve directly or surface to the user; once resolved, the quest moves back to execute and you continue.",
        "4. If you have other active quests, work on the next highest-priority one while waiting.",
        "",
        "Quality bar (agent best effort — the gate cannot enforce these but Cat / Guildmaster will reject vague escalations):",
        "- `reason` is a specific symptom, not a category. ❌ 'auth failed'. ✅ 'Telnyx API returns 403 \"insufficient permissions\" when calling /v2/messaging_profiles; the API key lacks messaging_profile:write scope.'",
        "- `unblock_path` is a single concrete next action by an identifiable actor. ❌ 'fix the auth'. ✅ 'User needs to regenerate the Telnyx API key with messaging scope enabled at https://portal.telnyx.com/#/auth-keys and put it in profiles.env_vars.'",
        "- **When the unblock is a DECISION the user must make (a fork among alternatives, not a missing input), do NOT hand them an open-ended question.** Populate `detail.options` AND render those same options in `unblock_path` as a NUMBERED LIST with linebreaks (`1.\\n2.\\n3.`), each a concrete choice, with your recommendation explicitly marked. The boss decides by picking a number — that is the GuildOS manager experience; making them author the option space is the tactical work the system exists to absorb. ❌ 'decide how to handle the duplicate contacts'. ✅ 'Choose one:\\n1. Merge on email — fast, may over-merge ~3%.\\n2. Merge on email+name — safer, leaves ~50 dupes.\\n3. Skip dedup, flag for manual review — RECOMMENDED, preserves all data.'",
        "",
        "Do NOT escalate for things you can figure out yourself or for permission to proceed — just proceed.",
        "Do NOT escalate with vague descriptions like 'something went wrong' or 'need help'. They will be bounced back.",
      ].join("\n"),
    },
    mountAuthBundle: {
      description: "Cursor agent: pull the user's exported browser auth bundle from Supabase Storage and write it to local disk so headless Playwright can mount it via `storageState`. Call ONCE at agent init for any task that needs authenticated UI access.",
      howTo: [
        "**When to use this.** Cursor cloud agents are sandboxed VMs that can't share the user's logged-in Chrome. For tasks needing authenticated UI access (Magento admin, Zoho UI, internal dashboards), the user runs `scripts/auth-capture.mjs --upload` locally; the bundle lands in Supabase Storage at `auth_bundles/<ownerId>/storageState.json`. This action pulls it.",
        "",
        "**Public-only tasks don't need this** (cherry farms, cabin-camping research, etc. — public sites). Skip if your quest is public-only.",
        "",
        "Usage:",
        "```javascript",
        "import { downloadBundle } from '@/libs/weapon/auth_state';",
        "const r = await downloadBundle({ ownerId: process.env.GUILDOS_OWNER_ID });",
        "if (!r.ok) {",
        "  // No bundle exists. Either ask the user to run auth-capture --upload,",
        "  // or proceed with a public-only path (and escalate if auth wall hit).",
        "}",
        "// r.path is now writeable; mount in Playwright:",
        "const browser = await chromium.launch();",
        "const context = await browser.newContext({ storageState: r.path });",
        "```",
        "",
        "**Freshness check.** The bundle has a default expiry (~7 days). Call `auth_state.readExpiryStatus({statePath})` before relying on it. If `needsRefresh: true`, escalate via `housekeeping.escalate` with `unblock_path: 'User to run scripts/auth-capture.mjs --upload to refresh expired cookies for: <expiredDomains>'`.",
        "",
        "**Don't ship credentials in screenshots.** If a deliverable screenshot inadvertently captures an account name, password, or session token, recapture before writeItem.",
      ].join("\n"),
    },
    verifyAdventurerSession: {
      description: "Pre-dispatch session-health probe. Calls cursor.syncSessionStatus to reconcile the adventurers DB row against the upstream Cursor API; returns whether the session can safely accept a followup. Required before any writeFollowup or writeAgent dispatch.",
      howTo: [
        "**Substrate guard.** Cursor sessions have an upstream lifecycle (CREATING → RUNNING → FINISHED → EXPIRED) that the adventurers DB does NOT auto-reflect. Sending a followup to an EXPIRED session returns 409 'Agent is deleted' silently breaking the dispatch. Always probe first.",
        "",
        "Call this before EVERY dispatch (writeFollowup or writeAgent). It costs one Cursor API call.",
        "",
        "```javascript",
        "import { syncSessionStatus } from '@/libs/weapon/cursor';",
        "const probe = await syncSessionStatus({ adventurerName: 'Researcher' });",
        "if (!probe.dispatch_safe) {",
        "  // probe.upstream_status: 'EXPIRED' | 'deleted_or_unreachable' | 'ERROR' | ...",
        "  // Either: respawn a fresh Cursor agent (cursor.writeAgent) and update adventurers.session_id,",
        "  //   OR pick a different adventurer for this task.",
        "  // Do NOT proceed to dispatch — the followup will silently fail.",
        "}",
        "// probe.dispatch_safe === true → followup is expected to land",
        "```",
        "",
        "Drift handling: if the local DB says `session_status='idle'` but upstream is EXPIRED, this function reconciles the DB to `'expired'` so subsequent code paths see the truth. `probe.was_drift === true` indicates a reconciliation just happened.",
        "",
        "Respawn protocol when `dispatch_safe === false`:",
        "  1. Per the no-repurpose rule (CLAUDE.md): the existing adventurer's identity is preserved; only the cursor session is refreshed.",
        "  2. Call `cursor.writeAgent({ repository, ref })` to spawn a fresh session bound to the adventurer's repo.",
        "  3. Update `adventurers.session_id` to the new session id; keep name, backstory, system_prompt, capabilities, skill_books unchanged.",
        "  4. Re-probe to confirm the new session is dispatch_safe.",
        "  5. Then send the followup.",
      ].join("\n"),
    },
    verifyQuestComplete: {
      description: "Programmatic check that a quest is in a valid user-ready end state. Required before any 'done' claim.",
      howTo: [
        "**This is the scripted version of the completion criteria.** Call it before declaring a quest done in any output (Mode A, BCS terminal verdict, agent task close-out). It returns ok:true only when:",
        "  - quest.stage === 'review' AND a `questReview.final_gate_pass` comment with the FINAL_GATE_PASS lockphrase exists, OR",
        "  - quest.stage === 'escalated' AND the latest `action='escalate'` comment carries non-empty `detail.reason` AND non-empty `detail.unblock_path`.",
        "Anything else returns ok:false with `missing` (the specific gate that failed) and `reason` (a one-sentence description for the agent to read).",
        "",
        "Usage:",
        "```javascript",
        "import { verifyQuestComplete } from '@/libs/quest';",
        "const v = await verifyQuestComplete({ questId });",
        "if (!v.ok) {",
        "  // v.missing: ['final_gate_pass' | 'final_gate_pass_lockphrase' | 'escalation_comment' | 'escalation_reason' | 'unblock_path' | 'end_state']",
        "  // v.reason: human-readable why this is not a valid end state",
        "  // Don't claim done; act on v.reason.",
        "}",
        "// v.ok === true when end_state ∈ ('review_ready', 'escalated_blocked')",
        "```",
        "",
        "Conventions on what counts as 'done' for this action:",
        "- **Code-checkable conditions** (existence) are enforced here.",
        "- **Quality conditions** (is the reason specific? is the unblock_path actionable?) are agent best-effort enforced upstream by housekeeping.escalate's contract and Cat's purrview review.",
        "Do not bypass this — write your own end-state check and you'll diverge from the canonical contract.",
      ].join("\n"),
    },
    presentPlan: {
      description: "Present a strategic plan with a separate, structured deliverables list. WBS lives in the strategy section as narrative; itemized artifacts live in `deliverables`.",
      howTo: `
**Read before you plan.** When the task references external resources (Figma files, URLs, docs, repos), read them BEFORE presenting. Plan from evidence, not speculation.

**Format — two sections:**

### Strategy
1–3 short paragraphs covering: goal, source of truth, scope (in / out), key dependencies, and the high-level approach. WBS-style phasing here is fine when sequence matters ("phase 1 ingest, phase 2 transform, phase 3 ship"). Do NOT itemize each artifact in this section — it's narrative, not a checklist.

### Deliverables (structured)
A list, one entry per artifact you'll ship. Each entry MUST have:
- \`item_key\` — stable kebab/snake identifier you'll reuse on the items row at submission time. Example: \`d1_route_response\`.
- \`expectation\` — the **reviewer-facing claim** about the artifact, written so the Questmaster + the user know what to look for at a glance. Required style:
  - For screenshots: **"In the screenshot, we should see &lt;subject/element&gt; showing &lt;state or content&gt; with these details: &lt;numbered, specific facts&gt;."**
  - For docs (.md, .json, .txt): **"In the document, we should see &lt;subject&gt; covering &lt;scope&gt; with these details: &lt;facts&gt;."**
  - For other artifacts: **"In the artifact, we should see &lt;subject&gt; demonstrating &lt;property&gt; with these details: &lt;facts&gt;."**
  Example (good): \`"In the screenshot, we should see the /api/track fire-response panel showing HTTP 200 with these details: response body is {inserted:8, rejected:[], errors:[]}; sentinel user_pseudo_id appears in the row preview."\`
  Example (bad — too terse, no anchor): \`"image shows HTTP 200 and {inserted:8} JSON body"\` — the reviewer doesn't know where on the page to look or what counts as success.
  The expectation is the literal claim the gpt-4o judge (T1) is given verbatim, AND the literal claim the user reads in the side panel before reviewing. Specific, anchored, numbered.

**Rules:**
- The submitForPurrview gate refuses to advance unless every items row has \`url IS NOT NULL\`. Each entry is one screenshot/file you'll actually ship — keep the list tight and real.
- Don't restate the WBS as deliverables. Phases ≠ artifacts. A phase can produce multiple deliverables, or none.
- Present both sections to the user for iteration before creating the quest.
`,
    },
    writeQuest: {
      description: "Create a quest in execute stage with structured deliverables (not WBS-in-description).",
      howTo: `
**Flow:**
1. User describes a project/task in chat
2. Present a strategic plan (presentPlan action) — strategy paragraphs + structured deliverables list
3. Iterate with the user until they approve

**Pre-execution checklist — do NOT create the quest until ALL are satisfied:**
- \`description\`: strategic context only (goal, source of truth, scope, dependencies). NO itemized artifact list.
- \`deliverables\`: array of \`{item_key, expectation}\` — one entry per planned artifact, expectation in the reviewer-facing style (see presentPlan). The submit gate enforces this.
- Priority assigned (high/medium/low).

4. Ask: "I have everything. Shall I create this quest and start working on it?"
5. On confirmation, insert the quest in \`execute\` stage:

\`\`\`javascript
import { writeQuest } from "@/libs/quest";
const { data, error } = await writeQuest({
  userId,
  title: '<quest title>',
  description: '<strategic context — goal, source of truth, scope. NO itemized artifact list>',
  deliverables: [
    {
      item_key: 'd1_route_response',
      expectation: 'In the screenshot, we should see the /api/track fire-response panel showing HTTP 200 with these details: response body is {inserted:8, rejected:[], errors:[]}; sentinel user_pseudo_id is visible in the request log.',
    },
    {
      item_key: 'd2_bq_rows',
      expectation: 'In the screenshot, we should see the BigQuery rows-landed preview showing 8 of 8 rows for the smoke run with these details: each row carries the same sentinel user_pseudo_id; event_name column lists the Tier-1 allowlist (page_view, session_start, purchase, form_submit, GAds Conversion, P1.Search, P2.ClickProductLink, P4.AddToCart).',
    },
    // ... one per planned artifact, each in the same reviewer-facing style
  ],
  stage: 'execute',
  priority: 'medium',
  assigneeId: '<your-adventurer-id>',
  assignedTo: '<your-name>',
});
\`\`\`

The \`deliverables\` column is JSONB. Each entry's \`item_key\` becomes the key for the corresponding \`items\` row at submission time — keep them stable, don't rename mid-quest.
`,
    },
    clarifyQuest: {
      description: "Clarify which quest the user means when their instructions are ambiguous.",
      howTo: `
1. Look up your currently assigned quests.
2. Present the relevant ones and ask: "Which quest is this for, or should I create a new one?"
`,
    },
    seekHelp: {
      description: "Contact the Questmaster (Cat) for approval or assistance.",
      howTo: `
Cat is a DB adventurer. Look up her session and send a followup via the cursor weapon:

\`\`\`javascript
// 1. Find Cat's session id
const { data } = await db.from('adventurers').select('session_id').eq('name', 'Cat').single();

// 2. Send a followup identifying yourself, your quest, and what you need
import { writeFollowup } from '@/libs/weapon/cursor';
await writeFollowup({
  agentId: data.session_id,
  message: "Quest <id> (<title>): <what you need>. Deliverable URLs if seeking review: ...",
});
\`\`\`

If Cat can't help, escalate to the Guildmaster via the escalate action.
`,
    },
    searchQuests: {
      description: "Search your assigned quests (default: not complete).",
      howTo: `
Query the quests table for your assigned quests that are not in complete stage. Order by priority (high > medium > low).
`,
    },
    verifyDeliverable: {
      description: "Pre-flight self-honesty checks for every item. Advisory — the questExecution.submit gate enforces a subset; the rest is on you.",
      howTo: [
        "Run this BEFORE `submitForPurrview`. The submit gate already enforces:",
        "  - every items row has `expectation` set (declared spec)",
        "  - every items row has `url` set (no pending items)",
        "  - every item has ≥1 item_comments rationale row",
        "  - every item.url responds with content > 0 bytes",
        "",
        "These additional checks are NOT enforced in code but they're what gets you bounced by Cat anyway. Self-reject before submitting:",
        "",
        "1. **File-size sanity.** A real page capture at 1400px is rarely under ~80KB. Under ~50KB is almost certainly a placeholder/empty/error page. Re-capture.",
        "2. **URL match.** The URL bar in the screenshot (or captured route) must match the route you claim. `/products` showing a 404 or `/login` fails.",
        "3. **HTTP 200 at capture time.** If the server returned 500/4xx/empty, the capture is invalid regardless of how it looks. Confirm before capturing.",
        "4. **No dev overlays.** Next.js/Vite error overlays, `N Issues` badges, hot-reload toasts visible = auto-bounce. Dismiss or fix.",
        "5. **Expected content present.** Blank product images, zero rows on a catalog, never-resolved skeleton loaders all fail. Each item's `expectation` defines what 'real content' means for that row — the imageJudge / Cat verifier grades against it.",
        "6. **Stock-image substitution = bounce.** If you uploaded a beach photo with a description claiming it's a sales chart, Cat will catch it on vision review and bounce. Always upload the actual artifact.",
        "7. **No fabricated scores.** Don't invent Figma-fidelity scores or test counts. If you didn't run/read it, don't claim it.",
        "8. **Per-item comment is the artifact rationale, not a commit log.** `commit abc123` is not a rationale; `catalog page shows 5 real products with filter sidebar — matches accept_criteria \"page must show ≥3 products with images\"` is.",
        "9. **Correct repo/workspace.** If the quest is for a different repo and your session is in GuildOS, switch before starting.",
        "",
        "### If a check fails",
        "Fix the artifact, re-upload to Supabase Storage, call `writeItem` with the SAME item_key (UPSERT replaces in place — never invent `_v2` keys).",
        "If the failure is environmental (system down) and you can't produce a real artifact, ESCALATE the quest. Do not submit known-bad work.",
        "",
        "### If all checks pass",
        "Proceed to `submitForPurrview`. The submit gate will run the enforced subset; Cat's review will run the rest.",
      ].join("\n"),
    },
    submitForPurrview: {
      description: "Submit quest for Questmaster review. Hard-gated by code via questExecution.submit — no agent shortcut around it.",
      howTo: [
        "**THE GATE IS THE ONLY PATH.** Do not write `quests.stage = 'purrview'` directly. Do not post a self-attestation comment claiming submission. The next consumer (`questPurrview.confirmSubmission`) refuses to act unless the SUBMIT lockphrase comment is present, and the only path that produces that comment is calling `questExecution.submit({questId})` from this skill book action. Bypass attempts are caught downstream and the quest is treated as untrusted — your work won't get reviewed.",
        "",
        "Submission is a code-enforced gate. The questExecution weapon refuses to advance the quest unless ALL of:",
        "  1. quest is in `execute` stage",
        "  2. items rows exist for the quest",
        "  3. every items row has `expectation` set (the declared spec, set at quest creation)",
        "  4. every items row has `url` set (no items still pending)",
        "  5. each item has ≥1 item_comments entry (your rationale: what it shows + why it satisfies the expectation)",
        "  6. each item.url responds with content > 0 bytes (no zero-byte placeholders, no 404s)",
        "",
        "On success, the gate writes a quest_comments row containing the SUBMIT lockphrase ('this quest now meets the criteria for purrview'). The Questmaster's `confirmSubmission` greps for that phrase before opening any screenshot — no phrase, no review.",
        "",
        "### Pre-submission steps (per deliverable spec entry)",
        "",
        "1. Produce the artifact and upload to Supabase Storage:",
        "   - Bucket: `GuildOS_Bucket`, path: `cursor_cloud/<questId>/<filename>`",
        "   - Use the `supabase_storage` weapon (`writeFile` to upload, `readPublicUrl` to get the URL).",
        "",
        "2. Upsert into `items` using the SAME `item_key` as the items row that was created with the quest:",
        "   ```javascript",
        "   import { writeDeliverable, writeItemComment } from '@/libs/quest';",
        "   const { data: item } = await writeDeliverable({",
        "     questId,",
        "     item_key: 'd1_route_response', // matches the items row's item_key",
        "     url: 'https://.../bucket/.../d1_route_response.png',",
        "     caption: 'what the screenshot actually shows',",
        "   });",
        "   ```",
        "   The upsert only writes the fields you pass — it will NOT overwrite the `expectation` that was set at quest creation. UNIQUE(quest_id, item_key) means resubmits replace in place. Do NOT invent `_v2` keys.",
        "",
        "3. Post a worker rationale comment on each item:",
        "   ```javascript",
        "   await writeItemComment(item.id, {",
        "     role: 'worker',",
        "     text: 'Screenshot shows /api/track responding with HTTP 200 and {inserted:8}. Matches accept_criteria.',",
        "   });",
        "   ```",
        "",
        "4. Call the gate:",
        "   ```javascript",
        "   import { submit } from '@/libs/weapon/questExecution';",
        "   const result = await submit({ questId });",
        "   if (!result.ok) {",
        "     // result.failed = ['<gate-id>'], result.report = { msg, fix, ...details }",
        "     // Read result.report.fix and address it. Do NOT retry without fixing.",
        "   }",
        "   ```",
        "",
        "If `submit` returns `ok: false`, follow `result.report.fix` verbatim. Do NOT attempt to write `stage='purrview'` directly — there's no other path that produces the lockphrase, and Cat will reject the quest on review.",
        "",
        "If submit fails 3 times in a row on a gate you can't resolve, escalate the quest with the failure report attached.",
      ].join("\n"),
    },
    summarizeComments: {
      description: "Compress old comments to prevent flooding. Keep latest 4, summarize the rest.",
      howTo: `
**When:** A quest has more than 10 comments.

**Steps:**
1. Fetch all comments for the quest, ordered by created_at ascending
2. Keep the latest 4 comments untouched
3. The 5th-from-last comment becomes the summary — update its text to a brief summary of everything before it
4. Delete all comments older than the 5th-from-last

**Result:** Quest has 5 comments — the oldest is the summary, then the 4 most recent. No new comments created.
`,
    },
    runDesignStage: {
      description: "Walk a quest through the design stage: for each items row, generate an AI baseline image from the expectation text, vision-judge it, retry up to 3x on fail. When all approved, advance to html_markup via questDesign.submit.",
      howTo: `
**When:** Quest is at \`stage='design'\`. This is the first stage in the new pipeline (design → html_markup → execute → ...). Local Claude (Guildmaster) drives this — Cat / cursor agents are not involved.

**Per-item loop (3-retry budget):**

1. **Read item context.**
   \`\`\`javascript
   import { searchDeliverables } from "@/libs/quest";
   const items = await searchDeliverables(questId);
   \`\`\`
   For each item where \`design_url\` is null OR \`design_approved !== true\`:

2. **Build the design prompt.** Use the expectation text + the per-repo design markdown at \`docs/design/\` to anchor visual identity. Pull the repo's design tokens (color, typography, density, tone) from \`docs/design/visual-identity.md\` if it exists.

3. **Generate via openai_images.generate.**
   \`\`\`javascript
   import { generate } from "@/libs/weapon/openai_images";
   const { images } = await generate({ prompt, size: "1536x1024", quality: "high", n: 1 });
   const b64 = images[0].b64_json;
   \`\`\`
   Save the bytes to Supabase Storage at \`quest-designs/<questId>/<item_key>.png\` and capture the public URL.

4. **Write the URL to items.design_url.**
   \`\`\`javascript
   import { writeDeliverable } from "@/libs/quest";
   await writeDeliverable({ questId, item_key, design_url, design_approved: null, design_feedback: null });
   \`\`\`

5. **Vision-judge the image vs the expectation.** Pass \`design_url\` + \`expectation\` to \`openai_images.judge\` (T1) AND a local-Claude multimodal Read (T3.5). Both must say "the image satisfies the claim."

6. **Write the verdict.** \`writeDeliverable({ questId, item_key, design_approved: true|false, design_feedback: '<one paragraph>' })\`. Also append a quest-comment summary so the user sees the iteration.

7. **Retry on fail.** If \`design_approved=false\`, regenerate with a tightened prompt that addresses the judge's feedback. Up to **3 total tries per item**. After 3 fails, halt the loop and surface the failed items in chat for the user to either edit the expectation or accept the closest design — do NOT advance.

**Advance:**
After every item passes, call \`questDesign.submit({ questId })\`. It re-checks the gate (every item: expectation + design_url + design_approved=true) and advances to \`html_markup\` with the lockphrase comment "this quest now meets the criteria for html_markup."

**Banned shortcuts:**
- Don't claim "approved" without actually running the judge.
- Don't write to \`design_approved\` without also writing \`design_feedback\`.
- Don't UPSERT items with new \`item_key\`s — that's invent-a-version, banned. Same key, replaced in place.
`,
    },
    runHtmlMarkupStage: {
      description: "Translate each AI-approved design baseline into a static-HTML proving-ground page under app/(town)/proving-ground/<questId>/<item_key>/page.js. When all items have html_markup_url, call questHtmlMarkup.submit; user reviews and approves on GM Desk.",
      howTo: `
**When:** Quest is at \`stage='html_markup'\` (questDesign.submit just advanced it). Local Claude (Guildmaster) drives this — same pattern as runDesignStage.

**Per-item loop:**

1. **Read item context** — pull \`design_url\` (the approved baseline) + \`expectation\` text + the per-repo \`docs/design/\` tokens.

2. **Write the page file.** Path: \`app/(town)/proving-ground/<questId>/<item_key>/page.js\`. Static React component, no live data. Mirror the design baseline image's layout, components, hierarchy. Use Tailwind v4 + DaisyUI v5 to match the GuildOS visual stack. Hard-code copy and stub data — this is a proofing-ground reference, not the real implementation.

3. **Self-judge.** After rendering (Next dev server picks the new file up automatically on save), screenshot the page via CIC and run \`openai_images.judge\` against the baseline image with claim = "the page implementation matches the baseline image at ≥90% similarity OR contains all key elements." Adjust the markup until pass.

4. **Write items.html_markup_url.** The route is the canonical \`/proving-ground/<questId>/<item_key>\` URL — no need to host elsewhere; GuildOS dev server (port 3002) and Vercel deploy both expose it.
   \`\`\`javascript
   await writeDeliverable({ questId, item_key, html_markup_url: \`/proving-ground/\${questId}/\${item_key}\` });
   \`\`\`

5. **After every item has html_markup_url:** call \`questHtmlMarkup.submit({ questId })\`. Stage stays at \`html_markup\` — the lockphrase tells the GM Desk that user review can begin. Present the user (in chat) with the design image + html_markup link for each item.

**User review:** GM Desk shows the html_markup section at the top. User clicks **Approve** → \`questHtmlMarkup.approve({ questId })\` advances to execute. Or **Bounce** with reason → \`questHtmlMarkup.bounce({ questId, reason, item_keys? })\` reverts to design and clears \`design_approved\` on the cited items.

**Cleanup:** proving-ground subpages are NOT auto-deleted. They live in the repo as committed files; clean via \`cleanupProvingGround\` when the user asks (manual prompt only — see that howTo).
`,
    },
    cleanupProvingGround: {
      description: "Manual-prompt-only cleanup of proving-ground subpages. Triggered by the user; never auto-fires.",
      howTo: `
**Trigger:** the user explicitly asks (e.g. "clean up proving ground" or "drop proving-ground for quest X"). Never auto-run on quest stage transitions — proving-ground pages are useful as historical references during purrview / review.

**Two modes:**

1. **Per-quest:** delete \`app/(town)/proving-ground/<questId>/\` recursively.

2. **Bulk by age:** list all proving-ground questId folders, look up each in DB, delete those whose \`stage='complete'\` AND \`updated_at < now() - olderThanDays\`.

**Steps:**
1. List \`app/(town)/proving-ground/\` subdirectories (each is a questId).
2. For each, decide delete vs keep based on the args.
3. \`rm -rf\` the matched folders.
4. Commit the deletions to main with message \`proving-ground: cleanup <N> quest folders (manual)\`.
5. Push.

**Banned:** never delete a folder while its quest is still in design / html_markup / execute / purrview / review. Markup is the canonical implementation reference at those stages — losing it derails the worker.
`,
    },
  },
};
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/auth_state/index.js -->
```js
/**
 * Auth State weapon — browser auth state management.
 *
 * Manages saved auth state (cookies/localStorage) exported by scripts/auth-capture.mjs.
 * Used by cloud agents that can't connect to the local CDP Chrome.
 * Default state file: playwright/.auth/user.json
 */

export const toc = {
  readState: "Read the Playwright auth state for a named service.",
  searchServices: "Search stored auth-state services by name.",
  writeState: "Write a Playwright auth state blob for a service.",
  readExpiryStatus: "Read the auth-state expiry status (fresh / stale).",
  deleteState: "Delete a stored auth state for a service.",
  uploadBundle:
    "Upload the local Playwright storageState file to Supabase Storage so cursor cloud agents (which can't reach the local filesystem) can pull it on init. Local-side action.",
  downloadBundle:
    "Download the latest auth-state bundle from Supabase Storage and write it to disk in the cursor agent's filesystem. Cursor-side action; pair with Playwright's storageState option.",
};
import { readFile, writeFile, stat } from "node:fs/promises";
import { join } from "node:path";

const DEFAULT_STATE_PATH = join(process.cwd(), "playwright/.auth/user.json");
const MAX_AGE_MS = 7 * 24 * 60 * 60 * 1000; // 7 days default expiry

// ---------------------------------------------------------------------------
// Read state
// ---------------------------------------------------------------------------

/**
 * Read the current auth state file.
 * @param {{ statePath?: string }} input
 * @returns {Promise<{ state: object|null, path: string, exists: boolean, age: number|null, expired: boolean }>}
 */
export async function readState({ statePath } = {}) {
  const path = statePath || DEFAULT_STATE_PATH;
  try {
    const stats = await stat(path);
    const age = Date.now() - stats.mtimeMs;
    const content = await readFile(path, "utf-8");
    const state = JSON.parse(content);
    const cookieCount = Array.isArray(state.cookies) ? state.cookies.length : 0;
    const originCount = Array.isArray(state.origins) ? state.origins.length : 0;

    return {
      state,
      path,
      exists: true,
      age,
      expired: age > MAX_AGE_MS,
      cookieCount,
      originCount,
      lastModified: stats.mtime.toISOString(),
    };
  } catch {
    return { state: null, path, exists: false, age: null, expired: true, cookieCount: 0, originCount: 0, lastModified: null };
  }
}

/**
 * Check which services have active cookies in the auth state.
 * @param {{ statePath?: string }} input
 * @returns {Promise<{ services: Array<{ domain: string, cookieCount: number, hasExpired: boolean }> }>}
 */
export async function searchServices({ statePath } = {}) {
  const { state } = await readState({ statePath });
  if (!state || !Array.isArray(state.cookies)) return { services: [] };

  const byDomain = new Map();
  const now = Date.now() / 1000;

  for (const cookie of state.cookies) {
    const domain = (cookie.domain || "").replace(/^\./, "");
    if (!domain) continue;
    const entry = byDomain.get(domain) || { domain, cookieCount: 0, hasExpired: false };
    entry.cookieCount++;
    if (cookie.expires && cookie.expires > 0 && cookie.expires < now) {
      entry.hasExpired = true;
    }
    byDomain.set(domain, entry);
  }

  return { services: Array.from(byDomain.values()).sort((a, b) => b.cookieCount - a.cookieCount) };
}

// ---------------------------------------------------------------------------
// Write state
// ---------------------------------------------------------------------------

/**
 * Save auth state to file.
 * @param {{ state: object, statePath?: string }} input
 */
export async function writeState({ state, statePath } = {}) {
  if (!state) throw new Error("state object is required");
  const path = statePath || DEFAULT_STATE_PATH;
  const { mkdir } = await import("node:fs/promises");
  const { dirname } = await import("node:path");
  await mkdir(dirname(path), { recursive: true });
  await writeFile(path, JSON.stringify(state, null, 2), "utf-8");
  return { ok: true, path };
}

// ---------------------------------------------------------------------------
// Expiry check
// ---------------------------------------------------------------------------

/**
 * Check if auth state needs refresh (expired cookies or stale file).
 * @param {{ statePath?: string, maxAgeMs?: number }} input
 * @returns {Promise<{ needsRefresh: boolean, reason: string|null, expiredDomains: string[] }>}
 */
export async function readExpiryStatus({ statePath, maxAgeMs } = {}) {
  const maxAge = maxAgeMs || MAX_AGE_MS;
  const { state, exists, age, expired } = await readState({ statePath });

  if (!exists) return { needsRefresh: true, reason: "State file does not exist", expiredDomains: [] };
  if (age > maxAge) return { needsRefresh: true, reason: `State file is ${Math.round(age / 3600000)}h old (max: ${Math.round(maxAge / 3600000)}h)`, expiredDomains: [] };

  // Check individual cookie expiry
  const now = Date.now() / 1000;
  const expiredDomains = new Set();
  if (state && Array.isArray(state.cookies)) {
    for (const cookie of state.cookies) {
      if (cookie.expires && cookie.expires > 0 && cookie.expires < now) {
        expiredDomains.add((cookie.domain || "").replace(/^\./, ""));
      }
    }
  }

  if (expiredDomains.size > 0) {
    return {
      needsRefresh: true,
      reason: `Expired cookies for: ${Array.from(expiredDomains).join(", ")}`,
      expiredDomains: Array.from(expiredDomains),
    };
  }

  return { needsRefresh: false, reason: null, expiredDomains: [] };
}

// ---------------------------------------------------------------------------
// Delete (clear state)
// ---------------------------------------------------------------------------

/**
 * Delete the auth state file.
 * @param {{ statePath?: string }} input
 */
export async function deleteState({ statePath } = {}) {
  const path = statePath || DEFAULT_STATE_PATH;
  const { unlink } = await import("node:fs/promises");
  try {
    await unlink(path);
    return { ok: true, deleted: path };
  } catch {
    return { ok: true, deleted: null, msg: "File did not exist" };
  }
}

// ---------------------------------------------------------------------------
// Distribution: local → Supabase Storage → cursor cloud agents
//
// The local user runs `auth-capture.mjs` to log into services interactively
// and produce `playwright/.auth/user.json`. Cursor agents can't reach the local
// filesystem, so we put the bundle in a private Supabase bucket location keyed
// by ownerId and let the agents pull it on init via downloadBundle.
// ---------------------------------------------------------------------------

const BUNDLE_BUCKET = "GuildOS_Bucket";
const BUNDLE_PATH = (ownerId) => `auth_bundles/${ownerId}/storageState.json`;

/**
 * Upload the local storageState JSON to Supabase Storage so cloud agents can
 * pull it. Local-side action (run from the user's machine after auth-capture).
 *
 * @param {{ ownerId: string, statePath?: string }} input
 * @returns {Promise<{ ok: true, path: string, sizeBytes: number, lastModified: string }>}
 */
export async function uploadBundle({ ownerId, statePath } = {}) {
  if (!ownerId) throw new Error("ownerId is required");
  const local = statePath || DEFAULT_STATE_PATH;
  const buf = await readFile(local);
  const stats = await stat(local);

  // Use the supabase_storage weapon — same convention as quest items.
  const { writeFile: writeStorageFile } = await import("../supabase_storage/index.js");
  const path = BUNDLE_PATH(ownerId);
  await writeStorageFile({
    path,
    content: buf,
    contentType: "application/json",
    bucket: BUNDLE_BUCKET,
    upsert: true,
  });
  return {
    ok: true,
    path,
    sizeBytes: buf.length,
    lastModified: stats.mtime.toISOString(),
  };
}

/**
 * Download the auth bundle for the given ownerId from Supabase Storage and
 * write it to local disk. Cursor-side action — call once during agent init,
 * then point Playwright at the resulting path via `storageState`.
 *
 * @param {{ ownerId: string, statePath?: string }} input
 * @returns {Promise<{ ok: true, path: string, sizeBytes: number }>}
 */
export async function downloadBundle({ ownerId, statePath } = {}) {
  if (!ownerId) throw new Error("ownerId is required");
  const dest = statePath || DEFAULT_STATE_PATH;

  const { readFile: readStorageFile } = await import("../supabase_storage/index.js");
  const path = BUNDLE_PATH(ownerId);
  const { content } = await readStorageFile({ path, bucket: BUNDLE_BUCKET });
  if (!content) {
    return { ok: false, msg: `No auth bundle at ${path}. Have the user run scripts/auth-capture.mjs --upload.` };
  }
  // Buffer or string both fine; ensure dir exists then write.
  const { mkdir, writeFile: writeFsFile } = await import("node:fs/promises");
  const { dirname: pathDirname } = await import("node:path");
  await mkdir(pathDirname(dest), { recursive: true });
  const buf = Buffer.isBuffer(content) ? content : Buffer.from(content);
  await writeFsFile(dest, buf);
  return { ok: true, path: dest, sizeBytes: buf.length };
}

// ---------------------------------------------------------------------------
// Credential check
// ---------------------------------------------------------------------------

export async function checkCredentials() {
  const { exists, expired, cookieCount, lastModified } = await readState();
  if (!exists) {
    return { ok: false, msg: "No auth state JSON — run scripts/auth-capture.mjs to create one." };
  }
  if (expired) {
    return { ok: false, msg: `Auth state expired (last modified: ${lastModified}). Re-run scripts/auth-capture.mjs.` };
  }
  return { ok: true, msg: `Auth state valid — ${cookieCount} cookies, last modified: ${lastModified}` };
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/cursor/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/weapon/cursor`

Rationale layer (per CLAUDE.md §6). The WHY behind Cursor dispatch + model policy.
Append; supersede in place when a decision changes.

---

## 2026-06-24 — Cursor is a CHEAP worker (Composer only); NEVER a bridge for Claude. Pin the model, never omit it. (operator directive, LOCKED)

**The decision.** We hold a **Claude 20X Max** subscription. Therefore:
- ALL Claude work runs on that flat-rate subscription via `claudecli` (OAuth login,
  `ANTHROPIC_API_KEY` stripped so it uses the Max plan, NOT API credits).
- Cursor is used ONLY as a **cheap worker on its own in-house Composer model**
  (`composer-2.5`). It is a *cheaper-than-Claude* second engine, valued for Composer.
- We **NEVER route a premium frontier model (claude-\*, gpt-\*, gemini-\*, grok-\*)
  THROUGH Cursor.** Doing so ("using Cursor as a bridge for Claude") both (a) bills
  Cursor's metered API-model pool — Pro's "$200 API model usage" — and (b) double-pays
  for Claude we already hold flat-rate. It is the worst of both billing worlds.

**The incident this fixes (2026-06-24).** A process burned **$200** of Cursor's
metered API-model allowance in one cycle by running premium models through Cursor.
Root cause = every cloud-agent creation path **OMITTED the `model` field**, and the
prior LOCKED guidance (`cursorCJGEO` skill book, "OMIT the model; Cursor picks its
current default") actively *taught* that omission. Cursor's configured default for
background agents is a **premium frontier model** (the recommended set from
`GET /v0/models` is `composer-2.5` + `claude-opus-4-8-*`, `gpt-5.5-*`, `gpt-5.3-codex-*`
— only `composer-2.5` is cheap). So every omitted-model dispatch silently ran premium.
The `nudgeCICPolice` cron heartbeat (`writeFollowup` every ~10 min, which makes the
agent RUN on its creation model) kept a premium agent burning around the clock.

**The fix (structural — make the violation impossible, not merely discouraged).**
1. `libs/weapon/cursor/model.js` — single source of truth: `CURSOR_CHEAP_MODEL`
   (`composer-2.5`) + `resolveCheapCursorModel(id)` which returns the cheap default
   when none is given and **THROWS on any non-`composer-*` id** (including `auto`,
   which can route premium) unless `CURSOR_ALLOW_PREMIUM=1` (the audited operator
   escape hatch).
2. Every creation surface now PINS the model via that guard, shape
   `model: { id: "composer-2.5" }` (a `ModelRef` OBJECT per the cloud-agents OpenAPI —
   NOT a top-level string): `cursor.writeAgent`, the local proxy
   `app/api/cloud_agents/route.js` `cursor_setup`, and the `cursorcli` weapon
   (local CLI `--model`).
3. The `cursorCJGEO` "OMIT model" LOCKED note is **superseded** in place — it was the
   discovery-layer substrate that propagated the bug.

**Why pin at CREATION specifically.** A Cursor agent's model is fixed when the agent
is created (`POST /v0/agents`); `followup` calls inherit it and take no model field. So
pinning `writeAgent`/`cursor_setup` is the load-bearing fix — it makes every NEW agent
(and every cron auto-respawn, which goes through `writeAgent`) cheap. Existing agents
created BEFORE this fix keep their premium model until they expire and are recreated.

**Rejected alternatives.**
- *Omit the model and trust Cursor's default* (the prior rule) — that default is
  premium; the omission was the bug.
- *Pass `model` as a top-level string* (the original `composer-2.0` that 400'd) — wrong
  shape AND stale id. The 400 was a shape/staleness bug, never a reason to omit; the
  correct shape is the `{ id }` ModelRef.
- *A doc note saying "use cheap models"* — discouragement is not enforcement; a future
  dispatch would omit the model again. The `resolveCheapCursorModel` throw is the wall.
- *`--model auto` for the CLI* (cursorcli's first default) — `auto` can route to a
  premium frontier model, so it violates "cheap only." Blocked by the guard.

**Consequence if violated.** Re-introduce an omitted/`auto`/premium model on any Cursor
dispatch and the metered API pool burns again (the exact $200 incident), while we keep
paying flat-rate for the same Claude capacity on Max — paying twice to go slower-and-
more-expensive than just using `claudecli`.

**Open follow-ups (NOT a code gap — operational).**
- *Existing premium cloud-agent sessions* (the 8 `cursor_cloud` adventurers created
  pre-fix) still carry a premium creation-model. They are currently halted by the
  account's exhausted budget. They auto-heal to `composer-2.5` when recreated (creation
  is now pinned), but until then the CIC-Police heartbeat would resume premium runs once
  budget returns. Retire/recreate them cheap after the budget resets (or set a spend
  limit) — surfaced to the operator rather than nulling live session handles autonomously.
- *Live re-verify the cloud `model:{id}` shape* once budget allows a real
  `POST /v0/agents` (the shape is OpenAPI-confirmed but could not be live-tested while
  the account was out of usage).
- *Account identity*: the headless `CURSOR_API_KEY` (key name "GuildOS") resolves to
  `<REDACTED-EMAIL>` via `GET /v0/me` — the user's own account, confirmed not a
  wrong-account artifact.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/cursor/index.js -->
```js
/**
 * Cursor weapon — Cursor Cloud Agent dispatch and management via Cursor API.
 *
 * Auth: CURSOR_API_KEY from profiles.env_vars or process.env.
 * Endpoint: https://api.cursor.com/v0/agents
 * Model: PINNED to Cursor's cheap Composer model via resolveCheapCursorModel
 *   (model.js). NEVER omit it — omitting makes Cursor pick its PREMIUM frontier
 *   default (a premium claude or gpt model), which bills the metered API pool and
 *   double-pays for Claude we already hold flat-rate on Claude Max. Burned $200 in one
 *   cycle (2026-06-24). The correct body shape is `model: { id: "composer-2.5" }`
 *   (ModelRef per the cloud-agents OpenAPI), NOT a top-level string (the stale
 *   `composer-2.0` string was the old HTTP-400; that note is now superseded).
 */

export const toc = {
  readAgent: "Read a Cursor cloud agent's status and metadata.",
  readConversation: "Read a Cursor agent's full conversation history.",
  writeFollowup: "Send a followup message to a Cursor agent (push-based dispatch).",
  searchAgents: "Search Cursor agents (wraps readAgent; API has no list endpoint).",
  readQueuedMessages: "Read queued user messages sent after the last assistant reply.",
  writeAgent: "Create a new Cursor cloud agent for a given repository.",
  readEnvSetupInstructions: "Generate an env setup script to bootstrap a fresh Cursor agent session.",
  syncSessionStatus: "Probe an adventurer's upstream Cursor session status and reconcile the adventurers DB row. Returns { upstream_status, alive, dispatch_safe, was_drift }.",
  // --- Agent-facing actions (skillActionOk/Err envelope; (userId, input) signature) ---
  dispatchTask: "Agent-facing: send a task to a Cursor cloud agent via followup. Input { agentId, instructions }. A followup runs the agent on its CREATION model (pinned cheap by writeAgent/cursor_setup) — the model input is inert; choose the model at creation, not here.",
  readStatus: "Agent-facing: check a Cursor cloud agent's status. Input { agentId }.",
  readConversationAction: "Agent-facing: read a Cursor agent's full conversation. Input { agentId }.",
  dispatchPptGeneration: "Agent-facing: dispatch a PowerPoint-generation task to a Cursor cloud agent. Input { agentId, topic, slides?, questId? }.",
};
import { database } from "@/libs/council/database";
import { guildosCredentialsSetupBlock } from "./setupBlock.js";
import { resolveCheapCursorModel } from "./model.js";
import { skillActionOk, skillActionErr } from "@/libs/skill_book/actionResult.js";
import { getAdventurerExecutionContext } from "@/libs/adventurer/advance.js";

const API_BASE = "https://api.cursor.com/v0/agents";

// ---------------------------------------------------------------------------
// Auth
// ---------------------------------------------------------------------------

async function getApiKey(userId) {
  if (process.env.CURSOR_API_KEY) return process.env.CURSOR_API_KEY;
  if (!userId) return null;
  const db = await database.init("service");
  const { data } = await db
    .from("profiles")
    .select("env_vars")
    .eq("id", userId)
    .single();
  return data?.env_vars?.CURSOR_API_KEY ?? null;
}

function makeAuthHeader(apiKey) {
  return `Basic ${Buffer.from(`${apiKey}:`).toString("base64")}`;
}

async function cursorFetch(path, opts = {}, userId) {
  const key = await getApiKey(userId);
  if (!key) throw new Error("Missing CURSOR_API_KEY");
  const url = path.startsWith("http") ? path : `${API_BASE}${path}`;
  const res = await fetch(url, {
    ...opts,
    headers: {
      Authorization: makeAuthHeader(key),
      "Content-Type": "application/json",
      ...opts.headers,
    },
  });
  if (!res.ok) {
    const body = await res.text();
    throw new Error(`Cursor API ${res.status}: ${body.slice(0, 500)}`);
  }
  return res.json();
}

// ---------------------------------------------------------------------------
// Agent management
// ---------------------------------------------------------------------------

/**
 * Read agent status.
 * @param {{ agentId: string }} input
 * @param {string} [userId]
 */
export async function readAgent({ agentId } = {}, userId) {
  if (!agentId) throw new Error("agentId is required");
  return cursorFetch(`/${agentId}`, {}, userId);
}

/**
 * Read agent conversation history.
 * @param {{ agentId: string }} input
 * @param {string} [userId]
 */
export async function readConversation({ agentId } = {}, userId) {
  if (!agentId) throw new Error("agentId is required");
  return cursorFetch(`/${agentId}/conversation`, {}, userId);
}

/**
 * Send a followup message to an agent (push-based dispatch).
 * @param {{ agentId: string, message: string, model?: string }} input
 * @param {string} [userId]
 */
export async function writeFollowup({ agentId, message, model } = {}, userId) {
  if (!agentId) throw new Error("agentId is required");
  if (!message) throw new Error("message is required");
  return cursorFetch(
    `/${agentId}/followup`,
    {
      method: "POST",
      body: JSON.stringify({
        prompt: { text: message },
      }),
    },
    userId,
  );
}

/**
 * Search for agents by partial ID or metadata.
 * Note: Cursor API doesn't have a list endpoint — this reads a known agent.
 * For now, wraps readAgent. Extend when API adds list support.
 * @param {{ agentId: string }} input
 * @param {string} [userId]
 */
export async function searchAgents({ agentId } = {}, userId) {
  if (!agentId) throw new Error("agentId is required for search (Cursor API has no list endpoint)");
  const agent = await readAgent({ agentId }, userId);
  return { agents: [agent] };
}

/**
 * Read queued (unprocessed) messages — user messages sent after the last assistant response.
 * @param {{ agentId: string }} input
 * @param {string} [userId]
 */
export async function readQueuedMessages({ agentId } = {}, userId) {
  if (!agentId) throw new Error("agentId is required");
  const conv = await readConversation({ agentId }, userId);
  const msgs = conv?.messages || [];
  const lastAssistantIdx = msgs.findLastIndex((m) => m.type === "assistant_message");
  return { queued: msgs.slice(lastAssistantIdx + 1).filter((m) => m.type === "user_message") };
}

/**
 * Create a new Cursor cloud agent.
 * @param {{ repository: string, ref: string, prompt?: string }} input
 * @param {string} [userId]
 */
/**
 * Spawn a new Cursor cloud agent. ALWAYS provisions GuildOS service-role
 * credentials in the spawn prompt as the first action, so the agent never
 * hits the documented "missing GuildOS Supabase credentials" blocker that
 * caused the 2026-04-26 ptglab smoke-test to escalate without bookkeeping.
 *
 * Contract: this is the ONLY supported path for creating an agent. Direct
 * POSTs to the Cursor API that skip the credentials-injection block are a
 * contract violation — agents spawned that way cannot complete GuildOS-
 * side bookkeeping (items rows, quest_comments, submitForPurrview).
 *
 * Credential source: process.env (the Guildmaster running this weapon must
 * have GuildOS env loaded — `--env-file=.env.local` or equivalent).
 *
 * @param {{ repository: string, ref: string, prompt?: string,
 *           skipCredentialsInjection?: boolean }} input
 * @param {string} [userId]
 */
export async function writeAgent({ repository, ref, prompt, skipCredentialsInjection = false, model } = {}, userId) {
  if (!repository) throw new Error("repository is required");
  if (!ref) throw new Error("ref is required");
  const key = await getApiKey(userId);
  if (!key) throw new Error("Missing CURSOR_API_KEY");
  // Cheap-worker policy: pin Cursor's Composer model (premium throws unless
  // CURSOR_ALLOW_PREMIUM=1). NEVER omit — omission = Cursor's premium default.
  const modelId = resolveCheapCursorModel(model);

  let finalPrompt = prompt || "Wait for initialization instructions.";

  if (!skipCredentialsInjection) {
    // Prefer the GUILDOS_-prefixed names so the GuildOS project is unambiguous
    // even when the orchestrator's env is also wired to a project's own
    // Supabase. Falls back to the basic names for backward compat.
    const guildosUrl =
      process.env.GUILDOS_NEXT_PUBLIC_SUPABASE_URL ||
      process.env.NEXT_PUBLIC_SUPABASE_URL;
    const guildosKey =
      process.env.GUILDOS_SUPABASE_SECRETE_KEY ||
      process.env.GUILDOS_SUPABASE_SECRET_KEY ||
      process.env.SUPABASE_SECRETE_KEY ||
      process.env.SUPABASE_SECRET_KEY;
    if (!guildosUrl || !guildosKey) {
      throw new Error(
        "writeAgent: GuildOS credentials missing from process.env. Required: GUILDOS_NEXT_PUBLIC_SUPABASE_URL + GUILDOS_SUPABASE_SECRETE_KEY (or the basic NEXT_PUBLIC_SUPABASE_URL + SUPABASE_SECRETE_KEY for backward compat). The agent cannot do GuildOS bookkeeping without these. Run with `--env-file=.env.local` or pass `skipCredentialsInjection: true` if you genuinely want a creds-less agent (rare — almost always wrong)."
      );
    }
    finalPrompt = guildosCredentialsSetupBlock({ supabaseUrl: guildosUrl, supabaseKey: guildosKey }) + finalPrompt;
  }

  const res = await fetch(API_BASE, {
    method: "POST",
    headers: {
      Authorization: makeAuthHeader(key),
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      prompt: { text: finalPrompt },
      source: { repository, ref },
      model: { id: modelId },
    }),
  });
  if (!res.ok) {
    const body = await res.text();
    throw new Error(`Cursor API ${res.status}: ${body.slice(0, 500)}`);
  }
  return res.json();
}

// ---------------------------------------------------------------------------
// Session-health probe — pre-dispatch check that the upstream Cursor session
// matches what the adventurers DB row says, and is in a state that can accept
// a followup. Reconciles drift (DB says idle, upstream says EXPIRED, etc.).
// ---------------------------------------------------------------------------

/**
 * Probe one adventurer's session against the Cursor API and reconcile the DB.
 *
 * Returns:
 *   {
 *     upstream_status: 'RUNNING' | 'FINISHED' | 'EXPIRED' | 'CREATING' | 'ERROR' | <other>,
 *     alive: boolean,            // upstream session exists at all
 *     dispatch_safe: boolean,    // followup is expected to land (FINISHED/RUNNING/CREATING ok; EXPIRED not)
 *     was_drift: boolean,        // db.session_status differed from upstream
 *     adventurer: { id, name, session_id, session_status }, // post-sync row
 *   }
 *
 * Drift handling: if upstream returns a status the local DB doesn't reflect
 * accurately, this updates `adventurers.session_status` to a normalized
 * "idle" (dispatch-safe) or "expired" (needs respawn) value. Does NOT auto-
 * respawn — caller decides.
 *
 * @param {{ adventurerId?: string, adventurerName?: string }} input — pass one
 * @param {string} [userId]
 */
export async function syncSessionStatus({ adventurerId, adventurerName } = {}, userId) {
  if (!adventurerId && !adventurerName) {
    throw new Error("syncSessionStatus: pass adventurerId or adventurerName");
  }
  const db = await database.init("service");
  const q = db.from("adventurers").select("id, name, session_id, session_status, worker_type").limit(1);
  const { data: rows } = await (adventurerId ? q.eq("id", adventurerId) : q.eq("name", adventurerName));
  const adv = rows?.[0];
  if (!adv) throw new Error(`syncSessionStatus: adventurer not found (${adventurerId || adventurerName})`);
  if (!adv.session_id) {
    return {
      upstream_status: "no_session",
      alive: false,
      dispatch_safe: false,
      was_drift: false,
      adventurer: adv,
    };
  }

  let upstreamStatus = "unknown";
  let alive = false;
  try {
    const j = await cursorFetch(`/${adv.session_id}`, {}, userId);
    upstreamStatus = String(j?.status || "unknown");
    alive = true;
  } catch (err) {
    // Cursor API returns 404 / 410 for hard-deleted sessions; surface that
    upstreamStatus = "deleted_or_unreachable";
    alive = false;
  }

  // Normalize to DB-friendly status. dispatch_safe: can a followup land?
  const dispatch_safe =
    alive && (upstreamStatus === "RUNNING" || upstreamStatus === "FINISHED" || upstreamStatus === "CREATING");
  const desiredDbStatus = dispatch_safe
    ? "idle"
    : upstreamStatus === "EXPIRED"
      ? "expired"
      : "unreachable";

  const was_drift = adv.session_status !== desiredDbStatus;
  if (was_drift) {
    await db.from("adventurers").update({ session_status: desiredDbStatus }).eq("id", adv.id);
  }

  return {
    upstream_status: upstreamStatus,
    alive,
    dispatch_safe,
    was_drift,
    adventurer: { ...adv, session_status: desiredDbStatus },
  };
}

// ---------------------------------------------------------------------------
// Environment setup helper (for fresh agent sessions)
// ---------------------------------------------------------------------------

/**
 * Generate the env setup instructions for a fresh Cursor agent session.
 * Reads required env vars from profiles and formats them for the agent.
 * @param {{ userId: string, envVarKeys?: string[] }} input
 */
export async function readEnvSetupInstructions({ userId, envVarKeys } = {}) {
  if (!userId) throw new Error("userId is required");
  const db = await database.init("service");
  const { data } = await db
    .from("profiles")
    .select("env_vars")
    .eq("id", userId)
    .single();

  const ev = data?.env_vars || {};
  const keys = Array.isArray(envVarKeys) && envVarKeys.length > 0
    ? envVarKeys
    : [
        "NEXT_PUBLIC_SUPABASE_URL",
        "NEXT_PUBLIC_SUPABASE_ANON_KEY",
        "SUPABASE_SECRETE_KEY",
        "OPENAI_API_KEY",
        "PIGEON_API_KEY",
        "PIGEON_POST_OWNER_ID",
      ];

  const lines = [];
  for (const k of keys) {
    const val = ev[k] || process.env[k] || "";
    if (val) lines.push(`${k}=${val}`);
  }

  return {
    envFileContent: lines.join("\n"),
    setupScript: [
      "npm install && npx playwright install chromium --with-deps && npm install -g @anthropic-ai/claude-code",
      `cat > .env.local << 'ENVEOF'\n${lines.join("\n")}\nENVEOF`,
    ].join("\n"),
  };
}

// ---------------------------------------------------------------------------
// Credential check
// ---------------------------------------------------------------------------

export async function checkCredentials(userId) {
  try {
    const key = await getApiKey(userId);
    if (key) {
      return { ok: true, msg: "CURSOR_API_KEY is set" };
    }
    return {
      ok: false,
      msg: "Missing CURSOR_API_KEY — add it to your profile env_vars in Council Hall > Formulary.",
    };
  } catch (e) {
    return { ok: false, msg: `Error: ${e.message}` };
  }
}

// ---------------------------------------------------------------------------
// Agent-facing wrappers — (userId, input) signature, skillActionOk/Err envelope.
// These are the methods the `cursor` skill book's toc points at. They were moved
// out of libs/skill_book/cursor/index.js so the skill book stays a pure prompt
// registry. The underlying logic delegates to the weapon functions above.
// ---------------------------------------------------------------------------

function resolveUserId(explicitUserId) {
  const ctx = getAdventurerExecutionContext();
  return explicitUserId || ctx?.userId || null;
}

/**
 * Send a task to a Cursor cloud agent via followup message.
 * NOTE: a followup runs the agent on the model it was CREATED with (the API followup
 * body takes no model field), so the `model` input here is inert — model selection
 * happens at creation (writeAgent / cursor_setup), which is PINNED cheap. If the agent
 * was created cheap (post-2026-06-24), the followup runs cheap; a pre-fix premium agent
 * stays premium until recreated. See libs/weapon/cursor/model.js + engineering_decisions.md.
 * @param {string} [_userId]
 * @param {{ agentId?: string, instructions?: string, model?: string }} input
 */
export async function dispatchTask(_userId, input) {
  const inObj = typeof input === "object" && input ? input : {};
  const agentId = String(inObj.agentId || "").trim();
  const instructions = String(inObj.instructions || "").trim();
  const model = String(inObj.model || "").trim() || undefined;

  if (!agentId) return skillActionErr("agentId is required");
  if (!instructions) return skillActionErr("instructions are required");

  const userId = resolveUserId(_userId);
  try {
    const result = await writeFollowup({ agentId, message: instructions, model }, userId);
    return skillActionOk({
      items: {
        result: JSON.stringify(result),
        agentId,
      },
      msg: `Task dispatched to agent ${agentId}`,
    });
  } catch (e) {
    return skillActionErr(`Dispatch failed: ${e.message}`);
  }
}

/**
 * Check the current status of a Cursor cloud agent.
 * @param {string} [_userId]
 * @param {{ agentId?: string }} input
 */
export async function readStatus(_userId, input) {
  const inObj = typeof input === "object" && input ? input : {};
  const agentId = String(inObj.agentId || "").trim();
  if (!agentId) return skillActionErr("agentId is required");

  const userId = resolveUserId(_userId);
  try {
    const agent = await readAgent({ agentId }, userId);
    return skillActionOk({
      items: {
        status: JSON.stringify(agent.status || "unknown"),
        agent: JSON.stringify(agent),
      },
      msg: `Agent ${agentId} status: ${agent.status || "unknown"}`,
    });
  } catch (e) {
    return skillActionErr(`Status check failed: ${e.message}`);
  }
}

/**
 * Read the full conversation history of a Cursor cloud agent.
 * (Named *Action to avoid clashing with the lower-level `readConversation` API wrapper.)
 * @param {string} [_userId]
 * @param {{ agentId?: string }} input
 */
export async function readConversationAction(_userId, input) {
  const inObj = typeof input === "object" && input ? input : {};
  const agentId = String(inObj.agentId || "").trim();
  if (!agentId) return skillActionErr("agentId is required");

  const userId = resolveUserId(_userId);
  try {
    const conv = await readConversation({ agentId }, userId);
    return skillActionOk({
      items: { conversation: JSON.stringify(conv) },
      msg: `Retrieved conversation for agent ${agentId}`,
    });
  } catch (e) {
    return skillActionErr(`Conversation read failed: ${e.message}`);
  }
}

/**
 * Dispatch a PowerPoint-generation task to a Cursor cloud agent. The agent
 * builds the .pptx and uploads it to Supabase Storage under cursor_cloud/.
 * @param {string} [_userId]
 * @param {{ agentId?: string, topic?: string, slides?: string, questId?: string }} input
 */
export async function dispatchPptGeneration(_userId, input) {
  const inObj = typeof input === "object" && input ? input : {};
  const agentId = String(inObj.agentId || "").trim();
  const topic = String(inObj.topic || "").trim();
  const slides = String(inObj.slides || "").trim();
  const questId = String(inObj.questId || "").trim();

  if (!agentId) return skillActionErr("agentId is required");
  if (!topic) return skillActionErr("topic is required");

  const userId = resolveUserId(_userId);
  const instructions = [
    `Generate a PowerPoint presentation about: ${topic}`,
    slides ? `\nSlide content:\n${slides}` : "",
    "",
    "Requirements:",
    "- Use pptxgenjs or python-pptx to create the PPT file",
    "- Upload the final .pptx to Supabase Storage at:",
    questId
      ? `  cursor_cloud/${questId}/${topic.replace(/[^a-zA-Z0-9]/g, "-").toLowerCase()}.pptx`
      : `  cursor_cloud/general/${topic.replace(/[^a-zA-Z0-9]/g, "-").toLowerCase()}.pptx`,
    "- Post the public URL as a quest comment",
    "- Take a screenshot of at least one slide as visual proof",
    "",
    "After generating, git push your changes.",
  ].join("\n");

  try {
    const result = await writeFollowup({ agentId, message: instructions }, userId);
    return skillActionOk({
      items: {
        result: JSON.stringify(result),
        agentId,
      },
      msg: `PPT generation dispatched to agent ${agentId}: "${topic}"`,
    });
  } catch (e) {
    return skillActionErr(`PPT dispatch failed: ${e.message}`);
  }
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/cursor/model.js -->
```js
/**
 * Cursor model policy — the single source of truth for "Cursor = cheap worker only".
 * Pure module, NO imports, so it is consumable identically from the Next runtime, a
 * Next API route, and a plain-node leaf (cursorcli) without adapter shims.
 *
 * DECISION (2026-06-24, operator directive): we hold a Claude 20X Max subscription,
 * so ALL Claude work runs on that flat-rate subscription via claudecli (OAuth, no API
 * credits). Cursor is used ONLY as a CHEAP worker on its own in-house Composer model.
 * We NEVER route a premium frontier model (Claude/GPT/Gemini/Grok) THROUGH Cursor —
 * Cursor bills those against its metered API-model pool ("$200 API model usage") AND
 * it double-pays for Claude we already have flat-rate ("using Cursor as a bridge for
 * Claude"). Omitting the model on a Cursor dispatch makes Cursor pick its PREMIUM
 * default — that omission is what burned $200 in one cycle (the bug this enforces away).
 * Full rationale: libs/weapon/cursor/engineering_decisions.md.
 */

/** Cursor's cheap in-house worker model id (from GET /v0/models — the only non-premium
 *  entry in the recommended set). Env-overridable, but it must stay a composer-* id. */
export const CURSOR_CHEAP_MODEL = process.env.CURSOR_CHEAP_MODEL || "composer-2.5";

/** Premium = anything that is NOT Cursor's own composer-* model. Everything else in
 *  Cursor's recommended list (claude-*, gpt-*, gemini-*, grok-*, and "auto", which may
 *  route to a frontier model) bills the metered API pool. */
export function isPremiumCursorModel(id) {
  const s = String(id || "").trim().toLowerCase();
  if (!s) return false; // empty/omitted → the caller substitutes the cheap default
  return !s.startsWith("composer");
}

/**
 * Resolve + ENFORCE the model for any Cursor dispatch. Returns the cheap default when
 * none is given; THROWS on a premium id unless `CURSOR_ALLOW_PREMIUM=1` is set — the
 * explicit, auditable operator escape hatch for a deliberate one-off. This is the
 * structural guard that makes "Cursor never bridges to Claude" impossible to violate by
 * accident (an omitted/defaulted/auto model can no longer silently land on a frontier model).
 *
 * @param {string} [id] requested model id (may be undefined/empty)
 * @returns {string} a composer-* model id (or the premium id if the escape hatch is set)
 */
export function resolveCheapCursorModel(id) {
  const chosen = (id && String(id).trim()) || CURSOR_CHEAP_MODEL;
  if (isPremiumCursorModel(chosen) && process.env.CURSOR_ALLOW_PREMIUM !== "1") {
    throw new Error(
      `Cursor model "${chosen}" is a premium (metered) model. POLICY: Cursor runs ONLY its cheap Composer ` +
        `model; Claude work goes through Claude Max (claudecli). Pass a composer-* id, set CURSOR_CHEAP_MODEL, ` +
        `or — for a deliberate, audited one-off — set CURSOR_ALLOW_PREMIUM=1. ` +
        `See libs/weapon/cursor/engineering_decisions.md.`,
    );
  }
  return chosen;
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/cursor/setupBlock.js -->
```js
/**
 * Pure-JS module: builds the GuildOS-credentials shell-setup block that
 * `cursor.writeAgent` prepends to every spawn prompt.
 *
 * Extracted so node scripts (which can't resolve `@/libs/...` Next path
 * aliases) and the Next-runtime weapon both consume the same source.
 * No imports — keep it that way so this module is consumable from any
 * runtime without adapter shims.
 */

/**
 * @param {{ supabaseUrl: string, supabaseKey: string }} input
 * @returns {string} shell-setup block to prepend to a Cursor agent's spawn prompt
 */
export function guildosCredentialsSetupBlock({ supabaseUrl, supabaseKey } = {}) {
  if (!supabaseUrl || !supabaseKey) {
    throw new Error(
      "guildosCredentialsSetupBlock: supabaseUrl + supabaseKey are required. " +
      "Caller is responsible for sourcing them from process.env (NEXT_PUBLIC_SUPABASE_URL + SUPABASE_SECRETE_KEY) " +
      "or profiles.env_vars before invoking the spawn."
    );
  }
  return [
    "═════════════════════════════════════════════════════════════════════",
    "FIRST ACTION — provision GuildOS credentials (do this BEFORE anything else)",
    "═════════════════════════════════════════════════════════════════════",
    "",
    "Set these env vars so GuildOS bookkeeping works regardless of which",
    "project repo you're working on. Both the GUILDOS_-prefixed names",
    "(canonical when working in multi-repo envs) AND the basic names",
    "(backward compat) are exported, so libs that read either form work.",
    "",
    "```bash",
    "mkdir -p ~/guildos",
    "cat > ~/.guildos.env <<'EOF'",
    `export GUILDOS_NEXT_PUBLIC_SUPABASE_URL='${supabaseUrl}'`,
    `export GUILDOS_SUPABASE_SECRETE_KEY='${supabaseKey}'`,
    `export NEXT_PUBLIC_SUPABASE_URL='${supabaseUrl}'`,
    `export SUPABASE_SECRETE_KEY='${supabaseKey}'`,
    "EOF",
    "echo 'source ~/.guildos.env' >> ~/.bashrc",
    "source ~/.guildos.env",
    "test -n \"$GUILDOS_SUPABASE_SECRETE_KEY\" && echo 'GuildOS creds loaded' || echo 'FAILED — escalate'",
    "```",
    "",
    "After that, run housekeeping.initAgent (clones GuildOS, loads skill books, finds your quests). The init step assumes the env is already loaded.",
    "",
    "═════════════════════════════════════════════════════════════════════",
    "TASK PROMPT",
    "═════════════════════════════════════════════════════════════════════",
    "",
  ].join("\n");
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/openai_images/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/weapon/openai_images`

Rationale layer (per CLAUDE.md §6). Append; supersede in place when a decision
changes.

---

## 2026-06-09 — never pass `response_format` to the images endpoints

**Decision.** `generate`/`edit` never send `response_format`. `b64_json` is always
returned regardless.

**Rejected alternative.** Pass `response_format: "b64_json"` for explicitness (the
intuitive, "correct-looking" call).

**Why.** OpenAI's image endpoints 400 with `unknown_parameter` when
`response_format` is supplied — the parameter isn't in the current schema, yet
b64_json is the only format and is always returned. Sending it for "clarity" breaks
every call.

**Consequence if violated.** All image calls fail with `unknown_parameter`; the
caller suspects their request shape when it's a service-side quirk.

---

## 2026-06-09 — the vision judge uses `gpt-4o`; `gpt-4o-mini` is banned for verification (commit 5f9b4c0)

**Decision.** `judge` runs on full `gpt-4o`. `gpt-4o-mini` is explicitly banned for
the verification path.

**Rejected alternative.** Use `gpt-4o-mini` to cut verification cost.

**Why.** Mini measured a ~49% false-alarm rate on dense UI screenshots (the actual
use case) — insufficient visual reasoning. Verification is a gate; a cheap judge
that hallucinates failures (or passes) defeats its purpose. The cost/accuracy
trade-off was settled at 5f9b4c0 so it isn't re-litigated.

**Consequence if violated.** The visual gate accepts wrong images (or false-alarms
on good ones), undermining the verification system that the rest of GuildOS leans
on.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/openai_images/index.js -->
```js
/**
 * OpenAI Images weapon — image generation and editing.
 *
 * Model: configured via OPENAI_IMAGE_MODEL env var, default "gpt-image-2".
 * To swap models cluster-wide (e.g. when a newer model ships), update the env
 * var in deployment + every locally-running .env.local — no code change.
 *
 * ─── CALLER DISCIPLINE (LOCKED 2026-05-22) ────────────────────────────────
 * Callers MUST be API routes (app/api/**) or skill-book actions
 * (libs/skill_book/**). One-off `.mjs` scripts in `scripts/` are NOT a
 * supported mechanism — they bypass the env-driven model source of truth in
 * practice (people hardcode the model name) and bypass the API-route audit
 * surface. The legacy `scripts/_generate-marketing-infographics.mjs` and
 * `scripts/_design_smoke_v4_gpt_image_2.mjs` are NOT precedent for new
 * one-offs; they're being deprecated. If the dev server is broken, surface
 * that — don't fall back to a script.
 *
 * Credential: OPENAI_API_KEY from profiles.env_vars, fallback to process.env.
 *
 * ─── API surface ──────────────────────────────────────────────────────────
 * Generate: POST https://api.openai.com/v1/images/generations
 *   body (JSON): { model, prompt, size, quality, n }
 *   response:    { data: [{ b64_json }], usage }
 *
 * Edit: POST https://api.openai.com/v1/images/edits  (multipart/form-data)
 *   fields: model, prompt, size, quality, n, image[] (Blob | Buffer)
 *   response: { data: [{ b64_json }], usage }
 *
 * ─── Verified behavior (smoke-tested 2026-04-23) ──────────────────────────
 * - DO NOT pass response_format — causes unknown_parameter error; b64_json is
 *   always returned regardless.
 * - Supported sizes: "1024x1024" | "1024x1536" | "1536x1024" | "auto"
 * - Supported quality: "low" | "medium" | "high"
 * - gpt-image-2 requires org verification on the OpenAI platform before the
 *   model unlocks for API calls.
 */

import { database } from "@/libs/council/database";

export const toc = {
  generate: {
    description:
      "Generate an image — infographic, chart, diagram, poster, illustration, logo, icon, banner, slide background, artwork, or photo — from a text prompt using gpt-image-2 (\"ChatGPT image\"). THE way to create any picture / visual / graphic from words. Returns array of b64_json strings. Via run-action, pass a long/multiline prompt with --input-file <json> and write the PNG(s) with --out <path> (n>1 → -1,-2… suffix).",
    // Binary-output contract honored by run-action's `--out`: the result IS an
    // array of base64 PNG payloads ("from":"$"), each decoded and written to a
    // .png file. This is the metadata the adapter cli.mjs used to carry.
    output: { binary: true, encoding: "base64", contentType: "image/png", ext: "png", from: "$" },
  },
  edit: {
    description:
      "Edit, restyle, redraw, composite, inpaint, or merge existing images (change style, swap / add / remove elements, combine multiple pictures) using gpt-image-2 edits endpoint. Accepts image Buffers + prompt. Returns array of b64_json strings. Via run-action, write the PNG(s) with --out <path>.",
    output: { binary: true, encoding: "base64", contentType: "image/png", ext: "png", from: "$" },
  },
  judge:
    "Contextless vision check: ask gpt-4o (full) whether a given image matches a written claim. Optional baselineUrl adds a second image (the AI-approved design baseline) — pass requires both claim satisfaction AND ≥90% visual similarity (or all key elements present). Returns { verdict, confidence, reasoning, model, baseline_compared }. No chat history, no narrative drift. gpt-4o-mini was retired 2026-04-26 (49% false-alarm rate, see CLAUDE.md WWCD bans).",
};

/**
 * API base URL. Defaults to OpenAI direct. Override with OPENAI_IMAGE_BASE_URL
 * (or OPENAI_BASE_URL) to route through a supported-region OpenAI-compatible
 * proxy — needed where OpenAI geo-blocks the images endpoint by request origin
 * ("Country, region, or territory not supported"). Value should be the full
 * base incl. version segment, e.g. "https://my-proxy.example.com/v1".
 */
function apiBase() {
  return (
    process.env.OPENAI_IMAGE_BASE_URL ||
    process.env.OPENAI_BASE_URL ||
    "https://api.openai.com/v1"
  );
}

/**
 * Read the active image-generation model. Single source of truth — every
 * caller (weapon + scripts) goes through this so a model bump is one env
 * change, not a code-wide grep-and-replace.
 */
export function imageModel() {
  return process.env.OPENAI_IMAGE_MODEL || "gpt-image-2";
}

async function loadKey(userId) {
  if (userId) {
    try {
      const db = await database.init("service");
      const { data } = await db
        .from("profiles")
        .select("env_vars")
        .eq("id", userId)
        .single();
      const k = data?.env_vars?.OPENAI_API_KEY;
      if (k) return k;
    } catch {
      /* fall through */
    }
  }
  return process.env.OPENAI_API_KEY || null;
}

/**
 * Generate images from a text prompt.
 *
 * @param {{ prompt: string, size?: string, quality?: string, n?: number, background?: "transparent"|"opaque"|"auto" }} input
 *   background — gpt-image-2 alpha control. "transparent" returns a true alpha PNG
 *   (use for stickers/badges/logos that overlay other imagery). Omit for default.
 * @param {string} [userId]
 * @returns {Promise<string[]>} Array of base64-encoded PNG strings (one per image)
 */
export async function generate(
  { prompt, size = "1024x1024", quality = "medium", n = 1, background } = {},
  userId
) {
  const key = await loadKey(userId);
  if (!key) throw new Error("OPENAI_API_KEY not configured");

  const body = { model: imageModel(), prompt, size, quality, n };
  if (background) body.background = background;

  const res = await fetch(`${apiBase()}/images/generations`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${key}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(body),
  });

  const json = await res.json();
  if (json.error) throw new Error(`OpenAI Images error: ${json.error.message}`);
  return json.data.map((d) => d.b64_json);
}

/**
 * Edit / restyle one or more images.
 *
 * @param {{ images: (Buffer|Blob)[], prompt: string, size?: string, quality?: string, n?: number }} input
 *   images — array of image Buffers (PNG preferred). Max 10 inputs.
 * @param {string} [userId]
 * @returns {Promise<string[]>} Array of base64-encoded PNG strings
 */
export async function edit(
  { images, prompt, size = "1024x1024", quality = "medium", n = 1 } = {},
  userId
) {
  const key = await loadKey(userId);
  if (!key) throw new Error("OPENAI_API_KEY not configured");
  if (!images?.length) throw new Error("At least one image is required for edit");

  const form = new FormData();
  form.append("model", imageModel());
  form.append("prompt", prompt);
  form.append("size", size);
  form.append("quality", quality);
  form.append("n", String(n));

  for (const img of images) {
    const blob =
      img instanceof Blob ? img : new Blob([img], { type: "image/png" });
    form.append("image[]", blob, "image.png");
  }

  const res = await fetch(`${apiBase()}/images/edits`, {
    method: "POST",
    headers: { Authorization: `Bearer ${key}` },
    body: form,
  });

  const json = await res.json();
  if (json.error) throw new Error(`OpenAI Images edit error: ${json.error.message}`);
  return json.data.map((d) => d.b64_json);
}

/**
 * Contextless image-vs-claim judge. Each call is its own zero-history request to
 * the chat-completions endpoint with a vision message. Used as a "second pair of
 * eyes" gate — the judging model has no narrative about the work, no pressure to
 * defend prior output, no risk of overanchoring on the agent's own description.
 * Strict JSON output for downstream gating.
 *
 * @param {{ imageUrl: string, claim: string, baselineUrl?: string, model?: string }} input
 *   imageUrl    — public URL of the ACTUAL image being verified (the worker's
 *                 deliverable). OpenAI server must be able to fetch it.
 *   claim       — what the image is supposed to show. THE canonical claim
 *                 (typically items.expectation verbatim).
 *   baselineUrl — OPTIONAL. The AI-approved design baseline image from the
 *                 design stage. When present, the judge ALSO checks that the
 *                 actual is ≥90% similar to the baseline OR contains all key
 *                 elements (layout, components, hierarchy). Both checks must
 *                 pass for a 'match' verdict. When absent, behavior is the
 *                 same as before — text-only claim check.
 *   model       — defaults to "gpt-4o" (full). gpt-4o-mini is BANNED for
 *                 verification (CLAUDE.md WWCD bans, 5f9b4c0): 49% false-alarm
 *                 rate on dense screenshots.
 * @param {string} [userId]
 * @returns {Promise<{ verdict: 'match'|'mismatch'|'inconclusive', confidence: number, reasoning: string, model: string, baseline_compared: boolean }>}
 */
export async function judge({ imageUrl, claim, baselineUrl, model = "gpt-4o" } = {}, userId) {
  if (typeof imageUrl !== "string" || !imageUrl.trim()) {
    throw new Error("judge: imageUrl is required");
  }
  if (typeof claim !== "string" || !claim.trim()) {
    throw new Error("judge: claim is required");
  }
  const hasBaseline = typeof baselineUrl === "string" && !!baselineUrl.trim();

  const key = await loadKey(userId);
  if (!key) throw new Error("OPENAI_API_KEY not configured");

  const systemPromptBase = [
    "You are a strict image verification judge.",
    "Your only job is to decide whether the deliverable image satisfies the claim.",
    "Be conservative: if you cannot confirm from the image(s) alone, return 'mismatch' or 'inconclusive'.",
    "Common failure modes to flag as mismatch:",
    "  - stock photo / generic image instead of the claimed subject",
    "  - blank / empty / loading state instead of populated content",
    "  - error page, 404, login screen, dev overlay instead of the claimed page",
    "  - placeholder text or skeleton loaders never resolved",
    "  - URL bar / page title clearly inconsistent with the claim",
  ];
  const systemPromptBaselineExtra = [
    "You will be given TWO images: the ACTUAL deliverable AND a BASELINE design that was AI-approved at the design stage.",
    "Pass requires BOTH:",
    "  (a) the actual image satisfies the claim text, AND",
    "  (b) the actual image is ≥90% visually similar to the baseline OR contains all key elements (layout, components, hierarchy, primary content blocks).",
    "Cosmetic differences (minor spacing, font weight) are tolerated. Missing whole sections, wrong layout, or different page entirely → mismatch.",
  ];
  const systemPromptOutput = [
    "Respond with VALID JSON ONLY, no markdown fences, no preamble:",
    '{ "verdict": "match" | "mismatch" | "inconclusive", "confidence": 0.0-1.0, "reasoning": "<one sentence>" }',
  ];
  const systemPrompt = (hasBaseline
    ? [...systemPromptBase, ...systemPromptBaselineExtra, ...systemPromptOutput]
    : [...systemPromptBase, ...systemPromptOutput]
  ).join("\n");

  const userMessage = hasBaseline
    ? [
        {
          type: "text",
          text: `Claim: ${claim}\n\nFirst image: ACTUAL deliverable.\nSecond image: BASELINE design (AI-approved reference).\nDoes the actual satisfy the claim AND match the baseline?`,
        },
        { type: "image_url", image_url: { url: imageUrl } },
        { type: "image_url", image_url: { url: baselineUrl } },
      ]
    : [
        { type: "text", text: `Claim: ${claim}\n\nDoes the image match this claim?` },
        { type: "image_url", image_url: { url: imageUrl } },
      ];

  const res = await fetch(`${apiBase()}/chat/completions`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${key}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model,
      messages: [
        { role: "system", content: systemPrompt },
        { role: "user", content: userMessage },
      ],
      temperature: 0,
      max_tokens: 200,
      response_format: { type: "json_object" },
    }),
  });

  const json = await res.json();
  if (json.error) throw new Error(`OpenAI judge error: ${json.error.message}`);
  const content = json?.choices?.[0]?.message?.content;
  if (typeof content !== "string") {
    throw new Error("OpenAI judge returned no content");
  }

  let parsed;
  try {
    parsed = JSON.parse(content);
  } catch {
    throw new Error(`OpenAI judge returned non-JSON content: ${content.slice(0, 200)}`);
  }

  const verdict =
    parsed.verdict === "match" || parsed.verdict === "mismatch" || parsed.verdict === "inconclusive"
      ? parsed.verdict
      : "inconclusive";
  const confidence =
    typeof parsed.confidence === "number" && parsed.confidence >= 0 && parsed.confidence <= 1
      ? parsed.confidence
      : 0;
  const reasoning = typeof parsed.reasoning === "string" ? parsed.reasoning : "";

  return { verdict, confidence, reasoning, model, baseline_compared: hasBaseline };
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/questExecution/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/weapon/questExecution` (and the gate weapons)

Rationale layer (per CLAUDE.md §6). The WHY behind the stage-transition gates
shared by `questExecution`, `questPurrview`, and `questReview` — not API docs, not
procedure. Append; supersede in place when a decision changes.

---

## 2026-06-13 — every stage transition is a lockphrase-bearing comment, written weapon-atomically with the stage write

**Decision.** A stage transition writes `quests.stage` AND inserts a
`quest_comments` row carrying a gate-specific **lockphrase**
(`SUBMIT_LOCKPHRASE` at `questExecution/index.js:40`, `APPROVE_LOCKPHRASE` at
`questPurrview/index.js:96`, `FINAL_GATE_PASS_LOCKPHRASE` at
`questReview/index.js:56`) together. The next consumer (`confirmSubmission`,
`confirmApproval`) **greps for the prior gate's lockphrase** before proceeding; it
never trusts a bare `.stage` value.

**Rejected alternative.** A simple stage-column check (`quest.stage === 'execute'`)
with no orthogonal audit comment.

**Why.** A bare stage write is invisible to auditing and forgeable by raw SQL or a
rogue call. The lockphrase comment IS the proof the gate ran: it makes every
transition tamper-evident, and couples the audit trail to the stage so a partial
transition (stage moved, comment missing) is detected and rejected by the next
gate rather than sailing through.

**Consequence if violated.** Rogue/partial stage writes bypass gates entirely — a
quest reaches review with unfinished objectives and the review gates become
theater (the leak class the operator repeatedly flags).

---

## 2026-06-13 — each gate stamps a `GATE_VERSION` into its audit comment

**Decision.** Every gate exports a `GATE_VERSION` constant and writes it to the
audit comment's `detail.gate_version` (`questExecution:34`, `questPurrview:86`,
`questReview:49`, `questDesign:28`). Quest comments are immutable; the version lets
a future reader know which gate logic a past approval passed.

**Rejected alternative.** Omit versioning (no forward tracking of gate changes).

**Why.** When a gate's criteria change, old quests carry comments from the old
gate. Without a version stamp, a past approval is ambiguous — did it clear the
current gate or a weaker predecessor? — making targeted re-validation impossible.

**Consequence if violated.** Gate-logic evolution becomes invisible in the trail;
re-validation after a gate change is guesswork.

---

## 2026-06-13 — visual verification is the reviewer's responsibility, NOT the worker's

**Decision.** The worker's `submit()` gate requires only a non-null deliverable
`ref` ("I made it exists"). The `verify_screenshot_ref` requirement lives
downstream at `questPurrview.approve()` ("it's real and satisfies the spec"), with
`autoVisualVerify()` capturing the screenshot for headless quests.

**Rejected alternative.** Symmetric gates (worker also blocks on a screenshot), or
the worker captures the screenshot itself.

**Why.** A headless worker leaf has no CIC / user-channel browser to capture a live
screenshot — only the CIC-capable reviewer (or local Claude at review) can. Putting
the screenshot gate on the worker would either block every headless quest or push
the worker to fake it; deferring it to purrview keeps the worker autonomous while
preserving a real visual gate.

**Consequence if violated.** Either headless workers stall on an unobtainable
screenshot, or unverified deliverables flow into review as "trusted" and the visual
gate goes toothless.

---

## 2026-06-13 — open unblocking questions at submit route to `escalated`, not `purrview`

**Decision.** If `submit()` finds any open unblocking question
(`qtype='unblocking', answered=false`), it routes the quest to `escalated` (with an
explanatory comment) instead of advancing to `purrview`.

**Rejected alternative.** Always `submit() → purrview` (gate only on objectives +
deliverables) and let the review path notice the questions later.

**Why.** A quest with unanswered user-facing questions is not "ready for review" —
it's waiting on the user. Parking it in purrview misrepresents its state;
escalating signals the user that input is needed now. (Companion to the
`questmaster` planner-side escalate ED — same principle at the submit boundary.)

**Consequence if violated.** Quests sit in purrview awaiting answers, breaking the
contract that purrview = "worker done, please review."
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/questExecution/index.js -->
```js
/**
 * questExecution weapon — worker-side quest stage transitions.
 *
 * Owns the WORKER's view: how an adventurer ships a quest from execute → purrview.
 * The agent CANNOT advance a quest by writing `stage` directly. The only
 * supported path is `submit({ questId })`, which refuses to move the quest unless
 * every gate passes.
 *
 * New steps/items model (post 20260515120000_steps_items_refactor):
 *   - readiness is the `steps` + `items` state, NOT the retired
 *     items.expectation / per-item item_comments / tier-check columns.
 *   - A quest is submittable when every step up to AND including the Checkpoint
 *     (the step whose view_results='Checkpoint') is status='done', and every
 *     asset those steps name in output[].asset exists as an `items` row with a
 *     non-null `ref`.
 *
 * Sister weapon: `questPurrview` owns the QUESTMASTER's view (confirmSubmission
 * for the read-side handoff check, approve/bounce for the write-side transitions
 * out of purrview).
 *
 * Auth: uses the service-role DB facade (`database.init("service")`).
 * No external API. Reads `steps`, `items`, `quests`; writes `quests`, `quest_comments`.
 */

import { database } from "@/libs/council/database";
import { publicTables } from "@/libs/council/publicTables";
import { getQuestDetails } from "@/libs/weapon/questmaster/index.js";

export const toc = {
  submit:
    "Move a quest from execute to purrview. Hard-gated on the new steps/items model: every step up to and including the Checkpoint is status='done', and the items those steps produce exist with a non-null ref. On success, leaves the lockphrase comment that questPurrview.confirmSubmission checks for.",
};

export const GATE_VERSION = 2;
/**
 * Stable phrase embedded in the success comment.
 * `questPurrview.confirmSubmission` greps for it verbatim.
 * Do not change without coordinating with every consumer.
 */
export const SUBMIT_LOCKPHRASE = "this quest now meets the criteria for purrview";

const CRITERIA_CHECKED = [
  "questId is a non-empty string",
  "quest exists",
  "quest.stage === 'execute'",
  "objectives rows exist for the quest",
  "every objective is status='done'",
  "every deliverable has a non-null ref",
  "stage write to purrview verified by SELECT-back",
];

/**
 * Submit a quest for Questmaster review.
 *
 * On full pass: writes `stage = "purrview"`, posts a `submit_for_purrview` audit
 * comment containing the lockphrase + full criteria list, returns
 * `{ ok: true, stage: "purrview", ... }`.
 *
 * On any failure: stage stays at `execute`, returns
 * `{ ok: false, failed: [<gate-id>], report: { msg, fix, ...details } }`.
 *
 * @param {{ questId: string }} args
 */
export async function submit({ questId }) {
  // Gate 1 — questId
  if (typeof questId !== "string" || !questId.trim()) {
    return {
      ok: false,
      failed: ["questId"],
      report: {
        msg: "questId is required (non-empty string).",
        fix: "Pass { questId: '<uuid>' } as the argument.",
      },
    };
  }

  const db = await database.init("service");

  // Gates 2 + 3 — read the quest (also pulls assignee so we can stamp
  // actor_name on the submit comment).
  const { data: quest, error: qErr } = await db
    .from(publicTables.quests)
    .select("id, stage, title, assignee")
    .eq("id", questId)
    .single();
  if (qErr || !quest) {
    return {
      ok: false,
      failed: ["quest_exists"],
      report: {
        msg: qErr?.message || "Quest not found.",
        fix: "Verify the questId. Confirm the quest exists with: SELECT id, stage FROM quests WHERE id = '<uuid>'.",
        questId,
      },
    };
  }
  if (quest.stage !== "execute") {
    return {
      ok: false,
      failed: ["stage"],
      report: {
        msg: `Quest stage is "${quest.stage}", expected "execute". Submission only valid from execute.`,
        fix:
          quest.stage === "purrview" || quest.stage === "review"
            ? "Quest is already past execute — no resubmission needed. If you have new artifacts, post them as comments and ping the Questmaster."
            : "If the quest is in closing/complete, contact the Guildmaster — submission only opens from execute.",
        current_stage: quest.stage,
      },
    };
  }

  // Gate 3b — open unblocking questions BLOCK submission → route to escalated.
  // If the agent has unanswered questions TO the user (qtype='unblocking',
  // answered=false), the quest is waiting on the user — it is NOT ready for
  // purrview/review. A quest can never pass review with the agent's own
  // questions unanswered, so we transition it to 'escalated' (awaiting the user)
  // instead of advancing. (operator rule 2026-06-13: "with N unblocking
  // questions unanswered, it should not progress to review but to escalated".)
  const { data: openUnblocking } = await db
    .from(publicTables.questQuestions)
    .select("id, body")
    .eq("quest_id", questId)
    .eq("qtype", "unblocking")
    .eq("answered", false);
  if (openUnblocking && openUnblocking.length > 0) {
    await db.from(publicTables.quests).update({ stage: "escalated" }).eq("id", questId);
    const { recordQuestComment } = await import("@/libs/quest/index.js");
    await recordQuestComment(questId, {
      source: "questExecution",
      action: "escalated_open_questions",
      summary: `Routed to escalated, not purrview: ${openUnblocking.length} open unblocking question(s) await the user. A quest cannot pass review while the agent's own questions to the user are unanswered.`,
      actor_name: quest.assignee || null,
      detail: { open_unblocking_count: openUnblocking.length, question_ids: openUnblocking.map((q) => q.id) },
    }, { client: db });
    return {
      ok: true,
      stage: "escalated",
      escalated: true,
      open_unblocking: openUnblocking.length,
      report: {
        msg: `${openUnblocking.length} open unblocking question(s) await the user — quest routed to escalated instead of purrview.`,
        fix: "The user must answer the unblocking questions (quest_questions, qtype='unblocking'). Once answered, the worker digests them and re-submits.",
      },
    };
  }

  // Gate 4 — objectives rows exist
  const { data: steps, error: sErr } = await db
    .from(publicTables.objectives)
    .select("id, n, name, status")
    .eq("quest_id", questId)
    .order("n", { ascending: true });
  if (sErr) {
    return {
      ok: false,
      failed: ["steps_read"],
      report: {
        msg: sErr.message,
        fix: "DB read against `steps` failed. Check service-role auth and table name (libs/council/publicTables.js: steps).",
      },
    };
  }
  const stepList = steps || [];
  if (stepList.length === 0) {
    return {
      ok: false,
      failed: ["no_steps"],
      report: {
        msg: "Quest has no steps rows. The materialized SMART plan is empty — there's nothing to execute or submit.",
        fix: "Steps are written at quest creation by questmaster.createQuest. If they're missing, the quest was created incorrectly — escalate to the Guildmaster.",
      },
    };
  }

  // Gate 5 — every objective is status='done'
  const unfinished = stepList.filter((s) => s.status !== "done");
  if (unfinished.length > 0) {
    return {
      ok: false,
      failed: ["steps_not_done"],
      report: {
        msg: `${unfinished.length} step(s) up to the Checkpoint are not status='done'.`,
        fix: `Finish each step via worker.executeStep until its status flips to 'done'. Unfinished (non-done) steps: ${unfinished
          .map((s) => `n${s.n} "${s.name}" (${s.status})`)
          .join(", ")}.`,
        unfinished_steps: unfinished.map((s) => ({ n: s.n, name: s.name, status: s.status })),
      },
    };
  }

  // Gate 6 — every deliverable has a non-null ref (the produced artifact).
  // Deliverables are the canonical list of what the quest must produce; the
  // reviewer grades them, not objective output[] (dropped in BCS migration).
  const details = await getQuestDetails({ questId });
  if (details?.error) {
    return {
      ok: false,
      failed: ["quest_unreadable_via_ui_path"],
      report: {
        msg: `getQuestDetails failed: ${details.error}. The reviewer's page would not render.`,
        fix: "Fix getQuestDetails / the read path so the quest page can load.",
      },
    };
  }
  const uiDeliverables = Array.isArray(details?.deliverables) ? details.deliverables : [];
  const missingRef = uiDeliverables.filter(
    (d) => d.ref == null || (typeof d.ref === "string" && d.ref.trim() === ""),
  );
  if (missingRef.length > 0) {
    return {
      ok: false,
      failed: ["deliverables_missing_ref"],
      report: {
        msg: `${missingRef.length} deliverable(s) have no ref (produced artifact): ${missingRef.map((d) => d.name).join(", ")}.`,
        fix: "Ship each deliverable via worker.shipItemArtifact / libs/quest writeDeliverable. UPSERT on (quest_id, name) replaces in place.",
        missing: missingRef.map((d) => d.name),
      },
    };
  }

  // NOTE: VISUAL VERIFICATION is the PURRVIEW agent's responsibility, NOT the
  // worker's — a headless worker leaf has no CIC/user-channel browser to capture
  // a verify screenshot. The submit gate therefore requires only a produced `ref`
  // (above). The verify_screenshot_ref requirement lives at the purrview→review
  // gate (questPurrview.approve), where the CIC-capable reviewer sets it. See
  // docs/quest-lifecycle.md (visual-verification responsibility).

  // All gates pass — write lockphrase comment FIRST (the DB trigger
  // enforce_stage_gate requires it before the stage write succeeds).
  const summary = `Gate v${GATE_VERSION} passed. ${stepList.length} objective(s) done, ${uiDeliverables.length} deliverable(s) verified. ${SUBMIT_LOCKPHRASE.charAt(0).toUpperCase() + SUBMIT_LOCKPHRASE.slice(1)}.`;
  const { recordQuestComment } = await import("@/libs/quest/index.js");
  // submit_for_purrview is a PASS_THROUGH gate action (the chokepoint always
  // inserts it fresh — the DB enforce_stage_gate trigger requires the lockphrase
  // comment before the stage write, and auditStageBypass times its freshness).
  const { error: commentErr } = await recordQuestComment(questId, {
    source: "questExecution",
    action: "submit_for_purrview",
    summary,
    actor_name: quest.assignee || null,
    detail: {
      gate_version: GATE_VERSION,
      lockphrase: SUBMIT_LOCKPHRASE,
      criteria_checked: CRITERIA_CHECKED,
      steps_count: stepList.length,
      assets: uiDeliverables.map((d) => d.name),
    },
  }, { client: db });
  if (commentErr) {
    return {
      ok: false,
      failed: ["lockphrase_comment_write"],
      report: {
        msg: `Lockphrase comment failed to write: ${commentErr.message}. Stage NOT advanced (comment-before-stage order).`,
        fix: `Debug the quest_comments insert. The DB trigger enforce_stage_gate blocks stage→purrview without this comment.`,
      },
    };
  }

  // Advance stage (the trigger allows it because the lockphrase comment exists).
  const { error: upErr } = await db
    .from(publicTables.quests)
    .update({ stage: "purrview" })
    .eq("id", questId);
  if (upErr) {
    return {
      ok: false,
      failed: ["stage_write"],
      report: {
        msg: upErr.message,
        fix: "Quest stage write failed at the DB layer. Check service-role auth or the enforce_stage_gate trigger — this is a system-level failure, not an agent fix.",
      },
    };
  }
  const { data: verify } = await db
    .from(publicTables.quests)
    .select("stage")
    .eq("id", questId)
    .single();
  if (verify?.stage !== "purrview") {
    return {
      ok: false,
      failed: ["stage_write_verify"],
      report: {
        msg: `Stage write did not persist; reads back as "${verify?.stage}".`,
        fix: "Stage write was silently rejected (likely RLS or the enforce_stage_gate trigger). Check that the lockphrase comment exists.",
      },
    };
  }

  return {
    ok: true,
    stage: "purrview",
    steps_count: stepList.length,
    assets_count: uiDeliverables.length,
    lockphrase: SUBMIT_LOCKPHRASE,
    criteria_checked: CRITERIA_CHECKED,
  };
}
```
<!-- /GUILDOS-FILE -->
