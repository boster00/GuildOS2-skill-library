# nexus

<!-- GUILDOS-SKILL name="nexus" -->

<!-- GUILDOS-DESC -->
Execute Boster Nexus / Nexus Armor development tasks: the locked module/action routing shape, adding module actions, migrating legacy /api routes, Zoho Books/CRM/Desk data sync (incremental vs manual-only total_sync), provider-agnostic transactional email, the cart/to-quote quote-request endpoint, the legacy Chrome-extension dispatcher, and RunPod container deploys.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/nexus/SKILL.md -->
```markdown
---
name: nexus
description: >-
  Execute Boster Nexus / Nexus Armor development tasks: the locked
  module/action routing shape, adding module actions, migrating legacy
  /api routes, Zoho Books/CRM/Desk data sync (incremental vs manual-only
  total_sync), provider-agnostic transactional email, the cart/to-quote
  quote-request endpoint, the legacy Chrome-extension dispatcher, and
  RunPod container deploys.
---

# Boster Nexus — Nexus Armor & Zoho Integration

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/nexus/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto nexus <action>` (or read the source directly).

## Actions

- **addModuleAction** — Add a new action to a module — the standard recipe agents must follow when building new features in boster_nexus.
- **cartToQuoteApi** — POST /api/cart/to-quote — the Boster Nexus quote-request endpoint. Takes a cart payload (customer + items + addresses) and creates a Zoho CRM Quote, finding-or-creating Contact and Products as it goes. Email is sent automatically via EmailOnCreate=true.
- **deployRunPod** — Build and push RunPod container image for the structure worker.
- **migrateLegacyApiRoute** — Recipe for migrating an existing /api/<x> route into the /<module>/<action>/api shape.
- **nexusArmorLegacyDispatcher** — The /api/nexus-armor route is a Chrome-extension companion API — a single POST dispatcher that dynamic-imports actions/<action>.js. NOT the same as the per-action module layout in readModuleArchitectureRule; this is legacy and lives under /api/nexus-armor only because it predates the rule.
- **readModuleArchitectureRule** — Read the locked module/action layout used by Boster Nexus (Nexus Armor + every other module).
- **sendEmail** — Send transactional email from boster_nexus via the provider-agnostic mail dispatcher (watchdog alerts, order notifications, credit emails).
- **syncNewZohoData** — Sync new Zoho Books/CRM/Desk records since the last sync (follows layered architecture).
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/nexus/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/skill_book/nexus`

Rationale layer (per CLAUDE.md §6). Append; supersede in place when a decision
changes.

---

## 2026-05-21 — Boster Nexus features use a strict `<module>/<action>/api/route.js` layout (the `/api` segment is required)

**Decision.** Every new Nexus feature conforms to:
`app/(private)/<module>/<action>/page.js` (UI) + `…/<action>/api/route.js` (POST
handler) + optional `…/<sub-action>/api/route.js`. The `/api` segment is mandatory
(not `/run`, `/exec`, `/handler`). Pre-existing top-level `/api/...` paths (auth
callbacks, webhooks, cron, Stripe redirects, migrations) do NOT migrate.

**Rejected alternative.** Let each feature pick its own shape
(`/api/<module>/<action>`, `/run`, `/exec`, …).

**Why.** A fixed convention keeps AI-driven feature generation deterministic.
Without the explicit `/api` segment, agents drift toward inventing alternative
endpoint names and the URL contract fragments.

**Consequence if violated.** New features proliferate in inconsistent
naming/layout; mixed URL contracts fragment the codebase and break the
generation pattern.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/nexus/index.js -->
```js
/**
 * Nexus skill book — knowledge registry for Boster Nexus / Nexus Armor.
 */

