# siteAnalytics

<!-- GUILDOS-SKILL name="siteAnalytics" -->

<!-- GUILDOS-DESC -->
Google Analytics 4 + Google Search Console via the Google REST APIs (OAuth). READ/REPORTING: pull GSC search-performance traffic (clicks, impressions, CTR, position) AND GA4 on-site traffic (sessions, users, pageviews, events, conversions) for a site + date range — the way to measure how traffic changed over time. PROVISION (set up a NEW site): GA4 property+stream, GSC add/verify/sitemap; API-first, CIC only for the one-time OAuth consent click + GA4 account-ToS gap; MCP is read-only and does NOT provision.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/siteAnalytics/SKILL.md -->
```markdown
---
name: siteAnalytics
description: >-
  Google Analytics 4 + Google Search Console via the Google REST APIs (OAuth).
  READ/REPORTING: pull GSC search-performance traffic (clicks, impressions, CTR,
  position) AND GA4 on-site traffic (sessions, users, pageviews, events,
  conversions) for a site + date range — the way to measure how traffic changed
  over time. PROVISION (set up a NEW site): GA4 property+stream, GSC
  add/verify/sitemap; API-first, CIC only for the one-time OAuth consent click +
  GA4 account-ToS gap; MCP is read-only and does NOT provision.
---

# Site Analytics Setup (GA4 + Search Console)

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/siteAnalytics/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto siteAnalytics <action>` (or read the source directly).

## Actions

- **ecosystemVerdict** — How to automate Google Analytics (GA4) and Google Search Console (GSC) setup for a new website, and why the architecture is API-first with CIC as gap-closer: community Claude skills are browser-automation/external (reference only), analytics MCP servers are read-only reporting (cannot create/provision a property or site), the Google REST APIs do the creation, CIC closes the OAuth consent + GA4 account-creation gaps.
- **injectTags** — Install the Google Analytics (GA4) gtag.js tracking tag + the Google Search Console verification meta tag + an XML sitemap into a Next.js/App-Router website repo, then deploy to Vercel. One push installs all three on the live website.
- **listProperties** — List the Search Console properties the consented OAuth user can access, with permission level (sites.list). Calls weapon/siteAnalytics → gscListSites().
- **oauthBootstrap** — One-time: mint a refresh token carrying all four scopes by driving the Google consent screen through the user's logged-in Chrome (CIC). Reuses the gmail-mcp Desktop OAuth client — no new GCP project needed.
- **provisionGA4** — Create the Google Analytics 4 (GA4) property + web data stream for the website and capture the Measurement ID (G-XXXXXXX) for the gtag. Dedup-check existing GA4 properties first. If the Google account has zero GA4 accounts, create one in the Google Analytics UI via CIC (the Admin API cannot create accounts — ToS).
- **provisionGSC** — Add + verify the website's Google Search Console (GSC) property and submit the sitemap for indexing. Default = URL-prefix property verified by an HTML META verification tag (works whenever you control the website code; no registrar DNS access needed).
- **pullGA4Traffic** — Pull GA4 on-site traffic — the Data API runReport: sessions, users, pageviews, events, conversions for a property + date range, optionally filtered (e.g. pagePath CONTAINS /services/) and broken down by date or pagePath. The on-site/behavioral counterpart to pullSearchTraffic (which is search-side). Use for engagement/conversion trends after a content push. Calls weapon/siteAnalytics → ga4RunReport({propertyId,startDate,endDate,dimensions,metrics,dimensionFilter,...}).
- **pullSearchTraffic** — Pull Google Search Console search-performance / Performance-report data: clicks, impressions, CTR, average position for a property + date range, optionally filtered to pages or queries and broken down by date or page. THE way to measure how a site's organic search traffic changed over time — trend lines, before/after, inflection-point analysis (e.g. CJGEO content impact on bosterbio.com /services/ pages). Calls weapon/siteAnalytics → gscSearchAnalytics({siteUrl,startDate,endDate,dimensions,filters,...}).
- **setupNewSite** — End-to-end runbook to set up / install / configure Google Analytics (GA4) tracking AND Google Search Console (GSC) for a new website. Ordering matters: get BOTH the GA4 Measurement ID and the Search Console verification meta tag BEFORE the single code deploy, so one push installs analytics everything; then verify Search Console ownership against the live tag and submit the sitemap.
- **verifyUserChannel** — Success-gate verification: prove it works in the USER's own Chrome (not an API echo). GA4 Realtime shows an active user after you load the site; GSC Sitemaps shows Success.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/siteAnalytics/index.js -->
```js
/**
 * siteAnalytics skill book.
 *
 * How to stand up GA4 + Google Search Console for a NEW website with minimal
 * human action. Companion weapon: `libs/weapon/siteAnalytics` (OAuth + the
 * GA4 Admin / Search Console / Site Verification / Service Usage REST calls;
 * CLI at `node libs/weapon/siteAnalytics/cli.mjs <cmd>`).
 *
 * --- ECOSYSTEM VERDICT (researched 2026-06-06) ---------------------------
 * Tiering the user's preference order (skills-by-others > MCP > API > CIC):
 *   1. SKILLS BY OTHERS — community Claude skills exist (mcpmarket "Analytics
 *      Setup GTM&GA4", "GA4 Expert", "GSC CLI"; davepoon/buildwithclaude
 *      google-analytics-automation). All are browser-automation or external
 *      CLIs, none in the GuildOS registry → mine for reference, don't adopt.
 *   2. MCP — official googleanalytics/google-analytics-mcp + community GSC
 *      MCPs are READ-ONLY/reporting (run_report, run_realtime_report,
 *      get_account_summaries). They CANNOT create a property/stream or add a
 *      GSC site. Use them only for post-setup reporting, never for setup.
 *   3. API — does ~90% of setup and is the workhorse (this weapon).
 *   4. CIC — closes exactly TWO gaps: the one-time OAuth consent "Allow"
 *      click, and GA4 top-level ACCOUNT creation (ToS; API only makes
 *      PROPERTIES under an existing account). Plus user-channel verification.
 *
 * --- LAB-RAT (protectandgrow.blog, proven 2026-06-06) --------------------
 *   Repo: github.com/boster00/bryanblog (Next.js/Vercel, branch main).
 *   GA4: account pAIpeline (331784572) → property 540458039 → stream
 *        15016999874 → Measurement ID G-3TGN5ME474.
 *   GSC: URL-prefix property https://protectandgrow.blog/, META-verified,
 *        sitemap /sitemap.xml = Success, 18 pages discovered.
 *   Auth: reused the gmail-mcp Desktop OAuth client (GCP project 962388978165);
 *        token at ~/.site-analytics/credentials.json (scopes analytics.edit,
 *        webmasters, siteverification, cloud-platform).
 * -------------------------------------------------------------------------
 */

export const skillBook = {
  id: "siteAnalytics",
  title: "Site Analytics Setup (GA4 + Search Console)",
  description:
    "Google Analytics 4 + Google Search Console via the Google REST APIs (OAuth). READ/REPORTING: pull GSC search-performance traffic (clicks, impressions, CTR, average position) AND GA4 on-site traffic (sessions, users, pageviews, events, conversions) for a site/property + date range, filter by page/query, break down by date or page — the way to measure how traffic changed over time (trend & inflection-point analysis). PROVISION (set up a NEW site): GA4 property+stream, GSC site add/verify/sitemap; API-first, CIC only for the one-time OAuth consent click + GA4 account-ToS gap; MCP is read-only and does NOT provision. Actions: pullSearchTraffic (GSC Search Analytics / Performance data), pullGA4Traffic (GA4 Data API runReport — on-site sessions/users/pageviews), listProperties, ecosystemVerdict, setupNewSite (end-to-end runbook), oauthBootstrap (one-time consent via CIC), provisionGA4 (property+stream→Measurement ID), provisionGSC (add+verify+sitemap), injectTags (gtag + GSC meta + sitemap into a Next.js/Vercel repo), verifyUserChannel (GA4 realtime + GSC console screenshots).",
  steps: [],
  toc: {
    pullSearchTraffic: {
      description:
        "Pull Google Search Console search-performance / Performance-report data: clicks, impressions, CTR, average position for a property + date range, optionally filtered to pages or queries and broken down by date or page. THE way to measure how a site's organic search traffic changed over time — trend lines, before/after, inflection-point analysis (e.g. CJGEO content impact on bosterbio.com /services/ pages). Calls weapon/siteAnalytics → gscSearchAnalytics({siteUrl,startDate,endDate,dimensions,filters,...}).",
      howTo: `
