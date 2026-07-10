# cursor

<!-- GUILDOS-SKILL name="cursor" -->

<!-- GUILDOS-DESC -->
Dispatch tasks to Cursor cloud agents, check status, read conversations, and generate PowerPoint presentations. Send a task via followup message, poll agent status, review conversation history to extract results, dispatch PPT generation that uploads to Supabase Storage, bootstrap a fresh agent session's env, and write minimal adventurer system prompts. All executable logic lives in the paired cursor weapon; this skill holds the prompt/procedure registry.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/cursor/SKILL.md -->
```markdown
---
name: cursor
description: >-
  Dispatch tasks to Cursor cloud agents, check status, read conversations, and
  generate PowerPoint presentations. Send a task via followup message, poll
  agent status, review conversation history to extract results, dispatch PPT
  generation that uploads to Supabase Storage, bootstrap a fresh agent session's
  env, and write minimal adventurer system prompts. All executable logic lives in
  the paired cursor weapon; this skill holds the prompt/procedure registry.
---

# Cursor Cloud Agents

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/cursor/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto cursor <action>` (or read the source directly).

## Actions

- **dispatchPptGeneration** — Dispatch a PowerPoint generation task to a Cursor cloud agent. The agent generates the PPT and uploads it to Supabase Storage. Describe the content in natural language — the agent decides tools and layout.
- **dispatchTask** — Send a task to a Cursor cloud agent via followup message. The agent receives the message and executes it autonomously. Always include explicit push instructions if code changes are expected.
- **prepareEnvironment** — Generate the env setup script for a fresh Cursor cloud agent session and send it as the first followup.
- **readConversation** — Read the full conversation history of a Cursor cloud agent. Useful for reviewing what the agent did and extracting results.
- **readStatus** — Check the current status of a Cursor cloud agent. Returns whether the agent is active, idle, or finished.
- **writeMinimalSystemPrompt** — Write a minimal adventurer system_prompt — only rules that change behavior from the default.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/cursor/index.js -->
```js
/**
 * Cursor skill book — PURE prompt registry.
 *
 * Dispatch tasks to Cursor cloud agents, check status, read conversations, and
 * generate PowerPoint presentations. All executable logic lives in the paired
 * weapon at `libs/weapon/cursor/index.js`; this book holds only prompt strings
 * (id/title/description/toc + one howTo string per action). NO imports, NO
 * functions.
 */

export const skillBook = {
  id: "cursor",
  title: "Cursor Cloud Agents",
  description:
    "Dispatch tasks to Cursor cloud agents, check status, read conversations, and generate PowerPoint presentations.",
  steps: [],
  toc: {
    dispatchTask: {
      description:
        "Send a task to a Cursor cloud agent via followup message. The agent receives the message and executes it autonomously. Always include explicit push instructions if code changes are expected.",
      howTo: `
Call the cursor weapon's agent-facing wrapper:

\`\`\`javascript
import { dispatchTask } from "@/libs/weapon/cursor/index.js";
const res = await dispatchTask(userId, { agentId, instructions });
// res = { ok, msg, items: { result, agentId } }
\`\`\`

Inputs:
- \`agentId\` (string, required) — Cursor agent ID (bc-XXXXXXXX-...).
- \`instructions\` (string, required) — natural-language task. Describe WHAT to do
  and the success criteria, not HOW.
- \`model\` (string, optional) — OMIT this field. Leave model selection to the
  proxy / Cursor API default. Do NOT pass "composer-2.0" (or any other hardcoded
  id) — that id is stale and the API rejects it with HTTP 400.
`,
    },
    readStatus: {
      description:
        "Check the current status of a Cursor cloud agent. Returns whether the agent is active, idle, or finished.",
      howTo: `
Call the cursor weapon's agent-facing wrapper:

\`\`\`javascript
import { readStatus } from "@/libs/weapon/cursor/index.js";
const res = await readStatus(userId, { agentId });
// res = { ok, msg, items: { status, agent } }
\`\`\`

Inputs:
- \`agentId\` (string, required) — Cursor agent ID.
`,
    },
    readConversation: {
      description:
        "Read the full conversation history of a Cursor cloud agent. Useful for reviewing what the agent did and extracting results.",
      howTo: `
Call the cursor weapon's agent-facing wrapper (note: \`readConversationAction\`,
to avoid clashing with the weapon's lower-level \`readConversation\` API call):

\`\`\`javascript
import { readConversationAction } from "@/libs/weapon/cursor/index.js";
const res = await readConversationAction(userId, { agentId });
// res = { ok, msg, items: { conversation } }
\`\`\`

Inputs:
- \`agentId\` (string, required) — Cursor agent ID.
`,
    },
    dispatchPptGeneration: {
      description:
        "Dispatch a PowerPoint generation task to a Cursor cloud agent. The agent generates the PPT and uploads it to Supabase Storage. Describe the content in natural language — the agent decides tools and layout.",
      howTo: `
Call the cursor weapon's agent-facing wrapper:

\`\`\`javascript
import { dispatchPptGeneration } from "@/libs/weapon/cursor/index.js";
const res = await dispatchPptGeneration(userId, { agentId, topic, slides, questId });
// res = { ok, msg, items: { result, agentId } }
\`\`\`

Inputs:
- \`agentId\` (string, required) — Cursor agent ID.
- \`topic\` (string, required) — PPT topic/title.
- \`slides\` (string, optional) — natural-language description of desired slides.
- \`questId\` (string, optional) — quest ID for the storage path; the agent uploads
  to \`cursor_cloud/{questId}/\` (falls back to \`cursor_cloud/general/\`).

The wrapper builds the full instruction (pptxgenjs/python-pptx, upload to Supabase
Storage, post the public URL as a quest comment, screenshot one slide as proof,
git push) and sends it as a followup.
`,
    },
    prepareEnvironment: {
      description: "Generate the env setup script for a fresh Cursor cloud agent session and send it as the first followup.",
      howTo: `
Use the cursor weapon:

\`\`\`javascript
import { readEnvSetupInstructions } from "@/libs/weapon/cursor";
const { setupScript } = await readEnvSetupInstructions({ userId });
// Send setupScript to the agent as the first followup message
\`\`\`
`,
    },
    writeMinimalSystemPrompt: {
      description: "Write a minimal adventurer system_prompt — only rules that change behavior from the default.",
      howTo: `
System prompts must be minimal:
- Only include instructions that change behavior from the default. If Claude would do it anyway, don't say it.
- No descriptions of what the agent "is" or "can do" — just actionable rules and constraints.
- Point to a guideline file if rules are detailed. One sentence referencing the file is better than repeating its contents.
- Bad: "You are a general-purpose agent. You handle research, analysis, browser automation..." (fluff, changes nothing)
- Good: "Read /docs/adventurer-claude-non-development-guideline.md before starting. Do not modify GuildOS source code." (actionable, changes behavior)
`,
    },
  },
};

export default skillBook;
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
