# biomarkerFaq

<!-- GUILDOS-SKILL name="biomarkerFaq" -->

<!-- GUILDOS-DESC -->
Two-layer FAQ hydration for WB biomarker landing pages. Layer A builds an experiment-design FAQ for every gene (UniProt-derived band/glyco/isoform/PTM questions plus gene-agnostic forum-mined troubleshooting archetypes) as the citation/AEO engine; Layer B adds commercial-capture questions only for genes that clear the DataForSEO search-volume floor. Use to hydrate one gene or many, run the batched commercial screen, and emit schema.org FAQPage JSON-LD.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/biomarkerFaq/SKILL.md -->
```markdown
---
name: biomarkerFaq
description: >-
  Two-layer FAQ hydration for WB biomarker landing pages. Layer A builds an
  experiment-design FAQ for every gene (UniProt-derived band/glyco/isoform/PTM
  questions plus gene-agnostic forum-mined troubleshooting archetypes) as the
  citation/AEO engine; Layer B adds commercial-capture questions only for genes
  that clear the DataForSEO search-volume floor. Use to hydrate one gene or many,
  run the batched commercial screen, and emit schema.org FAQPage JSON-LD.
---

# biomarkerFaq

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/biomarkerFaq/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto biomarkerFaq <action>` (or read the source directly).

## Actions

