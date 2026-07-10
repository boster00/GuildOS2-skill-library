# dailies

<!-- GUILDOS-SKILL name="dailies" -->

<!-- GUILDOS-DESC -->
Recurring curation and upkeep routines: morning email triage under the two-category reply model, daily LinkedIn scan, support-inbox follow-up monitoring, insight cementing, and buffer curation. Use for the daily "step 1" / "morning email" / "do the dailies" flows, support-thread miss detection (insufficient answers + stale threads), and placing user-correction insights into their proper home.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/dailies/SKILL.md -->
```markdown
---
name: dailies
description: >-
  Recurring curation and upkeep routines: morning email triage under the
  two-category reply model, daily LinkedIn scan, support-inbox follow-up
  monitoring, insight cementing, and buffer curation. Use for the daily
  "step 1" / "morning email" / "do the dailies" flows, support-thread miss
  detection (insufficient answers + stale threads), and placing user-correction
  insights into their proper home.
---

# Dailies

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/dailies/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto dailies <action>` (or read the source directly).

## Actions

- **checkLinkedIn** — Sub-action of triageEmailStep1 — daily LinkedIn check via CIC. Not a standalone user-callable plan.
- **checkMainDashboard** — Sub-action of triageEmailStep1 — calls the boster_nexus main dashboard API (slug watchdog-main) and reviews the workflow-owned self-evaluated health scores/warnings. Not a standalone user-callable plan.
- **checkSupportInsufficient** — Sub-action of monitorSupportFollowups — score a thread transcript for Miss A (team reply did not fully answer customer's question). Not a user-callable plan.
- **checkSupportNoReply** — Sub-action of monitorSupportFollowups — score a thread transcript for Miss B (stale thread past follow-up threshold). Not a user-callable plan.
- **curateInsightsBuffer** — Internal maintenance — periodic pass over the CLAUDE.md Insights buffer. Invoked by GM, not by user shorthand.
- **disciplineSmokeTest** — [SMART] Discipline smoke test for autonomous EL — exercises the full execution lifecycle (sequential steps, parallel group, MA-direct composition, SA-verify per deliverable, SA-report) with stub native actions that write scratch files. No real side effects. Aliases: 'discipline smoke test', 'el discipline', 'run discipline test'.
- **mergeInsight** — Internal protocol — invoked by GM/SAs whenever a user correction surfaces a new rule. Decides where the insight lives. Not a user-callable plan.
- **monitorSupportFollowups** — [SMART] Monitor support inbox for insufficient answers and stale customer threads. Recurring daily. Aliases: 'support followups', 'check support', 'monitor support'.
- **triageEmailStep1** — [SMART] Triage unread inbox under the two-category reply model (genuine + addressed directly to CJ/Sijie → LEAVE BE unread; genuine business whose only bosterbio.com address is boster@ → LEAVE BE for CJ, forward-to-Tracy DORMANT 2026-07-07 while CJ is back at work; everything else → MARK READ), actively unsubscribe clear marketing, then check LinkedIn + main dashboard. Recurring daily morning routine. Aliases: 'step 1', 'morning email', 'do the email thing', 'do the dailies'.
- **triageEmailStep2** — [SMART] Mark skippable emails from the remaining inbox. Runs only after user has reviewed step 1 output. Aliases: 'step 2', 'finish email triage'.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/dailies/INSIGHTS.md -->
```markdown
# Insights Buffer

Timestamped entries awaiting promotion to their permanent home.
Format: `- [YYYY-MM-DD] <insight>`

On the next `dailies.curateInsightsBuffer` pass, each entry is either:
(a) promoted to a skill book's `howTo` if domain-specific,
(b) integrated into an existing CLAUDE.md section if cross-cutting, or
(c) deleted if it turned out situational.

Do not leave entries here indefinitely.

---
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/dailies/engineering_decisions.md -->
```markdown
# Dailies — Engineering Decisions

Rationale / drift-prevention for non-native choices in the dailies skill book.
Format: *Decision · Rejected alternative · Why · Consequence-if-violated*.

---

## 2026-06-20 — Email triage uses a binary two-category reply model, NOT pattern-scoring

**Decision.** `triageEmailStep1` classifies the unread inbox by a single LOCKED
rule ("Data: Reply criteria"): the only mail CJ/Sijie acts on is **(1)** genuine
human mail with a CJ/Sijie identity (`<REDACTED-EMAIL>`, `<REDACTED-EMAIL>`,
`<REDACTED-EMAIL>`) in DIRECT `To` → **LEAVE BE** (untouched, unread), and **(2)**
genuine inbound business whose ONLY bosterbio.com address is the shared
`<REDACTED-EMAIL>` mailbox → **FORWARD to <REDACTED-EMAIL> asking Tracy
to process** (then mark the original read). Everything else → **MARK READ**. No
starring.

**Rejected alternative.** The prior model: bulk mark-read via "Patterns to
Ignore" minus "Leave-unread overrides", then **star** the must-reads via
"Patterns to Highlight" + an ambiguous-body-read pass with a points rubric. Those
sections still exist in the file but are now explicitly RETIRED as a decision
surface ("Patterns to Ignore" is kept only as the mark-read noise fast-path).

**Why.** Direct operator instruction (2026-06-20). CJ wants the keep-set defined
by *who it's addressed to* + *genuine vs noise*, not by an ever-growing,
drift-prone pattern/points catalog. Starring was dropped entirely — the two
keep-categories get distinct handling (leave-be vs delegate-to-Tracy) instead of
one "starred" bucket. `<REDACTED-EMAIL>` is deliberately treated as the
shared ops mailbox (a delegation signal), NOT as a personal CJ identity and NOT as
"another employee".

**Consequence-if-violated.** A future agent that "re-enables" the star/highlight
model, or that treats `<REDACTED-EMAIL>` marketing as Category-1, or that forwards
boster@-only *marketing/automated* noise to Tracy, will (a) re-clutter the inbox
with stars CJ no longer uses, and (b) flood Boster support with junk. The
classifier of record is "Data: Reply criteria" — not the legacy pattern sections.

**Edge case baked in.** A `protectandgrow.blog` "retirement income review request"
lead arrives From `<REDACTED-EMAIL>` (the only bosterbio address) and would
naively be Category 2 → forwarded to Tracy. It is CJ's personal venture, NOT
Boster support work, so it is special-cased as Category 1 (LEAVE BE).

## 2026-06-20 — Unsubscribe is ACTIVE, not harvest-and-present

**Decision.** Step 1 performs the one-click unsubscribe itself for
clearly-unsubscribable marketing (`List-Unsubscribe` http(s): POST for RFC-8058
one-click, else GET), then applies the `Unsubscribed` Gmail label. mailto-only /
no-link senders are reported as "manual" — not auto-actioned.

**Rejected alternative.** The prior policy harvested unsubscribe URLs into a
markdown table for the user to click manually (the default, lowest-risk behavior).

**Why.** Direct operator instruction (2026-06-20): "unsubscribe from emails that
I clearly can unsubscribe from." CJ does not want a to-do list of links; he wants
the agent to actually do it where the mechanism is unambiguous and reversible
(re-subscribe is always possible).

**Consequence-if-violated.** Reverting to harvest-only re-imposes manual work the
user explicitly delegated. Conversely, auto-actioning a *mailto* or a bare body
link (which may be a tracking/confirm trap) over-reaches — restrict auto-action to
`List-Unsubscribe` http(s) URLs only.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/dailies/index.js -->
```js
/**
 * Dailies skill book — recurring curation / upkeep routines.
 *
 * Each action is a [SMART]-prefixed baked plan: name + specific deliverables +
 * measurable criteria + actionable steps (referencing other skill book actions)
 * + realistic + time-bound. Tactical data (pattern lists, decision trees,
 * URLs, filter strings) lives below each plan in `## Data:` sections — the
 * Actionable section references them by name, never inlines them.
 */

/**
 * Boster_nexus Dashboards-module origin. Config-driven (env override), NOT a
 * brittle hardcode. Defaults to the boster_nexus dev origin (port 3001 per the
 * module standard in watchdog-data-sync-architecture.md §11). The Dailies
 * Step-1 sub-action `checkMainDashboard` reads the composed main dashboard from
 * `<origin>/dashboards/view/api?slug=<MAIN_DASHBOARD_SLUG>` and reviews the
 * workflow-owned, self-evaluated health scores + warnings.
 */
export const NEXUS_DASHBOARD_ORIGIN =
  process.env.NEXUS_DASHBOARD_ORIGIN ||
  process.env.BOSTER_NEXUS_ORIGIN ||
  "http://localhost:3001";

/** The designated main dashboard slug (watchdog-data-sync-architecture.md §9–10). */
export const MAIN_DASHBOARD_SLUG =
  process.env.MAIN_DASHBOARD_SLUG || "watchdog-main";

/** Built dashboard API URL for the designated main dashboard. */
export const mainDashboardApiUrl = () =>
  `${NEXUS_DASHBOARD_ORIGIN}/dashboards/view/api?slug=${encodeURIComponent(
    MAIN_DASHBOARD_SLUG
  )}`;

/**
 * Fetch + review the composed main dashboard's self-evaluated health.
 *
 * Reads `<NEXUS_DASHBOARD_ORIGIN>/dashboards/view/api?slug=<MAIN_DASHBOARD_SLUG>`
 * (the boster_nexus Dashboards assembler from quest steps 11–13). The assembler
 * returns the composed main dashboard incl. the workflow-owned sync-health panel
 * `{ html, score, signals, status }`. This sub-action does NOT compute health —
 * it only reads the API and surfaces the workflow's self-evaluation, mirroring
 * how `checkLinkedIn` reads an external surface and returns a
 * chat-bound summary line for `triageEmailStep1`.
 *
 * Degrades gracefully: never throws. If the dashboard API is unreachable / a
 * non-2xx / unparseable, returns `ok:false` with an explicit chat-bound note so
 * Step-1 still completes (email/LinkedIn are not blocked by the hook).
 *
 * @param {object} [opts]
 * @param {string} [opts.url] override URL (tests / non-default origin)
 * @param {function} [opts.fetchImpl] fetch override (tests); defaults to global fetch
 * @param {number} [opts.timeoutMs=8000]
 * @returns {Promise<{ok:boolean, slug:string, url:string, score:(number|null),
 *   status:(string|null), warnings:string[], panels:Array, line:string, error?:string}>}
 */
export async function checkMainDashboard(opts = {}) {
  const url = opts.url || mainDashboardApiUrl();
  const slug = MAIN_DASHBOARD_SLUG;
  const fetchImpl =
    opts.fetchImpl || (typeof fetch !== "undefined" ? fetch : null);
  const timeoutMs = Number.isFinite(opts.timeoutMs) ? opts.timeoutMs : 8000;

  const fail = (error) => ({
    ok: false,
    slug,
    url,
    score: null,
    status: null,
    warnings: [],
    panels: [],
    error,
    line: `main dashboard health: UNAVAILABLE — ${error} (checked ${url})`,
  });

  if (!fetchImpl) return fail("no fetch implementation available");

  let res;
  try {
    const ctl =
      typeof AbortController !== "undefined" ? new AbortController() : null;
    const timer = ctl ? setTimeout(() => ctl.abort(), timeoutMs) : null;
    try {
      res = await fetchImpl(url, ctl ? { signal: ctl.signal } : undefined);
    } finally {
      if (timer) clearTimeout(timer);
    }
  } catch (e) {
    return fail(`dashboard API unreachable (${e && e.message ? e.message : e})`);
  }

  if (!res || (typeof res.ok === "boolean" && !res.ok)) {
    return fail(`dashboard API returned HTTP ${res ? res.status : "?"}`);
  }

  let body;
  try {
    body = await res.json();
  } catch (e) {
    return fail("dashboard API response was not valid JSON");
  }

  // Assembler shape: { slug, panels: [{ id?, title?, score, signals[], status, html? }] }
  const panels = Array.isArray(body && body.panels) ? body.panels : [];
  if (!panels.length) {
    return fail("dashboard API returned no panels");
  }

  // Workflow-owned self-evaluation: lowest panel score drives the headline;
  // any signal whose status/level is degraded/critical/warn becomes a warning.
  const scores = panels
    .map((p) => (p && Number.isFinite(Number(p.score)) ? Number(p.score) : null))
    .filter((s) => s !== null);
  const score = scores.length ? Math.min(...scores) : null;

  const norm = (v) => String(v == null ? "" : v).toLowerCase();
  const isBad = (v) =>
    /\b(degrad|critical|fail|warn|stale|error|down|unhealthy)/.test(norm(v));

  const warnings = [];
  for (const p of panels) {
    const pName = (p && (p.title || p.id)) || "panel";
    if (p && isBad(p.status)) {
      warnings.push(`${pName}: status=${p.status}`);
    }
    const sigs = Array.isArray(p && p.signals) ? p.signals : [];
    for (const s of sigs) {
      if (s == null) continue;
      if (typeof s === "string") {
        if (isBad(s)) warnings.push(`${pName}: ${s}`);
        continue;
      }
      // object signal: { name/label, status/level/severity, message/detail }
      const sName = s.name || s.label || s.key || "signal";
      const sLevel = s.status || s.level || s.severity || "";
      const sMsg = s.message || s.detail || s.value || "";
      if (isBad(sLevel) || isBad(s.message) || isBad(s.detail) || s.ok === false) {
        warnings.push(
          `${pName}/${sName}: ${[sLevel, sMsg].filter(Boolean).join(" — ")}`.trim()
        );
      }
    }
  }

  const headStatus =
    panels.find((p) => p && isBad(p.status)) ? "DEGRADED" :
    score != null && score < 100 ? "CHECK" : "OK";

  const line =
    `main dashboard health: ${headStatus}` +
    (score != null ? ` (score ${score})` : "") +
    (warnings.length
      ? ` — ${warnings.length} warning(s): ${warnings.join("; ")}`
      : " — no warnings");

  return {
    ok: true,
    slug,
    url,
    score,
    status: headStatus,
    warnings,
    panels,
    line,
  };
}

export const skillBook = {
  id: "dailies",
  title: "Dailies",
  description: "Recurring curation and upkeep: morning email triage, daily LinkedIn scan, insight cementing, buffer curation.",

  toc: {
    triageEmailStep1: "[SMART] Triage unread inbox under the two-category reply model (genuine + addressed directly to CJ/Sijie → LEAVE BE unread; genuine business whose only bosterbio.com address is boster@ → LEAVE BE for CJ, forward-to-Tracy DORMANT 2026-07-07 while CJ is back at work; everything else → MARK READ), actively unsubscribe clear marketing, then check LinkedIn + main dashboard. Recurring daily morning routine. Aliases: 'step 1', 'morning email', 'do the email thing', 'do the dailies'.",
    triageEmailStep2: "[SMART] Mark skippable emails from the remaining inbox. Runs only after user has reviewed step 1 output. Aliases: 'step 2', 'finish email triage'.",
    monitorSupportFollowups: "[SMART] Monitor support inbox for insufficient answers and stale customer threads. Recurring daily. Aliases: 'support followups', 'check support', 'monitor support'.",
    checkLinkedIn: "Sub-action of triageEmailStep1 — daily LinkedIn check via CIC. Not a standalone user-callable plan.",
    checkMainDashboard: "Sub-action of triageEmailStep1 — calls the boster_nexus main dashboard API (slug watchdog-main) and reviews the workflow-owned self-evaluated health scores/warnings. Not a standalone user-callable plan.",
    checkSupportInsufficient: "Sub-action of monitorSupportFollowups — score a thread transcript for Miss A (team reply did not fully answer customer's question). Not a user-callable plan.",
    checkSupportNoReply: "Sub-action of monitorSupportFollowups — score a thread transcript for Miss B (stale thread past follow-up threshold). Not a user-callable plan.",
    mergeInsight: "Internal protocol — invoked by GM/SAs whenever a user correction surfaces a new rule. Decides where the insight lives. Not a user-callable plan.",
    curateInsightsBuffer: "Internal maintenance — periodic pass over the CLAUDE.md Insights buffer. Invoked by GM, not by user shorthand.",
    disciplineSmokeTest: "[SMART] Discipline smoke test for autonomous EL — exercises the full execution lifecycle (sequential steps, parallel group, MA-direct composition, SA-verify per deliverable, SA-report) with stub native actions that write scratch files. No real side effects. Aliases: 'discipline smoke test', 'el discipline', 'run discipline test'.",
  },

  triageEmailStep1: `
**Name:** Triage unread inbox, then check LinkedIn

**Context:** Daily morning routine that clears the inbox to the small set of mail CJ / Sijie genuinely needs to act on. The reply model is LOCKED and binary — see "Data: Reply criteria" below. Only two categories survive: **(1) genuine human mail addressed directly to CJ / Sijie → LEAVE BE** (untouched, unread, unstarred — CJ replies); **(2) genuine inbound business whose only bosterbio.com address is the shared \`<REDACTED-EMAIL>\` mailbox → FORWARD to <REDACTED-EMAIL> asking Tracy to help process**, then mark the original read (delegated). **Everything else is noise → MARK READ.** No starring (the old star / Patterns-to-Highlight / Leave-unread-override model is RETIRED — see "Data: Reply criteria"). The pass also **actively unsubscribes** (not merely harvests links) from clearly-unsubscribable marketing senders in the noise floor — see "Data: Unsubscribe (active)". Most of the inbox is noise classified from From + Subject alone (no body read) via "Data: Patterns to Ignore"; only the small genuine remainder gets a header read to split leave-be / forward / mark-read. Chains the LinkedIn check and the main-dashboard health check into one morning pass. Output: marked-read count + a LEAVE-BE table + a FORWARD-to-Tracy table + an unsubscribed table + LinkedIn + main-dashboard-health sections.

**Action plan:**

1. Use gmail.searchInbox to fetch the unread set (filter per "Data: Filter" below) → [unread set]. Up to 500; on truncation (exactly 500 returned) split into date windows and merge.
2. Fast-path classify [unread set] from From + Subject ONLY (no body reads) against "Data: Patterns to Ignore" → [noise floor] (the dominant bucket — marketing, automated, transactional, newsletters, DMARC). Everything not provably noise → [genuine candidates] (a small remainder).
3. Use gmail.readMessage on each [genuine candidate] for To/CC + snippet, then apply "Data: Reply criteria" below. Partition into: [leave-be] (Category 1 genuine human mail with a CJ/Sijie identity in direct To, OR Category 2 genuine business whose only bosterbio.com address is <REDACTED-EMAIL> — both LEAVE BE while forward-to-Tracy is DORMANT, see Data: Reply criteria), [candidate→noise] (marketing/automated on closer read, OR another @bosterbio.com employee in To/CC, OR CJ/Sijie only in CC, OR not actionable). Fold [candidate→noise] into [noise floor]. ⛔ Default for anything still ambiguous after the read = mark-read; only clear Category-1/Category-2 mail survives.
4. Use gmail.batchModify (removeLabelIds:["UNREAD"]) on [noise floor] (incl. the [candidate→noise] fold-in) → [marked-read noise]. ⛔ Never include [leave-be] in this batch.
5. Leave [leave-be] completely untouched — unread, unstarred, in inbox. These are CJ/Sijie's to reply to / process.
6. (DORMANT 2026-07-07 — CJ back at work) Forward-to-Tracy delegation is deactivated; Category 2 now folds into [leave-be] at step 3. Re-activate this step per "Data: Reply criteria" Category 2 note when CJ next travels.
7. Actively unsubscribe per "Data: Unsubscribe (active)" below: from [noise floor] senders that are clearly unsubscribable marketing (List-Unsubscribe http(s), ≥2 hits/30d, not on "Patterns to never unsubscribe", not already on the \`Unsubscribed\` label), perform the one-click unsubscribe DIRECTLY (POST for RFC-8058 one-click, else GET the http(s) link), then apply the \`Unsubscribed\` Gmail label → [unsubscribed set]. mailto-only / no-link senders are reported as "manual" (not auto-actioned).

[parallel]
8a. Use dailies.checkLinkedIn → [linkedin summary]
8b. Use dailies.checkMainDashboard (calls the boster_nexus main dashboard API per "Data: Main dashboard API" below; reviews the workflow-owned self-evaluated health scores + degraded/critical warnings; degrades gracefully if the API is unreachable) → [main dashboard health]
[/parallel]

9. [MA-direct] Compose the chat report combining [marked-read noise] count, [leave-be] table, [forwarded set] table, [unsubscribed set] table, [linkedin summary], [main dashboard health] → [chat report]

**Deliverables:**

| name | asset link/details | verification criteria | eval status | fail logs |
|---|---|---|---|---|
| marked-read noise | Gmail batchModify removeLabel UNREAD applied to [noise floor] message IDs | Count reported as exact integer; zero overlap with [leave-be] or [forward-to-Tracy]; spot-check 3 random IDs are genuinely non-actionable per "Data: Reply criteria" | pending | - |
| leave-be | [leave-be] message IDs left untouched (still UNREAD, no label change) | Every member is genuine human mail with a CJ/Sijie identity in DIRECT To; none were marked read; rendered as a table (sender / subject / reason) so CJ can scan what awaits a reply | pending | - |
| forwarded set | gmail.send forwards to <REDACTED-EMAIL> + original marked read | Each forwarded message had <REDACTED-EMAIL> as the ONLY bosterbio.com address and was genuine inbound business; each forward names Tracy + quotes the original; each original then marked read; rendered as a table | pending | - |
| unsubscribed set | one-click unsubscribe performed (POST/GET) + \`Unsubscribed\` Gmail label applied | Each row was clearly unsubscribable (List-Unsubscribe http(s)), ≥2 hits/30d, not on "Patterns to never unsubscribe"; HTTP result code recorded; mailto-only/no-link senders listed separately as "manual"; rendered as a table | pending | - |
| linkedin summary | dailies.checkLinkedIn return value (chat-bound text + Rienzi-routed emails) | Three buckets named (Rienzi-routed / noteworthy / skip) even if empty; counts present | pending | - |
| main dashboard health | dailies.checkMainDashboard return value (\`.line\` chat-bound text + \`.score\`/\`.warnings\`) | Sourced from the boster_nexus main dashboard API (slug per "Data: Main dashboard API"); reports the workflow-owned self-evaluated health score + any degraded/critical warning signals; if API unreachable, an explicit "main dashboard health: UNAVAILABLE — …" note is present (no throw, Step-1 still completes) | pending | - |
| chat report | final chat response text | Sections in order: Email (marked-read count + LEAVE-BE table + FORWARD-to-Tracy table + unsubscribed table) / LinkedIn / Main dashboard health; exact email counts reported (not estimated); each table renders or shows an explicit "none this run" note; main-dashboard-health section shows the API-sourced score + warnings (or the explicit UNAVAILABLE note) | pending | - |

**Realistic:** Recurring daily proven action. Reply criteria locked 2026-06-20 (see "Data: Reply criteria"); the Patterns-to-Ignore noise list grows via blacksmith.patchSkillBook + EL feedback.

**Time-bound:** Synchronous, single pass, ASAP. ⛔ Do NOT chain into triageEmailStep2 — wait for user review.

---

## Data: Filter

Triage filter query: \`in:unread in:inbox\`
Browser reference: https://mail.google.com/mail/u/0/#search/in%3Aunread+in%3Ainbox

The new reply model marks-read every non-actionable message regardless of Gmail's auto-"important" flag, and must evaluate (not pre-exclude) mail addressed to CJ/Sijie — so the fetch is the full unread inbox, NOT \`-label:important\`. \`-asana\` is also dropped: Asana notifications are noise → mark-read like any other.

⛔ ALWAYS call \`search_emails({ query, maxResults: 500 })\` — NEVER cap at 100 (the MCP default-ish low cap silently drops everything older than the newest ~100 unread). Canonical rule + truncation handling live in the \`gmail\` skill book → \`triageInbox\` Step 1. If exactly 500 IDs return, split into 30-day \`after:/before:\` windows and merge.

## Data: Reply criteria (LOCKED 2026-06-20)

The AUTHORITATIVE classifier. Supersedes the old star / "Patterns to Highlight" / "Leave-unread overrides" / "Hard-skip rules" model — those are RETIRED as a decision surface (the sender lists under "Patterns to Ignore" survive only as the mark-read noise fast-path). The ONLY mail CJ / Sijie needs to act on is these two categories; everything else → MARK READ.

**CJ / Sijie identities** — an address counts as "directly addressed to CJ/Sijie":
- \`<REDACTED-EMAIL>\`
- \`<REDACTED-EMAIL>\`
- \`<REDACTED-EMAIL>\`

\`<REDACTED-EMAIL>\` is the SHARED ops mailbox — it is NOT a personal CJ/Sijie identity, and it does NOT count as "another @bosterbio.com employee" either.

**Category 1 — Personal → LEAVE BE.** A *genuine, human-written* email whose DIRECT \`To\` includes a CJ/Sijie identity. Action: leave it completely untouched — unread, unstarred, in inbox. CJ replies personally. Precedence: if a CJ/Sijie identity is in direct \`To\`, it is Category 1 even when \`boster@\` is also a recipient.

**Category 2 — Shared-mailbox business → LEAVE BE.** A *genuine inbound business* email where the ONLY bosterbio.com address anywhere in From/To/CC is \`<REDACTED-EMAIL>\` (no other \`@bosterbio.com\` person, and no CJ/Sijie identity in direct \`To\`). Action: leave it completely untouched — unread, unstarred, in inbox. CJ processes it personally.

> ⚠ FORWARD-TO-TRACY DEACTIVATED 2026-07-07 (CJ back at work). While CJ was away, Category 2 was forwarded to \`<REDACTED-EMAIL>\` for Tracy to process. That delegation is now DORMANT — Category 2 stays as LEAVE BE so CJ handles shared-mailbox business directly. To re-activate when CJ next travels: restore the action to "forward to <REDACTED-EMAIL> per Data: Forward-to-Tracy format, then mark the original read (delegated)". The "Data: Forward-to-Tracy format" section below is kept intact as the dormant template.

**Everything else → MARK READ.** Including, explicitly:
- Marketing / newsletters / webinars / conferences / digests / automated / transactional / DMARC / receipts (the "Patterns to Ignore" list) — *even when addressed to \`<REDACTED-EMAIL>\`*. Marketing blasted to CJ's address is noise, NOT Category 1.
- Genuine business where ANOTHER \`@bosterbio.com\` employee is in To/CC → they handle it; CJ is FYI.
- CJ/Sijie in CC only (not direct \`To\`).

"Genuine" = written by a real person/company expecting a reply, not a bulk/automated/marketing send. When still ambiguous after a body read, DEFAULT = mark read (CJ optimizes for a clean inbox; mark-read is reversible).

**Exception — CJ personal-venture leads:** an inbound lead for one of CJ's own ventures (e.g. Subject contains "retirement income review request" + "protectandgrow.blog") is Category 1 (LEAVE BE) — it is CJ's to handle, NOT Boster support work, so do NOT forward it to Tracy.

## Data: Forward-to-Tracy format (DORMANT 2026-07-07 — CJ back at work)

> Template kept intact for re-activation when CJ next travels (see "Data: Reply criteria" Category 2 note). While dormant, Category 2 is LEAVE BE and NONE of the below runs.

For each Category-2 message, \`gmail.send\`:
- \`to\`: \`["<REDACTED-EMAIL>"]\`
- \`subject\`: \`"Fwd: " + <original subject>\`
- \`body\`: a short note then the quoted original, e.g.:
  \`\`\`
  Hi Tracy,

  Could you help process this one that came into <REDACTED-EMAIL>? Thanks.

  — CJ

  ---------- Forwarded message ----------
  From: <original From>
  Date: <original Date>
  Subject: <original subject>
  To: <REDACTED-EMAIL>

  <original body / snippet>
  \`\`\`
Then mark the ORIGINAL read (it is now delegated to Tracy). Do not star. If there are many, still forward individually (one thread per customer) so Tracy can reply in-thread — do NOT collapse into one digest.

⚠ Forwarding needs the Gmail \`gmail.send\` scope. The gmail skill book setup note says the stored token may be \`readonly+modify\` only; if \`send_email\` 403s on scope, surface it as a real blocker (re-OAuth with send scope) rather than silently skipping the forwards.

## Data: Patterns to Ignore

> Role under the 2026-06-20 model: this is the **mark-read noise fast-path** — the way to classify the bulk of the inbox as "everything else → MARK READ" from From/Subject alone, without a body read. It is NOT the reply decision; the reply decision lives in "Data: Reply criteria". A sender here is noise even if addressed to \`<REDACTED-EMAIL>\`.

Match any of these from From address / From name / Subject → mark read, no body read needed.

**From address / domain:**
- Any \`noreply@\`, \`no-reply@\`, \`reports@\`, \`dmarcreport@\` — generic automated senders. **No exceptions** — even subjects that look like security or privacy alerts (e.g. "Notetaker joined your Meeting Room") are skipped when the From is noreply/no-reply.
- \`<REDACTED-EMAIL>\` — Google Scholar Alerts
- \`<REDACTED-EMAIL>\` — Google Search Console automated notices
- \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\` — Google Ads tips/reminders
- \`*@vercel.com\`, \`<REDACTED-EMAIL>\` — all Vercel emails (deployment alerts, receipts, notifications)
- \`<REDACTED-EMAIL>\` — all Calendly emails (new events, reminders, cancellations)
- \`<REDACTED-EMAIL>\`, Groupon variants — deals/promo
- \`<REDACTED-EMAIL>\` — Lyft receipts
- \`<REDACTED-EMAIL>\` — Stripe promotional
- \`<REDACTED-EMAIL>\` — all Zoho SalesIQ alerts (missed chats, transcripts, daily reports); CJ is info-only CC
- \`<REDACTED-EMAIL>\` — Zoho SalesIQ daily digest reports
- \`<REDACTED-EMAIL>\` as the From — automated order forwarding
- \`<REDACTED-EMAIL>\` and other \`*@bosterbio.com\` internal senders on transaction emails — CJ is CC'd for info only
- \`<REDACTED-EMAIL>\` / Lillio routine updates — daily activity digests
- \`<REDACTED-EMAIL>\`
- \`<REDACTED-EMAIL>\` — Apollo task/reminder notifications
- \`<REDACTED-EMAIL>\` — Neil Patel newsletter
- \`<REDACTED-EMAIL>\` — agency marketing
- \`<REDACTED-EMAIL>\` — bioprocessing marketing
- \`<REDACTED-EMAIL>\` — Instantly newsletter
- \`<REDACTED-EMAIL>\` — Make.com promos
- \`<REDACTED-EMAIL>\` — ngrok newsletter
- \`<REDACTED-EMAIL>\` — Klarna promos
- \`<REDACTED-EMAIL>\` — AI newsletter
- \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\` — Vrbo deals, Robinhood notifications
- \`<REDACTED-EMAIL>\` — Stork publication alerts
- \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\` — product newsletters
- \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\` — DMARC aggregate report senders
- \`caseinterview.com\` — MBA/case interview marketing
- \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\` — life sci vendor marketing
- \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\` — drug discovery vendor marketing
- \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\` — conference booth / event marketing
- \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\` — bulk mail senders via shared1.ccsend.com
- \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\` — biotech newsletters
- \`<REDACTED-EMAIL>\` — UVA mass mail
- \`<REDACTED-EMAIL>\` — Google Cloud event invites
- \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\` — Scientist.com newsletters and webinars
- \`<REDACTED-EMAIL>\` — Inside Immunology / AAI newsletter
- \`<REDACTED-EMAIL>\` — Boster's own outbound marketing emails (CJ is CC'd, not addressee)
- \`<REDACTED-EMAIL>\` — R&D Systems marketing
- \`<REDACTED-EMAIL>\` — Zoho Social post-live automated notifications
- \`<REDACTED-EMAIL>\` — IZFS newsletter and webinar announcements
- \`<REDACTED-EMAIL>\` — Clay sales/marketing
- \`<REDACTED-EMAIL>\` — real estate newsletter (Built Different)
- \`<REDACTED-EMAIL>\` — Taconic Biosciences vendor marketing
- \`iluv.southwest.com\` — Southwest Airlines promo
- \`<REDACTED-EMAIL>\` — AACR Foundation fundraising emails
- \`bishealthcarereport.com\`, \`bisresearchinsights.us\` — BIS market research vendor
- \`us.goldstandarddiagnostics-info.com\` — Gold Standard Diagnostics vendor marketing
- \`<REDACTED-EMAIL>\` — BioSistemika vendor marketing
- \`citeab.co.uk\` — CiteAb listing/renewal marketing
- \`<REDACTED-EMAIL>\` — involve.me product newsletter
- \`alphatheranews.com\` — AlphaThera vendor marketing (AACR follow-ups)
- \`<REDACTED-EMAIL>\` — Semrush EyeOn weekly newsletter
- \`<REDACTED-EMAIL>\` — Airbnb promotional (distinct from \`<REDACTED-EMAIL>\` booking confirmations)
- \`<REDACTED-EMAIL>\` — Airbnb automated booking/review notifications
- \`<REDACTED-EMAIL>\` — Expedia promotional
- \`e.ralphsemail.com\` — Ralphs grocery deals
- \`<REDACTED-EMAIL>\` — UVA reunion fundraising
- \`<REDACTED-EMAIL>\` — ZHA zebrafish mailing list (community discussion, no CJ action needed)
- \`<REDACTED-EMAIL>\` — SfN virtual events newsletter
- \`abterrabio.com\` — Abterra Biosciences conference marketing
- \`mail.adobe.com\` — Adobe marketing
- \`bourne-partners.ccsend.com\` — Bourne Partners healthcare M&A newsletter
- \`<REDACTED-EMAIL>\` — Jolly SEO newsletter
- \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\` — SciLeads automated exports and newsletters
- \`biomedica-cro.com\` — Biomedica CRO vendor marketing
- \`croyezbio.com\` — Croyez Bioscience vendor marketing
- \`<REDACTED-EMAIL>\` — AWS marketing webinars
- \`<REDACTED-EMAIL>\` — Upwork newsletter/platform updates
- \`eml.labcorp-mail.com\` — Labcorp vendor marketing
- \`mail.biz-genius.com\` — BizGenius AI tool marketing
- \`<REDACTED-EMAIL>\` — EcoVadis webinar marketing
- \`sanramonmedctr.com\` — San Ramon Regional Medical Center newsletter
- \`mail.creditkarma.com\` — Credit Karma promotional
- \`mirasvit.com\` — Mirasvit Magento extension newsletter
- \`thermofisher.com\` — Thermo Fisher Scientific marketing (promo/events)
- \`searchapi.io\` — SearchApi changelog newsletter
- \`apollo.io\` — Apollo.io automated notifications (CSV exports, task reminders)
- \`pluto.bio\` — Pluto Bio event invites (MultiOmics Mixer etc.)
- \`<REDACTED-EMAIL>\` — Priceline/Barclays Rewards notifications
- \`<REDACTED-EMAIL>\` — Figma renewal/billing notifications
- \`levelup-sdr.com\` — LevelUp SDR sales pitch
- \`increaseleadconversion.com\` — LinkDoctor SEO / link-building cold pitch (recurring multi-touch thread)

**Subject patterns (substring match):**
- "DMARC", "Report Domain:", "Report domain:" — DMARC reports
- "New Order #", "PO26", "Purchase Order" — transaction / order notifications (CJ is CC'd for info only)
- "Your receipt from", "Your statement is now available" — utility billing
- "Zoho SalesIQ: Daily report", "Missed Chat", "Chat :" (from SalesIQ) — SalesIQ alerts (CJ info-only)
- "boster -booster -author:boster - new results" — Scholar Alert
- "Some fixes failed for Page indexing", "We're validating your Page indexing" — automated GSC
- "Veo is here", "Get the most out of your new YouTube Ads", "Grow revenue with Stripe", "You're invited: Best of Next" — vendor promo
- "Want Ongoing Revenue", "Referral Partner Program" — SEO vendor sales pitch
- "Save on stays", "Open and ready to Surf", "Glow with Youthful", "Transform Your Look", "AMC Black Ticket" — Vrbo/Groupon/deal promos
- "Every Platform Wants Video", "Why Smart Bioprocessing", "Why does a 95% hit rate", "What's slowing your design" — vendor marketing
- "The Most Overlooked Skill in Top MBA" — MBA marketing
- "Last chance to support Student Affairs", "Reminder: UVA Google Account" — UVA fundraising/admin
- "Earn rewards for what you already do", "Pay in a way that suits you" — loyalty/fintech promo
- "[ngrok news]", "Signals: Real-time buying intent" — tools newsletters
- "Is AI talking behind your back", "AI is now deciding what your emails say" — SEO/AI newsletter
- "If AI doesn't cite you", "Upcoming Webinar" (from marketing domains) — webinar marketing
- "REGISTER NOW", "CURRENTLY ON SALE", "NEW!" (from vendor domains) — life sci marketing
- "Thank You for Visiting", "Nice to connect after AACR" (from vendor domains) — post-conference marketing
- "5 Steps to Get You Ready for Abstract Submission" — conference abstract marketing
- "Call for Zebrafish Images", "Meet us at ISCT", "Accelerate Your Gene Delivery" — life sci marketing
- "Reminder to Share Your Feedback on the AACR Annual Meeting" — AACR survey (automated post-conference)
- "Monitor is UP:", "Monitor is DOWN:" — UptimeRobot automated alerts (informational; UP/DOWN pairs resolve each other)
- "New Sales Inquiry Received in Zoho CRM" when From is <REDACTED-EMAIL> — CJ's own CRM forwarding, not inbound action
- "Successful CSV Export" — Apollo automated export notification
- "Notify Marketing Team" (from <REDACTED-EMAIL>) — Google Calendar alert (already on calendar)
- "Boster Monthly Meeting" (from <REDACTED-EMAIL>) — Google Calendar alert
- "Your Crunchbase Updates" — Crunchbase digest newsletter
- "Latest Intelligence" (from scileads.com) — SciLeads newsletter
- "SearchApi Changelog" — SearchApi newsletter
- "Application Confirmation" + AACR — MapYourShow automated exhibitor confirmation
- "Get Started with Community-Based Algal Toxin" — Gold Standard Diagnostics marketing
- "San Diego MultiOmics Mixer" — Pluto Bio event invite
- "Benign Conditions Biospecimens" — Biomedica CRO marketing
- "Visualize Mitochondria" — Croyez Bioscience marketing
- "T5 Plant Lighting" — off-topic vendor (plant lighting, not life sci)
- "How does CSPM support" — AWS security webinar marketing
- "PDF tools that play well" — Adobe marketing
- "Rethink development toxicity" — Labcorp marketing
- "Your ChatGPT Brand Audit" — SEO newsletter (Jolly SEO)
- "Recent Healthcare Trends" — Bourne Partners newsletter
- "AI-Enhanced Histology" — Scientist.com webinar
- "Ask Your Citations" — BizGenius marketing
- "What's new at involve.me" — involve.me newsletter
- "Get your exclusive Mortal Kombat" — Microsoft Rewards promo
- "Nucleic Acid Sample Prep" — BIS Research market report marketing
- "Weekly Status Call" (from Wenhong You / immigration) — calendar invite, not action for CJ
- "5 Stroke Warning Flag Symptoms" — medical center newsletter
- "CYTO 2026 Exhibition Update" — CYTO conference marketing (Tristan Herlitz / LSE)
- "Re: [boster00/bosterbio.com2026]" (from <REDACTED-EMAIL> or vercel[bot]) — GitHub/Vercel PR bot notifications (CJ monitors via GuildOS)

## Data: Patterns to Highlight

> ⚠ RETIRED 2026-06-20. The model no longer stars anything — the keep-set is defined entirely by "Data: Reply criteria" (Category 1 LEAVE BE / Category 2 FORWARD). The protectandgrow.blog lead is now handled as the Category-1 exception in "Data: Reply criteria". This section is kept only as historical context; do NOT use it as a classifier.

Entries here must be decidable from Subject + sender category alone — no body read required to star. If confirmation requires reading the body, the email belongs in the ambiguous set (step 3), not here.
No contact-specific names or email addresses — patterns describe content and sender type only.

- **Subject: "Upcoming Summer Schedule", "[School Name]_Summer", "Reminder: … School"** from a school or childcare domain, To personal email → schedule/registration action required; star immediately
- **Subject contains "retirement income review request" + "protectandgrow.blog"** — an inbound contact-form lead from the protectandgrow.blog retirement blog (relayed via a form backend; the exact relay is swappable, the SUBJECT is owner-controlled and stable). A real reader requesting a retirement income review → star immediately, NEVER mark-read. Match on SUBJECT (the From is the shared sender \`Boster Nexus <<REDACTED-EMAIL>>\` - delivered by the Boster Nexus warning system via Zoho Mail; the subject is owner-controlled + stable, so it's the deterministic key). This takes precedence over the \`*@bosterbio.com\` internal-sender entry in "Patterns to Ignore"; the matching "Leave-unread overrides" entry guarantees it survives the bulk mark-read pass.
- *(This section grows as new clear-signal patterns are confirmed over repeated runs)*

## Data: Leave-unread overrides

> ⚠ RETIRED 2026-06-20 as a decision surface. Under the new model the ONLY mail left unread is "Data: Reply criteria" Category 1 (genuine human mail directly to a CJ/Sijie identity) + the protectandgrow.blog lead exception. Security/auth alerts, invoices, edit-requests etc. are no longer special-cased — if they are not Category 1 or 2 they get marked read. Kept as historical context only; do NOT use as a classifier.

Messages matching ANY of these MUST stay unread even when they also match a Patterns-to-Ignore entry. The bulk mark-as-read pass (step 3) must exclude them. Rationale: a user-actionable signal hiding inside a noisy sender domain is worse than a few extra unread items.

**Security / auth (user action may be required):**
- From \`<REDACTED-EMAIL>\` with Subject containing "Security alert", "New sign-in", "Critical security alert", "2-Step Verification was disabled", "Password changed"
- From \`<REDACTED-EMAIL>\` — single-use codes, sign-in verifications
- From \`<REDACTED-EMAIL>\` with Subject "2-Step verification was disabled" or "multi-factor authentication" (account-security state changes, not promo)
- From \`<REDACTED-EMAIL>\` with Subject containing "is requesting updated permissions", "deploy key was added", "new SSH key", "personal access token"

**Action-required from a real person/account:**
- From \`<REDACTED-EMAIL>\` with Subject containing "Edit request from" (collaborator wants access — distinct from billing notifications)
- From a named human (not a marketing list) addressed directly to CJ in TO (not CC), regardless of sender domain — handled by the existing Hard-skip rules (a)/(b) inversion: if CJ IS in direct TO and message is not auto-generated, it does not match Patterns to Ignore in the first place

**Inbound website leads (contact-form submissions):**
- Subject containing "retirement income review request" + "protectandgrow.blog" — a contact-form lead from the protectandgrow.blog retirement blog, delivered From \`Boster Nexus <<REDACTED-EMAIL>>\` (the Boster Nexus warning system via Zoho Mail). MUST stay unread AND get starred (mirror of the "Patterns to Highlight" entry). A real reader's lead arriving from a shared \`@bosterbio.com\` address would otherwise be swept up by the \`*@bosterbio.com\` internal-sender ignore; this override prevents that.

**Work invoices / billing where the user is the payer:**
- From \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\`, vendor invoice senders the user pays directly — Subject containing "Invoice", "Statement", "Payment Renewal Ahead", "Renewal Notice"
- From \`<REDACTED-EMAIL>\` Subject containing "Payment Renewal Ahead"

Pattern growth: new override entries enter this section via blacksmith.patchSkillBook with evidence ≥1 missed-action incident (override entries earn the bar at ≥1 because false negatives here cost more than false positives).

## Data: Unsubscribe (active)

> Policy change 2026-06-20: ACTIVELY unsubscribe — do NOT merely harvest a link for the user to click. For senders in [noise floor] that are clearly unsubscribable marketing, the agent performs the one-click unsubscribe itself (the user no longer wants a to-do list of links). Triggered ONCE per sender per inbox lifetime — once unsubscribed, the sender goes on the "Unsubscribed" Gmail label and is never re-processed.

"Clearly unsubscribable" = the message exposes a \`List-Unsubscribe\` header with an http(s) URL (best case: \`List-Unsubscribe-Post: List-Unsubscribe=One-Click\`, RFC 8058). Those are safe to auto-action. mailto-only or no-link senders are NOT auto-unsubscribed — report them as "manual" (reply STOP / block sender is the user's call).

**Trigger:**
- Sender hit the inbox ≥2 times in the past 30 days (verify via \`gmail.searchInbox\` \`from:<sender> newer_than:30d\`)
- Sender is NOT on Gmail label \`Unsubscribed\` (created lazily; check via \`gmail.list_email_labels\` + filter)
- Sender domain is NOT on "Patterns to never unsubscribe" below

**Patterns to never unsubscribe** (keep receiving even though it's high-volume):
- \`<REDACTED-EMAIL>\` — own company's outbound, CJ should see what's going out
- \`*@bosterbio.com\` internal senders — even on automated/transactional emails
- DMARC / Mimecast / NIH postmaster — operational signals (mark-as-read is fine; don't unsubscribe)
- Lillio \`<REDACTED-EMAIL>\` — Natalie's daycare; user chooses cadence elsewhere, not via unsubscribe
- Robinhood, Capital One, Apple, PayPal — first-party account email (statements, receipts)
- Asana / GitHub / Vercel / Calendly / Zoom / Figma operational notifications

**How to unsubscribe** — extract once, then act:
1. Collapse [noise floor] to one Gmail message ID per qualifying sender (most recent message wins; apply the Trigger + "never unsubscribe" filters first).
2. Call \`gmail.extractUnsubscribeLinks(messageIds)\` — the weapon fetches raw payloads via Gmail REST and returns the slim slice (List-Unsubscribe header URLs/mailtos + body anchors matching the unsubscribe regex). It does NOT load full bodies into agent context.
3. Reduce with \`gmail.pickBestUnsubscribeLink(results)\` — one best URL per sender (header > anchor > mailto).
4. ACT on each best link:
   - **RFC-8058 one-click** (message also carries \`List-Unsubscribe-Post: List-Unsubscribe=One-Click\`): HTTP \`POST\` to the http(s) URL with body \`List-Unsubscribe=One-Click\` (Content-Type \`application/x-www-form-urlencoded\`).
   - **http(s) link, no one-click POST advertised:** HTTP \`GET\` the URL. Treat 2xx/3xx as success.
   - **mailto: only, or no link found:** do NOT auto-action — record as "manual".
   Use a plain HTTP call (WebFetch / curl). Record the result code per sender.
5. For each sender successfully unsubscribed, apply the \`Unsubscribed\` Gmail label (\`get_or_create_label\` → \`batch_modify_emails addLabelIds\`) to one of its messages so future runs skip it.

**Output format** in chat report:

\`\`\`
unsubscribed (N done, M manual):
| # | Sender | Subject | Method | Result |
|---|--------|---------|--------|--------|
| 1 | <name> \`<email>\` | <subject snippet> | one-click POST / GET / manual | 200 / 302 / mailto-only |
...
\`\`\`

Implementation: \`extractUnsubscribeLinks\` and \`pickBestUnsubscribeLink\` are exports of \`libs/weapon/gmail\` (header rationale: bulk header/anchor parsing belongs in code because the MCP \`read_email\` decoded bodies are routinely 100k+ chars and bust agent context). The header parse must also expose whether \`List-Unsubscribe-Post\` is present so step 4 can choose POST vs GET. CLI wrapper: \`scripts/_extract-unsubscribe-links.mjs\` + \`scripts/_summarize-unsubscribe.mjs\` for ad-hoc runs outside the daily flow.

## Data: Main dashboard API

The designated **main dashboard** (watchdog-data-sync-architecture.md §9–10) is the
boster_nexus Watchdog main dashboard. Step 6c reads it via the boster_nexus
Dashboards-module assembler (built in quest steps 11–13):

- **Endpoint:** \`GET <NEXUS_DASHBOARD_ORIGIN>/dashboards/view/api?slug=<MAIN_DASHBOARD_SLUG>\`
- **Config-driven host:** \`NEXUS_DASHBOARD_ORIGIN\` const (env override
  \`NEXUS_DASHBOARD_ORIGIN\` or \`BOSTER_NEXUS_ORIGIN\`), default
  \`http://localhost:3001\` (boster_nexus dev origin per the module standard, §11).
  Slug = \`MAIN_DASHBOARD_SLUG\` (env override \`MAIN_DASHBOARD_SLUG\`), default
  \`watchdog-main\`. Built by \`mainDashboardApiUrl()\`. **Never hardcode the host
  at the call site** — always go through the const / \`checkMainDashboard()\`.
- **Response shape (assembler):** \`{ slug, panels: [ { id?, title?, score,
  signals: [...], status, html? } ] }\`. The sync-health panel is workflow-owned;
  Dailies only **reads** the self-evaluation — it does NOT recompute health.
- **Review rule:** headline = lowest panel \`score\`; warnings = any panel/signal
  whose status/level matches degraded / critical / fail / warn / stale / error /
  down / unhealthy (e.g. surfaces "sync currentness", "line-item integrity",
  "failed-verification" warnings into the Step-1 output).
- **Graceful degradation:** if the API is unreachable / non-2xx / unparseable,
  \`checkMainDashboard\` returns \`ok:false\` with an explicit
  \`"main dashboard health: UNAVAILABLE — …"\` line and **does not throw** — Step-1
  email/LinkedIn are never blocked by this hook.

Implementation: \`dailies.checkMainDashboard\` exported from this skill book module
(\`libs/skill_book/dailies/index.js\`).
`.trim(),

  triageEmailStep2: `
**Name:** Mark skippable emails from remaining unread inbox

**Context:** Second pass over the unread inbox after Step 1 has finished and the user has reviewed it. Identifies low-priority emails matching shared-mailbox, automated-report, utility, or marketing patterns and stars them so the user's starred view contains both must-reads (from Step 1) and skippables in one review pass. Does NOT delete or archive — only labels for user batch action.

**Action plan:**

1. Use gmail.searchInbox to fetch residual unread (filter constants per "Data: Filter" below)
2. Classify each message SKIP / KEEP using "Data: Skip signals" below
3. Use gmail.writeStars on the SKIP set → [starred skippables]
4. [MA-direct] Compose chat report covering scanned count, starred-as-skip count, and category breakdown → [step 2 report]

**Deliverables:**

| name | asset link/details | verification criteria | eval status | fail logs |
|---|---|---|---|---|
| starred skippables | Gmail STARRED label applied to message IDs from SKIP set | All starred emails match a Skip signal; no must-read pattern from Step 1 is starred; no emails deleted or archived | pending | - |
| step 2 report | final chat response text | Report contains scanned count, starred-as-skip count, category breakdown | pending | - |

**Realistic:** Recurring daily proven action, paired with triageEmailStep1.

**Time-bound:** Synchronous, single pass. ⛔ Must NOT run automatically after triageEmailStep1 — user reviews step 1 before this fires. ⛔ Do NOT delete or archive permanently.

---

## Data: Filter

Residual filter query (excludes step 1's starred set): \`in:unread in:inbox -label:starred\`

## Data: Skip signals

A message is SKIP if ANY apply:
- Shared mailbox sender: \`orders@\`, \`account@\`, \`support@\`, \`info@\`, \`sales@\`, \`products@\`, \`project@\`
- Automated report: DMARC, Search Console, Bing Webmaster
- Utility / billing: PG&E, energy reports
- Marketing / newsletter / webinar / conference
- "New Order #" transaction notifications
- \`<REDACTED-EMAIL>\`, \`<REDACTED-EMAIL>\`

KEEP = default. Only explicitly classify SKIP. No deletion or archive in this pass.
`.trim(),

  checkLinkedIn: `
**Name:** Daily LinkedIn check — messages, notifications, connection requests

**Specific (deliverables):**
- SEO/GEO-related messages emailed to \`<REDACTED-EMAIL>\` with subject prefix \`[LinkedIn → SEO/GEO]\`
- Chat output with four labeled lines: messages summary, noteworthy bulleted list, notifications summary, connection requests list

**Measurable (success criteria):**
- Every unread thread is opened and bucketed (SEO/GEO / Noteworthy / Skip)
- Every SEO/GEO message has a digest email recorded as sent
- Notifications scan covers everything beyond routine likes/reactions
- Connection requests listed with name + headline
- CIC tab closed when done

**Actionable:**

1. cic.openTab  — establish CIC session (variable: see "Data: URLs" below)
2. cic.readPage on Messages URL  — bucket each unread thread per "Data: Bucketing rules" below
3. For each SEO/GEO thread: gmail.send  — digest email to Rienzi (subject prefix \`[LinkedIn → SEO/GEO]\`)
4. cic.readPage on Notifications URL  — surface anything beyond routine likes/reactions
5. cic.readPage on Connection Requests URL  — list names + headlines
6. cic.closeTab
7. [MA-direct] Compose chat output per "Data: Output format" below

**Realistic:** Recurring daily action, integrated into triageEmailStep1 flow.

**Time-bound:** Synchronous, single pass. Runs after email triage or standalone.

---

## Data: URLs

- Messages (unread filter): \`https://www.linkedin.com/messaging/?filter=unread\`
- Notifications: \`https://www.linkedin.com/notifications/\`
- Connection requests: \`https://www.linkedin.com/mynetwork/invitation-manager/\`

LinkedIn has no MCP. Use CIC throughout. The user's main Chrome is already logged in.

## Data: Bucketing rules

- **SEO / GEO** = search ranking, GEO, AI search visibility, backlinks, schema markup, SERP, vendor pitches in this space → route to Rienzi
- **Noteworthy** = genuine intro from real contact, customer question, business opportunity outside SEO/GEO → surface in chat
- **Skip** = recruiter spam, generic sales, capital raises, funding offers, contact-list sales, connection-request banter with no substance → ignore silently

## Data: Output format

\`\`\`
linkedin messages: <N unread, X emailed to Rienzi, Y noteworthy>
- <sender> — <topic> — <ask>
- ...
notifications: <summary of anything beyond likes>
connection requests: <N pending>
- <name> — <headline>
- ...
\`\`\`
`.trim(),

  mergeInsight: `
**Name:** Place a new user-correction insight into its proper home

**Specific (deliverables):**
- Insight inserted (or edited in place) in exactly one home: weapon code, skill book howTo, adventurer system_prompt, or CLAUDE.md Insights buffer
- If a skill book or adventurer profile changed, an initAgent followup is queued

**Measurable (success criteria):**
- Insight lives in exactly one place — no duplicates
- Placement matches the decision tree (see "Data: Decision tree" below) first match
- If skill book/system_prompt changed, re-init is scheduled

**Actionable:**

1. Apply "Data: Decision tree" below to the new insight — first match wins
2. Use the matching write path:
   - Weapon code/comment → file edit via blacksmith.patchSkillBook or direct weapon edit (commit immediately)
   - Skill book howTo → blacksmith.patchSkillBook
   - Adventurer system_prompt → DB update + roster.spawnAgent re-init followup
   - Global / ambiguous → append timestamped entry to CLAUDE.md Insights buffer
3. Verify no duplicate exists; edit in place if found

**Realistic:** Recurring on every user correction event. Decision tree fine-tuned via EL.

**Time-bound:** Synchronous, single insight per dispatch. ASAP.

---

## Data: Decision tree (first match wins)

1. **Deterministic data-layer behavior** (auth refresh, retries, I/O format enforcement, error translation — anything that runs the same way every time with no judgment)
   → **weapon** (function code or weapon's module-level comment)

2. **A process or tactic that anyone in a given role should follow** (e.g. "every questmaster should check screenshot count matches WBS bullets")
   → **skill book** — inside the relevant action's \`howTo\`, or at top-level description if it spans multiple actions

3. **Specific to one adventurer's judgment, personality, or private constraints** (e.g. Cat's threshold for "unresponsive"; a tone rule)
   → **adventurer.system_prompt** (DB update + trigger re-initAgent)

4. **Cross-cutting rules or context that applies across roles / situations** OR **ambiguous — doesn't clearly fit 1–3**
   → **global** — append timestamped entry to CLAUDE.md Insights buffer. Format: \`- [YYYY-MM-DD] <insight>\`. Next curateInsightsBuffer pass handles promotion.

## Data: Tie-breakers and grey zones

- Insight spans multiple actions in one skill book → skill book top-level description
- Insight applies to multiple skill books → global (Insights buffer)
- OAuth/auth flows that touch both layers → split: weapon owns token lifecycle; skill book owns human-in-the-loop handling
- Insight that clarifies the ownership protocol itself → update this skill book's howTo
- Time-sensitive / situational insights → always to Insights buffer; dropped on next curation pass
`.trim(),

  curateInsightsBuffer: `
**Name:** Curate CLAUDE.md Insights buffer — promote or drop each entry

**Specific (deliverables):**
- Each entry in the buffer either relocated to its permanent home or deleted
- Buffer count reduced toward zero
- A short summary line per moved entry (source → destination)

**Measurable (success criteria):**
- Every buffer entry processed (none left untouched)
- Entries idle 14+ days without an obvious home are deleted (presumed situational)
- Resulting buffer reflects only entries that genuinely belong nowhere yet
- No duplicate creation in destination locations

**Actionable:**

1. Read CLAUDE.md Insights buffer entries
2. For each timestamped entry:
   - Apply dailies.mergeInsight decision tree to choose its home
   - Write to that home (skill book howTo, adventurer profile, or weapon)
   - Remove the entry from the buffer
   - If the entry is 14+ days old and has no clear home, delete it (situational)
3. [MA-direct] Report summary: entries processed, promoted, deleted, remaining.

**Realistic:** Periodic recurring action. Run when buffer has 3+ entries OR weekly, whichever first.

**Time-bound:** Synchronous, single pass. ASAP.
`.trim(),

  disciplineSmokeTest: `
**Name:** Discipline smoke test (autonomous EL stub plan)

**Context:** This action exists solely for autonomous EL validation of the GuildOS execution substrate. Exercises the full MA-runner / SA-execute / SA-verify / SA-report / MA-asset-archive lifecycle without any real-world side effects. Every step writes a small scratch file under \`/tmp/discipline-test/\` or appends to a marker log. Used by the orchestrator (Pig running EL loops) to confirm cascade discipline holds before running real-data smoke tests. NOT a user-facing daily routine.

**Action plan:**

1. Native: ensure directory \`/tmp/discipline-test/\` exists and write \`step-1-output\` to \`/tmp/discipline-test/step1.txt\` → [step 1 artifact]
2. Native: read [step 1 artifact], append its content + \`step-2-output\` to \`/tmp/discipline-test/step2.txt\` → [step 2 artifact]

[parallel]
3a. Native: write \`step-3a-output\` to \`/tmp/discipline-test/step3a.txt\` → [step 3a artifact]
3b. Native: write \`step-3b-output\` to \`/tmp/discipline-test/step3b.txt\` → [step 3b artifact]
[/parallel]

4. [MA-direct] Compose a summary listing all four scratch-file paths produced, their byte sizes, and a one-line confirmation that the discipline lifecycle ran end-to-end → [summary report]

**Deliverables:**

| name | asset link/details | verification criteria | eval status | fail logs |
|---|---|---|---|---|
| step 1 artifact | /tmp/discipline-test/step1.txt | File exists; content is exactly \`step-1-output\` (no trailing whitespace beyond a final newline) | pending | - |
| step 2 artifact | /tmp/discipline-test/step2.txt | File exists; content contains both \`step-1-output\` and \`step-2-output\` substrings | pending | - |
| step 3a artifact | /tmp/discipline-test/step3a.txt | File exists; content is exactly \`step-3a-output\` | pending | - |
| step 3b artifact | /tmp/discipline-test/step3b.txt | File exists; content is exactly \`step-3b-output\` | pending | - |
| summary report | chat response text returned by MA-runner | Lists all 4 scratch file paths + byte sizes; ends with one-line confirmation \`discipline lifecycle complete\` | pending | - |

**Realistic:** Stub plan, proven via autonomous EL loop iterations. Not a user-facing routine.

**Time-bound:** Synchronous, single pass, ASAP. ⛔ Do NOT delete the scratch files after — leaving them lets a subsequent EL iteration verify idempotence.
`.trim(),

  monitorSupportFollowups: `
**Name:** Monitor support inbox for insufficient answers and stale customer threads

**Context:** Daily monitoring routine across the team's customer-routed mailboxes (<REDACTED-EMAIL>, <REDACTED-EMAIL>, <REDACTED-EMAIL>, <REDACTED-EMAIL> — see "Data: Mailbox scope" below). Surfaces two miss types so the user can intervene before customers churn: (Miss A) team replies that didn't fully answer the customer's question; (Miss B) threads where the customer hasn't responded and the team should follow up, or where the team owes the customer a reply and is past the staleness threshold. Output is a labeled set of flagged threads in Gmail plus a two-section chat report the user can scan daily. Mailbox scope, lookback window, and follow-up threshold are locked constants — see Data sections below.

**Action plan:**

1. Use gmail.searchThreads on each mailbox in "Data: Mailbox scope" below, lookback per "Data: Lookback window" → [support thread set]
2. Use gmail.readThread on each thread in [support thread set] (ourDomains: ["bosterbio.com"]) → [thread transcripts]
3. Apply "Data: Support skip patterns" against [thread transcripts] to drop resolved, marketing, internal-only → [candidate threads]

[parallel]
4a. Use dailies.checkSupportInsufficient on [candidate threads] → [insufficient answer flags]
4b. Use dailies.checkSupportNoReply on [candidate threads] (threshold per "Data: Follow-up threshold") → [no reply flags]
[/parallel]

5. Use gmail.writeLabels to apply \`SupportFollowup/InsufficientAnswer\` to threads in [insufficient answer flags] and \`SupportFollowup/NeedsCustomerReply\` to threads in [no reply flags] → [labeled support set]
6. For each candidate new skip / scoring pattern surfaced: collect evidence (≥2 occurrences in past 6 months via gmail.searchThreads) and dispatch blacksmith.patchSkillBook with the proposal → [pattern updates]
7. [MA-direct] Compose the two-section chat report combining [insufficient answer flags] and [no reply flags] — per-thread row with Gmail thread URL, customer, last team reply 1-line summary, recommended follow-up action → [chat report]

**Deliverables:**

| name | asset link/details | verification criteria | eval status | fail logs |
|---|---|---|---|---|
| support thread set | Gmail thread IDs returned by gmail.searchThreads, grouped per mailbox | Count reported as exact integer per mailbox; each ID is a Gmail thread ID; all 4 mailboxes were queried | pending | - |
| thread transcripts | In-memory thread objects with ordered messages and direction tags | Every thread in [support thread set] has a transcript; each message carries from-address and direction tag (team / customer) | pending | - |
| candidate threads | Filtered subset after applying support skip patterns | Skip-pattern hits logged with reason; no resolved/marketing/internal-only threads remain in the candidate set | pending | - |
| insufficient answer flags | List of thread IDs scored as Miss A with one-line reason per thread | Each flagged thread cites the unanswered question or dissatisfaction signal; no thread appears in both Miss A and Miss B flags | pending | - |
| no reply flags | List of thread IDs scored as Miss B with one-line reason | Each flagged thread cites last-team-message timestamp and elapsed business days past threshold; last message direction recorded | pending | - |
| labeled support set | Gmail labels \`SupportFollowup/InsufficientAnswer\` and \`SupportFollowup/NeedsCustomerReply\` applied to flagged thread IDs | Every flagged thread carries the correct label; label IDs resolved (created if missing) before the modify call | pending | - |
| pattern updates | dailies skill book file diffs applied via blacksmith.patchSkillBook | Each proposal had ≥2 occurrence evidence; each was accepted or rejected with documented reason | pending | - |
| chat report | Final chat response text | Two sections in order (Insufficient Answers / Needs Customer Reply); per-thread row includes Gmail thread URL, customer, last reply summary, recommended action; exact counts reported | pending | - |

**Realistic:** New recurring daily plan built 2026-05-11. Locked parameters per user direction (mailbox scope = all 4, lookback = 14d, threshold = 5 business days). Sub-action scorers (checkSupportInsufficient, checkSupportNoReply) will refine via EL after first runs.

**Time-bound:** Synchronous daily scan, ASAP. ⛔ Do NOT auto-reply on the user's behalf — surface flags only. ⛔ Stop after labeling + chat report; user decides which to action.

---

## Data: Mailbox scope (locked 2026-05-11)
- <REDACTED-EMAIL>
- <REDACTED-EMAIL>
- <REDACTED-EMAIL>
- <REDACTED-EMAIL>

## Data: Lookback window (locked 2026-05-11)
14 days. Query operator: \`newer_than:14d\`.

## Data: Follow-up threshold (locked 2026-05-11)
5 business days (Mon-Fri, no holiday adjustment for v1).

## Data: Support skip patterns
Drop a thread from the candidate set if ANY:
- Last message is an automated bounce / out-of-office
- Thread subject begins with \`Auto-reply:\` / \`Out of office:\` / \`Delivery Status\`
- Sender domain is a known marketing source (extend list via blacksmith.patchSkillBook over time)
- Last message contains a clear resolution signal: "thanks, resolved", "all set", "no further questions", "confirmed received"
- Thread is internal-only (no customer participant — all messages from bosterbio.com)
- (Growth section — patterns added via EL feedback)
`.trim(),

  checkSupportInsufficient: `
Score a single support thread transcript for Miss A — team reply did not fully answer the customer's question.

## Input
- \`threadTranscript\`: object returned by gmail.readThread — \`{ threadId, messages: [...] }\`

## Procedure

1. Walk \`messages\` in order. For each customer→team transition (customer message followed by a team reply), do the following:
   - Extract the customer's questions: sentences ending with "?" OR imperative requests ("please send", "I need", "can you")
   - For each extracted question, scan the team reply body for a corresponding answer (keyword overlap, direct address of the topic)
2. Walk team→customer→team or customer→team→customer transitions to detect dissatisfaction follow-ups.

## Scoring signals (cumulative)
- Customer question goes unanswered in team reply → +5 Miss A
- Customer follow-up message says "you didn't answer X" / "still need X" / "asking again" → +10 Miss A
- Customer says "thanks, resolved" / "all set" → -5 Miss A (resolution signal — likely flips this thread out of Miss A entirely)
- Customer hasn't replied after team's most recent message → no Miss A signal added (use checkSupportNoReply instead)

## Decision
- Total score ≥ 5 → Miss A flag set
- Otherwise → Miss A flag not set

## Output
\`\`\`json
{
  "threadId": "...",
  "missA": true | false,
  "score": <integer>,
  "reason": "<one-line: the unanswered question or dissatisfaction signal that drove the verdict>"
}
\`\`\`

## Rules
- Bias toward Miss B classification when the customer hasn't replied yet — Miss A requires evidence of insufficient answer (the customer pushed back or visibly disengaged).
- Patterns for "resolution signals" and "dissatisfaction phrases" grow over time via blacksmith.patchSkillBook proposals.
- One thread per dispatch. The orchestrating MA-runner fans out across the candidate set.
`.trim(),

  checkSupportNoReply: `
Score a single support thread transcript for Miss B — thread is stale and waiting on either customer reply or team follow-up past the threshold.

## Input
- \`threadTranscript\`: object returned by gmail.readThread — \`{ threadId, messages: [...] }\`
- \`thresholdBusinessDays\`: int (default 5 — locked 2026-05-11)

## Procedure

1. Identify the last message in the thread (\`messages[messages.length - 1]\`).
2. Read its \`direction\` tag and \`internalDate\`.
3. Compute business days elapsed from \`internalDate\` to now (Mon-Fri, no holiday adjustment for v1).
4. Check for a resolution signal in the last message body — if present, no Miss B regardless of elapsed time.

## Decision rules
- Last direction is \`team\` AND elapsed ≥ thresholdBusinessDays → Miss B (waiting on customer; team may want to nudge)
- Last direction is \`customer\` AND elapsed ≥ thresholdBusinessDays → Miss B (team owes follow-up)
- Elapsed < thresholdBusinessDays → no Miss B
- Resolution signal in last message → no Miss B

## Output
\`\`\`json
{
  "threadId": "...",
  "missB": true | false,
  "lastDirection": "team" | "customer",
  "elapsedBusinessDays": <number>,
  "reason": "<one-line: who's waiting, last message timestamp, business days elapsed>"
}
\`\`\`

## Rules
- One thread per dispatch.
- Business days = Mon-Fri (no holidays for v1; refine via EL when needed).
- "Resolution signal" pattern list grows via blacksmith.patchSkillBook proposals.
`.trim(),
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/gmail/extractUnsubscribeLinks.js -->
```js
/**
 * Extract unsubscribe candidates from a set of Gmail message IDs.
 *
 * Deterministic data-layer behavior — exists in the weapon because it's
 * the same parser logic on every call (OAuth refresh, raw-message fetch,
 * RFC 2369/8058 header parse, HTML anchor extraction by regex). Lives
 * outside the per-message MCP loop because: the MCP `read_email` returns
 * the full decoded body (often >100k chars) and busts agent context
 * before the agent can grep for an unsubscribe URL. This function reads
 * the same raw payload but returns only the unsubscribe candidates —
 * orders of magnitude smaller payload, keeps agent context clean.
 *
 * Auth: reuses the gongrzhe gmail-mcp credential files at
 *   ~/.gmail-mcp/gcp-oauth.keys.json  (client_id, client_secret)
 *   ~/.gmail-mcp/credentials.json     (refresh_token)
 * No new OAuth flow — the MCP refresh token already grants `gmail.readonly`.
 *
 * @param {string[]} messageIds  Gmail message IDs (not thread IDs)
 * @param {object}   [opts]
 * @param {string}   [opts.gmailDir]    override creds dir (default ~/.gmail-mcp)
 * @param {function} [opts.fetchImpl]   fetch override (tests)
 * @returns {Promise<Array<{
 *   id: string,
 *   from?: string,
 *   subject?: string,
 *   oneClick?: boolean,           // true if List-Unsubscribe-Post header present
 *   listUnsubUrls?: string[],     // RFC 2369 URLs from List-Unsubscribe
 *   listUnsubMailtos?: string[],  // mailto: tokens from List-Unsubscribe
 *   anchors?: Array<{href:string,text:string}>,  // HTML anchors matching UNSUB_RX
 *   error?: string
 * }>>}
 */
export async function extractUnsubscribeLinks(messageIds, opts = {}) {
  const fs = await import("node:fs");
  const path = await import("node:path");
  const os = await import("node:os");

  const fetchImpl =
    opts.fetchImpl || (typeof fetch !== "undefined" ? fetch : null);
  if (!fetchImpl) throw new Error("no fetch implementation available");

  const gmailDir = opts.gmailDir || path.join(os.homedir(), ".gmail-mcp");
  const keysPath = path.join(gmailDir, "gcp-oauth.keys.json");
  const credPath = path.join(gmailDir, "credentials.json");

  const keys = JSON.parse(fs.readFileSync(keysPath, "utf8"));
  const creds = JSON.parse(fs.readFileSync(credPath, "utf8"));
  const installed = keys.installed || keys.web || keys;

  const tokenRes = await fetchImpl("https://oauth2.googleapis.com/token", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      client_id: installed.client_id,
      client_secret: installed.client_secret,
      refresh_token: creds.refresh_token,
      grant_type: "refresh_token",
    }).toString(),
  });
  if (!tokenRes.ok) {
    throw new Error(`gmail oauth refresh: ${tokenRes.status} ${await tokenRes.text()}`);
  }
  const { access_token: accessToken } = await tokenRes.json();

  const results = [];
  for (const id of messageIds) {
    try {
      const r = await fetchImpl(
        `https://gmail.googleapis.com/gmail/v1/users/me/messages/${id}?format=full`,
        { headers: { Authorization: `Bearer ${accessToken}` } }
      );
      if (!r.ok) throw new Error(`fetch ${id}: ${r.status} ${await r.text()}`);
      const msg = await r.json();
      const headers = msg.payload?.headers || [];
      const from = getHeader(headers, "From") || "?";
      const subject = getHeader(headers, "Subject") || "?";
      const listUnsub = parseListUnsub(getHeader(headers, "List-Unsubscribe"));
      const oneClick = !!getHeader(headers, "List-Unsubscribe-Post");
      const body = collectBody(msg.payload);
      const anchors = extractAnchors(body);
      results.push({
        id,
        from,
        subject,
        oneClick,
        listUnsubUrls: listUnsub.urls,
        listUnsubMailtos: listUnsub.mailtos,
        anchors: anchors.slice(0, 5),
      });
    } catch (e) {
      results.push({ id, error: e.message });
    }
  }
  return results;
}

