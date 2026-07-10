# biomarker

<!-- GUILDOS-SKILL name="biomarker" -->

<!-- GUILDOS-DESC -->
Actions for hydrating gene info card templates with structured external data. Phase 1: zero-LLM; fetches UniProt, Protein Atlas, and Boster catalog. Phase 2 (planned): LLM narrative generation from hydrated data facts. Includes the LOCKED hydration source policy, vendor-ingest indirection, deterministic MW→ protocol rules, PMC caption-proximity search, and the HPA protein-score control source of truth — read these refs before writing any hydration prompt.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/biomarker/SKILL.md -->
```markdown
---
name: biomarker
description: >-
  Actions for hydrating gene info card templates with structured external data.
  Phase 1: zero-LLM; fetches UniProt, Protein Atlas, and Boster catalog. Phase 2
  (planned): LLM narrative generation from hydrated data facts. Includes the
  LOCKED hydration source policy, vendor-ingest indirection, deterministic MW→
  protocol rules, PMC caption-proximity search, and the HPA protein-score control
  source of truth — read these refs before writing any hydration prompt.
---

# biomarker

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/biomarker/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto biomarker <action>` (or read the source directly).

## Actions

- **buildUrlMap** — Build the urlMap for a given biomarker from known IDs (UniProt accession, Ensembl ID, gene symbol)
- **discoverIds** — Discover UniProt accession and Ensembl ID for a gene symbol via UniProt search API
- **hydrateTemplate** — Resolve all script-hydratable slugs via the biomarker weapon and return a hydrated slug map
- **identifyGaps** — Compare a resolved slug map to SLUG_REGISTRY to list literature and LLM slugs still pending
- **refDemoTemplateLiteral** — LOCKED 2026-05-26 — when a MIRROR/renderer mirrors a reference HTML file, emit the exact same outer tag (section vs div) AND inner class names as the reference. The accumulator (Results stack / preview stream) MUST NOT wrap ported chunks in extra container/grid/aside divs — concatenate raw and let the reference CSS do layout. Audit cue: "outer tag of MIRROR.X[0] === outer tag of demo.html sectionX"?
- **refDirectFillVsAgentFill** — LOCKED — hydration fill methods, in order: direct-fill HTML+[slug] tokens, then AI agent fill (JSON-key mapping preferred, example-template fill as fallback).
- **refHpaProteinScoreSource** — LOCKED 2026-05-27 — single source of truth for tissue positive/negative control picks: HPA tissue page Protein-expression-score chart ($("#proteinChart").barChart) on https://www.proteinatlas.org/{ENSG}-{GENE}/tissue. Endpoint: GET /api/pg/hpa-protein-score?gene=X. Returns full tissues[] + ranked sorted.positive_candidates (top 5 by score desc, High/Medium first) + sorted.negative_candidates (top 5 by score asc, Not detected first). Score mapping: value 3.000→High, 2.000→Medium, 1.000→Low, 0→Not detected. Do NOT derive controls from GTEx mRNA, HPA search_download (drops protein cols), or any proxy signal; this endpoint is the ONLY supplier for MIRROR.controls + must-know negative-control row.
- **refHumanReactivityGate** — LOCKED 2026-05-26 — antibody picker MUST verify human is in reactivity[]. If no human-reactive antibody exists for a gene, return out_of_scope=true with the error "Boster does not have a human-reactive antibody for X, so this is out of scope for the Western blot content pipeline." Implemented in /api/pg/picoband-wb.
- **refMwDeterministicRules** — LOCKED deterministic MW → gel% / transfer / membrane / blocking rules; retires R2 (handbook citation).
- **refOmniPathHpa** — LOCKED HPA protein-level evidence via OmniPath /annotations?resources=HPA_tissue (replaces R7 strategy).
- **refPimImagePathOnly** — LOCKED 2026-05-26 — image URLs come ONLY from pim_product_images.url. No SKU-pattern guessing, no URL fabrication. If PIM URL is broken (null, "[object Object]", JSON), the card shows broken-image state — do NOT synthesize a fallback path.
- **refPmcCaptionProximity** — LOCKED v6 (live impl = boster_nexus pmc.js buildQueries): PMC pipeline finds candidates SKU-citation first, then caption-proximity ("<gene> western|immunoblot|immunoblotting"[capt:~30] OR-group), text-word, broad, then body~8 OR-group as a LAST-tier fallback (caption-sparse genes only); RRID rejected (⊆ SKU). Extracts candidates with full-article AI (methods + WB-fig captions + abstract + body intro + supp URLs as context), ranks by fill_count, returns best as presets. No quality gate — partial extractions surfaced honestly with their fill_count. Empirics: memory/reference_wb_pmc_query_tiers.md.
- **refRealScreenshotOnly** — LOCKED 2026-05-27, tool corrected 2026-06-08 — html2canvas / DOM-rendered canvas screenshots are NOT visual verification for biomarker mockup UI work. DEFAULT verification channel = a real CIC screenshot of the rendered result page via mcp__Claude_in_Chrome__* (computer action=screenshot, save_to_disk) — this is the authoritative channel per the browsercontrol skill book; it captures real browser paint, not a synthetic DOM canvas, so it already solves the html2canvas problem. Do NOT default to mcp__computer-use__screenshot (OS-level) — it is NOT the channel, and the user may decline to grant it (declined 2026-06-08). Reason for the no-html2canvas lock: html2canvas crops to the named element and misses parent-layout bugs (e.g. flex-basis as height in a flex-column ancestor); a full CIC screenshot of the real page avoids this. User triggers: "no html2canvas", "no synthetic canvas", "use CIC for verification", "internal browser". Pattern: CIC scroll the element into view → CIC screenshot (save_to_disk) → read the image.
- **refSourcePolicy** — LOCKED hydration policy — preferred sources per topic, UniProt-cross-check requirement, HPA protein-over-mRNA priority
- **refVendorIngestPattern** — LOCKED architecture — third-party-vendor data ingest goes through leaf-agent / sanitized-endpoint indirection. Orchestrator context must NEVER see brand names. Read before wiring any new vendor source.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/biomarker/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/skill_book/biomarker`

Rationale layer (per CLAUDE.md §6). The WHY behind the biomarker-page build
disciplines — the load-bearing reasoning the howTos enforce as rules. Append;
supersede in place. (Sourcing/validator *policy* lives in the howTos
`refSourcePolicy` / `refVendorIngestPattern`; this file records why those choices
override the obvious alternative.)

---

## 2026-06-13 — vendor brand names live ONLY in server endpoints + leaf howTos, never in the orchestrator

**Decision.** Third-party vendor brand names (Abcam, ProteinTech, Cell Signaling,
Boster, Sigma, …) appear ONLY in server-side endpoint `.js` (Node-only; never read
by the LLM orchestrator) and in skill-book howTos (read only by leaf agents). They
MUST NOT appear in chain prompts, MIRROR output, token/field names, endpoint URL
paths, helper names, state fields, or topic-table sources.

**Rejected alternative.** Hardcode the brand throughout (`ProteinTech-band-explanation`
instead of `vendor-band-explanation`) — simpler and more readable.

**Why.** Orchestrator prompts render into final user-facing output, and the LLM
readily picks up brand cues from variable names and propagates them into prose.
Routing brands through brand-neutral indirection (a) keeps output brand-neutral so
any current/future vendor plugs into the same template, and (b) prevents fair-use
leakage of commercial phrasing.

**Consequence if violated.** Pages cite branded vendors → vendor lock-in + fair-use
liability; future vendor swaps break existing prose; LLM adopts vendor tone.

---

## 2026-06-13 — vendor prose gets a dedicated Step-1 SANITIZE+REPHRASE; downstream steps never see the raw payload

**Decision.** When a chain ingests vendor free-text (background_explanation, image
legends, …), Step 1 is a dedicated sanitize+rephrase into Boster voice; subsequent
steps reference `{{step1.json}}` facts, never the raw vendor payload. Applies to
commercial vendors only — public DBs (UniProt, HPA, GTEx, PMC) may be quoted
exactly (fact-citation, not IP-borrowing).

**Rejected alternative.** Strip brand names and reuse the vendor prose as-is.

**Why.** Vendor-curated text is commercial IP and carries a characteristic voice the
LLM propagates. A rewrite step is the only thing that converts it to citable facts
in the house voice.

**Consequence if violated.** Verbatim vendor prose leaks into output (fair-use);
customer content reads like a vendor datasheet.

---

## 2026-06-13 — `references[]` carries only non-profit DBs + peer review; vendor URLs are BANNED

**Decision.** The "Source links (this section derived from)" footer contains ONLY
non-profit knowledge bases + peer-reviewed publications (UniProt, HPA, GTEx, PMC,
PubMed, Bgee, Expression Atlas, OmniPath, Allen Brain Atlas, Reactome, KEGG,
STRING, journal DOIs, biorxiv). Commercial vendor URLs (ptglab.com, abcam.com,
bosterbio.com, …) are banned regardless of label; a client-side `refsBlock` helper
strips banned hostnames (defense-in-depth).

**Rejected alternative.** Include vendor datasheets as "citable evidence."

**Why.** `references[]` is presented as scientific evidence; a vendor site is
reagent attribution, not evidence. Mixing them damages credibility and reintroduces
the fair-use problem.

**Consequence if violated.** Published pages cite commercial URLs as scientific
sources.

---

## 2026-06-13 — HPA protein evidence outranks mRNA; the HPA protein-score chart is the SOLE source for tissue control picks

**Decision.** Expression hydration prioritizes HPA protein-level evidence
(IHC-validated, categorical High/Medium/Low/Not-detected) over mRNA (nTPM); mRNA is
used only when protein is unavailable, with the inline tag "(mRNA-level only — no
protein evidence at HPA)". Tissue positive/negative control picks come ONLY from the
HPA tissue-page protein-expression-score barChart
(`/api/pg/hpa-protein-score?gene=X`), not GTEx mRNA, HPA search_download, or
OmniPath.

**Rejected alternative.** Treat mRNA and protein as interchangeable confidence; use
the simpler HPA search_download API or a proxy derivation for controls.

**Why.** Protein detection is immunologically validated; mRNA is a proxy that can
contradict protein biology (high transcript, no detectable protein). A single
authoritative, confidence-scored source prevents control-selection drift and keeps
the page consistent with the HPA data a reader would cite.

**Consequence if violated.** Controls chosen on signals that contradict protein
biology (false-positive control choice); controls diverge from HPA's own ranking.

---

## 2026-06-13 — deterministic MW→protocol rules are the source of truth, overriding handbook citations

**Decision.** Locked gel% / transfer / membrane / blocking rules (e.g. gel% by MW
band; semi-dry vs wet transfer by MW; PVDF pore by MW; BSA for phospho) fill
`MIRROR.protocols` deterministically, overriding any AI "recommended" cell sourced
from a vendor handbook.

**Rejected alternative.** Let the AI generate recommendations from literature,
treating Bio-Rad/Thermo/Abcam handbooks as equally authoritative.

**Why.** These are common-knowledge WB practice — all major vendors agree within
±5%, so no handbook is more reliable than the rule itself; the rule IS the practice.
Determinism guarantees reproducibility across pages.

**Consequence if violated.** AI recommendations drift from standard practice →
non-reproducible WB variance. (If a genuinely new method emerges, record the change
here; do not silently revert.)

---

## 2026-06-09 — PMC literature search uses LOCKED proximity query tiers (v6)

**Decision.** WB-paper search runs ordered tiers: (1) SKU citation; (2) caption
proximity `"{gene} western|immunoblot|immunoblotting"[capt:~30]` with **all three
spellings**; (3) text-word; (4) broad; (5) LAST-TIER body proximity `[body:~8]`,
drawn only after higher tiers exhaust the pool budget. RRID tier is REJECTED
(~0.7% net gain, a subset of SKU).

**Rejected alternative.** A loose `"{gene}" AND "Western blot"` query.

**Why.** Empirically tuned precision/recall (caption-western baseline ~15.7%
genuine-hit rate; body `~8` adds ~30% recall without displacing higher-precision
hits). Tight proximity bounds filter co-mention noise; three immunoblot spellings
are required because PMC does not stem them.

**Consequence if violated.** High false-positive rate (papers mentioning the gene
and WB separately) buries the real papers in the extraction step.

---

## 2026-06-09 — verification is a real CIC screenshot, never html2canvas; ported demo-HTML chunks keep their exact tag + classes

**Decision.** (a) Biomarker mockup UI is verified only by a real CIC screenshot of
the rendered page (`mcp__Claude_in_Chrome__*`), never an html2canvas/DOM canvas, and
NOT via OS-level `mcp__computer-use__screenshot`. (b) When a MIRROR builder "renders
the demo HTML for section X," it is taken literally: the outer tag + class names
must match `demo.html` exactly and ported chunks are concatenated raw — no wrapper
`div`/grid/aside containers.

**Rejected alternative.** (a) Synthetic canvas screenshots (no browser). (b) Add
wrapper containers for "better layout."

**Why.** (a) html2canvas crops to the named element and misses parent-layout bugs
(e.g. an ancestor's flex-basis acting as height); only real browser paint catches
them. (b) The reference CSS assumes no wrapper; extra containers fight it and the
streamed page stops resembling `demo.html`.

**Consequence if violated.** (a) Layout bugs ship undetected. (b) Pages look nothing
like the reference template.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/biomarker/index.js -->
```js
/**
 * Biomarker Pipeline Skill Book
 * Covers gene info card hydration: Phase 1 (script-only) and Phase 2 (LLM narrative).
 */
