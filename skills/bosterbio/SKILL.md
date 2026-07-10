# bosterbio

<!-- GUILDOS-SKILL name="bosterbio" -->

<!-- GUILDOS-DESC -->
Execute BosterBio website tasks: product migration, storefront, launch prep, CMS content. Knowledge registry for the bosterbio.com2026 website migration — Magento EAV → Medusa v2 product migration (85,929 products), PLP/PDP/checkout storefront build, Figma-fidelity validation, SEO/DNS launch prep, and CMS page migration. Also documents the live bosterbio.com Magento operations surface: SSH access (Tailscale ProxyJump fallback), the /bapi entrypoint and BAPI product/CMS lookup actions, the CMS-mutation API for adopting CJGEO articles, product re-enable + search reindex, and vendor-image hydration on bundle-and-save PDPs. Use for any bosterbio.com migration, storefront, or live-site task.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/bosterbio/SKILL.md -->
```markdown
---
name: bosterbio
description: >-
  Execute BosterBio website tasks: product migration, storefront, launch prep,
  CMS content. Knowledge registry for the bosterbio.com2026 website migration —
  Magento EAV → Medusa v2 product migration (85,929 products), PLP/PDP/checkout
  storefront build, Figma-fidelity validation, SEO/DNS launch prep, and CMS page
  migration. Also documents the live bosterbio.com Magento operations surface:
  SSH access (Tailscale ProxyJump fallback), the /bapi entrypoint and BAPI
  product/CMS lookup actions, the CMS-mutation API for adopting CJGEO articles,
  product re-enable + search reindex, and vendor-image hydration on bundle-and-save
  PDPs. Use for any bosterbio.com migration, storefront, or live-site task.
---

# BosterBio.com2026 — Website Migration

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/bosterbio/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto bosterbio <action>` (or read the source directly).

## Actions

- **adoptCJGEOIntoMagentoPage** — Canonical pipeline: CJGEO raw article HTML → bosterbio.com Magento CMS page via POST /rest/V1/bapi/cms-page. Magento-targeted analogue of [SB:BryanBlog->adoptCJGEORawContent]: same sanitize / deep-link / image-fill discipline, BUT no Tailwind .prose-article greediness, no not-prose escape hatch, no Next.js dangerouslySetInnerHTML — Magento serves the body inside its own theme.
- **buildStorefront** — Build storefront pages (PLP, PDP, cart, checkout) following brand and Figma designs.
- **connectSsh** — Connect to bosterbio.com production server via SSH, falling back through Tailscale ProxyJump via Carbon or Jetrails whitelist when the current IP is blocked.
- **get_cms_pages** — Live BAPI action: retrieve Magento CMS page records (title, content, meta fields, stores) via direct SQL against cms_page.
- **get_products_by_skus** — Live BAPI action (folded down): retrieve enriched product JSON for 1–100 SKUs. Output is a pure subset of lookupProducts by=sku — use that instead.
- **get_products_by_urls** — Live BAPI action: resolve product URLs/URL keys to enriched product JSON (same shape as get_products_by_skus).
- **hydrateVendorImagesOnBundleAndSavePDP** — Get vendor product images to actually land in Magento media_gallery AND render on the live bundle-and-save PDP. Covers: server-side fallback URL chain (NitroCDN unwrap + WP responsive-size strip), classDev.php placeholder hazards inside the template, and the client-side hero-<img> hydrator that works around the broken Magento gallery init.
- **launchPrep** — Handle SEO, DNS cutover, monitoring, and launch checklist.
- **lookupProducts** — Live BAPI action (added 2026-05-21): unified product lookup by SKU, URL, or EntityID — single action replaces 'pick the right get_products_by_X for your input' decision. Same enrichment + same fields[] filter across all three identifier types.
- **migrateContent** — Migrate CMS pages from Magento to Next.js.
- **migrateProducts** — Migrate 85,929 products from Magento EAV to Medusa v2 hybrid schema.
- **publishWorkflowSolutionsPage** — Create or update a bosterbio.com/workflow-solutions/[slug] Magento CMS page that embeds the Boster Nexus B&S guide-article iframe. ⚠ The product-LISTING iframe embed is SUPERSEDED (2026-06-17): B&S products are now normal PIM products with a badge on the normal storefront — link to normal product/category pages, not a product-listing iframe.
- **reenableProductsAndReindexSearch** — Re-enable disabled live products (e.g. by source_company) and make them searchable. Covers BOTH search layers: Magento-native MySQL catalogsearch AND the site's custom CJ search index. Includes the stock-index and Cloudflare-cache gotchas that make a naive status flip look broken.
- **refBapiEntrypoint** — Reference (LOCKED): the live /bapi endpoint on bosterbio.com is served by the Magento theme template api.phtml. Absolute file path: <REDACTED-HOST>bosterbio.com/html/app/design/frontend/Bosterbio/theme/Magento_Theme/templates/api.phtml. Routing chain (LOCKED, operator-corrected 2026-05-22): URL /bapi is a Magento CMS page; that CMS page renders a block; the block's template handle is api.phtml. Because the request goes through Magento's normal CMS page route, Magento IS fully bootstrapped by the time api.phtml runs — there is NO direct pub/index.php → require api.phtml shim (any prior skill-book wording that claimed there was, was wrong). Cart-to-quote, submitCJAntibodyForm2, and any other passToAPI=1 action short-circuits to <REDACTED-HOST>/zohobooksAPI/api.php inside api.phtml before any Magento data is touched. Read before editing /bapi behavior or extending bapi actions.
- **refCMSMutationAPI** — Reference (LOCKED): wire spec for the live Magento CMS mutation endpoint POST /rest/V1/bapi/cms-page on bosterbio.com. 9-field body, locked handshake=<REDACTED-SHARED-SECRET>, idempotency on identifier, production-incident recovery commands. Read before any chaperone-side or Cursor-side call into this endpoint.
- **validateFigma** — Compare developed pages against Figma designs via screenshots.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/bosterbio/index.js -->
```js
/**
 * BosterBio skill book — knowledge registry for bosterbio.com2026 website migration.
 */

export const skillBook = {
  id: "bosterbio",
  title: "BosterBio.com2026 — Website Migration",
  description: "Execute BosterBio website tasks: product migration, storefront, launch prep, CMS content.",
  steps: [],
  toc: {
    migrateProducts: {
      description: "Migrate 85,929 products from Magento EAV to Medusa v2 hybrid schema.",
      howTo: `