const UNSUB_RX =
  /unsubscribe|opt[\s\-_]?out|email\s*preferences|manage\s+(your\s+)?preferences|preference\s*center|stop\s+(receiving|these)/i;

function getHeader(headers, name) {
  const h = (headers || []).find(
    (x) => x.name.toLowerCase() === name.toLowerCase()
  );
  return h ? h.value : null;
}

function decode(data) {
  if (!data) return "";
  return Buffer.from(
    data.replace(/-/g, "+").replace(/_/g, "/"),
    "base64"
  ).toString("utf8");
}

function collectBody(payload) {
  const out = [];
  const walk = (p) => {
    if (!p) return;
    if (p.mimeType === "text/html" && p.body?.data) out.push(decode(p.body.data));
    if (p.mimeType === "text/plain" && p.body?.data) out.push(decode(p.body.data));
    (p.parts || []).forEach(walk);
  };
  walk(payload);
  return out.join("\n");
}

function parseListUnsub(value) {
  if (!value) return { urls: [], mailtos: [] };
  const tokens = value.match(/<([^>]+)>/g) || [];
  const urls = [];
  const mailtos = [];
  for (const t of tokens) {
    const v = t.slice(1, -1);
    if (v.startsWith("mailto:")) mailtos.push(v);
    else urls.push(v);
  }
  return { urls, mailtos };
}

