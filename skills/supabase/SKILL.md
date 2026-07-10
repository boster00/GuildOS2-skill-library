# supabase

<!-- GUILDOS-SKILL name="supabase" -->

<!-- GUILDOS-DESC -->
Operate against a Supabase project. UI-side actions (readTable/readLogs/readAPISettings/readStorageBuckets) drive the web dashboard via Browserclaw CDP. bulkUpload performs efficient idempotent COPY-based ingest directly against Postgres on port 5432. runSql + applyMigrations run arbitrary SQL and apply schema migrations (DDL: ALTER/CREATE TABLE) on the remote DB via the Management API + access token — the agent-side path for `supabase db push` when no CLI / DB password is available, and the ONLY way to run DDL (the service REST key cannot).
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/supabase/SKILL.md -->
```markdown
---
name: supabase
description: >-
  Operate against a Supabase project. UI-side actions
  (readTable/readLogs/readAPISettings/readStorageBuckets) drive the web dashboard
  via Browserclaw CDP. bulkUpload performs efficient idempotent COPY-based ingest
  directly against Postgres on port 5432. runSql + applyMigrations run arbitrary
  SQL and apply schema migrations (DDL: ALTER/CREATE TABLE) on the remote DB via
  the Management API + access token — the agent-side path for `supabase db push`
  when no CLI / DB password is available, and the ONLY way to run DDL (the
  service REST key cannot).
---

# Supabase

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/supabase/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto supabase <action>` (or read the source directly).

## Actions

- **applyMigrations** — Apply local supabase/migrations/*.sql to the remote DB and record each version in the schema_migrations
- **bulkUpload** — Idempotent bulk ingest of plain JS rows into a Supabase Postgres table via psql \COPY into a
- **readAPISettings** — Read API URL, anon key, and service role key from the Supabase settings page via CDP.
- **readLogs** — Read recent logs from the Supabase dashboard via CDP.
- **readStorageBuckets** — List storage buckets visible in the Supabase dashboard via CDP.
- **readTable** — Read rows from a database table via the Supabase REST API.
- **refBulkUploadPattern** — Locked discipline for bulk-ingest into Supabase Postgres. Read before adding ingest paths against a new table.
- **refDdlAndMigrations** — Locked discipline for running DDL / applying migrations against the remote Supabase DB. Read before any
- **runSql** — Run arbitrary SQL (including DDL: ALTER TABLE / CREATE TABLE) against the remote Postgres via the
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/supabase/index.js -->
```js
/**
 * Supabase skill book — PURE text-prompt registry.
 *
 * No imports, no functions. The executable JS lives in the paired weapon
 * (libs/weapon/supabase/index.js): the bare operations (readTable, readLogs,
 * readAPISettings, readStorageBuckets, bulkUpload, checkCredentials)
 * and the adventurer-execution wrappers (readTableAction, readLogsAction,
 * readAPISettingsAction, readStorageBucketsAction, bulkUploadAction) that wrap
 * them in the skillActionOk/Err envelope. The skill book describes WHAT each
 * action does and the locked discipline; the weapon is HOW.
 */

