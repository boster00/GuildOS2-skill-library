# NexusPimRefactor

<!-- GUILDOS-SKILL name="NexusPimRefactor" -->

<!-- GUILDOS-DESC -->
Reusable 7-step recipe to refactor an application's separate product systems into ONE unified, reality-tested Product Information Management system. Parameterized over `app` (the application being refactored) and `externalSystem` (the order/commerce system the adapters + order/bundle workflow target). Proven end-to-end on Boster Nexus + Zoho Books/Magento (quest a6fc9d83). Use when an app runs two-or-more separate product systems with no single source of truth and must become one canonical PIM as a permanent reusable capability.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/NexusPimRefactor/SKILL.md -->
```markdown
---
name: NexusPimRefactor
description: >-
  Reusable 7-step recipe to refactor an application's separate product systems
  into ONE unified, reality-tested Product Information Management system.
  Parameterized over `app` (the application being refactored) and
  `externalSystem` (the order/commerce system the adapters + order/bundle
  workflow target). Proven end-to-end on Boster Nexus + Zoho Books/Magento
  (quest a6fc9d83). Use when an app runs two-or-more separate product systems
  with no single source of truth and must become one canonical PIM as a
  permanent reusable capability.
---

# Nexus PIM Refactor

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/NexusPimRefactor/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto NexusPimRefactor <action>` (or read the source directly).

## Actions

- **buildCursorOperationalLayer** — Step 5 — build a Cursor operational layer (reusable venture + embedded comms UI + templated quick-prompt orders) that ONLY orchestrates/supervises/triggers/monitors; heavy lifting stays in the backend jobs.
- **buildExternalAdapters** — Step 3 — build API-driven PIM↔external adapters for {app}'s live site and {externalSystem}; API > script > DB-mutation-fallback-only, never direct DB mutation as the primary path.
- **buildOrderBundleWorkflow** — Step 6 — build the {externalSystem} post-order Bundle & Save screening workflow: discount + ops notification + one-or-many vendor POs + downstream fulfillment prep.
- **buildResumableJobSystem** — Step 4 — make ingestion tracked, resumable jobs whose canonical execution state lives in DB job tables + a dashboard, recoverable across process death (NOT in any agent/runner process).
- **commitCanonicalFormatAndConverter** — Step 1 — commit {app}'s EXISTING product structure as the canonical master product format (sourced + cited, never invented; user must explicitly approve it) and build a generic mapperJson-driven Product Format Converter plus a native in-{app} equivalent.
- **integrateRealityTestedMvp** — Step 7 — integrate all six systems into ONE reality-tested end-to-end {app} PIM MVP; a single real chained pass exercises every component together (no stubs), live irreversible external-production cutovers deferred behind a user go-live anchor.
- **mergeIntoUnifiedPim** — Step 2 — merge {app}'s separate product systems into ONE unified PIM through the converter, every product carrying an external-ID bridge JSON (including {externalSystem}).
- **refPhaseAVendorIngestionSubstrate** — Reference: 2026-05-27 operator decision — Phase A substrate (pim_sync_protocols + pim_field_authority) is now landed. Future sync engines register here instead of hardcoding worker labels.
- **refRefactorContract** — Reference: the invariants, parameters (`app`, `externalSystem`), and end-state contract every step below honors. Read this first.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/NexusPimRefactor/index.js -->
```js
/**
 * NexusPimRefactor skill book — pure text registry.
 *
 * Forged from quest a6fc9d83-c436-4d6d-86fd-b2fda7101cb5 ("Refactor Boster
 * Nexus Into Unified PIM"). It encodes that quest's proven 7-step SMART plan as
 * a REUSABLE refactor recipe: take an application that runs two-or-more separate
 * product systems with no single source of truth, and refactor it into ONE
 * unified Product Information Management system as a permanent reusable
 * capability (not a one-off migration), reality-tested end-to-end.
 *
 * The plan is parameterized over two variables so a DIFFERENT app + external
 * system pair can re-run the same refactor:
 *   - `app`            — the application being refactored into a PIM
 *                        (e.g. "Boster Nexus"). Names the repo whose existing
 *                        product structure is committed as canonical (never
 *                        invented) and whose libs/pim/* the build lands in.
 *   - `externalSystem` — the order/commerce system the adapters + the
 *                        order/bundle screening workflow target
 *                        (e.g. "Zoho Books" / "Magento"). Names the
 *                        API-driven adapter target + the post-order screening
 *                        workflow's host.
 *
 * Each action value is a natural-language prompt describing how the adventurer
 * performs that step. The adventurer uses weapons (database, claudecli, the
 * generic Product Format Converter, etc.) inline. No JS logic here — this is a
 * pure text-prompt skill book exactly like `blacksmith`.
 *
 * Every `{app}` / `{externalSystem}` placeholder below is substituted with the
 * caller-supplied values at dispatch time.
 */

export const skillBook = {
  id: "NexusPimRefactor",
  title: "Nexus PIM Refactor",
  description:
    "Reusable 7-step recipe to refactor an application's separate product systems into ONE unified, reality-tested Product Information Management system. Parameterized over `app` (the application being refactored) and `externalSystem` (the order/commerce system the adapters + order/bundle workflow target). Proven end-to-end on Boster Nexus + Zoho Books/Magento (quest a6fc9d83).",

  toc: {
    refRefactorContract:
      "Reference: the invariants, parameters (`app`, `externalSystem`), and end-state contract every step below honors. Read this first.",
    refPhaseAVendorIngestionSubstrate:
      "Reference: 2026-05-27 operator decision — Phase A substrate (pim_sync_protocols + pim_field_authority) is now landed. Future sync engines register here instead of hardcoding worker labels.",
    commitCanonicalFormatAndConverter:
      "Step 1 — commit {app}'s EXISTING product structure as the canonical master product format (sourced + cited, never invented; user must explicitly approve it) and build a generic mapperJson-driven Product Format Converter plus a native in-{app} equivalent.",
    mergeIntoUnifiedPim:
      "Step 2 — merge {app}'s separate product systems into ONE unified PIM through the converter, every product carrying an external-ID bridge JSON (including {externalSystem}).",
    buildExternalAdapters:
      "Step 3 — build API-driven PIM↔external adapters for {app}'s live site and {externalSystem}; API > script > DB-mutation-fallback-only, never direct DB mutation as the primary path.",
    buildResumableJobSystem:
      "Step 4 — make ingestion tracked, resumable jobs whose canonical execution state lives in DB job tables + a dashboard, recoverable across process death (NOT in any agent/runner process).",
    buildCursorOperationalLayer:
      "Step 5 — build a Cursor operational layer (reusable venture + embedded comms UI + templated quick-prompt orders) that ONLY orchestrates/supervises/triggers/monitors; heavy lifting stays in the backend jobs.",
    buildOrderBundleWorkflow:
      "Step 6 — build the {externalSystem} post-order Bundle & Save screening workflow: discount + ops notification + one-or-many vendor POs + downstream fulfillment prep.",
    integrateRealityTestedMvp:
      "Step 7 — integrate all six systems into ONE reality-tested end-to-end {app} PIM MVP; a single real chained pass exercises every component together (no stubs), live irreversible external-production cutovers deferred behind a user go-live anchor.",
  },

  refPhaseAVendorIngestionSubstrate: `