export const skillBook = {
  id: "nexus",
  title: "Boster Nexus — Nexus Armor & Zoho Integration",
  description: "Execute Nexus development tasks: Zoho data sync and RunPod deployment.",
  steps: [],
  toc: {
    readModuleArchitectureRule: {
      description: "Read the locked module/action layout used by Boster Nexus (Nexus Armor + every other module).",
      howTo: `
**Locked rule (boster_nexus repo). Routing in this repo follows a strict module/action shape; every new feature MUST conform.**

**Definitions:**
- A *module* is a top-level domain. The first one to land was \`nexus-armor\`. Other modules use the same shape: \`lifesci\`, \`pep\`, \`antigen-designer\`, \`in-silico\`, \`bic\`, \`webpages\`, \`bundle-save\`, \`dropship\`, \`data-synchronizer\`, \`webina\`.
- An *action* is one feature within a module — e.g. \`stuck-scan\`, \`items-journey\`, \`ignore\`, \`ship-on-friday\`. One folder per action.

**File layout:**
\`\`\`
app/(private)/<module>/<action>/
  page.js                 ← UI (the "manual action UI" — what the user opens)
  _<helper>.js            ← optional client helpers, modal components
  api/route.js            ← POST handler (the "API call")
  <sub-action>/api/route.js   ← optional sub-action API handlers
\`\`\`

**URL contract:**
- UI: \`GET /<module>/<action>\`
- API: \`POST /<module>/<action>/api\`
- Sub-action API: \`POST /<module>/<action>/<sub-action>/api\`

**Why the literal \`/api\` segment is required (do not drop it):** without an explicit segment, AI agents drift toward inventing alternative shapes (\`/run\`, \`/exec\`, \`/handler\`). The segment is a coding convention to keep generation deterministic, not a Next.js requirement.

**What stays at the legacy \`/api/...\` path (do NOT migrate):**
- \`/api/auth/**\`, \`/api/zoho/auth/**\` — OAuth callbacks (third-party services hit these URLs)
- \`/api/webhook/**\`, \`/api/data-synchronizer/webhook/**\`, \`/api/in-silico/**/webhook\`, \`/api/in-silico/**/job/[jobId]\` — external services post here
- \`/api/cron/**\`, \`/api/migrations/**\`, \`/api/stripe/{create-checkout,create-portal}\` — Stripe redirects, Vercel cron config

Anything else module-owned should live under \`(private)/<module>/<action>/\`.

**Existing reference implementation (read this code before adding a new action):**
\`app/(private)/nexus-armor/stuck-scan/\` —
- \`page.js\` (UI calls fetches \`/nexus-armor/stuck-scan/api\`, \`/nexus-armor/stuck-scan/ignore/api\`, \`/nexus-armor/stuck-scan/ship-on-friday/api\`, and \`/nexus-armor/stuck-scan/manual-sync/api\`)
- \`api/route.js\` (the GET that powers the dashboard)
- \`manual-sync/api/route.js\` (the POST that runs Total Sync)
- \`ignore/api/route.js\`, \`ship-on-friday/api/route.js\` (sub-action POSTs)

**Auth model:**
- \`(private)/layout.js\` redirects pages to login when no user — but layouts do NOT apply to \`route.js\` files, so API handlers under \`(private)/<module>/<action>/api\` are NOT auto-gated. If an action's API needs auth, gate it explicitly inside the route handler. Same posture as the legacy \`/api/...\` routes had.
- \`proxy.js\` runs Supabase token refresh on every path; doesn't block.
`,
    },
    addModuleAction: {
      description: "Add a new action to a module — the standard recipe agents must follow when building new features in boster_nexus.",
      howTo: `
**When to use:** A new manual-trigger feature is being added to an existing module (or a brand-new module is being introduced).

**Steps:**

1. **Pick the module.** If brand-new, name it lowercase-kebab-case (e.g. \`nexus-armor\`, not \`NexusArmor\`).

2. **Create the action folder:**
\`\`\`
app/(private)/<module>/<action>/
\`\`\`

3. **Add the UI page.** \`app/(private)/<module>/<action>/page.js\` — client component (\`"use client"\`). The page is the user-facing manual-action UI.

4. **Add the API handler.** \`app/(private)/<module>/<action>/api/route.js\` — exports \`POST(request)\` that does the work. Return \`NextResponse.json(...)\`.

5. **Wire UI → API.** In \`page.js\`, fetch \`POST /<module>/<action>/api\`. Do NOT use \`/api/<module>/<action>\` — that's the OLD legacy shape, banned for new work.

6. **Sub-actions.** If the action has multiple POST endpoints (e.g. stuck-scan has \`ignore\`, \`ship-on-friday\`, \`manual-sync\`), each lives at \`<action>/<sub-action>/api/route.js\` and the UI fetches \`/<module>/<action>/<sub-action>/api\`.

7. **Modules registry.** If the module is meant to appear in the home dashboard, add an entry in \`libs/modules.js\` with \`path: '/<module>/<action>'\`.

8. **Smoke test.** Load the UI URL in CIC, exercise the action, confirm the network panel shows the POST hitting \`/<module>/<action>/api\` (not \`/api/...\`).

**Anti-patterns to reject:**
- Dropping \`/api\` segment (e.g. \`/<module>/<action>/run\`) — banned.
- Putting route.js back under \`app/api/<module>/...\` — banned (use the new shape).
- Splitting UI and API into different module names — they MUST share the \`<module>/<action>\` prefix.
- Long-running synchronous POSTs that exceed Vercel's 60s function timeout. For heavy work (like Total Sync), use a job-queue pattern: kick off + return immediately, UI polls for completion.
`,
    },
    migrateLegacyApiRoute: {
      description: "Recipe for migrating an existing /api/<x> route into the /<module>/<action>/api shape.",
      howTo: `
**When to use:** An existing route at \`app/api/<x>/route.js\` is being moved to the module/action shape per the architecture rule.

**Steps (one commit per migrated route, ideally one commit per module):**

1. **Move the API file.** \`git mv app/api/<x>/route.js app/(private)/<module>/<action>/api/route.js\`. Keeping it under \`(private)\` mirrors the page next to it.

2. **Move the UI page** if applicable. \`git mv app/(private)/<old-path>/page.js app/(private)/<module>/<action>/page.js\` (and any \`_helpers.js\` siblings).

3. **Find every caller.** \`grep -rn "/api/<x>"\` across the repo (skip \`node_modules\`, \`.next\`, \`docs\`). Categories:
    - \`fetch("/api/<x>")\` in client code (page.js, modal helpers) → rewrite to \`fetch("/<module>/<action>/api")\`.
    - \`fetch(\\\`\${origin}/api/<x>\\\`)\` in cron/server code → rewrite similarly.
    - Module registry entries (\`libs/modules.js\`) → update \`path\`.
    - Index pages that link to the UI (\`app/(private)/<old-section>/page.js\` tiles) → update \`href\`.
    - Smoke-test scripts under \`scripts/_*-smoke.mjs\` → update URLs.
    - Doc comments inside the moved route.js itself — fix the JSDoc header path comment but don't gold-plate every internal comment unless it'd be misleading.

4. **Verify.** Restart dev server (\`npm run dev\`, port 3001). \`curl -o /dev/null -w "%{http_code}\\n" http://localhost:3001/<module>/<action>/api?...\` should return 200 (or whatever the route's success code is). Then load the UI in CIC and exercise the action — confirm Network shows the new path.

5. **Commit message.** \`<module>/<action>: migrate route to <module>/<action>/api shape\`. List the moved files, the renamed callers, and confirmation of smoke test.

**Out-of-scope (do not migrate):** see \`readModuleArchitectureRule\` for the list (auth callbacks, webhooks, cron, Stripe redirects, migrations).
`,
    },
    syncNewZohoData: {
      description: "Sync new Zoho Books/CRM/Desk records since the last sync (follows layered architecture).",
      howTo: `
**Non-negotiable:** All Zoho HTTP must use the \`Zoho\` class singleton from \`libs/zoho/index.js\`. Never call fetch/axios directly to Zoho APIs.

**Sync modes — \`total_sync\` is manual-only (locked design invariant; do NOT auto-schedule it).** The Zoho→Supabase sync has two distinct modes and they are NOT interchangeable:
- **Incremental sync** = the recurring freshness path. Keeping data current is the job of the recurring incremental sync workflow (the Watchdog-owned per-module incremental pull), NOT of \`total_sync\`. If a user asks to "keep it fresh / keep it current" on a schedule, that means the incremental sync workflow.
- **\`total_sync\`** = a **manual-only, one-time recovery / backfill** mode (heavy, configurable lookback, default ~180 days; also backfills cached headers with missing line items). It is **never auto-scheduled** by the daily scheduler or the five-minute worker — it is triggered explicitly by a human/agent when something needs repair. **Scheduling \`total_sync\` to run on a recurring cron (e.g. "schedule total_sync daily") contradicts this locked rule** — surface the conflict and steer to the incremental sync workflow; do not silently comply. Full design source-of-truth: \`docs/handoffs/watchdog-data-sync-architecture.md\` (§17 + §13).

**5-layer architecture:**

1. **API routes** (\`app/api/**\`) — Auth, validation, map body to DTOs, call application services, return JSON. Do NOT embed Zoho payload shaping or multi-step flows here.

2. **Application services** (\`libs/zoho/services/**\`) — Own use cases (e.g., resolve shipping_address_id). May use ZohoRepository, domain VOs, and the zoho client.

3. **Repository** (\`libs/zoho/repository/ZohoRepository.js\`) — DB cache read/write. Upsert after successful Zoho creates. No business rules.

4. **Domain** (\`libs/zoho/domain/**\`) — Value objects, equality, matchers. NO I/O.

5. **Entities** (\`libs/zoho/entities/**\`) — Column/API mapping. Each entity has \`extractFromResponse\` and \`transformToDbRecord\`.

**Example flow:**
checkout → BooksContactAddressService.resolveForSalesOrder → BooksContactAddress + BooksContactAddressMatcher + zoho.getBooksContact / createBooksContactAddress + ZohoRepository.upsertContactAddressFromApiResponse

**Tech debt:** BooksContactEntity / zoho_books_contacts cache table may be missing. Wire ZohoRepository.getById(BooksContactEntity, ...) when contacts cache is restored.
`,
    },
    sendEmail: {
      description: "Send transactional email from boster_nexus via the provider-agnostic mail dispatcher (watchdog alerts, order notifications, credit emails).",
      howTo: `
**Architecture rule (locked):**
- Domain libs (watchdog, order-notify, credit-issuance, etc.) BUILD and RETURN email payloads — they never send.
- Only API route handlers and cron routes TRIGGER sends by importing \`@/libs/mail\`.

**For API routes and cron handlers (the correct triggering layer):**
\`\`\`javascript
import { sendEmail } from '@/libs/mail';
await sendEmail({ to, subject, text, html, from, replyTo });
\`\`\`
\`@/libs/mail\` reads \`MAIL_PROVIDER\` env var (default: \`"zoho"\`) and dispatches to the right implementation. Supports \`"zoho"\` | \`"resend"\`.

**For agent/user-context sends (GuildOS weapon):** use \`zoho.sendMail({ to, subject, text?, html?, from?, replyTo? }, userId)\` — resolves OAuth token from the authenticated user's potions row. Requires ZohoMail.messages.ALL scope.

**Env vars for Zoho (server-side transport):**

| Env var | Purpose |
|---------|---------|
| \`ZOHO_MAIL_CLIENT_ID\` | OAuth app client ID |
| \`ZOHO_MAIL_CLIENT_SECRET\` | OAuth app client secret |
| \`ZOHO_MAIL_REFRESH_TOKEN\` | Long-lived offline refresh token |
| \`ZOHO_MAIL_ACCOUNT_ID\` | Zoho Mail account ID (run GET /api/accounts once to retrieve) |
| \`ZOHO_MAIL_FROM_ADDRESS\` | Default From address, e.g. \`"Boster Nexus <<REDACTED-EMAIL>>"\` |
| \`ZOHO_MAIL_REGION\` | Optional — "com" (default), "eu", "in", "com_au", "jp" |

**Important:** never import from \`@/libs/resend\` or \`@/libs/zoho-mail\` in new code — both are deprecated or internal. All net-new sends go through \`@/libs/mail\`.

**Domain lib pattern (how to refactor a lib that used to self-send):**
- Rename \`sendX()\` → \`buildXPayload()\` — returns \`{ to, subject, text, html }\` or \`null\`.
- Remove the email send from the lib entirely.
- In the calling API route: \`const payload = buildXPayload(...); if (payload) await sendEmail(payload);\`.

**Zoho Mail REST shape (what \`libs/zoho-mail.js\` sends):** POST \`https://mail.zoho.com/api/accounts/{accountId}/messages\` with JSON body \`{ fromAddress, toAddress, subject, content, mailFormat: "html"|"plaintext" }\`. Token in \`Authorization: Zoho-oauthtoken {token}\` header.

**Nexus Armor architecture note:** the "stuck scan" alert is a Nexus Armor feature (hourly injection into the sales order UI; last-inject tracked in a browser cookie), NOT a watchdog feature. Its alert email is sent from \`app/api/cron/stuck-scan-monitor/route.js\` via \`@/libs/mail\`.
`,
    },
    cartToQuoteApi: {
      description: "POST /api/cart/to-quote — the Boster Nexus quote-request endpoint. Takes a cart payload (customer + items + addresses) and creates a Zoho CRM Quote, finding-or-creating Contact and Products as it goes. Email is sent automatically via EmailOnCreate=true.",
      howTo: `
**Status:** production, gated by \`x-nexus-webhook-token: $CART_WEBHOOK_TOKEN\` (32-char env secret in \`.env.local\`).
**Origin:** TS/JS port of bioinvsync \`productionAPI.php\` action=cartToQuote (lines ~2430). Full PHP reference now in repo at \`docs/bioinvsync_reference/zohobooksAPI/\` (161 files, 6.4 MB — fetched May 18 build via SSH; partial older subset at \`temp-reference-only/Bioinvsync/\` left untouched).

**Endpoint:** \`POST https://<REDACTED-HOST>/api/cart/to-quote\` (or \`/api/cart/to-quote\` on the same Next.js app).
**File:** \`app/api/cart/to-quote/route.js\`. Helper auth + body-unwrap: \`libs/cart/index.js\` (\`checkAuth\`, \`unwrapCartBody\`). Zoho calls via \`libs/zoho\` barrel.
**Methods:**
- \`GET\` — returns a self-description \`{ ok:true, endpoint:"cart/to-quote", method:"POST with x-nexus-webhook-token header + JSON or form-data { data: <JSON> }" }\`. No auth.
- \`POST\` — the real endpoint. Auth required.

**Body (JSON or form-encoded \`data\` field wrapping JSON):**
\`\`\`json
{
  "email": "<REDACTED-EMAIL>",        // required
  "name":  "Customer Name",               // becomes Contact.Last_Name + Quote.Subject
  "phone": "",
  "items": [                              // required, non-empty
    { "sku": "EK0309", "name": "EK0309 ELISA Kit", "price": "599.00", "qty": 1 }
  ],
  "overview": { "shipping": 0, "coupon": 0, "coupon-code": "" },
  "shipping-address": { "Attn":"", "Street":"", "Street_2":"", "City":"", "State":"", "Zip":"", "Country":"US" },
  "billing-address":  { ... same shape ... }
}
\`\`\`
Address keys are passed verbatim with \`Shipping_\`/\`Billing_\` prefix into the Zoho Quote entity — i.e. the client sends Zoho-shaped field names (\`Attn\`, \`Street\`, \`City\`, \`State\`, \`Zip\`, \`Country\`), not generic ones. Empty addresses are OK (the loop just skips empty values).

**Server flow (5 steps, ordered):**
1. Find-or-create Zoho CRM Contact by Email (via \`searchOrWriteZCRM\` analog).
2. For each item: find-or-create Zoho CRM Product by Product_Code (= SKU).
3. Compose Quote entity with \`Quote_Stage:"Finalized"\`, \`EmailOnCreate:true\`, \`Valid_Till\` = today + 90 days, \`Subject\` = \`"Quote for <name>. <sku1> <sku2>..."\` (200-char cap), \`Description\` = the legacy "Note: This is a quote and is not an order..." paragraph, \`Product_Details[]\` (one entry per item + the hardcoded Shipping line item id \`1286149000000646000\` if \`overview.shipping > 0\`).
4. Spread shipping/billing address fields onto the Quote.
5. POST to Zoho CRM Quotes; Zoho auto-emails the contact because \`EmailOnCreate=true\`.

**Success response:**
\`\`\`json
{
  "ok": true,
  "quote_id": "1286149001438302001",
  "quote_number": null,                 // populated by Zoho async; null on immediate response
  "contact_id": "1286149000958973077",
  "contact_created": false,             // true if a new CRM Contact was created
  "products_created": 0,                // count of newly-created Products
  "raw": { ...Zoho's raw response... }
}
\`\`\`
**Errors:** 400 \`{ ok:false, error:"email is required" | "items must be a non-empty array" }\`; 500 \`{ ok:false, error:"<message>" }\` on any Zoho/network failure.

**curl example (verified 2026-05-21):**
\`\`\`bash
curl -X POST 'http://localhost:3001/api/cart/to-quote' \\
  -H 'x-nexus-webhook-token: <CART_WEBHOOK_TOKEN>' \\
  -H 'Content-Type: application/json' \\
  -d '{"email":"<REDACTED-EMAIL>","name":"CJ Xia","items":[{"sku":"EK0309","name":"EK0309 ELISA Kit","price":"599.00","qty":1}],"overview":{"shipping":0,"coupon":"","coupon-code":""}}'
\`\`\`

**CRITICAL gotcha — Zoho search vs list endpoint (FIXED 2026-05-21, but tag it for future Zoho consumers):**
The Zoho CRM v8 list endpoint \`/crm/v8/{module}\` SILENTLY IGNORES the \`criteria\` query param — it returns the most-recently-modified records regardless. The \`criteria\` filter ONLY works on the dedicated search endpoint \`/crm/v8/{module}/search\`. Calling \`zoho.getCrmRecords("Contacts", { criteria: "(Email:equals:<REDACTED-EMAIL>)" })\` returns the LATEST contact in the CRM, NOT the one matching the email. This caused early versions of cart/to-quote to attach every new quote to the same wrong contact.
**Always use \`zoho.searchCrmRecords(module, criteria, params?)\`** for find-or-match flows — it hits \`/search\` and honors criteria, and gracefully normalizes Zoho's 204-No-Content (zero matches) to an empty array. The wrong API was the silent-fail bug; the fix is in \`libs/zoho/index.js\` (\`searchCrmRecords\` method) + \`app/api/cart/to-quote/route.js\` (\`findOrCreateCrmRecord\` calls it instead of the list endpoint).

**OAuth/token model:** \`libs/zoho/auth.js\` \`ZohoAuthService\` (Supabase \`auth_tokens\` table, service-scoped client_id/secret from \`libs/zoho/config.js\`, refresh leeway 300s, max-age guard 3000s). Don't add env vars like \`ZOHO_REFRESH_TOKEN\` — that's the bioinvsync legacy pattern; Nexus stores tokens in DB.

**Hardcoded shipping product id \`1286149000000646000\`** is the legacy CRM Products id for "Shipping" — also in the route file as the constant \`SHIPPING_PRODUCT_ID\`. Move to env when convenient.

**Client-trusted price:** \`items[].price\` is whatever the client sends; the server does NOT re-lookup the catalog. Soft money-loss vector — flag for hardening if this endpoint is exposed beyond trusted webhook callers.

**Documentation page + playground:** \`app/(private)/nexus-armor/documentation/page.js\` (1030 lines, mirrors CJGEO docs-with-playground pattern). 14 endpoint cards across 3 groups. Server-side playground proxy at \`app/api/v1/nexus-playground/route.js\` (same-origin only; attaches \`x-nexus-webhook-token\` server-side for \`/api/cart/*\` paths so the secret never reaches the browser). Open at \`http://localhost:3001/nexus-armor/documentation\` (requires sign-in via the \`(private)\` layout).
`,
    },

    nexusArmorLegacyDispatcher: {
      description: "The /api/nexus-armor route is a Chrome-extension companion API — a single POST dispatcher that dynamic-imports actions/<action>.js. NOT the same as the per-action module layout in readModuleArchitectureRule; this is legacy and lives under /api/nexus-armor only because it predates the rule.",
      howTo: `
**Path:** \`app/api/nexus-armor/route.js\` — exports \`POST\` + \`OPTIONS\` (CORS wide-open). Imports \`@/libs/nexus-armor/cors\` (\`CORS_HEADERS\`, \`corsPreflight\`).
**Sub-routes (not part of the dispatcher):** \`app/api/nexus-armor/download-extension/route.js\`, \`app/api/nexus-armor/invoice-credit-hint/route.js\` — these are their own \`/api/nexus-armor/<name>\` paths, NOT \`action\` names.

**Dispatcher shape:**
\`\`\`js
POST /api/nexus-armor
Content-Type: application/json
{ "action": "<actionName>", "url": "<optional context url>", ...arbitrary body fields }
\`\`\`
- Validates \`action\` matches \`/^[a-zA-Z][a-zA-Z0-9]*$/\` (regex guard; junk → 400 \`{error:"Invalid action"}\`).
- Dynamic-imports \`./actions/<action>.js\` and calls \`handler({ action, url, body })\`.
- Returns the handler's JSON return value (typically \`{ html: "..." }\` HTML the Chrome extension injects into Zoho pages).
- Unknown action → 404 \`{error:"Unknown action: <action>"}\`. Handler throw → 500 \`{error:"<message>"}\`.

**Current actions (10):** \`initUI\` (domain-map dispatcher → routes to the right Zoho-service UI), \`dashboardUI\`/\`dashboardUINew\` (Zoho CRM dashboard injections), \`getZohoCRMUI\`/\`getZohoBooksUI\`/\`getZohoDeskUI\`/\`getZohoSalesIQUI\` (per-service UI HTML payloads), \`relatedRecordsLookup\` (CRM record lookup), \`showOpportunityNumberInProject\`/\`showProjectSOItems\` (Service Project widgets).

**Why it doesn't follow \`readModuleArchitectureRule\`:** the rule landed AFTER this route. New nexus-armor work uses \`app/(private)/nexus-armor/<action>/api/route.js\` (stuck-scan is the reference). This dispatcher stays as legacy because it's the Chrome extension's known endpoint URL; migrating would break the extension. Document new "actions" in the legacy dispatcher only if the Chrome extension needs them.
`,
    },

    deployRunPod: {
      description: "Build and push RunPod container image for the structure worker.",
      howTo: `
**Trigger:** When told to "update runpod" or "push live".

**Steps:**
\`\`\`bash
npm run runpod:push   # or: npm run push:live
\`\`\`

**What the script does:**
1. Commits uncommitted changes: \`git add -A\`, \`git commit -m "chore: update runpod container"\`, \`git push\`
2. Builds Docker image: \`docker build --platform linux/amd64 -t <REGISTRY>/boster-structure:latest runpod/structure\`
3. Pushes image: \`docker push <REGISTRY>/boster-structure:latest\`

**Prerequisites:**
- \`RUNPOD_IMAGE_REGISTRY\` set in \`.env.local\`
- Docker installed and logged in
- Git remote configured

**After push:** RunPod uses new image on next worker start. To force all workers: redeploy endpoint in RunPod Console (Endpoints → Edit → Save & Deploy).
`,
    },
  },
};
```
<!-- /GUILDOS-FILE -->