export const skillBook = {
  id: "supabase",
  title: "Supabase",
  description:
    "Operate against a Supabase project. UI-side actions (readTable/readLogs/readAPISettings/" +
    "readStorageBuckets) drive the web dashboard via Browserclaw CDP. The bulkUpload action " +
    "performs efficient idempotent COPY-based ingest directly against Postgres on port 5432. " +
    "runSql + applyMigrations run arbitrary SQL and apply schema migrations (DDL: ALTER/CREATE " +
    "TABLE) on the remote DB via the Management API + access token — the agent-side path for " +
    "`supabase db push` when no CLI / DB password is available, and the ONLY way to run DDL " +
    "(the service REST key cannot).",
  steps: [],
  toc: {
    readTable: {
      description: "Read rows from a database table via the Supabase REST API.",
      input: { table: "string — table name", limit: "int, default 50", filter: "string, optional PostgREST filter e.g. 'status=eq.active'" },
      output: { rows: "array of row objects", total: "int — total row count if available" },
      howTo:
        "Read rows from a Supabase table via PostgREST. Calls the weapon's readTableAction " +
        "(libs/weapon/supabase/index.js), which hits {NEXT_PUBLIC_SUPABASE_URL}/rest/v1/{table}?limit={limit} " +
        "with the service-role key and an optional PostgREST filter, then returns { rows, total } inside the " +
        "standard skillActionOk envelope. `table` is required. " +
        "NOTE: offline / no-MCP-connected fallback — prefer the Supabase MCP when available.",
    },
    readLogs: {
      description: "Read recent logs from the Supabase dashboard via CDP.",
      input: { service: "string — 'postgres' | 'auth' | 'realtime' | 'edge-functions' | 'postgrest', default 'postgres'", limit: "int, default 50" },
      output: { logs: "array of log entries", service: "string" },
      howTo:
        "Read recent Supabase project logs by driving the web dashboard via Browserclaw CDP. Calls the " +
        "weapon's readLogsAction, which navigates to /project/{ref}/logs/{service}, captures the log API " +
        "responses, and returns { logs, service }. service ∈ postgres | auth | realtime | edge-functions | " +
        "postgrest (default postgres). NOTE: offline / no-MCP-connected fallback — prefer the Supabase MCP when available.",
    },
    readAPISettings: {
      description: "Read API URL, anon key, and service role key from the Supabase settings page via CDP.",
      input: {},
      output: { text: "string — extracted page text", apiKeys: "object — raw API keys response if captured" },
      howTo:
        "Read the project's API URL and keys from the Supabase settings page via Browserclaw CDP. Calls the " +
        "weapon's readAPISettingsAction, which navigates to /project/{ref}/settings/api, captures the /api-keys " +
        "response, and returns { text, apiKeys }. NOTE: offline / no-MCP-connected fallback — prefer the Supabase MCP when available.",
    },
    readStorageBuckets: {
      description: "List storage buckets visible in the Supabase dashboard via CDP.",
      input: {},
      output: { buckets: "array of bucket objects" },
      howTo:
        "List the storage buckets visible in the Supabase dashboard via Browserclaw CDP. Calls the weapon's " +
        "readStorageBucketsAction, which navigates to /project/{ref}/storage/buckets, captures the /storage " +
        "response, and returns { buckets }. NOTE: offline / no-MCP-connected fallback — prefer the Supabase MCP when available.",
    },
    bulkUpload: {
      description:
        "Idempotent bulk ingest of plain JS rows into a Supabase Postgres table via psql \\COPY into a " +
        "TEMP staging table, then INSERT … ON CONFLICT DO UPDATE into the target. " +
        "Locked discipline: see refBulkUploadPattern.",
      input: {
        table: "string — target table name",
        rows: "array<object> — column→value rows; column list inferred from first row's keys",
        conflictKeys: "array<string> — ON CONFLICT target; REQUIRED (no blind inserts)",
        mergeExpr: "string, optional — SQL for DO UPDATE SET; defaults to overwrite-with-incoming on every non-conflict column",
        connectionUrl: "string, optional — full postgresql:// URL; defaults from SUPABASE_DB_URL or SUPABASE_DB_{HOST,PASSWORD,USER,PORT,NAME} (port defaults 5432, NEVER 6543 — the pooler can't run \\COPY)",
        dryRun: "bool, optional — print SQL + CSV path and skip execution",
      },
      output: { ok: "bool", table: "string", rows_in: "int", affected: "int — rows inserted+updated", csv_path: "string", duration_ms: "int" },
      howTo:
        "Idempotent bulk ingest of plain JS rows into a Supabase Postgres table. Calls the weapon's " +
        "bulkUploadAction (libs/weapon/supabase/index.js), which: writes the rows to a CSV in ./tmp; opens a " +
        "psql connection on port 5432; CREATE TEMP TABLE staging (LIKE <target> INCLUDING DEFAULTS); \\COPY the " +
        "CSV into staging; INSERT … SELECT FROM staging ON CONFLICT (<conflictKeys>) DO UPDATE SET <mergeExpr or " +
        "overwrite-all-non-conflict-columns>; and returns { ok, table, rows_in, affected, csv_path, duration_ms }. " +
        "conflictKeys is REQUIRED (no blind inserts). YOU MUST OBEY refBulkUploadPattern below before adding any " +
        "ingest path against a new table.",
    },
    runSql: {
      description:
        "Run arbitrary SQL (including DDL: ALTER TABLE / CREATE TABLE) against the remote Postgres via the " +
        "Supabase Management API. The ONLY in-repo path that can run DDL — the service REST key cannot.",
      input: { sql: "string — one or more `;`-separated statements" },
      output: { ok: "bool", rows: "array — result rows (empty for DDL)" },
      howTo:
        "Run arbitrary SQL against the remote DB. Calls the weapon's runSqlAction " +
        "(libs/weapon/supabase/index.js), which POSTs { query: sql } to " +
        "https://api.supabase.com/v1/projects/{ref}/database/query with Bearer SUPABASE_ACCESS_TOKEN (a personal " +
        "access token from supabase.com/dashboard/account/tokens) and returns { ok, rows }. Use for one-off DDL, " +
        "ledger fixes, or reads that need raw SQL. ref comes from NEXT_PUBLIC_SUPABASE_URL. See refDdlAndMigrations.",
    },
    applyMigrations: {
      description:
        "Apply local supabase/migrations/*.sql to the remote DB and record each version in the schema_migrations " +
        "ledger — the agent-side equivalent of `supabase db push`.",
      input: {
        versions: "array<string>, optional — version prefixes e.g. ['20260612160000']; omit = all migration files",
        record: "bool, optional (default true) — also write the supabase_migrations.schema_migrations ledger row",
      },
      output: { ok: "bool", results: "array<{file, version, applied, error?}>" },
      howTo:
        "Apply migration files to the remote DB via the Management API. Calls the weapon's applyMigrationsAction, " +
        "which reads each matching supabase/migrations/*.sql, runs it via runSql, then upserts (version, name) into " +
        "supabase_migrations.schema_migrations (ON CONFLICT DO NOTHING) so `supabase db push` / `migration list` see " +
        "it applied. Migrations MUST be idempotent (IF NOT EXISTS) — this re-runs the file body, not a ledger diff. " +
        "Stops on the first failure. See refDdlAndMigrations before using.",
    },
    refDdlAndMigrations: {
      description:
        "Locked discipline for running DDL / applying migrations against the remote Supabase DB. Read before any " +
        "`supabase db push`-shaped task.",
      howTo:
        "How to apply schema changes / migrations to the remote Supabase DB. Established 2026-06-15.\n" +
        "\n" +
        "RULE 1 — DDL goes through runSql / applyMigrations (Management API), NOT the service REST key.\n" +
        "  WHY: PostgREST (the SUPABASE_SECRETE_KEY path) can only read/write rows; ALTER/CREATE TABLE returns 42501 " +
        "or 'column ... does not exist'. The Management API SQL endpoint runs true DDL.\n" +
        "\n" +
        "RULE 2 — Auth is SUPABASE_ACCESS_TOKEN (a personal access token, sbp_...), in .env.local. No DB password " +
        "needed for this path.\n" +
        "  WHY: The PAT authenticates the Management API directly; it is full-account scoped, so treat it as a secret " +
        "and never echo it. Generate at supabase.com/dashboard/account/tokens.\n" +
        "\n" +
        "RULE 3 — The CLI (`supabase db push` / `migration list`) DOES work here, but ONLY with " +
        "`--db-url <SESSION_POOLER_URL>` (aws-0-<region>.pooler.supabase.com:5432). The CLI's default direct host " +
        "db.<ref>.supabase.co is UNREACHABLE on this box (tls error EOF — IPv4-only/network quirk).\n" +
        "  WHY: Same 5432-session-pooler reason as refBulkUploadPattern RULE 1. Omitting --db-url makes the CLI hang/fail.\n" +
        "\n" +
        "RULE 4 — Migrations must be idempotent (ADD COLUMN/CREATE TABLE IF NOT EXISTS). applyMigrations re-runs the " +
        "file body, so a non-idempotent migration double-applied will error.\n" +
        "  WHY: There is no local-vs-remote diff step; idempotency is what makes re-running safe.\n" +
        "\n" +
        "RULE 5 — After applying out-of-band (runSql/applyMigrations), ALWAYS record the version in " +
        "supabase_migrations.schema_migrations (applyMigrations does this for you). Otherwise a later `supabase db push` " +
        "re-attempts the migration.\n" +
        "  WHY: The ledger is the CLI's source of truth for what's applied; an unrecorded-but-applied migration is drift.",
    },
    refBulkUploadPattern: {
      description:
        "Locked discipline for bulk-ingest into Supabase Postgres. Read before adding ingest paths against a new table.",
      howTo:
        "Locked discipline for any bulkUpload work against Supabase Postgres. Each rule has a one-line rationale. " +
        "Read and obey. This is the reason this skill book exists.\n" +
        "\n" +
        "RULE 1 — ALWAYS connect on port 5432 (direct db.<ref>.supabase.co:5432 OR Supabase session pooler " +
        "aws-0-<region>.pooler.supabase.com:5432). NEVER the transaction pooler on 6543.\n" +
        "  WHY: \\COPY needs the stateful frontend/backend protocol. Direct + session-pooler both keep one " +
        "frontend bound to one backend; the 6543 transaction pooler multiplexes and breaks COPY and named " +
        "prepared statements. On networks without IPv6 to Supabase, the session pooler is the supported 5432 path.\n" +
        "\n" +
        "RULE 2 — ALWAYS ingest via psql \\COPY. NEVER row-by-row PostgREST POST.\n" +
        "  WHY: Row-by-row POST has blown the project's disk-IO quota before; a single COPY stream is 1–2 orders " +
        "of magnitude cheaper.\n" +
        "\n" +
        "RULE 3 — ALWAYS COPY into a TEMP staging table created with CREATE TEMP TABLE staging LIKE <target> " +
        "INCLUDING DEFAULTS, then INSERT…SELECT into the target.\n" +
        "  WHY: Keeps the prod table out of the long-running parse/load phase, lets you abort by dropping the " +
        "staging, and avoids hardcoding column types.\n" +
        "\n" +
        "RULE 4 — ALWAYS specify ON CONFLICT (<conflictKeys>) DO UPDATE. NEVER blind INSERT.\n" +
        "  WHY: Blind inserts on retry duplicate rows or violate UNIQUE constraints and corrupt cache tables; the " +
        "conflict target makes the action idempotent.\n" +
        "\n" +
        "RULE 5 — ALWAYS return { rows_in, affected } counts and surface them to the operator.\n" +
        "  WHY: Silent ingestion is a discipline failure — operator cannot tell partial loads from full ones; " +
        "counts catch row-loss and column-mismatch bugs early.\n" +
        "\n" +
        "RULE 6 — For >1M rows OR schema-shape transformations, OPT-IN to validateLocally: true (Docker PG → " +
        "ingest+inspect → pg_dump → re-COPY to Supabase). Default skips this.\n" +
        "  WHY: Catches schema/encoding/quoting bugs before they touch prod; cost is high enough that we don't " +
        "pay it for small idempotent loads.",
    },
  },
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/browserclaw/cdp.js -->
```js
/**
 * Browserclaw CDP weapon — headless scraper infrastructure.
 *
 * Use cases (NOT for agent-driven browsing):
 *  - Server-side headless scrapers that run deterministically: `libs/weapon/linkedin`,
 *    `libs/weapon/pubcompare`. These are weapons in the strict sense — no agent in
 *    the loop, scripted selectors, batch execution.
 *  - Not for local Claude's interactive browsing (use Claude-in-Chrome MCP instead).
 *  - Not reachable from cloud Cursor agents (port 9222 is local-only).
 *
 * Architecture: Chrome persistent profile at `~/.guildos-cdp-profile` on port 9222.
 * `ensureCdpChrome()` spawns Chrome if not already running; `executeSteps(steps)` drives it.
 * Auth cookies from `playwright/.auth/user.json` are injected on fresh launch.
 */

import { chromium } from "playwright-core";
import { exec } from "node:child_process";
import { join } from "node:path";
import { homedir } from "node:os";
import { readFileSync, existsSync } from "node:fs";

export const CDP_PORT = 9222;
export const CDP_URL = `http://localhost:${CDP_PORT}`;
export const CDP_PROFILE_DIR =
  process.env.GUILDOS_CDP_PROFILE_DIR ||
  join(homedir(), ".guildos-cdp-profile");
export const AUTH_STATE_PATH =
  process.env.GUILDOS_AUTH_STATE_PATH ||
  join(process.cwd(), "playwright/.auth/user.json");

const STEP_TIMEOUT = 25_000;

// ---------------------------------------------------------------------------
// Chrome launch helpers
// ---------------------------------------------------------------------------

function getChromeExe() {
  if (process.env.CHROME_EXECUTABLE_PATH) return process.env.CHROME_EXECUTABLE_PATH;
  if (process.platform === "win32") {
    return `${process.env.LOCALAPPDATA}\\Google\\Chrome\\Application\\chrome.exe`;
  }
  if (process.platform === "darwin") {
    return "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome";
  }
  return "/usr/local/bin/google-chrome";
}

/**
 * Returns true if CDP Chrome is reachable on port 9222.
 */
export async function isCdpRunning() {
  try {
    const resp = await fetch(`${CDP_URL}/json/version`, {
      signal: AbortSignal.timeout(1500),
    });
    return resp.ok;
  } catch {
    return false;
  }
}

/**
 * Inject cookies from playwright/.auth/user.json into the running CDP session.
 * Silently skips if the file doesn't exist.
 */
async function _injectAuthState(authPath = AUTH_STATE_PATH) {
  if (!existsSync(authPath)) return;
  try {
    const { cookies } = JSON.parse(readFileSync(authPath, "utf8"));
    if (!cookies?.length) return;
    const browser = await chromium.connectOverCDP(CDP_URL);
    const ctx = browser.contexts()[0] ?? await browser.newContext();
    await ctx.addCookies(cookies);
    await browser.close();
  } catch {
    // non-fatal — profile cookies may still be valid
  }
}

/**
 * Launch Chrome on port 9222 with the dedicated CDP profile if not already running.
 * No-op if CDP is already reachable. Returns { launched, msg }.
 */
export async function ensureCdpChrome({ profileDir = CDP_PROFILE_DIR } = {}) {
  if (await isCdpRunning()) {
    return { launched: false, msg: "CDP Chrome already running on port 9222" };
  }

  const exe = getChromeExe();
  const flags = [
    `--remote-debugging-port=${CDP_PORT}`,
    `--user-data-dir="${profileDir}"`,
    "--no-first-run",
    "--no-default-browser-check",
    "--disable-blink-features=AutomationControlled",
  ].join(" ");

  if (process.platform === "win32") {
    exec(`start "" "${exe}" ${flags}`);
  } else {
    exec(`"${exe}" ${flags} &`);
  }

  // Poll up to 5 seconds for CDP to become reachable
  for (let i = 0; i < 10; i++) {
    await new Promise((r) => setTimeout(r, 500));
    if (await isCdpRunning()) {
      await _injectAuthState();
      return { launched: true, msg: `CDP Chrome launched on port ${CDP_PORT}` };
    }
  }

  throw new Error(`Chrome launched but CDP not reachable on port ${CDP_PORT} after 5s. Check that Chrome is installed at: ${exe}`);
}

// ---------------------------------------------------------------------------
// Step execution
// ---------------------------------------------------------------------------

/**
 * Execute browser steps by connecting to the existing CDP Chrome on port 9222.
 * Does NOT close Chrome — it remains running for subsequent calls.
 *
 * @param {Array<{action: string, item: string, [key: string]: unknown}>} steps
 * @param {{ storageState?: string }} opts
 */
export async function executeSteps(steps, opts = {}) {
  const overallStart = Date.now();
  const stepResults = [];

  const browser = await chromium.connectOverCDP(CDP_URL);

  // Reuse the first existing context (the default CDP context), or create one
  const existingContexts = browser.contexts();
  const context = existingContexts.length > 0
    ? existingContexts[0]
    : await browser.newContext(opts.storageState ? { storageState: opts.storageState } : {});

  const page = await context.newPage();

  try {
    for (let i = 0; i < steps.length; i++) {
      const step = steps[i];
      const stepStart = Date.now();
      let result;
      try {
        result = await executeOneStep(page, step);
      } catch (e) {
        result = { ok: false, error: e.message };
      }
      stepResults.push({
        index: i,
        action: step.action,
        elapsed: Date.now() - stepStart,
        ok: result?.ok ?? false,
        value: result?.value ?? null,
        error: result?.error ?? null,
        ...(step.item ? { item: step.item } : {}),
        ...(result?.buffer ? { buffer: result.buffer } : {}),
      });
      if (!result?.ok) break;
    }
  } finally {
    await page.close().catch(() => {});
    // browser.close() disconnects the Playwright client — Chrome keeps running
    await browser.close().catch(() => {});
  }

  return {
    ok: stepResults.every((r) => r.ok),
    totalElapsed: Date.now() - overallStart,
    steps: stepResults,
  };
}

async function executeOneStep(page, step) {
  const timeout = Math.min(STEP_TIMEOUT, Number(step.timeout) || STEP_TIMEOUT);

  switch (step.action) {
    case "navigate":
      await page.goto(step.url, { waitUntil: "domcontentloaded", timeout });
      return { ok: true, value: step.url };

    case "wait": {
      const ms = Math.min(30, Math.max(0, Number(step.seconds) || 1)) * 1000;
      await page.waitForTimeout(ms);
      if (step.selector) {
        await page.waitForSelector(step.selector, { timeout: 30_000 }).catch(() => {});
      }
      return { ok: true, value: `waited ${step.seconds}s` };
    }

    case "typeText": {
      const sel = step.selector || "input";
      await page.waitForSelector(sel, { timeout: 5000 });
      if (step.clearContent !== false) await page.fill(sel, "");
      await page.type(sel, step.text || "", { delay: 30 });
      return { ok: true, value: step.text };
    }

    case "click": {
      const sel = step.selector || "button";
      await page.waitForSelector(sel, { timeout: 5000 });
      await page.click(sel);
      return { ok: true, value: "clicked" };
    }

    case "pressKey": {
      const key = step.key || "Enter";
      if (step.selector) await page.press(step.selector, key);
      else await page.keyboard.press(key);
      return { ok: true, value: `pressed:${key}` };
    }

    case "get":
    case "obtainText": {
      const sel = step.selector || "body";
      await page.waitForSelector(sel, { timeout: 5000 }).catch(() => {});
      if (step.getAll) {
        const values = await page.$$eval(
          sel,
          (els, attr) => els.map((el) =>
            attr && attr !== "innerText"
              ? el.getAttribute(attr) || ""
              : (el.innerText || el.textContent || "").trim()
          ),
          step.attribute || null,
        ).catch(() => []);
        return { ok: true, value: values };
      }
      const attr = step.attribute;
      if (attr && !["innerText", "textContent"].includes(attr)) {
        if (attr === "innerHTML") {
          const h = await page.$eval(sel, (el) => el.innerHTML).catch(() => "");
          return { ok: true, value: h.slice(0, 5000) };
        }
        if (attr === "outerHTML") {
          const h = await page.$eval(sel, (el) => el.outerHTML).catch(() => "");
          return { ok: true, value: h.slice(0, 5000) };
        }
        if (attr === "value") {
          const v = await page.$eval(sel, (el) => el.value || "").catch(() => "");
          return { ok: true, value: v.slice(0, 2000) };
        }
        const v = await page.$eval(sel, (el, a) => el.getAttribute(a) || "", attr).catch(() => "");
        return { ok: true, value: v.slice(0, 2000) };
      }
      const text = await page.$eval(
        sel,
        (el) => el.tagName === "A" ? el.href : (el.innerText || el.textContent || "").trim(),
      ).catch(() => "");
      return { ok: true, value: text.slice(0, 2000) };
    }

    case "obtainHtml": {
      const sel = step.selector || "body";
      await page.waitForSelector(sel, { timeout: 5000 }).catch(() => {});
      const html = await page.$eval(sel, (el) => el.innerHTML).catch(() => "");
      return { ok: true, value: html.slice(0, 5000) };
    }

    case "getUrl":
      return { ok: true, value: page.url() };

    case "getPageInfo": {
      const title = await page.title();
      return { ok: true, value: { title, url: page.url() } };
    }

    case "evaluate": {
      const val = await page.evaluate(step.expression || "document.title");
      return { ok: true, value: typeof val === "string" ? val.slice(0, 2000) : val };
    }

    case "screenshot": {
      const buf = await page.screenshot({
        type: "png",
        fullPage: step.fullPage === true,
        path: step.path || undefined,
      });
      return {
        ok: true,
        value: step.path ? `screenshot saved: ${step.path}` : `screenshot:${buf.length} bytes`,
        ...(step.path ? {} : { buffer: buf }),
      };
    }

    default:
      return { ok: false, error: `Unknown action: ${step.action}` };
  }
}

// ---------------------------------------------------------------------------
// Credential check
// ---------------------------------------------------------------------------

export async function checkCredentials() {
  const running = await isCdpRunning();
  if (!running) {
    return {
      ok: false,
      msg: `CDP Chrome not running on port ${CDP_PORT}. Call ensureCdpChrome() or launch Chrome with --remote-debugging-port=${CDP_PORT} --user-data-dir="${CDP_PROFILE_DIR}"`,
    };
  }
  try {
    const resp = await fetch(`${CDP_URL}/json/version`);
    const info = await resp.json();
    return { ok: true, msg: `CDP Chrome reachable: ${info.Browser || "Chrome"} on port ${CDP_PORT}` };
  } catch (e) {
    return { ok: false, msg: `CDP reachable but version check failed: ${e.message}` };
  }
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/supabase/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/weapon/supabase`

Rationale layer (per CLAUDE.md §6). Append; supersede in place when a decision
changes.

---

## 2026-06-09 — `bulkUpload` rejects the pooler port 6543 and requires direct port 5432

**Decision.** `bulkUpload` hard-errors if pointed at port 6543 (the Supabase
PgBouncer pooler); it accepts ONLY port 5432 (direct frontend connection).

**Rejected alternative.** Accept both ports and let the caller use whatever
`SUPABASE_DB_URL` happens to carry.

**Why.** PostgreSQL's `COPY` meta-command needs the stateful frontend protocol.
PgBouncer-class poolers use stateless session pooling, which breaks `COPY` — the
failure surfaces as a cryptic "syntax error", which callers misattribute to data
shape and chase for hours. A hard guard names the real (transport-layer) cause.

**Consequence if violated.** Bulk ingests fail with misleading parse errors; the
bug is blamed on the data instead of the connection.

---

## 2026-06-15 — DDL / migrations go through the Management API (`runSql`/`applyMigrations`), not psql / `supabase db push`

**Decision.** Schema changes against the remote DB are applied with `runSql` /
`applyMigrations`, which POST SQL to the Supabase **Management API**
(`api.supabase.com/v1/projects/{ref}/database/query`) authenticated by
`SUPABASE_ACCESS_TOKEN` (a personal access token). `applyMigrations` also records
each version in `supabase_migrations.schema_migrations`.

**Rejected alternative.** Tell the user to run `supabase db push` (the "obvious"
path), or shell out to the CLI / psql by default.

**Why.** Three native paths fail or are unavailable on this box: (1) the service
REST key (`SUPABASE_SECRETE_KEY`) **cannot run DDL** — PostgREST only does
row-level reads/writes; (2) `supabase db push` needs the Supabase CLI *and* the DB
password, and the CLI's default direct host `db.<ref>.supabase.co` is
**unreachable here** (tls error EOF — IPv4-only/network quirk); (3) even with the
password, the CLI works only when forced onto the session pooler via
`--db-url aws-0-<region>.pooler.supabase.com:5432`. The Management API + PAT
sidesteps all of that: no CLI, no DB password, no direct host. This is the
recurring "run db push to apply migrations" ask — it kept resurfacing because the
default path is broken on this machine; the PAT path is the durable fix.

**Consequence if violated.** A future agent re-derives "no DDL path exists" (an
archived quest literally recorded that dead end), declines the task, or hand-rolls
an inline Management-API curl instead of using the codified action — re-paying the
discovery cost every time. If a migration is applied via `runSql` but the
`schema_migrations` ledger row is skipped, a later `supabase db push` re-attempts
it (harmless only if idempotent).
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/supabase/index.js -->
```js
/**
 * supabase weapon — Supabase operations.
 *
 * UI-side actions (readTable/readLogs/readAPISettings/readStorageBuckets) drive
 * the Supabase dashboard at app.supabase.com via Browserclaw CDP and PostgREST.
 * The bulkUpload action runs psql \COPY directly against the project's Postgres
 * on port 5432 (NOT the 6543 pooler — \COPY needs the stateful frontend
 * protocol). See libs/skill_book/supabase/index.js → refBulkUploadPattern for
 * the locked discipline.
 *
 * Auth (UI actions): session cookies saved in ~/.guildos-cdp-profile via auth-capture.mjs.
 * Auth (bulkUpload): DB password via env (SUPABASE_DB_URL or SUPABASE_DB_PASSWORD + host parts).
 */

export const toc = {
  readTable: "Read rows from a Supabase table via PostgREST.",
  readLogs: "Read Supabase project logs (postgres/auth/realtime/edge-functions) via CDP dashboard.",
  readAPISettings: "Read API URL and keys from the Supabase settings page via CDP.",
  readStorageBuckets: "List storage buckets visible in the Supabase dashboard via CDP.",
  bulkUpload: "Idempotent COPY-based bulk ingest into a Supabase Postgres table via psql + TEMP staging + ON CONFLICT.",
  runSql: "Run arbitrary SQL (incl. DDL: ALTER/CREATE TABLE) against the remote Postgres via the Supabase Management API + a personal access token. The service REST key CANNOT do DDL; this can.",
  applyMigrations: "Apply local supabase/migrations/*.sql files to the remote DB via runSql, then record each version in the schema_migrations ledger (the agent-side equivalent of `supabase db push`).",
  // Skill-book-facing action wrappers: same operations, wrapped in the
  // skillActionOk/Err envelope + (userId, input) arg-normalization the
  // adventurer-execution pipeline expects. The skill book (pure prompt registry)
  // no longer holds these — the registry imports them from here.
  readTableAction: "Enveloped readTable for adventurer execution (skillActionOk/Err).",
  readLogsAction: "Enveloped readLogs for adventurer execution (skillActionOk/Err).",
  readAPISettingsAction: "Enveloped readAPISettings for adventurer execution (skillActionOk/Err).",
  readStorageBucketsAction: "Enveloped readStorageBuckets for adventurer execution (skillActionOk/Err).",
  bulkUploadAction: "Enveloped bulkUpload for adventurer execution (skillActionOk/Err).",
  runSqlAction: "Enveloped runSql for adventurer execution (skillActionOk/Err).",
  applyMigrationsAction: "Enveloped applyMigrations for adventurer execution (skillActionOk/Err).",
};

// Relative (not "@/…") so the weapon still loads under plain node — the
// bulkUpload smoke test imports this module directly without Next alias resolution.
import { skillActionOk, skillActionErr } from "../../skill_book/actionResult.js";

// UI actions (readLogs/readAPISettings/readStorageBuckets) load Browserclaw+playwright
// lazily — bulkUpload doesn't need them, and they pull a Next-aliased import path.
import { spawn } from "node:child_process";
import { mkdirSync, writeFileSync, existsSync, readFileSync, readdirSync } from "node:fs";
import path from "node:path";

const BASE = "https://supabase.com/dashboard";
const MGMT_API = "https://api.supabase.com/v1";

function getAccessToken() {
  const tok = process.env.SUPABASE_ACCESS_TOKEN;
  if (!tok) {
    throw new Error(
      "SUPABASE_ACCESS_TOKEN not set — generate a personal access token at " +
      "https://supabase.com/dashboard/account/tokens and put it in .env.local. See refDdlAndMigrations.",
    );
  }
  return tok;
}

async function getProjectRef(userId) {
  // Default to the project ref from env
  const ref = process.env.NEXT_PUBLIC_SUPABASE_URL?.match(/https:\/\/([a-z0-9]+)\.supabase\.co/)?.[1];
  if (ref) return ref;
  throw new Error("Cannot determine Supabase project ref from NEXT_PUBLIC_SUPABASE_URL");
}

async function withSupabasePage(fn) {
  const { ensureCdpChrome } = await import("@/libs/weapon/browserclaw/cdp");
  const { chromium } = await import("playwright-core");
  await ensureCdpChrome();
  const browser = await chromium.connectOverCDP("http://localhost:9222");
  const ctx = browser.contexts()[0];
  const page = await ctx.newPage();
  try {
    return await fn(page, browser);
  } finally {
    await page.close();
    await browser.close();
  }
}

async function ensureLoggedIn(page) {
  await page.goto(`${BASE}/projects`, { waitUntil: "domcontentloaded", timeout: 20000 });
  await page.waitForTimeout(3000);
  if (page.url().includes("/sign-in") || page.url().includes("/login")) {
    throw new Error("Supabase UI not logged in — run scripts/auth-capture.mjs to capture session");
  }
}

/**
 * Read rows from a table via the Supabase REST API (PostgREST).
 * Navigates the web UI to confirm the table exists; reads data via REST.
 */
export async function readTable({ table, limit = 50, filter } = {}, userId) {
  if (!table) throw new Error('"table" is required');

  const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL;
  const serviceKey = process.env.SUPABASE_SECRETE_KEY;
  if (!supabaseUrl || !serviceKey) throw new Error("NEXT_PUBLIC_SUPABASE_URL or SUPABASE_SECRETE_KEY not set");

  let url = `${supabaseUrl}/rest/v1/${table}?limit=${limit}`;
  if (filter) url += `&${filter}`;

  const resp = await fetch(url, {
    headers: {
      apikey: serviceKey,
      Authorization: `Bearer ${serviceKey}`,
      Prefer: "count=exact",
    },
  });
  if (!resp.ok) {
    const text = await resp.text();
    throw new Error(`Supabase readTable ${table} failed (${resp.status}): ${text}`);
  }
  const rows = await resp.json();
  const total = resp.headers.get("content-range")?.split("/")?.[1];
  return { rows, total: total ? Number(total) : undefined };
}

/**
 * Read Supabase project logs from the web UI via CDP.
 * service: "postgres" | "auth" | "realtime" | "edge-functions" | "postgrest"
 */
export async function readLogs({ service = "postgres", limit = 50 } = {}, userId) {
  const ref = await getProjectRef(userId);
  return withSupabasePage(async (page) => {
    await ensureLoggedIn(page);

    const logsUrl = `${BASE}/project/${ref}/logs/${service}`;
    await page.goto(logsUrl, { waitUntil: "domcontentloaded", timeout: 20000 });
    await page.waitForTimeout(5000);

    const text = await page.innerText("body").catch(() => "");
    if (text.includes("No logs") || text.includes("no events")) return { logs: [], service };

    // Capture the log API responses
    const logs = [];
    page.on("response", async (resp) => {
      if (resp.url().includes("/logs") && resp.status() === 200) {
        try {
          const ct = resp.headers()["content-type"] || "";
          if (ct.includes("json")) {
            const body = await resp.json().catch(() => null);
            if (body?.result) logs.push(...(body.result || []));
          }
        } catch { /* ignore */ }
      }
    });

    // Reload to capture the log fetch
    await page.reload({ waitUntil: "domcontentloaded", timeout: 20000 });
    await page.waitForTimeout(5000);

    return { logs: logs.slice(0, limit), service };
  });
}

/**
 * Read the project's API settings (URL, anon key, service role key) from the web UI.
 */
export async function readAPISettings({} = {}, userId) {
  const ref = await getProjectRef(userId);
  return withSupabasePage(async (page) => {
    await ensureLoggedIn(page);

    const responses = {};
    page.on("response", async (resp) => {
      if (resp.url().includes("/api-keys") && resp.status() === 200) {
        try {
          const body = await resp.json().catch(() => null);
          if (body) responses.apiKeys = body;
        } catch { /* ignore */ }
      }
    });

    await page.goto(`${BASE}/project/${ref}/settings/api`, { waitUntil: "domcontentloaded", timeout: 20000 });
    await page.waitForTimeout(5000);

    const text = await page.innerText("body").catch(() => "");
    return { text: text.slice(0, 2000), apiKeys: responses.apiKeys };
  });
}

/**
 * Read storage buckets from the web UI.
 */
export async function readStorageBuckets({} = {}, userId) {
  const ref = await getProjectRef(userId);
  return withSupabasePage(async (page) => {
    await ensureLoggedIn(page);

    const buckets = [];
    page.on("response", async (resp) => {
      if (resp.url().includes("/storage") && resp.status() === 200) {
        try {
          const ct = resp.headers()["content-type"] || "";
          if (ct.includes("json")) {
            const body = await resp.json().catch(() => null);
            if (Array.isArray(body)) buckets.push(...body);
          }
        } catch { /* ignore */ }
      }
    });

    await page.goto(`${BASE}/project/${ref}/storage/buckets`, { waitUntil: "domcontentloaded", timeout: 20000 });
    await page.waitForTimeout(5000);

    return { buckets };
  });
}

export async function checkCredentials(input = {}, userId) {
  try {
    return withSupabasePage(async (page) => {
      await page.goto(`${BASE}/projects`, { waitUntil: "domcontentloaded", timeout: 20000 });
      await page.waitForTimeout(3000);
      const url = page.url();
      if (url.includes("/sign-in") || url.includes("/login")) {
        return { ok: false, msg: "Supabase UI not logged in — run scripts/auth-capture.mjs" };
      }
      const text = await page.innerText("body").catch(() => "");
      const hasProjects = text.includes("project") || text.includes("Project");
      return { ok: hasProjects, msg: hasProjects ? "Supabase UI session active" : "Supabase UI loaded but no projects found" };
    });
  } catch (e) {
    return { ok: false, msg: `Supabase UI CDP error: ${e.message}` };
  }
}

// ---------------------------------------------------------------------------
// bulkUpload — psql \COPY into TEMP staging then INSERT … ON CONFLICT DO UPDATE.
// See libs/skill_book/supabase/index.js → refBulkUploadPattern for the locked
// discipline (port 5432 only, COPY only, TEMP staging only, ON CONFLICT only,
// always return rows_in + affected counts).
// ---------------------------------------------------------------------------

function csvEscape(v) {
  if (v === null || v === undefined) return "";
  let s;
  if (typeof v === "string") s = v;
  else if (typeof v === "number" || typeof v === "boolean") s = String(v);
  else if (v instanceof Date) s = v.toISOString();
  else s = JSON.stringify(v); // objects + arrays → JSON literal (jsonb-friendly)
  if (/[",\r\n]/.test(s)) return `"${s.replace(/"/g, '""')}"`;
  return s;
}

function buildConnectionUrl() {
  if (process.env.SUPABASE_DB_URL) {
    const u = process.env.SUPABASE_DB_URL;
    // Hard guardrail: refuse pooler port 6543 — \COPY breaks.
    if (/:6543\b/.test(u)) {
      throw new Error(
        "SUPABASE_DB_URL targets pooler port 6543; bulkUpload requires direct port 5432. " +
        "See refBulkUploadPattern."
      );
    }
    return u;
  }
  const host = process.env.SUPABASE_DB_HOST;
  const password = process.env.SUPABASE_DB_PASSWORD;
  const user = process.env.SUPABASE_DB_USER || "postgres";
  const port = process.env.SUPABASE_DB_PORT || "5432";
  const db = process.env.SUPABASE_DB_NAME || "postgres";
  if (port === "6543") {
    throw new Error("SUPABASE_DB_PORT=6543 (pooler) — bulkUpload requires 5432.");
  }
  if (!host || !password) {
    throw new Error(
      "bulkUpload: provide connectionUrl, OR SUPABASE_DB_URL, OR SUPABASE_DB_HOST + SUPABASE_DB_PASSWORD env."
    );
  }
  return `postgresql://${user}:${encodeURIComponent(password)}@${host}:${port}/${db}`;
}

function quoteIdent(name) {
  if (!/^[a-zA-Z_][a-zA-Z0-9_]*$/.test(name)) {
    throw new Error(`Invalid identifier: ${name}`);
  }
  return name;
}

/**
 * bulkUpload — see refBulkUploadPattern.
 *
 * @param {object} input
 * @param {string} input.table
 * @param {object[]} input.rows
 * @param {string[]} input.conflictKeys
 * @param {string} [input.mergeExpr] - SQL fragment for DO UPDATE SET; defaults to overwrite all non-conflict columns
 * @param {string} [input.connectionUrl]
 * @param {boolean} [input.dryRun]
 */
export async function bulkUpload(input = {}, userId) {
  const { table, rows, conflictKeys, mergeExpr, connectionUrl, dryRun = false } = input;
  const start = Date.now();

  if (!table || typeof table !== "string") throw new Error('"table" is required');
  quoteIdent(table);
  if (!Array.isArray(rows) || rows.length === 0) throw new Error('"rows" must be a non-empty array');
  if (!Array.isArray(conflictKeys) || conflictKeys.length === 0) throw new Error('"conflictKeys" is required (no blind inserts)');
  conflictKeys.forEach(quoteIdent);

  const cols = Object.keys(rows[0]);
  if (cols.length === 0) throw new Error("first row has no columns");
  cols.forEach(quoteIdent);
  const colSet = new Set(cols);
  for (const k of conflictKeys) {
    if (!colSet.has(k)) throw new Error(`conflictKey '${k}' missing from row columns`);
  }
  // Validate shape consistency
  for (let i = 1; i < rows.length; i++) {
    const rk = Object.keys(rows[i]);
    if (rk.length !== cols.length || rk.some((k) => !colSet.has(k))) {
      throw new Error(`row ${i} has mismatched keys vs row 0`);
    }
  }

  // Generate CSV
  const ts = Date.now();
  const tmpDir = path.resolve(process.cwd(), "tmp");
  if (!existsSync(tmpDir)) mkdirSync(tmpDir, { recursive: true });
  const csvPath = path.join(tmpDir, `bulk-${table}-${ts}.csv`);
  const headerLine = cols.map(csvEscape).join(",");
  const dataLines = rows.map((r) => cols.map((c) => csvEscape(r[c])).join(","));
  writeFileSync(csvPath, [headerLine, ...dataLines].join("\n") + "\n", "utf8");

  // Build SQL
  const stagingName = `staging_${table}_${ts}`;
  const colList = cols.join(", ");
  const nonConflictCols = cols.filter((c) => !conflictKeys.includes(c));
  const defaultMerge = nonConflictCols.length
    ? nonConflictCols.map((c) => `${c} = EXCLUDED.${c}`).join(", ")
    : null;
  const setClause = mergeExpr || defaultMerge;
  // psql \COPY requires the path on a single line; use forward slashes on Windows for safety.
  const csvPathForPsql = csvPath.replace(/\\/g, "/");

  const onConflictClause = setClause
    ? `ON CONFLICT (${conflictKeys.join(", ")}) DO UPDATE SET ${setClause}`
    : `ON CONFLICT (${conflictKeys.join(", ")}) DO NOTHING`;

  // We split COPY (meta-command, must be on its own line, no semicolon) from the SQL transaction.
  // psql will run the meta-command and then the SQL block.
  const sql = [
    `BEGIN;`,
    `CREATE TEMP TABLE ${stagingName} (LIKE ${table} INCLUDING DEFAULTS);`,
    `\\COPY ${stagingName}(${colList}) FROM '${csvPathForPsql}' WITH (FORMAT csv, HEADER true)`,
    `INSERT INTO ${table} (${colList})`,
    `SELECT ${colList} FROM ${stagingName}`,
    `${onConflictClause};`,
    `COMMIT;`,
  ].join("\n");

  if (dryRun) {
    return { ok: true, dryRun: true, table, rows_in: rows.length, csv_path: csvPath, sql };
  }

  const url = connectionUrl || buildConnectionUrl();

  // Run psql with --single-transaction (covers the file as one tx; our BEGIN/COMMIT is also OK)
  // and ON_ERROR_STOP so any failure aborts.
  // Flags must come BEFORE the dbname argument or psql treats them as filenames.
  // Do NOT pass --quiet — it suppresses the "INSERT 0 N" tag we parse for counts.
  const psqlArgs = [
    "--variable=ON_ERROR_STOP=on",
    "--no-psqlrc",
    "--echo-errors",
    "--file=-",
    "--dbname=" + url,
  ];

  const result = await new Promise((resolve) => {
    const child = spawn("psql", psqlArgs, { stdio: ["pipe", "pipe", "pipe"] });
    // UTF-8 decode at the stream level so multibyte chars (β, ×, →, …) split across
    // a pipe boundary aren't mangled into U+FFFD � (see claudecli for the rationale).
    child.stdout.setEncoding("utf8");
    child.stderr.setEncoding("utf8");
    let stdout = "";
    let stderr = "";
    child.stdout.on("data", (d) => { stdout += d.toString(); });
    child.stderr.on("data", (d) => { stderr += d.toString(); });
    child.on("error", (err) => resolve({ code: -1, stdout, stderr: stderr + `\nspawn error: ${err.message}` }));
    child.on("close", (code) => resolve({ code, stdout, stderr }));
    child.stdin.write(sql + "\n");
    child.stdin.end();
  });

  const duration_ms = Date.now() - start;

  if (result.code !== 0) {
    return {
      ok: false,
      error: result.stderr || `psql exited ${result.code}`,
      sql_used: sql,
      csv_path: csvPath,
      stdout: result.stdout,
      duration_ms,
    };
  }

  // Parse INSERT N M — psql emits e.g. "INSERT 0 3"
  let affected = 0;
  const insertMatch = (result.stdout + "\n" + result.stderr).match(/INSERT\s+\d+\s+(\d+)/);
  if (insertMatch) affected = parseInt(insertMatch[1], 10);

  return {
    ok: true,
    table,
    rows_in: rows.length,
    affected,
    csv_path: csvPath,
    duration_ms,
  };
}

// ---------------------------------------------------------------------------
// runSql / applyMigrations — DDL + migrations via the Supabase Management API.
// The service REST key (PostgREST) CANNOT run DDL; the CLI's `db push` needs a
// DB password AND can't reach the direct host on IPv4-only/quirky networks.
// The Management API SQL endpoint runs arbitrary SQL (incl. ALTER/CREATE TABLE)
// authenticated by a personal access token — no DB password, no direct host.
// See libs/skill_book/supabase/index.js → refDdlAndMigrations.
// ---------------------------------------------------------------------------

/**
 * Run arbitrary SQL against the remote Postgres via the Supabase Management API.
 * Handles DDL (ALTER/CREATE) that the service REST key cannot. Returns rows.
 * @param {object} input
 * @param {string} input.sql - one or more statements (`;`-separated allowed)
 */
export async function runSql({ sql } = {}, userId) {
  if (!sql || typeof sql !== "string") throw new Error('"sql" (string) is required');
  const ref = await getProjectRef(userId);
  const tok = getAccessToken();
  const resp = await fetch(`${MGMT_API}/projects/${ref}/database/query`, {
    method: "POST",
    headers: { Authorization: `Bearer ${tok}`, "Content-Type": "application/json" },
    body: JSON.stringify({ query: sql }),
  });
  const text = await resp.text();
  if (!resp.ok) throw new Error(`Supabase Management API query failed (${resp.status}): ${text}`);
  let rows;
  try { rows = JSON.parse(text); } catch { rows = text; }
  return { ok: true, rows };
}

/**
 * Apply local supabase/migrations/*.sql files to the remote DB via runSql, then
 * record each applied version in supabase_migrations.schema_migrations so the CLI
 * (`supabase db push` / `migration list`) sees them as applied. Agent-side
 * equivalent of `supabase db push`. Migrations MUST be idempotent (IF NOT EXISTS)
 * since this re-runs the file body, not a ledger diff.
 * @param {object} input
 * @param {string[]} [input.versions] - version prefixes (e.g. ["20260612160000"]); omit = all files
 * @param {boolean} [input.record=true] - also write the schema_migrations ledger row
 */
export async function applyMigrations({ versions, record = true } = {}, userId) {
  const dir = path.resolve(process.cwd(), "supabase", "migrations");
  if (!existsSync(dir)) throw new Error(`migrations dir not found: ${dir}`);
  const all = readdirSync(dir).filter((f) => f.endsWith(".sql")).sort();
  const want = Array.isArray(versions) && versions.length ? versions.map(String) : null;
  if (want) {
    for (const v of want) {
      if (!all.some((f) => f.startsWith(v))) throw new Error(`no migration file for version ${v} in ${dir}`);
    }
  }
  const targets = want ? all.filter((f) => want.some((v) => f.startsWith(v))) : all;
  const results = [];
  for (const file of targets) {
    const version = file.match(/^(\d+)/)?.[1];
    const name = file.replace(/^\d+_/, "").replace(/\.sql$/, "");
    const body = readFileSync(path.join(dir, file), "utf8");
    try {
      await runSql({ sql: body }, userId);
      if (record && version) {
        await runSql({
          sql:
            `insert into supabase_migrations.schema_migrations (version, name) ` +
            `values ('${version}', '${name.replace(/'/g, "''")}') on conflict (version) do nothing`,
        }, userId);
      }
      results.push({ file, version, applied: true });
    } catch (e) {
      results.push({ file, version, applied: false, error: e.message });
      break; // stop on first failure — later migrations may depend on this one
    }
  }
  return { ok: results.length > 0 && results.every((r) => r.applied), results };
}

// ---------------------------------------------------------------------------
// Skill-book-facing action wrappers.
// These wrap the bare weapon functions above in the standard skillActionOk/Err
// envelope and normalize the (userId, input) call shape used by the adventurer
// execution registry. Moved out of libs/skill_book/supabase/index.js so that
// skill book stays a PURE text-prompt registry (no imports, no functions).
// ---------------------------------------------------------------------------

export async function readTableAction(a, b) {
  const [userId, input] = typeof a === "string" ? [a, b ?? {}] : [undefined, a ?? {}];
  if (!input?.table) return skillActionErr("table required");
  try {
    const data = await readTable({ table: input.table, limit: input.limit, filter: input.filter }, userId);
    return skillActionOk(data);
  } catch (e) { return skillActionErr(e.message); }
}

export async function readLogsAction(a, b) {
  const [userId, input] = typeof a === "string" ? [a, b ?? {}] : [undefined, a ?? {}];
  try {
    const data = await readLogs({ service: input?.service, limit: input?.limit }, userId);
    return skillActionOk(data);
  } catch (e) { return skillActionErr(e.message); }
}

export async function readAPISettingsAction(a, b) {
  const [userId, input] = typeof a === "string" ? [a, b ?? {}] : [undefined, a ?? {}];
  try {
    const data = await readAPISettings({}, userId);
    return skillActionOk(data);
  } catch (e) { return skillActionErr(e.message); }
}

export async function readStorageBucketsAction(a, b) {
  const [userId, input] = typeof a === "string" ? [a, b ?? {}] : [undefined, a ?? {}];
  try {
    const data = await readStorageBuckets({}, userId);
    return skillActionOk(data);
  } catch (e) { return skillActionErr(e.message); }
}

/**
 * bulkUploadAction — enveloped bulkUpload. See libs/skill_book/supabase/index.js
 * → refBulkUploadPattern for the locked discipline.
 */
export async function bulkUploadAction(a, b) {
  const [userId, input] = typeof a === "string" ? [a, b ?? {}] : [undefined, a ?? {}];
  try {
    const data = await bulkUpload(input || {}, userId);
    return data?.ok === false ? skillActionErr(data.error || "bulkUpload failed", data) : skillActionOk(data);
  } catch (e) { return skillActionErr(e.message); }
}

export async function runSqlAction(a, b) {
  const [userId, input] = typeof a === "string" ? [a, b ?? {}] : [undefined, a ?? {}];
  if (!input?.sql) return skillActionErr("sql required");
  try { return skillActionOk(await runSql(input, userId)); }
  catch (e) { return skillActionErr(e.message); }
}

export async function applyMigrationsAction(a, b) {
  const [userId, input] = typeof a === "string" ? [a, b ?? {}] : [undefined, a ?? {}];
  try {
    const data = await applyMigrations(input || {}, userId);
    return data.ok ? skillActionOk(data) : skillActionErr("one or more migrations failed", data);
  } catch (e) { return skillActionErr(e.message); }
}
```
<!-- /GUILDOS-FILE -->