export const skillBook = {
  id: 'biomarker',
  name: 'Biomarker Pipeline',
  description:
    'Actions for hydrating gene info card templates with structured external data. ' +
    'Phase 1: zero-LLM; fetches UniProt, Protein Atlas, and Boster catalog. ' +
    'Phase 2 (planned): LLM narrative generation from hydrated data facts.',

  toc: {
    refSourcePolicy: {
      description: 'LOCKED hydration policy — preferred sources per topic, UniProt-cross-check requirement, HPA protein-over-mRNA priority',
    },
    refVendorIngestPattern: {
      description: 'LOCKED architecture — third-party-vendor data ingest goes through leaf-agent / sanitized-endpoint indirection. Orchestrator context must NEVER see brand names. Read before wiring any new vendor source.',
    },
    refMwDeterministicRules: {
      description: 'LOCKED deterministic MW → gel% / transfer / membrane / blocking rules; retires R2 (handbook citation).',
    },
    refOmniPathHpa: {
      description: 'LOCKED HPA protein-level evidence via OmniPath /annotations?resources=HPA_tissue (replaces R7 strategy).',
    },
    refPmcCaptionProximity: {
      description: 'LOCKED v6 (live impl = boster_nexus pmc.js buildQueries): PMC pipeline finds candidates SKU-citation first, then caption-proximity ("<gene> western|immunoblot|immunoblotting"[capt:~30] OR-group), text-word, broad, then body~8 OR-group as a LAST-tier fallback (caption-sparse genes only); RRID rejected (⊆ SKU). Extracts candidates with full-article AI (methods + WB-fig captions + abstract + body intro + supp URLs as context), ranks by fill_count, returns best as presets. No quality gate — partial extractions surfaced honestly with their fill_count. Empirics: memory/reference_wb_pmc_query_tiers.md.',
    },
    refHumanReactivityGate: {
      description: 'LOCKED 2026-05-26 — antibody picker MUST verify human is in reactivity[]. If no human-reactive antibody exists for a gene, return out_of_scope=true with the error "Boster does not have a human-reactive antibody for X, so this is out of scope for the Western blot content pipeline." Implemented in /api/pg/picoband-wb.',
    },
    refPimImagePathOnly: {
      description: 'LOCKED 2026-05-26 — image URLs come ONLY from pim_product_images.url. No SKU-pattern guessing, no URL fabrication. If PIM URL is broken (null, "[object Object]", JSON), the card shows broken-image state — do NOT synthesize a fallback path.',
    },
    refDirectFillVsAgentFill: {
      description: 'LOCKED — hydration fill methods, in order: direct-fill HTML+[slug] tokens, then AI agent fill (JSON-key mapping preferred, example-template fill as fallback).',
    },
    refDemoTemplateLiteral: {
      description: 'LOCKED 2026-05-26 — when a MIRROR/renderer mirrors a reference HTML file, emit the exact same outer tag (section vs div) AND inner class names as the reference. The accumulator (Results stack / preview stream) MUST NOT wrap ported chunks in extra container/grid/aside divs — concatenate raw and let the reference CSS do layout. Audit cue: "outer tag of MIRROR.X[0] === outer tag of demo.html sectionX"?',
    },
    refRealScreenshotOnly: {
      description: 'LOCKED 2026-05-27, tool corrected 2026-06-08 — html2canvas / DOM-rendered canvas screenshots are NOT visual verification for biomarker mockup UI work. DEFAULT verification channel = a real CIC screenshot of the rendered result page via mcp__Claude_in_Chrome__* (computer action=screenshot, save_to_disk) — this is the authoritative channel per the browsercontrol skill book; it captures real browser paint, not a synthetic DOM canvas, so it already solves the html2canvas problem. Do NOT default to mcp__computer-use__screenshot (OS-level) — it is NOT the channel, and the user may decline to grant it (declined 2026-06-08). Reason for the no-html2canvas lock: html2canvas crops to the named element and misses parent-layout bugs (e.g. flex-basis as height in a flex-column ancestor); a full CIC screenshot of the real page avoids this. User triggers: "no html2canvas", "no synthetic canvas", "use CIC for verification", "internal browser". Pattern: CIC scroll the element into view → CIC screenshot (save_to_disk) → read the image.',
    },
    refHpaProteinScoreSource: {
      description: 'LOCKED 2026-05-27 — single source of truth for tissue positive/negative control picks: HPA tissue page Protein-expression-score chart ($("#proteinChart").barChart) on https://www.proteinatlas.org/{ENSG}-{GENE}/tissue. Endpoint: GET /api/pg/hpa-protein-score?gene=X. Returns full tissues[] + ranked sorted.positive_candidates (top 5 by score desc, High/Medium first) + sorted.negative_candidates (top 5 by score asc, Not detected first). Score mapping: value 3.000→High, 2.000→Medium, 1.000→Low, 0→Not detected. Do NOT derive controls from GTEx mRNA, HPA search_download (drops protein cols), or any proxy signal; this endpoint is the ONLY supplier for MIRROR.controls + must-know negative-control row.',
    },
    discoverIds: {
      description: 'Discover UniProt accession and Ensembl ID for a gene symbol via UniProt search API',
    },
    buildUrlMap: {
      description: 'Build the urlMap for a given biomarker from known IDs (UniProt accession, Ensembl ID, gene symbol)',
    },
    hydrateTemplate: {
      description: 'Resolve all script-hydratable slugs via the biomarker weapon and return a hydrated slug map',
    },
    identifyGaps: {
      description: 'Compare a resolved slug map to SLUG_REGISTRY to list literature and LLM slugs still pending',
    },
  },

  actions: {
    refVendorIngestPattern: {
      description: 'LOCKED — vendor data ingest goes through leaf-agent / sanitized-endpoint indirection so orchestrator context never sees brand names. Read BEFORE wiring any new third-party vendor source.',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
LOCKED — VENDOR INGEST INDIRECTION PATTERN (do not deviate)
═══════════════════════════════════════════════════════════════════════════════

WHY THIS PATTERN
  The orchestrator's AI chain prompts get rendered into the final user-facing
  biomarker page. Any brand name (Abcam, ProteinTech, Cell Signaling, CST,
  Boster, BosterBio, Picoband, Santa Cruz, Sigma, Thermo, Invitrogen,
  ProteinSimple, R&D Systems, Bio-Techne, etc.) that touches the orchestrator
  context risks leaking into output prose — the LLM is good at picking up brand
  cues and citing them. Strip brands at the ROOT: orchestrator never sees them.

THE PATTERN — three rules

RULE V1 — VENDOR-NAME ISOLATION
  Brand names (URLs, Cat# prefixes that imply brand, vendor handle prose) live
  ONLY in:
    (a) Server-side endpoint .js files (Node-only; never read by the LLM
        orchestrator). Code may freely fetch ptglab.com / abcam.com / etc.
    (b) Skill book howTos (this file and per-vendor skill books). Read ONLY by
        leaf agents dispatched to do vendor research, never auto-loaded by the
        main orchestrator.
  Brand names MUST NOT appear in:
    · CHAINS prompts (the textarea content the chain runner sends to the LLM)
    · MIRROR builder output strings
    · Interpolation token names (use {{vendorBandExplanation}} NOT {{proteintechBand}})
    · Endpoint URL paths (use /api/pg/vendor-band-explanation NOT /api/pg/proteintech-band)
    · Helper function names in client code (ensureVendorBandExplanation NOT ensureProteintechBand)
    · State field names (st.vendorBandExplanation NOT st.proteintechBand)
    · Topic-table 'sources' pills (use 'vendor catalog' / 'vendor antibody datasheet')

RULE V2 — SANITIZED RESPONSE PAYLOAD
  The server endpoint / leaf agent MUST return ONLY the extracted facts, never
  the brand name. The Cat# field is fine (numbers like "28205-1-AP" don't imply
  a brand to the LLM the way "ProteinTech" does). The URL field is fine
  (verifiable when clicked, but the LLM should not mention the domain).
  Payload shape (the contract):
    {
      "product": {
        "cat_no": "28205-1-AP",
        "citations": 196,
        "applications": "WB, IHC, IF, ELISA",
        "url": "https://www.ptglab.com/products/.../..."
      },
      "molecular_weight": { "calculated": "...", "observed": "..." },
      "background_explanation": "...prose from vendor curated text, brand name stripped if present...",
      "wb_image_legends": ["...", "..."],
      "cross_check": { "vendor_uniprot": "P23560", "requested_uniprot": "P23560", "verified": true }
    }
  Endpoint code should run a final regex strip on any free-text field to remove
  brand names before responding.

RULE V4 — DEDICATED SANITIZE+REPHRASE AI STEP FOR VENDOR PROSE
  Server-side brandStrip() removes brand NAMES from the response payload, but
  the prose itself can still be a verbatim copy of vendor-curated text. We do
  not want to retain commercial-entity phrasing in our output — for two
  reasons: (a) IP / fair-use; (b) brand-tone leakage (vendor pages have a
  characteristic voice that an LLM picks up and propagates).

  THEREFORE: when a multi-step AI chain ingests a vendor payload that includes
  free-text prose (background_explanation, wb_image_legends, product descriptions,
  etc.), the chain MUST have a DEDICATED Step 1 whose only job is
  sanitize+REPHRASE. Subsequent steps reference {{step1.json}} for the
  vendor-derived facts — NEVER the raw vendor payload token directly.

  Step 1 shape:
    Input:  {{vendorXyzPayload}} (the raw server-side payload, brand-stripped)
    Output: structured JSON of FACTS extracted + prose REWRITTEN in our voice
    Hard rules in the prompt:
      · No verbatim sentences from input
      · No brand names (defense-in-depth — server-side strip is the first pass)
      · Keep numeric facts verbatim (kDa values, Cat#, citation counts, URLs)
      · Rephrase any prose to declarative, ≤2-sentence chunks
      · If you cannot rephrase without changing the meaning, omit (do NOT copy)

  Step 2+ then reads ONLY {{step1.json}}.factual_fields, never the raw vendor
  token. This eliminates "the vendor reports..." / "Background information says..."
  / sentence-fragment leakage in the final mirror output.

  This rule applies to COMMERCIAL VENDOR sources (antibody/reagent catalogs).
  It does NOT apply to public databases (UniProt, HPA, GTEx, PMC, NCBI) — exact
  phrasing from those is fact-citation, not IP-borrowing, and may be quoted
  freely.

  Currently applied in: band CHAIN (Step 1 = sanitize+rephrase vendor data;
  Step 2 = derive band-size mirror using {{step1.json}}).

RULE V3 — LEAF-AGENT DISPATCH FOR NEW VENDOR RESEARCH
  When the user asks "test another vendor" or "compare three vendors" or
  "feasibility study on vendor X": the orchestrator dispatches a LEAF AGENT
  with a brand-free task description ("test vendor candidate <descriptor>",
  not "test ProteinTech"). The leaf agent's brief is the only place brand
  names appear in the dispatch chain. The leaf agent runs the research,
  records the brand names in its own work files, and returns to the
  orchestrator ONLY:
    · A boolean PASS/FAIL/PARTIAL verdict
    · Numeric hit-rates per criterion
    · A brand-free recommendation ("wire candidate-A as primary; tier B as
      fallback")
    · The URL pattern + endpoint contract for wiring (in a brand-free name)
  The leaf agent MAY write a detailed report file (.tmp/<name>-feasibility.md)
  that names brands — that file is for human review, not orchestrator context.

CANONICAL EXAMPLES (current implementations)

A) Band-explanation vendor (currently wired)
   · Endpoint: /api/pg/vendor-band-explanation?gene=X&uniprot=Y
   · Token:    {{vendorBandExplanation}}
   · Helper:   ensureVendorBandExplanation(st)
   · State:    st.vendorBandExplanation
   · Vendor: top-WB-validated antibody from one vendor (see per-vendor skill
     book or .tmp/cst-feasibility.md / .tmp/proteintech-feasibility.md for
     which one — orchestrator does not need to know).
   · Cross-check: vendor's AlphaFold structureName accession matches input
     UniProt acc (per refSourcePolicy RULE 2).
   · References label: "WB-validated antibody Cat# {cat_no} — observed {mw} ({n}
     citations)" — NO vendor brand.

B) Internal-catalog hero antibody (currently wired)
   · Endpoint: /api/pg/biomarker-hydrate (existing)
   · Token:    {{vendorHeroProduct}} (rename pending; was {{bosterHeroProduct}})
   · Vendor: internal catalog. Same isolation rule — output labels say
     "WB-validated antibody Cat# {SKU}" not "{brand} hero".

C) New vendor research dispatch (template)
   · User: "test <vendor C> next"
   · Orchestrator: TaskCreate + Agent leaf-agent with prompt that names the
     vendor, runs the study, returns PASS/FAIL/PARTIAL + per-criterion %
     + brand-free wiring recommendation. Orchestrator inlines ONLY the
     numeric results into the dashboard R-panel, never the vendor name in
     mirror prompts.

REVERSAL RULE
  These rules can be relaxed for an internally-owned brand (BosterBio in this
  workspace) IF AND ONLY IF the user has explicitly asked for that brand to
  appear in output. Default is brand-anonymous even for internal brand —
  the biomarker landing pages we publish should look catalog-neutral so any
  vendor product (current or future) can plug into the same template.
═══════════════════════════════════════════════════════════════════════════════
      `,
    },

    refSourcePolicy: {
      description: 'LOCKED hydration policy — read this BEFORE writing any biomarker hydration prompt',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
LOCKED RULES (do not deviate without explicit user approval)
═══════════════════════════════════════════════════════════════════════════════

RULE 1 — HPA EVIDENCE-TYPE PRIORITY
  When hydrating from Human Protein Atlas (HPA), PRIORITIZE protein-level evidence
  over mRNA-level evidence. Use mRNA ONLY when protein-level evidence is unavailable
  for the gene-tissue or gene-cell-line combo. State the evidence type inline in any
  hydrated claim.
  HPA columns:
    Protein-level: 'proteintissue' (tissue), 'proteincellline' (cell line) — categorical
                   "High / Medium / Low / Not detected" calls from validated antibodies.
    mRNA-level:    'rnats', 'rnatsm', 'rnatd', 'rnacls', 'rnaclntpm' — nTPM expression.
  When BOTH exist and disagree, protein wins. When only mRNA exists, label the claim
  "(mRNA-level only — no protein evidence at HPA)".

RULE 2 — UNIPROT CROSS-CHECK STEP (REQUIRED FIRST STEP OF EVERY HYDRATION CHAIN)
  Every AI hydration chain MUST start with a step that resolves the input UniProt
  accession to its corresponding canonical link in EACH external source it will
  ingest, AND verifies that link maps back to the same protein/gene.

  Resolution pattern per source:
    UniProt → HPA: UniProt accession xref → Ensembl gene ID → 'proteinatlas.org/{ENSG}-{GENE}'
                   Verify: HPA page Gene == UniProt record gene symbol.
    UniProt → GTEx: gene symbol → GTEx /reference/gene → gencodeId (with version)
                    Verify: returned geneSymbol matches.
    UniProt → PubMed/PMC: search "({UniProt_acc}[All Fields] OR {gene}[Gene Symbol])
                          AND Western blot"
                          Verify: returned hits' MeSH terms include the gene.
    UniProt → Boster (catalog): gene symbol → 'bosterbio.com/bosterbio-gene-info-cards/{gene}'
                                Verify: page exists (HTTP 200) AND mentions UniProt accession.

  If verification fails for a source, the chain must MARK that source unavailable
  for this protein and proceed without it — never fabricate.

RULE 3 — PREFERRED SOURCES PER TOPIC (audit-approved)
  band            → UniProt (mass + processing + isoforms) [primary]
                  → HPA (protein-level tissue expression: where IS this protein?) [secondary]
  protocols       → PMC (eutils search; validated WB papers) [primary]
                  → Boster product DB (datasheet protocols, server-side) [secondary]
  controls        → HPA (protein-level tissue + cell-line "Not detected" calls) [primary]
                  → GTEx (median TPM across 54 tissues, quantitative) [confirmation]
                  (already wired via /api/pg/expression-data)
                  CROSS-REF: for tissue control PICKS the locked single source of truth is now
                  refHpaProteinScoreSource (the barChart endpoint); OmniPath/GTEx are breadth/confirmation only.
  advanced        → UniProt (topology + annotated PubMed IDs in features/comments) [primary]
                  → PMC (specific troubleshooting papers) [secondary]
  faqs            → derived view of already-resolved sections; no new source
  antibodies      → Boster product DB (gene + WB filter, server-side) [primary]
                  → no fabrication if catalog inaccessible
  validation      → Boster validation-image DB (server-side) [primary]
                  → PMC figure mining (Pass-2) [secondary]
  publication     → PMC eutils (search by gene + Western blot) [primary]
                  → no fabrication; degrade honestly if not run
  quant           → no fixed first-party source; interpretive
                  → UniProt facts as anchor; literature mining
  transfer        → UniProt mass (deterministic MW→method rule) [primary, sufficient]

RULE 4 — REFERENCES SECTION REQUIRED
  Every hydrated section's mirror builder MUST render a "References" / "Source links"
  list at the bottom containing the SPECIFIC URL(S) the section's content was derived
  from. Not domain names — full URLs that resolve to the exact resource queried.
  Examples:
    https://www.uniprot.org/uniprotkb/P23560
    https://www.proteinatlas.org/ENSG00000176697-BDNF/tissue
    https://gtexportal.org/home/gene/BDNF
    https://www.ncbi.nlm.nih.gov/pmc/articles/PMC1234567/
  Empty list is acceptable when no source was used (e.g. faqs derived view, quant
  interpretive). Fabricated URLs are a hard violation.

RULE 5 — INLINE EVIDENCE PROVENANCE
  Every quantitative or categorical claim in a hydrated section MUST carry an inline
  evidence tag:
    "(HPA: protein-level High)"
    "(GTEx 5.85 TPM)"
    "(UniProt feature SIGNAL 1-18)"
    "(PMC1234567 Fig. 2A)"
  Claims without provenance must be removed.

RULE 6 — REFERENCES LIST: NON-PROFIT DATABASES + PEER-REVIEWED PUBLICATIONS ONLY
  The "Source links (this section derived from)" footer (refsBlock) is citable
  evidence. Only URLs to non-profit knowledge bases and peer-reviewed publications
  qualify. Commercial vendor catalog URLs DO NOT qualify regardless of how they
  are labeled — they are not citable evidence, they are reagent attribution.

  ALLOWED in references[]:
    · UniProt              (uniprot.org, ebi.ac.uk/uniprotkb)
    · Human Protein Atlas  (proteinatlas.org)
    · GTEx                 (gtexportal.org)
    · NCBI PMC             (ncbi.nlm.nih.gov/pmc/articles/PMC…)
    · NCBI PubMed          (pubmed.ncbi.nlm.nih.gov/…)
    · NCBI Entrez          (ncbi.nlm.nih.gov/gene, /protein, etc.)
    · Bgee                 (bgee.org)
    · Expression Atlas     (ebi.ac.uk/gxa)
    · OmniPath             (omnipathdb.org)
    · Allen Brain Atlas    (brain-map.org)
    · Ensembl              (ensembl.org)
    · Reactome             (reactome.org)
    · KEGG                 (genome.jp/kegg)
    · STRING               (string-db.org)
    · Peer-reviewed journal URLs (publisher DOI links, biorxiv.org, medrxiv.org)

  BANNED in references[]:
    · ptglab.com, proteintech.com  (vendor)
    · abcam.com                    (vendor)
    · cellsignal.com, cst-science.com (vendor)
    · bosterbio.com                (vendor — including our own)
    · sigma-aldrich.com, sigmaaldrich.com (vendor)
    · thermofisher.com, lifetechnologies.com, invitrogen.com (vendor)
    · scbt.com, santacruzbiotech.com (vendor)
    · biotechne.com, rndsystems.com (vendor)
    · novusbio.com, abnova.com, abclonal.com (vendor)
    · agilent.com, perkinelmer.com (vendor)
    · ANY other commercial reagent/antibody vendor

  Where vendor data DOES belong (NOT in references[]):
    · product_cards[] — the inline "WB-validated antibody Cat# X" card
    · validated_reagent — a footer line attributing the reagent used
    · inline citation tags like "(observed 28-30 kDa)" — number is the fact,
      not the brand URL

  IMPLEMENTATION: client-side refsBlock helper strips any URL whose hostname
  matches the banned list (defense-in-depth — chain prompts also instructed
  not to include them).
═══════════════════════════════════════════════════════════════════════════════
═══════════════════════════════════════════════════════════════════════════════
      `,
    },

    refMwDeterministicRules: {
      description: 'LOCKED — deterministic MW → gel% / transfer / membrane / blocking rules driven by MIRROR.protocols RULE_OVERRIDES. Retires R2 (handbook citation): the rule IS the source of truth.',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
LOCKED — DETERMINISTIC PROTOCOL RULES
═══════════════════════════════════════════════════════════════════════════════
Inputs: matureMass_kDa (rec.matureChain + molWeight, else rec.molWeight); isPhospho (rec.ptm /phospho/i OR rec.modres).

Gel %: ≤20→15% · 20-80→12-15% · 80-200→8-10% · >200→4-12% gradient · unknown→12-15%
Transfer: ≤30→Semi-dry short + 0.2µm · 30-100→Semi-dry standard · >100→Wet/tank longer · Quantitative→always Wet/tank
Membrane: <20→PVDF 0.2µm · ≥20→PVDF 0.45µm · high-bg→Nitrocellulose
Blocking: phospho→5% BSA · default→5% milk OR 5% BSA in TBST
Detection: always ECL std/high-sensitivity, expose at multiple times, +/- controls; NEVER include MW in Detection text.

WHY NO HANDBOOK CITATION: common-knowledge WB practice, Bio-Rad/Thermo/Abcam/CST agree ±5%. The rule IS the source of truth; edge cases live in Advanced Tips accordion. R2 retired.

APPLIED IN: index.html MIRROR.protocols.RULE_OVERRIDES — overrides AI's "recommended" cells; deterministic fill.
═══════════════════════════════════════════════════════════════════════════════
      `,
    },

    refOmniPathHpa: {
      description: 'LOCKED — HPA protein-level evidence via OmniPath /annotations?resources=HPA_tissue. Replaces R7 — strategy PASS 90/100, ready to wire into /api/pg/expression-data.',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
LOCKED — HPA PROTEIN-LEVEL VIA OMNIPATH
═══════════════════════════════════════════════════════════════════════════════
PROBLEM: refSourcePolicy RULE 1 requires protein > mRNA. HPA's own search_download drops protein cols; SPA HTML scrape brittle; 730 MB bulk impractical.

SOLUTION endpoint:
  https://omnipathdb.org/annotations?resources=HPA_tissue&genesymbols={GENE}&format=json
Returns per-record {organ, tissue, cell_type, level, status}; level ∈ {"High","Medium","Low","Not detected"} IHC-derived. No auth, no key, 30-day cache.

VERIFIED (2026-05-20): EGFR 107 rows/22s · BDNF 912 rows/16s · AKT1 84 rows · TP53 134 rows.

WIRING: /api/pg/expression-data extension:
  expressionData.hpa_protein = {
    source_url, records[], protein_high_tissues[], protein_med_tissues[], protein_negative_tissues[]
  }
MIRROR.controls prefers protein_high_tissues > rna-high; inline tag "(HPA: protein-level High)" per RULE 5.

CONTROL-PICK SOURCE OF TRUTH (cross-ref): for tissue control PICKS the locked single source of truth is now refHpaProteinScoreSource (the barChart endpoint); OmniPath/GTEx are breadth/confirmation only.

FALLBACK: 0-row OmniPath → mRNA-only with LOCKED "(mRNA-level only — no protein evidence at HPA)" tag.
═══════════════════════════════════════════════════════════════════════════════
      `,
    },

    refPmcCaptionProximity: {
      description: 'LOCKED (v6) — PMC eutils proximity search for finding articles with a REAL WB of the biomarker. Caption tier spans (western OR immunoblot OR immunoblotting) within N words; body~8 added as a last-tier fallback for caption-sparse genes; RRID rejected (⊆ SKU). Live impl = boster_nexus libs/agc_pipeline/data_sources/pmc.js buildQueries.',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
LOCKED (v6) — PMC PROXIMITY SEARCH (caption + body tiers)
═══════════════════════════════════════════════════════════════════════════════
PROBLEM: Loose "BDNF[Title/Abstract] AND Western blot" returns 80-200 papers, most measure BDNF differently (ELISA) and have a WB of something else.

LIVE IMPLEMENTATION (do not re-fork): boster_nexus
  libs/agc_pipeline/data_sources/pmc.js → buildQueries(gene, bosterSkus, method).
  The /api/pg/literature-data path below is the RETIRED GuildOS prototype.

CAPTION TIER: ("{gene} western"[capt:~N] OR "{gene} immunoblot"[capt:~N] OR "{gene} immunoblotting"[capt:~N])
  PMC indexes figure captions separately. [capt:~N] = proximity within N words
  inside caption text only. N=30 balances precision/recall (tune 20 tight / 50 loose).
  WHY all three spellings: PMC does NOT stem immunoblot↔immunoblotting, and a
  proximity phrase can't contain OR — so OR three separate proximity phrases.
  immunoblot == western blot by definition (no precision loss); each of
  immunoblot / immunoblotting alone is ~8-9% (under the 10% genuine-hit bar) but
  together = 15.7%, clearing it. Measured non-overlap vs caption-western:
  CD68 633→718 (+85, ~13%).

BODY LAST-TIER FALLBACK: ("{gene} western"[body:~8] OR "{gene} immunoblot"[body:~8] OR "{gene} immunoblotting"[body:~8])
  WB mentions live in methods/body far more than captions. [body:~8] (tight ~8
  words) ≈ +30% recall (7/7 genes) and is a cleaner signal than loose [body:~30].
  Run ONLY as the LAST tier — drawn after sku_cite + caption + text-word + broad
  exhaust but pool budget remains (never displaces a higher-precision candidate;
  CD68 fills to cap from the trusted tiers so body contributes 0). The downstream
  AI-extraction fill_count step handles residual non-OA/abstract-only noise.
  GOTCHA: efetch returns abstract-only for non-OA articles even though PMC indexes
  their full body for SEARCH — so you can't always verify a [body] hit via efetch
  (availability artifact, not spuriousness).

REJECTED — RRID: ~0.7% net gain. RRID-mandate journals require "Cat# + RRID"
  together, so RRID ⊆ the SKU citation already covered by the sku_cite tier; adds
  ~1 of 143 SKU-citations across 6 correctly-paired Boster antibodies. (Raw data
  looked like 452 RRID-only but that was a regex mis-pairing — a Proteintech RRID
  wrongly attached to a Boster catalog # in a multi-antibody list.)

EXAMPLE:
  https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi?db=pmc&term=%22bdnf+western%22%5Bcapt%3A%7E30%5D&retmode=json

USE CASES:
  (a) §8 Publication section — list real WB papers ranked by citation count
  (b) §2 Protocols "Publication-derived" column — extraction hit rate is higher when paper truly contains the biomarker's WB
  (c) §1 Band sidebar carousel — figure-caption snippet + PMC fig URL for the "real WB image of THIS protein" slide

FULL EMPIRICS: memory/reference_wb_pmc_query_tiers.md.
═══════════════════════════════════════════════════════════════════════════════
      `,
    },

    refDirectFillVsAgentFill: {
      description: 'LOCKED — hydration fill methods, ordered by preference. Direct-fill HTML templates with [slug] tokens FIRST; AI agent fill only when format is fluid.',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
LOCKED — TWO HYDRATION FILL METHODS
═══════════════════════════════════════════════════════════════════════════════

METHOD 1 — DIRECT FILL (preferred: faster, cheaper, no LLM call)
  HTML template carries explicit [slug] tokens. Hydration = single JSON fetch + string-replace.
  USE WHEN output format is RIGID: tables, lists, SVG, product cards, JSON-LD, chips, page chrome.

  Example template (band TL;DR row):
    <div class="tldr-line"><b>Expected band</b>
      <span><strong>[band_min_kda]–[band_max_kda] kDa</strong></span>
    </div>
  Step:
    fetch /api/pg/band-data?uniprot=X → {band_min_kda: 27, band_max_kda: 38}
    tmpl.replaceAll('[band_min_kda]', d.band_min_kda).replaceAll('[band_max_kda]', d.band_max_kda)
  Convention: [snake_case_slug] in HTML; matching key in JSON.

METHOD 2 — AGENT FILL (use only when needed)
  USE WHEN output is FLUID: factor.text prose, decision_notes, Q&A answers, TL;DR phrasings that condense multi-source facts.

  2-A (preferred sub-pattern): JSON-key mapping
    Prompt: "Return JSON: {factor1_text: '...', factor2_text: '...'}"
    Template stays in our code. AI emits keys; we substitute.

  2-B (fallback): example-template fill
    Prompt: "Fill this template, return ONLY filled HTML: <div class='fb-tx'>Observed <strong>[X]</strong>...</div>"
    Hydration: take AI HTML, inject into mirror.
    Use 2-B only when structure varies per protein and 2-A's flat JSON can't express it.

DECISION TREE
  Format fixed? → Method 1
  Format varies per protein structurally? → Method 2-B
  Otherwise (prose text inside fixed structure) → Method 2-A
`,
    },

    refDemoTemplateLiteral: {
      description: 'LOCKED 2026-05-26 — when a MIRROR/renderer mirrors a reference HTML file, take the reference literally. Same outer tag, same class names, no extra wrappers at the porting stage.',
      howTo: `
═══════════════════════════════════════════════════════════════════════════════
LOCKED — USE THE TEMPLATE LITERALLY (demo.html parity rule)
═══════════════════════════════════════════════════════════════════════════════

When a MIRROR builder is supposed to "render the demo HTML for this section,"
the user means LITERALLY. Two failure modes seen in iter 27–32 of the biomarker
landing-page mockup that this rule exists to prevent:

FAILURE 1 — Wrong outer tag.
  demo.html wraps Section 3 in <section class="guide">. MIRROR.controls emitted
  <div style="grid-template-columns:...">. Looks "fine" rendered, but the user's
  first-glance audit ("wrapper is a div not a section") flagged it instantly.

  RULE: Outer tag of MIRROR.X output === outer tag of demo.html sectionX.
  AUDIT CUE: HTML-string-match the first <tag> per MIRROR vs demo.html. If they
  differ, MIRROR is wrong — not the audit.

FAILURE 2 — Accumulator re-wraps ported content.
  portToResults() wrapped each section in
    <div class="res-slot"><div class="res-slot-body">…MIRROR…</div><aside…></aside></div>
  with a 1180px grid and side commentary rail. demo.html has no such wrapper.
  Result: streamed page looked NOTHING like demo.html — the wrappers fought the
  reference CSS.

  RULE: The accumulator concatenates raw MIRROR output. NO extra container, NO
  grid, NO side rail in the rendered stream. (Meta-bar UI like topic count /
  clear-all is fine — but it sits OUTSIDE the stream container.)

CSS / WIDTH
  The reference template's own .container (or equivalent) handles centering and
  max-width. Don't fight it from the outer page — make the outer page a no-op
  for width so the reference CSS controls layout.

DIRECT QUOTE FROM USER 2026-05-26
  > "right off the bat, I see the wrapper is a div not a section. You are not
  > taking the instruction literally. Use the template."

This rule is the wrench against that drift. When in doubt: open demo.html,
read the first 30 chars of the section, make MIRROR start with the same chars.
`,
    },

    refHpaProteinScoreSource: {
      description: 'LOCKED 2026-05-27 — single source of truth for tissue positive/negative control picks: HPA tissue page Protein-expression-score chart ($("#proteinChart").barChart) on https://www.proteinatlas.org/{ENSG}-{GENE}/tissue. Endpoint: GET /api/pg/hpa-protein-score?gene=X. Returns full tissues[] + ranked sorted.positive_candidates (top 5 by score desc, High/Medium first) + sorted.negative_candidates (top 5 by score asc, Not detected first). Score mapping: value 3.000→High, 2.000→Medium, 1.000→Low, 0→Not detected. Do NOT derive controls from GTEx mRNA, HPA search_download (drops protein cols), or any proxy signal; this endpoint is the ONLY supplier for MIRROR.controls + must-know negative-control row. Implementation: app/api/pg/hpa-protein-score/route.js — scrapes the inline .barChart([...]) payload anchored to id="proteinChart", parses tooltip HTML into per-cell-type levels, caches per-ENSG for 30 days under .cache/hpa-protein-score/. Smoke-tested 2026-05-27 on VEGFA (broadly expressed; no zeros — biologically correct), BDNF (brain Medium, peripheral Not detected), EGFR (epithelial High, brain Not detected) — all match known biology.',
    },

    discoverIds: {
      description: 'Discover UniProt accession and Ensembl ID for a gene symbol via UniProt search API',
      howTo: `
Use the UniProt search REST endpoint to find the primary accession and Ensembl cross-reference for a gene:

  const searchUrl = \`https://rest.uniprot.org/uniprotkb/search?query=gene_exact:\${gene}+AND+organism_id:9606&fields=accession,xref_ensembl&format=json&size=1\`;
  const data = await fetch(searchUrl).then(r => r.json());
  const acc = data.results?.[0]?.primaryAccession;
  const ensRef = data.results?.[0]?.uniProtKBCrossReferences?.find(x => x.database === 'Ensembl');
  const ensemblId = ensRef?.id?.split('.')[0]; // strip isoform suffix e.g. ENST... → use gene ID

Common organism_id values: 9606 (human), 10090 (mouse), 10116 (rat).
Feed acc and ensemblId into biomarker.buildUrlMap to construct the full urlMap.
      `,
    },

    buildUrlMap: {
      description: 'Build the urlMap for a given biomarker from known IDs',
      howTo: `
Import and call buildUrlMap from the biomarker weapon:

  import { buildUrlMap } from '@/libs/weapon/biomarker';
  const urlMap = buildUrlMap({
    uniprotAccession: 'Q8TE56',           // UniProt primary accession
    ensemblId: 'ENSG00000138813',         // Ensembl gene ID
    gene: 'ADAMTS14',                      // Gene symbol for Boster gene card URL
  });
  // Returns:
  // {
  //   uniprot:         'https://rest.uniprot.org/uniprotkb/Q8TE56.json',
  //   proteinatlas:    'https://www.proteinatlas.org/ENSG00000138813-ADAMTS14.json',
  //   boster_gene_card:'https://www.bosterbio.com/bosterbio-gene-info-cards/ADAMTS14',
  // }

If Ensembl ID is unknown, omit it — proteinatlas will be skipped and tissue slugs will error gracefully.
      `,
    },

    hydrateTemplate: {
      description: 'Resolve all script-hydratable slugs and return hydrated map',
      howTo: `
Call via API (from client HTML) or server-side directly.

CLIENT (static HTML page): POST to /api/pg/biomarker-hydrate
  Body: { biomarker, urlMap }
  Returns: { [slug]: value } for all SCRIPT_HYDRATABLE_SLUGS
  CORS-enabled for localhost:3002 static pages.

SERVER (Next.js): import and call directly
  import { hydrateAll, SCRIPT_HYDRATABLE_SLUGS } from '@/libs/weapon/biomarker';
  const result = await hydrateAll(SCRIPT_HYDRATABLE_SLUGS, 'ADAMTS14', urlMap);

Phase 1 script-hydratable slugs (23 total):
  uniprot (11): gene_symbol, gene_full_name, gene_aliases, protein_superfamily, uniprot_id,
                wb_predicted_mw_kda, signal_peptide_range, ptm_glycosylation_sites,
                domain_list, subcellular_localization, isoform_count
  proteinatlas (3): tissue_high, tissue_med, tissue_low
  boster (7):   wb_image_url, ihc_image_url, main_wb_antibody_sku, wb_antibody_citations,
                wb_antibody_url, pub_count, assay_apps
  computed (3): uniprot_url, proteinatlas_url, boster_gene_url

After calling, check returned values for [error:...] strings and handle gracefully.
Boster scraping depends on gene card existing at bosterbio.com/bosterbio-gene-info-cards/{gene}.
      `,
    },

    identifyGaps: {
      description: 'List literature and LLM slugs still pending after Phase 1 hydration',
      howTo: `
After running hydrateTemplate, compare the resolved map to SLUG_REGISTRY:

  import { SLUG_REGISTRY } from '@/libs/weapon/biomarker';

  const pending = Object.entries(SLUG_REGISTRY)
    .filter(([slug, meta]) => meta.source === 'literature' || meta.source === 'llm')
    .map(([slug, meta]) => ({ slug, source: meta.source, desc: meta.desc }));

  // pending.literature → require manual research or Boster product page curation
  // pending.llm → Phase 2: call generate_narrative(slug, biomarker, hydratedFacts)
  //               where hydratedFacts is the resolved Phase 1 slug map

Phase 2 narrative generation will use the Phase 1 data as grounded facts to write
GEO-optimized full-sentence answer paragraphs, eliminating hallucination risk.
      `,
    },
  },
};
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/biomarker/index.js -->
```js
/**
 * Biomarker Hydration Weapon
 *
 * Zero-LLM, script-only Phase 1.
 * Fetches and parses structured gene data from external APIs and Boster catalog scraping.
 *
 * Primary export: hydrateAll(slugs, biomarker, urlMap) → { [slug]: value }
 *
 * urlMap keys:
 *   uniprot        — UniProt REST JSON endpoint, e.g. https://rest.uniprot.org/uniprotkb/Q8WXS8.json
 *   proteinatlas   — Human-readable Protein Atlas URL (browseable), e.g. https://www.proteinatlas.org/ENSG00000138316-ADAMTS14
 *                    NOTE: The .json endpoint pattern is an SPA and does NOT return JSON. Tissue data
 *                    is fetched internally via the search_download API using the biomarker gene symbol.
 *   boster_gene_card — Boster gene info card URL, e.g. https://www.bosterbio.com/bosterbio-gene-info-cards/ADAMTS14
 *
 * Slug source map:
 *   uniprot:      gene_symbol, gene_full_name, gene_aliases, protein_superfamily, uniprot_id,
 *                 wb_predicted_mw_kda, signal_peptide_range, ptm_glycosylation_sites,
 *                 domain_list, subcellular_localization, isoform_count
 *   proteinatlas: tissue_high, tissue_med, tissue_low (via search_download API + gzip)
 *   boster:       wb_image_url, ihc_image_url, main_wb_antibody_sku, wb_antibody_citations,
 *                 wb_antibody_url, pub_count, assay_apps
 *   computed:     uniprot_url, proteinatlas_url, boster_gene_url
 */

import { gunzipSync } from 'zlib';

// ── Source registry ────────────────────────────────────────────────────────────
export const SLUG_REGISTRY = {
  // Script-hydratable — UniProt
  gene_symbol:             { source: 'uniprot',      desc: 'HGNC gene symbol from UniProt genes entry' },
  gene_full_name:          { source: 'uniprot',      desc: 'Full recommended protein name from UniProt' },
  gene_aliases:            { source: 'uniprot',      desc: 'Gene synonyms and ORF names joined by ·' },
  protein_superfamily:     { source: 'uniprot',      desc: 'Protein family from UniProt similarity comment or keyword' },
  uniprot_id:              { source: 'uniprot',      desc: 'Primary UniProt accession number' },
  wb_predicted_mw_kda:     { source: 'uniprot',      desc: 'Predicted MW in kDa from UniProt sequence mass (rounded)' },
  signal_peptide_range:    { source: 'uniprot',      desc: 'Signal peptide residue start–end from UniProt features' },
  ptm_glycosylation_sites: { source: 'uniprot',      desc: 'N/O-glycosylation sites from UniProt PTM annotations' },
  domain_list:             { source: 'uniprot',      desc: 'Domain names from UniProt feature annotations' },
  subcellular_localization:{ source: 'uniprot',      desc: 'Subcellular location string from UniProt comment' },
  isoform_count:           { source: 'uniprot',      desc: 'Number of annotated isoforms from UniProt alternative products' },

  // Script-hydratable — Protein Atlas
  tissue_high: { source: 'proteinatlas', desc: 'Tissues with high protein expression (Protein Atlas)' },
  tissue_med:  { source: 'proteinatlas', desc: 'Tissues with medium protein expression (Protein Atlas)' },
  tissue_low:  { source: 'proteinatlas', desc: 'Tissues with low protein expression (Protein Atlas)' },

  // Script-hydratable — Boster catalog scraping
  wb_image_url:          { source: 'boster', desc: 'WB blot image URL scraped from Boster gene card' },
  ihc_image_url:         { source: 'boster', desc: 'IHC image URL scraped from Boster gene card' },
  main_wb_antibody_sku:  { source: 'boster', desc: 'Primary antibody SKU for WB (first SKU found on gene card)' },
  wb_antibody_citations: { source: 'boster', desc: 'Citation count for main WB antibody' },
  wb_antibody_url:       { source: 'boster', desc: 'Product page URL for main WB antibody' },
  pub_count:             { source: 'boster', desc: 'Total Boster publication count for this gene' },
  assay_apps:            { source: 'boster', desc: 'Validated assay applications detected from gene card' },

  // Computed
  uniprot_url:      { source: 'computed', desc: 'Human-readable UniProt URL derived from resolved uniprot_id' },
  proteinatlas_url: { source: 'computed', desc: 'Protein Atlas human-readable URL (strip .json from input)' },
  boster_gene_url:  { source: 'computed', desc: 'Boster gene info card URL from urlMap or derived from gene symbol' },

  // Literature / manual (Phase 1 stubs — caller supplies)
  wb_observed_mw_range:    { source: 'literature', desc: 'Observed WB band range from Boster WB image or publications' },
  wb_glycosylation_site:   { source: 'literature', desc: 'Specific glycosylation site (e.g. Asn-74) from UniProt PTM + lit' },
  wb_glycosylation_addon:  { source: 'literature', desc: 'kDa added by glycosylation (e.g. 5–11 kDa) from literature' },
  wb_recombinant_mw:       { source: 'literature', desc: 'Non-glycosylated recombinant MW from literature' },
  dimer_mw_range:          { source: 'literature', desc: 'Homodimer MW range under non-reducing SDS-PAGE' },
  pos_control_main:        { source: 'literature', desc: 'Main positive control cell line from Boster WB data' },
  pos_control_tissue:      { source: 'literature', desc: 'Tissue lysate positive control' },
  hypoxia_mimetic:         { source: 'literature', desc: 'Chemical hypoxia treatment protocol (e.g. cobalt chloride)' },
  neg_control_main:        { source: 'literature', desc: 'Negative control cell line' },
  species_identity_pct:    { source: 'literature', desc: 'Human vs mouse AA sequence identity %' },
  species_antibody_sku:    { source: 'literature', desc: 'Antibody SKU validated across species' },
  elisa_healthy_serum:     { source: 'literature', desc: 'Normal serum/plasma ELISA reference range' },
  elisa_cancer_serum:      { source: 'literature', desc: 'Cancer patient ELISA range' },
  elisa_kit_name:          { source: 'literature', desc: 'Boster ELISA kit name and catalog number' },
  elisa_kit_range:         { source: 'literature', desc: 'Boster ELISA kit detection range (pg/mL)' },

  // LLM (Phase 2)
  geo_wb_mw:           { source: 'llm', desc: 'Narrative: WB MW, glycosylation effect, isoform smear' },
  geo_dimer:           { source: 'llm', desc: 'Narrative: dimer/non-reducing gel behavior' },
  geo_isoforms:        { source: 'llm', desc: 'Narrative: isoform landscape and detection strategy' },
  geo_tissue:          { source: 'llm', desc: 'Narrative: tissue expression distribution' },
  geo_controls:        { source: 'llm', desc: 'Narrative: positive controls, negative controls, loading control' },
  geo_ihc_localization:{ source: 'llm', desc: 'Narrative: IHC staining pattern interpretation' },
  geo_species:         { source: 'llm', desc: 'Narrative: cross-species reactivity and zebrafish note' },
  geo_ab_selection:    { source: 'llm', desc: 'Narrative: antibody selection guide' },
  geo_elisa:           { source: 'llm', desc: 'Narrative: ELISA abundance by sample context' },
  gotcha_wb:           { source: 'llm', desc: 'Warning: WB detection common mistakes' },
  gotcha_elisa:        { source: 'llm', desc: 'Warning: ELISA vs WB interpretation' },
  gotcha_ihc:          { source: 'llm', desc: 'Warning: IHC artifact detection (necrosis)' },
};

// ── Internal fetch helpers ─────────────────────────────────────────────────────
async function _fetchJson(url, label) {
  const res = await fetch(url, {
    headers: {
      Accept: 'application/json',
      'User-Agent': 'BosterBio-ContentPipeline/1.0 (bosterbio.com)',
    },
  });
  if (!res.ok) throw new Error(`${label} HTTP ${res.status} — ${url}`);
  return res.json();
}

/**
 * Fetch tissue expression data from Protein Atlas search download API.
 * PA's ENSG-GENE.json URL pattern serves SPA HTML, not JSON — use this instead.
 * Returns the parsed JSON entry object for the first gene match, or throws.
 */
async function _fetchProteinAtlasTissue(biomarker) {
  const url = `https://www.proteinatlas.org/api/search_download.php?search=${encodeURIComponent(biomarker)}&format=json&columns=g,eg,rnats,rnatsm`;
  const res = await fetch(url, {
    headers: { 'User-Agent': 'BosterBio-ContentPipeline/1.0 (bosterbio.com)' },
  });
  if (!res.ok) throw new Error(`ProteinAtlas search API HTTP ${res.status}`);
  // Response is raw gzip — decompress with Node zlib
  const buf = Buffer.from(await res.arrayBuffer());
  const decompressed = gunzipSync(buf);
  const entries = JSON.parse(decompressed.toString('utf8'));
  if (!Array.isArray(entries) || entries.length === 0) throw new Error('No Protein Atlas entry for gene');
  return entries[0];
}

