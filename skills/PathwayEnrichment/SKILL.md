# PathwayEnrichment

<!-- GUILDOS-SKILL name="PathwayEnrichment" -->

<!-- GUILDOS-DESC -->
Turn a Boster pathway-map page into an interactive lead-gen demo (full-height 50/50 hero + accurate clickable map -> collapsible biomarker/antibody form -> CJGEO description with real figures -> assemble + deploy + CIC-verify). Parameterized over pathwayURL + demoURL. Runs under the GM/execution-leaf split. Read refOrchestration first — the GM/leaf separation is what makes the run actually finish.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/PathwayEnrichment/SKILL.md -->
```markdown
---
name: PathwayEnrichment
description: >-
  Turn a Boster pathway-map page into an interactive lead-gen demo (full-height
  50/50 hero + accurate clickable map -> collapsible biomarker/antibody form ->
  CJGEO description with real figures -> assemble + deploy + CIC-verify).
  Parameterized over pathwayURL + demoURL. Runs under the GM/execution-leaf
  split. Read refOrchestration first — the GM/leaf separation is what makes the
  run actually finish.
---

# Pathway Enrichment

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/PathwayEnrichment/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto PathwayEnrichment <action>` (or read the source directly).

## Actions

- **assembleAndDeployDemo** — Step 4 (params: pathwayURL, demoURL): assemble the page per refPageLayout, deploy to the Boster live Magento CMS via the stdin/base64 CMS write (NOT argv — ENAMETOOLONG), sync storage + quest refs, and CIC-verify the live demoURL renders every component with every invariant byte-exact. Output: the deployed, verified demo page.
- **buildBiomarkerAntibodyForm** — Step 2 (param: pathwayURL): build the v3 quote-builder biomarker panel — 6-col scrollable table (Target | SKU | Reactivity | Application | Size | Check), live pricing (Saved $X + Price $Y/N*size), Add-to-cart + Buy-now CTAs gated at ≥4 selected. Each Target links to its gene-info-card; per-antibody columns come from retrievePicobandAntibodies; CTAs POST to <REDACTED-HOST>/placeholderapi with action addBundleToCart / buyBundleNow per refRun.
- **buildInteractiveMapOverlay** — Step 1 (param: pathwayURL): measure every gene-node box on the native pathway PNG, emit gene_coords_json, build a ZERO-JS percentage-positioned anchor overlay (NOT an HTML image-map + JS scale-sync — that is unstable), and converge via the v2 debug overlay (border + gene-label tag ABOVE the border) and the per-node alignment-eval table (gene · dx · dy · dw · dh · verdict). Output: gene_coords_json + interactive_map_html.
- **enrichDescriptionViaCJGEO** — Step 3 (param: pathwayURL): drive CJGEO Content Magic via CIC to generate a pathway description draft (main keyword = the pathway name), read it back via the non-CIC Supabase path, trim off-page chrome, and replace every [Visual Placeholder] with a real generated figure. Output: cjgeo_enriched_draft + uploaded figures.
- **refOrchestration** — THE load-bearing architecture decision: the conversing GM (orchestrator/judge) is separated from one-shot execution leaves. Read FIRST — this is what makes the run actually finish.
- **refPageLayout** — The locked v4 page layout (colleague-provided design, 2026-05-22): narrow dark hero STRIP at top (NOT a 50/50 full-height hero) → FULL-WIDTH pathway map immediately below the hero (much larger than the v3 right-panel map → coordinates re-need calibration) → v3 quote-builder antibody panel below the map → enriched description broken into the locked named sub-sections (What Is / Key Takeaways / central biological figure / Crosstalk / Cell-type-specific roles / FAQ).
- **refRepeat** — Summary recipe: exactly how to repeat this for another pathway. Hand this (with the two params filled in) to the run.
- **refRun** — Reference: the end-to-end pathway-enrichment plan, the two run parameters (pathwayURL, demoURL), the non-negotiable invariants, and how the actions chain. Read after refOrchestration.
- **retrievePicobandAntibodies** — SSH/SQL PicoBand antibody-retrieval — developed & verified in quest 2bc664f8 step 2, extended 2026-05-17 for the v3 quote-builder. Maps each gene to its preferred Boster PicoBand antibody from the Boster live Magento 2 catalog over SSH: SKU + name + price + size + reactivity + applications. Reusable for ANY gene -> PicoBand antibody task.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/PathwayEnrichment/index.js -->
```js
/**
 * PathwayEnrichment skill book — pure text registry.
 *
 * Reusable, parameterized workflow for turning ONE Boster pathway-map page into
 * an interactive lead-gen demo page: full-height 50/50 hero with an accurate
 * clickable pathway-map overlay -> collapsible biomarker/PicoBand antibody-bundle
 * form -> CJGEO-enriched description with real generated figures -> assembled,
 * deployed, CIC-verified demo page.
 *
 * Parameterized over (pathwayURL, demoURL). Forged from quest
 * 2bc664f8-6588-408f-966b-029e3d11813a ("Build Interactive Hypoxia Pathway Map
 * Demo") and hardened on its post-delivery v2 iteration (the hypoxia page is the
 * canonical exemplar; live at /proving-grounds/pathway-map-enrichment).
 *
 * Read order: refOrchestration -> refRun -> refRepeat -> refPageLayout -> the
 * 4 step actions. Each value is a natural-language howTo prompt; no JS logic
 * here (pure text-prompt registry per blacksmith forge discipline).
 */

export const skillBook = {
  id: "PathwayEnrichment",
  title: "Pathway Enrichment",
  description:
    "Turn a Boster pathway-map page into an interactive lead-gen demo (full-height 50/50 hero + accurate clickable map -> collapsible biomarker/antibody form -> CJGEO description with real figures -> assemble + deploy + CIC-verify). Parameterized over pathwayURL + demoURL. Runs under the GM/execution-leaf split.",
  steps: [],
  toc: {
    refOrchestration: {
      description:
        "THE load-bearing architecture decision: the conversing GM (orchestrator/judge) is separated from one-shot execution leaves. Read FIRST — this is what makes the run actually finish.",
      howTo: `
**The architecture decision (load-bearing — this is why the run completes):**
Separate the conversing **GM (orchestrator/judge)** from the **one-shot execution
leaves**. They never swap roles.

- **The GM is the only stateful, continuity-holding agent.** It NEVER performs the
  noisy execution itself (no OCR, no SSH/deploy, no HTML munging, no image
  generation, no Supabase writes inside the GM context). Doing that work in the
  GM bloats its context, and a bloated GM stalls, loses the thread, and forces
  the user to nudge it forward. A stalled orchestrator is the failure mode this
  split exists to remove.