**Auth check:** \`node libs/weapon/siteAnalytics/cli.mjs check\` → ok:true with the \`webmasters\` scope. Reachable properties: \`... gsc-sites\` (consented user is siteOwner of sc-domain:bosterbio.com & https://www.bosterbio.com/).

**Run from PowerShell, NOT Git-Bash** — MSYS rewrites a leading-slash arg (\`--page /services/\` → a Windows path) before node sees it, so a path filter silently returns 0. (The in-app function call is unaffected.)
\`\`\`
node libs/weapon/siteAnalytics/cli.mjs gsc-query \\
  --site sc-domain:bosterbio.com --start 2025-12-04 --end 2026-06-02 \\
  [--page /services/] [--q "western blot"] [--op contains|equals|includingRegex] \\
  [--dimensions date|page] [--type web] [--dataState final|all] [--rowLimit 25000]
\`\`\`
In-app: \`import { gscSearchAnalytics } from "@/libs/weapon/siteAnalytics"\`.

**Recipes**
- Headline totals for a filtered set → omit \`--dimensions\` (single de-duplicated aggregate row; matches the GSC web-UI metric cards EXACTLY).
- Daily time series (inflection analysis) → \`--dimensions date\`.
- Per-URL totals → \`--dimensions page\`.

**Key facts**
- \`--dataState all\` includes the freshest (partial) days and matches the web UI; \`final\` = complete days only. GSC data lags ~2–3 days.
- A \`page contains /path/\` filter: the GSC API \`contains\` operator silently matches NOTHING when the expression has a LEADING SLASH. The weapon auto-rewrites leading-slash \`contains\` → \`includingRegex .*<expr>.*\` so the natural "URLs containing /services/" call works.
- Filtered/aggregated GSC data is flagged "may be partial" (anonymization) — trends reliable, absolute counts possibly slightly understated. VALIDATED 2026-06-06: weapon aggregate == web-UI CSV export (whole-site 85,803 clicks / 29.4M impr; /services/ 900 clicks / 549,196 impr / pos 21.9 / 31 pages).
`,
    },
    pullGA4Traffic: {
      description:
        "Pull GA4 on-site traffic — the Data API runReport: sessions, users, pageviews, events, conversions for a property + date range, optionally filtered (e.g. pagePath CONTAINS /services/) and broken down by date or pagePath. The on-site/behavioral counterpart to pullSearchTraffic (which is search-side). Use for engagement/conversion trends after a content push. Calls weapon/siteAnalytics → ga4RunReport({propertyId,startDate,endDate,dimensions,metrics,dimensionFilter,...}).",
      howTo: `
**Auth check:** \`node libs/weapon/siteAnalytics/cli.mjs check\` → ok:true WITH the \`analytics.readonly\` scope (Admin's \`analytics.edit\` is NOT enough for reporting — if missing, re-run \`oauthBootstrap\`). Find properties: \`ga4-accounts\` → \`ga4-properties --account <id>\`. (Boster Bio = account 38740509 → property **362731773**.)

**Run from PowerShell** for any \`--path\` filter (Git-Bash mangles the leading slash). CLI:
\`\`\`
node libs/weapon/siteAnalytics/cli.mjs ga4-report \\
  --property 362731773 --start 2026-05-01 --end 2026-06-02 \\
  [--dimensions date|pagePath] [--metrics sessions,totalUsers,screenPageViews] \\
  [--path /services/] [--matchType CONTAINS|EXACT|BEGINS_WITH] [--limit N]
\`\`\`
In-app: \`import { ga4RunReport } from "@/libs/weapon/siteAnalytics"\`.

**Recipes**
- Headline totals → omit \`--dimensions\` (single aggregate row).
- Daily engagement series → \`--dimensions date\`.
- Per-page → \`--dimensions pagePath\`.
- Common metrics: sessions, totalUsers, activeUsers, screenPageViews, engagementRate, averageSessionDuration, keyEvents, conversions. Common dimensions: date, pagePath, sessionDefaultChannelGroup, country, deviceCategory.

**Setup notes (one-time, already done 2026-06-08):** needs the \`analytics.readonly\` OAuth scope AND the \`analyticsdata.googleapis.com\` API enabled on the GCP project (962388978165) — \`enable-apis\` covers it. GA4 ≠ GSC: GA4 = on-site behavior (sessions/conversions), GSC = search visibility (impressions/clicks/position). For "CJGEO impact" the search story is GSC; GA4 adds the on-site/conversion angle. Validated 2026-06-08 on Boster Bio (May): 69,449 sessions site-wide; 1,292 sessions on /services/ pages.
`,
    },
    listProperties: {
      description:
        "List the Search Console properties the consented OAuth user can access, with permission level (sites.list). Calls weapon/siteAnalytics → gscListSites().",
    },
    ecosystemVerdict: {
      description:
        "How to automate Google Analytics (GA4) and Google Search Console (GSC) setup for a new website, and why the architecture is API-first with CIC as gap-closer: community Claude skills are browser-automation/external (reference only), analytics MCP servers are read-only reporting (cannot create/provision a property or site), the Google REST APIs do the creation, CIC closes the OAuth consent + GA4 account-creation gaps.",
      howTo: `See the file header "ECOSYSTEM VERDICT". Bottom line for any
"automate analytics setup" ask: do NOT reach for an MCP to CREATE things — the
GA/GSC MCPs only READ. Provisioning is the API (this weapon). The only
irreducible human-ish moments are a one-time consent click (drive via CIC) and,
if the Google account has zero GA4 accounts, a one-time account creation in the
GA4 UI (CIC). Everything else is scriptable.`,
    },

    setupNewSite: {
      description:
        "End-to-end runbook to set up / install / configure Google Analytics (GA4) tracking AND Google Search Console (GSC) for a new website. Ordering matters: get BOTH the GA4 Measurement ID and the Search Console verification meta tag BEFORE the single code deploy, so one push installs analytics everything; then verify Search Console ownership against the live tag and submit the sitemap.",
      howTo: `
## Preconditions
- Token present: \`node libs/weapon/siteAnalytics/cli.mjs check\` → ok:true.
  If not, run \`oauthBootstrap\` first.
- You control the site's code (for the gtag + GSC-meta injection). For sites you
  do NOT control the code of, use a DNS (domain) GSC property + the platform's
  own tag installer instead — out of scope for this runbook.

## Steps (per new site)
1. \`provisionGA4\` → Measurement ID (G-XXXXXXX).
2. \`provisionGSC\` step A only: get the META verification token (content value).
3. \`injectTags\`: add the GA4 gtag + the GSC \`verification.google\` meta + an
   app/sitemap.ts to the repo. Build locally, commit, push → Vercel deploys.
4. Poll the production URL until the meta token + gtag + /sitemap.xml are live.
5. \`provisionGSC\` steps B–D: verify ownership (META, now live), sites.add,
   sitemaps.submit.
6. \`verifyUserChannel\`: load the live site (fires a GA4 hit) → screenshot GA4
   Realtime (active user) and GSC Sitemaps (Success) in the USER's Chrome.

Each step is one CLI call; see the per-action howTos.`,
    },

    oauthBootstrap: {
      description:
        "One-time: mint a refresh token carrying all four scopes by driving the Google consent screen through the user's logged-in Chrome (CIC). Reuses the gmail-mcp Desktop OAuth client — no new GCP project needed.",
      howTo: `
ONE-TIME per machine (token persists + auto-refreshes afterward).

1. Start the loopback consent server (BACKGROUND):
   \`node libs/weapon/siteAnalytics/cli.mjs consent --port 4100\`
   It writes the auth URL to ~/.site-analytics/auth-url.txt and listens.

2. Read that URL and drive it through the user's Chrome via CIC:
   - Account chooser → pick the target gmail.
   - "Google hasn't verified this app" → Advanced → "Go to <app> (unsafe)".
     (Safe: the developer IS the user's own account / own GCP project.)
   - "wants additional access" → click "Select all" → Continue.

3. Google redirects to http://localhost:4100 (the loopback server, same
   machine CIC drives) → the server exchanges the code + saves the token.
   Confirm: \`cli.mjs check\` → tokenPresent:true with all 4 scopes.

GOTCHAS (learned on the lab rat):
- The gmail client registers only redirect "http://localhost". Google's
  loopback rule ignores the PORT but NOT an arbitrary PATH, so the redirect
  MUST be bare http://localhost:<port> (the weapon already does this) — a
  /oauth2callback path yields Error 400 redirect_uri_mismatch.
- After consent, enable the APIs once per GCP project:
  \`cli.mjs enable-apis\` (analyticsadmin, searchconsole, siteverification —
  authorized by the cloud-platform scope; no console clicks).`,
    },

    provisionGA4: {
      description:
        "Create the Google Analytics 4 (GA4) property + web data stream for the website and capture the Measurement ID (G-XXXXXXX) for the gtag. Dedup-check existing GA4 properties first. If the Google account has zero GA4 accounts, create one in the Google Analytics UI via CIC (the Admin API cannot create accounts — ToS).",
      howTo: `
1. \`cli.mjs ga4-accounts\` → pick the right account (content sites → the
   pipeline/content account, not a company account). If EMPTY: create an
   account once in the GA4 UI (analytics.google.com → Admin → Create account)
   via CIC, then re-run.
2. Dedup: \`cli.mjs ga4-properties --account <accountId>\` — skip create if a
   property for this site already exists.
3. \`cli.mjs ga4-create-property --account <accountId> --name "<domain>"\`
   → note name=properties/<id>.
4. \`cli.mjs ga4-create-stream --property <id> --name "<domain> Web" --uri https://<domain>\`
   → webStreamData.measurementId is the G-XXXXXXX you inject as the gtag.`,
    },

    provisionGSC: {
      description:
        "Add + verify the website's Google Search Console (GSC) property and submit the sitemap for indexing. Default = URL-prefix property verified by an HTML META verification tag (works whenever you control the website code; no registrar DNS access needed).",
      howTo: `
Use the trailing-slash site URL form: https://<domain>/

A. \`cli.mjs gsc-token --site https://<domain>/ --method META\`
   → the <meta name="google-site-verification" content="..."> value. Inject the
   content value (Next.js: metadata.verification.google) and DEPLOY before B.
B. After the tag is LIVE: \`cli.mjs gsc-verify --site https://<domain>/ --method META\`
   → owners:[<gmail>] on success.
C. \`cli.mjs gsc-add --site https://<domain>/\` (idempotent PUT).
D. \`cli.mjs gsc-sitemap --site https://<domain>/ --sitemap https://<domain>/sitemap.xml\`
   Confirm: \`cli.mjs gsc-sitemaps --site https://<domain>/\` → errors:0,
   isPending:false, contents[].submitted > 0.

Post-setup reporting (NOT setup): \`cli.mjs gsc-query\` (Search Analytics /
Performance) or the read-only google-analytics-mcp for GA4 traffic.`,
    },

    injectTags: {
      description:
        "Install the Google Analytics (GA4) gtag.js tracking tag + the Google Search Console verification meta tag + an XML sitemap into a Next.js/App-Router website repo, then deploy to Vercel. One push installs all three on the live website.",
      howTo: `
For a Next.js App Router site (the bryanblog/CJGEO blog family):
1. app/layout.tsx:
   - metadata.verification.google = "<GSC meta content>"
   - two next/script tags for gtag: the gtag/js?id=G-XXXX loader (afterInteractive)
     + an inline gtag('config','G-XXXX'). Keep any existing analytics (e.g. the
     Nexus na.js) — additive.
2. app/sitemap.ts: export a MetadataRoute.Sitemap built from the repo's canonical
   article list (e.g. lib/articles.ts) + static routes. Next serves /sitemap.xml.
3. public/robots.txt (if static): point Host + Sitemap at the PRODUCTION origin,
   not localhost. (Common stale-template bug — check it.)
4. Build locally (\`npm --prefix <repo> run build\`) to catch compile errors,
   commit the 3 files, push to the deploy branch → Vercel auto-deploys.
5. Poll the production URL until the meta token, the G-XXXX gtag, and a 200 on
   /sitemap.xml all appear before running GSC verify.`,
    },

    verifyUserChannel: {
      description:
        "Success-gate verification: prove it works in the USER's own Chrome (not an API echo). GA4 Realtime shows an active user after you load the site; GSC Sitemaps shows Success.",
      howTo: `
1. Load https://<domain>/ in a CIC tab (fires a real GA4 pageview).
2. New tab → https://analytics.google.com/analytics/web/#/p<propertyId>/realtime/overview
   → screenshot: "Active users in last 5 minutes" ≥ 1 + the page title row.
3. Navigate → https://search.google.com/search-console/sitemaps?resource_id=https%3A%2F%2F<domain>%2F
   → screenshot: /sitemap.xml Status = Success, Discovered pages > 0.
Both screenshots are the artifacts to surface. GA4 realtime can lag ~30–60s;
the gtag being present in the live HTML + the property/stream existing are the
backstop signals if realtime is slow.`,
    },
  },
};
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/siteAnalytics/cli.mjs -->
```js
#!/usr/bin/env node
/**
 * siteAnalytics CLI — bare-node dispatcher for the GA4 + GSC provisioning weapon.
 *
 * Usage:
 *   node libs/weapon/siteAnalytics/cli.mjs check
 *   node libs/weapon/siteAnalytics/cli.mjs consent [--port 4100]
 *   node libs/weapon/siteAnalytics/cli.mjs enable-apis
 *   node libs/weapon/siteAnalytics/cli.mjs ga4-accounts
 *   node libs/weapon/siteAnalytics/cli.mjs ga4-properties --account <id>
 *   node libs/weapon/siteAnalytics/cli.mjs ga4-create-property --account <id> --name "protectandgrow.blog"
 *   node libs/weapon/siteAnalytics/cli.mjs ga4-create-stream --property <id|properties/id> --name "Web" --uri https://protectandgrow.blog
 *   node libs/weapon/siteAnalytics/cli.mjs gsc-sites
 *   node libs/weapon/siteAnalytics/cli.mjs gsc-token --site https://protectandgrow.blog/ [--method META]
 *   node libs/weapon/siteAnalytics/cli.mjs gsc-add --site https://protectandgrow.blog/
 *   node libs/weapon/siteAnalytics/cli.mjs gsc-verify --site https://protectandgrow.blog/ [--method META]
 *   node libs/weapon/siteAnalytics/cli.mjs gsc-sitemap --site https://protectandgrow.blog/ --sitemap https://protectandgrow.blog/sitemap.xml
 */

import * as w from "./index.js";

function parseFlags(argv) {
  const f = {};
  for (let i = 0; i < argv.length; i++) {
    if (argv[i].startsWith("--")) {
      const k = argv[i].slice(2);
      const v = argv[i + 1] && !argv[i + 1].startsWith("--") ? argv[++i] : true;
      f[k] = v;
    }
  }
  return f;
}

const out = (x) => console.log(JSON.stringify(x, null, 2));

async function main() {
  const [cmd, ...rest] = process.argv.slice(2);
  const f = parseFlags(rest);
  switch (cmd) {
    case "check":
      out(w.checkCredentials());
      break;
    case "consent":
      out(await w.runConsent({ port: f.port ? Number(f.port) : 4100 }));
      break;
    case "token":
      out({ access_token: (await w.accessToken()).slice(0, 24) + "…(ok)" });
      break;
    case "enable-apis":
      out(await w.enableApis({}));
      break;
    case "ga4-accounts":
      out(await w.ga4ListAccounts());
      break;
    case "ga4-properties":
      out(await w.ga4ListProperties({ accountId: f.account }));
      break;
    case "ga4-create-property":
      out(await w.ga4CreateProperty({ accountId: f.account, displayName: f.name, ...(f.tz ? { timeZone: f.tz } : {}) }));
      break;
    case "ga4-create-stream":
      out(await w.ga4CreateDataStream({ propertyId: f.property, displayName: f.name || "Web stream", defaultUri: f.uri }));
      break;
    case "ga4-report": {
      const dimensions = f.dimensions ? String(f.dimensions).split(",").map((s) => s.trim()).filter(Boolean) : [];
      const metrics = f.metrics
        ? String(f.metrics).split(",").map((s) => s.trim()).filter(Boolean)
        : ["sessions", "totalUsers", "screenPageViews"];
      let dimensionFilter;
      if (f.path) dimensionFilter = { fieldName: "pagePath", matchType: f.matchType || "CONTAINS", value: f.path };
      out(await w.ga4RunReport({
        propertyId: f.property,
        startDate: f.start,
        endDate: f.end,
        dimensions,
        metrics,
        dimensionFilter,
        ...(f.limit ? { limit: Number(f.limit) } : {}),
      }));
      break;
    }
    case "gsc-sites":
      out(await w.gscListSites());
      break;
    case "gsc-token":
      out(await w.gscGetVerificationToken({ siteUrl: f.site, method: f.method || "META" }));
      break;
    case "gsc-add":
      out(await w.gscAddSite({ siteUrl: f.site }));
      break;
    case "gsc-verify":
      out(await w.gscVerify({ siteUrl: f.site, method: f.method || "META" }));
      break;
    case "gsc-sitemap":
      out(await w.gscSubmitSitemap({ siteUrl: f.site, sitemapUrl: f.sitemap }));
      break;
    case "gsc-sitemaps":
      out(await w.gscListSitemaps({ siteUrl: f.site }));
      break;
    case "gsc-query": {
      const dimensions = f.dimensions ? String(f.dimensions).split(",").map((s) => s.trim()).filter(Boolean) : [];
      const filters = [];
      if (f.page) filters.push({ dimension: "page", operator: f.op || "contains", expression: f.page });
      if (f.q) filters.push({ dimension: "query", operator: f.op || "contains", expression: f.q });
      if (f.country) filters.push({ dimension: "country", operator: "equals", expression: f.country });
      out(await w.gscSearchAnalytics({
        siteUrl: f.site,
        startDate: f.start,
        endDate: f.end,
        dimensions,
        filters,
        ...(f.rowLimit ? { rowLimit: Number(f.rowLimit) } : {}),
        ...(f.type ? { searchType: f.type } : {}),
        ...(f.dataState ? { dataState: f.dataState } : {}),
      }));
      break;
    }
    default:
      console.error("Unknown command:", cmd);
      console.error("Commands: check, consent, enable-apis, ga4-accounts, ga4-properties, ga4-create-property, ga4-create-stream, ga4-report, gsc-sites, gsc-token, gsc-add, gsc-verify, gsc-sitemap, gsc-sitemaps, gsc-query");
      console.error('  gsc-query --site sc-domain:bosterbio.com --start 2025-12-04 --end 2026-06-02 [--dimensions date|page] [--page /services/] [--q <query>] [--op contains|equals] [--type web] [--dataState final|all] [--rowLimit N]');
      console.error('  ga4-report --property 362731773 --start 2025-12-04 --end 2026-06-02 [--dimensions date|pagePath] [--metrics sessions,totalUsers,screenPageViews] [--path /services/] [--matchType CONTAINS|EXACT|BEGINS_WITH] [--limit N]');
      process.exit(1);
  }
}

main().catch((e) => {
  console.error("ERROR:", e.message);
  if (e.body) console.error(JSON.stringify(e.body, null, 2));
  process.exit(1);
});
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/siteAnalytics/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/weapon/siteAnalytics`

