# zoho

<!-- GUILDOS-SKILL name="zoho" -->

<!-- GUILDOS-DESC -->
Search Zoho Books/CRM modules and read Zoho Mail inbox via shared OAuth. Unified search across Zoho Books (sales orders, invoices, bills, ...) and Zoho CRM (Contacts, Quotes, Leads, Deals), plus read the authenticated user's Zoho Mail accounts, recent messages, and full message bodies (e.g. verification/OTP codes). Mail is per-user: the API only reads the authenticated user's OWN mailbox. Executable dispatch/OAuth lives in the zoho weapon.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/zoho/SKILL.md -->
```markdown
---
name: zoho
description: >-
  Search Zoho Books/CRM modules and read Zoho Mail inbox via shared OAuth.
  Unified search across Zoho Books (sales orders, invoices, bills, ...) and Zoho
  CRM (Contacts, Quotes, Leads, Deals), plus read the authenticated user's Zoho
  Mail accounts, recent messages, and full message bodies (e.g. verification/OTP
  codes). Mail is per-user: the API only reads the authenticated user's OWN
  mailbox. Executable dispatch/OAuth lives in the zoho weapon.
---

# Zoho

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/zoho/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto zoho <action>` (or read the source directly).

## Actions

- **readMailAccounts** — Read Zoho Mail accounts for the authenticated user. Returns accountId needed for readMailMessages.
- **readMailContent** — Read one message's full body. Use when the summary snippet truncates needed text (e.g. a verification/OTP code).
- **readMailMessages** — Read recent messages (subject + truncated summary) from the authenticated user's Zoho Mail inbox.
- **search** — Search any Zoho Books or CRM module and return up to N records.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/zoho/index.js -->
```js
/**
 * Zoho skill book — PURE text-prompt registry (no imports, no functions).
 * Unified search across Zoho Books, Zoho CRM, and Zoho Mail. The executable
 * dispatch/routing + OAuth + raw API calls all live in the paired weapon:
 * `libs/weapon/zoho/index.js`.
 *
 * Mail constraint (LOAD-BEARING): the Zoho Mail API only reads the
 * AUTHENTICATED user's OWN mailbox. There is no admin API to read other users'
 * inboxes — each user must authorize separately (per-user OAuth in `potions`).
 */

export const skillBook = {
  id: "zoho",
  title: "Zoho",
  description: "Search Zoho Books/CRM modules and read Zoho Mail inbox via shared OAuth.",
  steps: [],
  toc: {
    search: {
      description: "Search any Zoho Books or CRM module and return up to N records.",
      input: {
        module: "string, one of: salesorders, invoices, bills, purchaseorders, creditnotes, payments, estimates, Contacts, Quotes, Leads, Deals",
        limit: "int, e.g. 5",
      },
      output: { records: "array of objects" },
    },
    readMailAccounts: {
      description: "Read Zoho Mail accounts for the authenticated user. Returns accountId needed for readMailMessages.",
      input: {},
      output: { accounts: "array of { accountId, emailAddress, displayName }" },
    },
    readMailMessages: {
      description: "Read recent messages (subject + truncated summary) from the authenticated user's Zoho Mail inbox.",
      input: {
        accountId: "string — from readMailAccounts()",
        limit: "int, default 5",
      },
      output: { messages: "array of { messageId, subject, fromAddress, receivedTime, summary }" },
    },
    readMailContent: {
      description: "Read one message's full body. Use when the summary snippet truncates needed text (e.g. a verification/OTP code).",
      input: {
        accountId: "string — from readMailAccounts()",
        messageId: "string — from readMailMessages()",
      },
      output: { message: "{ messageId, subject, fromAddress, content }" },
    },
  },
  howTo: {
    search: [
      "Unified Books/CRM search. The executable router is the zoho WEAPON:",
      "call `search({ module, limit })` in `libs/weapon/zoho/index.js`.",
      "It routes by module name through two allowlists, then returns the",
      "standard skill-action envelope `{ ok, msg, items: { records } }`.",
      "",
      "BOOKS_MODULES (routed to weapon.searchBooks → GET /books/v3/{module},",
      "lowercase API path segments):",
      "  salesorders, invoices, bills, purchaseorders, creditnotes, payments, estimates",
      "",
      "CRM_MODULES (routed to weapon.searchCrm → GET /crm/v7/{Module}, PascalCase",
      "module names):",
      "  Contacts, Quotes, Leads, Deals",
      "",
      "A module outside both allowlists is rejected with the enumerated lists.",
      "`limit` defaults to 5 (weapon clamps to 1..200).",
    ].join("\n"),
    readMailAccounts: [
      "List the AUTHENTICATED user's Zoho Mail accounts. Call the weapon:",
      "`readMailAccountsAction()` in `libs/weapon/zoho/index.js` → envelope",
      "`{ ok, msg, items: { accounts: [{ accountId, emailAddress, displayName }] } }`.",
      "",
      "Mail constraint: the Zoho Mail API only reads the authed user's OWN",
      "mailbox — there is NO admin API to read other users' inboxes. Each user",
      "authorizes separately (per-user OAuth in the `potions` table). The",
      "returned accountId is required for readMailMessages / readMailContent.",
    ].join("\n"),
    readMailMessages: [
      "Read recent inbox messages (newest first) for one account. Call the",
      "weapon: `readMailMessagesAction({ accountId, limit })` in",
      "`libs/weapon/zoho/index.js` → envelope `{ ok, msg, items: { messages } }`",
      "where each message is { messageId, subject, fromAddress, receivedTime, summary }.",
      "`accountId` comes from readMailAccounts; `limit` defaults to 5.",
      "Only the authed user's own mailbox is reachable (see Mail constraint).",
    ].join("\n"),
    readMailContent: [
      "Read one message's FULL body when the readMailMessages `summary` snippet",
      "truncates needed text (e.g. a verification/OTP code past the snippet",
      "boundary). Call the weapon: `readMailContentAction({ accountId, messageId })`",
      "in `libs/weapon/zoho/index.js` → envelope",
      "`{ ok, msg, items: { message: { messageId, subject, fromAddress, content } } }`.",
      "Both accountId and messageId are required; only the authed user's own",
      "mailbox is reachable (see Mail constraint).",
    ].join("\n"),
  },
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/zoho/index.js -->
```js
/**
 * Zoho weapon — single module covering Zoho Books, Zoho CRM, and Zoho Mail (shared OAuth).
 * Public API: searchBooks (Books), searchCrm (CRM), readMailAccounts, readMailMessages,
 *             sendMail (Mail), readWeaponStatus, readOrganizationId (OAuth setup only).
 *
 * Token acquisition (readAccessToken) is internal plumbing — agents never decide when to refresh a token.
 * Auth: User identity resolves via adventurer execution context first, then Next.js requireUser.
 * This lets the weapon work in both execution context (cron/scripts) and HTTP handlers.
 *
 * Satellite-repo server-side sends (no user session): implement a local libs/zoho-mail.js using
 * env vars ZOHO_MAIL_CLIENT_ID/SECRET/REFRESH_TOKEN/ACCOUNT_ID/FROM_ADDRESS with the same
 * POST shape as sendMail. See boster_nexus/libs/zoho-mail.js as the reference implementation.
 */

