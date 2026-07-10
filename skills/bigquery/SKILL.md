# bigquery

<!-- GUILDOS-SKILL name="bigquery" -->

<!-- GUILDOS-DESC -->
Query Google BigQuery tables for event data via the REST API v2. List datasets, read the most recent rows from a partitioned table, run arbitrary StandardSQL queries against any table, and stream-insert rows. Use for fetching event data out of BigQuery datasets; runQuery is the path for non-partitioned tables or any custom SQL.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/bigquery/SKILL.md -->
```markdown
---
name: bigquery
description: >-
  Query Google BigQuery tables for event data via the REST API v2. List
  datasets, read the most recent rows from a partitioned table, run arbitrary
  StandardSQL queries against any table, and stream-insert rows. Use for
  fetching event data out of BigQuery datasets; runQuery is the path for
  non-partitioned tables or any custom SQL.
---

# BigQuery

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/bigquery/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto bigquery <action>` (or read the source directly).

## Actions

- **insertRows** — Stream-insert rows into a BigQuery table via tabledata.insertAll. Calls weapon/bigquery → insertRows({ datasetId, tableId, rows, userId }).
- **readRecentEvents** — Return the most recent rows from a PARTITIONED BigQuery table, ordered by _PARTITIONTIME DESC (non-partitioned tables will fail — use runQuery). Calls weapon/bigquery → readRecentEvents(datasetId, tableId, limit, userId).
- **runQuery** — Run an arbitrary StandardSQL query and return paged rows + schema. Use this for non-partitioned tables or any custom SQL. Calls weapon/bigquery → runQuery({ query, maxResults, userId }).
- **searchDatasets** — List BigQuery datasets available to the service account. Calls weapon/bigquery → searchDatasets(userId).
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/bigquery/index.js -->
```js
/**
 * BigQuery skill book — PURE prompt-registry. All executable logic lives in the
 * paired weapon `@/libs/weapon/bigquery`. No imports, no functions here.
 *
 * Weapon notes (text):
 * - The weapon hand-builds an RS256 service-account JWT (no `@google-cloud` dep)
 *   from the formulary / profiles.env_vars `GOOGLE_SERVICE_ACCOUNT`, then exchanges
 *   it for a BigQuery access token via the OAuth2 token endpoint.
 * - `readRecentEvents` hardcodes `ORDER BY _PARTITIONTIME DESC` — it works on
 *   time-partitioned tables ONLY. For non-partitioned tables (or any custom SQL)
 *   use `runQuery` instead.
 */

export const skillBook = {
  id: "bigquery",
  title: "BigQuery",
  description: "Query Google BigQuery tables for event data via the REST API v2.",
  steps: [],
  toc: {
    searchDatasets: {
      description:
        "List BigQuery datasets available to the service account. Calls weapon/bigquery → searchDatasets(userId).",
    },
    readRecentEvents: {
      description:
        "Return the most recent rows from a PARTITIONED BigQuery table, ordered by _PARTITIONTIME DESC (non-partitioned tables will fail — use runQuery). Calls weapon/bigquery → readRecentEvents(datasetId, tableId, limit, userId).",
    },
    runQuery: {
      description:
        "Run an arbitrary StandardSQL query and return paged rows + schema. Use this for non-partitioned tables or any custom SQL. Calls weapon/bigquery → runQuery({ query, maxResults, userId }).",
    },
    insertRows: {
      description:
        "Stream-insert rows into a BigQuery table via tabledata.insertAll. Calls weapon/bigquery → insertRows({ datasetId, tableId, rows, userId }).",
    },
  },
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/bigquery/engineering_decisions.md -->
```markdown
# BigQuery weapon — Engineering Decisions

## 2026-06-30 · `runQuery` uses the SYNCHRONOUS `jobs.query` endpoint, not `jobs.insert` + `getQueryResults`