**Hybrid schema model:**
- Type A (dedicated columns, search-critical): sku, title, handle, product_template, category, reactivity (text[]), applications (text[]), clone, host_species, badges, target_info (JSON), search_index
- Type B (flexible attributes): generic text columns attr_1...attr_25 + \`attribute_definitions\` table mapping (template, attr_key, label, type, display_order, required)

**Target info consolidation:** Merge gene/protein fields into single \`target_info\` JSON: gene_name, uniprot_id, synonyms, protein_function, gene_full_name, protein_name.

**Flexible attributes (Category 2, 14+ fields):** kit_components, cross_reactivity, reconstitution, predicted_reactivity, recommended_detection_systems, sensitivity, reproducibility, assay_range, sample_type, sequence_similarities, immunogen, purification, concentration, form, isotype, tissue_specificity, subcellular_localization, molecular_weight.

**Images:** Separate \`product_images\` table: product_id (FK), image_url, alt_text, ltx_description, position, type (hero/gallery/datasheet/swatch). Multiple products can reference same image.

**Data sources (ranked by preference):**
1. **Full product export CSV on the Magento server** — \`$MAGE_ROOT/pub/internal-export.csv\` (full path: \`<REDACTED-HOST>bosterbio.com/html/pub/internal-export.csv\`). Auto-regenerated; ~412 MB, ~568k lines (far more lines than products — description HTML fields contain embedded newlines inside quoted strings). **100 columns.** Key ones: sku(1), status(2), name(3), url_key(4), product_category(5), gene_name(6), price(8), size_1..10 + price_for_size_1..10 (9-28), badges(29), ribbons(30), transfer_price(31), clonality(33), clone_number(34), concentration(35), conjugate(36), description(38), short_description(39), uniprot_id(41), host(42), immunogen(43), form(44), purification(45), storage(46), cross_reactivity(47), isotype(48), sensitivity(50), kit_components(51), sample_type(52), applications(54), reactivity(57), predicted_reactivity(58), images(59), image_labels(60), custom_options(62), meta_title(69), meta_keyword(70), meta_description(71), source_company(74), background(77), research_category(79), synonyms(80), gene_full_name(81), molecular_weight(82), protein_function(83), subcellular_localization(84), tissue_specificity(85), protein_name(86), recommended_detection_systems(87), sequence_similarities(88), **template(100)**. The \`template\` column holds the plan's slug values (antibodies | elisa-kits | proteins | over-expression-lysates | …) — map directly to products.product_template; do NOT derive from product_category. NOTE: the sibling file \`pub/export.csv\` (81 cols, 260MB) is an older/thinner variant without template or meta_* — prefer internal-export.csv. The file \`pub/niches.csv\` is an empty header stub; ignore. **None of these are publicly served** — access via SSH only (see \`connectSsh\`). Smoke-test pattern:
   \`\`\`bash
   ssh -p 2223 boster_ooP9u@<REDACTED-IP> "head -c 20000000 <REDACTED-HOST>bosterbio.com/html/pub/internal-export.csv" > sample.csv
   \`\`\`
   Use byte-bounded head (\`-c\`) rather than line-bounded (\`-n\`), because HTML cells contain embedded newlines inside quoted strings — line count != product count. Parse the sample with a streaming CSV parser (csv-parse/stream) and stop after N product rows. Expect ~20MB ≈ 110 product rows in the ELISA-kit region of the file; for broader template coverage, pull more or seek past the first segment.
2. **Direct SQL on Magento DB** — SSH to the bosterbio.com production server (see \`connectSsh\`) and query MySQL (\`bosterbio_m2\` database). Use when you need data not in the CSV export (raw EAV rows, image blobs, custom tables).
3. **bosterbio.comLiveSite weapon (BAPI)** — only exposes gene actions today (readGenes, readGene, writeEnrichment). No product action yet; extend the BAPI PHP if you need live API-based product reads instead of a CSV snapshot.

**Migration flow:**
1. Extract products from the CSV export (or Magento EAV if CSV is missing fields)
2. Map dedicated fields (Type A)
3. Merge gene data into target_info JSON
4. Map flexible attrs to attr_1...N
5. Assign meanings via attribute_definitions table
6. Map images/LTX separately
7. Insert into Medusa
8. Validate per template

**Product options:** Use Medusa default model (115K options, 650K values). No custom consolidation.

**Dropped:** Magento system attributes, Yoast SEO, computed search weights, cost data, special pricing, related products system.

See \`docs/product-attributes-migration-plan.md\` for full schema details.
`,
    },
    buildStorefront: {
      description: "Build storefront pages (PLP, PDP, cart, checkout) following brand and Figma designs.",
      howTo: `
**Product listing page (PLP):**
- Filter by Type A attributes (category, reactivity, applications, clone, host_species)
- Preserve current URL format: \`/anti-xxx-antibody-xxxx.html\`

**Product detail page (PDP):**
- Scientific metadata, variant selection, image gallery, cart button
- Publications section (~20 entries per product, from publications table)
- Documents section (datasheets, etc.)

**Cart + Checkout:**
- Authorize.net payment (retain existing, not migrating to Stripe)
- Zoho Books = source of truth for tax

**Template system:** Template definitions drive rendering. Category must use Medusa category system (not collections).

**Brand:**
- Primary: deep blue #1a365d, Accent: warm orange #f97316
- Typography: Inter (next/font)
- Style: friendly, scientific, approachable
`,
    },
    validateFigma: {
      description: "Compare developed pages against Figma designs via screenshots.",
      howTo: `
**Process:**
1. Navigate to the developed page in Chrome
2. Take a viewport screenshot
3. Open the corresponding Figma design using the Figma weapon:
   \`\`\`javascript
   import { readExport } from '@/libs/weapon/figma';
   const png = await readExport({ fileKey, nodeId, format: 'png', scale: 2 });
   \`\`\`
4. Compare side-by-side: layout, colors, typography, spacing, component fidelity
5. Report differences with specific coordinates/elements

**Screenshot requirements:** Must provide screenshots for every page developed. Figma fidelity comparison is mandatory before marking work as complete.
`,
    },
    launchPrep: {
      description: "Handle SEO, DNS cutover, monitoring, and launch checklist.",
      howTo: `
**SEO:**
- Discard ALL Magento URL rewrites. Start clean with new routing.
- Generate XML sitemap
- Implement schema markups: Product, Organization
- NO bulk redirects — instead implement 404 monitoring system, surface high-frequency 404s, fix selectively

**GTM:** Migrate Google Tag Manager container. Validate all tags post-launch.

**Customers:** Migrate 2,352 accounts. Force password reset via email. Login requirement deferred.

**Orders:** Do NOT migrate historical orders. Zoho Books is system of record. Future: Medusa → Zoho (Phase 3).

**Images:** Migrate 171K product images from Magento server (via SSH) to CDN (R2/S3).

**Launch checklist:**
- [ ] All features tested (cart, checkout, payment, search, filters)
- [ ] No placeholder content; mobile responsive at all breakpoints
- [ ] Crawl all pages; eliminate dead/broken links
- [ ] Core Web Vitals audit
- [ ] Full backup; verify rollback procedure
- [ ] DNS cutover; re-submit sitemap; trigger reindexing
- [ ] Monitor 404s, indexing, performance for 2 weeks post-launch
`,
    },
    connectSsh: {
      description: "Connect to bosterbio.com production server via SSH, falling back through Tailscale ProxyJump via Carbon or Jetrails whitelist when the current IP is blocked.",
      howTo: `
**Primary:** \`ssh -p 2223 boster_ooP9u@<REDACTED-IP>\`

**Diagnosis when the primary times out:** TCP \`Connection timed out\` on port 2223 means JetRails' SSH whitelist does not include the client's egress IP. The SSHd is up; the firewall is silently dropping. Don't retry-loop — switch to Option 1 below.

---

**Option 1 — Tailscale ProxyJump via Carbon (LOCKED: VERIFIED 2026-05-29, the working fallback).**

Carbon (the Boster house workstation; Tailscale hostname \`<REDACTED-HOST>\` at \`<REDACTED-IP>\`) has a stable outbound IP that IS on the JetRails whitelist. Even when this client's IP isn't, Carbon's is. Tailscale gives a direct route to Carbon from anywhere with internet — no LAN dependency, no VPN, no key copying — so this fallback works from any networked environment as long as Tailscale is running on both ends.

**The working command (single line, uses local key end-to-end):**
\`\`\`bash
ssh -J xsj70@<REDACTED-IP> -p 2223 boster_ooP9u@<REDACTED-IP>
\`\`\`

\`-J\` (ProxyJump) opens a TCP tunnel through <REDACTED-HOST> to \`<REDACTED-IP>:2223\` and then runs the OUTER ssh handshake end-to-end against the live server through that tunnel. Auth at BOTH hops uses your **local** \`~/.ssh/id_ed25519\` (no key on Carbon, no agent forwarding fragility). Works for interactive shells AND scp:

\`\`\`bash
# interactive shell
ssh -J xsj70@<REDACTED-IP> -p 2223 boster_ooP9u@<REDACTED-IP>

# scp UP (subject to Rule 3 / Rule 4 — proposed-delta workflow only; operator applies)
scp -J xsj70@<REDACTED-IP> -P 2223 localfile boster_ooP9u@<REDACTED-IP>:/tmp/

# scp DOWN (read-only — fine)
scp -J xsj70@<REDACTED-IP> -P 2223 boster_ooP9u@<REDACTED-IP>:/path/file ./local
\`\`\`

**Prereqs (one-time, already set up on this client):**
- Tailscale CLI installed. On Windows: \`/c/Program Files/Tailscale/tailscale.exe\`.
- This client logged into the same tailnet as Carbon. Verify:
  \`\`\`bash
  tailscale status | grep <REDACTED-HOST>
  # Expect: <REDACTED-IP>  <REDACTED-HOST>  xsj706@  windows  ...
  \`\`\`
  If <REDACTED-HOST> is missing or offline → Tailscale path is unavailable; fall through to Option 2.
- Carbon has OpenSSH server running with this client's pubkey in \`authorized_keys\` (verified: \`C:\\Windows\\System32\\OpenSSH\\ssh.exe\` accepts BOOK-JMN7OQJKEO).

**How the auth chain works (why this beats the direct path):**
1. Client → <REDACTED-HOST> on Tailscale <REDACTED-IP>:22 — local key, WireGuard tunnel, independent of JetRails' firewall.
2. <REDACTED-HOST> → <REDACTED-IP>:2223 — TCP forward only, no auth at this layer, egress IP is Carbon's (whitelisted).
3. Client → <REDACTED-IP> SSH handshake through that tunnel, local key again. JetRails sees Carbon's IP as the source and allows it.

**Failure signatures (don't retry blindly):**
- \`connect to host <REDACTED-IP> port 22: Connection timed out\` — Tailscale is down on this client or on <REDACTED-HOST>. Open Tailscale tray, log in, retry.
- \`xsj70@<REDACTED-IP>: Permission denied (publickey)\` — local key not authorized on <REDACTED-HOST>. Surface for re-key (don't normally happen if connectSsh was set up).
- \`boster_ooP9u@<REDACTED-IP>: Permission denied (publickey,...)\` with \`-J\` — outer hop reached but key auth failed. Confirm \`~/.ssh/id_ed25519\` exists; do NOT switch to \`ssh -A\` (agent forwarding) as a workaround — its nested-SSH failure modes are why we use \`-J\`.
- \`boster_ooP9u@<REDACTED-IP>: Connection refused\` — sshd actually down on the live server; JetRails-side incident, escalate.
- \`Connection closed by <REDACTED-IP> port 2223\` (or \`...by UNKNOWN port 65535\` through \`-J\`) — sshd is ALIVE and answering at the SSH layer (a dead daemon gives "refused" or "timed out", never an active close). This is almost always rate-limiting / fail2ban triggered by a burst of probe attempts (e.g. liveness-checking in a loop). Do NOT keep retrying — that deepens the ban. Back off ~5–15 min, then one clean ProxyJump. Persisting past the cooldown → escalate to JetRails. NB: a banner-exchange timeout from THIS client but an active close from Carbon's whitelisted IP = the normal IP-whitelist condition, not an outage.

**Deprecated LAN hop.** Earlier versions of this skill book documented \`ssh carbon\` (LAN <REDACTED-IP>) then \`ssh -p 2223 boster_ooP9u@...\` from within Carbon. That needed (a) LAN reachability and (b) a copy of the boster key on Carbon. The Tailscale ProxyJump above replaces both. Do not use the LAN-then-jumphost pattern unless Tailscale is genuinely down — and even then prefer Option 2.

---

**Option 2 — Request IP whitelist via Jetrails support:**
If Tailscale is also unavailable, get the current IP and send an email from <REDACTED-EMAIL> to <REDACTED-EMAIL>:
\`\`\`
Subject: SSH Whitelist Request — Add IP [current IP] for bosterbio.com
Body: Hi Jetrails support, please add IP address [current IP] to the SSH whitelist for bosterbio.com (port 2223, user boster_ooP9u). Thank you.
\`\`\`
Use the Gmail weapon with the <REDACTED-EMAIL> account to send.
After sending, retry SSH in 20-minute intervals for up to 2 hours before escalating.

**Get current IP:** \`curl -s https://api.ipify.org\`
`,
    },
    migrateContent: {
      description: "Migrate CMS pages from Magento to Next.js.",
      howTo: `
**Inventory:** 755 total CMS pages from Magento.
- KEEP (migrate): 481 pages
- DROP: 7 pages (Magento system routes like no-route, enable-cookies)
- NEEDS REVIEW: 267 pages (inactive or short content — refine with Analytics/Search Console)

**URL prefix groups (by page count):** newsletter-archive (131), pathway-maps (85), protocol-and-troubleshooting (70), diseases (54), promotions (24), cell-types (23), FAQ (18), research-area (15), services/* (12+)

**URL placeholder convention:** Absolute bosterbio.com URLs in exports rewritten to \`https://SITE_ORIGIN_PLACEHOLDER\` for safe commits. Restore real origins when deploying.

**Strategy:** Migrate only high-traffic pages (per GA/BigQuery). Use template library system for CMS pages.

See \`docs/cms-html-template-guide.md\` and \`docs/cms-page-audit.md\` for full inventory.
`,
    },
    publishWorkflowSolutionsPage: {
      description: "Create or update a bosterbio.com/workflow-solutions/[slug] Magento CMS page that embeds the Boster Nexus B&S guide-article iframe. ⚠ The product-LISTING iframe embed is SUPERSEDED (2026-06-17): B&S products are now normal PIM products with a badge on the normal storefront — link to normal product/category pages, not a product-listing iframe.",
      howTo: `
⚠ **ARCHITECTURE CHANGE (2026-06-17, operator directive):** Bundle & Save PRODUCTS are no longer served via an iframe-embedded catalog — they are now ingested through the PIM as NORMAL products carrying a "Bundle & Save" badge (\`pim_products.badges[]\`), living on the normal bosterbio.com storefront/cart/checkout. So the \`workflow-products\` product-listing embed below (\`/bundle-and-save/supplier/…\`, \`/bundle-and-save/cat/…\`, \`/bundle-and-save\`) is RETIRED — link to the normal product/category pages instead. The \`workflow-article\` guide embed is NOT affected by that decision and remains valid. Rationale: boster_nexus \`libs/bundle-save/engineering_decisions.md\` + \`libs/pim/engineering_decisions.md\` (2026-06-17 entries).

**Overview:** /workflow-solutions/[slug] pages are Magento CMS pages with two B&S embeds:
1. \`workflow-article\` div — embeds the B&S guide article (e.g. \`/bundle-and-save/multiplex-if-solution\`)
2. \`workflow-products\` div — embeds the B&S supplier or category listing (e.g. \`/bundle-and-save/supplier/biotium.com\`)

Each embed uses the \`data-path\` + \`data-container\` attributes on the bundle-save-embed.js loader so the iframe path is hardcoded (not derived from window.location.pathname).

**Embed snippet template (paste into CMS page content):**
\`\`\`html
<div id="workflow-article" class="full-width-section"><script>(function(){var D="http://localhost:3001",P="https://<REDACTED-HOST>";function load(o){var s=document.createElement("script");s.setAttribute("data-path","/bundle-and-save/YOUR-ARTICLE-SLUG");s.setAttribute("data-container","workflow-article");s.src=o+"/bundle-save-embed.js?t="+Date.now();document.getElementById("workflow-article").appendChild(s);}var c=typeof AbortController!=="undefined"?new AbortController():null;var t=c?setTimeout(function(){c.abort();},1500):null;fetch(D+"/api/bundle-save/health",{signal:c?c.signal:null,cache:"no-store",mode:"no-cors"}).then(function(){clearTimeout(t);load(D);}).catch(function(){load(P);});})()</script></div><div id="workflow-products" class="full-width-section"><script>(function(){var D="http://localhost:3001",P="https://<REDACTED-HOST>";function load(o){var s=document.createElement("script");s.setAttribute("data-path","/bundle-and-save/supplier/YOUR-VENDOR-DOMAIN");s.setAttribute("data-container","workflow-products");s.src=o+"/bundle-save-embed.js?t="+Date.now();document.getElementById("workflow-products").appendChild(s);}var c=typeof AbortController!=="undefined"?new AbortController():null;var t=c?setTimeout(function(){c.abort();},1500):null;fetch(D+"/api/bundle-save/health",{signal:c?c.signal:null,cache:"no-store",mode:"no-cors"}).then(function(){clearTimeout(t);load(D);}).catch(function(){load(P);});})()</script></div>
\`\`\`

**For product data-path, prefer:**
- Specific vendor: \`/bundle-and-save/supplier/biotium.com\` (when article targets a vendor's products)
- Specific category: \`/bundle-and-save/cat/SLUG\` (only if the category has products — verify first via /api/bundle-save/storefront/products?category=SLUG)
- All products: \`/bundle-and-save\` (fallback when no specific category/vendor applies)

**Steps to publish a new page:**
1. Confirm article slug exists in boster_nexus \`app/(public)/embed/bundle-save/router.js\` — add the route if missing.
2. Use \`bosterbio.comLiveSite.writeCmsPage\` weapon to write the content. Pass \`identifier: 'workflow-solutions/[slug]'\`.
3. Weapon auto-flushes full_page cache. Immediately call \`bosterbio.comLiveSite.warmPageCache({ path: '/workflow-solutions/[slug]' })\` — this prevents Varnish 503 by pre-rendering the PHP page on nginx:8080 before Varnish tries to fetch it.
4. Verify the URL rewrite exists in Magento's \`url_rewrite\` table. If creating a brand-new page (not updating an existing one), also insert:
   - \`workflow-solutions/[slug]\` → \`cms/page/view/page_id/[id]\` (entity_type='cms-page', redirect_type=0, store_id=1)
   - \`bundle-and-save/[slug]\` → \`workflow-solutions/[slug]\` (entity_type='custom', redirect_type=301, store_id=1) if the old path existed.
   Use \`sshMysql\` via the weapon's internal helper, or SSH directly.
5. Reload the live page in browser to confirm both embeds render.

**Varnish warm-up note:** After any cache:flush, the first uncached request to a Magento page hits PHP cold, which takes 10–30s. Varnish's \`first_byte_timeout\` is shorter than this, causing a 503. The fix is always to hit the page directly on nginx:8080 with \`X-Forwarded-Proto: https\` before the public URL is requested. \`warmPageCache\` in the weapon does this automatically.

**Magento root:** \`<REDACTED-HOST>bosterbio.com/html/\`
**MySQL:** host=<REDACTED-IP>, db=bosterbio_m2, user=bosterbio_user (weapon reads creds from env).
`,
    },
    reenableProductsAndReindexSearch: {
      description: "Re-enable disabled live products (e.g. by source_company) and make them searchable. Covers BOTH search layers: Magento-native MySQL catalogsearch AND the site's custom CJ search index. Includes the stock-index and Cloudflare-cache gotchas that make a naive status flip look broken.",
      howTo: `
**There are TWO independent search layers on bosterbio.com — fixing one is not enough:**
1. **Magento-native MySQL catalogsearch** (\`catalog/search/engine = mysql\`) — powers \`/catalogsearch/result/?q=...\` keyword/name search. Backed by table \`catalogsearch_fulltext_scope1\`.
2. **Custom CJ search index** (bespoke site code; *do not reverse-engineer unsupervised*) — powers the site's bare-SKU search. Completely separate from Magento; a Magento reindex does NOT touch it.

**Key schema facts (Magento Open Source schema — link column is \`entity_id\`, NOT \`row_id\`; no staging):**
- \`status\`: EAV **int** attribute, \`attribute_id = 96\`. Value **1 = Enabled, 2 = Disabled**. Default scope = \`store_id = 0\`. Stored in \`catalog_product_entity_int\`.
- \`source_company\`: EAV **varchar** attribute, \`attribute_id = 270\`. Stored in \`catalog_product_entity_varchar\`.
- \`catalog_product\` entity_type_id = 4. Confirm ids before relying on them: \`SELECT a.attribute_id,a.attribute_code,a.backend_type FROM eav_attribute a JOIN eav_entity_type t ON a.entity_type_id=t.entity_type_id WHERE t.entity_type_code='catalog_product' AND a.attribute_code IN('source_company','status');\`

**Procedure (SSH → MySQL; see \`connectSsh\`. SSH: \`-p 2223 boster_ooP9u@<REDACTED-IP>\`, then \`mysql -h127.0.0.1 -ubosterbio_user -p... bosterbio_m2\`):**
1. **Scope it first.** \`SELECT s.value status,COUNT(*) FROM catalog_product_entity_varchar v JOIN catalog_product_entity_int s ON s.entity_id=v.entity_id AND s.attribute_id=96 AND s.store_id=0 WHERE v.attribute_id=270 AND v.value LIKE '%calbio%' GROUP BY s.value;\` Also check there are no store-scoped (\`store_id<>0\`) status overrides.
2. **Flip status:** \`UPDATE catalog_product_entity_int s JOIN catalog_product_entity_varchar v ON v.entity_id=s.entity_id AND v.attribute_id=270 SET s.value=1 WHERE s.attribute_id=96 AND s.store_id=0 AND v.value LIKE '%calbio%';\`
3. **STOCK-INDEX GOTCHA (the #1 reason "I enabled it but search can't find it"):** A raw SQL status write fires Magento's mview triggers for catalogsearch / EAV / price / category, but **NOT** the stock indexer. The product ends up with **zero rows in \`cataloginventory_stock_status\`**. Magento's MySQL catalogsearch query INNER-JOINs that table, so the product is **excluded from native search results even though its PDP loads fine** (the product detail page does not need the stock index). The mview backlog draining to 0 (via \`cron:run --group=index\`) is NOT sufficient — it never enqueued stock. Fix with a real reindex:
   \`\`\`bash
   cd <REDACTED-HOST>bosterbio.com/html && php bin/magento indexer:reindex cataloginventory_stock inventory && php bin/magento cache:flush
   \`\`\`
   This rebuilds \`cataloginventory_stock_status\` (and cascades price + catalog search). Takes ~2–3 min on the full catalog. Verify: \`SELECT COUNT(*) FROM cataloginventory_stock_status s JOIN catalog_product_entity_varchar v ON v.entity_id=s.product_id AND v.attribute_id=270 AND v.value LIKE '%calbio%';\` should now equal the product count. (Products must have \`cataloginventory_stock_item\` source rows for this to produce in-stock status — disabled products normally still have them.)
4. **CUSTOM CJ SEARCH INDEX (powers bare-SKU search; stock is NOT a criterion here — that was Magento-native only).** The native reindex above does NOT update the custom index. Feed it one of two ways:
   - **Bulk (works, but ASYNCHRONOUS — owner-confirmed):** GET \`https://www.bosterbio.com/cj-custom-script?action=reindexCJSearch\` — the \`data=\` query param is an **unused placeholder** (FULL reindex regardless of what you pass). The request holds the connection ~1 min+, then the index keeps populating **in the background for a while afterward**. Do NOT conclude a product "failed" because it isn't searchable immediately — coverage trickles in over minutes. (Worked example: right after a bulk reindex, EK7001 was searchable but EK7132 was not; EK7132 became searchable a few minutes later as the background pass caught up — an admin Save done in between was coincidental, not the cause. A later \`q=EK70\` prefix search returned 98 EK70xx kits, confirming broad coverage.) This is custom site code: trigger it, wait, re-verify; do not re-implement it.
   - **Force a SINGLE product immediately (skip the wait):** run a real Magento product save, which fires the custom CJ hook synchronously for that one product. Either (a) Magento admin \`https://www.bosterbio.com/badmin/catalog/product/edit/id/<entity_id>\` → **Save** (wait for "You saved the product."), or (b) the **\`saveProduct\` BAPI action** (added in \`pub/bapi.php\` — see below), which is the same path, scriptable, ~2s.

**\`saveProduct\` BAPI action** (lives in \`pub/bapi.php\`, routed via \`pub/index.php\`'s \`/bapi\` path shim — Magento is NOT bootstrapped by that shim, so the action bootstraps Magento itself):
\`GET https://www.bosterbio.com/bapi?action=saveProduct&sku=<SKU>&api_key=<BAPI_API_KEY>\` (the legacy \`?_bapi=cjgeo\` query-flag shim was removed 2026-05-20 for security — only the explicit \`/bapi\` path is honored now)
It bootstraps Magento, sets area \`adminhtml\`, then \`ProductRepositoryInterface::get(sku,true,0,true)\` + \`->save()\` — the identical save path the admin UI "Save" button uses, so all \`catalog_product_save_*\` observers/plugins (incl. the CJ hook) fire. Returns JSON with \`entity_id\`, \`updated_at_before\`, \`updated_at_after\` (the timestamp advancing proves the save committed). Verified <2s/call vs ~15–30s for a UI save. \`pub/bapi.php\` is plain PHP (fixed \`BAPI_API_KEY\`, direct mysqli for gene actions); back it up (\`cp -p\`) before editing and \`php -l\` after.

**Preferred call path (reusable):** use the weapon — \`bosterbio.comLiveSite.saveProduct({ sku })\` (it resolves the api_key, uses a 90s timeout for cold Magento boot, returns \`{ ok, sku, entity_id, updated_at_before, updated_at_after }\`). Only hand-build the raw \`/bapi?action=saveProduct\` URL for ad-hoc shell checks.

**Cloudflare gotchas (verification):**
- **Edge cache:** \`/catalogsearch/result/?q=...\` responses are cached by Cloudflare; a stale page shows "We cannot seem to find any products" (or a 404) even after the product is indexed. Append a throwaway param (\`?q=EK7001&cb=<random>\`, any extra \`&...\`) to bust it.
- **Bot challenge:** rapid/bulk \`curl\` against the public host gets the Cloudflare "Just a moment..." JS challenge (do NOT try to solve it). For reliable bulk verification, query the **origin** over SSH instead: \`curl -s -H 'Host: www.bosterbio.com' -H 'X-Forwarded-Proto: https' "http://<REDACTED-IP>:8080/catalogsearch/result/?q=<SKU>&cb=<rand>"\` (bypasses Cloudflare; same technique as \`warmPageCache\`) — though the catalogsearch page is heavy, so keep concurrency low. The real CIC browser passes the challenge automatically. A broad prefix query (e.g. \`q=EK70\`) in one CIC hit is the cheapest coverage check — far better than 84 per-SKU lookups.

**Magento root:** \`<REDACTED-HOST>bosterbio.com/html/\`. **MySQL:** host=<REDACTED-IP>, db=bosterbio_m2, user=bosterbio_user.
`,
    },
    get_products_by_skus: {
      description: "Live BAPI action (folded down): retrieve enriched product JSON for 1–100 SKUs. Output is a pure subset of lookupProducts by=sku — use that instead.",
      howTo: `
**Folded down (2026-06-05):** \`lookupProducts\` with \`by="sku"\` returns the byte-equivalent product shape this action returned (a pure subset — \`lookupProducts\` keys by the original input string and adds the unified envelope). Use \`lookupProducts({ by: 'sku', identifiers: [...] })\` (weapon) / \`?action=lookupProducts&by=sku\` (HTTP) instead. The standalone \`get_products_by_skus\` action + weapon function remain for backward compatibility; the full lookup contract (output shape, fields[] filter, partial-failure handling, smoke tests) lives in \`lookupProducts\` below.
`,
    },
    get_products_by_urls: {
      description: "Live BAPI action: resolve product URLs/URL keys to enriched product JSON (same shape as get_products_by_skus).",
      howTo: `
**Endpoint:** \`GET https://www.bosterbio.com/bapi?action=get_products_by_urls&urls=<comma-sep>&api_key=<key>\`

Accepts full URLs (\`https://www.bosterbio.com/foo.html\`), path-style (\`foo.html\`), or url-key only (\`foo\`). Up to **100 per request**. Either \`?urls=<comma-sep>\` or single \`?url=\`.

**Preferred call path:** \`bosterbio.comLiveSite.get_products_by_urls({ urls: [productUrlA, productUrlB], userId })\`.

**Output shape:**
\`\`\`json
{
  "success": true,
  "action": "get_products_by_urls",
  "count": 2,
  "products": {
    "<original input URL/path/key>": {
      "resolved_sku": "EK0308",
      "resolved_entity_id": 7,
      "matched_path": "rat-bdnf-picokine-trade-elisa-kit-ek0308-boster.html",
      "match_method": "url_rewrite" | "url_key_attribute",
      "product": { /* SAME enriched shape as get_products_by_skus.products[SKU] — byte-equivalent via shared helper */ }
    }
  },
  "not_found": [],
  "errors": []
}
\`\`\`

**Resolution path:** SQL primary against \`url_rewrite\` (\`entity_type='product'\`, scoped \`store_id IN (0,1)\`), EAV \`url_key\` fallback (\`catalog_product_entity_varchar\` join \`eav_attribute\` where \`attribute_code='url_key'\`) when the rewrite misses. Magento is bootstrapped ONCE per request to load the resolved products.

**Output keying:** by the **original input string** the caller sent (not by resolved SKU, not by normalized path) — preserves caller's array correlation.

**Partial failure:** unresolved URLs land in \`not_found[]\`; per-URL exceptions land in \`errors[{url,error}]\`.

**Smoke-tested with:** product URLs derived from \`EK0308\` + \`PA1568\`'s \`product_url\` field in the SKU response — round-trip closes (URL→SKU resolves back to the same SKUs).

**Future-agent rule:** When you have a product URL and want enriched product data, **call this BAPI action** — do NOT re-implement URL→entity_id SQL each time. The retrieval logic lives once, in \`pub/bapi.php\`, and is consumed via the \`bosterbio.comLiveSite\` weapon.

**\`fields[]\` filter:** \`get_products_by_urls\` accepts the same optional \`fields\` param documented canonically in \`lookupProducts\` below. The filter applies to the inner \`product\` object — the wrapper keys (\`resolved_sku\`/\`matched_path\`/\`match_method\`/\`resolved_entity_id\`) are unaffected.
`,
    },

    lookupProducts: {
      description: "Live BAPI action (added 2026-05-21): unified product lookup by SKU, URL, or EntityID — single action replaces 'pick the right get_products_by_X for your input' decision. Same enrichment + same fields[] filter across all three identifier types.",
      howTo: `
**Endpoint:** \`POST https://www.bosterbio.com/bapi\` (JSON body) — or \`GET ?action=lookupProducts&by=<type>&identifiers=<comma-sep>\`. Authorization: \`Bearer $BAPI_API_KEY\` header (or \`?api_key=\`).

**Body / params:**
- \`by\`: \`"sku"\` | \`"url"\` | \`"entity_id"\` (alias \`"id"\`) — required. Tells the resolver which kind of identifier you're sending.
- \`identifiers\`: array (\`["SKU1","SKU2"]\`) **or** comma-string (\`"SKU1,SKU2"\`). Up to **100 per request**. Singular shortcut also works (\`{by:"sku", sku:"EK0309"}\` or \`?sku=EK0309\` with \`by=sku\`).
- \`fields\`: optional array of top-level keys to keep. Empty/missing = return full enrichment. **(Canonical fields[]-filter doc — added 2026-05-21; all three product-lookup actions share it: \`lookupProducts\`, \`get_products_by_urls\`, and the folded-down \`get_products_by_skus\`.)** Three accepted syntaxes: \`?fields=sku,name,price\` (comma-string) · \`?fields[]=sku&fields[]=name\` (array) · JSON body \`{"fields":["sku","name","price"]}\`. Available keys: \`sku, entity_id, type_id, attribute_set_id, name, status, visibility, price, special_price, url_key, product_url, attributes, images, options, categories\`. For \`get_products_by_urls\` the filter narrows the inner \`product\` object only (the resolution wrapper is unaffected).

**Output shape:** unified across all three \`by\` modes — products keyed by the **original input string** (preserves caller's array correlation, regardless of identifier type):
\`\`\`json
{
  "success": true,
  "action": "lookupProducts",
  "by": "sku" | "url" | "entity_id",
  "count": 2,
  "max_allowed": 100,
  "fields": [],                    // echoes back the requested filter (empty array = no filter)
  "products": {
    "<original-input>": { /* enriched product (full or filtered) */ }
  },
  "not_found": [...],              // identifiers that didn't resolve to a product
  "errors": [{ identifier, error }] // per-identifier exceptions
}
\`\`\`

Notes:
- Unlike \`get_products_by_urls\`, the new action returns the enriched product **directly** under each key (no wrapper). If you need the URL-resolution metadata (\`matched_path\`/\`match_method\`/\`resolved_sku\`), keep using \`get_products_by_urls\`; otherwise \`lookupProducts by=url\` is the simpler ergonomic.
- \`by=sku\` resolves via a direct \`catalog_product_entity\` DB lookup before bootstrapping Magento — fast.
- \`by=url\` reuses the exact url_rewrite + url_key EAV resolution path from \`get_products_by_urls\`.
- \`by=entity_id\` validates digit-only input then \`ProductRepository::getById()\` directly — cheapest path.

**Smoke-tested 2026-05-21** with all three modes + the fields filter + GET-style + JSON-body invocation + not_found mixed with found.

**Future-agent rule:** When the caller might have an SKU OR a URL OR an EntityID, **use \`lookupProducts\`** — don't make the caller pre-classify. If they always have only one kind, the per-type \`get_products_by_*\` actions are fine and remain canonical for their specific shapes (URL action wraps with resolution metadata; SKU action is the simplest).
`,
    },

    adoptCJGEOIntoMagentoPage: {
      description:
        "Canonical pipeline: CJGEO raw article HTML → bosterbio.com Magento CMS page via POST /rest/V1/bapi/cms-page. Magento-targeted analogue of [SB:BryanBlog->adoptCJGEORawContent]: same sanitize / deep-link / image-fill discipline, BUT no Tailwind .prose-article greediness, no not-prose escape hatch, no Next.js dangerouslySetInnerHTML — Magento serves the body inside its own theme.",
      howTo: `
This is the codified end-to-end procedure to adopt a CJGEO-generated article into a bosterbio.com Magento CMS page (proving-grounds destination). It is the Magento-targeted sibling of [SB:BryanBlog->adoptCJGEORawContent] — same sanitize / cross-link / image-fill discipline, but the destination is Magento's \`cms_page\` table (served by Magento's PHP-rendered theme), NOT a Next.js / Tailwind page. Branding/voice on this destination is hand-reviewed by the operator (PG pages = internal review surface) — no automated branding-grep is applied here, unlike BryanBlog. Anything else not addressed below defaults to the BryanBlog adoption procedure per the locked precedent rule (see "BryanBlog precedent" below).

**Driver skill books.**
- CJGEO HTTP wire spec — see [SB:cjgeo->refApiActions]. Don't restate it here.
- BryanBlog adoption precedent — see [SB:BryanBlog->adoptCJGEORawContent]. When this howTo is silent on an adoption detail (anchor injection, deep-dive link injection idiom), default to BryanBlog's behavior. Branding/voice rules are NOT inherited (hand-reviewed on this destination — see Step 5).
- Image-placeholder fill — see [SB:cjgeo->refImagePlaceholderHandling]. **LOCKED mandatory step on every adoption.** This howTo overrides only the storage path (Magento media library, NOT \`public/images/\`).
- Magento CMS mutation API wire spec — see [SB:bosterbio->refCMSMutationAPI] for the endpoint shape, the locked \`handshake=<REDACTED-SHARED-SECRET>\` value, idempotency rules, the 9-field body, and the production-incident recovery commands.

---

### Inputs

- \`articleId\` — CJGEO article uuid (the source of \`content_html\`).
- \`slug\` — Magento CMS \`identifier\` for the destination page. Pages under proving-grounds live at \`proving-grounds/<slug>\` (Magento CMS \`identifier\` supports \`/\` — that is the canonical home for these internal-review pages). The 4 western-blot mockups are \`proving-grounds/cjgeo-mockup-1\` through \`proving-grounds/cjgeo-mockup-4\`. Also drives the page URL (\`https://www.bosterbio.com/proving-grounds/<slug>\`) and the media subdirectory (\`/media/images/proving-grounds/<slug>/\`).
- \`mainKeyword\` — the topic's main keyword. Used to build the hero \`<h1>\` text and as the OCR-hero-text match target during Cat-side verification per [SB:questmaster->ocrHeroText].
- \`title\` — the page title (Magento \`title\` field; also drives \`<title>\` in the head and the hero block). Usually a verbatim or near-verbatim restatement of the topic phrase (e.g. "How to decide gel percentage in Western blot").
- \`siblingSlugs\` (optional) — array of sibling Magento CMS identifiers for cross-linking (e.g. \`['proving-grounds/cjgeo-mockup-1','proving-grounds/cjgeo-mockup-2','proving-grounds/cjgeo-mockup-3','proving-grounds/cjgeo-mockup-4']\`). Drives the deep-dive link injection per BryanBlog precedent.
- \`metaTitle\` / \`metaDescription\` / \`metaKeywords\` (optional) — passed straight through to the Magento mutation API. If omitted, derive defaults from \`title\` + \`mainKeyword\`.
- \`storeId\` (default \`1\`) and \`pageLayout\` (default \`1column\`) — Magento defaults; only override on explicit instruction.

---

### Pre-flight

- \`CJGEO_API_KEY\` in \`.env.local\` for the \`x-api-key\` header on all CJGEO calls.
- \`BOSTERBIO_BAPI_KEY\` (or the equivalent CMS-mutation handshake env, currently the literal value \`<REDACTED-SHARED-SECRET>\` — see [SB:bosterbio->refCMSMutationAPI]).
- \`OPENAI_API_KEY\` for \`gpt-image-2\` image generation per [SB:cjgeo->refImagePlaceholderHandling].
- Magento CMS mutation module is deployed on bosterbio.com (\`bin/magento module:status Boster_CjgeoPage\` returns "enabled"; \`setup:upgrade\` + \`cache:flush\` were run after enable). If the endpoint returns 401, FIRST verify the literal \`<REDACTED-SHARED-SECRET>\` value is being sent; SECOND verify the module is enabled + upgraded + cache-flushed; THIRD redeploy.

---

### Adoption pipeline

**Step 1 — Fetch raw HTML.**
\`POST /api/articles/{articleId} { action: "get_html" }\` with the \`x-api-key\` header per [SB:cjgeo->refApiActions §1]. Capture \`content_html\`. The shape is the same one BryanBlog consumes.

**Step 1.5 — Pre-sanitize compliance verify (LOCKED 2026-05-21; route hydration live at cjgeo \`22fedb6d\`).** After fetching \`content_html\` via \`POST /api/articles/{id} {action:'get_html'}\`, grep the body for ANY of the Boster template signature classes: \`hero-section\`, \`col-md-4\`, \`dark-overlap\`, \`font-large\`, \`card-item\`, \`row-of-cards\`, \`hero-twin\`, \`font-bold\`. Sum occurrences across all eight. If TOTAL = 0, the draft was generated WITHOUT \`use_custom_template:true\` (or the CJGEO route failed to hydrate templates — should not happen post-\`22fedb6d\`) — abort the publish and trigger a regenerate per [SB:cursorCJGEO->refBosterbioDefault]. Do NOT publish a non-compliant article — it would render the same unstyled fallback that \`cjgeo-mockup-1\`'s prior stale publish produced. The regenerate loop is capped at 3 attempts upstream; after 3, emit \`STATUS: blocked\` with \`BLOCKERS: Boster template classes absent after 3 regenerates\` plus the per-class count breakdown and stop.

**Step 2 — Sanitizer — Preserve CJGEO's inline design, strip only page chrome (LOCKED 2026-05-21).**

CJGEO's default \`generateDraft\` output is INTENDED to be directly usable: it bundles a \`<style data-draft-styles="true">\` block + bespoke component classes that carry CJGEO's design system inline. Magento serves CMS page \`content\` raw inside its theme — so the inline \`<style>\` AND the bespoke classes need to survive sanitization to render properly.

**STRIP (page chrome only — Magento serves inside its own theme, these would collide):**
- \`<!DOCTYPE>\` declaration
- \`<html>\`, \`<head>\`, \`<body>\`, \`<main>\`, \`<article>\` outer wrappers (keep their inner contents)
- \`<script>\` tags (and contents)
- \`<meta>\`, \`<link rel="stylesheet">\`, \`<base>\`, \`<title>\` (Magento prepends title from the CMS page's title field)
- \`<header>\`, \`<footer>\`, \`<nav>\` page chrome
- HTML comments

**PRESERVE (CJGEO design surface — load-bearing for visual rendering):**
- In-body \`<h1>\` — preserve verbatim. Bosterbio's PG page theme does NOT prepend a page-title H1 (verified via CIC DOM inspection 2026-05-21: h1_count=0 on every PG page when in-body H1 was stripped). CJGEO's hero H1 is the page's only H1; sanitizing it out leaves the page H1-less.
- \`<style data-draft-styles="true">\` block — this carries CJGEO's design tokens and component CSS. PRESERVE INLINE.
- All bespoke CJGEO component classes verbatim: \`article-section\`, \`section-number\`, \`section-divider\`, \`qr-item\`, \`qr-item-label\`, \`qr-item-value\`, \`qr-item-detail\`, \`qr-grid\`, \`badge\`, \`badge-green\`, \`badge-yellow\`, \`badge-red\`, \`badge--blue\` (all \`badge-*\` / \`badge--*\` variants), \`callout\`, \`callout-title\`, \`flow-step\`, \`flow-step__num\`, \`flow-step__content\`, \`flow-grid\`, \`figure\`, \`figure-body\`, \`figure-caption\`, \`check-icon\`, \`card\`, \`card-header-red\` (all \`card-header-*\` variants), \`checklist\`, \`table-wrap\`, \`protocol-step__num\`, \`protocol-step__time\`, \`protocol-step__body\`, \`stat-card__num\`, \`stat-card__label\`, \`step__num\`, \`step__body\`, \`bullet-list__icon\`, \`section-label\`, \`text-balance\`, \`container\`, \`why-grid\`, \`why-grid__text\`, \`section\`, \`section--alt\`, and any \`data-draft-*\` attributes. If CJGEO introduces a new class in a future generation, it goes through by default — the rule is preserve-first, only strip if it causes a Magento collision.

**Test the result:** after sanitize, grep the output for \`<style data-draft-styles\` — must be present (count = 1). Grep for one of the bespoke classes (e.g. \`article-section\` or \`callout\`) — must be present. If either grep returns 0, the sanitizer is over-stripping; fix it before publish.

**Drop the BryanBlog 9-pass sanitizer adoption entirely** per G8 lazy adoption — none of those rules apply to a Magento destination where the inline style + classes ARE the design. Adopt BryanBlog rules ONLY if a specific Magento-collision problem surfaces (e.g. a CJGEO class name accidentally matches a Magento utility class and breaks layout — at that point port THAT rule, not the whole pipeline).

**What NOT to strip (Magento-specific):** do NOT inject a \`.prose-article\` wrapper (Magento doesn't use Tailwind), do NOT apply the \`not-prose\` escape hatch (no-op here), no drop-cap CSS opt-out needed, no inline-SVG default sizing override needed (Magento's theme sizes \`<svg>\` consistently). If on visual verify any of these assumptions break, port the BryanBlog rule that fixes it.

**Step 3 — Inject \`id\` on h2s.**

Default mode is slug-from-text (same as BryanBlog satellites): lowercase, replace whitespace with \`-\`, strip punctuation, cap at ~60 chars, de-duplicate collisions with \`-2\` / \`-3\` suffixes. Anchor-map mode is unused on bosterbio CMS pages (no sticky TOC on Magento pages).

**Step 4 — Inject inline deep-dive cross-links between siblings.**

If \`siblingSlugs\` is supplied AND has > 1 entry, inject inline deep-dive links between siblings using the BryanBlog precedent idiom — italic + arrow framing:

\`\`\`html
<p><em>→ Deep dive: <a href="/<sibling-slug>/">Sibling Title</a></em></p>
\`\`\`

Inject one such \`<p>\` per OTHER sibling at the end of the article body (NOT per section — there is no per-section sticky TOC on Magento, so per-section injection bloats without distributed-entry-point payoff). Skip the current page's own slug.

Note: the link href uses the Magento URL convention \`/<slug>/\` (no \`/cms/\` prefix; Magento's URL rewrite resolves \`identifier\` → CMS page). Verify the URL rewrite exists after publish (the CMS mutation module should auto-create it).

**Step 4.5 — Fill image placeholders via \`gpt-image-2\` (LOCKED — MANDATORY).**

Per [SB:cjgeo->refImagePlaceholderHandling]. Both \`<img>\` placeholders (the detector ruleset is in that ref — empty/missing src, \`data:\` stubs > ~4 KB, unreachable hosts, \`via.placeholder.com\` etc., explicit \`generate:\`/\`AI:\`/\`[image: ...]\` alt hints) AND any literal-text diagram placeholders (\`[Diagram: ...]\`, \`[Chart: ...]\`, \`[Visual: ...]\`, \`[Illustration: ...]\`, \`[Figure: ...]\`, \`[Graph: ...]\` — borrowed from BryanBlog's \`adoptCJGEORawContent\` Step 4.5) MUST be replaced with a \`gpt-image-2\`-generated illustration BEFORE the page body is POSTed.

**Storage — proving-grounds media directory on the Magento server (NOT \`public/images/\`, NOT \`pub/media/wysiwyg/\`):** the BryanBlog adoption uses Path A (commit PNGs to \`public/images/<slug>/\`) because bryanblog is a static Next.js site. For bosterbio proving-grounds pages, generated PNGs live on the prod server at \`<REDACTED-HOST>bosterbio.com/html/pub/media/images/proving-grounds/<slug>/<n>.png\`. Magento serves \`pub/media/...\` at the URL \`/media/...\`, so the rewritten \`<img src>\` becomes \`/media/images/proving-grounds/<slug>/<n>.png\`. Two acceptable wire paths:
1. **Preferred (single round-trip):** extend the \`POST /rest/V1/bapi/cms-page\` body with a \`media[]\` array of \`{filename, base64, alt}\` entries. The Magento module decodes each entry, writes it to \`pub/media/images/proving-grounds/<slug>/<filename>\` with mime detection, and resolves to the public URL \`/media/images/proving-grounds/<slug>/<filename>\`. The page body's \`<img src>\` MUST be pre-rewritten by the adopter to the resolved \`/media/images/proving-grounds/<slug>/<filename>\` URL before the body is included in the mutation payload (deterministic — the adopter knows the filename it will pick BEFORE the POST).
2. **Fallback (separate call) — document this path when the CMS mutation module does NOT yet accept \`media[]\`:** upload each PNG via SSH (\`scp\` to \`<REDACTED-HOST>bosterbio.com/html/pub/media/images/proving-grounds/<slug>/<filename>\`) or via a dedicated media sub-route on the CMS-mutation module if one is exposed (\`POST /rest/V1/bapi/cms-page/media\`). After upload, rewrite \`<img src>\` on the in-memory body to \`/media/images/proving-grounds/<slug>/<filename>\`, then POST the body via the main CMS mutation call.

Either path: filename convention is \`<n>.png\` (1-based index across all placeholders in the article, scoped to the slug's own subdirectory; deterministic so re-runs overwrite cleanly). Set \`alt\` to a human-readable subject description, \`loading="lazy"\`, and explicit \`width\` / \`height\` matching the generated size (1536×1024 landscape default per the openai_images weapon).

If image generation fails after 3 retries: log the failure, REMOVE the placeholder entirely (don't ship literal \`[Diagram: ...]\` text to users), and continue. Do NOT block the publish on image fill — the rest of the page is still useful.

**Step 5 — Branding / voice (hand-reviewed, no auto-grep).**

Branding/voice on proving-grounds pages is hand-reviewed by the operator at end-point review; no automated branding-grep on this destination. PG pages are for internal review only; brand voice is controlled manually during review.

**Step 6 — Build the Magento CMS mutation payload.**

The 9-field body shape (full spec in [SB:bosterbio->refCMSMutationAPI]):

\`\`\`json
{
  "handshake": "<REDACTED-SHARED-SECRET>",
  "identifier": "<slug>",
  "title": "<title>",
  "content": "<sanitized + cross-linked + image-filled HTML body>",
  "content_heading": "",
  "meta_title": "<metaTitle | derived from title>",
  "meta_description": "<metaDescription | derived from title + mainKeyword>",
  "meta_keywords": "<metaKeywords | derived from mainKeyword>",
  "is_active": true,
  "store_id": 1,
  "page_layout": "1column"
}
\`\`\`

The \`handshake\` value is LITERALLY \`<REDACTED-SHARED-SECRET>\` (all lowercase, one word, no space, no dash, no hyphen). Anything else returns 401. This is the most common drift cause — agents naturally type \`Mickey Mouse\` or \`<REDACTED-SHARED-SECRET>\`. Hardcode the literal lowercase token; do not template-interpolate from a casing-unstable source.

POST to \`https://www.bosterbio.com/rest/V1/bapi/cms-page\` with \`Content-Type: application/json\`. Idempotent on \`identifier\` — same slug updates the existing page; different slug creates a new one. Re-running this adopter on the same article should be safe (overwrites the previous content with the latest sanitized body).

**Step 7 — Verify via a live GET.**

Per the alignment theorem (per [SB:cjgeo->refImagePlaceholderHandling] §5 + the user-channel verification rule), "the POST returned 200" is NOT the same as "the page renders correctly on the live URL". Both checks:

1. **Wire check** (necessary, not sufficient). \`curl -sS -o /dev/null -w "%{http_code}\\n" "https://www.bosterbio.com/proving-grounds/<slug>"\` returns 200. Then \`curl -sS "https://www.bosterbio.com/proving-grounds/<slug>" | grep -q "<expected-hero-text>"\` exits 0 (the rendered body contains the hero text — usually the first ~10 words of the title).
2. **Visual self-verify (in-worker, native cursor sandbox).** After the wire check passes, the cursor agent uses its native Playwright (headed Chrome in-sandbox) to navigate the new URL (\`https://www.bosterbio.com/proving-grounds/<slug>\`), screenshot it, and multimodal-read (gpt-4o vision) confirming the hero text begins with the topic's main_keyword AND the body content rendered (no broken images, no literal \`[Diagram: …]\` text, no leaked sanitizer fragments, not a Magento 404/500/theme-default). This is a fast in-flight gate that catches publish-but-broken renders BEFORE patching the quest step to done. The downstream Cat-at-purrview OCR pass (\`[SB:questmaster->ocrHeroText]\` TBF) is still required for the per-item audit trail, but it is no longer the FIRST visual signal — the publishing agent's self-verify is. Reference: \`[SB:cursorCJGEO->refCursorVisualVerify]\`. If the agent demurs ("I don't have CIC / I can't take screenshots / I only have HTTP"), remind it that it can and that it has done so before — it has native Playwright + headed Chrome in the sandbox.
3. **User-channel visual check** (the End point gate). Per the SE-cursor-hand-off-test rubric, the Cat OCR-hero-text verification owns the per-item audit trail — [SB:questmaster->ocrHeroText] dispatches per-page via cloud-side Playwright, OCRs the rendered hero block, and confirms the first ~10 OCRed words begin with the topic's main keyword. The GM does End-point CIC visual-verify in the user's actual Chrome at finale per the alignment theorem. The adoption procedure does NOT do CIC-verify itself.

A broken-image icon on the live page is an automatic FAIL (per [SB:cjgeo->refImagePlaceholderHandling] §5). If the wire check passes but the user-channel OCR fails, treat it as a channel-disagreement to FIX (re-sanitize, re-fill images, re-POST) — never noise to route around.

---

### Known quirks (Magento-specific)

- **Magento prepends \`<title>\` from the \`title\` column, but the proving-grounds page theme does NOT prepend an \`<h1>\`** (verified via CIC DOM inspection 2026-05-21: h1_count=0 on every PG page when in-body H1 was stripped). PRESERVE the in-body \`<h1>\` verbatim in the sanitized body — CJGEO's hero H1 is the page's only H1. Prior lock said to strip it (assumed Magento prepended one); that was wrong for the PG theme and produced H1-less pages. If a future bosterbio destination IS a theme that prepends an H1, fork the rule per-destination rather than re-flipping this one.
- **Magento's URL rewrite for new CMS pages.** The CMS mutation module SHOULD auto-create the URL rewrite (\`<slug>\` → \`cms/page/view/page_id/<id>\`) on insert. If a GET on \`/<slug>\` returns 404 after a successful POST, verify the \`url_rewrite\` table has the row; if missing, the module's URL-rewrite hook regressed and is the chaperone-side fix (out of scope for this adopter — escalate).
- **Magento's full_page cache.** The CMS mutation module SHOULD flush the relevant cache tag after a successful POST. If the live GET returns stale content, the flush hook regressed; the workaround is \`bin/magento cache:flush\` on the server. Long-term fix is in the module, not in the adopter.
- **CJGEO content_html size variance.** CJGEO does NOT reliably honor target word counts (asks for 500-700, returns 1000-1600+). The Magento \`content\` column is a TEXT column — no size issue at typical CJGEO sizes. Accept the variance.
- **HTML-entity double-escape bug (carried over from BryanBlog commit \`e0c88a0\`).** Decode \`&mdash;\` / \`&nbsp;\` / \`&hellip;\` / \`&rsquo;\` / \`&amp;\` BEFORE any text-escape pass in the sanitizer. If a live page shows literal \`&mdash;\` text, the decode pass is being skipped — fix the sanitizer, don't string-replace in the body.

---

### BryanBlog precedent — what to defer to

Anything this howTo does NOT address explicitly, default to [SB:BryanBlog->adoptCJGEORawContent] behavior. Examples:
- Anchor injection mode selection (slug-from-text vs anchor-map) — BryanBlog precedent.
- Deep-dive link injection idiom (italic + arrow framing) — BryanBlog precedent, lifted verbatim above.
- Image-placeholder detection rules — [SB:cjgeo->refImagePlaceholderHandling] (which BryanBlog also defers to).
- Diagram-fill prompt template — [SB:BryanBlog->adoptCJGEORawContent] Step 4.5 references the canonical prompt in bryanblog's \`lib/fill-diagrams.mjs\`; the SAME prompt template applies to bosterbio (the style suffix is destination-specific only if the bosterbio brand demands it — currently it doesn't).
- Compliance re-check (product names, return projections, hard CTAs) — only relevant if bosterbio inherits compliance constraints (it does not at the time of this writing); defer to BryanBlog if any compliance constraint appears later.
`,
    },
    refCMSMutationAPI: {
      description:
        "Reference (LOCKED): wire spec for the live Magento CMS mutation endpoint POST /rest/V1/bapi/cms-page on bosterbio.com. 9-field body, locked handshake=<REDACTED-SHARED-SECRET>, idempotency on identifier, production-incident recovery commands. Read before any chaperone-side or Cursor-side call into this endpoint.",
      howTo: `
**The lock.** \`POST /rest/V1/bapi/cms-page\` is the canonical write surface for landing CJGEO-adopted content on bosterbio.com as a Magento CMS page. The endpoint lives in a custom Magento 2 module (\`Boster_CjgeoPage\`) deployed on bitbucket branch \`cjgeo-cms-mutation-module\` at the proven SHA \`668e22e7\`. Every CJGEO → bosterbio publish path (chaperone-side OR Cursor-cloud-agent-side, freestyle OR rubric-driven) goes through THIS endpoint. Do NOT bypass it with raw SQL into \`cms_page\`, do NOT re-implement it with another shim — the cache flush + URL rewrite hooks are tied to the module, and a bypass produces stale pages and 404s.

---

### Endpoint

\`\`\`
POST https://www.bosterbio.com/rest/V1/bapi/cms-page
Content-Type: application/json
\`\`\`

No \`Authorization\` header. No bearer token. No Magento admin token. The handshake is in the JSON body (see below).

---

### Body shape (9 fields)

\`\`\`json
{
  "handshake": "<REDACTED-SHARED-SECRET>",
  "identifier": "<slug>",
  "title": "<page-title>",
  "content": "<HTML body>",
  "content_heading": "",
  "meta_title": "<head <title> override>",
  "meta_description": "<meta description>",
  "meta_keywords": "<meta keywords>",
  "is_active": true,
  "store_id": 1,
  "page_layout": "1column"
}
\`\`\`

Note: the body has TEN named keys above (9 spec fields + the locked \`handshake\` shared-secret key). Some agents count "10 body fields" including \`handshake\`; some count "9" excluding it. The endpoint accepts (and requires) all 10 keys. Either count refers to the same payload — do not let off-by-one wording cause a missing field.

| Field | Type | Required | Notes |
|---|---|---|---|
| \`handshake\` | string | YES | LITERAL value \`<REDACTED-SHARED-SECRET>\` (all-lowercase, one word, no space, no dash). Missing / mismatched → \`401 Unauthorized\`. See "Handshake rule" below. |
| \`identifier\` | string | YES | Magento CMS \`identifier\` column. Also the URL path under the store base URL (\`https://www.bosterbio.com/<identifier>\`). Idempotency key — same identifier updates the existing page; new identifier creates a new page. Slash-namespaced slug format for proving-grounds pages: \`proving-grounds/<slug>\` (e.g. \`proving-grounds/cjgeo-mockup-1\`). Magento CMS identifiers accept the \`/\` character — that's the canonical home for these internal-review pages. |
| \`title\` | string | YES | Magento CMS \`title\` column. Renders as the page \`<title>\` in head. On the proving-grounds page theme, NO \`<h1>\` is prepended in body (verified via CIC DOM 2026-05-21) — PRESERVE the in-body \`<h1>\` from the sanitized content. |
| \`content\` | string | YES | The HTML body. Sanitized + cross-linked + image-filled per [SB:bosterbio->adoptCJGEOIntoMagentoPage]. Magento renders this inside its own theme; do NOT wrap in \`<html>\`/\`<body>\`/\`.prose-article\`. |
| \`content_heading\` | string | NO (empty allowed) | Magento CMS \`content_heading\` column. Usually empty when \`title\` carries the hero — leaving non-empty produces a visible secondary heading above the body. |
| \`meta_title\` | string | NO | \`<title>\` override in head. If omitted, Magento falls back to \`title\`. |
| \`meta_description\` | string | NO | \`<meta name="description">\` value. |
| \`meta_keywords\` | string | NO | \`<meta name="keywords">\` value. Low-impact for SEO but Magento writes it. |
| \`is_active\` | boolean | YES | \`true\` to publish, \`false\` to save-but-hide. Default in adopters: \`true\`. |
| \`store_id\` | integer | YES | Magento store id. Default \`1\` (the canonical bosterbio.com store). Multi-store deployments may differ; do NOT change without owner confirmation. |
| \`page_layout\` | string | NO | Magento CMS page-layout column. Default \`1column\` (canonical for CJGEO mockup pages). Other valid values: \`empty\`, \`1column\`, \`2columns-left\`, \`2columns-right\`, \`3columns\`. \`1column\` is the only theme-tested layout for CJGEO content; deviating risks visual breakage. |

**Idempotency.** \`identifier\` is the upsert key. POSTing twice with the same identifier updates the existing page (creates row if absent, updates row if present). Re-running an adopter is safe — the response's \`is_create\` flag distinguishes which path ran.

**Cache.** On success, the module flushes the relevant full_page cache tag for the affected page so the next GET returns the new content. If the GET returns stale content, the cache flush regressed — recover via \`bin/magento cache:flush\` on the server (see "Recovery commands" below).

**URL rewrite.** On a CREATE (\`is_create=true\`), the module inserts a row in Magento's \`url_rewrite\` table so \`https://www.bosterbio.com/<identifier>\` resolves the new CMS page. On an UPDATE, the existing rewrite is left intact. If a GET on \`/<identifier>\` returns 404 after a successful POST, the URL-rewrite hook regressed — escalate (do NOT raw-SQL the rewrite as a workaround; that masks the module bug).

---

### Handshake rule (LOCKED)

The \`handshake\` parameter value is LITERALLY the lowercase token \`<REDACTED-SHARED-SECRET>\`. The endpoint does a byte-for-byte string equality check:

- \`<REDACTED-SHARED-SECRET>\` → accepted.
- \`Mickey Mouse\` → 401.
- \`<REDACTED-SHARED-SECRET>\` → 401.
- \`mickey-mouse\` → 401.
- \`<REDACTED-SHARED-SECRET>\` → 401.
- \`<REDACTED-SHARED-SECRET> \` (trailing space) → 401.
- Missing \`handshake\` field → 401.
- Wrong type (number / boolean) → 401.

When debugging a 401, the FIRST check is always "is the literal lowercase value \`<REDACTED-SHARED-SECRET>\` being sent in the JSON body". Common drifts: title-cased ("Mickey Mouse"), camel-cased ("<REDACTED-SHARED-SECRET>"), hyphenated, sourced from a casing-unstable env var. Hardcode the literal in the adopter — do NOT template-interpolate from a source that could mutate the casing.

The handshake is NOT a real auth mechanism — it is a shared secret to prevent accidental writes from random crawlers / scanners. Treat the value as low-secret (it ships in chaperone-side code + Cursor agent dispatches); do NOT treat it as a credential to rotate.

---

### Responses

**Success — 200 OK:**

\`\`\`json
[<page_id>, "<identifier>", <is_create>]
\`\`\`

Three-element JSON array (NOT an object, NOT a wrapped \`{success,...}\` shape — REST convention from the Magento module's return type).

- \`page_id\` (integer): the Magento \`cms_page.page_id\` of the upserted page.
- \`identifier\` (string): echoes the input identifier (useful for sanity-checking the upsert landed on the slug you sent).
- \`is_create\` (boolean): \`true\` if a new page was created, \`false\` if an existing page was updated.

Smoke-test response shape: \`[42, "proving-grounds/cjgeo-mockup-1", true]\` (created), then \`[42, "proving-grounds/cjgeo-mockup-1", false]\` (updated on re-POST).

**Failure — 401 Unauthorized:**

Returned on missing or mismatched \`handshake\`. Body is the Magento default 401 error envelope. Recover by fixing the handshake value (see "Handshake rule" above).

**Failure — 400 Bad Request:**

Returned on missing required field (\`identifier\`, \`title\`, \`content\`, \`is_active\`, \`store_id\` are all required by the module) OR on type mismatch (e.g. \`is_active\` sent as a string \`"true"\` instead of boolean \`true\`). Body identifies the missing/bad field.

**Failure — 500 Internal Server Error:**

Returned when the module's DB write or cache-flush call throws. Almost always a deployment regression — recover via the production-incident recovery commands below.

---

### Production-incident recovery commands

The CMS mutation module was deployed via the standard Magento module-enable flow. On a clean deploy:

\`\`\`bash
cd <REDACTED-HOST>bosterbio.com/html
bin/magento module:enable Boster_CjgeoPage
bin/magento setup:upgrade
bin/magento cache:flush
\`\`\`

**Known production-incident hazard:** \`bin/magento module:enable\` CAN clobber the compiled DI / generated static files under some conditions (observed during the initial deploy), producing 500s site-wide until recovered. If that happens, the recovery sequence is:

\`\`\`bash
cd <REDACTED-HOST>bosterbio.com/html
rm -rf generated/* var/cache/*
bin/magento setup:upgrade
bin/magento setup:di:compile
bin/magento setup:static-content:deploy -f en_US
bin/magento cache:flush
\`\`\`

(\`setup:static-content:deploy -f en_US\` is the slow step — 5-10 min on the full theme — but is required to repopulate the static assets the clobber removed.)

Verify recovery: \`curl -sS -o /dev/null -w "%{http_code}\\n" https://www.bosterbio.com/\` returns 200, AND \`bin/magento module:status Boster_CjgeoPage\` returns "enabled". Then re-test the mutation endpoint with a known-good payload.

---

### Proven-deploy reference

- Bitbucket branch: \`cjgeo-cms-mutation-module\`
- Proven-deploy SHA: \`668e22e7\` (initial \`/cjgeo/cms-page\` route); later moved under the \`/bapi/cms-page\` route — appended SHA \`3524ae70\` confirms the bapi-namespaced wire spec.
- Magento module name: \`Boster_CjgeoPage\`
- Module install path: \`app/code/Boster/CjgeoPage/\` under the Magento root \`<REDACTED-HOST>bosterbio.com/html/\`.

When debugging a regression, compare the deployed module's contents against the proven SHA first. If the module differs, redeploy from the proven SHA before further triage.

---

### Adopter-side usage

The expected consumer of this endpoint is [SB:bosterbio->adoptCJGEOIntoMagentoPage] (forge-time companion). Other adopters (Cursor cloud agents per [SB:cursorCJGEO->refReportingContract], chaperone-side ad-hoc publishes) follow the same shape. Do NOT write a one-off shim per consumer — every consumer hits this single endpoint with the 9-field body and the literal \`<REDACTED-SHARED-SECRET>\` handshake.
`,
    },
    get_cms_pages: {
      description: "Live BAPI action: retrieve Magento CMS page records (title, content, meta fields, stores) via direct SQL against cms_page.",
      howTo: `
**Endpoint:** \`GET https://www.bosterbio.com/bapi?action=get_cms_pages&identifiers=<comma-sep>&api_key=<key>\`

Accepts up to **100 identifiers per request**. Either \`?identifiers=<comma-sep>\` or single \`?identifier=\`.

**Preferred call path:** \`bosterbio.comLiveSite.get_cms_pages({ identifiers: ['about-us'], userId })\`.

**Output shape:**
\`\`\`json
{
  "success": true,
  "action": "get_cms_pages",
  "count": 1,
  "pages": {
    "about-us": {
      "page_id": 10,
      "identifier": "about-us",
      "title": "About Us | Boster Bio: ...",
      "content": "<html body string>",
      "content_heading": "",
      "meta_title": "...",
      "meta_description": "...",
      "meta_keywords": "...",
      "is_active": 1,
      "stores": [1],
      "creation_time": "2014-07-16 06:08:18",
      "update_time": "2026-03-12 09:25:05",
      "sort_order": 0,
      "page_layout": "1column"
    }
  },
  "missing": [],
  "errors": []
}
\`\`\`

**Implementation:** direct SQL via the global \`$db\` mysqli — **no Magento bootstrap, no browser scrape**. Primary query against \`cms_page\`; second query against \`cms_page_store(page_id, store_id)\` for store assignment.

**Partial failure:** identifiers with no row land in \`missing[]\`. \`pages\` is JSON-cast to an object so an empty result serializes as \`{}\` not \`[]\`.

**Smoke-tested with:** \`about-us\` (page_id=10, store=[1], content 47499 bytes).

**Future-agent rule:** When you need CMS page content from the live site, **call this BAPI action** — do NOT re-implement raw SQL against \`cms_page\` each time. The retrieval logic lives once, in \`pub/bapi.php\`, and is consumed via the \`bosterbio.comLiveSite\` weapon.
`,
    },
    refBapiEntrypoint: {
      description: "Reference (LOCKED): the live /bapi endpoint on bosterbio.com is served by the Magento theme template api.phtml. Absolute file path: <REDACTED-HOST>bosterbio.com/html/app/design/frontend/Bosterbio/theme/Magento_Theme/templates/api.phtml. Routing chain (LOCKED, operator-corrected 2026-05-22): URL /bapi is a Magento CMS page; that CMS page renders a block; the block's template handle is api.phtml. Because the request goes through Magento's normal CMS page route, Magento IS fully bootstrapped by the time api.phtml runs — there is NO direct pub/index.php → require api.phtml shim (any prior skill-book wording that claimed there was, was wrong). Cart-to-quote, submitCJAntibodyForm2, and any other passToAPI=1 action short-circuits to <REDACTED-HOST>/zohobooksAPI/api.php inside api.phtml before any Magento data is touched. Read before editing /bapi behavior or extending bapi actions.",
      howTo: `
**Canonical /bapi entrypoint (LOCKED 2026-05-22, operator-confirmed):**
- **File (absolute path):** \`<REDACTED-HOST>bosterbio.com/html/app/design/frontend/Bosterbio/theme/Magento_Theme/templates/api.phtml\` on <REDACTED-HOST>. Magento root is \`<REDACTED-HOST>bosterbio.com/html/\`; the theme path beneath it is \`app/design/frontend/Bosterbio/theme/Magento_Theme/templates/\`.
- **Sibling:** \`apiclass.php\` in the same templates folder, required by api.phtml for the non-\`passToAPI\` action branches (uses Magento ObjectManager; Magento is already bootstrapped at this point because the route went through Magento).
- **Routing chain (LOCKED, operator-corrected):**
  1. Client hits URL \`/bapi\` (or \`/bapi/\`, or \`/bapi?action=X\`).
  2. The server config does NOT allow direct PHP rendering at exposed URLs — direct hits to anything ending in \`.php\` at a public URL return 500 by design. So \`/bapi\` does NOT route to any \`pub/*.php\` file directly.
  3. nginx/apache routes \`/bapi\` through Magento like any other front-end URL.
  4. Magento resolves \`/bapi\` as a **CMS page** (via \`url_rewrite\` → \`cms/page/view/page_id/<id>\`).
  5. That CMS page's content renders a Magento **block**.
  6. The block's template handle is \`api.phtml\`, which lives at the absolute path above.
  7. api.phtml runs inside a fully-bootstrapped Magento context, so its \`new api()\` instantiation + \`ObjectManager::getInstance()\` calls succeed normally.
- **Do NOT propagate any narrative that claims \`pub/index.php\` requires api.phtml directly, or that pub/bapi.php's removal repointed a "shim".** That was an unverified leaf hallucination, retracted from this skill book on 2026-05-22 after operator correction.

**Two execution paths inside api.phtml:**
1. **\`passToAPI=1\` short-circuit (early in the file).** Vanilla PHP only — uses \`\$_REQUEST\`, \`\$_SERVER\`, \`\$_FILES\`, \`cjPost()\` (defined at the bottom of the same file). Forwards the entire request to \`https://<REDACTED-HOST>/zohobooksAPI/api.php\`, echoes the upstream response, and \`die()\`s. **No Magento data touched.** This is the path that cart-to-quote, \`submitCJAntibodyForm2\`, and any other Zoho-bound action takes.
2. **Non-\`passToAPI\` actions.** Falls through, \`require_once("apiclass.php")\`, instantiates \`new api()\`, which calls \`\\Magento\\Framework\\App\\ObjectManager::getInstance()\`. **This works** because Magento is already bootstrapped via the CMS-page → block render chain. Any 500s on these actions are bugs INSIDE the action handler (or inside apiclass.php), NOT a Magento-bootstrap problem.

**Cart-to-quote wire (LOCKED, verified 2026-05-22):**
- **Client:** \`app/design/frontend/Bosterbio/theme/Magento_Checkout/templates/cart/methods.phtml:391\` — \`jQuery.post("/bapi", data)\` where \`data\` includes \`action: "cartToQuote"\` + \`passToAPI: 1\` + name/email/phone/items/overview/shipping-address/billing-address.
- **Server hop 1:** \`/bapi\` → pub/index.php → api.phtml — passToAPI short-circuit fires.
- **Server hop 2:** api.phtml POSTs the same payload as form-encoded data to \`https://<REDACTED-HOST>/zohobooksAPI/api.php\` via \`cjPost()\`.
- **Upstream:** bioinvsync's \`productionAPI.php\` action=cartToQuote does the find-or-create Contact + find-or-create Products + create Zoho CRM Quote (\`Quote_Stage:"Finalized"\`, \`EmailOnCreate:true\`) and returns Zoho's response (the SUCCESS payload + the auto-email to the customer).

**Sample successful response shape (truncated):**
\`\`\`json
{
  "data": [{
    "code": "SUCCESS",
    "details": {
      "id": "1286149001439509001",         // Zoho CRM Quote internal id
      "Created_Time": "2026-05-22T11:32:55-07:00",
      "Created_By": { "name": "CJ Xia", "id": "1286149000000073001" }
    },
    "message": "record added",
    "status": "success"
  }]
}
\`\`\`
Direct Zoho Quote URL: \`https://crmplus.zoho.com/bosterbio4/index.do/cxapp/crm/org42596972/tab/Quotes/<id>\`. Org id is \`42596972\`.

**Stale references to \`pub/bapi.php\` in this skill book.** Several other action entries in this same skill book (\`saveProduct\`, \`get_products_by_skus\`, \`get_products_by_urls\`, \`lookupProducts\`, \`get_cms_pages\`) describe their implementation as living in \`pub/bapi.php\`. Per the operator-corrected routing reality above, \`/bapi\` is served by the CMS-page → block → api.phtml chain — there is no \`pub/bapi.php\` involved on the live request path. Those entries are STALE on the file-path claim. The action behavior they document (endpoint URL, request/response shape, fields filter, etc.) may still be accurate as a contract — but anything in those entries that names a file path or claims a specific implementation file should be re-verified with the operator before relying on it. Do not propagate file-path claims from this skill book to other diagnoses without re-confirming against the live server.

**Future-agent rule:** When you need to add a new \`/bapi?action=X\` handler, the destination is the api.phtml file at the absolute path above (or its sibling \`apiclass.php\`). If it needs to forward to Zoho/bioinvsync, set \`passToAPI=1\` from the client and let the short-circuit handle it. If it needs Magento data, add the branch in the non-passToAPI section of api.phtml (or in \`apiclass.php\`) — Magento is already bootstrapped because the route goes CMS page → block → template. Operator collaboration is required before any production edit to api.phtml; this skill book is the reference, not authorization to deploy.

---

## Live-site file discipline (LOCKED — operator-set 2026-05-22)

Two non-negotiable rules for any work that touches the live bosterbio.com Magento server:

### Rule 1 — Action dispatch lives in api.phtml. Reusable helpers live in apiclass.php. NO OTHER new script files on the live server without operator-explicit instruction. (LOCKED — operator-set 2026-05-22, amended 2026-05-26)

**Allowed files (both at the locked absolute path):**
- \`<REDACTED-HOST>bosterbio.com/html/app/design/frontend/Bosterbio/theme/Magento_Theme/templates/api.phtml\` — every \`/bapi?action=X\` handler is a \`case 'X':\` here. **Action dispatch only; no helper functions.**
- \`<REDACTED-HOST>bosterbio.com/html/app/design/frontend/Bosterbio/theme/Magento_Theme/templates/apiclass.php\` — **sanctioned home** for reusable transport + adapter helpers (\`nexus_post\`, \`nexus_log\`, future helpers like \`nexus_get\`, payload adapters, etc.). Anything that's used by 2+ actions OR is structurally "infrastructure rather than business logic" belongs here.

**Forbidden (unless operator gives explicit instruction in the current chat turn):**
- Any NEW \`.php\` file under \`pub/\`, \`scripts/\`, or anywhere else on the live server. The server config does not serve PHP from arbitrary exposed URLs anyway — any new \`pub/*.php\` would return 500 on direct access.
- Re-creating \`pub/bapi.php\` or any sibling script-file dispatcher.
- New Magento controllers / modules to host BAPI actions (unless the operator explicitly authorizes it as a long-term refactor in a separate quest).
- New layout-XML files.
- "Drop a small .php helper next to api.phtml" / "a script file at /pub/utils.php" / any other server-side file create — no, **helpers go INTO apiclass.php**.

**Operator override clause:**
- The operator can explicitly authorize a new file in the chat (e.g. "create a new file at X for Y reason"). Without that, default = extend api.phtml or apiclass.php.

**Why amended (2026-05-26):** the original Rule 1 banned ALL new server-side \`.php\` files. In practice, reusable helpers (transport, adapters, logging) outgrow inlining into api.phtml. Operator clarified that \`apiclass.php\` IS the sanctioned home for these — it keeps api.phtml lean (dispatch only) while concentrating shared logic in one auditable file. The ban now applies to OTHER new files, not apiclass.php.

### Rule 2 — Generated images live ONLY in \`pub/media/images/proving-grounds/<slug>/\`.

- The single allowed image-write path on the live server is \`<REDACTED-HOST>bosterbio.com/html/pub/media/images/proving-grounds/<slug>/<n>.png\` (Magento serves it at the URL \`/media/images/proving-grounds/<slug>/<n>.png\`).
- DO NOT write images to \`pub/media/wysiwyg/\`, \`pub/media/catalog/\`, \`<REDACTED-HOST>.../var/import/\`, the theme \`web/images/\` folders, or anywhere else.
- DO NOT commit generated images into the GuildOS or boster_nexus repos for live-site consumption — they go on the server only. Local mockup PNGs in \`public/proving-grounds/.../\` of the GuildOS repo are FINE because they are local-only (served by the GuildOS Next.js dev server), but if an image is destined for the live site it goes only at the locked server path.
- The adopter pipeline in \`bosterbio.adoptCJGEOIntoMagentoPage\` (Step 4.5, image-placeholder fill) is the canonical path that already honors this rule — its storage section names this exact directory and is the reference for any future image-generation flow on this server.

### Why these rules are locked

- The server's config does not allow direct PHP rendering at exposed URLs, so any new \`.php\` file on the server is dead surface that just adds attack/maintenance burden. Routing has to go through Magento; the only sanctioned Magento-bootstrapped entry point for ad-hoc actions is the CMS-page → block → \`api.phtml\` chain.
- Concentrating every action in one file makes auditing/migration/security review tractable. Scattering API code across multiple PHP files (the prior \`pub/bapi.php\` era + a possible future "lots of small helpers" era) makes the live server a junk drawer.
- Images outside \`pub/media/images/proving-grounds/<slug>/\` end up in directories that Magento's media-cleanup tools rotate or that don't have correct cache headers, breaking rendered pages silently after a deploy.

Future-agent must NOT relitigate these rules. If a task seems to require a new \`.php\` file on the server or an image at a different path, stop and surface the conflict to the operator instead of working around it.

### Rule 3 — Live-server writes require an explicit operator-approved Change Request. Default is read-only. (LOCKED — operator-set 2026-05-26; amended 2026-06-08)

The AI does not write to the live server on its own initiative — past unsupervised writes caused destructive hallucination (wrong file-path narratives, wrong routing models, "fixes" that broke working actions). Writes are therefore **gated, not banned**: the AI may apply a change to \`bosterbio.com\` **only after the operator has explicitly approved a Change Request** (format in Rule 4) **in the current chat**. With no approved Change Request in hand, the AI stays strictly read-only.

**Allowed (read-only):**
- SSH read commands: \`ls\`, \`cat\`, \`head\`, \`tail\`, \`grep\`, \`find\`, \`stat\`, \`wc\`, \`md5sum\`, \`php -l\` (syntax check only).
- Read-only MySQL: \`SELECT\` queries only.
- Read-only HTTP probes: \`curl\` GETs are fine; no \`-X POST\` / \`-X PUT\` / \`-X DELETE\` against live endpoints unless the operator has personally authorized the specific call in the current chat turn.
- Read-only log inspection: \`tail var/log/*.log\`, \`grep\`, etc.
- Read-only \`bin/magento\` introspection: \`bin/magento module:status\`, \`bin/magento config:show\`, \`bin/magento info:*\`. NOT \`cache:flush\`, NOT \`setup:upgrade\`, NOT \`indexer:reindex\`, NOT any state-changing subcommand.

**Gated — each of these is permitted ONLY as part of an operator-approved Change Request (Rule 4); never ad-hoc, never without an explicit per-request approval:**
- Any file write: \`echo > file\`, \`>>\`, \`sed -i\`, \`mv\`, \`cp\` (writing into the server), \`touch\`, \`chmod\`, \`chown\`, \`rm\`, \`ln\`, \`mkdir\`, \`rmdir\`.
- Any text-editor session (\`vi\`, \`vim\`, \`nano\`, \`emacs\`) — even opening one is forbidden because of accidental save.
- Any MySQL write: \`INSERT\`, \`UPDATE\`, \`DELETE\`, \`ALTER\`, \`CREATE\`, \`DROP\`, \`TRUNCATE\`, \`REPLACE\`, \`GRANT\`, \`REVOKE\`.
- Any Magento state-changing CLI: \`bin/magento cache:flush\`, \`setup:upgrade\`, \`setup:di:compile\`, \`indexer:reindex\`, \`module:enable\`, \`module:disable\`, \`config:set\`, \`maintenance:enable\`, etc.
- Any composer/npm/pip/git command that mutates packages or repo state on the server.
- Any service control: \`systemctl\`, \`service\`, \`kill\`, \`pkill\`, \`killall\`, php-fpm reload, nginx reload, mysql restart.
- Any \`scp\`/\`rsync\` INTO the server. (Pulling FROM the server to a local read is fine.)
- Any HTTP write to the live BAPI or any other production endpoint without operator authorization in the current chat turn.

**The write workflow:**
1. **Build a Change Request** with the three required details (Rule 4) and save it under \`docs/proving-grounds/boster-live-site/\`.
2. **Surface it and WAIT for explicit approval** in chat — an unambiguous "approved" / "go" on THIS request. Silence, an emoji, or "looks good" on something else is NOT approval.
3. **On approval, apply exactly the approved bytes:** \`cp -p <file> <file>.bak-<ISO>\` first; apply the delta verbatim from the local source it references (do not improvise new code at apply time); \`php -l\` for PHP; run only the state-changing commands the request names (e.g. \`cache:flush\` + Varnish warm); then run the request's stated measurement to confirm the effect; keep the backup for rollback.
4. **If anything diverges** from what was approved (different file, extra edits, code changed since review), STOP and re-request — approval covers only the reviewed bytes.

### Rule 4 — Change Request format: 3 required details, lives in \`docs/proving-grounds/boster-live-site/\`. Delta-only. (LOCKED — operator-set 2026-05-26; amended 2026-06-08)

When the AI generates code intended to land on the live server (PHP, SQL, config snippet, layout XML, etc.), it goes here for operator review:

\`\`\`
C:\\Users\\xsj70\\GuildOS\\docs\\proving-grounds\\boster-live-site\\
\`\`\`

**Format rules:**
- **One markdown file per proposed change**, named \`<ISO-date>-<short-slug>.md\` (e.g. \`2026-05-26-add-upsertProduct-action.md\`).
- **Delta-only.** Do NOT paste the full target file. The operator reads the delta, applies it to the live file manually, uploads. Format the delta in unified-diff style OR with explicit "ADD THIS BLOCK between line X and line Y" framing — never as "here is the entire api.phtml with my changes mixed in".
- **The markdown file must contain these THREE required details** (the operator approves only after reviewing all three):
  1. **Target file path** — the absolute path of the file to edit on the live server.
  2. **The delta — with line references on BOTH sides.** *Before:* the exact existing code being replaced, identified by its line range on the live file (\`lines X–Y\`), shown verbatim in a code fence. *After:* the replacement code, which must already exist in a **local file referenced by path + line range** (\`<local file> lines A–B\`), shown in a code fence — the applied bytes come from that local source, so what the operator reviews is exactly what gets uploaded (no hand-retyping a different block at apply time). Frame insertions as "ADD between live lines X and X+1", deletions as "REMOVE live lines X–Y"; never paste the whole target file.
  3. **What it does + how to measure it** — 1-3 sentences on the change's effect AND the exact measurement (the specific CIC check / \`curl\` / admin-UI step / metric, with expected before→after values). This doubles as the acceptance test.
- Also include a **rollback** line (e.g. "restore \`<file>.bak-<ISO>\`, then \`cache:flush\`").
- **The proposed code itself (inline comments):**
  - Header comment with the proposing AI's name + ISO timestamp.
  - Pointer to the markdown delta file for context.
- **Never delete or overwrite a prior proposal file** without operator instruction — they accumulate as a review queue.

**Why these rules are locked:**
- The AI has repeatedly proposed file-path narratives that turned out wrong (e.g. the 2026-05-22 / 2026-05-23 chain where a diagnose-leaf invented a "pub/index.php directly requires api.phtml" story the operator had to correct twice). Gating every write behind an explicit per-change review cuts off that class of failure — the operator catches a wrong file-path or wrong delta at review time, before it reaches production.
- Delta-only review means the operator audits each change in seconds rather than reading a full file to find what changed.
- The proving-grounds folder accumulates a paper trail — every proposed change is reviewable post-hoc.

**Future-agent must NOT relitigate the approval gate.** A live-server write requires BOTH (a) a Change Request carrying all three details above, and (b) the operator's explicit per-request approval after review. Never write without both, never apply anything beyond the approved bytes, never skip a detail. If approval isn't given, the change stays a proposal.
`,
    },
    hydrateVendorImagesOnBundleAndSavePDP: {
      description: "Get vendor product images to actually land in Magento media_gallery AND render on the live bundle-and-save PDP. Covers: server-side fallback URL chain (NitroCDN unwrap + WP responsive-size strip), classDev.php placeholder hazards inside the template, and the client-side hero-<img> hydrator that works around the broken Magento gallery init.",
      howTo: `
**End-to-end pipeline for vendor PDP images** (verified 2026-05-28 on bs-biotium-92500):

Source → BAPI \`stage_image_from_url\` (apiclass.php) → Magento \`addImageToMediaGallery\` → \`/media/catalog/product/...\` → \`bundle-and-save-dev.html\` template → external \`bundle-and-save-pdp.js\` hydrator → visible <img> on PDP.

---

**Part 1 — Server-side: stage_image_from_url fallback chain**

Vendor source URLs (especially WordPress-on-NitroCDN like biotium.com) frequently 404 — the NitroCDN edge cache invalidates per-asset and the optimized URL stops resolving. Worse, NitroCDN serves a 404 HTML body with \`Content-Type: image/jpeg\` LYING about MIME — trusting the header gives you a "valid image" of 153 bytes of HTML, which Magento later rejects on first imagick read.

Fix: \`stage_image_from_url\` in \`apiclass.php\` runs an automatic fallback chain. Define \`image_fallback_urls($url)\` returning an ordered list:

1. **NitroCDN unwrap.** Regex \`#^https?://[^/]+\\.nitrocdn\\.com/[^/]+/assets/images/optimized/[^/]+/(.+)$#\` → \`https://\${m[1]}\` (strips proxy prefix to recover the original host path).
2. **WP responsive-size strip.** Regex \`#-\\d+x(?:\\d+)?(\\.[A-Za-z]{3,4})$#\` → \`\$1\` (turns \`Cat_92502_Packaging-336x.jpg\` into \`Cat_92502_Packaging.jpg\`). WP "scaled" / responsive thumbnails frequently 404 too; the unsized original survives.

Then \`http_fetch_image\` walks the candidate list with strict MIME validation:
\`\`\`php
$mime = finfo_buffer(finfo_open(FILEINFO_MIME_TYPE), $bytes);
// Content-Type headers from vendor CDNs LIE. Trust finfo on the actual bytes.
if (!$mime || strpos((string)$mime, 'image/') !== 0) return [null, null];
\`\`\`

Log \`stage_image.fallback_used\` on success and \`stage_image.all_candidates_failed\` on exhaustion — both flow into the in-memory \`bapi_trace\` so the operator can see in the \`triggerPIMTick\` response exactly which fallback rescued each image.

After staging, Imagick re-encodes to a clean sRGB JPEG (strips EXIF/color profiles, caps to 2000px) so Magento's strict media-gallery validation accepts the file.

**The classic failure signature** (pre-fix): \`upsert.image_skipped\` with \`reason: "stage failed (download or unsupported MIME)"\` and \`images_attached: 0\` — every image source was a NitroCDN-optimized URL that 404'd.

---

**Part 2 — Template: classDev.php's \`[X]\` placeholder substitution hazards**

\`bundle-and-save-dev.html\` is processed by \`classDev.php\` before serving. That preprocessor does a blind \`[token]\` regex substitution to expand template placeholders (\`[name]\`, \`[sku]\`, \`[description]\`, \`[order_form]\`, etc.). **Any \`[X]\` substring in template source is corrupted.** Four categories of pain:

- \`var arr = [];\` → \`var arr = ;\` → SyntaxError, entire \`<script>\` tag dies.
- \`obj[key]\` → \`obj\` → wrong reference (silent breakage).
- \`querySelector('input[name="qty"]')\` → \`querySelector('input')\` → wrong element selected.
- \`/[abc]/\` regex char class → mangled → regex fails to match.
- CSS \`.x [attr] { ... }\` → gallery HTML expanded into CSS body → \`</style>\` closed mid-stream; rest of CSS leaks as visible text in body.

**Survival rules for inline JS/CSS in the template:**
- **No array literals.** Use \`Array.of()\` for empty, \`Array.of(a, b, c)\` for populated.
- **No bracket property access.** Use Map (\`m.get(k)\`, \`m.set(k, v)\`, \`m.has(k)\`), or NodeList/Array methods (\`nodes.item(i)\`, \`arr.shift()\`, \`arr.at(i)\`, \`arr.forEach\`).
- **No CSS attribute selectors.** Switch to class selectors. Specifically: do NOT write \`.bs-gallery-col [gallery]\` — the bracketed token gets expanded to the gallery's HTML inside the \`<style>\` block, corrupting everything after.
- **No regex char classes.** Build via \`new RegExp(...)\` from a string, or do char-by-char loops.
- **Avoid bracket syntax even inside comments.** classDev does NOT respect JS comment boundaries — it regex-strips over the raw file text.

**Escape hatch for non-trivial JS: host it externally.**
Put the logic in \`boster_nexus/public/bundle-and-save-pdp.js\` (Vercel-served static asset) and load via a tiny inline loader:
\`\`\`html
<script>
(function(){
  var s = document.createElement('script');
  s.src = 'https://<REDACTED-HOST>/bundle-and-save-pdp.js?v=N&t=' + Date.now();
  s.async = false;
  document.head.appendChild(s);
})();
</script>
\`\`\`
**Why dynamic injection, not a static \`<script src>\`:** Boster's site runs a "defer JavaScript" rewriter that converts \`<script src=...>\` to \`<script defer data-src=...>\` and never loads the script. The rewriter only touches static script tags; dynamically-created ones go through.

---

**Part 3 — Client-side: hero <img> hydrator (Slick gallery init failure workaround)**

Even after images are correctly attached server-side and the page markup contains \`<img src="https://www.bosterbio.com/media/catalog/product/...">\`, the visible PDP gallery stays empty. Magento's bundle-and-save gallery container collapses to \`offsetHeight: 0\` because the Slick gallery JS init never fires on this template (the placeholder elements exist but the slider never wires up).

**Workaround** (in \`bundle-and-save-pdp.js\`'s \`ensureGalleryFallback\`): scan \`.bs-gallery-col img\` for distinct \`/media/catalog/product/\` URLs, build a simple \`<div class="bs-hero-img-host">\` containing a big \`<img>\` (420px max-height, object-fit:contain) plus a thumbnail strip below, insert it as the first child of \`.bs-gallery-col\`, and hide the collapsed native \`.product.media\` wrapper. Filter on the URL prefix to skip skip-link icons / validation badges / SVGs.

**Persistence guard:** Magento's gallery init sometimes runs LATE and wipes the gallery column, taking our hero with it. Register a MutationObserver on \`.bs-gallery-col\` that re-runs the hydrator (up to 8 times) whenever the hero disappears but product imgs are still present.

---

**Part 4 — Deploy + verify checklist**

1. **Deploy apiclass.php:** \`scp -P 2223 apiclass.php.NEW boster_ooP9u@<REDACTED-IP>:.../apiclass.php.new\` → \`php -l\` syntax check → \`mv\` into place → \`curl -H 'Authorization: Bearer <BAPI_KEY>' "https://www.bosterbio.com/bapi?action=opcacheReset"\`.
2. **Deploy template:** \`scp\` to \`Magento_Catalog/templates/product/view/NewSystem2020/productPageTemplates/bundle-and-save-dev.html\`. No opcache reset needed — Magento re-reads templates per request.
3. **Deploy external JS:** edit \`boster_nexus/public/bundle-and-save-pdp.js\`, \`git push origin main\`, wait for Vercel deploy (~30s; poll \`curl -s https://<REDACTED-HOST>/api/sit-rep | grep -oE 'dpl_[A-Za-z0-9]+'\` for new deploy id). Bump \`?v=N\` cache-buster in the template's loader.
4. **Re-enqueue + tick:** \`POST /pim/jobs/api {action:insert, product_id, worker:'liveSite', job_type:'upsert'}\` → \`GET /bapi?action=triggerPIMTick\` (Bearer auth). Look for \`images_attached: 3, images_skipped: 0\` in the result.
5. **CIC-verify** the live URL (\`?v=<bump>\` cache-buster). Real product photo should render in the left gallery column with a thumbnail strip below. The "Vendor product image pending" beige placeholder must NOT appear.

Reference deploy: bs-biotium-92500, 2026-05-29, URL \`https://www.bosterbio.com/mix-n-stain-nanobody-labeling-kits-bs-biotium-92500-boster.html\`. apiclass.php.v15 md5 \`6f25250799aa6c5999b94723fc165fa2\`; bundle-and-save-pdp.js v12 (with MutationObserver retry); bundle-and-save-dev.html with inline dynamic loader pointing at \`?v=12\`.
`,
    },
  },
};
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/figma/index.js -->
```js
/**
 * Figma weapon — Figma REST API connector for reading files, exporting assets,
 * and listing projects.
 *
 * Auth: FIGMA_ACCESS_TOKEN from profiles.env_vars or process.env.
 */

export const toc = {
  readFile: "Read a Figma file's document tree.",
  readNodes: "Read specific nodes from a Figma file by id.",
  readExport: "Export images of Figma nodes (PNG/SVG/etc).",
  searchProjects: "Search Figma projects within a team.",
  searchFiles: "Search Figma files within a project.",
  readComments: "Read comments on a Figma file.",
  readPages: "List all pages in a file (defaults to the Boster master file) — id, name, child count.",
  readPage: "Get the full node tree for a single named page.",
  exportFrames: "Export frames/components as image URLs by node id.",
  readComponents: "List all top-level frames/components across every page of a file.",
};
import { database } from "@/libs/council/database";
import { skillActionOk, skillActionErr } from "../../skill_book/actionResult.js";

const API_BASE = "https://api.figma.com/v1";

/**
 * Primary design file — Boster Design Master File. Default for the agent-facing
 * read* helpers below when no fileKey is given.
 * URL: https://www.figma.com/design/NMfOvoGgMVFPYM4nLtN8zD/Boster-Design-Master-File
 */
const PRIMARY_FILE_KEY = "NMfOvoGgMVFPYM4nLtN8zD";

// ---------------------------------------------------------------------------
// Auth
// ---------------------------------------------------------------------------

async function getToken(userId) {
  if (process.env.FIGMA_ACCESS_TOKEN) return process.env.FIGMA_ACCESS_TOKEN;
  if (!userId) return null;
  const db = await database.init("service");
  const { data } = await db
    .from("profiles")
    .select("env_vars")
    .eq("id", userId)
    .single();
  return data?.env_vars?.FIGMA_ACCESS_TOKEN ?? null;
}

async function figmaFetch(path, opts = {}, userId) {
  const token = await getToken(userId);
  if (!token) throw new Error("Missing FIGMA_ACCESS_TOKEN");
  const url = path.startsWith("http") ? path : `${API_BASE}${path}`;
  const res = await fetch(url, {
    ...opts,
    headers: {
      "X-Figma-Token": token,
      "Content-Type": "application/json",
      ...opts.headers,
    },
  });
  if (!res.ok) {
    const body = await res.text();
    throw new Error(`Figma API ${res.status}: ${body.slice(0, 500)}`);
  }
  return res.json();
}

// ---------------------------------------------------------------------------
// Files
// ---------------------------------------------------------------------------

/**
 * Read a Figma file's metadata and structure.
 * @param {{ fileKey: string, depth?: number }} input
 * @param {string} [userId]
 */
export async function readFile({ fileKey, depth } = {}, userId) {
  if (!fileKey) throw new Error("fileKey is required");
  const params = new URLSearchParams();
  if (depth != null) params.set("depth", String(depth));
  return figmaFetch(`/files/${fileKey}?${params}`, {}, userId);
}

/**
 * Read specific nodes from a Figma file.
 * @param {{ fileKey: string, nodeIds: string[] }} input
 * @param {string} [userId]
 */
export async function readNodes({ fileKey, nodeIds } = {}, userId) {
  if (!fileKey) throw new Error("fileKey is required");
  if (!Array.isArray(nodeIds) || nodeIds.length === 0) throw new Error("nodeIds is required");
  const params = new URLSearchParams({ ids: nodeIds.join(",") });
  return figmaFetch(`/files/${fileKey}/nodes?${params}`, {}, userId);
}

// ---------------------------------------------------------------------------
// Export
// ---------------------------------------------------------------------------

/**
 * Export renderings of nodes as images.
 * @param {{ fileKey: string, nodeIds: string[], format?: string, scale?: number }} input
 * @param {string} [userId]
 * @returns {Promise<{ images: Record<string, string> }>} — node ID to image URL map
 */
export async function readExport({ fileKey, nodeIds, format = "png", scale = 2 } = {}, userId) {
  if (!fileKey) throw new Error("fileKey is required");
  if (!Array.isArray(nodeIds) || nodeIds.length === 0) throw new Error("nodeIds is required");
  const params = new URLSearchParams({
    ids: nodeIds.join(","),
    format,
    scale: String(scale),
  });
  return figmaFetch(`/images/${fileKey}?${params}`, {}, userId);
}

// ---------------------------------------------------------------------------
// Projects & Team
// ---------------------------------------------------------------------------

/**
 * List projects in a team.
 * @param {{ teamId: string }} input
 * @param {string} [userId]
 */
export async function searchProjects({ teamId } = {}, userId) {
  if (!teamId) throw new Error("teamId is required");
  return figmaFetch(`/teams/${teamId}/projects`, {}, userId);
}

/**
 * List files in a project.
 * @param {{ projectId: string }} input
 * @param {string} [userId]
 */
export async function searchFiles({ projectId } = {}, userId) {
  if (!projectId) throw new Error("projectId is required");
  return figmaFetch(`/projects/${projectId}/files`, {}, userId);
}

// ---------------------------------------------------------------------------
// Comments
// ---------------------------------------------------------------------------

/**
 * Read comments on a file.
 * @param {{ fileKey: string }} input
 * @param {string} [userId]
 */
export async function readComments({ fileKey } = {}, userId) {
  if (!fileKey) throw new Error("fileKey is required");
  return figmaFetch(`/files/${fileKey}/comments`, {}, userId);
}

// ---------------------------------------------------------------------------
// Credential check
// ---------------------------------------------------------------------------

export async function checkCredentials(userId) {
  try {
    const token = await getToken(userId);
    if (token) {
      return { ok: true, msg: "FIGMA_ACCESS_TOKEN is set" };
    }
    return {
      ok: false,
      msg: "Missing FIGMA_ACCESS_TOKEN — generate a personal access token at figma.com/developers and add it to profile env_vars.",
    };
  } catch (e) {
    return { ok: false, msg: `Error: ${e.message}` };
  }
}

// ---------------------------------------------------------------------------
// Agent-facing read helpers (skillActionOk/Err envelope). Default fileKey is the
// Boster Design Master File. Driven by the `figma` skill book's howTos.
// ---------------------------------------------------------------------------

export async function readPages(a, b) {
  const [userId, input] = typeof a === "string" ? [a, b ?? {}] : [undefined, a ?? {}];
  const fileKey = input?.fileKey || PRIMARY_FILE_KEY;
  try {
    const data = await readFile({ fileKey, depth: 1 }, userId);
    const pages = (data?.document?.children || []).map((p) => ({
      id: p.id,
      name: p.name,
      childCount: p.children?.length ?? 0,
    }));
    return skillActionOk({ pages, fileKey });
  } catch (e) {
    return skillActionErr(e.message);
  }
}

export async function readPage(a, b) {
  const [userId, input] = typeof a === "string" ? [a, b ?? {}] : [undefined, a ?? {}];
  if (!input?.pageName) return skillActionErr("pageName required");
  const fileKey = input?.fileKey || PRIMARY_FILE_KEY;
  try {
    const meta = await readFile({ fileKey, depth: 1 }, userId);
    const pages = meta?.document?.children || [];
    const page = pages.find((p) => p.name.toLowerCase() === input.pageName.toLowerCase());
    if (!page) {
      const names = pages.map((p) => p.name).join(", ");
      return skillActionErr(`Page "${input.pageName}" not found. Available: ${names}`);
    }
    const nodes = await readNodes({ fileKey, nodeIds: [page.id] }, userId);
    const pageNode = nodes?.nodes?.[page.id]?.document ?? page;
    return skillActionOk({ page: pageNode, fileKey });
  } catch (e) {
    return skillActionErr(e.message);
  }
}

export async function exportFrames(a, b) {
  const [userId, input] = typeof a === "string" ? [a, b ?? {}] : [undefined, a ?? {}];
  if (!Array.isArray(input?.nodeIds) || input.nodeIds.length === 0)
    return skillActionErr("nodeIds (array) required");
  const fileKey = input?.fileKey || PRIMARY_FILE_KEY;
  try {
    const data = await readExport(
      { fileKey, nodeIds: input.nodeIds, format: input.format ?? "png", scale: input.scale ?? 2 },
      userId
    );
    return skillActionOk({ images: data?.images ?? {}, fileKey });
  } catch (e) {
    return skillActionErr(e.message);
  }
}

export async function readComponents(a, b) {
  const [userId, input] = typeof a === "string" ? [a, b ?? {}] : [undefined, a ?? {}];
  const fileKey = input?.fileKey || PRIMARY_FILE_KEY;
  try {
    const data = await readFile({ fileKey, depth: 2 }, userId);
    const components = [];
    for (const page of data?.document?.children || []) {
      for (const node of page.children || []) {
        components.push({ id: node.id, name: node.name, page: page.name, type: node.type });
      }
    }
    return skillActionOk({ components, fileKey });
  } catch (e) {
    return skillActionErr(e.message);
  }
}
```
<!-- /GUILDOS-FILE -->