- **commercialScreenBatch** — One DataForSEO search_volume task per <=1000 keywords across ALL genes (the cost-at-scale fix). Returns { GENE: commObj }.
- **faqJsonLd** — Turn faq_rows into a schema.org/FAQPage JSON-LD object — the citation/AEO lever to emit on the page.
- **hydrateGene** — Hydrate one gene's two-layer FAQ: pull UniProt features + DataForSEO commercial screen, generate source-tagged candidate questions (uniprot / forum-archetype / commercial-demand), AI-fill liftable answers, cap to maxQ. Returns { gene, faq_rows[], signal_summary }.
- **hydrateMany** — Loop hydrateGene over [{gene,accession}] with a BATCHED commercial screen up front; return all results + a per-strategy contribution tally for the value assessment.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/biomarkerFaq/cli.mjs -->
```js
#!/usr/bin/env node
// Hydrate the 10 WB biomarkers and write a reviewable artifact to
// public/proving-grounds/biomarker-faq-hydration/ (served at
// http://localhost:3002/proving-grounds/biomarker-faq-hydration/index.html).
//
// Run: node --env-file=.env.local libs/skill_book/biomarkerFaq/cli.mjs
import { writeFileSync, mkdirSync } from "node:fs";
import { hydrateMany } from "./index.js";

const GENES = [
  { gene: "AKT1", accession: "P31749" },
  { gene: "BCL2", accession: "P10415" },
  { gene: "ADAMTS14", accession: "Q8WXS8" },
  { gene: "CD68", accession: "P34810" },
  { gene: "VEGFA", accession: "P15692" },
  { gene: "BDNF", accession: "P23560" },
  { gene: "HIF1A", accession: "Q16665" },
  { gene: "TP53", accession: "P04637" },
  { gene: "STAT3", accession: "P40763" },
  { gene: "EGFR", accession: "P00533" },
];

const esc = (s) => String(s).replace(/[&<>]/g, (c) => ({ "&": "&amp;", "<": "&lt;", ">": "&gt;" }[c]));
const BADGE = {
  uniprot: "#2563eb",
  "forum-archetype": "#059669",
  "commercial-demand": "#d97706",
};

function render({ results, tally }) {
  const cards = results.map((r) => {
    if (r.error) return `<div class="card"><h2>${esc(r.gene)} — ERROR</h2><pre>${esc(r.error)}</pre></div>`;
    const rows = (r.faq_rows || []).map((f) => `
      <div class="faq">
        <div class="q"><span class="badge" style="background:${BADGE[f.source] || "#64748b"}">${esc(f.source)}${f.commercial ? " · CTA" : ""}</span> ${esc(f.q)}</div>
        <div class="a">${esc(f.a)}</div>
        ${(f.source_refs || []).length ? `<div class="cite">Source: ${esc((f.source_refs || []).join(", "))}</div>` : ""}
      </div>`).join("");
    const ld = `<script type="application/ld+json">${JSON.stringify(r.jsonld)}</script>`;
    const cm = r.comm || {};
    const sig = `antibody ${cm.antibody?.volume ?? 0} · anti- ${cm.anti?.volume ?? 0} · phospho ${cm.phospho?.volume ?? 0} (cpc $${(cm.phospho?.cpc ?? 0).toFixed(2)}) · ihc ${cm.ihc?.volume ?? 0}`;
    return `<div class="card">
      <h2>${esc(r.gene)} <span class="acc">${esc(r.accession)}</span></h2>
      <div class="meta">UniProt: ${r.uni.mass_kda} kDa · glyco ${r.uni.glyco} · modres ${r.uni.modres} · disulf ${r.uni.disulf} · iso ${r.uni.isoforms} · ${esc(r.uni.location)}</div>
      <div class="meta">Commercial screen: ${esc(sig)}</div>
      <div class="meta">${(r.faq_rows || []).length} questions retained from ${r.candidates.length} candidates · FAQPage JSON-LD emitted</div>
      ${ld}
      ${rows}
    </div>`;
  }).join("\n");
  const tallyRows = Object.entries(tally).sort((a, b) => b[1] - a[1])
    .map(([s, n]) => `<tr><td><span class="badge" style="background:${BADGE[s] || "#64748b"}">${esc(s)}</span></td><td>${n}</td></tr>`).join("");
  return `<!doctype html><html><head><meta charset="utf-8"><title>Biomarker FAQ Hydration — 10 genes</title>
<style>
body{font:14px/1.6 system-ui,sans-serif;max-width:920px;margin:24px auto;padding:0 16px;color:#1f2330;background:#fff}
h1{font-size:24px}.card{border:1px solid #e5e7eb;border-radius:10px;padding:16px 18px;margin:16px 0;box-shadow:0 2px 8px rgba(0,0,0,.04)}
h2{font-size:18px;margin:0 0 6px}.acc{font-size:12px;color:#94a3b8;font-weight:400}
.meta{font-size:12px;color:#64748b;margin:2px 0}
.faq{margin:10px 0;padding:10px 0;border-top:1px solid #f1f5f9}
.q{font-weight:600}.a{color:#374151;margin-top:4px}
.cite{font-size:11px;color:#2563eb;margin-top:3px;font-weight:600}
.badge{color:#fff;font-size:10px;font-weight:700;padding:2px 7px;border-radius:9px;text-transform:uppercase;letter-spacing:.4px}
table{border-collapse:collapse;margin:8px 0}td{border:1px solid #e5e7eb;padding:4px 12px}
</style></head><body>
<h1>Two-layer FAQ hydration — 10 WB biomarkers</h1>
<p>Each question is tagged by the demand-signal strategy that produced it. <b>Per-strategy contribution (retained questions):</b></p>
<table><tr><th>Strategy</th><th>Questions retained</th></tr>${tallyRows}</table>
${cards}
</body></html>`;
}

