# hpa

<!-- GUILDOS-SKILL name="hpa" -->

<!-- GUILDOS-DESC -->
Per-tab map of what data lives on each Human Protein Atlas per-gene page. Tells you WHERE the protein-level (IHC / mass-spec) evidence lives vs the mRNA-level (nTPM) evidence per tab. Use BEFORE building any HPA scraper or choosing which tab to point a user at for a given evidence claim. Also documents the whole-catalog normal-tissue-IHC bulk seed, canonical download URLs (with the stale-path 404 trap), the cache row shape, and bulk-vs-OmniPath-vs-scrape supplier tradeoffs.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/hpa/SKILL.md -->
```markdown
---
name: hpa
description: >-
  Per-tab map of what data lives on each Human Protein Atlas per-gene page. Tells
  you WHERE the protein-level (IHC / mass-spec) evidence lives vs the mRNA-level
  (nTPM) evidence per tab. Use BEFORE building any HPA scraper or choosing which
  tab to point a user at for a given evidence claim. Also documents the
  whole-catalog normal-tissue-IHC bulk seed, canonical download URLs (with the
  stale-path 404 trap), the cache row shape, and bulk-vs-OmniPath-vs-scrape
  supplier tradeoffs.
---

# hpa

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/hpa/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto hpa <action>` (or read the source directly).

## Actions

- **antibodyTab** — /antibody — antibody validation data: WB / IHC / ICC images, validation status
- **bloodTab** — /blood — plasma proteome (mass-spec protein-level) + blood-cell RNA
- **brainTab** — /brain — brain region IHC + RNA + cross-species comparison
- **bulkIngest** — Full howTo for whole-catalog bulk seed of normal-tissue IHC into agc_data_sources_cache (download URL with stale-path 404 trap, TSV schema, streaming groupBy, batched upsert, throughput tip, refresh cadence)
- **cancerTab** — /cancer — IHC in tumor tissues + prognostic associations
- **cellLineTab** — /cell+line — per-cell-line expression (RNA + protein for the ~64-line HPA panel)
- **interactionTab** — /interaction — PPI graph (no expression data)
- **overview** — URL pattern + list of known tabs + which tabs hold protein-level data
- **pathologyTab** — /pathology — cancer-pathology IHC + survival analysis (more detail than /cancer)
- **proteinEvidenceQuickRef** — Quick-ref: which tab to read for which kind of protein-level evidence claim
- **refDataShape** — Cache row shape for HPA bulk-ingested entries in agc_data_sources_cache.data (mirrors fetchHPA per-gene contract so cache-first lookups read transparently)
- **refDownloadUrls** — Canonical HPA download URLs (normal_ihc, full XML, full JSON) with sizes and what each covers — stale-path 404 trap flagged
- **refSupplierTradeoffs** — When to use bulk seed vs OmniPath per-gene vs per-tab SPA scrape — cross-references biomarker.refSourcePolicy + biomarker.refOmniPathHpa
- **singleCellTab** — /single+cell — single-cell RNA per cell type (mRNA-only; no protein on this tab)
- **structureTab** — /structure — sequence variants, AF model, ipTM (no expression data)
- **summaryTab** — Gene-summary tab (no /suffix): right-side box with Protein evidence + IHC reliability + antibody list
- **tissueTab** — /tissue — per-tissue IHC High/Medium/Low/Not-detected + RNA tissue specificity
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/hpa/index.js -->
```js
/**
 * Human Protein Atlas (HPA) — Data Source Mapping Skill Book
 *
 * Per-source mapping of WHAT is on each HPA per-gene tab and WHERE the
 * protein-level evidence (vs mRNA-level) lives. Use this BEFORE writing any
 * scrape, prompt, or chain step that ingests HPA data — knowing which tab
 * holds the data you want eliminates wrong-tab scraping and saves round-trips.
 *
 * URL pattern (per gene):
 *   https://www.proteinatlas.org/{ENSG_ID}-{GENE_SYMBOL}/{tab}
 *
 * Tabs known to exist (each has its own data layout):
 *   (none)          — Gene summary (right-side box has "Protein evidence" + "IHC reliability" + antibody list)
 *   /tissue         — IHC per tissue: per-tissue High/Medium/Low/Not-detected calls + RNA tissue specificity
 *   /brain          — Brain region IHC + RNA + mouse/pig comparison
 *   /single+cell    — Single-cell RNA per cell type (transcript-level)
 *   /cancer         — Pathology atlas: IHC in tumor tissues, prognostic associations
 *   /blood          — Plasma proteome (mass-spec protein-level) + blood-cell expression
 *   /cell+line      — Per-cell-line expression (RNA + protein for ~64 lines)
 *   /structure      — Protein structure / sequence variants
 *   /interaction    — Protein-protein interactions
 *   /antibody       — Antibody validation data (WB / IHC / ICC images)
 *   /pathology      — Cancer pathology (similar to /cancer but more detailed)
 *
 * First populated 2026-05-20. THE FIRST per-source data-mapping skill book —
 * this sets the precedent: each external data source we ingest gets a skill
 * book like this so future agents know what's on what page without re-probing.
 *
 * SCRAPING GOLDEN RULE (verified on VEGFA / ENSG00000112715, 2026-05-20):
 *   HPA per-tab pages are SERVER-SIDE RENDERED static HTML — no JS SPA, no
 *   client-side hydration needed for any of the data this skill book references.
 *   `curl -sL <url>` returns the full DOM including all tables, IHC scores,
 *   nTPM values, and inline `<script>$('#id').barChart([...])</script>` JSON
 *   blobs that ship the structured data alongside the rendered visualization.
 *   You DO NOT need a headless browser for any tab.
 *
 *   Inline data blob anchors to grep for (each tab ships data this way):
 *     · `$('#rnaThumb1').barChart([{...}])`    — per-tissue / per-region nTPM
 *     · `$('#barchart_id').barChart([...])`    — generic bar charts
 *     · `intPlot.addData({"nodes":[...],"links":[...]})`  — interaction graph
 *     · `structureData = Object.assign(...)`    — structure metadata
 *     · `boxplot` SVGs with TCGA cancer rect.box per-cohort fill colors
 *     · `<div class="tissuepop">` blocks contain `title="<table>...</table>"`
 *       attributes with per-cell-type IHC scores ("Not detected"/"Low"/"Medium"/"High")
 */
export const skillBook = {
  id: 'hpa',
  name: 'Human Protein Atlas — Data Mapping',
  description:
    'Per-tab map of what data lives on each Human Protein Atlas per-gene page. ' +
    'Tells you WHERE the protein-level (IHC / mass-spec) evidence lives vs the ' +
    'mRNA-level (nTPM) evidence per tab. Use BEFORE building any HPA scraper or ' +
    'choosing which tab to point a user at for a given evidence claim.',

  toc: {
    overview: { description: 'URL pattern + list of known tabs + which tabs hold protein-level data' },
    summaryTab: { description: 'Gene-summary tab (no /suffix): right-side box with Protein evidence + IHC reliability + antibody list' },
    tissueTab: { description: '/tissue — per-tissue IHC High/Medium/Low/Not-detected + RNA tissue specificity' },
    brainTab: { description: '/brain — brain region IHC + RNA + cross-species comparison' },
    singleCellTab: { description: '/single+cell — single-cell RNA per cell type (mRNA-only; no protein on this tab)' },
    cancerTab: { description: '/cancer — IHC in tumor tissues + prognostic associations' },
    bloodTab: { description: '/blood — plasma proteome (mass-spec protein-level) + blood-cell RNA' },
    cellLineTab: { description: '/cell+line — per-cell-line expression (RNA + protein for the ~64-line HPA panel)' },
    structureTab: { description: '/structure — sequence variants, AF model, ipTM (no expression data)' },
    interactionTab: { description: '/interaction — PPI graph (no expression data)' },
    antibodyTab: { description: '/antibody — antibody validation data: WB / IHC / ICC images, validation status' },
    pathologyTab: { description: '/pathology — cancer-pathology IHC + survival analysis (more detail than /cancer)' },
    proteinEvidenceQuickRef: { description: 'Quick-ref: which tab to read for which kind of protein-level evidence claim' },
    bulkIngest: { description: 'Full howTo for whole-catalog bulk seed of normal-tissue IHC into agc_data_sources_cache (download URL with stale-path 404 trap, TSV schema, streaming groupBy, batched upsert, throughput tip, refresh cadence)' },
    refDownloadUrls: { description: 'Canonical HPA download URLs (normal_ihc, full XML, full JSON) with sizes and what each covers — stale-path 404 trap flagged' },
    refDataShape: { description: 'Cache row shape for HPA bulk-ingested entries in agc_data_sources_cache.data (mirrors fetchHPA per-gene contract so cache-first lookups read transparently)' },
    refSupplierTradeoffs: { description: 'When to use bulk seed vs OmniPath per-gene vs per-tab SPA scrape — cross-references biomarker.refSourcePolicy + biomarker.refOmniPathHpa' },
  },

  actions: {
    overview: {
      description: 'URL pattern + list of known tabs + which tabs hold protein-level data',
      howTo: `