Reference — Phase A of the 2026-05-27 PIM vendor-ingestion refactor is now
substrate. Operator decision: the existing pim_jobs + tickPimWorker flow is
ONE sync protocol (PIM -> live Magento upsert); Phase A generalizes this into
named protocols + a field-authority rules table so future protocols
(web <-> CDB, CDB -> Zoho, etc.) register into the same architecture.

What landed (in boster_nexus):
  - Migration 109 (supabase/migrations/109_pim_sync_protocols.sql)
      pim_sync_protocols — registry of named protocols. Each row is a
      directed edge (source_system -> target_system) backed by a
      queue table + a worker_label. schedule_cron NULL = trigger-only.
      4 seed rows: liveSite-upsert / -delete / -disable / -imagesUpdate.
  - Migration 110 (supabase/migrations/110_pim_field_authority.sql)
      pim_field_authority — per-field ownership rules. Sync engines
      consult this BEFORE writing. 6 seed rows: PIM default + zoho-books
      / website / live-magento / sync-worker overrides.
  - Idempotent seeder (scripts/_seed-sync-substrate.mjs)
  - Read endpoint (app/(private)/pim/sync/list/api/route.js)
  - Admin tab (app/(private)/pim/admin?tab=protocols → _ProtocolsPanel.js)

What did NOT change (deliberately, tracer-bullet scope):
  - pim_jobs.worker is still the dispatch key — no protocol_id added yet.
  - tickPimWorker is unchanged — worker IS the protocol name for now.
  - The Sync Protocols tab is READ-ONLY (no add/edit/delete UI).