Rationale layer (per CLAUDE.md §6). Append; supersede in place when a decision
changes.

---

## 2026-06-06 — reuse the gmail-mcp OAuth client (GCP 962388978165) with extra scopes, not a separate GCP project

**Decision.** Site Analytics reuses the Desktop OAuth client from the gmail-mcp
install (`~/.gmail-mcp/gcp-oauth.keys.json`, GCP project 962388978165) and requests
four added scopes (analytics.edit, webmasters, siteverification, cloud-platform)
beyond Gmail's.

**Rejected alternative.** Create a dedicated GCP project + OAuth client for site
analytics.

**Why.** One GCP project means one consent flow and one credentials file. The
gmail-mcp client already exists wherever Gmail is set up; extending its scopes is
strictly cheaper than standing up a second project, and the added scopes are safe
grants on the same token (no credential sharing risk).

**Consequence if violated.** Every new user must provision a second GCP project +
OAuth credentials + key file — duplicated infra and a confusing setup. (OAuth
gotcha: the redirect must be a bare `http://localhost:<port>`.)
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/siteAnalytics/index.js -->
```js
/**
 * siteAnalytics weapon — provision GA4 + Google Search Console for a website
 * via the Google REST Admin APIs, with a loopback-OAuth consent flow.
 *
 * WHY THIS EXISTS: the official google-analytics-mcp (and community GA/GSC MCP
 * servers) are READ-ONLY (reporting). None can CREATE a GA4 property, create a
 * data stream, add a Search Console site, verify ownership, or submit a
 * sitemap. That provisioning work is API-only. This weapon is that API surface.
 *
 * AUTH: reuses the existing Desktop OAuth client from the gmail-mcp install
 * (~/.gmail-mcp/gcp-oauth.keys.json — GCP project 962388978165). One consent
 * mints a refresh token carrying the four scopes below; it is stored, gitignored,
 * at ~/.site-analytics/credentials.json and auto-refreshed thereafter.
 *
 * The CIC tier closes exactly two gaps the API cannot: (1) the one-time consent
 * "Allow" click, (2) GA4 top-level ACCOUNT creation (ToS acceptance — the Admin
 * API can only create properties UNDER an existing account).
 *
 * All functions are plain ESM with global fetch + node: builtins so the CLI
 * (cli.mjs) runs under bare `node` with no Next.js app context.
 */

import { readFileSync, writeFileSync, existsSync, mkdirSync } from "node:fs";
import { homedir } from "node:os";
import { join } from "node:path";
import http from "node:http";
import { createServer } from "node:http";

export const toc = {
  runConsent:
    "One-time loopback-OAuth consent. Starts a localhost server, builds the Google auth URL (scopes: analytics.edit, webmasters, siteverification, cloud-platform), writes it to ~/.site-analytics/auth-url.txt, waits for the redirect, exchanges the code, and saves the refresh token. Drive the printed URL through the logged-in Chrome via CIC to click Allow.",
  accessToken: "Refresh the stored refresh token into a live access token. All API calls use this.",
  checkCredentials: "Report whether the gmail OAuth client + a stored site-analytics refresh token are present.",
  enableApis:
    "Enable the required Google APIs (analyticsadmin, searchconsole, siteverification) on the GCP project via the Service Usage API. Needs the cloud-platform scope.",
  ga4ListAccounts: "List GA4 accounts the consented user can administer (accounts.list).",
  ga4ListProperties: "List GA4 properties under an account (properties.list?filter=parent:accounts/{id}).",
  ga4CreateProperty: "Create a GA4 property under an account (properties.create). Returns the property resource incl. name=properties/{id}.",
  ga4CreateDataStream:
    "Create a WEB data stream on a property (properties.dataStreams.create). Returns the stream incl. webStreamData.measurementId (G-XXXXXXX).",
  ga4RunReport:
    "Pull GA4 on-site traffic — the Data API runReport (sessions, users, pageviews, events, conversions). Params: propertyId, startDate, endDate (YYYY-MM-DD), dimensions[] (e.g. ['date'] for a time series, ['pagePath']), metrics[] (default sessions/totalUsers/screenPageViews), dimensionFilter ({fieldName,matchType,value} convenience, e.g. pagePath CONTAINS /services/, or a raw FilterExpression), limit, offset, orderBys. The read counterpart to GSC's gscSearchAnalytics for the on-site/behavioral side.",
  gscAddSite: "Add a URL-prefix site to Search Console (webmasters sites.add — PUT /sites/{siteUrl}).",
  gscListSites: "List Search Console sites the user owns (webmasters sites.list).",
  gscGetVerificationToken:
    "Get a Site Verification token for a method (siteVerification token.getToken). For META, returns the meta tag content to embed in <head>.",
  gscVerify: "Verify ownership after the token is live on the site (siteVerification webResource.insert).",
  gscSubmitSitemap: "Submit a sitemap for a verified site (webmasters sitemaps.submit — PUT /sites/{site}/sitemaps/{sitemapUrl}).",
  gscSearchAnalytics:
    "Query the Search Analytics (Performance) report — the ACTUAL traffic data (clicks, impressions, CTR, position). Params: siteUrl, startDate, endDate (YYYY-MM-DD), dimensions[] (e.g. ['date'] for a time series, ['page'] for per-URL), filters[] ({dimension,operator,expression} — e.g. page contains /services/), rowLimit, searchType, dataState. Paginates and returns rows + summed totals. For the TRUE de-duplicated aggregate (matches the GSC web-UI cards) query with dimensions=[].",
};

// ---- constants -------------------------------------------------------------

const GMAIL_KEYS = join(homedir(), ".gmail-mcp", "gcp-oauth.keys.json");
const STORE_DIR = join(homedir(), ".site-analytics");
const TOKEN_FILE = join(STORE_DIR, "credentials.json");
const AUTH_URL_FILE = join(STORE_DIR, "auth-url.txt");

const PROJECT_NUMBER = "962388978165"; // from ~/.gmail-mcp client_id prefix
const TOKEN_URL = "https://oauth2.googleapis.com/token";
const AUTH_BASE = "https://accounts.google.com/o/oauth2/v2/auth";

export const SCOPES = [
  "https://www.googleapis.com/auth/analytics.edit",
  "https://www.googleapis.com/auth/analytics.readonly", // GA4 Data API (runReport) — Admin's analytics.edit is NOT sufficient for reporting
  "https://www.googleapis.com/auth/webmasters",
  "https://www.googleapis.com/auth/siteverification",
  "https://www.googleapis.com/auth/cloud-platform",
];

const GA4_ADMIN = "https://analyticsadmin.googleapis.com/v1beta";
const GA4_DATA = "https://analyticsdata.googleapis.com/v1beta";
const GSC_API = "https://www.googleapis.com/webmasters/v3";
const SITEVERIFY_API = "https://www.googleapis.com/siteVerification/v1";
const SERVICEUSAGE = "https://serviceusage.googleapis.com/v1";

// ---- credential plumbing ---------------------------------------------------

function loadClient() {
  if (!existsSync(GMAIL_KEYS)) {
    throw new Error(`OAuth client not found at ${GMAIL_KEYS}. The gmail-mcp install provides it.`);
  }
  const raw = JSON.parse(readFileSync(GMAIL_KEYS, "utf8"));
  const c = raw.installed || raw.web;
  if (!c?.client_id || !c?.client_secret) throw new Error("gcp-oauth.keys.json missing client_id/client_secret.");
  return { clientId: c.client_id, clientSecret: c.client_secret };
}

function ensureStore() {
  if (!existsSync(STORE_DIR)) mkdirSync(STORE_DIR, { recursive: true });
}

export function loadToken() {
  if (!existsSync(TOKEN_FILE)) return null;
  return JSON.parse(readFileSync(TOKEN_FILE, "utf8"));
}

function saveToken(tok) {
  ensureStore();
  writeFileSync(TOKEN_FILE, JSON.stringify(tok, null, 2), { mode: 0o600 });
}

export function checkCredentials() {
  const clientOk = existsSync(GMAIL_KEYS);
  const tok = loadToken();
  return {
    ok: clientOk && !!tok?.refresh_token,
    clientPresent: clientOk,
    tokenPresent: !!tok?.refresh_token,
    scopes: tok?.scope || null,
    projectNumber: PROJECT_NUMBER,
  };
}

/** Refresh the stored refresh token into a live access token. */
export async function accessToken() {
  const tok = loadToken();
  if (!tok?.refresh_token) throw new Error("No refresh token. Run runConsent first.");
  const { clientId, clientSecret } = loadClient();
  const body = new URLSearchParams({
    client_id: clientId,
    client_secret: clientSecret,
    refresh_token: tok.refresh_token,
    grant_type: "refresh_token",
  });
  const res = await fetch(TOKEN_URL, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: body.toString(),
  });
  if (!res.ok) throw new Error(`Token refresh failed (${res.status}): ${await res.text()}`);
  const data = await res.json();
  return data.access_token;
}

// ---- generic Google REST helper -------------------------------------------

async function gapi(method, url, { body, token, query } = {}) {
  const at = token || (await accessToken());
  let full = url;
  if (query) {
    const qs = new URLSearchParams(query).toString();
    full += (url.includes("?") ? "&" : "?") + qs;
  }
  const res = await fetch(full, {
    method,
    headers: {
      Authorization: `Bearer ${at}`,
      ...(body ? { "Content-Type": "application/json" } : {}),
    },
    ...(body ? { body: JSON.stringify(body) } : {}),
  });
  const text = await res.text();
  let json = null;
  try { json = text ? JSON.parse(text) : {}; } catch { json = { raw: text }; }
  if (!res.ok) {
    const msg = json?.error?.message || text || res.statusText;
    const err = new Error(`${method} ${url} → ${res.status}: ${msg}`);
    err.status = res.status;
    err.body = json;
    throw err;
  }
  return json;
}

// ---- OAuth consent (loopback) ---------------------------------------------

/**
 * Start a loopback server, build the consent URL, write it to AUTH_URL_FILE,
 * and resolve once the redirect delivers a code (which we exchange + store).
 * Run this BACKGROUNDED; meanwhile drive AUTH_URL_FILE's URL through CIC.
 */
export async function runConsent({ port = 4100, scopes = SCOPES } = {}) {
  ensureStore();
  const { clientId, clientSecret } = loadClient();
  // NOTE: the gmail Desktop OAuth client registers only "http://localhost".
  // Google's loopback rule ignores the PORT but not an arbitrary PATH, so the
  // redirect must be bare http://localhost:<port> (root) — not /oauth2callback.
  const redirectUri = `http://localhost:${port}`;
  const authUrl =
    `${AUTH_BASE}?` +
    new URLSearchParams({
      client_id: clientId,
      redirect_uri: redirectUri,
      response_type: "code",
      scope: scopes.join(" "),
      access_type: "offline",
      prompt: "consent",
      include_granted_scopes: "true",
    }).toString();
  writeFileSync(AUTH_URL_FILE, authUrl, "utf8");
  console.log("AUTH_URL:", authUrl);
  console.log("Listening on", redirectUri);

  return await new Promise((resolve, reject) => {
    const server = createServer(async (req, res) => {
      try {
        const u = new URL(req.url, redirectUri);
        const code = u.searchParams.get("code");
        const oerr = u.searchParams.get("error");
        if (!code && !oerr) {
          // favicon or stray request — ignore, keep listening
          res.writeHead(204).end();
          return;
        }
        if (oerr) {
          res.writeHead(400, { "Content-Type": "text/html" }).end(`<h2>OAuth error: ${oerr}</h2>`);
          server.close();
          reject(new Error(`OAuth error: ${oerr}`));
          return;
        }
        if (!code) { res.writeHead(400).end("missing code"); return; }
        // exchange
        const tokenRes = await fetch(TOKEN_URL, {
          method: "POST",
          headers: { "Content-Type": "application/x-www-form-urlencoded" },
          body: new URLSearchParams({
            code, client_id: clientId, client_secret: clientSecret,
            redirect_uri: redirectUri, grant_type: "authorization_code",
          }).toString(),
        });
        const tok = await tokenRes.json();
        if (!tokenRes.ok || !tok.refresh_token) {
          res.writeHead(500, { "Content-Type": "text/html" })
            .end(`<h2>Token exchange failed</h2><pre>${JSON.stringify(tok)}</pre>`);
          server.close();
          reject(new Error(`Token exchange failed: ${JSON.stringify(tok)}`));
          return;
        }
        saveToken(tok);
        res.writeHead(200, { "Content-Type": "text/html" })
          .end("<h2>✅ siteAnalytics authorized. You can close this tab.</h2>");
        server.close();
        resolve({ ok: true, scope: tok.scope, stored: TOKEN_FILE });
      } catch (e) {
        try { res.writeHead(500).end(String(e)); } catch {}
        server.close();
        reject(e);
      }
    });
    server.on("error", reject);
    server.listen(port);
  });
}