HPA per-gene URL pattern:
  https://www.proteinatlas.org/{ENSG_ID}-{GENE_SYMBOL}/{tab}

ENSG_ID: Ensembl gene ID (e.g. ENSG00000112715 for VEGFA)
GENE_SYMBOL: HGNC gene symbol (e.g. VEGFA)
{tab}: omit for gene summary; or one of: tissue, brain, single+cell, cancer, blood,
       cell+line, structure, interaction, antibody, pathology

Tabs holding PROTEIN-LEVEL evidence (IHC / mass-spec):
  · (summary)     — Protein evidence call ("Evidence at protein level"), IHC reliability, antibody list
  · /tissue       — Per-tissue IHC High/Medium/Low/Not-detected
  · /brain        — Brain region IHC
  · /cancer       — Tumor IHC
  · /blood        — Plasma mass-spec
  · /cell+line    — Cell line IHC (where available)
  · /antibody     — Per-antibody validation images (WB / IHC / ICC)
  · /pathology    — Cancer-pathology IHC + survival

Tabs that are mRNA-only (no protein evidence on these specific tabs):
  · /single+cell  — single-cell RNA only

Tabs with no expression data:
  · /structure, /interaction

API limitation (LOCKED in libs/skill_book/biomarker → refSourcePolicy):
  HPA's search_download API exposes ONLY mRNA-level data (rnats, rnatsm, etc.).
  Protein-level data is rendered into the HTML of the per-tab pages — must be
  HTML-scraped server-side (not client-side; CORS-blocked on raw HTML fetches).
  Bulk download files (normal_tissue.tsv.zip, etc.) contain full protein-level
  matrices but are too large for runtime fetch — load at build time only.
      `,
    },

    summaryTab: {
      description: 'Gene-summary tab — right-side box with the most important per-gene metadata',
      howTo: `[POPULATED 2026-05-20 from VEGFA ENSG00000112715]
URL: https://www.proteinatlas.org/{ENSG}-{GENE} (no suffix)

HEADLINE: Top-level gene card. The single best "did this gene get a protein-level call?" page.
Mix of categorical strings (Protein evidence / Tissue specificity) and small thumbnail bar
charts (RNA tissue + brain barCharts) — no per-tissue rows.