async function _fetchHtml(url) {
  const res = await fetch(url, {
    headers: {
      'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/120 Safari/537.36',
      Accept: 'text/html,application/xhtml+xml',
    },
  });
  if (!res.ok) throw new Error(`Boster HTML fetch HTTP ${res.status} — ${url}`);
  return res.text();
}

// ── UniProt field resolvers ────────────────────────────────────────────────────
function _fromUniProt(slug, data) {
  switch (slug) {
    case 'gene_symbol':
      return data.genes?.[0]?.geneName?.value ?? data.primaryAccession ?? '';

    case 'gene_full_name': {
      const rec = data.proteinDescription?.recommendedName?.fullName?.value;
      if (rec) return rec;
      return data.proteinDescription?.submittedNames?.[0]?.fullName?.value ?? '';
    }

    case 'gene_aliases': {
      const synonyms = (data.genes?.[0]?.synonyms ?? []).map(s => s.value);
      const orf = (data.genes?.[0]?.orfNames ?? []).map(o => o.value);
      return [...synonyms, ...orf].join(' · ') || '(no aliases listed)';
    }

    case 'protein_superfamily': {
      const sim = data.comments?.find(c => c.commentType === 'SIMILARITY');
      if (sim?.texts?.[0]?.value) {
        return sim.texts[0].value.replace(/^Belongs to the /, '').replace(/\.$/, '');
      }
      const kw = data.keywords?.find(k => k.category === 'Family & Domains');
      return kw?.name ?? '(family not annotated)';
    }

    case 'uniprot_id':
      return data.primaryAccession ?? '';

    case 'wb_predicted_mw_kda': {
      const mw = data.sequence?.molWeight;
      return mw ? String(Math.round(mw / 1000)) : 'N/A';
    }

    case 'signal_peptide_range': {
      const sp = data.features?.find(
        f => f.type === 'Signal' || f.type === 'signal peptide' ||
             (typeof f.type === 'string' && f.type.toLowerCase().includes('signal'))
      );
      if (!sp) return 'No signal peptide annotated';
      const s = sp.location?.start?.value ?? '?';
      const e = sp.location?.end?.value ?? '?';
      return `Residues ${s}–${e}`;
    }

    case 'ptm_glycosylation_sites': {
      const sites = (data.features ?? []).filter(f =>
        f.type === 'Glycosylation' ||
        (typeof f.type === 'string' && f.type.toLowerCase().includes('glycosy'))
      );
      if (!sites.length) return 'No glycosylation sites annotated in UniProt';
      return sites.slice(0, 5).map(s => {
        const pos = s.location?.position?.value ?? s.location?.start?.value ?? '?';
        const desc = s.description ?? s.type ?? 'Glycosylation';
        return `${desc} at position ${pos}`;
      }).join('; ');
    }

    case 'domain_list': {
      const domains = (data.features ?? []).filter(f =>
        f.type === 'Domain' || (typeof f.type === 'string' && f.type.toLowerCase() === 'domain')
      );
      if (!domains.length) return 'No domains annotated';
      return domains.map(d => d.description ?? d.name ?? 'unnamed').join(', ');
    }

    case 'subcellular_localization': {
      const loc = data.comments?.find(c => c.commentType === 'SUBCELLULAR LOCATION');
      if (!loc) return 'Unknown';
      const locs = (loc.subcellularLocations ?? [])
        .map(l => l.location?.value).filter(Boolean);
      return locs.join(', ') || 'Unknown';
    }

    case 'isoform_count': {
      const ap = data.comments?.find(c => c.commentType === 'ALTERNATIVE PRODUCTS');
      const count = ap?.isoforms?.length;
      return count ? `${count} isoforms annotated` : '1 (canonical sequence only)';
    }

    default:
      return null;
  }
}