// ---- Service Usage (enable APIs) ------------------------------------------

export async function enableApis({ apis, projectNumber = PROJECT_NUMBER } = {}) {
  const services = apis || [
    "analyticsadmin.googleapis.com",
    "analyticsdata.googleapis.com", // GA4 Data API (runReport / reporting)
    "searchconsole.googleapis.com",
    "siteverification.googleapis.com",
  ];
  const out = [];
  for (const svc of services) {
    const r = await gapi(
      "POST",
      `${SERVICEUSAGE}/projects/${projectNumber}/services/${svc}:enable`,
      { body: {} },
    );
    out.push({ service: svc, result: r });
  }
  return out;
}

// ---- GA4 Admin -------------------------------------------------------------

export async function ga4ListAccounts() {
  const r = await gapi("GET", `${GA4_ADMIN}/accounts`);
  return r.accounts || [];
}

export async function ga4ListProperties({ accountId }) {
  const r = await gapi("GET", `${GA4_ADMIN}/properties`, {
    query: { filter: `parent:accounts/${accountId}`, pageSize: "200" },
  });
  return r.properties || [];
}

export async function ga4CreateProperty({
  accountId, displayName, timeZone = "America/New_York", currencyCode = "USD",
}) {
  return await gapi("POST", `${GA4_ADMIN}/properties`, {
    body: {
      parent: `accounts/${accountId}`,
      displayName,
      timeZone,
      currencyCode,
    },
  });
}

