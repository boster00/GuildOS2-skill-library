# bioinvsync

<!-- GUILDOS-SKILL name="bioinvsync" -->

<!-- GUILDOS-DESC -->
SSH into <REDACTED-HOST>, navigate the zohobooksAPI codebase, and execute Zoho CRM/Books workflows (quotes, orders, syncs). Server: <REDACTED-HOST> (<REDACTED-IP>), cPanel shared hosting; main app at <REDACTED-HOST>public_html/zohobooksAPI/. Use to run shell commands over SSH, map the server file layout, create + email Zoho CRM quotes from a SKU, and check web/PHP process health.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/bioinvsync/SKILL.md -->
```markdown
---
name: bioinvsync
description: >-
  SSH into <REDACTED-HOST>, navigate the zohobooksAPI codebase, and execute Zoho
  CRM/Books workflows (quotes, orders, syncs). Server: <REDACTED-HOST>
  (<REDACTED-IP>), cPanel shared hosting; main app at
  <REDACTED-HOST>public_html/zohobooksAPI/. Use to run shell commands over
  SSH, map the server file layout, create + email Zoho CRM quotes from a SKU,
  and check web/PHP process health.
---

# Bioinvsync — Server & Zoho Integration

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/bioinvsync/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto bioinvsync <action>` (or read the source directly).

## Actions

- **checkProcesses** — Check web server and PHP process health on <REDACTED-HOST>.
- **connectSsh** — Run a command on <REDACTED-HOST> via SSH using the guildos_ai key.
- **createQuote** — Create a Zoho CRM quote from a product SKU and email it to a customer.
- **createQuoteCIC** — MAP of the BosterBio website quote flow (where the cart/quote controls live + the price-mutation gotcha). For actual execution, prefer the createQuote API path — this is a fallback orientation map, not a click macro.
- **fileStructure** — Reference map of the <REDACTED-HOST> server file layout and key files.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/bioinvsync/index.js -->
```js
/**
 * Bioinvsync skill book — knowledge registry for <REDACTED-HOST> server and Zoho integration.
 *
 * Server: <REDACTED-HOST> (IP: <REDACTED-IP>), cPanel shared hosting (Jetrails-adjacent)
 * Main app: <REDACTED-HOST>public_html/zohobooksAPI/
 */

export const skillBook = {
  id: "bioinvsync",
  title: "Bioinvsync — Server & Zoho Integration",
  description: "SSH into <REDACTED-HOST>, navigate the zohobooksAPI codebase, and execute Zoho CRM/Books workflows (quotes, orders, syncs).",
  steps: [],
  toc: {
    connectSsh: {
      description: "Run a command on <REDACTED-HOST> via SSH using the guildos_ai key.",
      howTo: `