// ── Protein Atlas field resolvers ──────────────────────────────────────────────
// Data shape from search_download API:
//   { Gene, Ensembl, "RNA tissue specificity": string, "RNA tissue specific nTPM": { tissue: "nTPM_value", … } }
// nTPM thresholds: High > 10, Medium 1–10, Low 0.1–1  (tissues at 0 are unexpressed, skip)
function _fromProteinAtlas(slug, data) {
  const nTPM = data?.['RNA tissue specific nTPM'] ?? {};
  const specificity = data?.['RNA tissue specificity'] ?? '';

  // Sort by expression descending
  const sorted = Object.entries(nTPM)
    .map(([tissue, val]) => [tissue, parseFloat(val)])
    .filter(([, v]) => !isNaN(v) && v > 0)
    .sort(([, a], [, b]) => b - a);

  const high   = sorted.filter(([, v]) => v > 10).map(([t]) => t);
  const medium = sorted.filter(([, v]) => v >= 1 && v <= 10).map(([t]) => t);
  const low    = sorted.filter(([, v]) => v > 0 && v < 1).map(([t]) => t);

  switch (slug) {
    case 'tissue_high':
      if (high.length) return high.slice(0, 6).join(', ');
      // No high-expression tissues — surface the specificity category as context
      if (specificity) return `(none above 10 nTPM — ${specificity})`;
      return '(none)';
    case 'tissue_med':
      if (medium.length) return medium.slice(0, 6).join(', ');
      return '(none)';
    case 'tissue_low':
      if (low.length) return low.slice(0, 6).join(', ');
      return '(none)';
    default:
      return null;
  }
}