**Decision.** `runQuery` POSTs to `/projects/{p}/queries` (synchronous `jobs.query`)
and reads the first page of rows inline, paging the remainder only if
`jobComplete` and a `pageToken` are present.

**Rejected alternative.** The original async pattern — `jobs.insert` to create a
job, poll `/jobs/{jobId}`, then fetch `/jobs/{jobId}/getQueryResults`. This is
the "textbook" BigQuery flow and is what a competent agent would write by
default.

**Why.** On this project (`boster-cbi`) the async `getQueryResults` call returns
an **HTML 404 from Google's frontend** (the generic robot 404 page, not a JSON
API error) **even with the correct `location=US` echoed back** on the URL. The
job runs and completes; only the result-fetch leg 404s — a routing quirk that
made `runQuery` completely unusable. The synchronous `/queries` endpoint returns
schema+rows in the insert response itself and sidesteps the broken leg.

**Consequence if violated.** Revert to `jobs.insert`+`getQueryResults` and every
`runQuery` call dies with `BigQuery getQueryResults failed (404): <!DOCTYPE html>…`.

## 2026-06-30 · Query auth requests the full `bigquery` scope, not `bigquery.readonly`

**Decision.** `runQuery` mints its access token with
`https://www.googleapis.com/auth/bigquery`.

