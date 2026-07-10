# competitorGeneInfoEnrich

<!-- GUILDOS-SKILL name="competitorGeneInfoEnrich" -->

<!-- GUILDOS-DESC -->
Cache-first sitemap walk → per-product-page Observed MW + UniProt extract → upsert into boster_nexus agc_data_sources_cache as type='competitors', key=uniprot_id. Designed to repeat per competitor brand (Proteintech, Boster, Abcam, ...) without code changes — only the source-name + URL regexes vary. The cache row supplies hedged "some scientists report observing it at xx kDa" copy when a competitor's Observed MW differs from Boster's reference.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/competitorGeneInfoEnrich/SKILL.md -->
```markdown
---
name: competitorGeneInfoEnrich
description: >-
  Cache-first sitemap walk → per-product-page Observed MW + UniProt extract →
  upsert into boster_nexus agc_data_sources_cache as type='competitors',
  key=uniprot_id. Designed to repeat per competitor brand (Proteintech, Boster,
  Abcam, ...) without code changes — only the source-name + URL regexes vary. The
  cache row supplies hedged "some scientists report observing it at xx kDa" copy
  when a competitor's Observed MW differs from Boster's reference.
---

# competitorGeneInfoEnrich

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/competitorGeneInfoEnrich/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto competitorGeneInfoEnrich <action>` (or read the source directly).

## Actions

