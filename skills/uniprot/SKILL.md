# uniprot

<!-- GUILDOS-SKILL name="uniprot" -->

<!-- GUILDOS-DESC -->
Per-endpoint map of UniProt data: per-protein .txt flat-file (live REST), per-organism reference-proteome .dat.gz (bulk), TSV column export, and cache row shapes. Use BEFORE building any UniProt scraper or choosing between bulk seed and per-protein live fetch. Documents the whole-human-proteome bulk seed, canonical download URLs, the merged type='uniprot' cache shape, and the bulk-vs-REST-vs-TSV supplier tradeoffs.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/uniprot/SKILL.md -->
```markdown
---
name: uniprot
description: >-
  Per-endpoint map of UniProt data: per-protein .txt flat-file (live REST),
  per-organism reference-proteome .dat.gz (bulk), TSV column export, and cache
  row shapes. Use BEFORE building any UniProt scraper or choosing between bulk
  seed and per-protein live fetch. Documents the whole-human-proteome bulk seed,
  canonical download URLs, the merged type='uniprot' cache shape, and the
  bulk-vs-REST-vs-TSV supplier tradeoffs.
---

# uniprot

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/uniprot/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto uniprot <action>` (or read the source directly).

## Actions

- **bulkIngest** — Full howTo for whole-human-proteome bulk seed via UP000005640_9606.dat.gz
- **refDataShape** — Cache row shapes for type='uniprot' (parsed-txt, 29 fields) and
- **refDownloadUrls** — Canonical UniProt download URLs (per-organism reference proteomes,
- **refSupplierTradeoffs** — When to use bulk .dat.gz vs per-protein REST .txt vs TSV column export
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/uniprot/index.js -->
```js
/**
 * UniProt — Data Source Mapping Skill Book
 *
 * Per-source mapping of WHAT lives in each UniProt artifact (per-protein .txt,
 * per-organism .dat.gz, REST APIs) and HOW to bulk-seed `agc_data_sources_cache`
 * with the parsed-txt shape. Use BEFORE writing any UniProt scrape, prompt, or
 * chain step.
 *
 * Companion to the HPA skill book (libs/skill_book/hpa/index.js) — sets the
 * second per-source pattern. Each external data source gets a book like this
 * so future agents know what's on what endpoint without re-probing.
 *
 * SCRAPING/INGEST GOLDEN RULES (verified 2026-05-29, boster_nexus commit TBD):
 *   1. The per-organism reference-proteome flat-file
 *      (`UP000005640_9606.dat.gz` for human) decompresses to concatenated
 *      copies of the SAME `.txt` flat files served by
 *      `rest.uniprot.org/uniprotkb/{acc}.txt`, separated by `//\n`.
 *      → existing `parseUniprotTxt()` in
 *      `boster_nexus/libs/agc_pipeline/data_sources/uniprot.js` parses each
 *      chunk as-is. Zero new parsing code for bulk.
 *   2. The reference proteome (~20.4k entries) is the right subset for the
 *      WB biomarker pipeline — it contains canonical Swiss-Prot reviewed
 *      human entries plus isoforms. The full Swiss-Prot dump (570k entries,
 *      all species) is overkill; TrEMBL (250M predicted entries) is a hard no.
 *   3. The TSV-customized export from the UniProtKB search UI OVERLAYS its
 *      denormalized columns (Function [CC], Pathway, Subcellular location, raw
 *      sequence, etc.) onto the SAME type='uniprot' rows — single unified
 *      cache shape. See `refDataShape` for the contract.
 */
export const skillBook = {
  id: 'uniprot',
  name: 'UniProt — Data Mapping',
  description:
    'Per-endpoint map of UniProt data: per-protein .txt flat-file (live REST), ' +
    'per-organism reference-proteome .dat.gz (bulk), TSV column export, and ' +
    'cache row shapes. Use BEFORE building any UniProt scraper or choosing ' +
    'between bulk seed and per-protein live fetch.',

  toc: {
    bulkIngest: {
      description:
        'Full howTo for whole-human-proteome bulk seed via UP000005640_9606.dat.gz ' +
        '(canonical URL, flat-file format, parser reuse, REST batching, throughput, cadence)',
    },
    refDownloadUrls: {
      description:
        'Canonical UniProt download URLs (per-organism reference proteomes, ' +
        'full Swiss-Prot/TrEMBL dumps, TSV column export) with sizes and use cases',
    },
    refDataShape: {
      description:
        'Cache row shapes for type=\'uniprot\' (parsed-txt, 29 fields) and ' +
        'type=\'uniprot_tsv\' (denormalized 33-col TSV) in agc_data_sources_cache',
    },
    refSupplierTradeoffs: {
      description:
        'When to use bulk .dat.gz vs per-protein REST .txt vs TSV column export ' +
        '— cross-references biomarker.refSourcePolicy',
    },
  },

  actions: {
    bulkIngest: {
      description:
        'Whole-human-proteome bulk seed via UP000005640_9606.dat.gz (132 MB, ~20.4k entries) ' +
        'parsed into the SAME type=\'uniprot\' shape as per-protein live fetches',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
UNIPROT BULK INGEST — UP000005640_9606 human reference proteome
(validated 2026-05-29, boster_nexus scripts/ingest-uniprot-bulk.mjs)
═══════════════════════════════════════════════════════════════════════════════
WHAT: One-shot seed of UniProt-parsed protein records for the ENTIRE human
reviewed proteome (~20.4k canonical Swiss-Prot entries + isoforms) into
\`agc_data_sources_cache\` as \`type='uniprot'\`, written in the SAME row shape
as per-protein live fetches (so cache-first lookups read transparently).
Replaces ~20k slow REST round-trips with one ~2-3 min download+parse+upsert.

═══════════════════════════════════════════════════════════════════════════════
1 — CANONICAL DOWNLOAD URL
═══════════════════════════════════════════════════════════════════════════════
CORRECT:  https://ftp.uniprot.org/pub/databases/uniprot/current_release/
            knowledgebase/reference_proteomes/Eukaryota/UP000005640/
            UP000005640_9606.dat.gz

SIZE:    ~132 MB compressed (gzipped flat-file)
COV:     ~20,400 canonical reviewed human Swiss-Prot entries + isoforms
RELEASE: Last-Modified header carries the release timestamp; UniProt ships
         one release every ~4 weeks.

NB: \`UP000005640\` is the human reference-proteome ID. For other organisms
substitute the organism's reference-proteome ID (e.g. UP000000589_10090 for
mouse). See \`refDownloadUrls\` for the full table.

═══════════════════════════════════════════════════════════════════════════════
2 — FILE FORMAT (the key insight)
═══════════════════════════════════════════════════════════════════════════════
The decompressed \`.dat\` file is CONCATENATED per-protein flat-files,
identical to what \`https://rest.uniprot.org/uniprotkb/{acc}.txt\` returns,
separated by \`//\\n\` (the standard UniProt entry terminator).

→ The existing per-protein parser at
  \`boster_nexus/libs/agc_pipeline/data_sources/uniprot.js → parseUniprotTxt()\`
  works as-is. Split the stream on \`\\n//\\n\` and feed each chunk through.
  ZERO new parsing code.

The parser MUST be exported from that file (one-line change: prefix the
function with \`export\`) so the bulk script can import it.

═══════════════════════════════════════════════════════════════════════════════
3 — STREAMING PARSE (constant memory)
═══════════════════════════════════════════════════════════════════════════════
Stream the gunzipped data, buffer until the next \`\\n//\\n\`, flush, repeat:

  import { createReadStream } from 'node:fs';
  import { createGunzip } from 'node:zlib';
  import { parseUniprotTxt } from '../libs/agc_pipeline/data_sources/uniprot.js';

  async function* streamEntries(gzPath) {
    const src = createReadStream(gzPath).pipe(createGunzip());
    let buf = '';
    for await (const chunk of src) {
      buf += chunk.toString('utf8');
      let idx;
      while ((idx = buf.indexOf('\\n//\\n')) !== -1) {
        yield buf.slice(0, idx) + '\\n//\\n';
        buf = buf.slice(idx + 4);
      }
    }
    if (buf.trim().length) yield buf;
  }

  for await (const entryTxt of streamEntries(GZ_PATH)) {
    const parsed = parseUniprotTxt(entryTxt);
    if (!parsed.id) continue;
    parsed._source = 'UP000005640_9606.dat.gz';
    batch.push({ type: 'uniprot', key: parsed.id, data: parsed });
    if (batch.length >= 500) { await postBatch(batch); batch = []; }
  }

→ ~150 MB peak resident (download buffer + parse working set).
DO NOT load the whole file into memory — works but wastes ~600 MB.

═══════════════════════════════════════════════════════════════════════════════
4 — EXPECTED COVERAGE
═══════════════════════════════════════════════════════════════════════════════
  · ~20,400 entries total (canonical + isoforms)
  · Every entry has: id, entryName, gene, proteinName, organism, molWeight,
    sequenceLength, keywords, subcellLocation, oligomer
  · Most entries have: signalTxt, proPepTxt, modres[], glyco[], disulfides[],
    isoformCount, isoformNames[]
  · DR-line cross-refs (added 2026-05-29 to parseUniprotTxt):
    ensemblGeneId, ensemblTranscriptIds[], hpaIds[], hgncId, refSeqIds[],
    pdbIds[], reactomeIds[], keggIds[]
    → \`hpaIds[]\` is usually EMPTY: UniProt deprecated DR HPA antibody-ID lines.
      Source HPA antibody IDs from proteinatlas.org via fetchHPA instead.

═══════════════════════════════════════════════════════════════════════════════
5 — CACHE ROW SHAPE (must mirror parseUniprotTxt output — see refDataShape)
═══════════════════════════════════════════════════════════════════════════════
\`agc_data_sources_cache.data\` (JSONB) is the FULL parseUniprotTxt return
object, plus a provenance tag:
  {
    id, entryName, gene, proteinName, organism,
    molWeight, sequenceLength,
    signalTxt, proPepTxt,
    modres[], modresCount,
    glyco[], glycoCount,
    disulfides[], disulfideCount,
    isoformCount, isoformNames[],
    oligomer, subcellLocation, keywords[],
    ensemblGeneId, ensemblTranscriptIds[], hpaIds[],
    hgncId, refSeqIds[], pdbIds[], reactomeIds[], keggIds[],
    _featuresRaw,                         // raw FT feature count (debug)
    _source: 'UP000005640_9606.dat.gz'    // bulk provenance
  }

Per-protein live fetch (cache-MISS via \`fetchUniprot\` in
\`libs/agc_pipeline/data_sources/uniprot.js\`) writes the same shape WITHOUT
\`_source\` — that's the only on-the-wire difference. Both are readable by
the same downstream consumer.

═══════════════════════════════════════════════════════════════════════════════
6 — THROUGHPUT TIP (bypass per-row cache helper; server-side merge for overlay)
═══════════════════════════════════════════════════════════════════════════════
\`cachePut(type, key, data)\` round-trips once per write — fine for one-off
per-protein fetches, painfully slow for 20k inserts.

BULK INSERT (.dat.gz seed): BYPASS the helper and batch directly via REST:
  · chunk size: 500 rows (~41 batches for 20.4k)
  · POST /rest/v1/agc_data_sources_cache?on_conflict=type,key with
    Prefer: resolution=merge-duplicates,return=minimal
  · ~30s parse+upsert end-to-end after ~10s download

TSV OVERLAY (selective-fill merge onto existing uniprot rows): use the
SERVER-SIDE staging+RPC pattern from migration 116. Do NOT fetch-then-merge
per row — that's 80+ REST round-trips and ~3 minutes of network.

  Pattern (validated 2026-05-29, scripts/ingest-uniprot-tsv.mjs):
    1. POST batches (~1000/req) into agc_uniprot_tsv_staging
       (~30s for 20k rows; pure upload, no fetch)
    2. Call \`SELECT * FROM agc_apply_uniprot_tsv_ovl()\` via direct pg
       (NOT via PostgREST RPC — REST has a ~15s statement timeout that
       kills the single 20k JOIN+UPDATE; the pg connection lets you
       \`SET statement_timeout = '300s'\` first). One server-side
       statement merges all rows in ~12s.
    3. TRUNCATE staging.
  Total: ~48s end-to-end (vs ~3 min for client-side fetch-then-merge).

The merge function \`agc_merge_uniprot_ovl(cur, ovl)\` does selective fill:
TSV value only writes if the current row's key is null OR an empty array.
Structured arrays (modres / glyco / disulfides) and DR cross-refs (pdbIds /
ensemblGeneId / etc.) owned by the .dat.gz bulk are NEVER clobbered.

DOCUMENT THE BYPASS IN THE INGEST SCRIPT:
  // SERVER-SIDE MERGE: staging + agc_apply_uniprot_tsv_ovl() via direct pg.
  // See libs/skill_book/uniprot/index.js → bulkIngest §6 for rationale.

═══════════════════════════════════════════════════════════════════════════════
7 — REFRESH CADENCE
═══════════════════════════════════════════════════════════════════════════════
UniProt ships every ~4 weeks. Monthly cron is sufficient.
  · check release: HEAD the .dat.gz URL, compare Last-Modified to cached
    source release timestamp
  · if newer → re-run bulk ingest (upsert by (type,key) — no orphan
    cleanup needed for same accession set; new entries just get added)
  · obsoleted accessions are merged/secondary in UniProt; the live
    \`fetchUniprot\` handles those redirects.

═══════════════════════════════════════════════════════════════════════════════
8 — RUNTIME LOCATION
═══════════════════════════════════════════════════════════════════════════════
This skill book documents PROCEDURE only. The executable script lives in
boster_nexus: \`scripts/ingest-uniprot-bulk.mjs\`. Run via:
  node --env-file=.env.local scripts/ingest-uniprot-bulk.mjs
  node --env-file=.env.local scripts/ingest-uniprot-bulk.mjs --dry-run
  node --env-file=.env.local scripts/ingest-uniprot-bulk.mjs --limit=100
  node --env-file=.env.local scripts/ingest-uniprot-bulk.mjs --dat=<path>

If the script is missing or stale: re-author from §2 (file format) + §3
(streaming parse) + §5 (shape) + §6 (REST batching) — that's the full spec.
═══════════════════════════════════════════════════════════════════════════════
      `,
    },

    refDownloadUrls: {
      description: 'Canonical UniProt download URLs by use case',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
UNIPROT DOWNLOAD URLS — by use case
═══════════════════════════════════════════════════════════════════════════════

PER-ORGANISM REFERENCE PROTEOME (the bulk-seed sweet spot):
  Pattern: https://ftp.uniprot.org/pub/databases/uniprot/current_release/
           knowledgebase/reference_proteomes/<Kingdom>/<UPID>/<UPID>_<taxon>.dat.gz

  Human:    UP000005640_9606.dat.gz   (~132 MB)  ← used by bulkIngest
  Mouse:    UP000000589_10090.dat.gz  (~80 MB)
  Rat:      UP000002494_10116.dat.gz  (~55 MB)
  Zebrafish UP000000437_7955.dat.gz   (~50 MB)
  Yeast:    UP000002311_559292.dat.gz (~6 MB)

  Each .dat.gz decompresses to concatenated per-protein .txt flat-files
  separated by \`//\\n\`. Parse with the SAME parseUniprotTxt() used for
  live per-protein REST fetches.

FULL SWISS-PROT (all species, reviewed only):
  https://ftp.uniprot.org/pub/databases/uniprot/current_release/
    knowledgebase/complete/uniprot_sprot.dat.gz
  SIZE: ~700 MB compressed
  COV:  ~570k reviewed entries across all species
  USE:  Only if you need NON-HUMAN reviewed entries beyond the per-organism
        reference proteomes. Otherwise overkill.

FULL TrEMBL (unreviewed, predicted):
  https://ftp.uniprot.org/pub/databases/uniprot/current_release/
    knowledgebase/complete/uniprot_trembl.dat.gz
  SIZE: multi-GB compressed
  COV:  ~250M unreviewed predicted entries
  USE:  HARD NO for general bulk seed. Computational pipeline only.

PER-PROTEIN LIVE (cache-miss fallback):
  https://rest.uniprot.org/uniprotkb/{accession}.txt   (~5-50 KB per entry)
  https://rest.uniprot.org/uniprotkb/{accession}.json  (parsed JSON)
  https://rest.uniprot.org/uniprotkb/{accession}.xml   (full XML)

  The .txt format is what parseUniprotTxt expects. The .json/.xml are
  alternative parsings — convenient but the .txt parser is already in place.

TSV CUSTOM EXPORT (UniProtKB web UI):
  Path: https://www.uniprot.org/uniprotkb?query=<filter>
        → Customize columns → Download (TSV)
  USE:  Denormalized columns (Function [CC], Pathway, Subcellular location,
        cross-refs as semicolon-joined lists). Different shape from .txt
        parser — cached as \`type='uniprot_tsv'\` to coexist with the
        parsed-txt cache. Useful for quick column-level lookups that don't
        need the structured features array.
  COV (validated 2026-05-29): 20,431 rows for query
        \`(organism_id:9606) AND (reviewed:true)\` × 33 columns
  Bulk-ingest script: boster_nexus \`scripts/ingest-uniprot-tsv.mjs\`

PER-ORGANISM FASTA (sequence only):
  Pattern: <reference_proteomes path>/<UPID>_<taxon>.fasta.gz
  USE:  ONLY if you need sequences without annotations. Misses FT/CC/KW/DR.
        Don't use for the WB pipeline — annotations are the point.

RELEASE METADATA:
  https://www.uniprot.org/help/release-statistics  (release counts)
  HEAD <any .dat.gz URL>                            (Last-Modified)
═══════════════════════════════════════════════════════════════════════════════
      `,
    },

    refDataShape: {
      description: 'Single merged cache row shape for type=\'uniprot\' (parsed-txt structured arrays + TSV-overlay raw text)',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
UNIPROT CACHE ROW SHAPE — single merged type='uniprot' (locked 2026-05-29)
═══════════════════════════════════════════════════════════════════════════════

LOCKED ARCHITECTURAL DECISION (2026-05-29): there is exactly ONE UniProt cache
type — \`type='uniprot'\`. The historical split into \`uniprot\` (parsed-txt) vs
\`uniprot_tsv\` (denormalized columns) was collapsed: both ingest paths now
write into the same row, with the bulk \`.dat.gz\` owning the structured
arrays and the TSV overlay owning the raw CC / FT text and the sequence.

═══════════════════════════════════════════════════════════════════════════════
STRUCTURED HALF — owned by ingest-uniprot-bulk.mjs (parseUniprotTxt output)
═══════════════════════════════════════════════════════════════════════════════
Source: parseUniprotTxt() in boster_nexus
        libs/agc_pipeline/data_sources/uniprot.js
Written by: (a) bulkIngest from UP000005640_9606.dat.gz
            (b) fetchUniprot() per-protein live REST .txt fetches

Shape (29 fields):
  {
    // Identity
    id:               'P00533',                  // primary accession
    entryName:        'EGFR_HUMAN',              // ID line entry name
    gene:             'EGFR',                    // GN line primary gene symbol
    proteinName:      'Epidermal growth factor receptor', // DE line RecName
    organism:         'Homo sapiens (Human)',    // OS line

    // Sequence
    molWeight:        134277,                    // SQ line MW
    sequenceLength:   1210,                      // SQ line length

    // Features (raw FT-line text — for parsed positions use _featuresRaw + parser)
    signalTxt:        '1..24',                   // SIGNAL FT range or null
    proPepTxt:        null,                      // PROPEP FT range or null
    modres:           [ { position, modification }, ... ],
    modresCount:      25,
    glyco:            [ { position, info }, ... ],
    glycoCount:       8,
    disulfides:       [ { from, to }, ... ],
    disulfideCount:   3,

    // Isoforms (from CC ALTERNATIVE PRODUCTS)
    isoformCount:     4,
    isoformNames:     [ '1', '2', '3', '4' ],

    // Subunit / localization / keywords
    oligomer:         'Monomer ... Forms ...',   // CC SUBUNIT first line
    subcellLocation:  'Cell membrane; ...',      // CC SUBCELLULAR LOCATION
    keywords:         [ 'Glycoprotein', ... ],   // KW lines split

    // DR-line cross-references (added 2026-05-29)
    ensemblGeneId:        'ENSG00000146648',
    ensemblTranscriptIds: [ 'ENST00000275493', ... ],
    hpaIds:               [],                    // USUALLY EMPTY — UniProt
                                                  // deprecated DR HPA antibody
                                                  // IDs; source HPA antibody
                                                  // IDs from proteinatlas.org
    hgncId:               'HGNC:3236',
    refSeqIds:            [ 'NM_005228.5', ... ],
    pdbIds:               [ '1IVO', '1M14', ... ],
    reactomeIds:          [ 'R-HSA-177929', ... ],
    keggIds:              [ 'hsa:1956' ],

    // Provenance / debug
    _featuresRaw:     142,                       // total FT count (debug)
    _source:          'UP000005640_9606.dat.gz'  // bulk-seed provenance
  }

═══════════════════════════════════════════════════════════════════════════════
TSV-OVERLAY HALF — owned by ingest-uniprot-tsv.mjs (merged into same row)
═══════════════════════════════════════════════════════════════════════════════
Additional keys overlaid on the SAME \`type='uniprot'\` row (these don't
clobber the structured arrays — they fill where the parsed-txt shape has
nothing):
  {
    // Raw CC text (the parsed-txt shape doesn't have these as text blobs)
    function_cc, pathway, induction,
    subcellular_location_cc, transmembrane, topological_domain,
    subunit_structure, alternative_sequence,

    // Full primary sequence (parsed-txt only has length/molWeight)
    sequence,

    // Cross-ref / metadata not in parsed-txt
    ec_number[],
    reviewed,                    // 'reviewed' / 'unreviewed'
    protein_existence,           // e.g. 'Evidence at protein level'
    protein_names,               // long alt-names string
    gene_names_synonym[], gene_names_primary,
    date_of_last_modification,

    // Provenance
    _tsv_source: 'uniprotkb_organism_id_9606_AND_reviewed_<YYYY_MM_DD>.tsv'
  }

OVERLAY RULES (LOCKED):
  · TSV-overlay fields are written ONLY where the bulk row has \`null\`
    or an empty array. The structured arrays (modres / glyco / disulfides /
    isoformNames / keywords / pdbIds / etc.) are NEVER clobbered by TSV.
  · DR cross-refs (ensembl/hgnc/hpa/kegg/refseq/pdb/reactome) come from
    the .dat.gz bulk and are NOT overwritten by the TSV's same-named
    columns (parsed-txt arrays are richer and version-aligned).
  · Glycosylation / disulfide_bond / modified_residue / signal_peptide are
    NOT overlaid from TSV — the parsed arrays (glyco/disulfides/modres/
    signalTxt) are the source of truth.
  · Rows present in TSV but not in the .dat.gz bulk (102 non-canonical
    isoforms / recently-added — observed 2026-05-29) are created as new
    \`type='uniprot'\` rows seeded from the TSV alone (no structured
    arrays — only the overlay fields).

═══════════════════════════════════════════════════════════════════════════════
CROSS-REFERENCE
═══════════════════════════════════════════════════════════════════════════════
  · Canonical executable contract (structured half):
      boster_nexus libs/agc_pipeline/data_sources/uniprot.js
      → parseUniprotTxt + fetchUniprot
  · TSV-overlay column → key normalizer + merge rules:
      boster_nexus scripts/ingest-uniprot-tsv.mjs
      (headerKey + splitList + TSV_MERGE_FIELDS + flushBatch fetch-then-merge)
  · Source policy gate:
      biomarker.refSourcePolicy (UniProt-cross-check FIRST before any
      per-topic hydration)
═══════════════════════════════════════════════════════════════════════════════
      `,
    },

    refSupplierTradeoffs: {
      description: 'When to use bulk .dat.gz vs per-protein REST .txt vs TSV column export',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
UNIPROT SUPPLIER TRADEOFFS — three valid sources for different scopes
═══════════════════════════════════════════════════════════════════════════════

| Need                                          | Use                          | Why |
|---|---|---|
| Seed the WHOLE human proteome (~20k entries) with structured features | BULK: \`UP000005640_9606.dat.gz\` (uniprot.bulkIngest) | One ~2-3 min job vs ~20k REST round-trips. Same parseUniprotTxt output shape as per-protein live fetch — cache-first reads transparent. |
| Per-protein lookup for ONE accession at runtime, not in bulk cache | REST: \`https://rest.uniprot.org/uniprotkb/{acc}.txt\` (fetchUniprot) | Canonical per-protein live API. Handles obsoleted-accession redirects automatically. Writes the parsed-txt structured half of the merged \`type='uniprot'\` row. |
| Add raw CC text blobs / full sequence / EC numbers to already-cached rows | TSV column overlay (scripts/ingest-uniprot-tsv.mjs) | Run AFTER bulkIngest. Fetch-then-merge: fills function_cc / pathway / subcellular_location_cc / sequence / etc. onto the SAME \`type='uniprot'\` row without clobbering structured arrays. See refDataShape OVERLAY RULES. |
| Cross-organism (mouse, rat) bulk seed | Substitute per-organism reference proteome ID | Same .dat.gz format, same parser, just different file. See uniprot.refDownloadUrls. |
| Non-human reviewed entries beyond reference proteomes | FULL: \`uniprot_sprot.dat.gz\` (~700 MB) | Only when per-organism subset is insufficient. Same parser. |
| Predicted/unreviewed entries | FULL: \`uniprot_trembl.dat.gz\` (multi-GB) | HARD NO for general use. Computational pipelines only. |

═══════════════════════════════════════════════════════════════════════════════
ORDERING IN A FRESH BUILD
═══════════════════════════════════════════════════════════════════════════════
  1. Run uniprot.bulkIngest once (~2-3 min) → ~20.4k rows seeded under
     type='uniprot' with the structured parsed-txt half, fast cache hits
     for the overwhelming majority of biomarker lookups.
  2. Run the TSV overlay (\`scripts/ingest-uniprot-tsv.mjs\`) → fills the
     raw-CC / sequence / EC / protein_existence half on the SAME rows
     (~20.3k merged + ~100 created for TSV-only accessions). Single
     unified cache type — there is no separate \`uniprot_tsv\`.
  3. Wire \`fetchUniprot\` as the per-protein fallback for cache misses
     (obsoleted accessions, new entries, non-human).
  4. Schedule monthly cron: HEAD the .dat.gz URL, compare Last-Modified,
     re-run bulkIngest on bump (uniprot.bulkIngest §7), then re-run the
     TSV overlay to refresh raw-text fields.

═══════════════════════════════════════════════════════════════════════════════
LOCKED-POLICY CROSS-REFERENCE
═══════════════════════════════════════════════════════════════════════════════
  · biomarker.refSourcePolicy — UniProt cross-check is the FIRST step of any
    per-topic biomarker hydration. Bulk seed makes that step ~zero-latency
    for the human reviewed set.
  · hpa.bulkIngest §1 documents the SAME pattern for HPA (different file,
    different shape). This skill book is the UniProt parallel.
═══════════════════════════════════════════════════════════════════════════════
      `,
    },
  },
};
```
<!-- /GUILDOS-FILE -->
