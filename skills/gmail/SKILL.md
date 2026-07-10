# gmail

<!-- GUILDOS-SKILL name="gmail" -->

<!-- GUILDOS-DESC -->
Search, read, triage, and label emails via the gmail MCP server. Use for inbox triage (score unread and star the top ~2%), thread/conversation analysis, follow-up monitoring, and label management. Gmail is agent-driven: call the mcp__gmail__* tools directly. Includes context-budget-beating unsubscribe-link parsers (gmail weapon) for marketing emails whose decoded bodies exceed the context window.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/gmail/SKILL.md -->
```markdown
---
name: gmail
description: >-
  Search, read, triage, and label emails via the gmail MCP server. Use for
  inbox triage (score unread and star the top ~2%), thread/conversation
  analysis, follow-up monitoring, and label management. Gmail is agent-driven:
  call the mcp__gmail__* tools directly. Includes context-budget-beating
  unsubscribe-link parsers (gmail weapon) for marketing emails whose decoded
  bodies exceed the context window.
---

# Gmail

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/gmail/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto gmail <action>` (or read the source directly).

## Actions

- **readMessage** — Read a single message — use mcp__gmail__read_email directly.
- **readThread** — Read all messages in a thread, ordered chronologically, with team-vs-customer direction tags. Use for Miss A/B detection in support workflows.
- **searchInbox** — Search Gmail messages — use mcp__gmail__search_emails directly.
- **searchThreads** — Search Gmail for THREADS (not messages) — wraps search_emails by grouping hits by threadId. Use for thread-level workflows (conversation analysis, follow-up monitoring).
- **setup** — MCP server setup (gongrzhe gmail-autoauth), OAuth scope note, and the full mcp__gmail__* tool catalog (read/modify/delete/write).
- **triageInbox** — Score unread inbox and star the top ~2% via MCP tools search_emails + batch_modify_emails.
- **writeLabels** — Add/remove arbitrary labels — use mcp__gmail__modify_email / batch_modify_emails directly.
- **writeStars** — Add STARRED label — use mcp__gmail__modify_email / batch_modify_emails directly.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/gmail/index.js -->
```js
/**
 * Gmail skill book — pure text registry.
 *
 * Gmail is agent-driven via the `gmail` MCP server
 * (`@gongrzhe/server-gmail-autoauth-mcp`). Agents call `mcp__gmail__*` tools
 * directly; this file only documents intent and the triage scoring rules.
 * No JS action functions here by design.
 */

export const skillBook = {
  id: "gmail",
  title: "Gmail",
  description: "Search, read, triage, and label emails via the gmail MCP server.",

  toc: {
    setup: "MCP server setup (gongrzhe gmail-autoauth), OAuth scope note, and the full mcp__gmail__* tool catalog (read/modify/delete/write).",
    searchInbox: "Search Gmail messages — use mcp__gmail__search_emails directly.",
    searchThreads: "Search Gmail for THREADS (not messages) — wraps search_emails by grouping hits by threadId. Use for thread-level workflows (conversation analysis, follow-up monitoring).",
    readMessage: "Read a single message — use mcp__gmail__read_email directly.",
    readThread: "Read all messages in a thread, ordered chronologically, with team-vs-customer direction tags. Use for Miss A/B detection in support workflows.",
    triageInbox: "Score unread inbox and star the top ~2% via MCP tools search_emails + batch_modify_emails.",
    writeStars: "Add STARRED label — use mcp__gmail__modify_email / batch_modify_emails directly.",
    writeLabels: "Add/remove arbitrary labels — use mcp__gmail__modify_email / batch_modify_emails directly.",
  },

  setup: `