KEY PANELS (in DOM order):
  1. GENERAL INFORMATION — \`<table class="summary_info border dark">\` (left side):
       Gene name, Gene description, Protein class, Protein evidence ← KEY ROW
       Number of transcripts, Protein interactions
  2. PROTEIN EXPRESSION AND LOCALIZATION — same table-class, separate \`<table>\`:
       Tissue profile (free-text summary), Subcellular location, Predicted location,
       Extracellular location (if secretome-annotated)
  3. TISSUE RNA EXPRESSION — barChart thumbnails (rnaThumb1 = consensus tissues,
     rnaThumb2 = brain regions) with inline \`$('#rnaThumb1').barChart([...])\` JSON
  4. SINGLE CELL TYPE / BLOOD / SUBCELLULAR / CANCER mini-cards further down
  5. ANTIBODIES table at bottom: HPA###### / CAB###### IDs per row

PROTEIN-LEVEL EVIDENCE ON THIS TAB:
  · Row \`<th>Protein evidence</th><td>...</td>\` — one of:
      "Evidence at protein level" / "Evidence at transcript level" / "Inferred from homology"
    Critical for LOCKED rule (biomarker.refSourcePolicy).
  · IHC reliability under "IMMUNOHISTOCHEMISTRY DATA RELIABILITY" head row
      (Enhanced / Supported / Approved / Uncertain) — appears in subsection lower on page.

mRNA-LEVEL EVIDENCE ON THIS TAB:
  · Tissue specificity (RNA): "Low tissue specificity" / "Tissue enhanced (X)" / etc.
  · Inline barCharts ship full per-tissue nTPM arrays as JSON in \`barChart([...])\` arg.

STATIC vs JS:
  100% server-rendered static HTML. Curl returns the full \`<table class="summary_info">\`
  rows AND the inline \`<script>\` JSON blocks. No headless browser needed.

BEST SELECTORS:
  · Protein evidence:    XPath \`//tr[th[normalize-space()="Protein evidence"]]/td\`
  · Tissue specificity:  XPath \`//tr[th//span[normalize-space()="Tissue specificity (RNA)"]]/td\`
  · IHC reliability:     under section \`<th colspan="2">IMMUNOHISTOCHEMISTRY DATA RELIABILITY</th>\`
  · Antibody list:       grep \`/(HPA|CAB)\\d{6}/\` (Atlas Antibodies + Sigma-Aldrich product URLs)
  · nTPM JSON arrays:    regex \`\\$\\('#rnaThumb1'\\)\\.barChart\\((\\[.+?\\])\\)\` (greedy until \`, \\{ barWidth)

CROSS-REFERENCE:
  Each subsection has a colored "summary_atlas_link" anchor (e.g. /tissue, /brain, /blood)
  that jumps to the dedicated tab. The summary is a SUPERSET INDEX — every datum here is
  also on the tab the colored link points to, usually with more detail.
      `,
    },

    tissueTab: {
      description: 'Tissue tab — per-tissue IHC calls + RNA tissue specificity',
      howTo: `[POPULATED 2026-05-20 from VEGFA ENSG00000112715/tissue]
URL: https://www.proteinatlas.org/{ENSG}-{GENE}/tissue

HEADLINE: The richest per-normal-tissue page. Pairs (a) per-tissue IHC High/Medium/Low/Not-detected
calls across ~44–51 tissues with (b) per-tissue nTPM RNA bars in the SAME row layout, side by side.

KEY PANELS (in DOM order):
  1. GENERAL INFORMATION + HUMAN PROTEIN ATLAS INFORMATION header tables (same as summary)
  2. Tissue expression cluster (RNA) — e.g. "Ciliated tissues - Cilium organization"
  3. Tissue specificity (RNA) — categorical
  4. Tissue distribution (RNA) — "Detected in all" / "many" / "some" / "single" / "not detected"
  5. \`<div class="tissue_summary menu_margin">\` — the main per-tissue table:
     One \`<tr>\` per organ + nested \`<div class="tissuepop">\` blocks per sub-tissue/cell-type
  6. IHC IMAGES panel (right side) — selected representative tissue images
  7. Per-antibody staining summary (lower) — IHC score per antibody per tissue

PROTEIN-LEVEL EVIDENCE ON THIS TAB:
  · Per-tissue IHC call: encoded inside the \`title="..."\` attribute of \`<td>\` cells inside
    \`<div class="tissuepop">\`. The title contains an HTML \`<table>\` with rows like:
        <td class='score'>Neuronal cells:</td><th class='score_center'>Medium</th>
    Values are one of: "Not detected" / "Low" / "Medium" / "High"
  · IHC reliability (Enhanced/Supported/Approved/Uncertain) — separate row near top
  · Per-antibody IHC staining table below the main matrix

mRNA-LEVEL EVIDENCE ON THIS TAB:
  · Per-tissue nTPM as bar widths (\`width: NNNpx\` on pixel.gif background) AND in numeric
    \`title="N.N nTPM"\` attribute on each bar img. UBERON IDs in \`<a class="uberon" uberon="UBERON_NNNNNNN">\`.

STATIC vs JS:
  Fully static HTML. The whole 467k-byte response renders without JS.

BEST SELECTORS:
  · Per-cell-type IHC scores: regex on titles in tissuepop —
      \`title="\\s*<table>.*?<td class='score'>([^<]+)</td>.*?<th class='score_center'>(Not detected|Low|Medium|High)</th>\`
  · Per-tissue nTPM: XPath \`//a[@class="uberon"]/@nx\` (the \`nx\` attr is the nTPM value)
      and \`//a[@class="uberon"]/@uberon\` for the UBERON ID
  · IHC reliability: under \`<th>IMMUNOHISTOCHEMISTRY DATA RELIABILITY</th>\` section

CROSS-REFERENCE:
  Summary tab's "Tissue profile" sentence + Tissue specificity (RNA) value are the
  one-line headlines for what's expanded here. /brain tab covers brain regions in
  more detail (with mouse + pig). /pathology has the same per-tissue IHC pattern for
  tumor tissues.
      `,
    },

    brainTab: {
      description: 'Brain tab — region-level IHC + RNA + cross-species',
      howTo: `[POPULATED 2026-05-20 from VEGFA ENSG00000112715/brain]
URL: https://www.proteinatlas.org/{ENSG}-{GENE}/brain

HEADLINE: Brain regional deep-dive across THREE species (human / mouse / pig) plus an
Allen Mouse Brain ISH dataset. Per-region nTPM bars for each species, plus IHC for human
brain (under "HUMAN BRAIN PROTEIN LOCATION" section).

KEY PANELS (in DOM order):
  1. Standard GENERAL + HPA INFORMATION header tables
  2. Three-column comparison block: \`<th>Human brain</th><th>Pig brain</th><th>Mouse brain</th>\`
     — categorical "Regional specificity" + "Regional distribution" per species
  3. HPA Pig brain RNA-Seq dataset (region barCharts)
  4. HPA Mouse brain RNA-Seq dataset (region barCharts)
  5. Allen Mouse brain ISH dataset (ISH images)
  6. HUMAN BRAIN PROTEIN LOCATION — per-region IHC (cells: Neuronal cells, Glial cells)
  7. GTEx brain dataset (RNA-seq mean nTPM per region)

PROTEIN-LEVEL EVIDENCE ON THIS TAB:
  · Per-brain-region IHC calls (Neuronal cells / Glial cells) in the "HUMAN BRAIN PROTEIN
    LOCATION" section. Same \`title="<table>..."\` tissuepop encoding as /tissue tab.
  · ONLY human-brain protein data — no mouse/pig protein staining on this tab.

mRNA-LEVEL EVIDENCE ON THIS TAB:
  · Three species × ~13 regions of nTPM (HPA human, HPA pig, HPA mouse)
  · Plus GTEx human brain region nTPM
  · All ship as inline \`.barChart([...])\` JSON blocks

STATIC vs JS:
  Static HTML, ~165k bytes. Allen ISH images are external; HPA's data is all inline.

BEST SELECTORS:
  · Regional specificity per species: XPath
      \`//tr[th//span[contains(text(),"Regional specificity")]]/td[1..3]\`
  · Per-region nTPM: same \`barChart([...])\` JSON pattern as /tissue
  · Per-region IHC: tissuepop divs in HUMAN BRAIN PROTEIN LOCATION section

CROSS-REFERENCE:
  /tissue tab has brain only at organ-level (one row "Brain" with sub-regions). /brain is
  the expanded view. /single+cell has single-nuclei brain at cell-type level (sn brain).
      `,
    },

    singleCellTab: {
      description: 'Single cell tab — single-cell RNA only (no protein on this tab)',
      howTo: `[POPULATED 2026-05-20 from VEGFA ENSG00000112715/single+cell]
URL: https://www.proteinatlas.org/{ENSG}-{GENE}/single+cell

HEADLINE: mRNA-ONLY per-cell-type expression atlas — 154 cell types across 34 tissues
from scRNAseq, plus three sub-datasets: Tissue cell type (bulk-RNAseq-predicted), Single
nuclei brain (snRNAseq), and Immune cell (sorted populations). NO IHC or protein staining
on this tab — for protein-level cell-type information go to /tissue (cell-type IHC scores).

KEY PANELS (in DOM order):
  1. Standard GENERAL + HPA INFORMATION header tables (uses /single_cell_type/*.css/js)
  2. Cell type enrichment + Cell type specificity (RNA category like "Cell type enhanced")
  3. UMAP scatter plot (Plotly: plotly-custom.min.js)
  4. Per-cell-type bar chart (~110 single cell types × nTPM)
  5. Sub-sections: Single nuclei brain / Immune cells / Tissue cell type predictions

PROTEIN-LEVEL EVIDENCE ON THIS TAB:
  None. This tab is mRNA-ONLY. Confirmed: no "IHC" / "protein staining" / "Western blot"
  data appears in the rendered body. Cell-type protein info lives on /tissue (tissuepop
  per-cell-type score tables).

mRNA-LEVEL EVIDENCE ON THIS TAB:
  · Per-cell-type nTPM bar chart (the main visualization)
  · Cell type specificity category (e.g. "Cell type enhanced (Alveolar cells type 1, ...)")
  · Cell type Tau score
  · UMAP coordinates (Plotly JSON)
  · Expression-cluster annotation (e.g. cluster 12: "Endothelial cells - Angiogenesis & vascular homeostasis")

STATIC vs JS:
  HTML is static, but the UMAP scatter is rendered by Plotly client-side from inline data.
  The per-cell-type bar chart data ships inline (barChart pattern) and can be scraped
  without executing Plotly.

BEST SELECTORS:
  · Cell type specificity: XPath \`//tr[th//span[contains(text(),"Cell type specificity")]]/td\`
  · Per-cell-type nTPM: \`barChart([...])\` JSON in inline \`<script>\` (~110 entries)
  · Cell type cluster: search-form \`<option>\` list also exposes the cluster naming convention

CROSS-REFERENCE:
  Cell-type PROTEIN evidence (IHC on cells): /tissue → tissuepop per-cell-type score tables.
  Brain cell types in more detail: /brain (snRNAseq sub-section also on /single+cell).
      `,
    },

    cancerTab: {
      description: 'Cancer tab — IHC in tumor tissues + prognostic associations',
      howTo: `[POPULATED 2026-05-20 from VEGFA ENSG00000112715/cancer]
URL: https://www.proteinatlas.org/{ENSG}-{GENE}/cancer

HEADLINE: Tumor expression atlas. IHC across 20 cancers + mass-spec across 11 cancer
types + TCGA mRNA boxplots + Kaplan-Meier prognostic curves. Per the user's earlier
note: confirmed — this tab DOES carry protein-level evidence (IHC AND mass-spec MS),
not just mRNA, in addition to /tissue's normal-tissue IHC.

KEY PANELS (in DOM order):
  1. Standard GENERAL + HUMAN PROTEIN ATLAS INFORMATION header tables
  2. CANCER RNA EXPRESSION: TCGA boxplots, one rect.box per cohort, fill colors via
     CSS \`.boxplot .<Cancer>_Carcinoma_(TCGA) rect.box { fill:#XXXXXX }\`
  3. CANCER PROTEIN EXPRESSION (IHC): per-cancer tissuepop tables — High/Medium/Low/Not-detected
     across 20 cancers, similar tissuepop pattern to /tissue
  4. CANCER PROTEIN EXPRESSION (MS-MS): mass-spec normalized values for 11 cancer types
  5. PROGNOSTIC SUMMARY: Kaplan-Meier thumbnails — each \`<a id="kaplan_NNNN_NNN">\` linking to
     /ENSG-GENE/cancer/<cancer>#<COHORT>_TCGA with title text like "Liver Hepatocellular
     Carcinoma<br>Potential prognostic - unfavorable"

PROTEIN-LEVEL EVIDENCE ON THIS TAB:
  · IHC: per-tumor staining calls (20 cancers) under "CANCER PROTEIN EXPRESSION"
  · Mass spec: 11 cancer types with quantitative MS data (visible in has_data_type
    enum as "Cancer - Mass spectrometry")

mRNA-LEVEL EVIDENCE ON THIS TAB:
  · TCGA per-cohort nTPM boxplots (~20 cancer types)
  · Validation cohort boxplots for ~10 cancers
  · Cancer specificity (RNA category)

STATIC vs JS:
  Static HTML. Boxplots are SVG generated server-side; only the styling fills are in <style>.

BEST SELECTORS:
  · Prognostic links: XPath \`//a[starts-with(@id,"kaplan_")]\` — title attr has cancer + direction
  · IHC per-cancer score: same tissuepop \`<td title="<table>...">\` pattern as /tissue
  · TCGA cohorts (colors imply available cancers): regex \`.boxplot \\.([^ ]+)_\\(TCGA\\) rect\\.box\`

CROSS-REFERENCE:
  /pathology = MORE detailed version of /cancer (per-patient IHC + survival). /cancer is
  the SUMMARY level. /tissue covers normal-tissue baseline for comparison.
      `,
    },

    bloodTab: {
      description: 'Blood tab — plasma proteome + blood-cell RNA',
      howTo: `[POPULATED 2026-05-20 from VEGFA ENSG00000112715/blood]
URL: https://www.proteinatlas.org/{ENSG}-{GENE}/blood

HEADLINE: Plasma-protein-centric tab. Per user's earlier note: protein-level evidence
lives under "Protein Profiles" subsection here. SEVEN distinct quantitative protein
technologies are enumerated by HPA's data-type taxonomy on this tab (confirmed via
the has_data_type \`<option>\` list).

KEY PANELS (in DOM order):
  1. Standard GENERAL + HPA INFORMATION header tables
  2. Predicted/Extracellular location (e.g. "Secreted to blood")
  3. Immune cell expression cluster (RNA) — blood-cell sub-population mRNA
  4. PROTEIN PROFILES section — multi-platform plasma quantification (the protein-level
     headline section):
        · Blood - conc. immunoassay      (antibody-based concentration)
        · Blood - conc. mass spectrometry (MS-based concentration; pg/mL or ng/mL)
        · Blood - Olink HT               (~5400-plex PEA panel)
        · Blood - Olink 1536             (1536-plex PEA panel)
        · Blood - SomaScan               (~7000-plex aptamer)
        · Blood - ProteomEdge            (HPA in-house targeted MS)
        · Blood - Targeted proteomics    (MS-MRM)
  5. Per-immune-cell-type RNA expression (B-cells, T-cells, Neutrophils, NK, monocytes, etc.)

PROTEIN-LEVEL EVIDENCE ON THIS TAB:
  · Plasma concentration (mass-spec, immunoassay) — main scrape target for "what is the
    circulating plasma level of this protein?" questions
  · Six high-plex panel readouts (Olink HT / Olink 1536 / SomaScan / ProteomEdge /
    Targeted proteomics)

mRNA-LEVEL EVIDENCE ON THIS TAB:
  · Sorted immune-cell-population nTPM (per-cell-type bar chart)
  · Immune cell expression cluster annotation

STATIC vs JS:
  Static HTML. Protein Profiles section ships values inline.

BEST SELECTORS:
  · Available platforms for this gene: XPath \`//th[normalize-space()="Protein Profiles"]\`
    then the following sibling sections per technology
  · Plasma concentration value: section header → adjacent \`<td>\` (units: pg/mL or ng/mL
    typically appended)
  · Immune cell nTPM: \`barChart([...])\` JSON

CROSS-REFERENCE:
  Summary tab shows "Extracellular location: Secreted to blood" → /blood is the deep dive.
  Single+cell tab has overlapping immune-cell RNA but at scRNAseq granularity.
      `,
    },

    cellLineTab: {
      description: 'Cell line tab — per-cell-line expression for ~64-line HPA panel',
      howTo: `[POPULATED 2026-05-20 from VEGFA ENSG00000112715/cell+line]
URL: https://www.proteinatlas.org/{ENSG}-{GENE}/cell+line

HEADLINE: Cell-line expression atlas. ~1206 cell lines from DepMap + HPA's in-house
~40-line panel (visible in search-form: A-431, A-549, MCF-7, HeLa, Hep-G2, K-562, etc.).
RNA-DOMINANT — protein evidence on this tab is limited to MS (no IHC; cell lines don't
get IHC in HPA's pipeline, IHC is for tissue sections). Compare/correlate with TCGA on /cancer.

KEY PANELS (in DOM order):
  1. Standard GENERAL + HPA INFORMATION header tables
  2. Cell line expression cluster (RNA) annotation
  3. Cell line specificity (RNA) — category across 28 cancer types
  4. Cell line distribution (RNA)
  5. Cell line nTPM bar chart (grouped by cancer type)
  6. RNA-vs-Protein comparative bar (where nRPX MS protein data is available):
     "comparatively showing RNA level (nTPM) and Protein level (nRPX) of various cell
     line groups"
  7. Per-cell-line detailed table (~1200 lines)

PROTEIN-LEVEL EVIDENCE ON THIS TAB:
  · Cell line MS (nRPX) — only where the line has been MS-profiled in HPA's pipeline
    (subset of the 1200; visible in has_data_type as "Cell line - Mass spectrometry").
    No IHC for cell lines on HPA.

mRNA-LEVEL EVIDENCE ON THIS TAB:
  · Per-cell-line nTPM (~1206 lines from DepMap + HPA)
  · Cell line specificity category (RNA), Tau score, expression cluster

STATIC vs JS:
  Static HTML; large response. Per-line nTPM ships as inline JSON.

BEST SELECTORS:
  · Cell line specificity (RNA): XPath \`//tr[th//span[contains(text(),"Cell line specificity")]]/td\`
  · Per-line nTPM JSON: inline \`.barChart([...])\` (one big array of ~1200 entries)
  · MS-detected lines: under "Cell line - Mass spectrometry" / nRPX comparison plot

CROSS-REFERENCE:
  /cancer + /pathology cover tumor tissue (NOT cell lines) — but mapping cell-line → cancer
  type is done in HPA's specificity classification on this tab. /antibody also shows ICC
  for cell-line subcellular localization (different from cell-line MS quantification).
      `,
    },

    structureTab: {
      description: 'Structure tab — sequence variants, AF model, ipTM',
      howTo: `[POPULATED 2026-05-20 from VEGFA ENSG00000112715/structure]
URL: https://www.proteinatlas.org/{ENSG}-{GENE}/structure

HEADLINE: Protein-structure resource only. NO expression / NO IHC. All ~20 protein-coding
isoforms get AlphaFold v2.3.2 structures. Per-isoform variant counts + antigen positions
(for HPA antibodies) are exposed via inline \`structureData\` JSON.

KEY PANELS (in DOM order):
  1. Standard GENERAL + HPA INFORMATION header tables
  2. GENE INFORMATION
  3. PROTEIN BROWSER — per-isoform AlphaFold viewer (NGL Viewer, structureData JSON)
  4. PREDICTED STRUCTURE / ANTIGEN STRUCTURE / KNOWN POPULATION VARIANTS / KNOWN CLINICAL
     VARIANTS / PREDICTED BENIGN VARIANTS / PREDICTED PATHOGENIC VARIANTS — six NGL viewers,
     each with a separate structureData payload
  5. AlphaMissense population + clinical variants
  6. ipTM (interface predicted-TM) for any structured interactions

PROTEIN-LEVEL EVIDENCE ON THIS TAB:
  None (no IHC, no MS). This tab is about 3D structure + sequence variation only.

mRNA-LEVEL EVIDENCE ON THIS TAB:
  None.

STATIC vs JS:
  HTML + inline JSON is static; the 3D rendering requires NGL Viewer JS (client-side).
  The metadata (per-isoform antigen ranges, variant counts) is in the static
  \`structureData = Object.assign(structureData, { ... })\` block — fully scrapable.

BEST SELECTORS:
  · Per-isoform metadata: regex \`structureData = Object\\.assign\\(structureData,\\s*(\\{.+?\\})\\);\`
  · Antigen positions per antibody: inside structureData["ENSP_NNNNN_N-v1"].chains.A.antigens[]
    — each has \`start_pos, end_pos, ab (HPA######), identity_percent\`
  · Variant counts: structureData[...].chains.A.variantCount.{c, p, ac, ap}.{variants, sites}
    (c=clinical, p=population, ac=AlphaMissense clinical, ap=AlphaMissense population)
  · ipTM (interaction tab cross-ref): not on this tab — see /interaction

CROSS-REFERENCE:
  Antigens here link the antibodies validated on /antibody to specific isoforms. /interaction
  uses the same structure assets for PPI 3D predictions.
      `,
    },

    interactionTab: {
      description: 'Interaction tab — PPI graph',
      howTo: `[POPULATED 2026-05-20 from VEGFA ENSG00000112715/interaction]
URL: https://www.proteinatlas.org/{ENSG}-{GENE}/interaction

HEADLINE: Protein-protein interaction (PPI) graph. NO expression data. Aggregates IntAct,
BioGRID, BioPlex, and OpenCell. Consensus + per-source networks. For each consensus
edge there's also an AlphaFold-3-predicted interaction 3D structure.

KEY PANELS (in DOM order):
  1. Standard GENERAL + HPA INFORMATION header tables
  2. GENE INFORMATION
  3. ASSAYS thumbnail strip (links to /structure, /interaction, etc.)
  4. INTERACTION main panel — five network plots (consensus + IntAct + BioGRID + BioPlex
     + OpenCell), inline \`intPlot.addData({...}, 'idx1')\` JSON for the d3-based graph

PROTEIN-LEVEL EVIDENCE ON THIS TAB:
  None (no IHC, no MS quantification). The "evidence" here is interaction existence —
  curated from external PPI databases.

mRNA-LEVEL EVIDENCE ON THIS TAB:
  None directly. Each node's tooltip carries Cell type specificity (RNA category) for the
  partner — pulled from /single+cell summarization.

STATIC vs JS:
  Static HTML + inline \`intPlot.addData(...)\` JSON. The d3 network rendering is client-side
  but the underlying graph (nodes, links, ENSG IDs, structures) is in the static payload.

BEST SELECTORS:
  · Interaction graph data: regex
    \`intPlot\\.addData\\((\\{"nodes":\\[.+?\\],"links":\\[.+?\\]\\}),\\s*'idx\\d+'\\)\`
  · Per-node ENSG IDs: inside parsed nodes[].ensg_id
  · Per-edge dataset support: links[].title contains "Interaction found in datasets: Intact, Biogrid"
  · ipTM scores per consensus structure: links[].class + structure ID lookups

CROSS-REFERENCE:
  Edge AlphaFold-3 structures live on this tab (not /structure — /structure is per-isoform
  monomer only). Partner gene specificity tooltips reference /single+cell.
      `,
    },

    antibodyTab: {
      description: 'Antibody tab — per-antibody validation data (WB / IHC / ICC images)',
      howTo: `[POPULATED 2026-05-20 from VEGFA ENSG00000112715/antibody]
URL: https://www.proteinatlas.org/{ENSG}-{GENE}/antibody

HEADLINE: Per-antibody (HPA######, CAB######) validation dashboard. One column per
antibody (e.g. VEGFA has 4: HPA069116, CAB005429, CAB039240, CAB069907). For each:
ICC, IHC, and WB validation badges + image links + supplier URLs (Atlas Antibodies +
Sigma-Aldrich).

KEY PANELS (in DOM order):
  1. Standard GENERAL + HPA INFORMATION header tables
  2. ANTIBODY INFORMATION grid — one \`<th class="head last roundtop">Antibody HPA######</th>\`
     per antibody. Rows:
       · Supplier link (Atlas Antibodies + Sigma-Aldrich URLs)
       · Antibody ID
       · Validation summary — three icon badges (ICC / IHC / WB), each \`<div title="...">\`
         linking to #ICC / #IHC / #WB anchors
  3. IHC images section (anchor #IHC) — per-tissue stained sections
  4. ICC images section (anchor #ICC) — per-cell-line subcellular images
  5. WB images section (anchor #WB) — Western blot bands per cell line / tissue lysate

PROTEIN-LEVEL EVIDENCE ON THIS TAB:
  This tab IS the antibody-validation evidence layer:
  · Each badge encodes whether the antibody was validated by WB / IHC / ICC
  · Linked images are the raw evidence
  · This is the page to cite when claiming "an HPA antibody exists and was WB-validated"

mRNA-LEVEL EVIDENCE ON THIS TAB:
  None. Antibody-only.

STATIC vs JS:
  Static HTML. Image URLs use //images.proteinatlas.org/<image_id>/<file>.jpg — also static.

BEST SELECTORS:
  · Antibody IDs: regex \`Antibody\\s+(HPA|CAB)(\\d{6})\` against headers
  · Per-antibody validation badges: XPath
    \`//div[@title="Immunohistochemistry"]\` / \`@title="Western blot"\` / \`@title="Immunocytochemistry"\`
    — presence/absence + bg color indicates pass/fail
  · IHC images: XPath \`//a[@href="#IHC"]/img\` (per-tissue thumbnails further down)
  · Supplier URLs: regex \`atlasantibodies\\.com/products/(HPA\\d{6})\` and \`sigmaaldrich\\.com/product/sigma/(HPA\\d{6})\`

CROSS-REFERENCE:
  Summary tab lists antibody IDs (HPA######, CAB######) without validation detail.
  /antibody is the validation-detail page. Antigen positions on each isoform are on
  /structure (linked by HPA ID).
      `,
    },

    pathologyTab: {
      description: 'Pathology tab — cancer-pathology IHC + survival',
      howTo: `[POPULATED 2026-05-20 from VEGFA ENSG00000112715/pathology]
URL: https://www.proteinatlas.org/{ENSG}-{GENE}/pathology

HEADLINE: Cancer pathology DEEP DIVE — more detail than /cancer. Per-patient IHC scoring
(not just aggregate), Kaplan-Meier survival curves, IHC reliability for tumor staining.
Cross-references TCGA cohort survival data.

KEY PANELS (in DOM order):
  1. Standard GENERAL + HPA INFORMATION header tables
  2. IMMUNOHISTOCHEMISTRY DATA RELIABILITY — head row dedicated to tumor IHC reliability
     (separate from normal-tissue IHC reliability on /tissue)
  3. PROGNOSTIC SUMMARY — Kaplan-Meier thumbnails (\`<a id="kaplan_NNNN_NNN">\`), each with
     title="<Cancer name><br>Potential|Validated prognostic - favorable|unfavorable" and
     URL like /ENSG-GENE/cancer/<cancer-name>#<COHORT>_TCGA
  4. Per-patient IHC table for each of 20 cancer types — patient-level High/Medium/Low/
     Not-detected calls with patient IDs (Patient ID IHC search field exists on this tab)
  5. Survival statistics tables (p-values, hazard ratios) per cohort

PROTEIN-LEVEL EVIDENCE ON THIS TAB:
  · Per-PATIENT IHC scores in tumor tissues (vs /cancer which aggregates)
  · Tumor IHC reliability call
  · Antibody-staining patterns specific to malignant cells vs stroma

mRNA-LEVEL EVIDENCE ON THIS TAB:
  · TCGA per-cohort nTPM (same boxplots as /cancer)
  · Survival vs nTPM (Kaplan-Meier driven by mRNA expression cutoff)

STATIC vs JS:
  Static HTML. Kaplan-Meier curves render as static SVG thumbnails (the \`background-image\`
  URL points at a pre-rendered image — no JS plotting needed to scrape the metadata).

BEST SELECTORS:
  · Prognostic associations: XPath \`//a[starts-with(@id,"kaplan_")]\` → title attr text
    matches \`(?<cancer>[^<]+)<br>(?<class>Potential|Validated) prognostic - (?<direction>favorable|unfavorable)\`
  · Tumor IHC reliability: under \`<th colspan="2">IMMUNOHISTOCHEMISTRY DATA RELIABILITY</th>\`
  · Per-patient IHC: same tissuepop \`title="<table>...<th class='score_center'>X</th>"\` pattern
    as /tissue, but for tumor samples; patient IDs visible via the "Patient ID" search field

CROSS-REFERENCE:
  /cancer is the SUMMARY view of this tab. Same Kaplan-Meier IDs and per-cancer pages.
  /tissue gives the normal-tissue baseline for the same antibody. /antibody shows the raw
  IHC images that drive these per-patient scores.
      `,
    },

    proteinEvidenceQuickRef: {
      description: 'Quick-ref: which tab to read for which evidence claim',
      howTo: `[POPULATED 2026-05-20 — cross-tab decision table]

When you have a specific evidence claim to support, this table tells you which HPA tab
to scrape first. All listed targets are STATIC HTML (curl + regex sufficient — no
headless browser anywhere).

| Evidence claim type | Best tab | Field name on tab | Static HTML or JS? | Notes |
|---|---|---|---|---|
| "Is this gene expressed at protein level?" (categorical) | (summary) | "Protein evidence" row | static | One of 3 values: "Evidence at protein level" / "Evidence at transcript level" / "Inferred from homology". LOCKED gate for biomarker.refSourcePolicy. |
| "How reliable is HPA's IHC for this gene?" | (summary) or /tissue | "IMMUNOHISTOCHEMISTRY DATA RELIABILITY" section | static | Enhanced > Supported > Approved > Uncertain. |
| "Which normal tissues express it at protein level?" | /tissue | per-cell-type IHC in \`<div class="tissuepop">\` titles | static | Encoded as inline HTML inside \`title="..."\` attr. Scores: Not detected / Low / Medium / High per Glial cells / Neuronal cells / specific cell types. |
| "Which normal tissues express it at mRNA level?" | /tissue | UBERON anchors \`nx="<nTPM>" uberon="UBERON_NNNNNNN"\` | static | nTPM bar widths also visible; nx attr is the cleanest numeric. |
| "Which brain regions express it (protein)?" | /brain | "HUMAN BRAIN PROTEIN LOCATION" section | static | Per-region tissuepop tables (Neuronal / Glial scores). Human only — no mouse/pig protein. |
| "Which brain regions express it (RNA, multi-species)?" | /brain | Three species × 13 region barCharts | static | HPA human + HPA pig + HPA mouse + GTEx. Inline \`barChart([...])\`. |
| "Which single cell types express it?" | /single+cell (mRNA) + /tissue (protein) | scRNAseq nTPM bar / tissuepop scores | static | /single+cell is mRNA-only. Cell-type PROTEIN scores are on /tissue's tissuepop blocks. |
| "Which cell lines express it (RNA)?" | /cell+line | per-cell-line nTPM JSON | static | ~1206 lines (DepMap + HPA). |
| "Which cell lines express it (protein)?" | /cell+line | MS nRPX comparison plot ("Cell line - Mass spectrometry") | static | Subset of lines only; no IHC on cell lines (HPA does IHC on tissues, ICC on cell lines for subcellular only). |
| "Which cancers express it at protein level (IHC)?" | /cancer or /pathology | tumor IHC tissuepop | static | /pathology has per-patient detail; /cancer has aggregate. |
| "Which cancers have mass-spec protein quantification?" | /cancer | "CANCER PROTEIN EXPRESSION (MS-MS)" section | static | 11 cancer types. |
| "Is this gene prognostic in a cancer?" | /cancer or /pathology | \`<a id="kaplan_NNNN">\` Kaplan-Meier thumbnails | static | Title attr: "Potential|Validated prognostic - favorable|unfavorable". |
| "Plasma concentration (protein, mass-spec)?" | /blood | "Blood - conc. mass spectrometry" under Protein Profiles | static | Concentration in pg/mL or ng/mL adjacent to label. |
| "Plasma concentration (protein, immunoassay)?" | /blood | "Blood - conc. immunoassay" | static | Antibody-based concentration. |
| "Olink panel readout?" | /blood | "Blood - Olink HT" / "Blood - Olink 1536" sections | static | Two Olink panels (~5400-plex and 1536-plex PEA). |
| "SomaScan aptamer readout?" | /blood | "Blood - SomaScan" | static | ~7000-plex aptamer array. |
| "Targeted MS (ProteomEdge/MRM)?" | /blood | "Blood - ProteomEdge" / "Blood - Targeted proteomics" | static | HPA in-house targeted MS panels. |
| "Which immune cells express it (RNA)?" | /blood (sorted) or /single+cell (scRNA) | immune cell barchart | static | /blood = sorted bulk populations; /single+cell = scRNAseq granularity. |
| "Antibody validation evidence (WB)?" | /antibody | anchor \`#WB\` per antibody column | mix | Image URLs static; image rendering trivially scrapable via \`//images.proteinatlas.org/...\`. |
| "Antibody validation evidence (IHC images)?" | /antibody | anchor \`#IHC\` per antibody column | mix | Same; per-tissue images. |
| "Antibody validation evidence (ICC images)?" | /antibody | anchor \`#ICC\` per antibody column | mix | Subcellular images per cell line. |
| "Antibody IDs and supplier links?" | /antibody or (summary) | "ANTIBODY INFORMATION" table | static | HPA###### + CAB###### with Atlas Antibodies + Sigma-Aldrich URLs. |
| "Subcellular location (ICC)?" | (summary) link to /subcellular | "Subcellular location (ICC)" row | static | Note: /subcellular is its own tab (not in this skill book's 11-tab list yet). |
| "Predicted 3D structure (per isoform)?" | /structure | inline \`structureData = Object.assign(...)\` | mix | Metadata (antigen ranges, variant counts) is static JSON; 3D rendering is JS (NGL Viewer) but rarely needed for scraping. |
| "Protein-protein interactions?" | /interaction | inline \`intPlot.addData({nodes,links},'idxN')\` | mix | Graph data static JSON; rendering is JS (d3) but data is fully scrapable. |
| "Antigen sequence used to make the antibody?" | /structure | structureData[isoform].chains.A.antigens[] | static | Each antigen: \`start_pos, end_pos, ab (HPA######), identity_percent\`. |

GENERAL RULE: If the page renders the value as visible text, it's in the static curl response.
If it renders as an interactive viewer/graph, the data still ships in an inline \`<script>\`
JSON blob alongside the viewer init — scrape that JSON, ignore the viewer.

API SHORTCUT (LOCKED via biomarker.refSourcePolicy): HPA's JSON search_download API
exposes only RNA columns (rnats, rnatsm, etc.). For protein-level data you MUST scrape
the per-tab HTML — there is no JSON API surface for IHC / MS / antibody validation.
      `,
    },

    bulkIngest: {
      description: 'Whole-catalog bulk seed of normal-tissue IHC into agc_data_sources_cache (the lean 5.7 MB TSV — not the 730 MB full dump)',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
HPA BULK INGEST — normal-tissue IHC (validated 2026-05-29, boster_nexus commit 370c22e)
═══════════════════════════════════════════════════════════════════════════════
WHAT: One-shot seed of HPA per-tissue/per-cell-type IHC scores for ALL ~13.5k
HPA-IHC-covered genes into \`agc_data_sources_cache\` (key prefix \`hpa:\`),
written in the SAME row shape as per-gene \`fetchHPA\` so cache-first lookups
read transparently. Replaces ~13k slow per-gene OmniPath calls (~hours) with
one ~70-second download+parse+upsert.

═══════════════════════════════════════════════════════════════════════════════
1 — CANONICAL DOWNLOAD URL (the one stale-doc trap)
═══════════════════════════════════════════════════════════════════════════════
CORRECT:  https://www.proteinatlas.org/download/tsv/normal_ihc_data.tsv.zip
STALE 404: https://www.proteinatlas.org/download/normal_tissue.tsv.zip
                                              ^^^^^ no \`/tsv/\` → 404

Older HPA docs (and some internal notes) reference the path WITHOUT the \`/tsv/\`
segment. That path 404s. If you see "normal_tissue.tsv.zip" with no \`/tsv/\`
prefix anywhere, it's the stale name — replace before downloading.

Sizes: 5.7 MB compressed · 90 MB unzipped · 1,199,675 rows TSV.
Version metadata: https://www.proteinatlas.org/about/releases (also a
\`version.txt\` ships in some bundles — record as \`source_version: 'HPA v<N>'\`).

═══════════════════════════════════════════════════════════════════════════════
2 — TSV SCHEMA (HPA-stable across recent releases)
═══════════════════════════════════════════════════════════════════════════════
Columns (tab-separated, one header row):
  Gene         — Ensembl gene ID, e.g. \`ENSG00000125245.13\`
                 NB: strip the \`.13\` version suffix → cache key uses bare
                 \`ENSG00000125245\` (matches OmniPath / UniProt cross-ref).
  Gene name    — HGNC symbol (e.g. \`CD68\`)
  Tissue       — tissue label (e.g. \`Cerebellum\`)
  Cell type    — sub-tissue cell type (e.g. \`Processes in granular layer\`)
  Level        — IHC score: \`Not detected\` / \`Low\` / \`Medium\` / \`High\` / \`N/A\`
  Reliability  — \`Enhanced\` / \`Supported\` / \`Approved\` / \`Uncertain\`

SCORE MAPPING:
  Not detected → 0
  Low          → 1
  Medium       → 2
  High         → 3
  N/A          → DROP the row (don't cache an N/A — it's noise, not absence)

═══════════════════════════════════════════════════════════════════════════════
3 — STREAMING GROUPBY (the sort-order trick)
═══════════════════════════════════════════════════════════════════════════════
The input TSV is PRE-SORTED by \`Gene\`. This means a streaming parser can:
  · accumulate rows for the current Ensembl gene in memory
  · when the next row's Ensembl differs → flush the accumulated gene to DB
  · zero out the accumulator, keep going

→ constant-memory parse of all 1.2M rows. Do NOT load the whole file into an
array — it works but wastes ~500 MB resident.

Pseudocode:
  let cur = null;     // { ensembl, gene, tissues: Map<tissue, Array<cellRow>> }
  for await (const row of streamTsv(stdin)) {
    if (row.Level === 'N/A') continue;
    const ensembl = row.Gene.split('.')[0];
    if (cur && cur.ensembl !== ensembl) { await flush(cur); cur = null; }
    if (!cur) cur = { ensembl, gene: row['Gene name'], tissues: new Map() };
    const t = cur.tissues.get(row.Tissue) || [];
    t.push({ cell_type: row['Cell type'], level: row.Level,
             score_num: SCORE_MAP[row.Level], reliability: row.Reliability });
    cur.tissues.set(row.Tissue, t);
  }
  if (cur) await flush(cur);

═══════════════════════════════════════════════════════════════════════════════
4 — EXPECTED COVERAGE (sanity-check numbers from the validated run)
═══════════════════════════════════════════════════════════════════════════════
  · 13,478 genes have at least one IHC row (~95% of ~14k HPA-annotated genes)
  · 603,766 total \`(gene, tissue, cell_type)\` entries after N/A drop
  · Genes with NO IHC data (e.g. ADAMTS14 / ENSG00000138316) simply don't
    appear in the bulk file. The bulk seed CANNOT cache an honest "no data"
    for them — that's not what this file ships.

→ Per-gene live fetch (\`fetchHPA\` in boster_nexus
  \`libs/agc_pipeline/data_sources/hpa.js\`) is still the cache-MISS fallback
  for genes uncovered by the bulk file. On miss it writes
  \`{ error: 'no_ihc_protein_data' }\` to the cache, so the second lookup is
  fast even for honest-empty genes.

═══════════════════════════════════════════════════════════════════════════════
5 — CACHE ROW SHAPE (must mirror fetchHPA output — see refDataShape)
═══════════════════════════════════════════════════════════════════════════════
Each gene's flushed row writes \`agc_data_sources_cache.data\` (JSONB) as:
  {
    gene, ensembl,
    source: 'normal_ihc_data.tsv.zip',
    source_url: 'https://www.proteinatlas.org/download/tsv/normal_ihc_data.tsv.zip',
    source_version: 'HPA v<N>',
    fetched_at: '<ISO>',
    tissues: [
      { tissue, cells: [{ cell_type, level, score_num, reliability }, ...] },
      ...
    ],
    sorted: {
      positive_candidates: [ /* top 5 cells by score_num desc */ ],
      negative_candidates: [ /* top 5 cells by score_num asc (Not detected first) */ ]
    }
    // NO \`error\` field — bulk rows are always present-with-data.
    // Absence in the cache table = use per-gene fallback.
  }

The \`sorted.positive_candidates\` / \`negative_candidates\` pre-compute the top-5
high-/low-scoring (tissue, cell_type) entries so downstream MIRROR.controls
doesn't re-sort 60+ cell rows per render. See refDataShape for the contract.

═══════════════════════════════════════════════════════════════════════════════
6 — THROUGHPUT TIP (bypass the per-row cache helper)
═══════════════════════════════════════════════════════════════════════════════
The standard \`agcCache.set(key, value)\` helper is per-row and round-trips per
write — fine for one-off fetches, painfully slow for 13k inserts.

For bulk seed, BYPASS the helper and batch directly via Supabase REST:
  · chunk size: 500 rows (~26 batches for 13k)
  · POST /rest/v1/agc_data_sources_cache?on_conflict=key with
    Prefer: resolution=merge-duplicates
  · ~70 seconds end-to-end on home internet (download + parse + upsert)

DOCUMENT THE BYPASS IN THE INGEST SCRIPT — leaving a comment like
  // BULK BYPASS: not using agcCache.set() — batching 500/req via REST.
  // See libs/skill_book/hpa/index.js → bulkIngest §6 for rationale.
prevents a future maintainer from "fixing" it back to the helper.

═══════════════════════════════════════════════════════════════════════════════
7 — REFRESH CADENCE
═══════════════════════════════════════════════════════════════════════════════
HPA ships ~1-2 releases/year. Quarterly cron is sufficient.
  · check version: GET https://www.proteinatlas.org/about/releases
  · if version > cached source_version → re-run bulk ingest
  · existing rows are upserted by key (no orphan cleanup needed for the same
    gene set; only worry about NEW genes appearing in a release — they just
    get added)

═══════════════════════════════════════════════════════════════════════════════
8 — RUNTIME LOCATION
═══════════════════════════════════════════════════════════════════════════════
This skill book documents PROCEDURE only. The executable script lives in
boster_nexus: \`scripts/ingest-hpa-bulk.mjs\` (commit 370c22e, validated
2026-05-29). Run via:
  node --env-file=.env.local scripts/ingest-hpa-bulk.mjs

If the script is missing or stale: re-author from §3 (streaming groupBy) +
§5 (shape) + §6 (REST batching) — that's the full spec.
═══════════════════════════════════════════════════════════════════════════════
      `,
    },

    refDownloadUrls: {
      description: 'Canonical HPA download URLs with sizes and what each covers',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
HPA DOWNLOAD URLS — canonical list (the \`/tsv/\` path matters)
═══════════════════════════════════════════════════════════════════════════════

USE THESE EXACTLY — older HPA docs reference paths missing the \`/tsv/\` segment
and they 404. Always include \`/tsv/\` for the TSV bundles.

PROTEIN-LEVEL (IHC):
  normal_ihc_data.tsv.zip
    URL:  https://www.proteinatlas.org/download/tsv/normal_ihc_data.tsv.zip
    SIZE: 5.7 MB zip → 90 MB unzipped · 1,199,675 rows
    COV:  ~13,478 genes × 44-51 tissues × N cell-types · per-cell-type IHC
          score (Not detected/Low/Medium/High) + reliability
    USE:  Bulk seed of normal-tissue IHC for the whole catalog (bulkIngest).
    STALE-PATH 404: \`https://www.proteinatlas.org/download/normal_tissue.tsv.zip\`
                    (no \`/tsv/\` → 404 — common in older docs)

  pathology.tsv.zip
    URL:  https://www.proteinatlas.org/download/tsv/pathology.tsv.zip
    SIZE: ~10 MB zip
    COV:  Per-tumor IHC + Kaplan-Meier prognostic data (per-cancer aggregated)
    USE:  Cancer/pathology bulk seed (not yet implemented).

mRNA-LEVEL:
  rna_tissue_consensus.tsv.zip / rna_tissue_hpa.tsv.zip / rna_tissue_gtex.tsv.zip
    URL:  https://www.proteinatlas.org/download/tsv/<file>
    COV:  Per-tissue consensus nTPM (and source datasets)

  rna_single_cell_type.tsv.zip
    URL:  https://www.proteinatlas.org/download/tsv/rna_single_cell_type.tsv.zip
    COV:  Per-cell-type scRNAseq nTPM (~110 cell types)

FULL DUMPS (LARGE — load at build time only, NEVER runtime fetch):
  proteinatlas.xml.gz
    URL:  https://www.proteinatlas.org/download/proteinatlas.xml.gz
    SIZE: ~730 MB compressed
    COV:  EVERYTHING — IHC, RNA, subcellular, antibody validation, structure,
          interaction, pathology — every gene, every tab, structured XML
    USE:  Only if you need a field this skill book doesn't cover. Almost
          always overkill. (This is the file the biomarker LOCKED
          \`refOmniPathHpa\` policy rejected as "impractical" — that rejection
          applies to THIS file, not to \`normal_ihc_data.tsv.zip\`.)

  proteinatlas.json.gz
    URL:  https://www.proteinatlas.org/download/proteinatlas.json.gz
    SIZE: ~600 MB compressed
    COV:  Same as XML, JSON-serialized
    USE:  Same caveat as XML.

LIVE PER-GENE (no bulk download needed):
  OmniPath /annotations?resources=HPA_tissue&genesymbols={GENE}&format=json
    Per the biomarker LOCKED \`refOmniPathHpa\` policy — canonical per-gene
    live API for HPA protein-level evidence. Use as cache-MISS fallback for
    genes not covered by \`normal_ihc_data.tsv.zip\`.

VERSION CHECK:
  https://www.proteinatlas.org/about/releases (HTML; parse for version + date)
═══════════════════════════════════════════════════════════════════════════════
      `,
    },

    refDataShape: {
      description: 'Cache row shape for HPA bulk-seeded entries in agc_data_sources_cache.data',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
HPA CACHE ROW SHAPE — must mirror fetchHPA per-gene output
═══════════════════════════════════════════════════════════════════════════════

CANONICAL CONTRACT: per-gene shape in \`agc_data_sources_cache.data\` (JSONB).
This shape is shared between (a) bulk seed via \`bulkIngest\` and (b) per-gene
live fetch via \`fetchHPA\` (boster_nexus \`libs/agc_pipeline/data_sources/hpa.js\`)
so that cache-first lookups read the same shape regardless of source.

═══════════════════════════════════════════════════════════════════════════════
SHAPE
═══════════════════════════════════════════════════════════════════════════════
{
  // Identity (always set on present-with-data rows)
  gene:          'CD68',                            // HGNC symbol
  ensembl:       'ENSG00000125245',                 // bare — strip .NN version

  // Provenance
  source:        'normal_ihc_data.tsv.zip'          // bulk seed
                 | 'omnipath:HPA_tissue'            // per-gene fallback
                 | 'hpa:tissue-html'                // per-tab scrape (rare)
                 ,
  source_url:    '<URL of the actual download/endpoint>',
  source_version:'HPA v<N>',                        // from version.txt / releases page
  fetched_at:    '<ISO 8601 timestamp>',

  // Data (present-with-data rows)
  tissues: [
    { tissue: 'Cerebellum',
      cells: [
        { cell_type:    'Processes in granular layer',
          level:        'Medium',                   // raw HPA string
          score_num:    2,                          // 0/1/2/3 (per SCORE_MAP)
          reliability:  'Enhanced',                 // Enhanced/Supported/Approved/Uncertain
        },
        ...
      ]
    },
    ...
  ],

  // Pre-computed top-5s (so MIRROR.controls doesn't re-sort 60+ rows per render)
  sorted: {
    positive_candidates: [ /* top 5 cells by score_num desc, tie-break by reliability */ ],
    negative_candidates: [ /* top 5 cells by score_num asc — Not detected first */ ]
  }
}

═══════════════════════════════════════════════════════════════════════════════
HONEST-EMPTY ROW SHAPE (per-gene fallback only — NEVER written by bulk seed)
═══════════════════════════════════════════════════════════════════════════════
When \`fetchHPA\` runs for a gene that the bulk file doesn't cover AND OmniPath
returns 0 rows, it writes:
  {
    gene, ensembl,
    source, source_url, fetched_at,
    error: 'no_ihc_protein_data'      // sentinel — MIRROR shows mRNA-only tag
  }

Bulk seed NEVER writes \`error\` (it only sees genes that have data). The
DISTINCTION is load-bearing for the cache-miss-vs-honest-empty branch in
MIRROR.controls per biomarker \`refSourcePolicy\` RULE 5.

═══════════════════════════════════════════════════════════════════════════════
CROSS-REFERENCE
═══════════════════════════════════════════════════════════════════════════════
  · Canonical executable contract:
      boster_nexus \`libs/agc_pipeline/data_sources/hpa.js\` → \`fetchHPA(gene)\`
  · Score mapping:
      hpa.bulkIngest §2 (Not detected/Low/Medium/High → 0/1/2/3; N/A → drop)
  · Source policy gate:
      biomarker.refSourcePolicy (HPA protein > mRNA; UniProt cross-check first)
═══════════════════════════════════════════════════════════════════════════════
      `,
    },

    refSupplierTradeoffs: {
      description: 'When to use bulk seed vs OmniPath per-gene vs per-tab SPA scrape',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
HPA SUPPLIER TRADEOFFS — three valid sources for different scopes
═══════════════════════════════════════════════════════════════════════════════

The biomarker LOCKED \`refOmniPathHpa\` policy used to read "730 MB impractical"
as a blanket rejection of HPA bulk files. That rejection applies ONLY to the
FULL \`proteinatlas.xml.gz\` / \`proteinatlas.json.gz\` dumps. The LEAN
\`normal_ihc_data.tsv.zip\` (5.7 MB) is a DIFFERENT subset — protein-level IHC
only, no subcellular/pathology/cell-line. For per-tissue protein scoring (the
only HPA data the WB pipeline currently consumes), bulk wins. Both bulk and
OmniPath remain valid for different scopes; this section is the decision table.

═══════════════════════════════════════════════════════════════════════════════
DECISION TABLE
═══════════════════════════════════════════════════════════════════════════════

| Need                                          | Use                          | Why |
|---|---|---|
| Seed the WHOLE catalog (~13k genes) of normal-tissue IHC | BULK: \`normal_ihc_data.tsv.zip\` (hpa.bulkIngest) | One ~70s job vs ~hours of per-gene calls. Same row shape as fetchHPA so cache-first lookups work. |
| Per-gene lookup for ONE gene at runtime, IHC not in bulk cache | OmniPath \`/annotations?resources=HPA_tissue\` (biomarker.refOmniPathHpa) | LOCKED canonical per-gene live API. No auth, 30-day cache. Covers genes the bulk file may miss for new releases. |
| Per-gene lookup for IHC + something the bulk file doesn't carry (subcellular, pathology, cell-line MS) | Per-tab SPA scrape (this skill book's tab actions) | Bulk only covers normal-tissue IHC. For /pathology, /subcellular, /cell+line, /blood — scrape the static HTML per the per-tab howTos. |
| Build-time enrichment of EVERY gene with EVERY HPA field | FULL DUMP: \`proteinatlas.xml.gz\` (~730 MB) | Only do this if you need a field this skill book doesn't cover. Load offline, NEVER runtime fetch. (This is the file biomarker.refOmniPathHpa rejects as impractical for live use — that rejection stands.) |
| Honest "no data" for a gene HPA hasn't IHC'd | Per-gene fallback writes \`error: 'no_ihc_protein_data'\` | Bulk seed never writes the sentinel — only per-gene \`fetchHPA\` does, on the cache-miss → 0-row-OmniPath path. See hpa.refDataShape. |

═══════════════════════════════════════════════════════════════════════════════
ORDERING IN A FRESH BUILD
═══════════════════════════════════════════════════════════════════════════════
  1. Run bulkIngest once (~70s) → 13k+ rows seeded, fast cache hits for the
     overwhelming majority of biomarker lookups.
  2. Wire \`fetchHPA\` as the per-gene fallback for cache misses. It uses
     OmniPath under the hood (per biomarker.refOmniPathHpa).
  3. Schedule quarterly cron: re-check HPA release version, re-run bulkIngest
     if version bumped (hpa.bulkIngest §7).
  4. For other HPA tabs (cancer, blood, structure, antibody) — use the per-tab
     scrape actions ad-hoc. No bulk seed planned for those (lower hit rate /
     more page diversity).

═══════════════════════════════════════════════════════════════════════════════
LOCKED-POLICY CROSS-REFERENCE
═══════════════════════════════════════════════════════════════════════════════
  · biomarker.refSourcePolicy — HPA protein > mRNA, UniProt cross-check first,
    per-topic source rules. Read FIRST before wiring any HPA-consuming chain.
  · biomarker.refOmniPathHpa — LOCKED canonical per-gene live API. The "730 MB
    impractical" line in that doc refers to the FULL XML dump, not
    \`normal_ihc_data.tsv.zip\`. Bulk seed is COMPATIBLE with that policy, not a
    violation.
═══════════════════════════════════════════════════════════════════════════════
      `,
    },
  },
};
```
<!-- /GUILDOS-FILE -->