**Rejected alternative.** `bigquery.readonly` (the intuitive choice for
"read-only" reporting queries, and the weapon's original default).

**Why.** Running ANY query — even a pure `SELECT` — creates a job, and the
read-only scope does not authorize job creation → `403
ACCESS_TOKEN_SCOPE_INSUFFICIENT (ACCESS_TOKEN_SCOPE_INSUFFICIENT)` on
`jobs`/`queries`. The data the query can touch is still bounded by the service
account's dataset-level IAM, so the broader token scope grants no extra data
reach.

**Consequence if violated.** Drop back to `.readonly` and every `runQuery` call
fails with a 403 before BigQuery ever sees the SQL.

## 2026-06-30 · `run-action.mjs` can only call weapon fns with the opts-object signature

Note (not a code change): `bigquery.searchDatasets(userId)` and
`readRecentEvents(datasetId, tableId, limit, userId)` take **positional** args, but
`libs/adventurer/run-action.mjs` invokes every weapon fn as `fn(inputObject)`. So
those two are NOT callable via the CLI (the input object lands in the first
positional, mangling `userId`). `runQuery({query, userId})` and
`insertRows({…})` take a single opts object and ARE CLI-callable. To list
datasets from the CLI, query `INFORMATION_SCHEMA.SCHEMATA` via `runQuery` rather
than calling `searchDatasets`.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/bigquery/index.js -->
```js
/**
 * BigQuery weapon — queries Google BigQuery via the REST API v2 (jobs.query).
 * Auth: GOOGLE_SERVICE_ACCOUNT in profiles.env_vars (service account JSON), falls back to process.env.
 */

export const toc = {
  searchDatasets: "Search BigQuery datasets available to the service account.",
  readRecentEvents: "Read the most recent rows from a BigQuery table.",
  runQuery: "Run an arbitrary StandardSQL query and return rows.",
  insertRows: "Stream-insert rows into a BigQuery table via tabledata.insertAll.",
};
import { getGoogleCredentials } from "@/libs/council/profileEnvVars";

const BIGQUERY_API = "https://bigquery.googleapis.com/bigquery/v2";
const TOKEN_URL = "https://oauth2.googleapis.com/token";

async function resolveUserId(explicit) {
  if (explicit) return explicit;
  try {
    const { getAdventurerExecutionUserId } = await import("@/libs/adventurer/advance.js");
    const ctxId = getAdventurerExecutionUserId();
    if (ctxId) return ctxId;
  } catch { /* not in adventurer context */ }
  const { requireUser } = await import("@/libs/council/auth/server");
  const user = await requireUser();
  return user.id;
}

async function getServiceAccountKey(userId) {
  const creds = await getGoogleCredentials(userId);
  const raw = creds.serviceAccountJson;
  if (!raw) throw new Error("GOOGLE_SERVICE_ACCOUNT is not set in profiles.env_vars or process.env.");
  const key = typeof raw === "string" ? JSON.parse(raw) : raw;
  if (!key.client_email || !key.private_key) {
    throw new Error("GOOGLE_SERVICE_ACCOUNT missing client_email or private_key.");
  }
  return key;
}

/**
 * Build a signed JWT and exchange it for a BigQuery access token.
 * @param {string} userId
 * @returns {Promise<string>}
 */
async function getAccessToken(userId, scope = "https://www.googleapis.com/auth/bigquery.readonly") {
  const key = await getServiceAccountKey(userId);
  const now = Math.floor(Date.now() / 1000);
  const header = { alg: "RS256", typ: "JWT" };
  const payload = {
    iss: key.client_email,
    scope,
    aud: key.token_uri || TOKEN_URL,
    iat: now,
    exp: now + 3600,
  };

  const enc = (obj) => Buffer.from(JSON.stringify(obj)).toString("base64url");
  const unsigned = `${enc(header)}.${enc(payload)}`;

  const crypto = await import("node:crypto");
  const sign = crypto.createSign("RSA-SHA256");
  sign.update(unsigned);
  const signature = sign.sign(key.private_key, "base64url");
  const jwt = `${unsigned}.${signature}`;

  const res = await fetch(key.token_uri || TOKEN_URL, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: `grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer&assertion=${jwt}`,
  });
  if (!res.ok) {
    const text = await res.text();
    throw new Error(`Token exchange failed (${res.status}): ${text}`);
  }
  const data = await res.json();
  return data.access_token;
}

async function getProjectId(userId) {
  const key = await getServiceAccountKey(userId);
  return key.project_id || "";
}

/**
 * List datasets in the project.
 * @param {string} [userId]
 * @returns {Promise<Array<{ datasetId: string, location: string }>>}
 */
export async function searchDatasets(userId) {
  const uid = await resolveUserId(userId);
  const token = await getAccessToken(uid);
  const projectId = await getProjectId(uid);
  if (!projectId) throw new Error("project_id missing from service account key.");

  const res = await fetch(`${BIGQUERY_API}/projects/${projectId}/datasets`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  if (!res.ok) {
    const text = await res.text();
    throw new Error(`searchDatasets failed (${res.status}): ${text}`);
  }
  const body = await res.json();
  return (body.datasets || []).map((d) => ({
    datasetId: d.datasetReference?.datasetId || "",
    location: d.location || "",
  }));
}

/**
 * Query recent events from a table using jobs.query.
 * @param {string} datasetId
 * @param {string} tableId
 * @param {number} [limit=10]
 * @param {string} [userId]
 * @returns {Promise<{ rows: Array<Record<string, unknown>> }>}
 */
export async function readRecentEvents(datasetId, tableId, limit = 10, userId) {
  if (!datasetId || !tableId) throw new Error("datasetId and tableId are required.");
  const uid = await resolveUserId(userId);
  const token = await getAccessToken(uid);
  const projectId = await getProjectId(uid);
  if (!projectId) throw new Error("project_id missing from service account key.");

  const n = Math.max(1, Math.min(Number(limit) || 10, 1000));
  const query = `SELECT * FROM \`${projectId}.${datasetId}.${tableId}\` ORDER BY _PARTITIONTIME DESC LIMIT ${n}`;

  const res = await fetch(`${BIGQUERY_API}/projects/${projectId}/jobs`, {
    method: "POST",
    headers: { Authorization: `Bearer ${token}`, "Content-Type": "application/json" },
    body: JSON.stringify({ configuration: { query: { query, useLegacySql: false } } }),
  });
  if (!res.ok) {
    const text = await res.text();
    throw new Error(`BigQuery jobs.query failed (${res.status}): ${text}`);
  }
  const job = await res.json();
  const jobId = job.jobReference?.jobId;
  if (!jobId) throw new Error("No jobId returned from BigQuery.");

  let status = job.status?.state;
  while (status !== "DONE") {
    await new Promise((r) => setTimeout(r, 1000));
    const pollRes = await fetch(`${BIGQUERY_API}/projects/${projectId}/jobs/${jobId}`, {
      headers: { Authorization: `Bearer ${token}` },
    });
    if (!pollRes.ok) {
      const text = await pollRes.text();
      throw new Error(`BigQuery job poll failed (${pollRes.status}): ${text}`);
    }
    const pollData = await pollRes.json();
    status = pollData.status?.state;
    if (pollData.status?.errorResult) {
      throw new Error(`BigQuery job error: ${JSON.stringify(pollData.status.errorResult)}`);
    }
  }

  const resultsRes = await fetch(
    `${BIGQUERY_API}/projects/${projectId}/jobs/${jobId}/getQueryResults?maxResults=${n}`,
    { headers: { Authorization: `Bearer ${token}` } },
  );
  if (!resultsRes.ok) {
    const text = await resultsRes.text();
    throw new Error(`BigQuery getQueryResults failed (${resultsRes.status}): ${text}`);
  }
  const resultsData = await resultsRes.json();
  const fields = (resultsData.schema?.fields || []).map((f) => f.name);
  const rows = (resultsData.rows || []).map((row) => {
    const obj = {};
    (row.f || []).forEach((cell, i) => { obj[fields[i] || `col_${i}`] = cell.v; });
    return obj;
  });
  return { rows };
}