Implication for future sync engines:
  When building a new sync engine (e.g. web -> CDB sync, CDB -> Zoho Books),
  do NOT hardcode a worker label in scripts/cron. Instead:
    1. INSERT a row into pim_sync_protocols (name, source_system,
       target_system, worker_label, schedule_cron).
    2. INSERT field-authority rules into pim_field_authority for any
       fields the new engine writes (or relies on as authoritative).
    3. The worker code reads the protocol row to discover its
       worker_label + schedule.
  This keeps protocol registration data-driven and visible in the admin UI.

See: libs/skill_book/pim → pimSyncProtocols, pimFieldAuthority for the
table schemas + match algorithm.
`.trim(),

  refRefactorContract: `
Reference contract for the Nexus → unified PIM refactor. Every step honors this.

## Parameters (substituted at dispatch)
- {app} — the application being refactored into a PIM (e.g. "Boster Nexus"). Its
  repo is where the existing product structure is committed-as-canonical and
  where the libs/pim/* modules land. NEVER invent a product model — adopt what
  {app} (and any sibling repo the user names) already defines.
- {externalSystem} — the order/commerce system the Step-3 adapters AND the
  Step-6 order/bundle screening workflow target (e.g. "Zoho Books", "Magento").

## Invariants (non-negotiable, proven on the source quest)
1. Canonical-not-invented: the canonical master product format is whatever {app}
   already defines (central product table + named indexed attributes + a
   nameless attr_1..N per-template block resolved via an attribute-definitions
   table). Commit THAT, cite where each element came from, never reinvent.
2. User-approval gate: the canonical format is final ONLY when the user
   explicitly approves it in a quest comment. This is a planned mid-quest
   co-develop halt — surface it, do not self-approve.
3. Generic converter: convert(sourcePayload, sourceFormat, destFormat,
   mapperJson) -> convertedPayload — declarative mapperJson, no shape
   hard-coded; Type-A by name, Type-B attr_1..N by attr_key resolved per
   template; round-trippable on identity keys. A native in-{app} equivalent
   must exist.
4. ONE PIM: merge {app}'s separate product systems into one canonical
   structure — do not keep them as two. Every product carries an external-ID
   mapping JSON (bridge to {externalSystem}, the site, CRM, GMC, future).
5. API-driven adapters: the live-site adapter is API-driven, NOT direct DB
   mutation. Priority API > script > DB-fallback-only (fallback inert + must
   refuse without explicit acknowledgement).
6. Execution state in the DB: jobs are resumable/inspectable/retryable and
   survive process death because state lives in job tables + dashboard, never
   in the runner/agent process.
7. Cursor orchestrates only: the operational layer's import graph must be
   structurally unable to reach ingestion compute; heavy lifting is backend
   jobs + scripts + ChatGPT API.
8. One-or-many POs: one Sales Order may split into MULTIPLE vendor POs — never
   a hard-wired single-PO path; prove it BOTH ways.
9. Reuse-not-fork: reuse {app}'s existing tested clients/resolvers via wrappers
   or injection; never duplicate/fork them.
10. Reality-tested MVP is the deliverable: the deliverable is the reusable
    feature set built end-to-end as a permanent capability, integrated into one
    MVP proven by a single real chained pass with NO stubs — NOT a one-off
    migration run, and NOT the live external-production cutover.

## End-state contract
The buildable + reality-tested scope is the deliverable and must be genuinely
verified. Irreversible live external-production cutovers (e.g. non-dryRun live
product create on the site; live {externalSystem} financial writes) are
DEFERRED-WITH-RECORDED-REASON behind a user [co-develop-with-user] go-live
anchor — they need real prod credentials + explicit in-person approval. Record
the exact reason in the step notes/items; never fake a "live in production"
claim and never fail the refactor for a correctly-deferred cutover.
`.trim(),

  commitCanonicalFormatAndConverter: `
STEP 1 — Commit {app}'s canonical master product format + build the generic Product Format Converter.

1. Look up the EXISTING product structure {app} (and any sibling repo the user
   names) already defines: the central product table, the named indexed
   attributes, and the nameless attr_1..N per-product-template block resolved
   via an attribute-definitions table. [IMPORTANT DETAIL] commit THAT exact
   structure as the canonical master product format, citing per element which
   repo/file it came from — do NOT invent new structures.
2. [IMPORTANT DETAIL] the canonical format is final ONLY when the user
   explicitly approves it in a quest comment. Store the committed format as a
   json item on the quest; what the user must see to approve = the central
   product table definition + the named indexed attributes + the nameless
   attr_1..N per-template mapping (each citing its source repo/file) + the
   converter's mapper-JSON contract. [co-develop-with-user] this explicit
   user-approval gate is a planned mid-quest co-develop halt — surface it and
   wait; never treat a draft as final.
3. Build a generic Product Format Converter with signature
   convert(sourcePayload, sourceFormat, destFormat, mapperJson) ->
   convertedPayload. Declarative mapperJson per destination field
   { from, transform?, default? }; Type-A named columns by name, Type-B
   attr_1..N by attr_key resolved per template via attribute_definitions;
   storage stays nameless TEXT; round-trippable on identity keys.
   [IMPORTANT DETAIL] an equivalent native function must exist inside {app}
   (e.g. {app}/libs/pim/convert.js) — the canonical format IS {app}'s
   products-table shape; the converter is the generic mapper around it.
4. Verify for real: convert a real sample product BOTH directions through a
   real mapper JSON and confirm the round-trip; confirm the native {app}
   function produces the same result (native ≡ exposed wrapper). Prove generic
   by a second partner mapper producing a different shape with zero code change.
`.trim(),

  mergeIntoUnifiedPim: `
STEP 2 — Merge {app}'s separate product systems into ONE unified PIM with the external-ID bridge.

Input: the approved canonical format + the converter (Step 1), {app}'s existing
product systems (e.g. a live-site sync/export structure and an ingestion
structure), and {app}'s daily/internal product export.

1. First enhance the ingestion path (competitor/upstream products, AI-summarized
   if applicable, converted to canonical via the Step-1 converter), then import
   the current {app} products from the internal/daily export.
2. [IMPORTANT DETAIL] merge the two-or-more existing product systems into ONE
   PIM (one canonical structure) — do NOT keep them as separate systems. Both
   flow THROUGH the Step-1 converter into the single canonical products table;
   track origin per row (e.g. a pim_source column).
3. [IMPORTANT DETAIL] every product carries an external-ID mapping JSON field
   ({externalSystem}, the {app} website, CRM, Google Merchant Center, future)
   as the bridge layer the Step-3 adapters fill in.
4. Verify for real on real representative data, non-destructively (insert →
   readback → delete; restore the table). If applying schema DDL to the live DB
   needs a privileged runner not available, persist the bridge under a
   metadata fallback and DEFER the full bulk import with the exact recorded
   reason (it is a Step-4 resumable-job concern; bulk-upserting the full
   catalog would overwrite real rows).
`.trim(),

  buildExternalAdapters: `
STEP 3 — Build API-driven PIM↔external adapters ({app} live site + {externalSystem}).

1. Build adapters between the unified PIM and external systems — initially the
   {app} live site and {externalSystem}. Both adapters REUSE the Step-1
   converter (not forked).
2. [IMPORTANT DETAIL] the {app} live-site adapter is API-driven, NOT direct DB
   mutation. Build a custom {app} API that accepts batch product JSON +
   category assignments + metadata and creates products correctly (correct
   categories + reindex). Enforce structurally: the adapter source has no DB
   client/SQL (assert by source-scan); any DB fallback is inert and refuses
   without explicit { allowDbFallback, acknowledgeNotPreferred }.
3. [IMPORTANT DETAIL] preferred priority is API > script > DB-mutation
   fallback-only; DB export acceptable, DB import not preferred.
4. Verify the mechanism for real on real data via the non-mutating dryRun path
   (real HTTP call against the live running route, NO external write).
   [co-develop-with-user] the live non-dryRun external-production cutover is
   DEFERRED-with-recorded-reason — needs real prod creds + explicit in-person
   go-live approval; the live cutover is a one-line wiring once they exist.
`.trim(),

  buildResumableJobSystem: `
STEP 4 — Build the resumable job system + dashboard-driven ingestion workflow.

1. Make ingestion tracked jobs, not fire-and-forget. [IMPORTANT DETAIL] the
   canonical execution state lives in the jobs / workflow-jobs / watchdog-jobs
   tables + the dashboard, NOT in any runner/Cursor process — the runner can
   die and the work resumes from the persisted job state. Jobs are resumable,
   inspectable, retryable, recoverable. Weapons/scripts hydrate the dashboards
   so ChatGPT, Cursor, and humans can all monitor.
2. Use a single-flight claim (conditional UPDATE-RETURNING / lock pattern) so
   one job is claimed by exactly one runner; persist a progress cursor.
3. Verify for real: kill the trigger/runner mid-job and confirm a FRESH tick
   resumes from the persisted cursor and processes ONLY the remaining rows
   (provably not a restart). Prove it survives multiple separate process
   deaths in the DB row alone.
`.trim(),

  buildCursorOperationalLayer: `
STEP 5 — Build the Cursor cloud-agent operational layer.

1. Build a dedicated reusable Cursor venture + an embedded Cursor-agent comms
   UI inside {app} with templated quick-prompt direct orders over the tested
   flows (e.g. ingest a product line / competitor company / bundle batch).
2. [IMPORTANT DETAIL] heavy lifting is ChatGPT API + weapon scripts + backend
   jobs; Cursor ONLY orchestrates/supervises/triggers/monitors. Enforce
   structurally: the operational-layer module imports ONLY the job-enqueue
   symbol — its import graph must be provably unable to reach any
   ingestion-compute symbol; an enqueued job has zero compute until the Step-4
   backend runner executes it.
3. Verify for real: a templated order through the embedded UI enqueues a real
   backend job that the Step-4 runner executes (real rows written by the
   runner, none by the UI/enqueue path). [co-develop-with-user] a live STAFFED
   Cursor venture operated by the real team is DEFERRED-with-recorded-reason —
   needs a provisioned Cursor cloud account + the actual human team; the
   mechanism (orders → enqueue → runner → monitor agent) is built + verified.
`.trim(),

  buildOrderBundleWorkflow: `
STEP 6 — Build the {externalSystem} order + bundle screening workflow.

1. Orders are entered manually into {externalSystem} initially. After Sales
   Order creation a {externalSystem} function screens the order contents; if
   Bundle & Save products are detected it (A) applies the BNS discount,
   (B) notifies the ops team with dropship + vendor + required-action
   fulfillment instructions, and (C) creates one-or-many vendor POs and
   prepares the downstream fulfillment workflow.
2. [IMPORTANT DETAIL] ONE Sales Order may split into MULTIPLE Purchase Orders —
   one-or-many POs is not negotiable; never a hard-wired single-PO path.
3. [IMPORTANT DETAIL] reuse {app}'s existing tested {externalSystem} client +
   bundle resolvers + ops-notify builder via injection/wrappers — never
   import-couple or fork them (autonomous-run safety).
4. Verify for real on real representative data: prove one-or-many BOTH ways
   (multi-vendor SO → multiple POs, each PO only its own vendor's lines, no
   bleed; single-vendor SO → exactly one PO), exact discount numbers, and the
   ops notification contents. [co-develop-with-user] the live {externalSystem}
   write (SO/discount/PO) is DEFERRED-with-recorded-reason — irreversible
   external financial mutation; needs real prod creds + explicit in-person
   go-live approval. The commit=true path is the single wired cutover; the
   module refuses to write without an injected live client.
`.trim(),

  integrateRealityTestedMvp: `
STEP 7 — Integrate all six systems into ONE reality-tested end-to-end MVP.

1. Integrate the six systems into ONE functioning end-to-end {app} PIM and
   reality-test it: a single real end-to-end pass that exercises every
   component together — canonical format + converter → unified PIM ingestion
   via the resumable jobs → adapter creates a product on the live site (dryRun
   for the deferred cutover) → Cursor-triggered templated order → the
   {externalSystem} order/bundle screening with one-or-many POs. NOT a stub or
   demo; every module's real output feeds the next (assert the chain).
2. [IMPORTANT DETAIL] the deliverable is the reusable feature set built
   end-to-end into {app} as a permanent capability (NOT a one-off migration
   run), integrated into one reality-tested MVP that actually works end to end.
3. [co-develop-with-user] the integrated, irreversible-ish end-to-end go-live
   of the multi-system build is co-developed with and supervised by the user at
   execution time — surface the go-live anchor for an explicit user decision;
   the two irreversible live external-production cutovers (live non-dryRun site
   product create; live {externalSystem} financial writes) remain
   DEFERRED-with-recorded-reason behind that anchor.
4. Confirm the integrated MVP works end to end with the user. The reality-
   tested MVP — not the live prod cutover — is the verified deliverable.
`.trim(),
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->