- **cacheFirstFetch** — Three-tier fetch ladder per URL: Common Crawl index → Wayback
- **cadence** — When to re-run (sitemap monthly bump, new brand added, MW null backfill).
- **extract** — Per-page extractors: UniProt accession from uniprot.org link,
- **rowShape** — Cache row contract: type='competitors', key=<uniprot accession>,
- **sources** — Where competitor sitemaps live, how to fetch them, and per-brand URL
- **upsert** — How to upsert into agc_data_sources_cache via cachePut so multiple
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/competitorGeneInfoEnrich/index.js -->
```js
/**
 * competitorGeneInfoEnrich — Competitor sitemap → cache enrichment skill book
 *
 * Procedure for hydrating `type='competitors'` rows in boster_nexus
 * `agc_data_sources_cache`, keyed by UniProt accession, populated from each
 * competitor's public product sitemap (Proteintech first; Boster / Abcam /
 * CST / R&D / Bio-Techne / Invitrogen to follow).
 *
 * The cache row is supplementary hydration data — the consuming biomarker
 * template uses it to compose hedged copy of the form
 * "some scientists report observing it at xx kDa" only when the competitor's
 * Observed MW differs materially from Boster's reference.
 *
 * Companion skill books:
 *   · uniprot — the canonical per-protein cache type this overlays alongside
 *   · biomarker — refSourcePolicy (when this source slots into hydration order)
 *   · hpa — sibling per-source mapping skill book pattern
 */
export const skillBook = {
  id: 'competitorGeneInfoEnrich',
  name: 'Competitor Gene-Info Enrichment',
  description:
    'Cache-first sitemap walk → per-product-page Observed MW + UniProt extract ' +
    '→ upsert into boster_nexus agc_data_sources_cache as type=\'competitors\', ' +
    'key=uniprot_id. Designed to repeat per competitor brand (Proteintech, ' +
    'Boster, Abcam, ...) without code changes — only the source-name + URL ' +
    'regexes vary.',

  toc: {
    sources: {
      description:
        'Where competitor sitemaps live, how to fetch them, and per-brand URL ' +
        'shape notes (Proteintech /products/<GENE-or-Name>-Antibody-<CAT>.htm, ...)',
    },
    cacheFirstFetch: {
      description:
        'Three-tier fetch ladder per URL: Common Crawl index → Wayback ' +
        'availability+snapshot → live with ≥10s/host gap. Honors robots/throttling ' +
        'and minimizes upstream load.',
    },
    extract: {
      description:
        'Per-page extractors: UniProt accession from uniprot.org link, ' +
        'Observed Molecular Weight from Proteintech\'s subtitle-row pattern, ' +
        'skip-rules for conjugate/Pair/PBS/CL-fluor pages that lack a WB block.',
    },
    rowShape: {
      description:
        'Cache row contract: type=\'competitors\', key=<uniprot accession>, ' +
        'data={competitors:{<brand>:[url,...]}, observedmolecularweight:"xx kDa"|null}',
    },
    upsert: {
      description:
        'How to upsert into agc_data_sources_cache via cachePut so multiple ' +
        'competitor brands merge into the same uniprot row across runs.',
    },
    cadence: {
      description:
        'When to re-run (sitemap monthly bump, new brand added, MW null backfill).',
    },
  },

  actions: {
    sources: {
      description: 'Competitor sitemap inventory + URL-slug notes',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
COMPETITOR SITEMAPS — inventory + slug shape
═══════════════════════════════════════════════════════════════════════════════
Each brand publishes a sitemap (or sitemap index) at a well-known path.
Download the XML to the user's Downloads folder OR fetch in-memory; the
ingest procedure only needs the <loc> list.

PROTEINTECH (ptglab.com) — VALIDATED 2026-05-29
  Index:        https://www.ptglab.com/sitemap-index.xml
  Products:     https://www.ptglab.com/sitemap-products-1.xml  (~50k URLs)
                https://www.ptglab.com/sitemap-products-2.xml  (paged)
  Local copy in this repo's smoke test:
                C:\\Users\\xsj70\\Downloads\\ptglab-sitemap-products-1.xml
  URL shape:    /products/<GENE>-Antibody-<CAT>.htm
                /products/<GENE>-Antibody-<CONJUGATE>-<CAT>.htm
                /products/<NAME>-Recombinant-Matched-Antibody-Pair-...-<CAT>.htm
  Gene from URL: regex /products/(.+?)-Antibody-  → group 1
    Caveats observed: "Human-GCSH-Recombinant-Matched" (Matched Pair noise),
    "FCGR3A-CD16a" (alias glued), "Annexin-A1" (display name not symbol).
    Real gene symbol is recovered later from the UniProt accession on-page.

BOSTER (bosterbio.com) — NOT YET VALIDATED, expected shape
  Index:        https://www.bosterbio.com/sitemap.xml (or sitemap_index.xml)
  Anchor on the product-detail URL pattern when added.

ABCAM, CST, R&D, BIO-TECHNE, INVITROGEN — to be added
  Pattern: scan the brand's root for /sitemap.xml or /robots.txt → Sitemap:
  Confirm the URL-to-gene heuristic and any conjugate/non-WB exclusions BEFORE
  enabling the brand in extract().

GENERAL RULE: never assume a brand has only one sitemap. Always parse the
sitemap-index first; iterate every product-shaped child.
═══════════════════════════════════════════════════════════════════════════════
      `,
    },

    cacheFirstFetch: {
      description: 'Three-tier fetch ladder per URL (CC → Wayback → live)',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
CACHE-FIRST FETCH LADDER — per URL
═══════════════════════════════════════════════════════════════════════════════
GOAL: avoid hitting the competitor's live server when possible. Coverage on
deep B2B catalogs is partial in Common Crawl, better in Wayback for
recently-archived URLs, and live is the unavoidable fallback for SKUs that
neither cache holds.

ORDER (per URL):

1. COMMON CRAWL INDEX LOOKUP (anonymous, no key)
     GET http://index.commoncrawl.org/<INDEX>-index?url=<urlencoded>&output=json&limit=1
     Try multiple recent monthly indexes (e.g. CC-MAIN-2026-21, ...-17, ...-12,
     ...-08). 404 = no hit for this URL in that index; iterate to next.
     On hit (status=200), JSON line carries {filename, offset, length}.
     Then GET https://data.commoncrawl.org/<filename> with
       Range: bytes=<offset>-<offset+length-1>
     gunzip the chunk → WARC record (WARC headers, blank, HTTP headers, blank,
     payload). Split on \\r\\n\\r\\n twice; last block is the HTML.
     Tag source = "cc:<INDEX>".

2. WAYBACK AVAILABILITY → SNAPSHOT FETCH (anonymous, no key)
     GET https://archive.org/wayback/available?url=<urlencoded>
     If archived_snapshots.closest exists, GET that .url; tag source =
     "wayback:<timestamp>".
     Note: snapshots may pre-date a page revamp; the UniProt link or the
     Observed MW row may be absent from older snapshots even if present live.

3. LIVE WITH ≥10s/host GAP (last resort)
     Throttle: enforce ≥10 seconds between live requests to the same host.
     Headers: realistic UA, Accept-Language en-US, Accept-Encoding gzip.
     Decompress per Content-Encoding. Tag source = "live".

OBSERVED HIT RATES (Proteintech antibody product pages, n≈19 sampled
2026-05-29):
  Common Crawl: ~0% hit on deep SKU URLs (wildcard /products/* returns
    different SKUs — CC samples, doesn't exhaust the catalog).
  Wayback:      ~55% hit. Of those, occasional snapshot is too old to
    carry the field of interest; falls through to live on retry runs.
  Live:         ~45% (all of the above succeeded; 10s throttle held; no 4xx/5xx).

Net: cache-first absorbed ~55% of would-be live traffic in the validation run.
═══════════════════════════════════════════════════════════════════════════════
      `,
    },

    extract: {
      description: 'UniProt accession + Observed MW extractors, skip-rules',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
PER-PAGE EXTRACTORS — what to pull from a competitor product page
═══════════════════════════════════════════════════════════════════════════════
PROTEINTECH product pages are server-rendered static HTML. No JS hydration
required; no JSON-LD with the fields we want.

A. UNIPROT ACCESSION (the row key)
   Regex: /uniprot\\.org\\/uniprot[e]?\\/([A-Z0-9]{6,10})/i
   The product page links to UniProt for the antigen. Group 1 is the accession.
   Null → skip the row (we cannot key it).

B. OBSERVED MOLECULAR WEIGHT (the payload)
   Pattern: <td class="subtitle">Observed Molecular Weight</td><td>...kDa</td>
   Regex:   /<td class="subtitle">Observed Molecular Weight<\\/td>\\s*<td[^>]*>(.*?)<\\/td>/s
   Strip inner tags + trim. Examples: "10 kDa", "60-65 kDa", "47 kDa, 51 kDa".

C. SKIP-RULES (pre-filter the candidate URL list)
   Pages that lack a WB block by design — UniProt may still be present, but
   Observed MW will be null. Drop these from the candidate list to spend the
   10s/host budget elsewhere:
     Pair, ELISA, -Recombinant-, -Protein-, CL488, CL594, CL647, FITC,
     -PE-, -APC-, -PBS-, -Biotin-, CoraLite
   These tokens in the URL slug correlate ~100% with absent WB validation
   panels (conjugates, kits, antigens, fluor-tagged, BSA-free buffer SKUs).

D. NULL-MW POLICY
   If both UniProt and MW are needed, skip the URL. If UniProt is present
   but MW is null on every candidate URL for a gene, write the row with
   observedmolecularweight=null — the upsert is still useful because it
   records the competitor URL list under the right uniprot key.

E. MULTI-SKU-PER-GENE RETRY
   When a competitor carries several antibodies for the same target, walk
   their URLs in order; the first non-null Observed MW wins. Stop iterating
   once MW is set. Other SKUs still merge into competitors[<brand>] later
   when their gene is rediscovered via UniProt accession.
═══════════════════════════════════════════════════════════════════════════════
      `,
    },

    rowShape: {
      description: 'Locked cache row contract for type=\'competitors\'',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
CACHE ROW SHAPE — type='competitors' (locked 2026-05-29)
═══════════════════════════════════════════════════════════════════════════════
Table:  boster_nexus Supabase, public.agc_data_sources_cache
Type:   "competitors"
Key:    UniProt primary accession (string, e.g. "O60262")
Data:   JSONB, exactly:
  {
    "competitors": {
      "<BrandName>": ["<product_url1>", "<product_url2>", ...]
    },
    "observedmolecularweight": "<MW string>" | null
  }

FIELD RULES
  · competitors : Record<string, string[]>
      One key per competitor brand that carries this antigen. Order =
      insertion order; dedupe URLs on insert. Brand name is the display
      string (e.g. "Proteintech", "Boster", "Abcam"); pick a canonical
      spelling once and reuse.
      An empty array under a brand means the brand is known to carry the
      target but no URLs are captured yet — useful for partial scans.

  · observedmolecularweight : string | null
      A SINGLE consensus value per row (locked decision — the hydration
      copy is hedged: "some scientists report observing it at xx kDa", so
      one value is enough). Format = the kDa string verbatim from the
      strongest source (e.g. "58 kDa", "60-65 kDa", "67 kDa, 57 kDa").
      null = no competitor page in this run carried a WB MW block for
      this gene; safe to backfill on a later run.

CONSENSUS RULE (when multiple competitors disagree)
  1. First non-null Observed MW encountered wins. Stable iteration order:
     competitor names ascending, then URLs in their stored array order.
  2. Conjugate / fluor-tagged / Matched-Pair SKUs are skipped per
     extract() rule C — they never carry MW by design.
  3. Do NOT range-merge across competitor values. The downstream copy
     is hedged and first-found is sufficient.

WHY NOT PER-COMPETITOR MW?
  Hedged hydration copy renders one number with caveats; per-competitor
  storage would force a UI choice the consumer doesn't make. Keep the
  shape minimal until a real consumer needs more.
═══════════════════════════════════════════════════════════════════════════════
      `,
    },

    upsert: {
      description: 'Where to write + how to merge multiple competitor runs',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
UPSERT — agc_data_sources_cache via cachePut
═══════════════════════════════════════════════════════════════════════════════
REPO:   boster_nexus
WRITER: libs/agc_pipeline/cache.js → cachePut(type, key, data)
READER: libs/agc_pipeline/cache.js → cacheGet(type, key)
LIST:   libs/agc_pipeline/cache.js → cacheList({ type: 'competitors' })

UNDER THE HOOD
  POST /rest/v1/agc_data_sources_cache?on_conflict=type,key
    Prefer: resolution=merge-duplicates,return=minimal
    Body:   [{ type: "competitors", key: "<uniprot>", data: { ... } }]
  Trigger advances modified_at on update; created_at preserved on conflict.

ONE-SHOT WRITE (10 rows or fewer):
  cd C:\\Users\\xsj70\\boster_nexus
  node --env-file=.env.local --input-type=module -e "
    import { cachePut } from './libs/agc_pipeline/cache.js';
    const rows = [ /* { uniprot_id, mw, url, brand } */ ];
    for (const r of rows) {
      await cachePut('competitors', r.uniprot_id, {
        competitors: { [r.brand]: [r.url] },
        observedmolecularweight: r.mw,
      });
    }
  "
  This validated 2026-05-29 against 10 Proteintech rows — all 10 PUT
  succeeded, all 10 cacheGet read-backs returned the locked shape.

BULK WRITE (>500 rows): use the same REST batching as
uniprot.bulkIngest §6 (chunk 500, Prefer: merge-duplicates,return=minimal,
direct REST not the per-row helper).

CROSS-BRAND MERGE (the critical correctness rule)
  When a SECOND competitor brand walk hits the same UniProt key, the new
  PUT REPLACES the row's data (Supabase merge-duplicates is a row-level
  replace, not a JSONB deep-merge). The ingest script MUST:
    1. cacheGet('competitors', uniprot) first
    2. Deep-merge the existing data.competitors with the new brand's
       URLs (dedupe), and pick observedmolecularweight per the rowShape
       consensus rule
    3. cachePut the merged data
  See extract() and rowShape() consensus rules for the merge math.

  Alternative (avoids the read): a SQL RPC that merges JSONB server-side,
  mirroring the agc_apply_uniprot_tsv_ovl() pattern from uniprot.bulkIngest §6.
  Recommended when the cross-brand walk grows beyond a few hundred rows.