export async function ga4CreateDataStream({ propertyId, displayName, defaultUri }) {
  // propertyId may be "properties/123" or "123"
  const pid = String(propertyId).startsWith("properties/") ? propertyId : `properties/${propertyId}`;
  return await gapi("POST", `${GA4_ADMIN}/${pid}/dataStreams`, {
    body: {
      type: "WEB_DATA_STREAM",
      displayName,
      webStreamData: { defaultUri },
    },
  });
}

// ---- GA4 Data API (reporting) ---------------------------------------------

/** Accept a convenience {fieldName,matchType,value} or a raw GA4 FilterExpression. */
function normalizeGa4Filter(f) {
  if (!f || typeof f !== "object") return undefined;
  if (f.filter || f.andGroup || f.orGroup || f.notExpression) return f; // already a FilterExpression
  if (f.fieldName) {
    return {
      filter: {
        fieldName: f.fieldName,
        stringFilter: { matchType: f.matchType || "CONTAINS", value: f.value, caseSensitive: !!f.caseSensitive },
      },
    };
  }
  return f;
}

/**
 * Pull GA4 on-site traffic via the Data API runReport — the read counterpart to
 * gscSearchAnalytics (GSC is search-side; this is on-site/behavioral).
 *
 * @param {{
 *   propertyId: string,                 // 'properties/362731773' or '362731773'
 *   startDate: string, endDate: string, // 'YYYY-MM-DD' (also 'NdaysAgo'/'today' accepted by GA4)
 *   dimensions?: Array<string|{name:string}>,   // e.g. ['date'], ['pagePath']
 *   metrics?: Array<string|{name:string}>,       // default sessions/totalUsers/screenPageViews
 *   dimensionFilter?: { fieldName?: string, matchType?: string, value?: string, caseSensitive?: boolean } | object,
 *   limit?: number, offset?: number, orderBys?: object[], keepEmptyRows?: boolean,
 * }} p
 * @returns {Promise<{ rows: Array<Record<string, string|number>>, totals: Record<string, number>, rowCount: number, dimensionHeaders: string[], metricHeaders: string[] }>}
 */