function extractAnchors(html) {
  const out = [];
  const re = /<a\b[^>]*?href\s*=\s*(["'])(.*?)\1[^>]*>([\s\S]*?)<\/a>/gi;
  let m;
  while ((m = re.exec(html)) !== null) {
    const href = m[2].trim();
    const text = m[3].replace(/<[^>]+>/g, " ").replace(/\s+/g, " ").trim();
    if (!href || href.startsWith("#") || href.startsWith("mailto:?")) continue;
    if (UNSUB_RX.test(href) || UNSUB_RX.test(text)) {
      out.push({ href, text: text.slice(0, 80) });
    }
  }
  // Plain-text URLs near the word unsubscribe (for text/plain bodies)
  const plain = html.replace(/<[^>]+>/g, " ");
  const lines = plain.split(/\r?\n/);
  for (let i = 0; i < lines.length; i++) {
    if (!UNSUB_RX.test(lines[i])) continue;
    const ctx = lines.slice(Math.max(0, i - 1), i + 2).join(" ");
    const urlRe = /https?:\/\/[^\s)<>"']+/g;
    let u;
    while ((u = urlRe.exec(ctx)) !== null) {
      const url = u[0].replace(/[.,;:!?\)\]]+$/, "");
      if (!out.find((x) => x.href === url)) {
        out.push({
          href: url,
          text: `[plain] ${lines[i].slice(0, 80).trim()}`,
        });
      }
    }
  }
  return out;
}