- **Each heavy phase is dispatched to a fresh one-shot leaf** with a fully
  self-contained brief (the leaf has zero memory of the conversation — the brief
  must carry every path, URL, credential-env name, invariant, and the proven
  deploy/verify recipe). The leaf absorbs the noise (hundreds of tool calls,
  large files, retries) and returns only clean structured signal.
- **The GM owns ONLY:** (1) briefing leaves; (2) trust-but-verify of every leaf
  claim against ground truth — never accept "done" on the leaf's word;
  (3) CIC visual verification *the user's way* (the user's real browser, which
  passes the Cloudflare WAF — curl 403s and is NOT authoritative); (4) the
  iterative visual-alignment loop (diagnose the residual error from the CIC
  debug overlay, hand the leaf a precise correction, re-verify); (5) judging /
  scoring; (6) **continuity — driving the run end-to-end without the user
  nudging it.** Continuity is the orchestrator's job, never the user's.
- **One concern per leaf, sequential when they touch the same artifact.** Leaves
  editing the same assembled HTML must run one-at-a-time (parallel leaves clobber
  the file). Independent concerns (e.g. two different weapon-bug fixes in
  different files) may run in parallel.
- **A leaf's summary describes intent, not reality.** Always independently verify:
  file exists + has the change, storage byte size changed, the live page renders
  it via CIC, commits landed. A prior leaf in the canonical run falsely claimed
  storage was synced; trust-but-verify caught it.

**Why this guarantees completion:** the GM's context stays small and stable, so
it can dispatch-and-verify across an unbounded number of cycles (build, fix,
re-fix, re-verify) and self-pace (ScheduleWakeup) through long/unattended runs
without ever needing the user to push it. The leaves are disposable; the GM is
the spine.
`.trim(),
    },

    refRun: {
      description:
        "Reference: the end-to-end pathway-enrichment plan, the two run parameters (pathwayURL, demoURL), the non-negotiable invariants, and how the actions chain. Read after refOrchestration.",
      howTo: `
The workflow turns ONE Boster pathway-map page into one interactive lead-gen
demo page. Two run parameters:

- **pathwayURL** — the source Boster pathway-map page (e.g.
  \`https://www.bosterbio.com/pathway-maps/neuroscience/alzheimers-disease-pathway\`).
  The pathway image, the gene list, and the pathway NAME are all DERIVED from it.
- **demoURL** — where the finished demo page is deployed (e.g.
  \`https://www.bosterbio.com/proving-grounds/pathway-map-enrichment-2-alzheimers\`).
  Its path becomes the Magento \`cms_page\` identifier.

Run the whole book again with a different (pathwayURL, demoURL) for a different
pathway — see \`refRepeat\`.

**The 4 steps (each its own action), chained:**
1. \`buildInteractiveMapOverlay\` — pathwayURL → gene_coords_json + interactive_map_html
   (zero-JS percentage-anchor overlay; see the action for the locked technique).
2. \`buildBiomarkerAntibodyForm\` — gene list from step 1 → biomarker_table_form_html
   (calls \`retrievePicobandAntibodies\` for the recommended-antibody column).
3. \`enrichDescriptionViaCJGEO\` — the pathway name → cjgeo_enriched_draft.
4. \`assembleAndDeployDemo\` — outputs of 1–3, laid out per \`refPageLayout\` →
   the deployed, CIC-verified demo page at demoURL.

**Non-negotiable invariants (verbatim — do NOT paraphrase or "improve"):**
- Each gene box links to \`https://www.bosterbio.com/bosterbio-gene-info-cards/[gene_name]\`.
- The biomarker/antibody panel is the **v3 quote-builder UI** (validated 2026-05-17,
  supersedes the v2 email-quote/promo-banner design): a 6-column table
  **Target | SKU | Reactivity | Application | Size | Check** inside a fixed-height
  internally-scrollable container (~420px, \`overflow-y:auto\`, no external "View all"
  expander). Below the table: green-check "In stock items Ship Next Business Day. For Europe,
  ships in: 2-3 weeks." Side/footer: "Saved $<S>" + "Price: $<B>/<N>*<avg-size>"
  (live-updated from the row checkboxes). Two CTAs: outlined "Add to cart" and
  filled "Buy now". Initial state: all checkboxes UNCHECKED on render.
- **Select gate:** both CTAs disabled until **≥4** targets are checked (no email
  required — the v3 design moves email collection to the cart/checkout step).
- **CTA wiring:** both CTAs POST JSON to \`https://<REDACTED-HOST>/placeholderapi\`
  with the action literal — \`action = addBundleToCart\` for Add to cart,
  \`action = buyBundleNow\` for Buy now — and a payload \`{ pathwayURL, selections:
  [{target, sku, size, qty:1}], bundle: {price, savings, listTotal} }\`. Endpoint +
  the two action strings are fixed and byte-exact.
- **Pricing rule (live-updated on each toggle):** \`listTotal\` = sum of per-antibody
  list prices for currently-selected rows (from Magento \`price\` attribute via
  \`retrievePicobandAntibodies\`); \`price\` (bundle) = \`listTotal * 0.5\` rounded;
  \`savings\` = \`listTotal - price\`. Format with the average pack size shown as
  \`/<N>*<avg-size>\` (e.g. \`/4*10ug\`).
- PicoBand match is a CASE-INSENSITIVE SUBSTRING on the antibody NAME (not the SKU).
- A gene with no "picoband"-named antibody gets a NULL antibody field — never substitute
  from a different family — and that row's checkbox is disabled (cannot be selected).

**Page layout:** v4 (locked 2026-05-22 from the colleague's reference design) —
narrow dark hero strip at top → full-width interactive map below the hero
(the map is the focal upper-page element, larger than the v3 right-panel
map) → v3 quote-builder panel below the map → enriched description in the
locked named sub-sections (What Is / Key Takeaways / central biological
figure / Crosstalk / Cell-type-specific roles / FAQ). See \`refPageLayout\`.

**Map-alignment loop:** convergence is gated by the per-node alignment-eval
table emitted by \`buildInteractiveMapOverlay\` (gene · dx · dy · dw · dh ·
label-match · verdict). All rows must read \`pass\` BEFORE the GM CIC-verifies
on the live page. The CIC verify uses debug overlay v2 (border + gene-label
tag ABOVE the box) — the GM grades both box-on-node AND that the label tag
matches the gene text baked into the PNG inside the box.

**Discipline:** verify on real artifacts, not proxies (SE rule #8). The ONLY
authoritative visual check is CIC on the live demoURL in the user's real browser.
`.trim(),
    },

    refRepeat: {
      description:
        "Summary recipe: exactly how to repeat this for another pathway. Hand this (with the two params filled in) to the run.",
      howTo: `
To produce another pathway demo, the GM runs this book end-to-end under the
\`refOrchestration\` split with two params filled in. Recipe:

**0. Params.** pathwayURL = the source Boster pathway-map page; demoURL = the
target \`/proving-grounds/<slug>\` page. Derive: pathwayName (from the pathway-map
page title), pathwayImagePNG (the collateral PNG that page embeds — its native
WxH is the coordinate space), demoSlug (demoURL path).

**1. Quest.** Create a quest (assignee Guildmaster, stage \`execute\`) whose 4
steps mirror \`refRun\`'s chain, each with clear input/method/output and the
invariants flagged \`[IMPORTANT DETAIL]\` in step.method. Items: the source page
link + the pathway PNG.

**2. Execute via leaves (GM dispatches, never executes):**
  - Leaf A → \`buildInteractiveMapOverlay\`: fetch the native PNG, measure every
    gene-node box in native px, emit gene_coords_json, build the zero-JS
    %-anchor overlay, run the offline composite self-check loop, hand back.
  - Leaf B → \`buildBiomarkerAntibodyForm\` (+ \`retrievePicobandAntibodies\` via
    SSH/SQL): the collapsible biomarker/antibody panel.
  - Leaf C → \`enrichDescriptionViaCJGEO\`: CJGEO draft for pathwayName, trimmed;
    generate + SSH-upload the figures for every \`[Visual Placeholder]\`.
  - Leaf D → \`assembleAndDeployDemo\`: lay out per \`refPageLayout\`, deploy to
    demoURL via the stdin/base64 CMS write, sync storage + quest refs.

**3. GM verifies the user's way (every cycle):** CIC-open demoURL; confirm the
v4 layout (narrow dark hero strip; full-width pathway map immediately below;
v3 quote-builder panel below the map — 6-col table
Target/SKU/Reactivity/Application/Size/Check inside a ~420px scrollable region;
pricing block "Saved $X" + "Price: $Y/N*size" live-updates as boxes are checked;
both Add-to-cart and Buy-now disabled until ≥4 selected, enabled at 4; the
enriched-description sub-sections in the locked order: What Is, Key Takeaways
3-card row, central biological figure, Crosstalk, Cell-type-specific roles,
FAQ); real figures render; and — with \`?mapdebug=1\` — every gene box tightly
on its node across the WHOLE map AND every label-above-the-box tag matches
the gene text baked into the PNG underneath. Confirm UTF-8 (α/β/κ) survived
the deploy round-trip.

**4. Iterate the mask until aligned.** This is expected to take a few GM↔leaf
cycles. The leaf attaches an alignment-eval table (per-node dx/dy/dw/dh +
label-match + verdict) with every overlay revision; the GM REJECTS any
revision whose table isn't all-pass before even CIC-verifying. On a residual
visual mismatch caught by CIC, the GM diagnoses the offset from the v2 debug
overlay (which gene · which axis · how many px), hands the leaf a precise
correction, re-grades the table, re-verifies on CIC. Do not declare done on
"close".

**5. Finale.** When CIC confirms every invariant + alignment + layout on BOTH
existing demo pages (hypoxia and Alzheimer's — re-confirm any sibling demo
under v4 isn't regressed by this run), pull both finished demoURLs up in the
user's browser in \`?mapdebug=1\` mode and report. Continuity throughout is the
GM's job — never ask the user to push the run forward.

The hypoxia run is the worked reference end-to-end (canonical truth) and is
re-converged under v4 layout when the alzheimers run lands (the two pages
share \`refPageLayout\`).
`.trim(),
    },

    refPageLayout: {
      description:
        "The locked v4 page layout (colleague-provided design, 2026-05-22): narrow dark hero STRIP at top (NOT a 50/50 full-height hero) → FULL-WIDTH pathway map immediately below the hero (much larger than the v3 right-panel map → coordinates re-need calibration) → v3 quote-builder antibody panel below the map → enriched description broken into the locked named sub-sections (What Is / Key Takeaways / central biological figure / Crosstalk / Cell-type-specific roles / FAQ).",
      howTo: `
The assembled demo page MUST follow this layout (v4, validated 2026-05-22 from
the colleague's reference design — supersedes the v3 50/50 hero; do not regress
to either v2 single-column-stack OR v3 50/50 hero):

**1. Hero — narrow dark STRIP at the top (NOT full-viewport-height).**
- Hero wrapper: dark navy/blue solid or subtle-gradient background, light text,
  full-bleed width, fixed compact height ~\`160–220px\` (NOT \`min-height:100vh\`).
- Centered content stack: pathway title (h1, large), one-line intro/subtitle
  (small caps or muted), a single primary CTA button "Build your antibody
  bundle ↓" that jumps to the antibody panel. No image lives in the hero — the
  map is its own section below.
- Responsive: same compact strip; CTA wraps below title on narrow screens.

**2. Interactive pathway map — FULL-WIDTH section IMMEDIATELY below the hero.**
- Wrapper \`<section id="pathway-map">\` with light background; the
  \`<div class="hypoxia-imap" style="position:relative;width:100%;max-width:<NATW>px;
  margin:0 auto">\` sits inside, centered, at its native max width with
  \`width:100%; height:auto\` on the inner \`<img>\`. THE MAP IS THE FOCAL ELEMENT
  OF THE UPPER PAGE — significantly larger than the v3 right-panel map.
- This larger render size is exactly why the v4 alignment loop must re-converge
  per-node from scratch (do not assume v3 coordinates port over; the rendered
  pixel error tolerance shrinks as the map grows). See
  \`buildInteractiveMapOverlay\` for the per-node alignment-eval-table loop.
- A small caption above the map: "Interactive Pathway Map — click a gene".

**3. Biomarker / PicoBand antibody panel — IMMEDIATELY below the map**
(before the enriched description). **v3 quote-builder design retained**:
- Panel heading "Biomarker Roles & PicoBand® Antibody Bundle".
- The biomarker table has 6 columns: **Target | SKU | Reactivity | Application
  | Size | Check** (header for column 6 is \`Check\` with the small instruction
  \`Please select at least 4 targets.\` underneath). \`Target\` and \`SKU\` cells
  are orange-on-light (Boster brand); \`Reactivity\`/\`Application\`/\`Size\` cells
  are plain comma-joined text from Magento EAV. \`Target\` links to the gene-info
  card; \`SKU\` links to the antibody product page when known.
- The table is wrapped in a fixed-height region \`max-height:420px; overflow-y:auto\`
  with a styled scrollbar so it is visually obvious there is much more below.
  There is NO "View all" expander — the whole panel is one fixed-height area
  with a scrollable table; pricing + CTAs sit below the scrollable region.
- Rows whose antibody is null are rendered with the cell text "No PicoBand®
  antibody for this target — null" and the checkbox is \`disabled\`.
- Below the table (centered): green-check "In stock items Ship Next Business Day. For Europe,
  ships in: 2-3 weeks."
- Live pricing block (right side or stacked below the ships notice): a smaller
  "Saved $<S>" line above a larger "Price: $<B>/<N>*<avg-size>" line.
  Re-computes on every checkbox toggle per the pricing rule in \`refRun\`.
- Two CTAs centered below pricing: outlined "Add to cart" (with 🛒) and filled
  "Buy now ›". Both disabled until ≥4 selected. POST per the wiring rule in
  \`refRun\` (\`addBundleToCart\` / \`buyBundleNow\`).
- Initial state on render: all checkboxes UNCHECKED, both CTAs disabled,
  pricing block shows \`Saved $0\` / \`Price: $0/0\`.

**4. Enriched description — below the antibody panel, broken into the locked
named sub-sections.** Step 3's trimmed \`cjgeo_enriched_draft\`, structured as:
- \`<h2>What is the <PathwayName>?</h2>\` — opening explainer + a small inline
  figure (the "neurobiology basics" / cellular-context diagram).
- \`<h2>Key Takeaways at a Glance</h2>\` — a 3-card row (icon + ~25-word
  takeaway each). The cards stack on narrow screens.
- \`<h2>The <CoreProtein/SubPathway> Pathway in <PathwayName></h2>\` — a focused
  section with the CENTRAL BIOLOGICAL FIGURE inline (e.g. for Alzheimer's, the
  Tau-protein pathway figure; for hypoxia, the HIF stabilization figure).
- \`<h2>Pathway Crosstalk: Integrating <PathwayName> Signaling Into Your Maps</h2>\`
  — crosstalk section, often with a hub-and-spoke diagram and 2-col bullet
  list of upstream/downstream contacts.
- \`<h2>Cell-type-specific Roles in the <PathwayName></h2>\` — a numbered
  circular-bullet list of 4-6 cell types with role descriptions.
- \`<h2>Frequently Asked Questions</h2>\` — bottom FAQ list (Q in bold, A
  expanded inline — NOT collapsed/accordion-only).
- For sub-sections the CJGEO draft did not directly produce, derive the header
  from the draft's existing structure rather than inserting empty scaffolding;
  the layout target above is the goal, but the leaf MUST preserve what CJGEO
  actually wrote and only re-shape headers + nesting. NEVER invent content.

**Real figures for placeholders:** for each \`[Visual Placeholder]\` in the draft,
read its caption, generate a scientifically-appropriate diagram via the ChatGPT
image-generation skill book (discover it in the registry — \`libs/skill_book/index.js\`),
SSH-upload the PNGs to the live server path \`pub/media/images/pathway-pages/\`
(public URL \`https://www.bosterbio.com/media/images/pathway-pages/<file>\`), and
swap the placeholder for the figure. The Key Takeaways 3-card row and the FAQ
section may NOT need image figures — replace only the actual \`[Visual
Placeholder]\` markers; do not fabricate figures into sections that didn't ask
for them.
`.trim(),
    },

    buildInteractiveMapOverlay: {
      description:
        "Step 1 (param: pathwayURL): measure every gene-node box on the native pathway PNG, emit gene_coords_json, build a ZERO-JS percentage-positioned anchor overlay (NOT an HTML image-map + JS scale-sync — that is unstable), and converge via the v2 debug overlay (border + gene-label tag ABOVE the border) and the per-node alignment-eval table (gene · dx · dy · dw · dh · verdict). Output: gene_coords_json + interactive_map_html.",
      howTo: `
**Input:** \`pathwayURL\` → the native pathway PNG it embeds (e.g. the
\`...-Pathway.png\` collateral). Download it; its natural WxH is the coordinate
space (read it from the PNG header — e.g. hypoxia is 1296×1728; the v4
Alzheimer's collateral is its own native size — read from the PNG header,
do not assume v3 hypoxia dimensions).

**Why this loop needs to converge harder under v4 layout:** in v4 the map is
the focal full-width section under a narrow hero (no longer the right half of
a 50/50 hero), so the rendered map is significantly larger and the per-pixel
visual tolerance is correspondingly tighter — a drift that was invisible at
the smaller v3 size will now read as obviously off-node. Re-converge from
scratch; do not reuse v3 coordinates.

**Truth-box definition (LOCKED — same essence that made the hypoxia run look
right; locked 2026-05-23 after an alzheimer's re-run drifted because this was
implicit).** A gene node on a pathway PNG has three plausible bounding
rectangles — pick the wrong one and "tight enough" lands in a different
visual place per run, which is exactly the divergence to prevent:

  - (a) the **text-label sub-region** alone (just the gene name characters)
  - (b) the **colored shape** the gene is drawn in (capsule / oval / hexagon
    / rounded rectangle), INCLUDING its stroke outline
  - (c) the entire **node cluster + adjacent labels/decorations** around the
    gene (too loose — never this one)

The truth box is **(b) — the full colored shape including its stroke
outline** for EVERY gene-node anchor. Verbatim rules:

  1. For a gene drawn in an oval/capsule (e.g. \`TLR\` in the orange capsule
     on hypoxia, \`IL-1\`/\`TNF\`/\`NMDAR\` in the elongated membrane-receptor
     capsules on alzheimer's, \`SNCA\` in an oval on alzheimer's): the truth
     box is the tight bounding rectangle of the WHOLE capsule/oval — left
     edge of the stroke to right edge of the stroke, top of the stroke to
     bottom of the stroke. NOT the inner text-only sub-region. NOT the
     text-and-a-bit-of-padding sub-region. The whole shape including the
     stroke. Even when the colored fill is mostly empty whitespace below
     the label (long vertical membrane receptors), the truth box covers
     the entire shape so the mask is clickable across the whole node.
  2. For a gene drawn in a hexagon (e.g. \`Aβ Accumulation\`, \`Aggr. Amyloid\`
     on alzheimer's): the truth box is the axis-aligned bounding rectangle
     of the hexagon's outer stroke. The corners of the box will fall
     outside the hexagon's slanted edges — that is correct; the rectangle
     bounds the shape, not vice versa.
  3. For a gene drawn in a rounded-rectangle "box" (e.g. the small orange
     receptor boxes \`TLR\`/\`IKK\`/\`NF-kB\` on hypoxia, \`SNCA\` orange box on
     alzheimer's): the truth box is the rounded-rectangle's bounding
     rectangle including the stroke.
  4. For a gene that has BOTH a colored shape AND an outside label text
     (e.g. \`NEP\` on alzheimer's where "NEP" sits in an orange box ABOVE
     a smaller oval): the truth box is the colored SHAPE the gene name
     sits inside — usually the higher, more prominent one. If the diagram
     repeats the same gene's text in multiple visually distinct shapes,
     each shape gets its OWN anchor (per the multi-occurrence rule below);
     do not collapse them.
  5. When the colored shape's stroke is partly cut by an arrow or overlay,
     extend the truth box to where the stroke WOULD continue if uncut —
     i.e. measure the inferred full shape, not the visible-only fragment.

This single rule is what aligns the alzheimer's run visually with the
hypoxia run. Both maps under rule (b) produce masks that cover the whole
clickable gene node, not just its text — and the visual perception of
"tight" stabilizes because both runs are now measuring the same thing.

**The overlay TECHNIQUE is locked (hard-won on the hypoxia v2 run):**
- **DO NOT** use an HTML image-map (\`<img usemap>\` + \`<map>/<area>\`) with a JS
  scale-sync script. It has a horizontal mapping bug under responsive scaling
  (left-edge nodes drift ~100px+ while center nodes look fine) and is the exact
  instability that wasted multiple cycles. Forbidden.
- **DO** use **zero-JS percentage-positioned anchors over a position:relative
  wrapper**:
  \`<div class="hypoxia-imap" style="position:relative;width:100%;max-width:<NATW>px;margin:0 auto">\`
  containing \`<img style="display:block;width:100%;height:auto" src="<PNG>">\`
  then one \`<a class="gx" data-gene="<GENE>" href="https://www.bosterbio.com/bosterbio-gene-info-cards/[gene]"
  target="_blank" style="position:absolute;left:L%;top:T%;width:W%;height:H%">\`
  per node, where, for a measured native box \`(x1,y1,x2,y2)\`:
  L = x1/NATW*100, T = y1/NATH*100, W = (x2-x1)/NATW*100, H = (y2-y1)/NATH*100.
  Percentages are relative to the same rendered \`<img>\` box → perfect scaling at
  any size, zero JS, no offset possible. The \`data-gene\` attribute MUST be set
  to the gene's display label (the text baked into the PNG at that node) — the
  v2 debug overlay reads it and renders it as a tag so the GM can verify that
  the mask's label matches the PNG text directly underneath.

**Debug overlay v2 — border AND label-above-border (locked 2026-05-22):** the
v1 debug overlay only outlined the box, so a slightly-off mask still looked
acceptable until the GM zoomed in. v2 also renders the gene label as a small
high-contrast tag ANCHORED to the OUTSIDE TOP edge of the box (NOT inside the
box — labels inside the box visually overlap the gene name baked into the PNG
and defeat the verification). Spec:

\`\`\`html
<script>(function(){if(/[?&]mapdebug=1/.test(location.search))document.body.classList.add('mapdebug')})()</script>
<style>
  body.mapdebug .hypoxia-imap .gx{outline:2px solid #ff7a00;background:rgba(255,140,0,.18)}
  body.mapdebug .hypoxia-imap .gx::before{
    content: attr(data-gene);
    position: absolute;
    left: 0;
    bottom: 100%;            /* sits ABOVE the box's top edge */
    transform: translateY(-2px);
    background: #ff7a00;
    color: #fff;
    font: 600 11px/1 system-ui, sans-serif;
    padding: 2px 5px;
    border-radius: 3px;
    white-space: nowrap;
    pointer-events: none;
    box-shadow: 0 1px 2px rgba(0,0,0,.35);
  }
</style>
\`\`\`

The label tag sitting above the box gives the GM a side-by-side: the orange
tag text (what the anchor THINKS the node is) vs the gene text inside the box
baked into the PNG (the truth). A mismatch = the anchor is over the wrong
node, even if the box geometry looks OK. Anchors stay transparent without
\`?mapdebug=1\`.

**Method:**
1. View the native PNG (multimodal, zoom/tile). For EACH gene node, measure a
   tight bounding box in native px. Genes that appear at multiple node locations
   get one anchor PER location (e.g. HIF-α appears 4× on the hypoxia map).
   Keep the gene→href mapping (gene-card slug, e.g. TLR→TLR4, IL-6R→IL6R); the
   PNG-visible label (used for \`data-gene\`) may differ from the slug — keep
   both ( \`{slug, label, boxes:[{x,y,w,h}]}\` ).
2. Emit \`gene_coords_json\` ( \`{gene:{slug,label,boxes:[{x,y,w,h},...]}}\` native
   px ) and the %-anchor \`interactive_map_html\` per the locked technique, with
   \`data-gene\` populated from \`label\` for every anchor.
3. **MANDATORY offline composite self-check (do NOT skip; do NOT deploy before
   it passes).** Programmatically draw every anchor's rectangle + its gene
   label tag (above the box, matching the v2 debug overlay) onto a copy of the
   native PNG; view that composite; for any box not tightly on its labeled node
   OR whose label tag points to a different node label than the one underneath,
   fix the coords and re-composite. Iterate this cheap offline loop until the
   alignment-eval table (next step) passes.
4. **Per-node alignment-eval table (MANDATORY — the iteration gate before any
   CIC deploy/verify).** For every anchor, compute the residual:
     - \`truth\` box = the bounding box of **the gene's full colored shape
       including its stroke outline** on the native PNG, per the truth-box
       definition above (rule b). NOT the inner text-label sub-region. NOT
       the visible-only fragment when the shape is cut by an arrow. Re-measure
       the PNG visually; do not trust the prior coords as truth — the prior
       coords are what is being graded.
     - \`anchor\` box = \`gene_coords_json[gene].boxes[k]\` (in native px).
     - \`dx = anchor.x - truth.x\`  (px; sign = direction of drift)
     - \`dy = anchor.y - truth.y\`
     - \`dw = anchor.w - truth.w\`
     - \`dh = anchor.h - truth.h\`
     - \`verdict\` = \`pass\` iff \`|dx|, |dy| ≤ 6 px\` AND \`|dw|, |dh| ≤ 8 px\` AND
       the underlying PNG label at the anchor box matches \`gene.label\`;
       otherwise \`drift\`. (Thresholds are native-px on the source PNG, NOT
       rendered-px on the live page.)
   Emit the table as Markdown — one row per anchor occurrence — in this column
   order with this header verbatim (so downstream eyeballing + diffs are
   stable):

   \`\`\`
   | gene  | occ | truth(x,y,w,h)        | anchor(x,y,w,h)       | dx | dy | dw | dh | label-match | verdict |
   |-------|-----|-----------------------|-----------------------|----|----|----|----|-------------|---------|
   | HIF1A | 1   | 612, 188, 64, 28      | 615, 192, 66, 30      | +3 | +4 | +2 | +2 | yes         | pass    |
   | TLR4  | 1   | 88, 412, 80, 28       | 142, 410, 80, 28      | +54| -2 | 0  | 0  | yes         | drift   |
   ...
   \`\`\`

   Persist the table as \`alignment_eval_table_md\` (kind=markdown) on the step
   so the GM can read it without round-tripping the leaf. Sort drift rows to
   the top, biggest \`max(|dx|,|dy|,|dw|,|dh|)\` first — the leaf MUST fix the
   worst-offender row, regenerate the overlay, and re-emit the table. The loop
   exits only when every row reads \`pass\`. A leaf that returns "looks aligned"
   without an attached alignment-eval table is a discipline violation
   (rubber-stamping; the GM rejects the result and re-dispatches).
5. The GM then CIC-confirms on the live page with \`?mapdebug=1\` — sees the v2
   border + label-above-border for every anchor — and grades two things
   visually: (a) every box sits tightly on its labeled node; (b) every orange
   tag above-the-box reads the SAME gene label as the text baked into the PNG
   inside the box. Any mismatch is handed back as a precise correction
   (specific gene + which direction off + by how many px on which axis) for
   another offline pass + a refreshed alignment-eval table. Membrane-receptor
   / left-column nodes are the historical worst offenders — scrutinize them
   specifically.

**Outputs (persist as quest items / step outputs):**
- \`gene_coords_json\` (kind=json) — per-gene native-px boxes + slug + label.
- \`interactive_map_html\` (kind=html) — the zero-JS %-anchor overlay with the
  v2 debug-overlay CSS inlined and \`data-gene\` on every anchor.
- \`alignment_eval_table_md\` (kind=markdown) — the final all-pass per-node
  alignment-eval table from the converged offline loop.
`.trim(),
    },

    retrievePicobandAntibodies: {
      description:
        "SSH/SQL PicoBand antibody-retrieval — developed & verified in quest 2bc664f8 step 2, extended 2026-05-17 for the v3 quote-builder. Maps each gene to its preferred Boster PicoBand antibody from the Boster live Magento 2 catalog over SSH: SKU + name + price + size + reactivity + applications. Reusable for ANY gene -> PicoBand antibody task.",
      howTo: `
**Source:** developed + executed (no CIC fallback needed) in quest
\`2bc664f8-6588-408f-966b-029e3d11813a\` step 2 (step id
\`edded2f4-1971-44e0-898b-78028eff2026\`). Reusable for any "map genes → Boster
PicoBand antibodies" task.

**Goal:** given gene symbols, return each gene's preferred PicoBand antibody SKU
(or null) from the Boster live Magento 2 catalog — NOT fabricated.

**Access:** SSH into the Boster live site using the bosterbio/live-site skill
book's SSH access pattern (the in-quest form was \`ssh -p 2223\`; Magento 2,
MariaDB \`bosterbio_m2\`). Research the Magento 2 table relations FIRST; do as
much natively as possible before any fallback.

**Magento 2 EAV schema (confirmed in-quest):**
- \`gene_name\` is an EAV product attribute: \`eav_attribute.attribute_id = 191\`,
  \`backend_type = text\` → values in \`catalog_product_entity_text\`
  (\`entity_id\`, \`attribute_id\`, \`value\`).
- \`status\` and \`name\` are denormalized onto \`catalog_product_entity\`; link
  key is \`entity_id\`.

**Query shape (v3 — extended for the quote-builder; v2 fields kept first for
back-compat):**
\`\`\`sql
SELECT
  e.entity_id, e.name, e.sku, t.value AS gene_name,
  dp.value AS price,
  rx.value_ids AS reactivity_ids,
  ap.value_ids AS application_ids,
  sz.value AS size
FROM catalog_product_entity_text t
JOIN catalog_product_entity e
  ON e.entity_id = t.entity_id
LEFT JOIN catalog_product_entity_decimal dp
  ON dp.entity_id = e.entity_id
 AND dp.attribute_id = (SELECT attribute_id FROM eav_attribute
                        WHERE attribute_code = 'price' AND entity_type_id = 4)
LEFT JOIN catalog_product_entity_varchar rx
  ON rx.entity_id = e.entity_id
 AND rx.attribute_id = (SELECT attribute_id FROM eav_attribute
                        WHERE attribute_code = 'reactivity' AND entity_type_id = 4)
LEFT JOIN catalog_product_entity_varchar ap
  ON ap.entity_id = e.entity_id
 AND ap.attribute_id = (SELECT attribute_id FROM eav_attribute
                        WHERE attribute_code IN ('application','applications')
                          AND entity_type_id = 4)
LEFT JOIN catalog_product_entity_varchar sz
  ON sz.entity_id = e.entity_id
 AND sz.attribute_id = (SELECT attribute_id FROM eav_attribute
                        WHERE attribute_code IN ('size','pack_size','volume')
                          AND entity_type_id = 4)
WHERE t.attribute_id = 191
  AND LOWER(e.name) LIKE '%picoband%'
  AND e.status = 1
  AND t.value = '<GENE>';
\`\`\`
Run per gene (or batch with \`t.value IN (...)\`, key by \`entity_id\`).
For multi-select EAV columns (reactivity / application), join \`eav_attribute_option_value\`
on \`option_id IN (split value_ids)\` to resolve to label text ("Human, Mouse, Rat" /
"Flow Cytometry, IHC, WB"). If the schema deviates (different attribute_code or
\`backend_type\`), discover via \`SELECT attribute_id, attribute_code, backend_type
FROM eav_attribute WHERE attribute_code IN ('price','reactivity','application',
'applications','size','pack_size','volume') AND entity_type_id = 4;\` first.
Discover Boster-specific attribute codes by sampling 2–3 known products first
(\`SELECT attribute_code FROM eav_attribute JOIN ... WHERE entity_id = <known sku>\`)
before bulk-querying — the schema is the source of truth, not the column names above.

**Selection rules (verbatim — IMPORTANT DETAILs):**
- Match "picoband" as a CASE-INSENSITIVE SUBSTRING on the antibody NAME, NOT the SKU.
- Require \`status = 1\`. A PicoBand only at \`status = 2\` does NOT count.
- No "picoband"-named antibody for a gene → antibody field is **null**; never
  substitute from another family.

**Fallback:** only fall back to CIC after 10 unsuccessful native attempts PLUS
ample Magento 2 doc research. In-quest the SSH/SQL path succeeded with no CIC
fallback (34/38 matched; AKT1[status=2 only], MKNK1, HIF1B, NAA10 → null).

**Output (v3 — extended; v2 \`{sku,name}\` keys preserved):**
\`{ gene -> { sku, name, price: number|null, size: string|null,
              reactivity: string[]|null, applications: string[]|null } | null }\`
persisted as \`antibody_picoband_map_json\`. \`price\` is the Magento list price as a
number (the v3 quote-builder computes bundle = listTotal × 0.5 and savings =
listTotal − bundle, per \`refRun\`). When an EAV column is genuinely missing for a
product, that field is \`null\` (the table cell renders \`—\`) — never fabricate.
`.trim(),
    },

    buildBiomarkerAntibodyForm: {
      description:
        "Step 2 (param: pathwayURL): build the v3 quote-builder biomarker panel — 6-col scrollable table (Target | SKU | Reactivity | Application | Size | Check), live pricing (Saved $X + Price $Y/N*size), Add-to-cart + Buy-now CTAs gated at ≥4 selected. Each Target links to its gene-info-card; per-antibody columns come from retrievePicobandAntibodies; CTAs POST to <REDACTED-HOST>/placeholderapi with action addBundleToCart / buyBundleNow per refRun.",
      howTo: `
**Input:** the gene list from step 1's \`gene_coords_json\`; each gene's
gene-info-card at \`https://www.bosterbio.com/bosterbio-gene-info-cards/[gene]\`.

**Method (v3 quote-builder; supersedes the v2 email-quote form, 2026-05-17):**
1. Dedupe the gene list to one row per official symbol (multi-location diagram
   labels collapse to a single row by their slug).
2. Call \`retrievePicobandAntibodies\` (SSH/SQL EAV — see that action's v3 query)
   to fetch each gene's preferred PicoBand antibody PLUS its \`price\`, \`size\`,
   \`reactivity\`, \`applications\`. Honor the selection rules verbatim (case-
   insensitive substring on NAME, status=1, null on no PicoBand — no cross-family
   sub; nulls render \`No PicoBand® antibody for this target — null\` and that
   row's checkbox is \`disabled\`).
3. Render the v3 panel HTML (self-contained — step 4 drops it in unchanged):
   - Heading "Biomarker Roles & PicoBand® Antibody Bundle".
   - 6-col table \`Target | SKU | Reactivity | Application | Size | Check\`
     (column-6 header is \`Check\` with the small instruction
     \`Please select at least 4 targets.\` underneath).
   - Each row: \`Target\` cell = \`<a href="https://www.bosterbio.com/bosterbio-gene-info-cards/<slug>">SLUG (DIAGRAM_LABEL)</a>\`
     in Boster-brand orange; \`SKU\` cell = the antibody SKU as a link to the
     antibody product page when known, else plain text; \`Reactivity\` /
     \`Application\` cells = comma-joined label text from EAV options;
     \`Size\` cell = e.g. \`10ug\` / \`20ul\`; \`Check\` cell = \`<input type="checkbox"
     class="bx" data-sku="..." data-price="..." data-size="...">\`.
   - The table is wrapped in \`<div class="bx-scroll" style="max-height:420px;
     overflow-y:auto;">…</div>\` so it scrolls internally. NO expander.
   - Ships notice below (centered, green check):
     "In stock items Ship Next Business Day. For Europe, ships in: 2-3 weeks."
   - Pricing block (\`<div class="bx-pricing">\`): \`<div class="bx-saved">Saved
     $<span id="bxSaved">0</span></div><div class="bx-price">Price: $<span
     id="bxPrice">0</span>/<span id="bxQty">0</span>*<span id="bxAvgSize">0</span></div>\`.
   - Two CTAs \`<button id="bxCart" disabled>Add to cart 🛒</button>
     <button id="bxBuy" disabled>Buy now ›</button>\`, outlined and filled
     respectively, Boster orange.
4. Inline JS (no external deps) that on every checkbox \`change\`:
   - Count checked \`.bx\` boxes \`n\`.
   - Sum \`data-price\` -> \`listTotal\`; \`price = round(listTotal * 0.5)\`;
     \`savings = listTotal - price\`. Compute \`avgSize\` from the \`data-size\`
     values of selected rows (numeric units stripped + averaged + the modal
     unit appended, e.g. \`10ug\`).
   - Update the pricing spans (\`bxSaved\`/\`bxPrice\`/\`bxQty\`/\`bxAvgSize\`).
   - \`bxCart.disabled = bxBuy.disabled = (n < 4)\`.
   - On CTA click: \`fetch('https://<REDACTED-HOST>/placeholderapi',
     {method:'POST', headers:{'Content-Type':'application/json'},
      body: JSON.stringify({ action: e.target.id==='bxCart' ?
        'addBundleToCart' : 'buyBundleNow', pathwayURL, selections: [...],
        bundle: { price, savings, listTotal } }) })\`. Endpoint + the two
     action strings are byte-fixed (see \`refRun\`).
5. Self-byte-check the rendered HTML before persisting:
   - Exactly 1 \`bx-scroll\` wrapper with \`overflow-y:auto\`, NO "View all"
     expander button anywhere, NO "Mix-and-match" promo banner (the v2
     banner is dropped; CTAs replace the "quote emailed" call-to-action).
   - The two action strings \`addBundleToCart\` / \`buyBundleNow\` each appear
     ≥1× (in the JS payload literal and/or a hidden form).
   - 6 \`<th>\` columns in the table head matching the locked names exactly.
   - Both CTAs have \`disabled\` on initial render.

**Output:** \`biomarker_table_form_html\` (kind=html). Persist
\`antibody_picoband_map_json\` (with the extended fields) for traceability.
`.trim(),
    },

    enrichDescriptionViaCJGEO: {
      description:
        "Step 3 (param: pathwayURL): drive CJGEO Content Magic via CIC to generate a pathway description draft (main keyword = the pathway name), read it back via the non-CIC Supabase path, trim off-page chrome, and replace every [Visual Placeholder] with a real generated figure. Output: cjgeo_enriched_draft + uploaded figures.",
      howTo: `
**Input:** the pathway NAME (from pathwayURL, e.g. "Alzheimer's disease pathway")
as the CJGEO main keyword.

**Consult the \`cjgeo\` skill book:**
- \`cjgeo.driveContentMagicUI\` — the locked CIC flow (New article → Research &
  Plan → Generate Draft → adopt step + CIC render-freeze caveat).
- \`cjgeo.connectSupabase\` — the reliable NON-CIC read of the adopted draft's
  \`content_html\` from CJGEO's OWN Supabase (project \`<REDACTED-PROJECT-REF>\`,
  creds in \`C:/Users/xsj70/cjgeo/.env.local\`). CIC reproducibly freezes on
  rendered CJGEO draft content — use this path, do not CIC-dance.

**Method:**
1. Drive the CJGEO UI via CIC: ICP = baseline life scientist (offer a new ICP
   "<pathwayName> + antibodies"); auto-accept the main keyword (pathwayName);
   keyword research after the AI relevance filter; competitor-benchmark topic
   research; generate the draft with the **custom template ON**; wait; ensure
   the draft is adopted (an Edit-Draft open/close adopts it server-side).
2. Read the adopted \`content_html\` via \`cjgeo.connectSupabase\`.
3. **Trim off-page chrome** (agent judgment): drop the extracted-styles wrapper +
   remote CSS, hero/TOC-nav/"Start Learning" CTA, trailing dead-\`#\`-anchor CTA,
   any duplicate hero/footer/newsletter/cookie/add-to-cart chrome. De-chrome
   interactive accordions into plain headings + always-visible answers (zero
   broken anchors, zero JS dependency, all content preserved). Keep verbatim
   every pathway-relevant content section.
4. **Replace every \`[Visual Placeholder]\`** with a real generated figure per
   \`refPageLayout\` (ChatGPT image skill book → SSH-upload to
   \`pub/media/images/pathway-pages/\` → \`<figure><img loading="lazy"><figcaption>\`,
   original placeholder text kept as an HTML comment). Replace ALL of them;
   report the real count.

**Output:** \`cjgeo_enriched_draft\` (kind=html) — content-only, no off-page
chrome, no broken anchors, placeholders replaced with uploaded figures.
`.trim(),
    },

    assembleAndDeployDemo: {
      description:
        "Step 4 (params: pathwayURL, demoURL): assemble the page per refPageLayout, deploy to the Boster live Magento CMS via the stdin/base64 CMS write (NOT argv — ENAMETOOLONG), sync storage + quest refs, and CIC-verify the live demoURL renders every component with every invariant byte-exact. Output: the deployed, verified demo page.",
      howTo: `
**Input:** interactive_map_html (1), biomarker_table_form_html (2),
cjgeo_enriched_draft (3); target = demoURL.

**Method:**
1. **Assemble per \`refPageLayout\` (v4):** narrow dark hero STRIP at top (title +
   subtitle + "Build your antibody bundle ↓" CTA — NOT a 50/50 full-height
   hero) → \`<section id="pathway-map">\` full-width with the interactive map
   centered at native max width → v3 quote-builder biomarker panel (6-col
   scrollable table + live pricing + Add-to-cart/Buy-now CTAs) immediately
   after the map → enriched description structured into the locked named
   sub-sections (What Is / Key Takeaways 3-card row / central biological
   figure / Crosstalk / Cell-type-specific roles / FAQ) with real figures
   replacing every \`[Visual Placeholder]\`. If a partial page already has
   map+form inline with a description slot, FILL the slot rather than
   re-assembling (keep map/form blocks byte-identical to the converged
   alignment-eval-table-pass overlay output). For a v4 swap of an existing v3
   deploy: do scoped regex-bounded replacements of the hero block (50/50 →
   strip) and move the map out of the hero into its own section, in two
   separate edits, with a CIC re-verify between them.
2. **Preserve byte-exact** through assembly + deploy: the gene-card hrefs, the
   two CTA action literals \`addBundleToCart\` / \`buyBundleNow\`, the POST endpoint
   \`https://<REDACTED-HOST>/placeholderapi\`. Confirm 0 broken / \`#\`
   anchors. Multibyte UTF-8 (α/β/κ) MUST survive the deploy round-trip.
3. **Deploy to the Boster live Magento CMS — locked transport:** use the
   bosterbio live-site weapon's CMS-page write, which now pipes the SQL over SSH
   **stdin** with the page body transited as **base64** and decoded server-side
   via \`CONVERT(FROM_BASE64('<b64>') USING utf8mb4)\` (commit \`ffb7e32\`). NEVER
   inline the page HTML into the SSH argv/\`-e\` — an ~80KB page exceeds the OS
   arg limit and fails \`spawn ENAMETOOLONG\` (the old bug). Target the \`cms_page\`
   row by the demoURL path identifier; \`php bin/magento cache:flush full_page
   config\`; warm nginx:8080 with \`X-Forwarded-Proto: https\` before the public hit.
4. **Sync the artifact-of-record:** upload the assembled HTML to Supabase storage
   and update the quest's \`final_pathway_page\` item ref + step-4 \`output[].ref\`
   so the quest View Results matches the live deliverable. (Note: rely on the
   raw \`items\`/\`steps\` tables or a verified getQuestDetails — the read path was
   hardened in \`4f3e26e\` to surface sub-query errors instead of returning empty.)
5. **Verify the user's way (SE rule #8):** the GM CIC-opens demoURL in the user's
   real browser (it passes the Cloudflare WAF; \`curl\` gets a 403 challenge page
   and is NOT authoritative — never verify by curl). Confirm: v4 narrow hero
   strip (NOT a 50/50 full-height hero); the full-width pathway map directly
   under the hero; v3 quote-builder panel directly under the map (6-col
   scrollable table; ships notice; pricing block updates live as boxes are
   checked; Add-to-cart + Buy-now both flip from disabled → enabled at the 4th
   check); enriched description in the locked named sub-section order (What
   Is, Key Takeaways 3-card, central biological figure, Crosstalk,
   Cell-type-specific roles, FAQ); real figures render; UTF-8 (α/β/κ) survived
   the deploy round-trip.
   With \`?mapdebug=1\` (debug overlay v2): every gene box tight on its node
   across the WHOLE map AND every label-above-the-box tag matches the gene
   text baked into the PNG inside that box (label-mismatch is a wrong-node
   anchor and must be re-aligned). The leaf's last-emitted
   \`alignment_eval_table_md\` must be all-pass before the GM declares done.
   Then CIC-open the SIBLING demo URL (the other pathway in the
   hypoxia + alzheimers pair) and re-confirm v4 layout + masks aligned + labels
   matching — the v4 layout is shared across both pages so neither should be
   left regressed when the other is shipped. Never proxy/self-attest.

**Output:** the deployed, CIC-verified demo page at demoURL (the final deliverable).
`.trim(),
    },
  },
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->
