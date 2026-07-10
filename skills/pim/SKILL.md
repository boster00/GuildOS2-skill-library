# pim

<!-- GUILDOS-SKILL name="pim" -->

<!-- GUILDOS-DESC -->
Authoritative reference for the three Boster Nexus PIM tables (pim_products, pim_attribute_definitions, pim_product_images) plus the sync-substrate tables (pim_sync_protocols, pim_field_authority) and the mapProductInfo function that hydrates the catalog from a Magento internal-export CSV. Read before reading or writing PIM data, adding a products column (the Type-A vs Type-B bucketing rule matters), designing a new ingest pipeline, wiring a product_template, or rolling back a bulk ingest. Covers column-by-column table structure, the Magento-row → PIM-row converter and its transforms, the end-to-end internal-export hydration path, and the migration history (072 → 110).
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/pim/SKILL.md -->
```markdown
---
name: pim
description: >-
  Authoritative reference for the three Boster Nexus PIM tables (pim_products,
  pim_attribute_definitions, pim_product_images) plus the sync-substrate tables
  (pim_sync_protocols, pim_field_authority) and the mapProductInfo function that
  hydrates the catalog from a Magento internal-export CSV. Read before reading or
  writing PIM data, adding a products column (the Type-A vs Type-B bucketing rule
  matters), designing a new ingest pipeline, wiring a product_template, or
  rolling back a bulk ingest. Covers column-by-column table structure, the
  Magento-row → PIM-row converter and its transforms, the end-to-end
  internal-export hydration path, and the migration history (072 → 110).
---

# pim

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/pim/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto pim <action>` (or read the source directly).

## Actions

- **hydrateVendorImagesOnBundleAndSavePDP** — When vendor images do not render on the live bundle-and-save PDP (NitroCDN 404s, classDev.php [X] strip hazards, Magento Slick gallery init failure), the full procedure lives in the bosterbio skill book under the same action name. See bosterbio.hydrateVendorImagesOnBundleAndSavePDP.
- **indexVsKeep** — Index vs Keep: same schema treatment; the distinction lives in docs/env vars, not in the table
- **ingestPipeline** — How to hydrate the PIM from the internal-export CSV end-to-end
- **mapProductInfo** — mapProductInfo(input, map?) — the canonical Magento-row → PIM-row converter
- **migrationHistory** — The migrations that shape the current PIM schema (072 → 102)
- **overview** — High-level: what the PIM is, the three tables, the mapping function
- **pimAttributeDefinitions** — pim_attribute_definitions — per-template resolver for attr_1..attr_30
- **pimFieldAuthority** — pim_field_authority — per-field ownership rules consulted by sync engines before writes (migration 110)
- **pimProductImages** — pim_product_images — normalized image rows linked to pim_products
- **pimProducts** — pim_products — main catalog (1 row per sku); column-by-column structure
- **pimSyncProtocols** — pim_sync_protocols — registry of named sync protocols between PIM and external systems (migration 109)
- **rollback** — How to reverse a bulk ingest using pim_source tagging
- **typeABucketing** — Type-A vs Type-B: which Magento columns become dedicated columns vs generic attr_N slots
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/pim/index.js -->
```js
/**
 * PIM — Boster Nexus Product Information Management Skill Book
 *
 * Authoritative reference for the three boster_nexus tables that hold the
 * canonical Boster product catalog, plus the mapping function that hydrates
 * them from a Magento internal-export CSV (or any equivalent source).
 *
 * READ THIS BEFORE:
 *   - Writing any code that reads or writes product data in boster_nexus
 *   - Designing a new ingest pipeline that targets the PIM
 *   - Adding columns to the products table (the bucketing rule matters)
 *   - Wiring a new product_template into the system
 *
 * Tables (all live in boster_nexus's Supabase under schema `public`):
 *   pim_products              — one row per SKU, the main catalog
 *   pim_attribute_definitions — per-template meaning of pim_products.attr_1..attr_30
 *   pim_product_images        — normalized images per pim_products row (1:N)
 *   pim_sync_protocols        — registry of named sync protocols (migration 109)
 *   pim_field_authority       — per-field ownership rules (migration 110)
 *
 * Renamed from `products` / `attribute_definitions` / `product_images` in
 * migration 102 to namespace the PIM domain under the `pim_` prefix (consistent
 * with pim_ingestion_jobs from migration 093 and the libs/pim/ engine layout).
 *
 * Mapping function (canonical Magento-row → PIM-row converter):
 *   GuildOS-side weapon : libs/weapon/pim/index.js → mapProductInfo(input, map?)
 *   Boster Nexus mirror : libs/pim/mapProductInfo.js (verbatim mirror)
 *   Engine              : libs/pim/convert.js (Boster Nexus) — generic
 *                          declarative converter shared by all mappers.
 *
 * Default map shape encoded in mapProductInfo (THE source of truth at runtime):
 *   { fields: { <magento_col>: { to, transform?, relation? } },
 *     typeB:  { <product_template>: { <magento_col>: <attr_index>, … } } }
 *
 *   Reference TSVs (documentation, not loaded at runtime):
 *     boster_nexus/docs/pim/attr-mapping-1-index-keep-nameless.tsv  (Type-A bucket decisions)
 *     boster_nexus/docs/pim/attr-mapping-2-nameless-attr-assignment.tsv  (Type-B per-template)
 */

export const skillBook = {
  id: 'pim',
  name: 'Boster Nexus PIM — Tables & Mapping',
  description:
    'Authoritative reference for the three PIM tables (pim_products, ' +
    'pim_attribute_definitions, pim_product_images) and the mapProductInfo ' +
    'function that hydrates them from a Magento internal-export CSV. Use ' +
    'before reading/writing PIM data or designing a new ingest pipeline.',

  toc: {
    overview: {
      description: 'High-level: what the PIM is, the three tables, the mapping function',
    },
    pimProducts: {
      description: 'pim_products — main catalog (1 row per sku); column-by-column structure',
    },
    pimAttributeDefinitions: {
      description: 'pim_attribute_definitions — per-template resolver for attr_1..attr_30',
    },
    pimProductImages: {
      description: 'pim_product_images — normalized image rows linked to pim_products',
    },
    hydrateVendorImagesOnBundleAndSavePDP: {
      description: 'When vendor images do not render on the live bundle-and-save PDP (NitroCDN 404s, classDev.php [X] strip hazards, Magento Slick gallery init failure), the full procedure lives in the bosterbio skill book under the same action name. See bosterbio.hydrateVendorImagesOnBundleAndSavePDP.',
    },
    pimSyncProtocols: {
      description: 'pim_sync_protocols — registry of named sync protocols between PIM and external systems (migration 109)',
    },
    pimFieldAuthority: {
      description: 'pim_field_authority — per-field ownership rules consulted by sync engines before writes (migration 110)',
    },
    typeABucketing: {
      description: 'Type-A vs Type-B: which Magento columns become dedicated columns vs generic attr_N slots',
    },
    indexVsKeep: {
      description: 'Index vs Keep: same schema treatment; the distinction lives in docs/env vars, not in the table',
    },
    mapProductInfo: {
      description: 'mapProductInfo(input, map?) — the canonical Magento-row → PIM-row converter',
    },
    ingestPipeline: {
      description: 'How to hydrate the PIM from the internal-export CSV end-to-end',
    },
    rollback: {
      description: 'How to reverse a bulk ingest using pim_source tagging',
    },
    migrationHistory: {
      description: 'The migrations that shape the current PIM schema (072 → 102)',
    },
  },

  actions: {
    overview: {
      description: 'High-level: what the PIM is, the three tables, the mapping function',
      howTo: `