Gmail is agent-driven via the \`gmail\` MCP server
(\`@gongrzhe/server-gmail-autoauth-mcp\`), which wraps the Gmail REST API
through a Google OAuth refresh token. The runtime surface is agent-only:
Claude (local) or a Cursor cloud agent with this MCP mounted calls the
\`mcp__gmail__*\` tools directly. No browser, no CIC. There are no server-code
call sites for routine read/triage/label work — to do ordinary Gmail behavior
from server code, dispatch an agent rather than inlining REST calls.

## Setup (one-time, local)
1. \`~/.gmail-mcp/gcp-oauth.keys.json\` — Google OAuth "installed" client:
     { "installed": { "client_id": <GMAIL_MCP_CLIENT_ID>,
                      "client_secret": <GMAIL_MCP_CLIENT_SECRET>,
                      "redirect_uris": ["http://localhost"] } }
   Credentials live in profiles.env_vars (formulary):
     GMAIL_MCP_CLIENT_ID, GMAIL_MCP_CLIENT_SECRET.

2. \`~/.gmail-mcp/credentials.json\` — refresh-token bundle:
     { "access_token": "",
       "refresh_token": <GOOGLE_GMAIL_REFRESH_TOKEN from formulary>,
       "scope": "https://www.googleapis.com/auth/gmail.readonly
                 https://www.googleapis.com/auth/gmail.modify",
       "token_type": "Bearer",
       "expiry_date": 1 }
   gongrzhe's server auto-refreshes the access token on every call.

3. Register in \`~/.claude.json\` under the GuildOS project:
     "mcpServers": { "gmail": { "type":"stdio",
       "command":"npx", "args":["-y","@gongrzhe/server-gmail-autoauth-mcp"] } }

## Scope note
The stored refresh token grants \`gmail.readonly\` + \`gmail.modify\`.
Read / triage / star / label tools work. \`send_email\` ALSO works on this token
(verified 2026-06-20 — the Gmail API authorizes \`users.messages.send\` under the
\`gmail.modify\` scope, so forwards/replies send fine; the earlier "send needs
broader scopes" claim here was wrong). \`create_filter\` and label-management
(\`create_label\`/\`update_label\`/\`get_or_create_label\`) also work under modify.
Only permanent deletion (\`delete_*\`) needs the full \`https://mail.google.com/\`
scope.

## MCP tool names (all prefixed \`mcp__gmail__\` in agent context)
Read:    search_emails, read_email, list_email_labels, list_filters,
         get_filter, download_attachment
Modify:  modify_email, batch_modify_emails
Delete:  delete_email, batch_delete_emails, delete_label, delete_filter
         (destructive — require user confirmation)
Write:   draft_email, send_email, create_label, update_label,
         get_or_create_label, create_filter, create_filter_from_template
         (blocked until scopes expanded)
`.trim(),

  searchInbox: `
Use \`mcp__gmail__search_emails\` directly with \`{ query, maxResults }\`. Gmail query syntax applies (\`from:\`, \`subject:\`, \`in:unread\`, \`-label:important\`, etc.).
`.trim(),

  searchThreads: `
Search Gmail for THREADS, not individual messages. Wraps \`mcp__gmail__search_emails\` and groups hits by \`threadId\`.

## Inputs
- \`query\`: Gmail query syntax string (e.g. \`to:<REDACTED-EMAIL> newer_than:14d\`)
- \`maxThreads\`: int, max threads to return (default 100)

## Procedure
1. Call \`mcp__gmail__search_emails({ query, maxResults: maxThreads * 5 })\` to fetch message hits with overflow buffer (multiple messages can share a thread).
2. Group results by \`threadId\` — keep the first (most recent) message per thread as the sample.
3. Return up to \`maxThreads\` thread records: \`[{ threadId, sampleMessageId, subject, date }]\`.

## Notes
- Gmail returns messages in reverse-chronological order by default; the first message per thread in the response is typically the most recent.
- For one-off message searches that don't need thread grouping, use \`gmail.searchInbox\` instead.
- Use \`newer_than:<N>d\` for lookback windows.
`.trim(),

  readMessage: `
Use \`mcp__gmail__read_email\` directly with \`{ messageId }\`. Returns headers + parsed body.
`.trim(),

  readThread: `
Read all messages in a Gmail thread, ordered chronologically, with sender direction tags.

## Inputs
- \`threadId\`: Gmail thread ID (from gmail.searchThreads output)
- \`ourDomains\`: array of domains considered "team" (default \`["bosterbio.com"]\`)

## Procedure
1. Query messages in the thread. Use \`mcp__gmail__search_emails({ query: "threadId:<id>", maxResults: 100 })\` — Gmail's query operator returns all messages sharing that threadId.
   - If the operator isn't supported by the MCP version, fall back: fetch broader query and filter by \`message.threadId === threadId\` in-memory.
2. For each message ID, call \`mcp__gmail__read_email\` to get headers + body.
3. For each message, compute direction:
   - \`team\` if the From: address domain is in \`ourDomains\`
   - \`customer\` otherwise
4. Sort messages ascending by \`internalDate\`.
5. Return \`{ threadId, messages: [{ id, from, to, cc, date, internalDate, direction, snippet, body }] }\`.

## Notes
- Direction tags drive Miss A / Miss B detection downstream.
- Multi-recipient messages: direction is determined by the From header only. To/CC carving (e.g. boster@ shared mailbox) happens in the consumer scorer, not here.
- Body length cap: if the MCP returns truncated body for very long threads, log a warning in the return record; downstream scorers can still operate on snippets.
`.trim(),

  triageInbox: `
Goal: scan unread inbox, skip low-value, score the rest, star the top ~2%.

Step 1 — fetch ALL unread in one call, then check for truncation.
  Query: "-label:important in:unread in:inbox -asana"
  NOTE: The gmail MCP tool search_emails only accepts { query, maxResults } — there is NO
  pageToken parameter. Pagination via cursor is not supported by this MCP.

  Call: search_emails({ query, maxResults: 500 })
  Count the returned IDs and report the exact number: "Fetched N emails."
  Do NOT claim "500 emails" unless you actually counted 500 distinct IDs.
  Do NOT cap at 100 — always use maxResults: 500.

  TRUNCATION CHECK: If exactly 500 IDs are returned, the result may be truncated
  (inbox could hold more). In that case, split by date window instead:
    Run separate queries with: after:YYYY/MM/DD before:YYYY/MM/DD (30-day windows back from today)
    Merge all unique IDs across windows before scoring.
  If fewer than 500 are returned, no truncation — proceed directly to Step 2.

Step 2 — skip these (do NOT score):
  • Shared mailboxes: orders@, account@, support@, info@, sales@, products@, project@
  • Specific senders: <REDACTED-EMAIL>, <REDACTED-EMAIL>, anything at asana.com
  • Automated: "New Order #", DMARC reports, Search Console, Bing Webmaster
  • Utility: PG&E, energy reports
  • Marketing: newsletters, webinars, conferences, anything with "unsubscribe"

Step 3 — score remaining (signals → points):
  +10  wire transfer / invoice / purchase order / freight quote
  +9   Calendly new or updated event
  +9   customer reply to a quote (subject starts "Re: ... quote")
  +8   SalesIQ / live chat / missed chat
  +8   security alert (new sign-in, 2FA, verification code)
  +7   Boster sales order
  +7   meeting request directed at CJ / Sijie
  +3   addressed directly to CJ / Sijie
  +1   real sender (not noreply / marketing / newsletter)

Step 4 — star the top ~2%.
  starCount = max(1, ceil(messages.length * 0.02))
  MCP: batch_modify_emails { messageIds: [...top N...], addLabelIds: ["STARRED"] }

Decision framework when unsure:
  1. Addressed directly to CJ/Sijie? No → skip.
  2. Team member already replied and CJ was just CC'd? → skip.
  3. Automated / system-generated? → skip.
  4. Requires CJ's personal decision? → star.
  5. When in doubt, do NOT star. A clean inbox is preferred.
`.trim(),

  writeStars: `
Use \`mcp__gmail__batch_modify_emails\` directly with \`{ messageIds, addLabelIds: ["STARRED"] }\` (or \`modify_email\` for a single message).
`.trim(),

  writeLabels: `
Use \`mcp__gmail__modify_email\` / \`batch_modify_emails\` directly: \`addLabelIds: [...]\` to add, \`removeLabelIds: [...]\` to remove. Resolve label IDs first via \`list_email_labels\` or \`get_or_create_label\`.
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