export const toc = {
  search: "Unified search: route a module name to Zoho Books or Zoho CRM via the BOOKS_MODULES / CRM_MODULES allowlists and return up to N records (skill-action envelope).",
  searchBooks: "Search a Zoho Books module (sales orders, invoices, bills, contacts, quotes).",
  searchCrm: "Search a Zoho CRM module (leads, deals, contacts, accounts).",
  readMailAccounts: "Read Zoho Mail accounts for the authenticated user.",
  readMailMessages: "Read recent messages (subject + summary snippet) from the authenticated Zoho Mail inbox.",
  readMailContent: "Read the full plain/HTML body of one Zoho Mail message (use when the summary snippet truncates needed text, e.g. verification codes).",
  readMailAccountsAction: "Adventurer-facing wrapper over readMailAccounts that returns the skill-action envelope.",
  readMailMessagesAction: "Adventurer-facing wrapper over readMailMessages that returns the skill-action envelope.",
  readMailContentAction: "Adventurer-facing wrapper over readMailContent that returns the skill-action envelope.",
  sendMail: "Send an email via Zoho Mail API using the authenticated user's token.",
  readWeaponStatus: "Read the Zoho weapon's auth/credential status for the current user.",
  readOrganizationId: "OAuth setup only: fetch the org id after exchanging the code.",
};
import { getZohoBooksAppCredentials } from "@/libs/council/profileEnvVars";
import { database } from "@/libs/council/database";
import { publicTables } from "@/libs/council/publicTables";
import { getSiteUrl } from "@/libs/council/auth/urls";
import { skillActionOk, skillActionErr } from "@/libs/skill_book/actionResult.js";

const ZOHO_KIND = "zoho_books";
const EXPIRY_SKEW_MS = 120_000;

/**
 * Module allowlists for the unified `search` router.
 * BOOKS_MODULES → Zoho Books API path segments (lowercase, GET /books/v3/{module}).
 * CRM_MODULES   → Zoho CRM module names (PascalCase, GET /crm/v7/{Module}).
 */
const BOOKS_MODULES = new Set([
  "salesorders", "invoices", "bills",
  "purchaseorders", "creditnotes", "payments", "estimates",
]);

const CRM_MODULES = new Set([
  "Contacts", "Quotes", "Leads", "Deals",
]);

/**
 * Coalesce an adventurer-action call signature (userId, payload) | (payload) to the payload object.
 * @param {unknown} [a]
 * @param {unknown} [b]
 * @returns {Record<string, unknown>}
 */
function normalizeInput(a, b) {
  if (b !== undefined && typeof b === "object" && b !== null && !Array.isArray(b)) return /** @type {Record<string, unknown>} */ (b);
  if (a !== undefined && typeof a === "object" && a !== null && !Array.isArray(a)) return /** @type {Record<string, unknown>} */ (a);
  return {};
}