/**
 * Pick the single best unsubscribe URL per result, ranked: header URL >
 * highest-scoring anchor > mailto. Returns a slim row array suitable for
 * a markdown table or CSV.
 *
 * @param {Array} results  output of extractUnsubscribeLinks
 * @returns {Array<{id,from,sender,subject,url,source,oneClick}>}
 */
export function pickBestUnsubscribeLink(results) {
  const rows = [];
  for (const r of results) {
    if (r.error) {
      rows.push({ id: r.id, from: "?", sender: "", subject: "(error)", url: r.error, source: "err", oneClick: false });
      continue;
    }
    const fromName = (r.from || "").replace(/<.*>/, "").replace(/"/g, "").trim();
    const sender = r.from?.match(/<([^>]+)>/)?.[1] || "";
    let url = r.listUnsubUrls?.[0];
    let source = "header";
    if (!url && r.anchors?.length) {
      const ranked = r.anchors
        .map((a) => {
          let score = 0;
          if (/unsubscribe/i.test(a.text)) score += 10;
          if (/unsubscribe/i.test(a.href)) score += 5;
          if (/opt[\s\-_]?out/i.test(a.text + a.href)) score += 5;
          if (/preferences/i.test(a.text)) score += 2;
          return { ...a, score };
        })
        .sort((a, b) => b.score - a.score);
      url = ranked[0]?.href;
      source = "body";
    }
    if (!url && r.listUnsubMailtos?.length) {
      url = r.listUnsubMailtos[0];
      source = "mailto";
    }
    rows.push({
      id: r.id,
      from: fromName,
      sender,
      subject: (r.subject || "").slice(0, 80),
      url: url || "(none found)",
      source,
      oneClick: !!r.oneClick,
    });
  }
  return rows;
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/gmail/index.js -->
```js
/**
 * Gmail weapon — unsubscribe parsers only.
 *
 * The only runtime logic here is the pair of context-budget-beating
 * unsubscribe parsers. The MCP `read_email` returns the fully decoded body,
 * which on marketing emails is routinely 100k+ chars and exceeds the agent
 * context window before the agent can grep for what it needs. For
 * deterministic parses where the agent only wants a small slice of each
 * message (e.g. unsubscribe URLs), the weapon exports a CLI-callable function
 * that fetches the raw payload via the Gmail REST API directly and returns
 * just the slice. Same OAuth surface (gongrzhe gmail-mcp credential files),
 * no new auth, just a smaller return value.
 *
 * Everything else Gmail-related (MCP setup, OAuth scopes, the full
 * `mcp__gmail__*` tool catalog, triage/read/label procedures) is text and
 * lives in the gmail SKILL BOOK (`libs/skill_book/gmail/index.js`).
 */

export { extractUnsubscribeLinks, pickBestUnsubscribeLink } from "./extractUnsubscribeLinks.js";

export const toc = {
  // Most Gmail actions are MCP tools, not weapon exports. Agents discover
  // those via the MCP server at connection time. Exports below are the
  // context-budget exceptions only (see header comment for rationale).
  extractUnsubscribeLinks:
    "Fetch raw bodies of given message IDs via Gmail REST, return only the unsubscribe candidates per message (List-Unsubscribe header URLs/mailtos + body anchors matching the unsubscribe regex). Same OAuth as MCP; reuses ~/.gmail-mcp/credentials.json. CLI wrapper: scripts/_extract-unsubscribe-links.mjs.",
  pickBestUnsubscribeLink:
    "Reduce extractUnsubscribeLinks output to one best URL per message (header > best anchor > mailto), suitable for a markdown table.",
};
```
<!-- /GUILDOS-FILE -->
