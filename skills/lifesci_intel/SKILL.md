# lifesci_intel

<!-- GUILDOS-SKILL name="lifesci_intel" -->

<!-- GUILDOS-DESC -->
Track and operate the multi-stage ingestion pipeline (PMC, PubMed, NIH grants, university crawl, tagger), the customer-dossier smoke tests, and the recurring backfill-status report. Codifies the cadence and self-correction rule. Use for the 2-hourly backfill-status report, running the locked 10-customer and 20-customer smoke cohorts, and self-correcting queued ingestion work.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/lifesci_intel/SKILL.md -->
```markdown
---
name: lifesci_intel
description: >-
  Track and operate the multi-stage ingestion pipeline (PMC, PubMed, NIH grants,
  university crawl, tagger), the customer-dossier smoke tests, and the recurring
  backfill-status report. Codifies the cadence and self-correction rule. Use for
  the 2-hourly backfill-status report, running the locked 10-customer and
  20-customer smoke cohorts, and self-correcting queued ingestion work.
---

# LifeSci Intel — ingestion & backfill operations

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/lifesci_intel/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto lifesci_intel <action>` (or read the source directly).

## Actions

- **reportBackfillStatus** — Run the 2-hourly backfill-status report: a table of all known LifeSci Intel ingestion backfills with %, delta since last report, and status (active/queued/blocked/complete). Self-corrects when there are queued items and no active ones.
- **runSmokeCohort** — Run the locked 10-customer smoke #24 (customer-dossier) against the cohort in smoke-cohort.json. Quality bar: ≥80% PASS of the active (verified) cohort. See ~/CLAUDE.md customer-dossier section for the inclusion rule.
- **runSmokeExpanded** — Run the 20-customer expanded smoke against a fresh Zoho Books customer sample. Tests pipeline robustness over a broader population. Currently stable at 19/20 PASS after the smoke #6 author-id-union fix.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/lifesci_intel/index.js -->
```js
/**
 * LifeSci Intel skill book — operational knowledge for the BosterNexus LifeSci
 * Intel ingestion initiative (the work governed by ~/CLAUDE.md primary
 * responsibility). All scripts live under `~/bosternexus/scripts/lifesci-intel/`.
 *
 * This book is the canonical reference for the recurring "backfill status"
 * report, the smoke-test cadence, and the self-correction rule for queued work.
 */

export const skillBook = {
  id: "lifesci_intel",
  title: "LifeSci Intel — ingestion & backfill operations",
  description:
    "Track and operate the multi-stage ingestion pipeline (PMC, PubMed, NIH grants, university crawl, tagger), the customer-dossier smoke tests, and the recurring backfill-status report. Codifies the cadence and self-correction rule.",
  steps: [],
  toc: {
    reportBackfillStatus: {
      description:
        "Run the 2-hourly backfill-status report: a table of all known LifeSci Intel ingestion backfills with %, delta since last report, and status (active/queued/blocked/complete). Self-corrects when there are queued items and no active ones.",
      input: {},
      output: { markdown_table: "string", correction_signal: "string or null" },
      howTo: `