export async function ga4RunReport({
  propertyId, startDate, endDate,
  dimensions = [], metrics = ["sessions", "totalUsers", "screenPageViews"],
  dimensionFilter, limit = 100000, offset = 0, orderBys, keepEmptyRows = false,
} = {}) {
  if (!propertyId) throw new Error("propertyId is required (e.g. 'properties/362731773').");
  if (!startDate || !endDate) throw new Error("startDate and endDate (YYYY-MM-DD) are required.");
  const pid = String(propertyId).startsWith("properties/") ? propertyId : `properties/${propertyId}`;

  const body = {
    dateRanges: [{ startDate, endDate }],
    dimensions: dimensions.map((d) => (typeof d === "string" ? { name: d } : d)),
    metrics: metrics.map((m) => (typeof m === "string" ? { name: m } : m)),
    limit: String(Math.max(1, Math.min(Number(limit) || 100000, 250000))),
    offset: String(Math.max(0, Number(offset) || 0)),
    keepEmptyRows: !!keepEmptyRows,
  };
  if (orderBys) body.orderBys = orderBys;
  const filt = normalizeGa4Filter(dimensionFilter);
  if (filt) body.dimensionFilter = filt;

  const r = await gapi("POST", `${GA4_DATA}/${pid}:runReport`, { body });

  const dimNames = (r.dimensionHeaders || []).map((h) => h.name);
  const metNames = (r.metricHeaders || []).map((h) => h.name);
  const rows = (r.rows || []).map((row) => {
    const o = {};
    (row.dimensionValues || []).forEach((v, i) => { o[dimNames[i] || `dim${i}`] = v.value; });
    (row.metricValues || []).forEach((v, i) => { o[metNames[i] || `met${i}`] = Number(v.value); });
    return o;
  });
  const totals = {};
  if (r.totals && r.totals[0]) {
    (r.totals[0].metricValues || []).forEach((v, i) => { totals[metNames[i] || `met${i}`] = Number(v.value); });
  }
  return { rows, totals, rowCount: r.rowCount ?? rows.length, dimensionHeaders: dimNames, metricHeaders: metNames };
}