const TOKEN_HOST = {
  com: "https://accounts.zoho.com",
  eu: "https://accounts.zoho.eu",
  in: "https://accounts.zoho.in",
  com_au: "https://accounts.zoho.com.au",
  jp: "https://accounts.zoho.jp",
};

const ZOHO_API_BASE = {
  com: "https://www.zohoapis.com",
  eu: "https://www.zohoapis.eu",
  in: "https://www.zohoapis.in",
  com_au: "https://www.zohoapis.com.au",
  jp: "https://www.zohoapis.jp",
};

const ZOHO_MAIL_BASE = {
  com: "https://mail.zoho.com",
  eu: "https://mail.zoho.eu",
  in: "https://mail.zoho.in",
  com_au: "https://mail.zoho.com.au",
  jp: "https://mail.zoho.jp",
};

function rowToLegacyShape(row) {
  if (!row) return null;
  const s = row.secrets && typeof row.secrets === "object" ? row.secrets : {};
  return {
    access_token: s.access_token ?? null,
    refresh_token: s.refresh_token ?? null,
    region: s.region ?? null,
    token_type: s.token_type ?? null,
    expires_at: s.expires_at ?? null,
    organization_id: s.organization_id ?? null,
  };
}

function accessTokenLikelyExpired(expiresAtIso) {
  if (!expiresAtIso || typeof expiresAtIso !== "string") return false;
  const t = Date.parse(expiresAtIso);
  if (Number.isNaN(t)) return false;
  return Date.now() > t - EXPIRY_SKEW_MS;
}

/**
 * Resolve userId: adventurer execution context → Next.js requireUser → explicit arg.
 * @param {string | null | undefined} [explicit]
 */
async function resolveUserId(explicit) {
  if (explicit) return explicit;
  const { getAdventurerExecutionUserId } = await import("@/libs/adventurer/advance.js");
  const ctxId = getAdventurerExecutionUserId();
  if (ctxId) return ctxId;
  const { requireUser } = await import("@/libs/council/auth/server");
  const user = await requireUser();
  return user.id;
}

/**
 * Get a DB client: prefer service-role from execution context (works outside Next.js HTTP),
 * fall back to server-scoped client.
 * @param {import("@supabase/supabase-js").SupabaseClient | null | undefined} [injected]
 */
async function resolveClient(injected) {
  if (injected) return injected;
  const { getAdventurerExecutionContext } = await import("@/libs/adventurer/advance.js");
  const ctx = getAdventurerExecutionContext();
  if (ctx?.client) return ctx.client;
  return database.init("service");
}

async function getZohoConnection(userId, injectedClient) {
  const db = await resolveClient(injectedClient);
  const { data: row, error } = await db
    .from(publicTables.potions)
    .select("secrets")
    .eq("owner_id", userId)
    .eq("kind", ZOHO_KIND)
    .maybeSingle();
  if (error) {
    throw new Error(`Could not read Zoho connection: ${error.message}`);
  }
  return rowToLegacyShape(row);
}

export async function upsertZohoConnection(payload, injectedClient) {
  const db = await resolveClient(injectedClient);
  const { user_id, ...rest } = payload;
  const secrets = { ...rest };
  const expiresRaw = secrets.expires_at;
  const expires_at =
    typeof expiresRaw === "string" && expiresRaw.length > 0 ? expiresRaw : null;
  return db.from(publicTables.potions).upsert(
    {
      owner_id: user_id,
      kind: ZOHO_KIND,
      secrets,
      expires_at,
      updated_at: new Date().toISOString(),
    },
    { onConflict: "owner_id,kind" }
  );
}

/** @returns {Record<string, string>} */
function getAuthLinks() {
  const base = getSiteUrl().replace(/\/$/, "");
  return {
    connectZoho: `${base}/api/weapon/zoho?action=connect`,
    inn: `${base}/quest-board`,
    forgeZoho: `${base}/town-square/forge/zoho`,
    status: `${base}/api/weapon/zoho?action=status`,
  };
}

/** Short, Forge-centric message used in all "not connected / expired" errors. */
function getAuthLinksMessage() {
  const L = getAuthLinks();
  return `Go to the Forge to activate or re-forge the Zoho weapon: ${L.forgeZoho}`;
}

/** Message specifically for missing / insufficient CRM scope (401 from CRM API). */
function getCrmScopeMessage() {
  const L = getAuthLinks();
  return (
    `WEAPON_REAUTH_REQUIRED: The Zoho weapon does not have CRM scope (received 401 from Zoho CRM API). ` +
    `This means the current OAuth token was issued before CRM permissions were added. ` +
    `Go to the Forge, click "Scrap weapon & forge again", then re-authorize to grant CRM access: ${L.forgeZoho}`
  );
}

/**
 * @param {{ refreshToken: string, region: string, clientId: string, clientSecret: string }} p
 */
async function refreshAccessToken({ refreshToken, region, clientId, clientSecret }) {
  const host = TOKEN_HOST[region] || TOKEN_HOST.com;
  const body = new URLSearchParams({
    grant_type: "refresh_token",
    client_id: clientId,
    client_secret: clientSecret,
    refresh_token: refreshToken,
  });

  const response = await fetch(`${host}/oauth/v2/token`, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body,
  });

  if (!response.ok) {
    const text = await response.text();
    return { ok: false, error: `HTTP ${response.status}: ${text.slice(0, 200)}` };
  }

  const token = await response.json();
  return { ok: true, token };
}