**Connection details** (all stored in formulary):
- Host: \`BIOINVSYNC_SSH_HOST\` = <REDACTED-HOST>
- Port: \`BIOINVSYNC_SSH_PORT\` = 2223
- User: \`BIOINVSYNC_SSH_USER\` = bioinvsync
- Key:  \`BIOINVSYNC_SSH_KEY_PATH\` = C:/Users/xsj70/guildos_ai.ppk  (PuTTY PPK v3, Argon2id)
- Pass: \`BIOINVSYNC_SSH_KEY_PASSPHRASE\` (from formulary)

**Command pattern (plink, passphrase piped to stdin):**
\`\`\`bash
printf "PASSPHRASE\\n" | "C:\\Program Files\\PuTTY\\plink.exe" -i "C:/Users/xsj70/guildos_ai.ppk" -P 2223 -ssh "bioinvsync@<REDACTED-HOST>" "YOUR_COMMAND_HERE"
\`\`\`

**Example — check uptime:**
\`\`\`bash
printf "PASSPHRASE\\n" | plink -i "C:/Users/xsj70/guildos_ai.ppk" -P 2223 -ssh bioinvsync@<REDACTED-HOST> "uptime"
\`\`\`

**Notes:**
- \`PASSPHRASE\` is a placeholder — read the real value from the formulary key \`BIOINVSYNC_SSH_KEY_PASSPHRASE\` at call time. Never inline the literal passphrase in this file or in committed scripts. ⚠ The passphrase that was previously hardcoded here has been exposed in git history and MUST be rotated (regenerate the PPK passphrase + update the formulary key).
- Host key is cached in PuTTY registry under \`HKCU\\SOFTWARE\\SimonTatham\\PuTTY\\SshHostKeys\`
- Do NOT use \`-batch\` flag — it blocks passphrase stdin injection
- Timeout: default 20000ms is usually sufficient for simple commands
- The server has been up 900+ days; load is typically near zero
`
    },

    fileStructure: {
      description: "Reference map of the <REDACTED-HOST> server file layout and key files.",
      howTo: `
**Home directory:** <REDACTED-HOST>

| Path | Purpose |
|------|---------|
| \`~/.env\` | Credentials: OPENAI_API_KEY and other secrets. Config is loaded by config.php via \`__DIR__ . '/../../.env'\` |
| \`~/public_html/\` | Web root |
| \`~/public_html/main.html\` | Landing page (no root index.php — 404 on bare domain is normal) |
| \`~/public_html/zohobooksAPI/\` | **Main application** — all API logic lives here |
| \`~/public_html/cjbase/\` | CJ-related utilities |
| \`~/public_html/elisa-analysis-backend/\` | ELISA analysis tool backend |
| \`~/public_html/erp/\` | ERP interface |
| \`~/public_html/spectra-viewer/\` | Spectral viewer tool |
| \`~/public_html/ortholog_database/\` | Ortholog lookup |

**zohobooksAPI/ key files:**

| File | Size | Purpose |
|------|------|---------|
| \`productionAPI.php\` | 439KB | Main action dispatcher — all \`?action=X\` routes live here. Entry for cartToQuote, cartToOrder, addEmailSequenceForUnredeemedQuote, etc. |
| \`class.php\` | 215KB | Core class \`ZohoBooks\` — all Zoho API calls (readZCRM, writeZCRM, searchOrWriteZCRM, ZCRMQuoteToZBEstimate, syncTickets, etc.) |
| \`api.php\` | 14KB | Lightweight entry point / router — includes config.php and routes to productionAPI.php or other handlers |
| \`devAPI.php\` | 54KB | Dev/debug version of productionAPI |
| \`config.php\` | 620B | Loads \`~/.env\` into PHP env vars |
| \`emailClass.php\` | 8.2KB | Email helper class |
| \`vtiger.php\` | 6.8KB | VTiger CRM integration |
| \`simonAPI.php\` | 11KB | Simon-related actions |
| \`eddyAPI.php\` | 54KB | Eddy-related actions |
| \`index.php\` | 3.9KB | HTML front-end shell |
| \`queue_processor.php\` | 4.9KB | Async queue processor |

**zohobooksAPI/ key subdirectories:**

| Dir | Purpose |
|-----|---------|
| \`vendor/\` | Composer dependencies |
| \`logs/\` | Runtime logs |
| \`tempuploads/\` | Temporary file uploads (47 subdirs) |
| \`CBI/\` | CBI-specific data/scripts |
| \`inventory/\` | Inventory tracking |
| \`pAIpeline/\` | AI pipeline components |
| \`genes/\` | Gene database |
| \`google-api/\` | Google API integration |
| \`FedEx/\` | FedEx shipping integration |
| \`erp/\` | ERP sub-app |

**Stack:**
- nginx (master at /opt/backups/bin/nginx) + PHP-FPM (PHP 7.2/7.3/7.1/7.0 pools)
- cPanel hosting, SSH port 2223
- No Node.js / no npm on server — pure PHP
`
    },

    createQuote: {
      description: "Create a Zoho CRM quote from a product SKU and email it to a customer.",
      howTo: `
**Endpoint:** POST (or GET) to \`https://<REDACTED-HOST>/zohobooksAPI/api.php?action=cartToQuote\`

**Full workflow (from cartToQuote in productionAPI.php):**
1. Look up or create ZCRM Contact by email (\`searchOrWriteZCRM("Contacts","Email",...)\`)
2. For each item: look up or create ZCRM Product by SKU (\`searchOrWriteZCRM("Products","Product_Code",...)\`)
3. Build quote entity: Contact, Quote_Stage="Finalized", Product_Details[], Subject, Valid_Till (+3 months), EmailOnCreate=true
4. Write quote to ZCRM via \`writeZCRM("Quotes", $entity)\`
5. Zoho CRM automatically sends email to the contact because EmailOnCreate=true

**Key quote entity fields:**
\`\`\`json
{
  "Contact_Name": {"id": "<contactZBID>"},
  "Quote_Stage": "Finalized",
  "Subject": "Quote for <name>. <sku1> <sku2>...",
  "Valid_Till": "<today + 3 months>",
  "EmailOnCreate": true,
  "Quote_Type": "Product",
  "Description": "Note: This is a quote and is not an order...",
  "Product_Details": [
    {"product": {"id": "<productZBID>"}, "quantity": 1, "list_price": 99.00},
    {"product": {"id": 1286149000000646000}, "quantity": 1, "list_price": 0}
  ]
}
\`\`\`
Note: Product id \`1286149000000646000\` = Shipping line item (always added last).

**Shipping address fields:** \`Shipping_Street\`, \`Shipping_City\`, \`Shipping_State\`, \`Shipping_Code\`, \`Shipping_Country\` (and Billing_ equivalents). Pass empty strings if not needed for test.

**Working curl pattern (confirmed 2026-05-04) — use the \`data\` JSON wrapper:**
\`\`\`bash
DATA='{"email":"<REDACTED-EMAIL>","name":"Customer Name","phone":"","items":[{"sku":"A02075-1","price":"279.00","name":"Product Name","qty":1}],"overview":{"shipping":0,"coupon-code":"","coupon":""},"shipping-address":{"Street":"","City":"","State":"","Code":"","Country":"US"},"billing-address":{"Street":"","City":"","State":"","Code":"","Country":"US"}}'

curl -s -X POST "https://<REDACTED-HOST>/zohobooksAPI/api.php" \\
  -d "action=cartToQuote" \\
  --data-urlencode "data=\$DATA"
\`\`\`

**IMPORTANT:** Always use the \`data\` JSON wrapper (not individual POST fields). Passing \`items\`, \`overview\`, \`shipping-address\` as separate fields sends them as plain strings — PHP cannot index into them as arrays and the items loop is skipped entirely. The \`data\` param triggers \`json_decode(\$_REQUEST['data'], 1)\` which correctly parses all nested arrays.

**Email template:** Handled by Zoho CRM's built-in quote email (triggered by EmailOnCreate=true). Template ID for ZB Estimate sync: \`2592134000000017003\`.

**Error log:** Failures written to \`~/public_html/zohobooksAPI/lastCartToQuoteRequestError\`

**To check product price for a SKU before quoting:**
Use the Zoho weapon: \`searchCrm("Products", limit)\` and filter by Product_Code, or look up via the Zoho CRM Products module directly.
`
    },

    createQuoteCIC: {
      description: "MAP of the BosterBio website quote flow (where the cart/quote controls live + the price-mutation gotcha). For actual execution, prefer the createQuote API path — this is a fallback orientation map, not a click macro.",
      howTo: `
**Execution default: use the \`createQuote\` API path, not this CIC flow.** The cart UI's selectors and radio choreography drift; the \`cartToQuote\` API is the robust path and is what the form ultimately writes to anyway. Drive CIC by hand here only when the API is unavailable. This action is a MAP of where things live + the one load-bearing gotcha — observe→act→observe via CIC, reading the live DOM each step; do not replay a hardcoded selector chain.

**Input:** customer name, email, optional phone, zipcode, list of SKUs with prices.

**Where the controls live (orientation, not a script):**
- **Product lookup** — search the live site (\`/catalogsearch/result/?q=<SKU>\`), confirm SKU + price from results, then open the product page URL *read from the DOM* (never guess the URL format).
- **Add to cart** — the product page has a variant radio group (pick the intended SKU variant) and a "free secondary antibody" opt-in/opt-out radio (choose "do not want" for a clean quote), then the add-to-cart control. Find these by their visible labels in the live DOM, not by index.
- **Cart page** (\`/checkout/cart/\`) — verify only the intended item(s) are present; remove strays via their per-row delete control.
- **Shipping estimator** — the "Estimate Shipping and Tax" section remembers the prior zip. Confirm zip = customer zip and that FedEx Priority Overnight is selected; expand + enter the zip if not pre-filled.
- **Get A Quote modal** — opened by a "Get A Quote" control on the cart page (it is rendered as a styled \`<span>\`, not a native \`<button>\` — locate it by its visible text). The modal carries name / phone / email fields (phone is required; use a placeholder if unknown) and a submit control whose label flips to a "Submitted!" confirmation on success.

**THE GOTCHA (price mutation is display-only):**
The quote reads price from Magento's **internal cart model**, NOT from the DOM. Editing the displayed Price cell changes the pixels only — the submitted price is unaffected. So a real discount (e.g. 30% off) is NOT achievable by DOM mutation. **Coupon-vs-API fork** for an actual discount:
  - **(a) Coupon path** — apply a real Magento coupon code that maps to the discount in the cart's "Apply Discount Code" field *before* opening the quote modal; the cart model then carries the discounted price.
  - **(b) API path (preferred)** — skip the UI and call \`cartToQuote\` directly with \`overview.coupon = "30%"\` (see the \`createQuote\` action for the exact payload).

**What the quote submits regardless of path:** product price = Magento's stored cart-model price (never the DOM-mutated value); shipping = the cart's FedEx estimate; no discount unless a real coupon was applied or the API \`overview.coupon\` was set. The quote lands in Zoho CRM and emails the customer via EmailOnCreate=true.
`
    },

    checkProcesses: {
      description: "Check web server and PHP process health on <REDACTED-HOST>.",
      howTo: `
Run via connectSsh:
\`\`\`bash
# Web server status
ps aux | grep -E 'nginx|php-fpm|apache' | grep -v grep

# Disk and memory
df -h / && free -m

# Error logs
tail -50 ~/public_html/zohobooksAPI/error.log
tail -20 ~/public_html/zohobooksAPI/logs/*.log 2>/dev/null
\`\`\`

**Known running processes:**
- nginx master at \`/opt/backups/bin/nginx\` (PID ~4350, up since 2024)
- 4x nginx workers
- PHP-FPM pools: php73, php72, php71, php70 (all managed by root)
- bioinvsync php-fpm pool for the site itself

**Restart note:** Processes are managed by cPanel/root. As the \`bioinvsync\` user, you cannot restart nginx or php-fpm directly. Use cPanel terminal or ask the hosting provider if a restart is needed.
`
    },
  },
};
```
<!-- /GUILDOS-FILE -->