// ---- Search Console + Site Verification -----------------------------------

const enc = (s) => encodeURIComponent(s);

export async function gscListSites() {
  const r = await gapi("GET", `${GSC_API}/sites`);
  return r.siteEntry || [];
}

export async function gscAddSite({ siteUrl }) {
  await gapi("PUT", `${GSC_API}/sites/${enc(siteUrl)}`);
  return { ok: true, siteUrl };
}

export async function gscGetVerificationToken({ siteUrl, method = "META" }) {
  // For META, identifier should be the site origin (no trailing slash needed).
  const r = await gapi("POST", `${SITEVERIFY_API}/token`, {
    body: {
      verificationMethod: method,
      site: { type: "SITE", identifier: siteUrl },
    },
  });
  return r; // { token: "<meta ... content=...> or content value", method }
}

export async function gscVerify({ siteUrl, method = "META" }) {
  return await gapi("POST", `${SITEVERIFY_API}/webResource`, {
    query: { verificationMethod: method },
    body: { site: { type: "SITE", identifier: siteUrl } },
  });
}

export async function gscSubmitSitemap({ siteUrl, sitemapUrl }) {
  await gapi("PUT", `${GSC_API}/sites/${enc(siteUrl)}/sitemaps/${enc(sitemapUrl)}`);
  return { ok: true, siteUrl, sitemapUrl };
}