The Boster Nexus PIM is the canonical product catalog inside boster_nexus's
Supabase project. The Boster website (Magento) is source of truth during the
migration; PIM becomes truth after cutover.

Three tables form the PIM, all namespaced under the \`pim_\` prefix:

  pim_products              — one row per sku, the main catalog
                              (renamed from \`products\` in migration 102)
  pim_attribute_definitions — per-template meaning of pim_products.attr_1..attr_30
                              (renamed from \`attribute_definitions\` in migration 102)
  pim_product_images        — normalized images per pim_products row (1:N)
                              (renamed from \`product_images\` in migration 102)

Plus one bridge / pipeline table outside this skill book's scope:
  pim_ingestion_jobs        — ingest run records (migration 093)

The MAPPING FUNCTION that converts Magento product rows into PIM rows is
mapProductInfo(input, map?). It lives in two locations that stay byte-identical:

  GuildOS weapon : libs/weapon/pim/index.js   (action: mapProductInfo)
  Boster Nexus   : libs/pim/mapProductInfo.js  (verbatim mirror)

Both export { mapProductInfo, DEFAULT_MAP }. Contract: single object in → single
object out, array in → array out. The default map encodes the locked Boster
decision tables (Type-A buckets + Type-B per-template attr_x assignment).

Touch point for the live ingest:
  boster_nexus/scripts/ingest-internal-export.mjs
    → reads CSV via libs/pim/adapters/csv.js (streaming, quote-aware)
    → maps each row via mapProductInfo
    → batch UPSERTs into pim_products on sku
    → DELETE-then-INSERT into pim_product_images for affected product_ids
    → tags every row with pim_source = 'internal_csv_<YYYY-MM-DD>'
`,
    },

    pimProducts: {
      description: 'pim_products — main catalog (1 row per sku); column-by-column structure',
      howTo: `
\`\`\`
TABLE public.pim_products (renamed from products in migration 102)

  Identity:
    id                   UUID PK (gen_random_uuid)
    sku                  TEXT NOT NULL UNIQUE       — upsert key
    title                TEXT NOT NULL
    handle               TEXT UNIQUE                — URL slug
    product_template     TEXT                       — antibodies | elisa-kits | proteins | ...

  Indexed Type-A (search-critical):
    reactivity           TEXT[]  NOT NULL DEFAULT '{}'
    applications         TEXT[]  NOT NULL DEFAULT '{}'
    clone                TEXT
    host_species         TEXT
    badges               TEXT[]  NOT NULL DEFAULT '{}'
    target_info          JSONB   NOT NULL DEFAULT '{}'   — {gene_name, uniprot_id, synonyms, protein_name, gene_full_name, ...}
    search_index         TEXT

  Other Type-A (dedicated columns, Keep bucket):
    price                NUMERIC(12,2)
    status               TEXT NOT NULL DEFAULT 'draft' — enabled | disabled | draft
    visibility           TEXT
    storage              TEXT
    shipping_storage_handling TEXT
    background           TEXT
    description          TEXT
    short_description    TEXT
    meta_title           TEXT
    meta_description     TEXT
    meta_keywords        TEXT
    size                 TEXT                       — (migration 099)
    immunogen            TEXT
    research_category    TEXT
    competitor_equivalent_skus TEXT[] NOT NULL DEFAULT '{}'
    ribbons              TEXT[] NOT NULL DEFAULT '{}'
    source_company       TEXT
    product_category     TEXT
    custom_listing_information TEXT
    sensitivity          TEXT                       — promoted from elisa-kits attr_1 in 099
    assay_range          TEXT                       — promoted from elisa-kits attr_2 in 099
    sample_type          TEXT                       — promoted from elisa-kits attr_3 in 099
    transfer_price       NUMERIC(12,2)
    category_ids         TEXT
    rating_value         NUMERIC(5,2)               — 0–100 scale (NOT 0–5 stars), widened in 101

  Type-B generic slots (meaning per template via pim_attribute_definitions):
    attr_1 .. attr_30    TEXT      — attr_26..attr_30 added in 099 for antibodies (27 slots)

  Catch-alls / bridges:
    options              JSONB NOT NULL DEFAULT '[]'   — variant options array
    documents            JSONB NOT NULL DEFAULT '[]'   — vendor PDFs / datasheets / SDS;
                                                          shape: [{label, url, type, description}];
                                                          added in 111 (2026-05-28);
                                                          backfilled from metadata.unmapped.documents;
                                                          authority for vendor doc links going forward
    metadata             JSONB NOT NULL DEFAULT '{}'   — anything not modeled elsewhere
    external_ids         JSONB NOT NULL DEFAULT '{}'   — platform → external-id map (e.g. {"boster_magento":"77600"});
                                                          hydrated by scripts/hydrate-pim-external-ids-magento.mjs;
                                                          column added in 092, shape stabilized 2026-05-23
    pim_source           TEXT                          — which ingest pipeline wrote this row
                                                       (e.g. 'internal_csv_2026-05-23' — used for rollback by source)
    created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()
    updated_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()   — trigger keeps fresh
\`\`\`

Indexes today (kept generic until env-var-driven index config lands):
  idx_pim_products_template    (B-tree)
  idx_pim_products_status      (B-tree)
  idx_pim_products_reactivity_gin     (GIN on TEXT[])
  idx_pim_products_applications_gin   (GIN on TEXT[])
  idx_pim_products_gene_name          (B-tree on (target_info->>'gene_name'))

Current row count post-internal-export hydration (2026-05-23): 83,606.
`,
    },

    pimAttributeDefinitions: {
      description: 'pim_attribute_definitions — per-template resolver for attr_1..attr_30',
      howTo: `
\`\`\`
TABLE public.pim_attribute_definitions (renamed from attribute_definitions in migration 102)

  id                   UUID PK (gen_random_uuid)
  product_template     TEXT NOT NULL                 — must match pim_products.product_template
  attr_index           SMALLINT NOT NULL             — which attr_N column (CHECK 1..30)
  attr_key             TEXT NOT NULL                 — snake_case machine name (e.g. 'kit_components')
  attr_label           TEXT NOT NULL                 — display label
  data_type            TEXT NOT NULL DEFAULT 'text'  — text | numeric | date | enum | url (hint; storage stays TEXT)
  source_field         TEXT                          — Magento attribute_code for ingestion
  description          TEXT
  display_order        INT
  searchable           BOOLEAN NOT NULL DEFAULT FALSE   — feeds pim_products.search_index when true
  created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()
  updated_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()

  UNIQUE (product_template, attr_index)
  UNIQUE (product_template, attr_key)
  CHECK  (attr_index BETWEEN 1 AND 30)
\`\`\`

The same attr_N slot can have different meaning for different templates. Example:
  pim_attribute_definitions['antibodies', 9]   → attr_key = 'contents'
  pim_attribute_definitions['elisa-kits', 9]   → attr_key = 'kit_components'

Current state (post-migration 100): 237 rows across 23 templates. The seed comes
from docs/pim/attr-mapping-2-nameless-attr-assignment.tsv; that TSV is reference
material, NOT loaded at runtime. The runtime source of truth for the mapping
is the DEFAULT_MAP.typeB constant inside libs/pim/mapProductInfo.js (and its
GuildOS mirror).

Per-template slot count (descending):
  antibodies:                   27 slots  (the only one > 25)
  custom-description:           25
  elisa-kits:                   20
  antibody-quick-elisa-kits:    18
  tag-quick-elisa-kits:         18
  quick-elisa-kits:             17
  ez-set:                       16
  proteins:                     15
  elisa-kits-custom-components: 14
  reporter-cell-lines:          10
  isotype-control-antibodies:    8
  cell-based-phospho-elisa-kits: 7
  over-expression-lysates:       7
  cell-based-elisa-kits:         6
  veterinary-diagnostic-kits:    6
  real-time-pcr-machine:         5
  detection-kits:                4
  elisa-plate-reader:            4
  secondary-antibodies:          4
  dna-rna-extraction-kit:        2
  instruments-and-machines:      2
  consumables:                   1
  multiplex-elisa-kits:          1
`,
    },

    pimProductImages: {
      description: 'pim_product_images — normalized image rows linked to pim_products',
      howTo: `
\`\`\`
TABLE public.pim_product_images (renamed from product_images in migration 102)

  id              UUID PK (gen_random_uuid)
  product_id      UUID NOT NULL REFERENCES pim_products(id) ON DELETE CASCADE
  url             TEXT NOT NULL
  storage_path    TEXT                          — Supabase storage path if mirrored locally; NULL if served from source URL
  alt_text        TEXT
  position        INT NOT NULL DEFAULT 0        — display order; lower = earlier
  role            TEXT                          — primary | gallery | datasheet | sds | manual_upload
  source          TEXT                          — magento | manual | ai_generated | partner
  source_id       TEXT                          — stable id in source system (idempotent ingest)
  width, height   INT
  mime_type       TEXT
  metadata        JSONB NOT NULL DEFAULT '{}'
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
\`\`\`

Indexes:
  idx_pim_product_images_product  (B-tree on product_id)
  idx_pim_product_images_role     (B-tree on role)
  uq_pim_product_images_source    (UNIQUE on (product_id, source, source_id) WHERE source AND source_id IS NOT NULL)

Ingest strategy is delete-then-insert per affected product_id (kept idempotent;
re-running the same source replaces the prior image set cleanly).

Current row count post-internal-export hydration (2026-05-23): 149,435.
`,
    },

    pimSyncProtocols: {
      description: 'pim_sync_protocols — registry of named sync protocols between PIM and external systems (migration 109)',
      howTo: `
\`\`\`
TABLE public.pim_sync_protocols (migration 109, added 2026-05-27)

  id              UUID PK (gen_random_uuid)
  name            TEXT NOT NULL UNIQUE          — protocol identifier (e.g. 'liveSite-upsert')
  description     TEXT
  source_system   TEXT NOT NULL                  — 'pim' | 'web' | 'cdb' | 'zoho-books' | 'zoho-crm' | 'live-magento'
  target_system   TEXT NOT NULL                  — same enum
  job_table       TEXT NOT NULL DEFAULT 'pim_jobs'  — which queue table backs this protocol
  worker_label    TEXT NOT NULL                  — value of pim_jobs.worker this protocol claims
  schedule_cron   TEXT                           — NULL = trigger-only (no auto-tick); cron string = scheduled poll
  active          BOOLEAN NOT NULL DEFAULT true
  notes           TEXT
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()

Indexes:
  idx_pim_sync_protocols_active   (partial: active = true)
\`\`\`

This is a REGISTRY. It does NOT yet drive dispatch — pim_jobs.worker continues
to be the dispatch key. worker_label simply RECORDS which value of
pim_jobs.worker a protocol claims, so a future refactor can add
pim_jobs.protocol_id and route off this table.

Cross-references:
  - protocols are queued via pim_jobs (worker_label maps to pim_jobs.worker)
  - field-level write rules live in pim_field_authority — sync engines consult
    it BEFORE writes to avoid corrupting authoritative data

Phase A seeds (4 rows; idempotent seeder at scripts/_seed-sync-substrate.mjs):
  liveSite-upsert         pim -> live-magento  (worker=liveSite, trigger-only)
  liveSite-delete         pim -> live-magento  (worker=liveSite, trigger-only)
  liveSite-disable        pim -> live-magento  (worker=liveSite, trigger-only)
  liveSite-imagesUpdate   pim -> live-magento  (worker=liveSite, trigger-only)

Read endpoint:
  GET /pim/sync/list/api  — returns { ok, protocols, field_authority }
  In-app viewer at /pim/admin?tab=protocols (read-only Sync Protocols tab).

How to register a new protocol (no code change required):
  INSERT INTO pim_sync_protocols (name, source_system, target_system,
    worker_label, schedule_cron, notes)
  VALUES ('web-cdb-sync', 'web', 'cdb', 'webToCdb', NULL,
    'Future: nightly sync of website -> CDB');
`,
    },

    pimFieldAuthority: {
      description: 'pim_field_authority — per-field ownership rules consulted by sync engines before writes (migration 110)',
      howTo: `
\`\`\`
TABLE public.pim_field_authority (migration 110, added 2026-05-27)

  id              UUID PK (gen_random_uuid)
  field_path      TEXT NOT NULL                  — e.g. 'pim_products.price', 'pim_products.inventory_*', '*'
  authority       TEXT NOT NULL                  — 'pim' | 'zoho-books' | 'website' | 'live-magento' | 'sync-worker' | ...
  condition_when  JSONB DEFAULT '{}'             — optional row condition (e.g. {"product_template":"antibodies"}); empty = unconditional
  priority        INT NOT NULL DEFAULT 0         — higher wins when multiple rules match same field
  notes           TEXT
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()

Indexes:
  idx_pim_field_authority_path       (B-tree on field_path)
  idx_pim_field_authority_priority   (B-tree on priority DESC)
\`\`\`

Each row is a write-permission rule for a (system, field) pair. Sync engines
should consult this table BEFORE writing — if you are not the authority for a
field, do not overwrite it.

field_path conventions:
  '*'                                     — global default catch-all
  'pim_products.price'                    — exact column
  'pim_products.inventory_*'              — prefix wildcard (engine-side glob)
  'pim_products.target_info.gene_name'    — JSON path inside JSONB

Match algorithm (engine-side; not enforced in DB):
  1. Filter rows whose field_path matches the target field
     (exact > prefix > '*'); apply any condition_when against the row.
  2. Order matching rows by priority DESC.
  3. The top row's authority IS the authoritative system.
  4. If authority !== caller, skip the write (or log a violation).

Phase A seeds (6 rows; idempotent seeder at scripts/_seed-sync-substrate.mjs):
  pri  field_path                                  authority
    0  *                                           pim
   10  pim_products.inventory_*                    zoho-books
   10  pim_products.usage_*                        website
   10  pim_products.external_ids.boster_magento    live-magento
   10  pim_jobs.status                             sync-worker
   10  pim_jobs.result                             sync-worker

Read endpoint:
  GET /pim/sync/list/api  — returns { ok, protocols, field_authority }
  In-app viewer at /pim/admin?tab=protocols.

Adding a rule (no code change required):
  INSERT INTO pim_field_authority (field_path, authority, priority, notes)
  VALUES ('pim_products.price', 'zoho-books', 20, 'Price authoritative in Zoho');
`,
    },

    typeABucketing: {
      description: 'Type-A vs Type-B: which Magento columns become dedicated columns vs generic attr_N slots',
      howTo: `
Two storage strategies for product attributes:

  TYPE A — dedicated column on pim_products
    · Columns the schema knows about by name (sku, title, price, status,
      reactivity, applications, target_info JSONB, etc.).
    · Query-friendly; can be indexed; safe to JOIN on.
    · Shared across templates — meaning is the same for every product.
    · Examples: sku, title, handle, status, price, host_species, sensitivity,
      assay_range, sample_type, target_info.gene_name.

  TYPE B — generic slot attr_N on pim_products + pim_attribute_definitions row
    · The MEANING of attr_5 depends on the row's product_template.
    · Resolved via pim_attribute_definitions[template][attr_index].attr_key.
    · Use this for fields that are template-specific (e.g. 'kit_components'
      means something for elisa-kits but doesn't apply to antibodies the same way).
    · attr_1..attr_30 available (extended from 25 to 30 in migration 099 to
      accommodate antibodies' 27-slot mapping).

DECISION RULE (encoded in libs/weapon/pim/index.js DEFAULT_MAP):
  fields:       map.fields[<magento_col>] → Type-A column
  typeB:        map.typeB[<product_template>][<magento_col>] → Type-B attr_N slot
  relation:     fields[<magento_col>].relation = 'images' | 'options'
                → handled by ingest post-processor (writes to pim_product_images
                or pim_products.options JSONB)

Reference docs (NOT loaded at runtime; the runtime constant is DEFAULT_MAP):
  boster_nexus/docs/pim/attr-mapping-1-index-keep-nameless.tsv
  boster_nexus/docs/pim/attr-mapping-2-nameless-attr-assignment.tsv
`,
    },

    indexVsKeep: {
      description: 'Index vs Keep: same schema treatment; the distinction lives in docs/env vars, not in the table',
      howTo: `
"Index" and "Keep" are sub-buckets within Type-A. Both result in a dedicated
column on pim_products — they are stored identically at the schema level.

  Index bucket : "I'll filter/sort/search by this column." Gets a DB index
                 (B-tree, GIN, or trigram depending on type). Examples: sku,
                 title, host_species, applications[], gene_name.

  Keep bucket  : "Store the value, don't index it." Cheaper writes; fewer index
                 maintenance costs. Examples: meta_title, background, status,
                 source_company.

The Index/Keep decision is METADATA that lives in:
  - NOW:  docs/pim/attr-mapping-1-index-keep-nameless.tsv
  - LATER: env vars on the new website (to be enabled per-environment)

It is NOT encoded in DEFAULT_MAP. The mapper doesn't care which fields are
indexed; it just writes the canonical row. Index creation is a separate
concern, deferred from migration 099 (no CREATE INDEX statements were added
there for the new columns — intentional, awaiting env-var-driven config).
`,
    },

    mapProductInfo: {
      description: 'mapProductInfo(input, map?) — the canonical Magento-row → PIM-row converter',
      howTo: `
Two byte-identical implementations:
  GuildOS weapon  : libs/weapon/pim/index.js
  Boster Nexus    : libs/pim/mapProductInfo.js

Both export { mapProductInfo, DEFAULT_MAP }. Treat them as the same code — when
one changes, the other MUST change in the same commit (verbatim mirror, not a
fork). A round-trip equivalence test asserts native ≡ weapon.

Contract:
  mapProductInfo(input, map?) -> output
    input  : single Magento product row (object) OR an array of rows
    map    : optional. When omitted/null/undefined, DEFAULT_MAP is used.
    output : matches input shape — single → single, array → array

Each output is a canonical-PIM row shaped like pim_products + a non-enumerable
\`_unmapped\` audit map carrying any source keys the map didn't know about.

Tested at: GuildOS/app/(town)/proving-grounds/pim-mapping-test (paste from
Excel, see contract badges + result JSON before committing to a full ingest).

Transforms available in DEFAULT_MAP field specs:
  identity, trim, toNull, splitList (CSV-multi-value → TEXT[]),
  splitListPipe (pipe-only, for image labels), joinList,
  magentoStatusToCanonical (1→enabled, 2→disabled),
  canonicalStatusToMagento (inverse), numeric (→ JS Number),
  parseCustomOptions (Magento custom_options string → JSONB array)
`,
    },

    ingestPipeline: {
      description: 'How to hydrate the PIM from the internal-export CSV end-to-end',
      howTo: `
End-to-end live path that hydrated pim_products to 83,606 rows on 2026-05-23:

  CSV    : C:/Users/xsj70/Downloads/internal-export.csv  (~363 MB)
  Stream : boster_nexus/libs/pim/adapters/csv.js
           → state-machine parser, memory-bounded by largest single row
           → preserves multi-line quoted cells (description, custom_options)
           → applies backpressure when consumer is slower than disk read
  Map    : boster_nexus/libs/pim/mapProductInfo.js
           → DEFAULT_MAP (Type-A 42 fields + Type-B 23 templates)
  Sink   : pim_products UPSERT on sku (batch size 200)
           pim_product_images DELETE-then-INSERT per affected product_id
  Tag    : pim_source = 'internal_csv_<YYYY-MM-DD>'  (every row)
  Driver : boster_nexus/scripts/ingest-internal-export.mjs

Driver flags:
  --path=<csv>          override input file
  --limit=N             stop after N source rows (0 = no limit)
  --skip=N              skip first N source rows (resume / chunking)
  --batch-size=N        UPSERT batch size (default 200)
  --dry-run             parse + map + report telemetry, no DB writes
  --source-tag=<str>    override pim_source value

Runs FOREGROUND in chunks when wall-clock exceeds the Bash tool's 10-min cap:
  node --max-old-space-size=8192 scripts/ingest-internal-export.mjs --limit=30000
  node --max-old-space-size=8192 scripts/ingest-internal-export.mjs --skip=30000 --limit=30000
  node --max-old-space-size=8192 scripts/ingest-internal-export.mjs --skip=60000

After a bulk ingest: \`ANALYZE pim_products; ANALYZE pim_product_images;\` to
refresh planner statistics so the Supabase Table Editor's row-count estimate
matches reality.

NOT-NULL gotcha: Supabase batch UPSERT puts ALL columns from ALL rows in the
INSERT column list. Even if one row omits \`badges\`, another row's badges
forces Postgres to substitute NULL for the missing row's badges, which
violates NOT NULL. The ingest script's pickProductRow backfills schema
defaults for array/JSONB NOT-NULL columns (reactivity, applications, badges,
competitor_equivalent_skus, ribbons, target_info, options, metadata, external_ids).
Do the same in any new ingest script.
`,
    },

    rollback: {
      description: 'How to reverse a bulk ingest using pim_source tagging',
      howTo: `
Every ingest run tags rows with \`pim_source\` (e.g. 'internal_csv_2026-05-23').
To undo a specific ingest:

  -- Order matters: delete child images first, THEN parent products.
  DELETE FROM pim_product_images
   WHERE product_id IN (
     SELECT id FROM pim_products WHERE pim_source = 'internal_csv_2026-05-23'
   );
  DELETE FROM pim_products
   WHERE pim_source = 'internal_csv_2026-05-23';

Or rely on FK cascade (pim_product_images.product_id has ON DELETE CASCADE):
  DELETE FROM pim_products WHERE pim_source = 'internal_csv_2026-05-23';

This deletes ONLY rows from that ingest, leaving other-source rows untouched.

For an UPSERT that updated existing rows, rollback can't recover the pre-UPSERT
values — the previous-value diff isn't kept. Take a snapshot before any
destructive bulk operation if you need true undo.
`,
    },

    migrationHistory: {
      description: 'The migrations that shape the current PIM schema (072 → 102)',
      howTo: `
Boster Nexus PIM migration timeline (all in supabase/migrations/):

  072  create_products.sql
       → Initial products table (Type-A columns + attr_1..attr_25 + JSONB).

  076  attribute_definitions_and_product_images.sql
       → attribute_definitions (per-template attr_N resolver).
       → product_images (normalized 1:N images, FK to products).

  092  products_external_ids_bridge.sql
       → Added products.external_ids JSONB column for cross-platform IDs.
       → Current shape: { "<platform>": "<external_id>", ... }
         (e.g. { "boster_magento": "77600" }). Other platforms can be added
         alongside (zoho_books, crm, gmc, etc.) as those integrations land.
       → Hydration script: scripts/hydrate-pim-external-ids-magento.mjs
         (populates boster_magento from a sku→entity_id TSV dump of the
          live Magento catalog).

  093  create_pim_ingestion_jobs.sql
       → pim_ingestion_jobs table for ingest-run telemetry (separate concern;
         not part of this skill book's core three tables).

  099  pim_promote_and_expand.sql
       → Added 14 new Type-A dedicated columns (size, immunogen,
         research_category, competitor_equivalent_skus, ribbons,
         source_company, product_category, custom_listing_information,
         sensitivity, assay_range, sample_type, transfer_price, category_ids,
         rating_value).
       → Promoted sensitivity / assay_range / sample_type from elisa-kits
         attr_1..attr_3 to dedicated columns.
       → Added attr_26..attr_30 (extended slot count 25 → 30 for antibodies).
       → Bumped attribute_definitions CHECK constraint to (1..30).

  100  attribute_definitions_seed.sql
       → Deleted the 12 elisa-kits seed rows from 076.
       → Inserted 237 rows across 23 templates from docs/pim/attr-mapping-2.tsv.

  101  widen_rating_value.sql
       → Widened products.rating_value from NUMERIC(3,2) to NUMERIC(5,2)
         after observing rating=100 in the internal-export (Magento uses
         0–100 scale, not 0–5 stars).

  102  rename_pim_tables.sql
       → products → pim_products
       → attribute_definitions → pim_attribute_definitions
       → product_images → pim_product_images
       → Renamed associated indexes + constraints for consistency.
       → Data-preserving; no row-level changes.

  109  pim_sync_protocols.sql  (2026-05-27)
       → New table: pim_sync_protocols (registry of named sync protocols).
       → Phase A of the operator's 2026-05-27 PIM vendor-ingestion refactor.
       → Registry only; pim_jobs.worker remains the dispatch key.
       → Seeded with 4 liveSite-* rows by scripts/_seed-sync-substrate.mjs.

  110  pim_field_authority.sql  (2026-05-27)
       → New table: pim_field_authority (per-field ownership rules).
       → Sync engines consult before writes to avoid corrupting
         authoritative data.
       → Seeded with 6 rows (pim default + zoho-books/website/live-magento/
         sync-worker overrides) by scripts/_seed-sync-substrate.mjs.
`,
    },
  },
};
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/pim/index.js -->
```js
/**
 * pim weapon — GuildOS-side Magento-row → canonical-PIM converter.
 *
 * Single public action: mapProductInfo(input, map?). Takes one Magento product
 * row (or an array of rows) and returns the canonical-PIM shape, applying the
 * locked DEFAULT_MAP (Type-A buckets + Type-B per-template attr_x assignment)
 * or a caller-supplied override. Same shape in → out (object→object, array→array).
 *
 * Full contract, table schemas, bucketing rules, and the mirror plan live in the
 * `pim` skill book → libs/skill_book/pim/index.js (action: mapProductInfo).
 * This weapon is mirrored verbatim into boster_nexus/libs/pim/mapProductInfo.js.
 */

const TRANSFORMS = {
  identity: (v) => v,
  trim: (v) => (v == null ? v : String(v).trim()),
  toNull: (v) => (v === "" || v === undefined ? null : v),
  splitList: (v) =>
    v == null || v === ""
      ? []
      : String(v)
          .split(/[,;|]/)
          .map((s) => s.trim())
          .filter(Boolean),
  // Image labels in Magento are pipe-separated only (commas appear inside labels).
  splitListPipe: (v) =>
    v == null || v === ""
      ? []
      : String(v)
          .split(/\|/)
          .map((s) => s.trim())
          .filter(Boolean),
  joinList: (v) => (Array.isArray(v) ? v.join(",") : v == null ? "" : String(v)),
  magentoStatusToCanonical: (v) => {
    const s = String(v ?? "").trim();
    if (s === "1") return "enabled";
    if (s === "2") return "disabled";
    if (s === "enabled" || s === "disabled" || s === "draft") return s;
    return "draft";
  },
  canonicalStatusToMagento: (v) => (v === "enabled" ? "1" : "0"),
  numeric: (v) =>
    v == null || v === ""
      ? null
      : Number.isFinite(Number(v))
        ? Number(v)
        : String(v),
  // Parse Magento's pipe-delimited custom_options string into a JSONB-friendly
  // array. Each option is a string of `k=v` pairs joined by `,`; sub-options
  // are joined by `|`. We do a best-effort parse — anything that looks
  // unparseable is preserved verbatim under `__raw` so the next stage can fix
  // it without data loss.
  parseCustomOptions: (v) => {
    if (v == null || v === "") return [];
    const out = [];
    for (const optStr of String(v).split(/\|/)) {
      if (!optStr.trim()) continue;
      const obj = {};
      for (const kv of optStr.split(/,(?=[a-z_]+=)/i)) {
        const m = kv.match(/^([a-z_][a-z0-9_]*)=(.*)$/i);
        if (m) obj[m[1]] = m[2];
      }
      if (Object.keys(obj).length === 0) obj.__raw = optStr;
      out.push(obj);
    }
    return out;
  },
};

// ───────────────────────────────────────────────────────────────────────────
// mapProductInfo — Magento-row → canonical-PIM converter (the only public path).
//
// DEFAULT_MAP below is the locked source of truth at runtime (Type-A/Keep/
// Nameless/Relation buckets + per-template attr_x assignment). It is NOT loaded
// from JSON/TSV/DB; the TSVs in boster_nexus/docs/pim/ are reference only.
//
// Full contract, table schemas, bucketing rules, transforms, and the verbatim-
// mirror plan (→ boster_nexus/libs/pim/mapProductInfo.js): see the `pim` skill
// book at libs/skill_book/pim/index.js (action: mapProductInfo).
// ───────────────────────────────────────────────────────────────────────────

/**
 * Default PIM mapping. Encodes the two locked decision tables that live in
 * the boster_nexus repo (docs/pim/attr-mapping-1.tsv + attr-mapping-2.tsv).
 * THIS IS THE SOURCE OF TRUTH AT RUNTIME — the TSV files are documentation.
 *
 * Shape:
 *   fields: { <magento_col>: { to, transform?, relation? } }
 *     - to:        canonical destination path; dotted path supported
 *                  (e.g. 'target_info.gene_name', 'metadata.source_created_at')
 *     - transform: key in TRANSFORMS above (trim, splitList, numeric, …)
 *     - relation:  'images' | 'options' — handled by relation post-processors
 *                  in the ingest pipeline (not by this function)
 *
 *   typeB: { <product_template>: { <magento_col>: <attr_index>, ... } }
 *     Per-template assignment of "nameless" Magento columns into the generic
 *     attr_1..attr_30 slots on the products row.
 *
 * NOTE: Index-vs-Keep bucket distinction (which columns get a DB index) is
 * NOT encoded in this map. Index/Keep columns are added to the products
 * table the SAME WAY at the schema level. The Index decision is metadata
 * that lives in docs/pim/attr-mapping-1-index-keep-nameless.tsv now and
 * will move to env vars on the new website later. The converter doesn't
 * need to know which fields are indexed.
 */
export const DEFAULT_MAP = {
  fields: {
    // Identity / handle
    sku:                          { to: "sku",                                 transform: "trim" },
    name:                         { to: "title",                               transform: "trim" },
    url_key:                      { to: "handle",                              transform: "trim" },

    description:                  { to: "description" },
    short_description:            { to: "short_description" },
    storage:                      { to: "storage",                             transform: "trim" },
    size:                         { to: "size",                                transform: "trim" },
    applications:                 { to: "applications",                        transform: "splitList" },
    reactivity:                   { to: "reactivity",                          transform: "splitList" },
    host:                         { to: "host_species",                        transform: "trim" },
    immunogen:                    { to: "immunogen" },
    research_category:            { to: "research_category",                   transform: "trim" },
    competitor_equivalent_skus:   { to: "competitor_equivalent_skus",          transform: "splitList" },
    badges:                       { to: "badges",                              transform: "splitList" },
    clone_number:                 { to: "clone",                               transform: "trim" },
    ribbons:                      { to: "ribbons",                             transform: "splitList" },

    // target_info JSONB subfields
    gene_name:                    { to: "target_info.gene_name" },
    uniprot_id:                   { to: "target_info.uniprot_id",              transform: "trim" },
    synonyms:                     { to: "target_info.synonyms" },
    gene_full_name:               { to: "target_info.gene_full_name" },
    protein_name:                 { to: "target_info.protein_name" },

    // (Index-vs-Keep distinction lives in docs/env vars, not here.)
    source_company:               { to: "source_company",                      transform: "trim" },
    product_category:             { to: "product_category",                    transform: "trim" },
    meta_title:                   { to: "meta_title" },
    meta_description:             { to: "meta_description" },
    meta_keyword:                 { to: "meta_keywords" },
    custom_listing_information:   { to: "custom_listing_information" },
    sensitivity:                  { to: "sensitivity" },                                          // promoted from elisa-kits attr_1
    background:                   { to: "background" },
    assay_range:                  { to: "assay_range" },                                          // promoted from elisa-kits attr_2
    sample_type:                  { to: "sample_type" },                                          // promoted from elisa-kits attr_3
    status:                       { to: "status",                              transform: "magentoStatusToCanonical" },
    price:                        { to: "price",                               transform: "numeric" },
    transfer_price:               { to: "transfer_price",                      transform: "numeric" },
    created_at:                   { to: "metadata.source_created_at" },
    updated_at:                   { to: "metadata.source_updated_at" },
    category_ids:                 { to: "category_ids" },
    rating_value:                 { to: "rating_value",                        transform: "numeric" },
    template:                     { to: "product_template",                    transform: "trim" },

    // ── Relation: Images (writes to product_images table downstream) ───────
    images:                       { to: "_relations.images",                   transform: "splitList",                  relation: "images" },
    image_labels:                 { to: "_relations.image_labels",             transform: "splitListPipe",              relation: "images" },

    // ── Relation: Options (parsed into products.options JSONB downstream) ──
    custom_options:               { to: "options",                             transform: "parseCustomOptions",         relation: "options" },
  },

  typeB: {
    "antibodies": {
      custom_attribute_1: 1, custom_attribute_2: 2, custom_attribute_3: 3,
      application_details: 4, application_notes: 5, clonality: 6,
      concentration: 7, conjugate: 8, contents: 9, cross_reactivity: 10,
      description_after_attributes: 11, description_before_attributes: 12,
      form: 13, isotype: 14, kit_components: 15, molecular_weight: 16,
      phospho_site: 17, predicted_reactivity: 18, protein_function: 19,
      purification: 20, recommended_detection_systems: 21, reconstitution: 22,
      reproducibility: 23, sample_data: 24, sequence_similarities: 25,
      subcellular_localization: 26, tissue_specificity: 27,
    },
    "antibody-quick-elisa-kits": {
      custom_attribute_1: 1, custom_attribute_3: 2, anticoagulant: 3,
      assay_procedure_step_1: 4, cross_reactivity: 5,
      description_before_attributes: 6, intra_inter_assay_cv: 7,
      kit_components: 8, molecular_weight: 9, normal_range_citations: 10,
      principle: 11, protein_function: 12, reproducibility: 13,
      sample_data: 14, sequence_similarities: 15,
      subcellular_localization: 16, tissue_specificity: 17,
      tmb_incubation_time: 18,
    },
    "cell-based-elisa-kits": {
      custom_attribute_1: 1, kit_components: 2, molecular_weight: 3,
      protein_function: 4, subcellular_localization: 5,
      tissue_specificity: 6,
    },
    "cell-based-phospho-elisa-kits": {
      custom_attribute_1: 1, custom_attribute_2: 2, kit_components: 3,
      molecular_weight: 4, protein_function: 5,
      subcellular_localization: 6, tissue_specificity: 7,
    },
    "consumables": {
      custom_attribute_1: 1,
    },
    "custom-description": {
      custom_attribute_1: 1, custom_attribute_3: 2, application_details: 3,
      application_notes: 4, clonality: 5, concentration: 6, conjugate: 7,
      contents: 8, cross_reactivity: 9, description_after_attributes: 10,
      description_before_attributes: 11, form: 12, isotype: 13,
      kit_components: 14, molecular_weight: 15, phospho_site: 16,
      predicted_reactivity: 17, protein_function: 18, purification: 19,
      recommended_detection_systems: 20, reconstitution: 21,
      reproducibility: 22, source: 23, subcellular_localization: 24,
      tissue_specificity: 25,
    },
    "detection-kits": {
      clonality: 1, conjugate: 2, form: 3, kit_components: 4,
    },
    "dna-rna-extraction-kit": {
      description_after_attributes: 1, description_before_attributes: 2,
    },
    "elisa-kits": {
      anticoagulant: 1, application_details: 2, application_notes: 3,
      assay_procedure_step_1: 4, clonality: 5, cross_reactivity: 6,
      description_before_attributes: 7, intra_inter_assay_cv: 8,
      kit_components: 9, molecular_weight: 10, normal_range_citations: 11,
      predicted_reactivity: 12, principle: 13, protein_function: 14,
      reproducibility: 15, sample_data: 16, sequence_similarities: 17,
      subcellular_localization: 18, tissue_specificity: 19,
      tmb_incubation_time: 20,
    },
    "elisa-kits-custom-components": {
      application_details: 1, assay_procedure_step_1: 2, cross_reactivity: 3,
      description_after_attributes: 4, description_before_attributes: 5,
      isotype: 6, kit_components: 7, predicted_reactivity: 8,
      protein_function: 9, reproducibility: 10, sample_data: 11,
      subcellular_localization: 12, tissue_specificity: 13,
      tmb_incubation_time: 14,
    },
    "elisa-plate-reader": {
      custom_attribute_1: 1, custom_attribute_3: 2, custom_attribute_4: 3,
      custom_attribute_5: 4,
    },
    "ez-set": {
      anticoagulant: 1, assay_procedure_step_1: 2, cross_reactivity: 3,
      description_before_attributes: 4, intra_inter_assay_cv: 5,
      kit_components: 6, molecular_weight: 7, normal_range_citations: 8,
      principle: 9, protein_function: 10, reproducibility: 11,
      sample_data: 12, sequence_similarities: 13,
      subcellular_localization: 14, tissue_specificity: 15,
      tmb_incubation_time: 16,
    },
    "instruments-and-machines": {
      custom_attribute_1: 1, description_before_attributes: 2,
    },
    "isotype-control-antibodies": {
      custom_attribute_1: 1, custom_attribute_2: 2, custom_attribute_3: 3,
      application_notes: 4, concentration: 5, conjugate: 6, contents: 7,
      isotype: 8,
    },
    "multiplex-elisa-kits": {
      description_before_attributes: 1,
    },
    "over-expression-lysates": {
      custom_attribute_1: 1, custom_attribute_2: 2, custom_attribute_3: 3,
      custom_attribute_4: 4, kit_components: 5, source: 6, tag: 7,
    },
    "proteins": {
      custom_attribute_1: 1, concentration: 2, contents: 3,
      description_after_attributes: 4, description_before_attributes: 5,
      form: 6, molecular_weight: 7, protein_function: 8, purification: 9,
      reconstitution: 10, sequence_similarities: 11, source: 12,
      subcellular_localization: 13, tag: 14, tissue_specificity: 15,
    },
    "quick-elisa-kits": {
      custom_attribute_1: 1, anticoagulant: 2, assay_procedure_step_1: 3,
      cross_reactivity: 4, description_before_attributes: 5,
      intra_inter_assay_cv: 6, kit_components: 7, molecular_weight: 8,
      normal_range_citations: 9, principle: 10, protein_function: 11,
      reproducibility: 12, sample_data: 13, sequence_similarities: 14,
      subcellular_localization: 15, tissue_specificity: 16,
      tmb_incubation_time: 17,
    },
    "real-time-pcr-machine": {
      custom_attribute_1: 1, custom_attribute_3: 2, custom_attribute_4: 3,
      custom_attribute_5: 4, custom_attribute_6: 5,
    },
    "reporter-cell-lines": {
      application_details: 1, application_notes: 2, concentration: 3,
      conjugate: 4, contents: 5, description_before_attributes: 6, form: 7,
      isotype: 8, kit_components: 9, purification: 10,
    },
    "secondary-antibodies": {
      clonality: 1, concentration: 2, conjugate: 3,
      description_before_attributes: 4,
    },
    "tag-quick-elisa-kits": {
      custom_attribute_1: 1, custom_attribute_3: 2, anticoagulant: 3,
      assay_procedure_step_1: 4, cross_reactivity: 5,
      description_before_attributes: 6, intra_inter_assay_cv: 7,
      kit_components: 8, molecular_weight: 9, normal_range_citations: 10,
      principle: 11, protein_function: 12, reproducibility: 13,
      sample_data: 14, sequence_similarities: 15,
      subcellular_localization: 16, tissue_specificity: 17,
      tmb_incubation_time: 18,
    },
    "veterinary-diagnostic-kits": {
      custom_attribute_1: 1, custom_attribute_2: 2, custom_attribute_3: 3,
      custom_attribute_4: 4, custom_attribute_5: 5, custom_attribute_6: 6,
    },
  },
};

function applyTransformByName(name, value) {
  if (!name) return value;
  const fn = TRANSFORMS[name];
  if (!fn) throw new Error(`mapProductInfo: unknown transform '${name}'`);
  return fn(value);
}

function setNestedPath(target, dottedPath, value) {
  // Writes `value` at `dottedPath` inside `target`. Creates intermediate
  // objects as needed. Top-level paths fall through to a flat write.
  if (!dottedPath.includes(".")) {
    target[dottedPath] = value;
    return;
  }
  const parts = dottedPath.split(".");
  let cur = target;
  for (let i = 0; i < parts.length - 1; i++) {
    const k = parts[i];
    if (cur[k] == null || typeof cur[k] !== "object" || Array.isArray(cur[k])) {
      cur[k] = {};
    }
    cur = cur[k];
  }
  cur[parts[parts.length - 1]] = value;
}

function isPlainObject(v) {
  return v != null && typeof v === "object" && !Array.isArray(v);
}

/**
 * Convert one Magento product row → canonical-PIM product object.
 * Internal helper; mapProductInfo() handles single-vs-array shape dispatch.
 */
function mapOne(row, map) {
  if (!isPlainObject(row)) {
    throw new Error("mapProductInfo: each row must be a plain object (key→value)");
  }
  const out = {};
  const unmapped = {};
  const template = row.template || row.product_template || null;

  // First pass: Type-A + Relation columns (driven by map.fields).
  const handledKeys = new Set();
  for (const [srcKey, spec] of Object.entries(map.fields || {})) {
    handledKeys.add(srcKey);
    if (!(srcKey in row)) continue;
    let value = row[srcKey];
    if (value === "" || value === undefined) {
      value = null;
    }
    if (value !== null && spec.transform) {
      value = applyTransformByName(spec.transform, value);
    }
    setNestedPath(out, spec.to, value);
  }

  // Second pass: Type-B (Nameless) — per-template attr_N assignment.
  const typeBForTemplate = (map.typeB && template && map.typeB[template]) || null;
  if (typeBForTemplate) {
    for (const [srcKey, attrIndex] of Object.entries(typeBForTemplate)) {
      handledKeys.add(srcKey);
      if (!(srcKey in row)) continue;
      let value = row[srcKey];
      if (value === "" || value === undefined) value = null;
      out[`attr_${attrIndex}`] = value;
    }
  }

  // Third pass: anything left over goes into _unmapped (audit trail; not
  // written to the products table by downstream ingest unless a follow-up
  // map upgrade adopts it).
  for (const key of Object.keys(row)) {
    if (handledKeys.has(key)) continue;
    // Skip known-internal noise that Magento exports sometimes carry.
    if (key === "" || key.startsWith("_")) continue;
    unmapped[key] = row[key];
  }
  if (Object.keys(unmapped).length > 0) {
    Object.defineProperty(out, "_unmapped", {
      value: unmapped,
      enumerable: true,
    });
  }

  // Carry the template through explicitly for downstream routing even if the
  // map didn't include it (defensive — DEFAULT_MAP does include it).
  if (template && !out.product_template) out.product_template = template;

  return out;
}

/**
 * Convert Magento product info (single row or array of rows) to the
 * canonical-PIM shape using a declarative map. See contract block above.
 *
 * @param {object|object[]} input
 * @param {object} [map] - defaults to DEFAULT_MAP when omitted/null/undefined
 * @returns {object|object[]} - same shape as input
 */
export function mapProductInfo(input, map) {
  const resolvedMap = map && typeof map === "object" ? map : DEFAULT_MAP;
  if (!resolvedMap.fields || typeof resolvedMap.fields !== "object") {
    throw new Error("mapProductInfo: map.fields is required");
  }

  if (Array.isArray(input)) {
    return input.map((row) => mapOne(row, resolvedMap));
  }
  if (isPlainObject(input)) {
    return mapOne(input, resolvedMap);
  }
  throw new Error(
    "mapProductInfo: input must be a plain object or an array of objects",
  );
}

```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-SKIPPED-BINARY: libs/weapon/pim/_attr-mapping-1.tsv, libs/weapon/pim/_attr-mapping-2.tsv -->