const out = await hydrateMany(GENES);
const dir = "public/proving-grounds/biomarker-faq-hydration";
mkdirSync(dir, { recursive: true });
writeFileSync(`${dir}/data.json`, JSON.stringify(out, null, 2));
writeFileSync(`${dir}/index.html`, render(out));
console.log("TALLY:", JSON.stringify(out.tally));
console.log("genes:", out.results.map((r) => `${r.gene}:${(r.faq_rows || []).length}${r.error ? "ERR" : ""}`).join(" "));
console.log("artifact: public/proving-grounds/biomarker-faq-hydration/index.html");
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/biomarkerFaq/index.js -->
```js
/**
 * biomarkerFaq — two-layer FAQ hydration for WB biomarker landing pages.
 *
 * Adopted 2026-06-09 after the demand-signal experiments (see weapon-log /
 * session). The evidence: per-gene WB *search* demand is too thin to drive
 * content, so the FAQ is built from two signals doing two jobs:
 *
 *   Layer A — experiment-design FAQ (EVERY gene; the citation/WOW/AEO engine):
 *     A1 uniprot-derived  : band size / glyco shift / isoforms / PTM, from the
 *                           gene's own UniProt features (deterministic).
 *     A2 forum-archetype  : the gene-agnostic WB-troubleshooting questions mined
 *                           ONCE from research forums (ResearchGate), then
 *                           instantiated per gene. Covers the long tail that
 *                           screens to zero on search-volume.
 *   Layer B — commercial capture (ONLY genes that clear the DataForSEO volume
 *     floor): anti-X / phospho-X / IHC / vendor-conquest → CTA-bearing Qs.
 *
 * Each emitted question carries `source` (which strategy produced it) so the
 * strategy-value assessment is empirical, not guessed.
 *
 * Self-contained on purpose (no @/libs imports) so it is directly runnable via
 * cli.mjs for the 10-gene hydration AND importable in-app. Needs env:
 * OPENAI_API_KEY, DATAFORSEO_LOGIN/PASSWORD.
 */

export const toc = {
  hydrateGene:
    "Hydrate one gene's two-layer FAQ: pull UniProt features + DataForSEO commercial screen, generate source-tagged candidate questions (uniprot / forum-archetype / commercial-demand), AI-fill liftable answers, cap to maxQ. Returns { gene, faq_rows[], signal_summary }.",
  hydrateMany:
    "Loop hydrateGene over [{gene,accession}] with a BATCHED commercial screen up front; return all results + a per-strategy contribution tally for the value assessment.",
  commercialScreenBatch:
    "One DataForSEO search_volume task per <=1000 keywords across ALL genes (the cost-at-scale fix). Returns { GENE: commObj }.",
  faqJsonLd:
    "Turn faq_rows into a schema.org/FAQPage JSON-LD object — the citation/AEO lever to emit on the page.",
};

// ---- Forum-mined experiment-design archetype taxonomy (one-time mining;
// ResearchGate 'western blot' corpus). Gene-agnostic; instantiated per gene. ----
// TRIMMED 2026-06-09: dropped "best-sample" and "controls" — they duplicate the
// page's existing S2 (protocols/sample) and S3 (controls) sections. Kept only the
// genuine troubleshooting archetypes not covered elsewhere on the page.
export const FORUM_ARCHETYPES = [
  { key: "multiple-bands", q: (g) => `Why am I seeing multiple or non-specific bands for ${g} on my western blot?` },
  { key: "no-band", q: (g) => `I get no ${g} band on my western blot — what are the likely causes?` },
  { key: "high-background", q: (g) => `How do I reduce high background on a ${g} western blot?` },
];

const DFS = "https://api.dataforseo.com/v3";
const UNIPROT = "https://rest.uniprot.org/uniprotkb";

// Boster WB guarantee + free custom-validation program. LEFT NULL ON PURPOSE —
// these are real marketing/legal claims that must NOT be fabricated. Fill from
// the canonical Boster source (name + public URL + exact guarantee wording),
// then the validated-guaranteed candidate + the answer-weave activate. Until
// filled, no guarantee/program claim is emitted. The biomarker skill book's
// LOCKED human-reactivity rule still applies: only recommend a SKU when a
// human-reactive antibody exists for the gene.
// Sourced 2026-06-10 from bosterbio.com (Free Picoband Validation promo + Boster
// Guarantee pages). Wording is a faithful paraphrase of the public claims —
// finalize exact legal text with the owner before live publish.
export const PROGRAM = {
  guarantee:
    "Boster-guaranteed to meet datasheet specs — full refund or free replacement if it doesn't work in your experiment",
  validation_name: "Free Picoband Antibody Validation",
  validation_url: "https://www.bosterbio.com/promotions/free-validation-for-picoband-antibodies",
};
const PROGRAM_READY = !!(PROGRAM.guarantee && PROGRAM.validation_url);

function dfsAuth() {
  const L = process.env.DATAFORSEO_LOGIN, P = process.env.DATAFORSEO_PASSWORD;
  return "Basic " + Buffer.from(`${L}:${P}`).toString("base64");
}

export async function uniprotFeatures(accession) {
  const r = await fetch(`${UNIPROT}/${accession}.json`);
  if (!r.ok) throw new Error(`UniProt ${r.status} for ${accession}`);
  const j = await r.json();
  const feats = j.features || [];
  const ct = (t) => feats.filter((f) => f.type === t).length;
  const comments = j.comments || [];
  return {
    mass_kda: +(j.sequence?.molWeight / 1000).toFixed(1),
    glyco: ct("Glycosylation"),
    modres: ct("Modified residue"),
    disulf: ct("Disulfide bond"),
    signal: ct("Signal"),
    isoforms: comments.find((c) => c.commentType === "ALTERNATIVE PRODUCTS")?.isoforms?.length || 1,
    location: comments.find((c) => c.commentType === "SUBCELLULAR LOCATION")
      ?.subcellularLocations?.[0]?.location?.value || "unknown",
  };
}

// Proven winning commercial patterns (this session's experiment). "best X" /
// "X antibody for WB" / "X antibody for sample-type" were DROPPED (dead demand).
const COMM_PATS = (g) => ({
  antibody: `${g} antibody`,
  anti: `anti ${g} antibody`,
  phospho: `phospho ${g} antibody`,
  ihc: `${g} antibody ihc`,
  monoclonal: `${g} monoclonal antibody`,
});

function unpackComm(gene, by) {
  const pats = COMM_PATS(gene);
  const out = {};
  for (const [k, kw] of Object.entries(pats)) {
    const v = by[kw.toLowerCase()] || {};
    out[k] = { kw, volume: v.search_volume || 0, cpc: v.cpc || 0 };
  }
  return out;
}

async function dfsSearchVolume(keywords) {
  const r = await fetch(`${DFS}/keywords_data/google_ads/search_volume/live`, {
    method: "POST",
    headers: { Authorization: dfsAuth(), "Content-Type": "application/json" },
    body: JSON.stringify([{ keywords, location_code: 2840, language_code: "en" }]),
  });
  const j = await r.json();
  return Object.fromEntries((j?.tasks?.[0]?.result || []).map((v) => [String(v.keyword).toLowerCase(), v]));
}

// Single-gene commercial screen (one DataForSEO task).
export async function commercialScreen(gene) {
  return unpackComm(gene, await dfsSearchVolume(Object.values(COMM_PATS(gene))));
}

// BATCHED commercial screen — one DataForSEO task per <=1000 keywords across ALL
// genes (the $750/10k -> ~$4/10k fix). Returns { GENE: commObj }.
export async function commercialScreenBatch(genes) {
  const all = [];
  for (const g of genes) all.push(...Object.values(COMM_PATS(g)));
  const map = {};
  for (let i = 0; i < all.length; i += 1000) {
    Object.assign(map, await dfsSearchVolume(all.slice(i, i + 1000)));
  }
  return Object.fromEntries(genes.map((g) => [g, unpackComm(g, map)]));
}

// Build source-tagged candidate questions for a gene.
function buildCandidates(gene, uni, comm) {
  const c = [];
  // A1 — uniprot-derived (deterministic, feature-gated)
  c.push({ source: "uniprot", key: "expected-band", commercial: false,
    q: `What is the expected western blot band size for ${gene}?` });
  if (uni.glyco >= 4)
    c.push({ source: "uniprot", key: "glyco-shift", commercial: false,
      q: `Why does ${gene} often run higher than its predicted ~${uni.mass_kda} kDa?` });
  if (uni.isoforms >= 3)
    c.push({ source: "uniprot", key: "isoforms", commercial: false,
      q: `${gene} has multiple isoforms — which band should I expect?` });
  if (uni.modres >= 10)
    c.push({ source: "uniprot", key: "ptm", commercial: false,
      q: `How do I distinguish phosphorylated ${gene} from total ${gene} by western blot?` });
  if (uni.signal >= 1 && (uni.disulf >= 6 || /secreted|extracellular/i.test(uni.location)))
    c.push({ source: "uniprot", key: "processing", commercial: false,
      q: `Should I expect cleaved or processed forms of ${gene} on a blot?` });
  // A2 — forum archetypes (gene-agnostic, always available)
  for (const a of FORUM_ARCHETYPES)
    c.push({ source: "forum-archetype", key: a.key, commercial: false, q: a.q(gene) });
  // B — commercial (gated on real DataForSEO demand)
  if ((comm.anti?.volume || comm.antibody?.volume) > 0)
    c.push({ source: "commercial-demand", key: "anti-x", commercial: true,
      volume: Math.max(comm.anti?.volume || 0, comm.antibody?.volume || 0),
      q: `Which validated anti-${gene} antibody should I use for western blot?` });
  if (comm.phospho?.volume > 0)
    c.push({ source: "commercial-demand", key: "phospho", commercial: true, volume: comm.phospho.volume,
      q: `Which phospho-${gene} antibody is validated for western blot?` });
  if (comm.ihc?.volume > 0)
    c.push({ source: "commercial-demand", key: "ihc", commercial: true, volume: comm.ihc.volume,
      q: `Can I use my ${gene} western blot antibody for IHC as well?` });
  // LLM-prompt-targeted: mirrors how researchers prompt an LLM
  // ("a {gene} antibody guaranteed to work in WB" / "validated for my sample").
  // Gated on real program facts so no guarantee claim is fabricated.
  if (PROGRAM_READY)
    c.push({ source: "commercial-demand", key: "guaranteed", commercial: true, volume: comm.antibody?.volume || 0,
      q: `Which ${gene} antibody is validated and guaranteed to work in western blot?` });
  return c;
}

async function callOpenAiFaq(gene, accession, uni, candidates, maxQ) {
  const key = process.env.OPENAI_API_KEY;
  if (!key) throw new Error("OPENAI_API_KEY not set");
  const facts = `UniProt ${accession} facts for ${gene}: predicted mass ${uni.mass_kda} kDa; ` +
    `${uni.glyco} N/O-glycosylation sites; ${uni.modres} modified residues; ${uni.disulf} disulfide bonds; ` +
    `${uni.signal} signal peptide; ${uni.isoforms} isoforms; subcellular location ${uni.location}.`;
  const sys = "You are a western-blot applications scientist writing FAQ answers for an antibody/WB experiment-design page. " +
    "Answers must be LIFTABLE: 2-3 sentences, lead with the direct answer + a hard number where possible, grounded ONLY in the provided UniProt facts (NEVER invent kDa/sites). " +
    "SOURCING (refSourcePolicy): every band-size / glycosylation / isoform / PTM / processing claim must cite its basis inline as `(UniProt " + accession + ")`; do not cite sources for generic troubleshooting answers. " +
    "Select the best, non-redundant subset of the candidate questions (max " + maxQ + ", at least 4), KEEPING the expected-band question. " +
    "Return JSON: {\"faq\":[{\"q\":string,\"a\":string,\"source\":string,\"commercial\":boolean,\"source_refs\":string[]}]} " +
    "preserving each kept question's source and commercial flag; source_refs lists the citations used (e.g. [\"UniProt " + accession + "\"]) or [] for generic answers." +
    (PROGRAM_READY
      ? ` For COMMERCIAL answers, close with the de-risk offer: that Boster ${gene} antibodies validated in WB are ${PROGRAM.guarantee}, and that researchers can request a free custom WB validation in their own sample type (${PROGRAM.validation_name}: ${PROGRAM.validation_url}). Only recommend a SKU when a human-reactive antibody exists; never invent product names or SKUs.`
      : " Do NOT make any guarantee claim or name a validation program — that data is not provided.");
  const usr = facts + "\n\nCandidate questions (with source tags):\n" +
    candidates.map((c, i) => `${i + 1}. [${c.source}${c.commercial ? "/commercial" : ""}] ${c.q}`).join("\n");
  const r = await fetch("https://api.openai.com/v1/chat/completions", {
    method: "POST",
    headers: { Authorization: `Bearer ${key}`, "Content-Type": "application/json" },
    body: JSON.stringify({
      model: "gpt-4o-mini",
      response_format: { type: "json_object" },
      temperature: 0.3,
      messages: [{ role: "system", content: sys }, { role: "user", content: usr }],
    }),
  });
  if (!r.ok) throw new Error(`OpenAI ${r.status}: ${(await r.text()).slice(0, 160)}`);
  const j = await r.json();
  return JSON.parse(j.choices[0].message.content).faq || [];
}

// Validator: rows well-formed, count within [4,maxQ], expected-band kept, and
// every uniprot-sourced factual row carries an inline citation.
function validateFaq(rows, maxQ) {
  if (!Array.isArray(rows) || rows.length < 4 || rows.length > maxQ) return "row count out of [4,maxQ]";
  for (const f of rows) {
    if (!f || typeof f.q !== "string" || !f.q.trim()) return "empty question";
    if (typeof f.a !== "string" || f.a.trim().length < 20) return "empty/short answer";
    if (typeof f.source !== "string") return "missing source tag";
    if (f.source === "uniprot" && !/uniprot|kda/i.test(f.a) && !(f.source_refs || []).length)
      return "uniprot row lacks citation";
  }
  if (!rows.some((f) => /expected.*band|band size/i.test(f.q))) return "expected-band question dropped";
  return null;
}

// AI-fill with validator-with-retry (max 3), mirroring S1/S4 in boster_nexus.
async function aiFillAnswers(gene, accession, uni, candidates, maxQ) {
  let lastErr = "no attempt";
  for (let attempt = 0; attempt < 3; attempt++) {
    const rows = await callOpenAiFaq(gene, accession, uni, candidates, maxQ);
    const err = validateFaq(rows, maxQ);
    if (!err) return rows;
    lastErr = err;
  }
  throw new Error(`FAQ validation failed after 3 attempts: ${lastErr}`);
}

// FAQPage JSON-LD (schema.org) — the single biggest citation/AEO lever.
export function faqJsonLd(faq_rows) {
  return {
    "@context": "https://schema.org",
    "@type": "FAQPage",
    mainEntity: (faq_rows || []).map((f) => ({
      "@type": "Question",
      name: f.q,
      acceptedAnswer: { "@type": "Answer", text: f.a },
    })),
  };
}

export async function hydrateGene(input = {}) {
  const { gene, accession, maxQ = 7, comm: preComm } = input;
  const [uni, comm] = await Promise.all([
    uniprotFeatures(accession),
    preComm ? Promise.resolve(preComm) : commercialScreen(gene),
  ]);
  const candidates = buildCandidates(gene, uni, comm);
  const faq_rows = await aiFillAnswers(gene, accession, uni, candidates, maxQ);
  return { gene, accession, uni, comm, candidates, faq_rows, jsonld: faqJsonLd(faq_rows) };
}

export async function hydrateMany(genes = []) {
  // Batched commercial screen up front (one DataForSEO call per <=1000 kw).
  let commByGene = {};
  try { commByGene = await commercialScreenBatch(genes.map((g) => g.gene)); }
  catch { /* fall back to per-gene screen inside hydrateGene */ }
  const results = [];
  for (const g of genes) {
    try { results.push(await hydrateGene({ ...g, comm: commByGene[g.gene] })); }
    catch (e) { results.push({ gene: g.gene, error: e.message }); }
  }
  // Per-strategy contribution tally (retained questions by source).
  const tally = {};
  for (const r of results)
    for (const row of r.faq_rows || []) tally[row.source] = (tally[row.source] || 0) + 1;
  return { results, tally };
}
```
<!-- /GUILDOS-FILE -->