/**
 * Run an arbitrary StandardSQL query against the project. Polls the job to
 * completion and pages through results up to maxResults rows.
 *
 * @param {{ query: string, maxResults?: number, userId?: string }} params
 * @returns {Promise<{ rows: Array<Record<string, unknown>>, schema: Array<{name:string,type:string}>, totalRows: number }>}
 */
export async function runQuery({ query, maxResults = 5000, userId } = {}) {
  if (!query || typeof query !== "string") throw new Error("query is required.");
  const uid = await resolveUserId(userId);
  // Running a query creates a job via jobs.insert, which the read-only scope does
  // NOT authorize (403 ACCESS_TOKEN_SCOPE_INSUFFICIENT). Request the full bigquery
  // scope — query results are still bounded by the SA's dataset-level IAM grants.
  const token = await getAccessToken(uid, "https://www.googleapis.com/auth/bigquery");
  const projectId = await getProjectId(uid);
  if (!projectId) throw new Error("project_id missing from service account key.");

  const cap = Math.max(1, Math.min(Number(maxResults) || 5000, 50000));

  // Use the SYNCHRONOUS jobs.query endpoint (POST /queries) rather than
  // jobs.insert + getQueryResults. The async path's getQueryResults call returns
  // an HTML 404 from Google's frontend (not a JSON API error) even with the
  // correct location echoed back — a routing quirk that made runQuery unusable.
  // jobs.query returns the first page of rows inline; we page the remainder with
  // getQueryResults ONLY if jobComplete and a pageToken is present, threading the
  // location the response reports.
  const queryRes = await fetch(`${BIGQUERY_API}/projects/${projectId}/queries`, {
    method: "POST",
    headers: { Authorization: `Bearer ${token}`, "Content-Type": "application/json" },
    body: JSON.stringify({
      query,
      useLegacySql: false,
      maxResults: Math.min(1000, cap),
      timeoutMs: 60000,
    }),
  });
  if (!queryRes.ok) {
    const text = await queryRes.text();
    throw new Error(`BigQuery jobs.query failed (${queryRes.status}): ${text}`);
  }
  const first = await queryRes.json();
  const jobId = first.jobReference?.jobId;
  const location = first.jobReference?.location || "";
  const locParam = (u) => { if (location) u.searchParams.set("location", location); };

  const rows = [];
  let schemaFields = (first.schema?.fields || []).map((f) => ({ name: f.name, type: f.type }));
  let totalRows = Number(first.totalRows || 0);
  const collect = (data) => {
    if (!schemaFields.length) {
      schemaFields = (data.schema?.fields || []).map((f) => ({ name: f.name, type: f.type }));
    }
    const fields = schemaFields.map((f) => f.name);
    for (const row of data.rows || []) {
      const obj = {};
      (row.f || []).forEach((cell, i) => { obj[fields[i] || `col_${i}`] = cell.v; });
      rows.push(obj);
      if (rows.length >= cap) break;
    }
  };
  collect(first);

  let pageToken = first.pageToken || "";
  let complete = first.jobComplete !== false;
  // If the query didn't finish within timeoutMs, poll via getQueryResults until done.
  while ((!complete || (pageToken && rows.length < cap)) && jobId) {
    const url = new URL(`${BIGQUERY_API}/projects/${projectId}/jobs/${jobId}/getQueryResults`);
    url.searchParams.set("maxResults", String(Math.min(1000, cap - rows.length)));
    url.searchParams.set("timeoutMs", "60000");
    locParam(url);
    if (pageToken) url.searchParams.set("pageToken", pageToken);
    const resultsRes = await fetch(url, { headers: { Authorization: `Bearer ${token}` } });
    if (!resultsRes.ok) {
      const text = await resultsRes.text();
      throw new Error(`BigQuery getQueryResults failed (${resultsRes.status}): ${text}`);
    }
    const data = await resultsRes.json();
    complete = data.jobComplete !== false;
    totalRows = Number(data.totalRows || totalRows);
    if (complete) collect(data);
    pageToken = data.pageToken || "";
    if (!pageToken && complete) break;
  }

  return { rows, schema: schemaFields, totalRows };
}