**Run the report (one-shot):**
\`\`\`
cd ~/bosternexus
node scripts/lifesci-intel/report-backfill-status.mjs
\`\`\`
For machine-readable JSON, add \`--json\`.

**State + delta semantics:**
- State persisted at \`D:/textmining/state/backfill-status.json\` — last 20 runs are
  kept so deltas can compare against the most recent prior run.
- Each backfill row has \`{id, done, total}\`. \`%\` = done/total (or "n/a" when
  total is unknown/open-ended). \`Δ\` = done(current) - done(prior); "new" on
  first appearance.
- Status is one of: ✅ \`complete\`, 🔵 \`active\` (process running),
  ⚪ \`queued\` (not started or paused), 🔴 \`blocked\` (waiting on external).

**Cadence rule (locked 2026-05-21):**
- The report SHOULD be run every 2 hours. This is independent of the locked
  smoke-test cadence (which runs only at "critical points" per the customer-
  dossier protocol in ~/CLAUDE.md).
- The 2h cron in this Claude session (\`yes proceed\`, job 95935585) is one
  trigger. The OS-level Startup-folder loop (\`ensure-stack-loop.cmd\`) is
  another trigger and the one that survives Claude session restarts.

**Self-correction rule (locked 2026-05-21):**
- If a report shows 0 \`active\` AND ≥1 \`queued\`, the chaperon SHOULD pick the
  highest-impact queued backfill and start it. The report prints the recommended
  ids on the line beginning \`⚠️ SELF-CORRECT:\`.
- "Highest impact" hierarchy: (1) Per-researcher PubMed publication backfill
  (Kipnis-class gap), (2) Tagger sweep over remaining PMC fulltext, (3) ORCID
  enrichment, (4) Conference abstracts, (5) Supplier citations, (6) author-row
  consolidation, (7) PubMed daily updatefiles.
- Do NOT auto-kick-off if any tracker shows \`blocked\` and the block is on an
  external dependency (e.g., Supabase down, no admin). In that case the report
  states the block and waits.

**Backfills currently tracked (15 rows):**
1. PMC corpus ingest
2. PubMed bulk baseline ingest
3. PubMed author-email backfill (corresp records → li_pubmed_author_emails)
4. edge_authors total (PMC + PubMed combined)
5. Grant-PI email linker (Item 16)
6. Tagger sweep — PubMed abstracts
7. Tagger sweep — PMC fulltext (original 614K gap)
8. mv_author_active_pi (campaign-audience MV)
9. Per-researcher PubMed publication backfill (Kipnis-class gap)
10. Conference abstracts ingest (ASH/AACR/ASCO/bioRxiv)
11. Supplier citation extraction (Item 13)
12. edge_authors consolidation (split-row merge)
13. ORCID enrichment (li_jobs #2)
14. PubMed daily updatefiles (post-baseline)
15. Bucket 2 JS-form crawl (Item 17)

**Adding a new backfill to the report:** edit \`gatherBackfills()\` in
report-backfill-status.mjs and add a row with \`{id, name, done, total, status,
label}\`. The state file backward-compats automatically (new ids show \`new\`
in Δ on first run after adding).
`,
    },

    runSmokeCohort: {
      description:
        "Run the locked 10-customer smoke #24 (customer-dossier) against the cohort in smoke-cohort.json. Quality bar: ≥80% PASS of the active (verified) cohort. See ~/CLAUDE.md customer-dossier section for the inclusion rule.",
      input: {},
      output: { result_summary: "string" },
      howTo: `
\`\`\`
cd ~/bosternexus
node scripts/lifesci-intel/smoke-cohort-dossier.mjs
\`\`\`
Cohort lives at \`docs/smoke-cohort.json\`. The script enforces the open-web
verification rule: only members with \`open_web_verification.status === 'verified'\`
count toward the PASS bar. Quarantined entries are reported separately.

**Run only at declared critical points** (post-backfill, pre-campaign-launch,
monthly drift check). Not on idle cycles; not as a progress metric.
`,
    },

    runSmokeExpanded: {
      description:
        "Run the 20-customer expanded smoke against a fresh Zoho Books customer sample. Tests pipeline robustness over a broader population. Currently stable at 19/20 PASS after the smoke #6 author-id-union fix.",
      input: {},
      output: { result_summary: "string" },
      howTo: `
\`\`\`
cd ~/bosternexus
node scripts/lifesci-intel/expand-smoke-20.mjs
\`\`\`
Uses cached \`docs/zb-customer-candidates.json\`. Re-pull via curl to
\`zb_contacts\` (Boster Nexus / Supabase) to refresh the sample.

PASS criterion (post-2026-05-21 relaxation): email present (= institutional
credential, by construction of Zoho Books) + tagged≥3 + author-id matched
via the email-direct ∪ name-match UNION. Strict institution-token match is
display-only and not gating.
`,
    },
  },
};
```
<!-- /GUILDOS-FILE -->