async function refreshAndPersist(userId, current, injectedClient) {
  const { clientId, clientSecret } = await getZohoBooksAppCredentials(userId);
  if (!clientId || !clientSecret) {
    throw new Error(
      `Missing Zoho OAuth app credentials (ZOHO_BOOKS_CLIENT_ID / ZOHO_BOOKS_CLIENT_SECRET). ` +
      `Set them in Council Hall → Formulary, then ${getAuthLinksMessage()}`
    );
  }
  const region = current.region || "com";
  const refreshed = await refreshAccessToken({
    refreshToken: current.refresh_token,
    region,
    clientId,
    clientSecret,
  });
  if (!refreshed.ok) {
    throw new Error(
      `Zoho token refresh failed (${refreshed.error}). ${getAuthLinksMessage()}`
    );
  }
  const t = refreshed.token;
  const expiresAt = new Date(Date.now() + (t.expires_in || 3600) * 1000).toISOString();
  const newSecrets = {
    region,
    access_token: t.access_token,
    refresh_token: t.refresh_token ?? current.refresh_token,
    token_type: t.token_type ?? current.token_type,
    expires_at: expiresAt,
    organization_id: current.organization_id,
  };
  const { error: upErr } = await upsertZohoConnection({ user_id: userId, ...newSecrets }, injectedClient);
  if (upErr) {
    throw new Error(`Could not save refreshed token: ${upErr.message}`);
  }
  return newSecrets;
}

/**
 * @param {string} userId
 * @param {import("@supabase/supabase-js").SupabaseClient | null | undefined} [injectedClient]
 * @returns {Promise<{ access_token: string, region: string, organization_id: string | null, refresh_token: string | null }>}
 */
async function ensureSecretsForCurrentUser(userId, injectedClient) {
  let current = await getZohoConnection(userId, injectedClient);

  if (!current?.access_token && !current?.refresh_token) {
    throw new Error(`Zoho weapon not connected. ${getAuthLinksMessage()}`);
  }

  const needsRefresh = !current.access_token || accessTokenLikelyExpired(current.expires_at);

  if (needsRefresh) {
    if (!current.refresh_token) {
      throw new Error(`Zoho access token expired and no refresh token found. ${getAuthLinksMessage()}`);
    }
    current = await refreshAndPersist(userId, current, injectedClient);
  }

  if (!current?.access_token) {
    throw new Error(`Zoho access token unavailable after refresh. ${getAuthLinksMessage()}`);
  }

  return {
    access_token: current.access_token,
    region: current.region || "com",
    organization_id: current.organization_id ?? null,
    refresh_token: current.refresh_token ?? null,
  };
}

/**
 * Exchange an OAuth authorization code for tokens.
 * @param {{ code: string, region: string, clientId: string, clientSecret: string, redirectUri: string }} p
 */
export async function exchangeZohoCode({ code, region, clientId, clientSecret, redirectUri }) {
  const host = TOKEN_HOST[region] || TOKEN_HOST.com;
  const body = new URLSearchParams({
    grant_type: "authorization_code",
    client_id: clientId,
    client_secret: clientSecret,
    redirect_uri: redirectUri,
    code,
  });
  const response = await fetch(`${host}/oauth/v2/token`, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body,
  });
  if (!response.ok) {
    const text = await response.text();
    return { ok: false, error: `HTTP ${response.status}: ${text.slice(0, 200)}` };
  }
  const token = await response.json();
  return { ok: true, token };
}

/**
 * Fetch the Zoho Books organization_id using a fresh access token.
 * @param {string} accessToken
 * @param {string} [region]
 */
export async function readOrganizationId(accessToken, region = "com") {
  const baseUrl = ZOHO_API_BASE[region] || ZOHO_API_BASE.com;
  const response = await fetch(`${baseUrl}/books/v3/organizations`, {
    headers: { Authorization: `Zoho-oauthtoken ${accessToken}` },
    cache: "no-store",
  });
  if (!response.ok) return null;
  const json = await response.json();
  const orgs = json?.organizations;
  return Array.isArray(orgs) && orgs[0]?.organization_id ? String(orgs[0].organization_id) : null;
}

/**
 * @param {string} [userId] — optional; resolves from execution context or requireUser if omitted
 * @returns {Promise<string>} Bearer access token for Zoho API.
 */
async function readAccessToken(userId) {
  const uid = await resolveUserId(userId);
  const s = await ensureSecretsForCurrentUser(uid);
  return s.access_token;
}

/**
 * Search any Zoho Books module (salesorders, invoices, bills, etc.).
 * Returns up to `limit` records sorted by date descending.
 *
 * @param {string} module — API path segment (e.g. "salesorders", "invoices")
 * @param {number} limit
 * @param {string} [userId] — optional; resolves from execution context if omitted
 * @returns {Promise<unknown[]>}
 */