export async function gscListSitemaps({ siteUrl }) {
  const r = await gapi("GET", `${GSC_API}/sites/${enc(siteUrl)}/sitemaps`);
  return r.sitemap || [];
}

/**
 * Query the Search Analytics (Performance) report — the actual traffic data.
 * This is the read counterpart to the provisioning calls above. The consented
 * OAuth user must have access to the property (any permissionLevel).
 *
 * @param {{
 *   siteUrl: string,                // 'sc-domain:bosterbio.com' or 'https://www.bosterbio.com/'
 *   startDate: string,              // 'YYYY-MM-DD'
 *   endDate: string,                // 'YYYY-MM-DD'
 *   dimensions?: string[],          // [] = single aggregate row; ['date'] = daily series; ['page'] = per-URL
 *   filters?: Array<{ dimension: string, operator?: string, expression: string }>,
 *   rowLimit?: number,              // per page, max 25000 (paginates beyond)
 *   searchType?: string,            // 'web' | 'image' | 'video' | 'news' | 'discover' | 'googleNews'
 *   dataState?: string,             // 'final' | 'all' (all includes fresh, partial data)
 *   aggregationType?: string,       // 'auto' | 'byPage' | 'byProperty'
 * }} p
 * @returns {Promise<{ rows: Array<{keys?:string[],clicks:number,impressions:number,ctr:number,position:number}>, totals:{clicks:number,impressions:number}, rowCount:number }>}
 */
export async function gscSearchAnalytics({
  siteUrl, startDate, endDate,
  dimensions = [], filters = [],
  rowLimit = 25000, searchType = "web", dataState = "final", aggregationType,
} = {}) {
  if (!siteUrl) throw new Error("siteUrl is required (e.g. 'sc-domain:bosterbio.com').");
  if (!startDate || !endDate) throw new Error("startDate and endDate (YYYY-MM-DD) are required.");

  const pageLimit = Math.min(Math.max(Number(rowLimit) || 25000, 1), 25000);
  const base = { startDate, endDate, type: searchType, dataState, rowLimit: pageLimit };
  if (dimensions.length) base.dimensions = dimensions;
  if (aggregationType) base.aggregationType = aggregationType;
  if (filters.length) {
    base.dimensionFilterGroups = [{
      groupType: "and",
      filters: filters.map((f) => {
        let operator = f.operator || "contains";
        let expression = f.expression;
        // GSC API quirk: the `contains` operator with a LEADING SLASH silently
        // matches zero rows (even though the page URLs contain it). Replicate the
        // web-UI "URLs containing /path/" by switching to an anchored regex.
        if (operator === "contains" && typeof expression === "string" && expression.startsWith("/")) {
          const escaped = expression.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
          operator = "includingRegex";
          expression = `.*${escaped}.*`;
        }
        return { dimension: f.dimension, operator, expression };
      }),
    }];
  }

  // Page through results via startRow until a short page returns.
  const rows = [];
  let startRow = 0;
  for (;;) {
    const page = await gapi("POST", `${GSC_API}/sites/${enc(siteUrl)}/searchAnalytics/query`, {
      body: { ...base, startRow },
    });
    const got = page.rows || [];
    rows.push(...got);
    if (got.length < pageLimit || !dimensions.length) break; // aggregate query = single row
    startRow += got.length;
    if (startRow >= 100000) break; // safety cap
  }

  const totals = rows.reduce(
    (a, r) => ({ clicks: a.clicks + (r.clicks || 0), impressions: a.impressions + (r.impressions || 0) }),
    { clicks: 0, impressions: 0 },
  );
  return { rows, totals, rowCount: rows.length };
}
```
<!-- /GUILDOS-FILE -->