/**
 * Stream-insert rows into a BigQuery table.
 * @param {{ datasetId: string, tableId: string, rows: Array<{insertId?: string, json: Record<string, unknown>}>, userId?: string }} params
 * @returns {Promise<{ inserted: number, errors: Array<{index: number, errors: unknown}> }>}
 */
export async function insertRows({ datasetId, tableId, rows, userId }) {
  if (!datasetId || !tableId) throw new Error("datasetId and tableId are required.");
  if (!Array.isArray(rows) || rows.length === 0) return { inserted: 0, errors: [] };
  const uid = await resolveUserId(userId);
  const token = await getAccessToken(uid, "https://www.googleapis.com/auth/bigquery");
  const projectId = await getProjectId(uid);

  const res = await fetch(
    `${BIGQUERY_API}/projects/${projectId}/datasets/${datasetId}/tables/${tableId}/insertAll`,
    {
      method: "POST",
      headers: { Authorization: `Bearer ${token}`, "Content-Type": "application/json" },
      body: JSON.stringify({ rows, skipInvalidRows: false, ignoreUnknownValues: false }),
    },
  );
  if (!res.ok) {
    const text = await res.text();
    throw new Error(`insertAll failed (${res.status}): ${text}`);
  }
  const body = await res.json();
  const errors = (body.insertErrors || []).map((e) => ({ index: e.index, errors: e.errors }));
  return { inserted: rows.length - errors.length, errors };
}

/**
 * Check whether Google service account credentials are available.
 * @param {string} [userId]
 */
export async function checkCredentials(userId) {
  try {
    const uid = await resolveUserId(userId);
    const creds = await getGoogleCredentials(uid);
    if (!creds.serviceAccountJson) return { ok: false, projectId: "" };
    const key = typeof creds.serviceAccountJson === "string"
      ? JSON.parse(creds.serviceAccountJson)
      : creds.serviceAccountJson;
    return { ok: true, projectId: key.project_id || "" };
  } catch {
    return { ok: false, projectId: "" };
  }
}
```
<!-- /GUILDOS-FILE -->