export async function searchBooks(module, limit, userId) {
  const uid = await resolveUserId(userId);
  const secrets = await ensureSecretsForCurrentUser(uid);
  const region = secrets.region || "com";
  const baseUrl = ZOHO_API_BASE[region] || ZOHO_API_BASE.com;
  const organizationId = secrets.organization_id;

  if (!organizationId) {
    throw new Error(
      `Zoho organization_id is missing. Re-forge the weapon so the callback can store it. ${getAuthLinksMessage()}`
    );
  }

  const mod = String(module ?? "").replace(/^\//, "").replace(/\/$/, "");
  if (!mod) {
    throw new Error("searchBooks: module is required.");
  }

  const n = Math.max(1, Math.min(200, Number(limit) || 10));

  const url = new URL(`${baseUrl}/books/v3/${encodeURIComponent(mod)}`);
  if (process.env.NODE_ENV === "development") {
    // eslint-disable-next-line no-console
    console.log(`[Zoho:searchBooks] ${url.toString()}`);
  }
  url.searchParams.set("organization_id", organizationId);
  url.searchParams.set("per_page", String(n));
  url.searchParams.set("sort_column", "date");
  url.searchParams.set("sort_order", "D");

  const response = await fetch(url.toString(), {
    headers: { Authorization: `Zoho-oauthtoken ${secrets.access_token}` },
    cache: "no-store",
  });

  if (!response.ok) {
    const text = await response.text();
    throw new Error(`ZOHO_BOOKS_FETCH_FAILED:${response.status}:${text.slice(0, 500)}`);
  }

  const json = await response.json();
  if (json && typeof json === "object") {
    const arrKey = Object.keys(json).find((k) => Array.isArray(json[k]));
    if (arrKey && Array.isArray(json[arrKey])) {
      return json[arrKey].slice(0, n);
    }
  }
  return [];
}

/**
 * Default fields to request per CRM module. Keeps responses concise and
 * avoids returning hundreds of custom fields. Unknown modules omit the
 * `fields` param and let the CRM API return its default set.
 */
const CRM_MODULE_FIELDS = {
  Contacts: "First_Name,Last_Name,Email,Phone,Account_Name,Lead_Source",
  Quotes: "Subject,Quote_Stage,Grand_Total,Account_Name,Contact_Name,Expiry_Date",
  Leads: "First_Name,Last_Name,Email,Phone,Company,Lead_Source",
  Deals: "Deal_Name,Stage,Amount,Account_Name,Closing_Date",
};

/**
 * Search any Zoho CRM module (Contacts, Quotes, Leads, Deals, …).
 * Uses `GET /crm/v7/{Module}`. Module names are PascalCase.
 *
 * @param {string} module — PascalCase CRM module name (e.g. "Contacts", "Quotes")
 * @param {number} limit — max records to return (default 5, max 200)
 * @param {string} [userId] — optional; resolves from execution context if omitted
 * @returns {Promise<unknown[]>}
 */
export async function searchCrm(module, limit, userId) {
  const uid = await resolveUserId(userId);
  const secrets = await ensureSecretsForCurrentUser(uid);
  const region = secrets.region || "com";
  const baseUrl = ZOHO_API_BASE[region] || ZOHO_API_BASE.com;
  const n = Math.max(1, Math.min(200, Number(limit) || 5));

  const mod = String(module ?? "").trim();
  if (!mod) {
    throw new Error("searchCrm: module is required.");
  }

  const url = new URL(`${baseUrl}/crm/v7/${encodeURIComponent(mod)}`);
  url.searchParams.set("per_page", String(n));
  const fields = CRM_MODULE_FIELDS[mod];
  if (fields) {
    url.searchParams.set("fields", fields);
  }

  const response = await fetch(url.toString(), {
    headers: { Authorization: `Zoho-oauthtoken ${secrets.access_token}` },
    cache: "no-store",
  });

  if (!response.ok) {
    const text = await response.text();
    if (response.status === 401 || response.status === 403) {
      throw new Error(getCrmScopeMessage());
    }
    throw new Error(`ZOHO_CRM_FETCH_FAILED:${response.status}:${text.slice(0, 500)}`);
  }

  const json = await response.json();
  return Array.isArray(json?.data) ? json.data.slice(0, n) : [];
}

/**
 * Unified search dispatch: route a module name to Books or CRM via the
 * BOOKS_MODULES / CRM_MODULES allowlists. Returns the standard skill-action
 * envelope so adventurer execution can consume it directly.
 *
 * @param {unknown} [a] userId or payload `{ module, limit }`
 * @param {unknown} [b] payload when `a` is a userId
 * @returns {Promise<import("@/libs/skill_book/actionResult.js").SkillActionResult>}
 */
export async function search(a, b) {
  const raw = normalizeInput(a, b);
  const module = String(raw.module ?? "").trim();
  const limit = Number(raw.limit ?? 5);

  if (!module) {
    return skillActionErr(
      `"module" is required. Books: ${[...BOOKS_MODULES].join(", ")}. CRM: ${[...CRM_MODULES].join(", ")}.`
    );
  }

  if (BOOKS_MODULES.has(module)) {
    try {
      const rows = await searchBooks(module, limit);
      return skillActionOk({ records: rows });
    } catch (e) {
      return skillActionErr(e instanceof Error ? e.message : String(e));
    }
  }

  if (CRM_MODULES.has(module)) {
    try {
      const rows = await searchCrm(module, limit);
      return skillActionOk({ records: rows });
    } catch (e) {
      return skillActionErr(e instanceof Error ? e.message : String(e));
    }
  }

  return skillActionErr(
    `Unknown Zoho module: "${module}". ` +
    `Books: ${[...BOOKS_MODULES].join(", ")}. CRM: ${[...CRM_MODULES].join(", ")}.`
  );
}

/**
 * Build the Zoho OAuth authorization URL.
 * Scopes cover both Zoho Books and Zoho CRM (shared OAuth flow).
 *
 * @param {{ region?: string, clientId: string, redirectUri: string, extraScopes?: string[] }} opts
 */
export function buildZohoOAuthAuthorizeUrl({ region = "com", clientId, redirectUri, extraScopes = [] }) {
  const host = TOKEN_HOST[region] || TOKEN_HOST.com;
  const defaultScopes = [
    "ZohoBooks.fullaccess.all",
    "ZohoCRM.modules.contacts.READ",
    "ZohoCRM.modules.contacts.WRITE",
    "ZohoCRM.modules.quotes.READ",
    "ZohoCRM.modules.leads.READ",
    "ZohoCRM.modules.deals.READ",
    "ZohoMail.messages.ALL",
    "ZohoMail.accounts.READ",
  ];
  const scopes = [...new Set([...defaultScopes, ...extraScopes])].join(",");
  const url = new URL(`${host}/oauth/v2/auth`);
  url.searchParams.set("response_type", "code");
  url.searchParams.set("client_id", clientId);
  url.searchParams.set("scope", scopes);
  url.searchParams.set("redirect_uri", redirectUri);
  url.searchParams.set("access_type", "offline");
  url.searchParams.set("prompt", "consent");
  return url.toString();
}

/** OAuth callback URL registered with Zoho (must match console + token exchange). */
export function getZohoOAuthCallbackUrl() {
  const base = getSiteUrl().replace(/\/$/, "");
  return `${base}/api/weapon/zoho?action=callback`;
}

/**
 * Delete the Zoho OAuth token row for a user (scrap the weapon).
 * The Formulary OAuth app credentials (clientId/clientSecret) are untouched.
 * @param {string} userId
 * @param {import("@supabase/supabase-js").SupabaseClient | null | undefined} [injectedClient]
 */
export async function deleteZohoConnection(userId, injectedClient) {
  const db = await resolveClient(injectedClient);
  return db
    .from(publicTables.potions)
    .delete()
    .eq("owner_id", userId)
    .eq("kind", ZOHO_KIND);
}

async function readZohoSecretsRow(userId) {
  const db = await database.init("server");
  const { data: row, error } = await db
    .from(publicTables.potions)
    .select("secrets")
    .eq("owner_id", userId)
    .eq("kind", ZOHO_KIND)
    .maybeSingle();
  if (error) return { data: null, error };
  return { data: rowToLegacyShape(row), error: null };
}

/**
 * Sanitized status for forge / diagnostics (no secrets exposed).
 * @param {string} userId
 */
export async function readWeaponStatus(userId) {
  const { data, error } = await readZohoSecretsRow(userId);
  if (error) {
    return {
      potionsReadOk: false,
      code: "ZOHO_POTIONS_READ_FAILED",
      message: error.message,
      hasAccessToken: false,
      hasRefreshToken: false,
      hasOrganizationId: false,
      region: null,
      expiresAt: null,
    };
  }
  const d = data;
  return {
    potionsReadOk: true,
    code: d?.access_token ? "ZOHO_ROW_PRESENT" : "ZOHO_NOT_CONNECTED",
    message: d?.access_token ? "potions row has access_token" : "no access_token in potions.secrets",
    hasAccessToken: Boolean(d?.access_token),
    hasRefreshToken: Boolean(d?.refresh_token),
    hasOrganizationId: Boolean(d?.organization_id),
    region: d?.region || null,
    expiresAt: d?.expires_at || null,
  };
}

function getMailScopeMessage() {
  const L = getAuthLinks();
  return (
    `WEAPON_REAUTH_REQUIRED: The Zoho weapon does not have Mail scope (received 401 from Zoho Mail API). ` +
    `Go to the Forge, click "Scrap weapon & forge again", then re-authorize to grant Mail access: ${L.forgeZoho}`
  );
}

/**
 * List Zoho Mail accounts for the authenticated user.
 * Returns account IDs needed for reading messages.
 *
 * @param {string} [userId]
 * @returns {Promise<Array<{ accountId: string, emailAddress: string, displayName: string }>>}
 */
export async function readMailAccounts(userId) {
  const uid = await resolveUserId(userId);
  const secrets = await ensureSecretsForCurrentUser(uid);
  const region = secrets.region || "com";
  const baseUrl = ZOHO_MAIL_BASE[region] || ZOHO_MAIL_BASE.com;

  const response = await fetch(`${baseUrl}/api/accounts`, {
    headers: { Authorization: `Zoho-oauthtoken ${secrets.access_token}` },
    cache: "no-store",
  });

  if (!response.ok) {
    const text = await response.text();
    if (response.status === 401 || response.status === 403) {
      throw new Error(getMailScopeMessage());
    }
    throw new Error(`ZOHO_MAIL_ACCOUNTS_FAILED:${response.status}:${text.slice(0, 500)}`);
  }

  const json = await response.json();
  const accounts = Array.isArray(json?.data) ? json.data : [];
  return accounts.map((a) => ({
    accountId: String(a.accountId ?? a.account_id ?? ""),
    emailAddress: a.emailAddress ?? a.incomingUserName ?? "",
    displayName: a.displayName ?? a.name ?? "",
  }));
}

/**
 * List Zoho Mail messages for a specific account.
 * Defaults to the authenticated user's primary inbox.
 *
 * @param {string} accountId — from readMailAccounts()
 * @param {{ limit?: number, folder?: string }} [opts]
 * @param {string} [userId]
 * @returns {Promise<Array<{ messageId: string, subject: string, fromAddress: string, receivedTime: string, summary: string }>>}
 */
export async function readMailMessages(accountId, opts = {}, userId) {
  const uid = await resolveUserId(userId);
  const secrets = await ensureSecretsForCurrentUser(uid);
  const region = secrets.region || "com";
  const baseUrl = ZOHO_MAIL_BASE[region] || ZOHO_MAIL_BASE.com;
  const limit = Math.max(1, Math.min(200, Number(opts.limit) || 5));

  const url = new URL(`${baseUrl}/api/accounts/${accountId}/messages/view`);
  url.searchParams.set("limit", String(limit));
  url.searchParams.set("sortorder", "false"); // newest first

  const response = await fetch(url.toString(), {
    headers: { Authorization: `Zoho-oauthtoken ${secrets.access_token}` },
    cache: "no-store",
  });

  if (!response.ok) {
    const text = await response.text();
    if (response.status === 401 || response.status === 403) {
      throw new Error(getMailScopeMessage());
    }
    throw new Error(`ZOHO_MAIL_MESSAGES_FAILED:${response.status}:${text.slice(0, 500)}`);
  }

  const json = await response.json();
  const messages = Array.isArray(json?.data) ? json.data : [];
  return messages.slice(0, limit).map((m) => ({
    messageId: String(m.messageId ?? ""),
    subject: m.subject ?? "(no subject)",
    fromAddress: m.fromAddress ?? "",
    receivedTime: m.receivedTime ?? "",
    summary: m.summary ?? "",
  }));
}

/**
 * Read the full body of a single Zoho Mail message. Use when readMailMessages'
 * `summary` snippet truncates the text you need (e.g. a verification code that
 * sits past the snippet boundary).
 *
 * @param {string} accountId — from readMailAccounts()
 * @param {string} messageId — from readMailMessages()
 * @param {string} [userId]
 * @returns {Promise<{ messageId: string, subject: string, fromAddress: string, content: string }>}
 */
export async function readMailContent(accountId, messageId, userId) {
  const uid = await resolveUserId(userId);
  const secrets = await ensureSecretsForCurrentUser(uid);
  const region = secrets.region || "com";
  const baseUrl = ZOHO_MAIL_BASE[region] || ZOHO_MAIL_BASE.com;

  if (!accountId) throw new Error("readMailContent: accountId is required.");
  if (!messageId) throw new Error("readMailContent: messageId is required.");

  const response = await fetch(
    `${baseUrl}/api/accounts/${accountId}/messages/${messageId}/content`,
    {
      headers: { Authorization: `Zoho-oauthtoken ${secrets.access_token}` },
      cache: "no-store",
    },
  );

  if (!response.ok) {
    const text = await response.text();
    if (response.status === 401 || response.status === 403) {
      throw new Error(getMailScopeMessage());
    }
    throw new Error(`ZOHO_MAIL_CONTENT_FAILED:${response.status}:${text.slice(0, 500)}`);
  }

  const json = await response.json();
  const d = json?.data ?? {};
  return {
    messageId: String(d.messageId ?? messageId),
    subject: d.subject ?? "(no subject)",
    fromAddress: d.fromAddress ?? "",
    content: String(d.content ?? ""),
  };
}

/**
 * Adventurer-facing wrapper over readMailAccounts → skill-action envelope.
 * @param {unknown} [a]
 * @param {unknown} [b]
 * @returns {Promise<import("@/libs/skill_book/actionResult.js").SkillActionResult>}
 */
export async function readMailAccountsAction(a, b) {
  try {
    const accounts = await readMailAccounts();
    return skillActionOk({ accounts });
  } catch (e) {
    return skillActionErr(e instanceof Error ? e.message : String(e));
  }
}

/**
 * Adventurer-facing wrapper over readMailMessages → skill-action envelope.
 * @param {unknown} [a] userId or payload `{ accountId, limit }`
 * @param {unknown} [b] payload when `a` is a userId
 * @returns {Promise<import("@/libs/skill_book/actionResult.js").SkillActionResult>}
 */
export async function readMailMessagesAction(a, b) {
  const raw = normalizeInput(a, b);
  const accountId = String(raw.accountId ?? "").trim();
  const limit = Number(raw.limit ?? 5);

  if (!accountId) return skillActionErr('"accountId" is required. Call readMailAccounts first.');

  try {
    const messages = await readMailMessages(accountId, { limit });
    return skillActionOk({ messages });
  } catch (e) {
    return skillActionErr(e instanceof Error ? e.message : String(e));
  }
}

/**
 * Adventurer-facing wrapper over readMailContent → skill-action envelope.
 * @param {unknown} [a] userId or payload `{ accountId, messageId }`
 * @param {unknown} [b] payload when `a` is a userId
 * @returns {Promise<import("@/libs/skill_book/actionResult.js").SkillActionResult>}
 */
export async function readMailContentAction(a, b) {
  const raw = normalizeInput(a, b);
  const accountId = String(raw.accountId ?? "").trim();
  const messageId = String(raw.messageId ?? "").trim();

  if (!accountId) return skillActionErr('"accountId" is required. Call readMailAccounts first.');
  if (!messageId) return skillActionErr('"messageId" is required. Call readMailMessages first.');

  try {
    const message = await readMailContent(accountId, messageId);
    return skillActionOk({ message });
  } catch (e) {
    return skillActionErr(e instanceof Error ? e.message : String(e));
  }
}

/**
 * Send an email via Zoho Mail API using the authenticated user's OAuth token.
 *
 * Requires the Zoho weapon to be connected with ZohoMail.messages.ALL scope.
 * The account to send from is resolved automatically (first account in the user's Zoho Mail).
 *
 * For satellite repos (boster_nexus, etc.) running server-side cron/alert sends without a user
 * session: implement a local `libs/zoho-mail.js` using the same POST shape but with env-var
 * refresh tokens (ZOHO_MAIL_CLIENT_ID, ZOHO_MAIL_CLIENT_SECRET, ZOHO_MAIL_REFRESH_TOKEN,
 * ZOHO_MAIL_ACCOUNT_ID, ZOHO_MAIL_FROM_ADDRESS) instead of per-user potions OAuth.
 *
 * @param {{ to: string | string[], subject: string, text?: string, html?: string, from?: string, replyTo?: string }} params
 * @param {string} [userId]
 * @returns {Promise<{ ok: boolean, messageId: string | undefined }>}
 */
export async function sendMail({ to, subject, text, html, from, replyTo }, userId) {
  const uid = await resolveUserId(userId);
  const secrets = await ensureSecretsForCurrentUser(uid);
  const region = secrets.region || "com";
  const baseUrl = ZOHO_MAIL_BASE[region] || ZOHO_MAIL_BASE.com;

  // Resolve the sending account — use ZOHO_MAIL_ACCOUNT_ID env if set, else fetch first account.
  let accountId = process.env.ZOHO_MAIL_ACCOUNT_ID;
  if (!accountId) {
    const accounts = await readMailAccounts(userId);
    if (!accounts.length) throw new Error("sendMail: no Zoho Mail accounts found for this user.");
    accountId = accounts[0].accountId;
  }

  const fromAddress = from || process.env.ZOHO_MAIL_FROM_ADDRESS;
  if (!fromAddress) throw new Error("sendMail: no from address — set ZOHO_MAIL_FROM_ADDRESS or pass from param.");

  const toAddress = Array.isArray(to) ? to.join(",") : to;
  if (!toAddress) throw new Error("sendMail: to is required.");
  if (!subject) throw new Error("sendMail: subject is required.");
  if (!text && !html) throw new Error("sendMail: text or html is required.");

  const payload = {
    fromAddress,
    toAddress,
    subject,
    content: html || text,
    mailFormat: html ? "html" : "plaintext",
    ...(replyTo ? { replyTo } : {}),
  };

  const res = await fetch(`${baseUrl}/api/accounts/${accountId}/messages`, {
    method: "POST",
    headers: {
      Authorization: `Zoho-oauthtoken ${secrets.access_token}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(payload),
  });

  const data = await res.json().catch(() => null);
  if (res.status === 401 || res.status === 403) throw new Error(getMailScopeMessage());
  if (!res.ok) throw new Error(`ZOHO_MAIL_SEND_FAILED:${res.status}:${JSON.stringify(data)}`);
  return { ok: true, messageId: data?.data?.messageId };
}

/**
 * JSON-safe error payload for API routes.
 * @param {unknown} error
 */
export function zohoErrorToJsonPayload(error) {
  if (error && typeof error === "object" && "message" in error) {
    const e = /** @type {{ message?: string, code?: string, links?: unknown }} */ (error);
    const payload = { error: String(e.message ?? error) };
    if (e.code) payload.code = e.code;
    if (e.links) payload.links = e.links;
    return payload;
  }
  return { error: error instanceof Error ? error.message : String(error) };
}
```
<!-- /GUILDOS-FILE -->