// ── Boster scraping resolvers ──────────────────────────────────────────────────
function _fromBosterHtml(slug, html, biomarker) {
  switch (slug) {
    case 'wb_image_url': {
      const m = html.match(
        /https:\/\/(?:www\.)?bosterbio\.com\/media\/catalog\/product\/[^"' ]+?(?:wb|western-blot|WB)[^"' ]*?\.(?:jpg|jpeg|png)/i
      );
      return m?.[0] ?? '[WB image not found on Boster gene card]';
    }

    case 'ihc_image_url': {
      const m = html.match(
        /https:\/\/(?:www\.)?bosterbio\.com\/media\/catalog\/product\/[^"' ]+?ihc[^"' ]*?\.(?:jpg|jpeg|png)/i
      );
      return m?.[0] ?? '[IHC image not found on Boster gene card]';
    }

    case 'main_wb_antibody_sku': {
      // Match Boster SKU patterns: PA####, PB####, BA####-##, A#####, M#####
      const m = html.match(/\b(PA\d{3,6}|PB\d{3,6}|BA\d{3,6}(?:-\d+)?|A\d{4,6}|M\d{4,6}(?:-\d+)?)\b/);
      return m?.[1] ?? '[antibody SKU not found]';
    }

    case 'wb_antibody_citations': {
      const m = html.match(/(\d{1,4})\s*(?:citation|publication|paper|reference|cited)/i);
      return m?.[1] ?? '[citation count not found]';
    }

    case 'pub_count': {
      const m = html.match(/(\d{1,4})\s*(?:publication|paper|article|citation)/i);
      return m?.[1] ?? '[publication count not found]';
    }

    case 'assay_apps': {
      const apps = [];
      if (/\bwestern\s*blot\b|\bWB\b/i.test(html)) apps.push('WB');
      if (/\bimmunohistochem|\bIHC\b/i.test(html)) apps.push('IHC');
      if (/\bELISA\b|\benzyme.linked/i.test(html)) apps.push('ELISA');
      if (/\bflow\s*cytom|\bFACS\b|\bFC\b/i.test(html)) apps.push('FC');
      if (/\bimmunofluores|\bIF\b/i.test(html)) apps.push('IF');
      if (/\bimmunoprec|\bIP\b/i.test(html)) apps.push('IP');
      return apps.length ? apps.join(' · ') : '[assay apps not detected]';
    }

    case 'wb_antibody_url': {
      const m = html.match(
        /href="(https:\/\/(?:www\.)?bosterbio\.com\/[^"]+(?:antibod|picoband|PA|PB|BA)[^"]+)"/i
      );
      return m?.[1] ?? `https://www.bosterbio.com/catalogsearch/result/?q=${biomarker}+antibody`;
    }

    default:
      return `[${slug}: not extractable from Boster HTML]`;
  }
}

// ── Computed resolvers ─────────────────────────────────────────────────────────
function _computed(slug, biomarker, urlMap, resolved) {
  switch (slug) {
    case 'uniprot_url': {
      const acc = resolved['uniprot_id'] ?? '';
      if (acc && acc !== 'N/A' && !acc.startsWith('[')) {
        return `https://www.uniprot.org/uniprot/${acc}`;
      }
      // Try to extract accession from the REST URL
      const m = (urlMap.uniprot ?? '').match(/\/uniprotkb\/([A-Z0-9]+)/i);
      return m ? `https://www.uniprot.org/uniprot/${m[1]}` : '[uniprot_url: provide uniprot in urlMap]';
    }

    case 'proteinatlas_url': {
      const url = urlMap.proteinatlas ?? '';
      // Strip trailing .json if present (legacy urlMaps), otherwise use as-is
      return url ? url.replace(/\.json$/, '') : `https://www.proteinatlas.org/${biomarker}/tissue`;
    }

    case 'boster_gene_url':
      return urlMap.boster_gene_card ?? `https://www.bosterbio.com/bosterbio-gene-info-cards/${biomarker}`;

    default:
      return null;
  }
}

// ── Main export: hydrateAll ────────────────────────────────────────────────────
const UNIPROT_SLUGS = new Set([
  'gene_symbol', 'gene_full_name', 'gene_aliases', 'protein_superfamily', 'uniprot_id',
  'wb_predicted_mw_kda', 'signal_peptide_range', 'ptm_glycosylation_sites',
  'domain_list', 'subcellular_localization', 'isoform_count',
]);
const PA_SLUGS = new Set(['tissue_high', 'tissue_med', 'tissue_low']);
const BOSTER_SLUGS = new Set([
  'wb_image_url', 'ihc_image_url', 'main_wb_antibody_sku', 'wb_antibody_citations',
  'wb_antibody_url', 'pub_count', 'assay_apps',
]);
const COMPUTED_SLUGS = new Set(['uniprot_url', 'proteinatlas_url', 'boster_gene_url']);

/**
 * Resolve a list of slugs to their values in one batched call.
 * Fetches each source at most once; parallelizes UniProt + PA fetches.
 *
 * @param {string[]} slugs  — which slugs to resolve
 * @param {string}   biomarker — gene symbol, e.g. "ADAMTS14"
 * @param {object}   urlMap — { uniprot, proteinatlas, boster_gene_card }
 * @returns {Promise<Record<string, string>>}
 */
export async function hydrateAll(slugs, biomarker, urlMap = {}) {
  const needsUniprot   = slugs.some(s => UNIPROT_SLUGS.has(s));
  const needsPA        = slugs.some(s => PA_SLUGS.has(s));
  const needsBoster    = slugs.some(s => BOSTER_SLUGS.has(s));

  // Parallel fetch all needed sources
  const [uniprotResult, paResult, bosterResult] = await Promise.all([
    needsUniprot && urlMap.uniprot
      ? _fetchJson(urlMap.uniprot, 'UniProt').catch(e => ({ _error: e.message }))
      : Promise.resolve(urlMap.uniprot ? null : { _error: 'uniprot URL not provided' }),
    needsPA
      ? _fetchProteinAtlasTissue(biomarker).catch(e => ({ _error: e.message }))
      : Promise.resolve(null),
    needsBoster
      ? (() => {
          const url = urlMap.boster_gene_card
            ?? `https://www.bosterbio.com/bosterbio-gene-info-cards/${biomarker}`;
          return _fetchHtml(url).catch(e => ({ _htmlError: e.message }));
        })()
      : Promise.resolve(null),
  ]);

  const resolved = {};

  // First pass: all source-backed slugs
  for (const slug of slugs) {
    if (UNIPROT_SLUGS.has(slug)) {
      if (!uniprotResult || uniprotResult?._error) {
        resolved[slug] = `[UniProt error: ${uniprotResult?._error ?? 'no data'}]`;
      } else {
        resolved[slug] = _fromUniProt(slug, uniprotResult) ?? `[${slug}: not found in UniProt]`;
      }
    } else if (PA_SLUGS.has(slug)) {
      if (!paResult || paResult?._error) {
        resolved[slug] = `[ProteinAtlas error: ${paResult?._error ?? 'no data'}]`;
      } else {
        resolved[slug] = _fromProteinAtlas(slug, paResult) ?? `[${slug}: not found in PA]`;
      }
    } else if (BOSTER_SLUGS.has(slug)) {
      if (!bosterResult) {
        resolved[slug] = `[${slug}: Boster fetch skipped]`;
      } else if (bosterResult?._htmlError) {
        resolved[slug] = `[Boster error: ${bosterResult._htmlError}]`;
      } else {
        resolved[slug] = _fromBosterHtml(slug, bosterResult, biomarker);
      }
    }
    // Computed slugs done in second pass
  }

  // Second pass: computed slugs (may reference resolved values)
  for (const slug of slugs) {
    if (COMPUTED_SLUGS.has(slug)) {
      resolved[slug] = _computed(slug, biomarker, urlMap, resolved) ?? `[${slug}: compute failed]`;
    }
  }

  return resolved;
}

/**
 * Resolve a single slug. Convenience wrapper around hydrateAll.
 */
export async function hydrate(slug, biomarker, urlMap = {}) {
  const result = await hydrateAll([slug], biomarker, urlMap);
  return result[slug] ?? `[${slug}: unknown]`;
}

/**
 * Build a standard urlMap from known IDs.
 */
export function buildUrlMap({ uniprotAccession, ensemblId, gene }) {
  const map = {};
  if (uniprotAccession) {
    map.uniprot = `https://rest.uniprot.org/uniprotkb/${uniprotAccession}.json`;
  }
  if (ensemblId && gene) {
    // Human-readable browseable URL — tissue data is fetched via search_download API internally
    map.proteinatlas = `https://www.proteinatlas.org/${ensemblId}-${gene}`;
  }
  if (gene) {
    map.boster_gene_card = `https://www.bosterbio.com/bosterbio-gene-info-cards/${gene}`;
  }
  return map;
}

/**
 * All script-hydratable slugs (Phase 1 — no LLM required).
 */
export const SCRIPT_HYDRATABLE_SLUGS = [
  ...UNIPROT_SLUGS, ...PA_SLUGS, ...BOSTER_SLUGS, ...COMPUTED_SLUGS,
];
```
<!-- /GUILDOS-FILE -->