═══════════════════════════════════════════════════════════════════════════════
      `,
    },

    cadence: {
      description: 'When to re-run',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
CADENCE — when to re-run competitorGeneInfoEnrich
═══════════════════════════════════════════════════════════════════════════════
1. Sitemap bump (monthly per brand)
   HEAD the brand's sitemap-index; if Last-Modified moved, walk new URLs
   only (diff against the stored competitors[<brand>] URL lists).

2. New competitor brand onboarded
   Add the brand's sitemap inventory to sources(), define its URL→gene
   heuristic + skip tokens, run a full walk. New rows merge into existing
   uniprot keys via the cross-brand merge rule in upsert().

3. MW null backfill
   Periodically re-scan rows where observedmolecularweight=null. The null
   may have been a Wayback snapshot artifact rather than a true page
   absence; a live retry often resolves it.

4. Refresh trigger from downstream
   When the biomarker hydration template starts rendering the
   "some scientists report observing it at …" line for a gene whose
   competitors row is stale or null, surface that gene to the backfill
   queue.

This skill book is the procedure spec. The executable script (when promoted
from inline one-shots) belongs in boster_nexus under
\`scripts/ingest-competitors.mjs\`, modeled on \`scripts/ingest-uniprot-bulk.mjs\`
and \`scripts/ingest-uniprot-tsv.mjs\`. Until then, inline node -e against
\`cachePut\` is the supported path.
═══════════════════════════════════════════════════════════════════════════════
      `,
    },
  },
};
```
<!-- /GUILDOS-FILE -->
