# cjgeo

<!-- GUILDOS-SKILL name="cjgeo" -->

<!-- GUILDOS-DESC -->
Run CJGEO article-pipeline and content-rendering workflows for the CJGEO SEO Content Writer SaaS. Use to drive the 8-step article writing pipeline (create article, SEO keyword research, competitor topic benchmark, GEO prompts, generate/poll/adopt draft), operate the Content Magic web UI via CIC, read CJGEO's own Supabase (a SEPARATE project from any consuming app), generate/fill images via gpt-image-2, screenshot articles, dedup articles by title, audit the draft-state cron, and render content with Shadow DOM CSS isolation. Includes the wire-level REST API catalog, the locked verify-after-mutate discipline, and the DataForSEO seed-fallback pattern. CJGEO sits at the inductive (broad-generation) end of the content lifecycle — do not editorially reject its raw output.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/cjgeo/SKILL.md -->
```markdown
---
name: cjgeo
description: >-
  Run CJGEO article-pipeline and content-rendering workflows for the CJGEO SEO
  Content Writer SaaS. Use to drive the 8-step article writing pipeline (create
  article, SEO keyword research, competitor topic benchmark, GEO prompts,
  generate/poll/adopt draft), operate the Content Magic web UI via CIC, read
  CJGEO's own Supabase (a SEPARATE project from any consuming app), generate/fill
  images via gpt-image-2, screenshot articles, dedup articles by title, audit the
  draft-state cron, and render content with Shadow DOM CSS isolation. Includes the
  wire-level REST API catalog, the locked verify-after-mutate discipline, and the
  DataForSEO seed-fallback pattern. CJGEO sits at the inductive (broad-generation)
  end of the content lifecycle — do not editorially reject its raw output.
---

# CJGEO — SEO Content Writer Platform

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/cjgeo/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto cjgeo <action>` (or read the source directly).

## Actions

- **adoptDraft** — Adopt the generated draft into content_html via POST /api/articles/{id}/edit-draft action=adopt_draft. Idempotent no-op when not yet ready (returns 200 not_ready, NOT 409).
- **captureArticleScreenshot** — Screenshot a CJGEO article or demo page and upload to Supabase Storage for quest deliverables — use CIC, NOT Playwright scripts.
- **connectSupabase** — Read CJGEO's Supabase tables (content_magic_articles, etc.) directly — CJGEO is a SEPARATE Supabase project from any consuming app (GuildOS). Use when CIC can't read CJGEO content or you need an article's content_html / status programmatically.
- **createArticle** — Create a new CJGEO article via POST /api/articles and retain the returned article_id. LOCKED: must dedup-check by title against existing articles BEFORE creating — duplication has become a real operational issue.
- **deduplicateArticles** — Locate and remove duplicate CJGEO articles by title. Keep the row with the longest content_html; delete the rest. Run when [SB:cjgeo->createArticle] surfaces ≥ 2 matches, or as a periodic housekeeping sweep.
- **demoMode** — Understand and debug the CJGEO Demo Mode pipeline (tick.js state machine, screenshot storage, error patterns).
- **driveContentMagicUI** — Drive the CJGEO Content Magic web UI (cjgeoai.com/content-magic) via CIC: create article, research, generate, and ADOPT the draft. Includes the locked adopt step and the CIC-render-freeze caveat.
- **generateDraft** — Kick off async draft generation via POST /api/articles/{id}/edit-draft action=generate_draft. Returns 202; orchestrator owns polling.
- **generateImage** — Generate or restyle a UI image using gpt-image-2 via the openai_images weapon.
- **inspectCron** — Static inspection: read the CJGEO cron implementation files and verify code handles pending generations, stalled generations, retry logic, timeout windows, and failure resolution.
- **inspectRuntime** — Dynamic inspection: fetch runtime cron logs / observable evidence (cloud browser, server logs, or /tmp/cjgeo3000.log for dev) and confirm cron actively manages draft states.
- **keywordFullAuto** — Run the 8-stage SEO keyword research chain to completion via POST /api/articles/{id}/keyword with empty body (default action full_auto).
- **observability** — Discipline (testing-run only): API responses must surface enriched debug payload (action name, called backend function, state diff, mutation timestamp, correlation ID, sub-call latencies). Missing fields = test failure.
- **pollDraft** — Poll draft status every 5 minutes via POST /api/articles/{id}/edit-draft action=pull_status. 20-minute timeout triggers reinitiate. Orchestration owns cadence and timeout — API only owns the wire.
- **promptFullAuto** — Run the GEO prompt suggestion + commit chain via POST /api/articles/{id}/prompt with empty body (default action full_auto).
- **refApiActions** — Reference catalog: every CJGEO REST API endpoint and action, with HTTP method, path, body fields, success/error shapes, commit flag, and one-line behavior. The canonical wire spec for [SB:cjgeo->*] orchestration actions.
- **refDataForSEOSeedStrategy** — Reference (LOCKED): seed-keyword strategy for keywordFullAuto + topicFullAuto when DataForSEO returns empty data (kept=0). Head-term fallback pattern + serial-vs-parallel execution guidance.
- **refFullAgenticMode** — Reference: full_agentic_mode column + cron state machine on content_magic_articles. Cron-driven end-to-end pipeline for plan-gated tenants. New feature surface (May 2026).
- **refFullAutoGate** — Reusable precondition gate: verify keywordFullAuto + topicFullAuto have committed before calling generateDraft. ABORT on missing main_keyword / empty keywords[] / empty topics[].
- **refImagePlaceholderHandling** — Reference (LOCKED): when adopting a CJGEO article into a destination repo, detect <img> placeholders, fill them with gpt-image-2 (per [SB:cjgeo->generateImage]) via the openai_images weapon, store them so the destination serves them, and rewrite `src` so every image renders on the live page without 404 / broken-image errors.
- **refSubstrateFixesTimeline** — Reference: substrate fixes landed during the May 2026 hydration sweep. Localhost-loopback bug class on Vercel serverless, the 3 fixed routes, the 1 still-open route.
- **reinitiateDraft** — Re-trigger generation when pollDraft hits the 20-minute timeout. Re-calls generate_draft on the same article. Skill book owns when; API only owns that re-calling re-triggers.
- **renderContent** — Render content with Shadow DOM CSS isolation.
- **resolveSkillbook** — Overview for an executing agent: how to use this skill book to drive the CJGEO REST API end-to-end.
- **reviewSummary** — Closing report shape for an end-to-end CJGEO run: what succeeded, what retried, what mutated, any anomalies.
- **setMainKeyword** — Commit the article's main keyword via POST /api/articles/{id}/keyword action=set_main_keyword. Required before any topic or full-auto downstream workflow.
- **surfaceResult** — End of run: open the final article in CIC at /content-magic/{article_id} and surface the full asset state for user review.
- **topicFullAuto** — Run the competitor-topic benchmark chain to completion via POST /api/articles/{id}/topic with empty body (default action full_auto).
- **verifyCron** — Combine inspectCron (static) + inspectRuntime (dynamic) into a single verdict: cron behavior matches expectations.
- **verifyMutation** — Discipline: every commit-yes action MUST be verified by re-reading via get_assets (asset mutations) or get_html (content_html mutations). Pre/post diff is recorded in the agent log.
- **writeMigration** — Write a new database migration for CJGEO and refresh the schema dump.
- **writePipeline** — Run or modify the 8-step article writing pipeline. Steps 3 (keywordFullAuto) and 4 (topicFullAuto) are LOCKED prerequisites for Step 6 (generateDraft); Step 5 (promptFullAuto) is recommended but not LOCKED.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/cjgeo/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/skill_book/cjgeo`

Rationale layer (per CLAUDE.md §6). Append; supersede in place when a decision
changes.

---

## 2026-06-12 — CJGEO is the INDUCTIVE end: keep-longest dedup, no quality-gate, no spam-anxiety

**Decision.** CJGEO does broad inductive generation — wide subject sweep, affiliate
links, brand-naming, exploratory framing. When deduplicating CJGEO rows the rule is
"keep the longest `content_html`." Do NOT reject or alarm on affiliate links,
brand-name SEO patterns, or broad-stroke claims at this stage.

**Rejected alternative.** Apply editorial/quality filtering (drop "spammy" rows,
prefer concise extracts) at the CJGEO level.

**Why.** CJGEO's job is to maximize raw inductive material. Curation, reality-check,
and editorial gating live downstream (sanitize-cjgeo, adoption skill books,
rendered page). Keeping the longest variant maximizes the surface area downstream
curation has to work with; early filtering throws away signal that can't be
recovered.

**Consequence if violated.** Short "clean" extracts win over long messy ones,
starving downstream curation; a spam-anxious agent guts the output and breaks the
CJGEO→adoption pipeline.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/cjgeo/index.js -->
```js
/**
 * CJGEO skill book — knowledge registry for SEO content writer SaaS.
 *
 * ── LIFECYCLE ROLE (LOCKED) ─────────────────────────────────────────────────
 * CJGEO sits at the **inductive** end of the content lifecycle — broad
 * generation, wide subject sweep, brand-naming, affiliate references,
 * exploratory framing. It deliberately does NOT reality-check or editorially
 * curate; downstream processes do that (sanitize-cjgeo.mjs, the destination's
 * adoption skill book, the editorialized content file, the rendered page).
 *
 * Do NOT reject or alarm on a CJGEO article because it contains affiliate
 * links, brand-name SEO patterns (e.g. NCARB / ArchiPrep / Harbor Compliance
 * style references), broad-stroke claims, or other inductive-stage byproducts.
 * Those are expected output; quality concerns belong downstream.
 *
 * When deduplicating at CJGEO level, `keep longest content_html` is the right
 * rule (more raw inductive material = more downstream-curation surface area).
 * Don't second-guess based on perceived "spam" in the raw CJGEO row.
 */

export const skillBook = {
  id: "cjgeo",
  title: "CJGEO — SEO Content Writer Platform",
  description: "Run CJGEO article-pipeline and content-rendering workflows.",
  steps: [],
  toc: {
    writePipeline: {
      description: "Run or modify the 8-step article writing pipeline. Steps 3 (keywordFullAuto) and 4 (topicFullAuto) are LOCKED prerequisites for Step 6 (generateDraft); Step 5 (promptFullAuto) is recommended but not LOCKED.",
      howTo: `
**Pipeline flow (full_auto orchestration order — LOCKED ordering between SEO/GEO research and draft kickoff):**
1. [SB:cjgeo->createArticle] — create article row, retain \`article_id\`.
2. [SB:cjgeo->setMainKeyword] — commit the main keyword (required by steps 3 + 4).
3. [SB:cjgeo->keywordFullAuto] — SEO keyword research. **LOCKED prerequisite for step 6.**
4. [SB:cjgeo->topicFullAuto] — competitor topic benchmark. **LOCKED prerequisite for step 6.**
5. [SB:cjgeo->promptFullAuto] — GEO prompt suggestions. Recommended but **NOT LOCKED** — its absence is a soft warning, not an abort.
6. [SB:cjgeo->generateDraft] — kick off async draft generation. **ABORT if step 3 or step 4 did not commit** (gate at [SB:cjgeo->refFullAutoGate]).
7. [SB:cjgeo->pollDraft] → [SB:cjgeo->adoptDraft] — poll on a 5-min cadence; on \`ready\`, adopt to \`content_html\`.
7.5. [SB:cjgeo->refImagePlaceholderHandling] — on adoption into a destination repo, detect \`<img>\` placeholders, fill via \`gpt-image-2\` ([SB:cjgeo->generateImage]), store to the destination's public assets, and rewrite \`src\`. **LOCKED step on every adoption** — broken-image icons on the live page fail the destination's CIC visual-verify gate.
8. [SB:cjgeo->surfaceResult] + [SB:cjgeo->reviewSummary] — final user-channel surface.

**Legacy reference (internal pipeline naming — separate from full_auto orchestration above):**
1. Interpret Intent → IntentModel (ICP, claims, USPs, competitor hints)
2. Interpret Talk Points → TalkPoint[] (categorized, prioritized)
3. Collect & Validate Competitors → competitor URLs
4. Benchmark + Extract Competitor Talk Points → competitive analysis
5. Plan Sections → SectionPlan[]
6. Choose Sections → selected sections with rationale
7. Write Sections → prose content per section
8. Render HTML → final output

**Key files:**
- Pipeline: \`libs/monkey/pipelines/writeArticleLandingPipeline.ts\`
- Types: \`libs/monkey/references/marketingTypes.ts\` (IntentModel, TalkPoint, SectionPlan)
- Schemas: \`libs/monkey/references/marketingPageSchemas.ts\`
- See \`docs/CODE_RESPONSIBILITY_MAP.md\` for exact line numbers and manual override instructions per step.

**Content Magic asset whitelisting:** Only 4 asset keys allowed in outline generation: main_keyword, topics, keywords, prompts. Never allow arbitrary asset access.

**AI module:** All AI calls route through \`@/libs/monkey\`. Use \`runTask()\` for new features. Model tiers via env vars only, never hardcoded.
`,
    },
    driveContentMagicUI: {
      description:
        "Drive the CJGEO Content Magic web UI (cjgeoai.com/content-magic) via CIC: create article, research, generate, and ADOPT the draft. Includes the locked adopt step and the CIC-render-freeze caveat.",
      howTo: `
Operating the CJGEO Content Magic UI at \`https://cjgeoai.com/content-magic\` (authenticated as the operator's account).

**Flow (Writing Guide, left→right):**
1. **New article** → Article Title = the main keyword/topic; pick ICP (e.g. "General Life Scientist (Baseline ICP)"); optional Offer. → Create Article (opens \`/content-magic/<articleId>\`).
2. **Research and Plan:** Determine Main Keyword (Search → Save Main Keyword); Benchmark Competitors (Search → select competitors → Analyze selected competitors → review AI topic scores → Save topics); Research Keywords (SEO) (Vertical/Horizontal research → add → Evaluate Keywords with AI → Save keywords). Research Prompts (GEO) / Internal Links are optional.
3. **Write & Optimize → Edit Draft → "Generate New Draft":** select competitor pages, configure context/prompt/Page Template, set the **"Use custom templates"** switch as required, then **Generate Draft** (~25 credits). Generation runs server-side 3–15+ min (longer with custom template).

**LOCKED — a generated draft does NOT auto-populate the article body until ADOPTED (operator-confirmed 2026-05-16):**
> Intended path: after generation completes, click **"Edit Draft"** then click **"Adopt Draft"** — then the generated content is applied into the article editor body. If the editor still shows "Start writing your content here…", the draft was generated but NOT adopted.
> **Observed (unintended but actual) behavior:** simply ENTERING the Edit Draft UI and then EXITING it also adopts the draft — i.e. opening the Edit Draft modal and closing/navigating away applies the generated draft to the article body even without explicitly clicking "Adopt Draft". So after ANY Edit-Draft open/close cycle, treat the draft as adopted (it's persisted server-side on the article). Do not assume it's un-adopted just because you didn't click the explicit button.

**LOCKED — CIC render-freeze caveat (observed reproducibly 2026-05-16):** CIC can drive the CJGEO editor SHELL (toolbar, config modals, buttons via element refs) but the moment the actual generated-draft CONTENT renders (adopted body, "Improve Existing Draft" tab, fullscreen preview), CIC's renderer hangs — BOTH \`screenshot\` and \`get_page_text\` time out, reproducibly, fresh tab or not. The operator's own (non-CIC) browser is unaffected. Also: CIC-clicking the "Use custom templates" toggle itself reproducibly freezes the renderer (have the operator flip it, or it's already on). Implications: drive shell actions via DOM refs (find/read_page work when screenshot doesn't); to CAPTURE the adopted draft content, use **"Copy HTML to clipboard"** or **"Save article"** (DOM-clickable) and/or have the operator paste the HTML from their browser — never proxy/assume the draft, never loop CIC-recovery (locked validateCIC anti-dance). Final user-channel verification of the downstream artifact happens on the consuming app (e.g. the GuildOS quest at localhost:3002), which is a separate, non-frozen surface.

**Article persistence:** research steps + generation persist server-side per \`articleId\`; reloading recovers a frozen renderer without losing saved research. Articles are stored in CJGEO's Supabase \`content_magic_articles\` (separate project from any consuming app).
`,
    },
    connectSupabase: {
      description:
        "Read CJGEO's Supabase tables (content_magic_articles, etc.) directly — CJGEO is a SEPARATE Supabase project from any consuming app (GuildOS). Use when CIC can't read CJGEO content or you need an article's content_html / status programmatically.",
      howTo: `
CJGEO's data (articles, drafts) lives in **CJGEO's own Supabase project**, NOT the consuming app's. From a GuildOS (or other) session you have NO CJGEO Supabase creds in-env — get them from the CJGEO repo.

**Locked path (operator-confirmed 2026-05-16):**
1. **CJGEO repo:** \`C:/Users/xsj70/cjgeo\` (Next.js 15, dev port 3000). Its \`.env.local\` holds CJGEO's Supabase creds.
2. **Project:** CJGEO Supabase ref \`<REDACTED-PROJECT-REF>\` (\`https://<REDACTED-PROJECT-REF>.supabase.co\`) — distinct from GuildOS \`<REDACTED-PROJECT-REF>\`. Env var names in \`C:/Users/xsj70/cjgeo/.env.local\`: \`NEXT_PUBLIC_SUPABASE_URL\`, \`SUPABASE_SERVICE_ROLE_KEY\` (service role bypasses RLS), \`NEXT_PUBLIC_SUPABASE_ANON_KEY\`. (Supabase CLI in that repo is linked to this ref — \`supabase ...\` from there targets CJGEO; or use the connection string for psql.)
3. **Read pattern (no secret ever printed):** run a one-shot FROM the cjgeo repo so \`@supabase/supabase-js\` resolves and the env loads:
\`\`\`bash
cd C:/Users/xsj70/cjgeo && node --env-file=.env.local <<'EOF'
import { createClient } from "@supabase/supabase-js";
const db = createClient(process.env.NEXT_PUBLIC_SUPABASE_URL, process.env.SUPABASE_SERVICE_ROLE_KEY);
const { data } = await db.from("content_magic_articles")
  .select("id,title,status,content_html").eq("id", "<articleId>").maybeSingle();
// write data.content_html to a file in the consuming repo for downstream use
EOF
\`\`\`
Key tables: \`content_magic_articles\` (id, title, status [draft/adopted/...], content_html, outline jsonb, campaign_id, ...). The adopted/saved draft body is \`content_html\`. Article id is the \`/content-magic/<id>\` URL segment.
4. **Never** echo the service key to stdout. Never guess CJGEO creds/URLs — always source from \`C:/Users/xsj70/cjgeo\`. This is the reliable non-CIC path (CIC reproducibly freezes on CJGEO draft content — see \`driveContentMagicUI\`).
`,
    },
    renderContent: {
      description: "Render content with Shadow DOM CSS isolation.",
      howTo: `
**No @scope CSS** — never generate @scope CSS. Use Shadow DOM for isolation.

**Deprecated functions:** \`_wrapCssInScope\`, \`renderCustomCSS\` — do not use.

**Shadow DOM CSS injection:**
- External CSS: \`<link rel="stylesheet">\` inside shadow \`<head>\`
- Inline CSS: strip scope wrappers, place in \`<style>\` inside shadow root
`,
    },
    generateImage: {
      description: "Generate or restyle a UI image using gpt-image-2 via the openai_images weapon.",
      howTo: `
Use \`openai_images\` weapon (\`@/libs/weapon/openai_images\`).

**Generate (text → image):**
\`\`\`js
import { generate } from "@/libs/weapon/openai_images";
const [b64] = await generate({ prompt, size: "1024x1024", quality: "medium" }, userId);
// b64 is a base64 PNG string — decode with Buffer.from(b64, "base64")
\`\`\`

**Edit / style-transfer (image(s) + prompt → image):**
\`\`\`js
import { edit } from "@/libs/weapon/openai_images";
const imageBuffer = Buffer.from(existingB64, "base64");
const [b64] = await edit({ images: [imageBuffer], prompt, size: "1024x1024", quality: "medium" }, userId);
\`\`\`

**Model:** \`gpt-image-2\` — released April 2026, best text rendering, real-world visual fidelity.
**Sizes:** \`1024x1024\` | \`1024x1536\` (portrait) | \`1536x1024\` (landscape) | \`auto\`
**Quality:** \`low\` ($0.011/img) | \`medium\` ($0.042/img) | \`high\` ($0.167/img) — all per 1024×1024.
**DO NOT** pass \`response_format\` — unsupported param, causes error; b64_json is always returned.
**Credential:** OPENAI_API_KEY in profiles.env_vars → process.env fallback.
**Org verification:** gpt-image-2 requires OpenAI org verification; if the call returns a model-access error, the org must be verified at platform.openai.com before this works.

**CJGEO image editor context:**
The CJGEO main editor has an AI image generation panel previously powered by DALL-E
(\`dall-e-3\` or \`dall-e-2\`). The upgrade path is:
1. Find the API route or server action that calls OpenAI for image generation.
2. Change the model from \`dall-e-3\` (or \`dall-e-2\`) to \`gpt-image-2\`.
3. Remove \`response_format: "b64_json"\` from the request body if present — gpt-image-2 ignores it and may error; the response already contains b64_json.
4. Remove the \`style\` param if present — not supported by gpt-image-2.
5. Update supported size options if the UI exposes them (add \`1024x1536\`, \`1536x1024\`; remove \`1792x1024\` / \`1024x1792\` which were DALL-E 3 only).
6. Smoke-test: generate one image end-to-end from the editor UI and confirm it renders.
`,
    },
    writeMigration: {
      description: "Write a new database migration for CJGEO and refresh the schema dump.",
      howTo: `
**After any schema change, run:**
\`\`\`bash
supabase db dump --schema public > libs/reference-for-ai/database-schema.sql
\`\`\`

**Profiles key columns:** subscription_plan (free/starter/pro), stripe_customer_id, stripe_subscription_id, credits_remaining, payg_wallet, credits_reset_at.
`,
    },

    captureArticleScreenshot: {
      description: "Screenshot a CJGEO article or demo page and upload to Supabase Storage for quest deliverables — use CIC, NOT Playwright scripts.",
      howTo: `
**The correct tool for local Claude is CIC (Claude-in-Chrome MCP), never a Playwright node script.**
Playwright node scripts are a SW violation. The server-side \`screenshotUrl()\` in \`libs/demo/screenshot.js\` uses playwright-core for the Next.js API route — that is production pipeline code, not local-Claude tooling.

**Pattern: CIC navigate → screenshot → upload via supabase_storage weapon**

1. Get or create a CIC tab:
   \`tabs_context_mcp({ createIfEmpty: true })\`

2. Navigate to the article preview route (serves HTML with correct Content-Type):
   \`navigate({ url: "http://localhost:3000/api/demo/preview/{articleId}", tabId })\`
   Wait 2–3s for Tailwind CDN and images to load.

3. Take the screenshot:
   \`computer({ action: "screenshot", tabId, save_to_disk: true })\`
   The returned \`imageId\` (e.g. \`ss_abc123\`) is the handle.

4. **Uploading**: CIC screenshots are saved to a temp path returned in the result.
   Read that file, then upload using the \`supabase_storage\` weapon (\`libs/weapon/supabase_storage/index.js\`):
   \`\`\`js
   import { writeFile, readPublicUrl } from "@/libs/weapon/supabase_storage";
   // bucket is GuildOS_Bucket for quest artifacts:
   const buf = fs.readFileSync(tempPath);
   await writeFile({ path: \`quests/{questId}/{itemKey}.jpeg\`, content: buf, contentType: "image/jpeg", bucket: "GuildOS_Bucket" });
   const url = readPublicUrl({ path: \`quests/{questId}/{itemKey}.jpeg\`, bucket: "GuildOS_Bucket" });
   \`\`\`
   For CJGEO demo-screenshots bucket, set \`bucket: "demo-screenshots"\`.

5. Update the quest deliverable URL:
   \`\`\`js
   await sb.from("deliverables").update({ url, caption: "..." }).eq("id", deliverableId);
   \`\`\`

**Thumio notes (DO NOT use for stored artifacts):**
- Thumio (\`https://image.thum.io/get/fullpage/width/1200/crop/900/\`) is unreliable: free tier has 30+ second pending states.
- Apache Tomcat rejects \`%2F\` (URL-encoded slashes). Pass URLs unencoded: \`\${THUMIO_BASE}/\${pageUrl}\` — NO \`encodeURIComponent\`.
- Thumio renders Supabase Storage HTML as \`text/plain\` → blank screenshot. Always point Thumio at a Next.js route that returns \`Content-Type: text/html\`.
- Use Thumio ONLY as a \`<img src>\` fallback in the UI, never for stored quest artifact URLs.

**Demo article preview route:**
- \`/api/demo/preview/[articleId]\` → \`app/api/demo/preview/[articleId]/route.js\`
- Serves \`content_magic_articles.content_html\` as full HTML page with Tailwind CDN.
- Headers: \`Content-Type: text/html; charset=utf-8\`, \`X-Frame-Options: ALLOWALL\`, \`Cache-Control: no-store\`.
- No auth — intentionally public so Thumio and screenshot services can reach it.
`,
    },

    refApiActions: {
      description: "Reference catalog: every CJGEO REST API endpoint and action, with HTTP method, path, body fields, success/error shapes, commit flag, and one-line behavior. The canonical wire spec for [SB:cjgeo->*] orchestration actions.",
      howTo: `
Canonical REST API reference for the CJGEO content-magic pipeline. Used by SE scripts and orchestrators that reference per-action howTos via \`[SB:cjgeo->X]\` tags.

**Base URL:** env-aware — \`https://cjgeoai.com\` in prod, \`http://localhost:3000\` in dev.
**Auth:** every request MUST carry \`x-api-key: <CJGEO_API_KEY>\` header. Missing/invalid → \`401 {error:"Unauthorized"}\`.
**Error contract:** non-2xx responses are JSON \`{error: "<message>"}\`. Status codes: \`400\` bad input · \`401\` auth · \`404\` article not found · \`409\` state conflict · \`501\` not implemented (env-blocked) · \`500\` internal.
**Default JSON content type:** \`Content-Type: application/json\`.

### Conventions (apply across all endpoints)
- **UI Function Reuse.** Each action wraps the same underlying function the CJGEO web UI calls — API and UI mutations are observationally identical.
- **Action-based dispatch.** Endpoints route on a top-level \`action\` field in the request body. If \`action\` is omitted on endpoints that declare a default, the default action runs.
- **\`full_auto\` default.** Endpoints whose pipeline has a sensible end-to-end chain expose \`full_auto\` as default — POST with empty body \`{}\` runs the full chain.
- **Commit convention.** Each action is either commit-yes (writes to [DB:content_magic_articles]) or commit-no (computes/returns only). Commit-yes responses include \`{committed:true, ...}\`.

---

### 1. \`POST /api/articles/{articleId}\` — article core

No default \`full_auto\` for this endpoint.

- **\`import\`** *(commit)* — \`{action:"import", source_url}\` → imports a URL into content_html. Returns \`{committed, length}\`.
- **\`get_html\`** *(read)* — \`{action:"get_html"}\` → \`{content_html}\`. The verify path for any body-mutating commit.
- **\`get_assets\`** *(read)* — \`{action:"get_assets"}\` → \`{main_keyword, keywords[], topics[], prompts[], status, ...}\`. The verify path for asset-mutating commits. **Quirk vs older skill book text:** \`get_assets\` is on the article ROOT (\`/api/articles/{id}\`), NOT on \`/api/articles/{id}/keyword\`. Verified during the May 2026 pillar hydration when the \`/keyword\` path returned 404. The wire spec here is the correct one.
- **\`edit_title\`** *(commit)* — \`{action:"edit_title", title}\` → \`{committed, title}\`.
- **\`edit_html\`** *(commit)* — \`{action:"edit_html", content_html}\` → \`{committed, length}\`.

### 2. \`POST /api/articles/{articleId}/keyword\` — main keyword + SEO keyword research

Default action: \`full_auto\`.

- **\`set_main_keyword\`** *(commit)* — \`{action:"set_main_keyword", main_keyword}\` → \`{committed, main_keyword}\`. Required before topic/full-auto downstream.
- **\`search\`** *(read)* — \`{action:"search", query?}\` → keyword search candidates.
- **\`research_keyword_vertical\`** *(read)* — \`{action:"research_keyword_vertical", seed}\` → vertical expansion candidates.
- **\`research_keyword_horizontal\`** *(read)* — \`{action:"research_keyword_horizontal", seed}\` → horizontal expansion candidates.
- **\`evaluate_keyword_relevance\`** *(read)* — \`{action:"evaluate_keyword_relevance", keywords[]}\` → scored relevance.
- **\`save_keywords\`** *(commit)* — \`{action:"save_keywords", keywords[]}\` → \`{committed, count}\`.
- **\`remove_keywords\`** *(commit)* — \`{action:"remove_keywords", keywords[]}\` → \`{committed, count}\`.
- **\`full_auto\`** *(commit, default)* — \`{}\` → 8-stage chain: verify main_keyword → search → vertical → horizontal (first 3) → eval → commit. Returns \`{committed, summary:{search_count, vertical, horizontal, evaluated, kept}}\`.

### 3. \`POST /api/articles/{articleId}/topic\` — competitor topic benchmark

Default action: \`full_auto\`. Main keyword required for \`search\`, \`extract_topics\`, and \`full_auto\`.

- **\`search\`** *(read)* — competitor URL search seeded by main_keyword.
- **\`crawl_url\`** *(read)* — \`{action:"crawl_url", url}\` → page text/HTML for benchmark.
- **\`extract_topics\`** *(read)* — extracts topic candidates from crawled pages.
- **\`evaluate_topics\`** *(read)* — scores topics for relevance/competitive value.
- **\`save_topics\`** *(commit)* — \`{action:"save_topics", topics[]}\` → \`{committed, count}\`.
- **\`remove_topics\`** *(commit)* — \`{action:"remove_topics", topics[]}\` → \`{committed, count}\`.
- **\`full_auto\`** *(commit, default)* — \`{}\` → search → crawl → extract → evaluate → commit. Requires committed main_keyword.

### 4. \`POST /api/articles/{articleId}/prompt\` — GEO prompt suggestion

Default action: \`full_auto\`. Out of scope for this endpoint: prompt research, internal-link research.

- **\`suggest_prompts\`** *(read)* — \`{action:"suggest_prompts", funnel?:"bottom"|"top"|"middle"}\` → candidate prompts (default bottom funnel).
- **\`save_prompts\`** *(commit)* — \`{action:"save_prompts", prompts[]}\` → \`{committed, count}\`.
- **\`remove_prompts\`** *(commit)* — \`{action:"remove_prompts", prompts[]}\` → \`{committed, count}\`.
- **\`full_auto\`** *(commit, default)* — \`{}\` → suggest (bottom funnel) → commit.

### 5. \`POST /api/articles/{articleId}/edit-draft\` — draft generation + adoption

**No \`full_auto\`.** Async-by-contract draft lifecycle: \`generate_draft\` → poll → \`adopt_draft\`.

- **\`generate_draft\`** *(async write — outline only, not yet content_html)* — \`{action:"generate_draft", ...settings}\` → \`202 {article_id, status:"generating", poll, chat_id}\`. Supported settings (per locked spec):
  - \`mode\`: \`"new_draft"\` | \`"improve_existing_draft"\`
  - \`provide_feedback\`: boolean
  - \`competitor_pages\`: \`[url, ...]\`
  - \`competitor_page_titles\`: \`[title, ...]\`
  - \`context_prompt\`: string
  - \`use_default_context_prompt\`: boolean
  - \`generation_prompt_mode\`: \`"default"\` | \`"custom"\`
  - \`custom_generation_prompt\`: string
  - \`use_custom_template\`: boolean
  - \`page_layout_example_url\`: string
  - \`allow_image_generation\`: boolean (default true per SE script)
- **\`pull_status\`** *(read)* — \`{action:"pull_status"}\` → \`{status:"ready"|"not_ready"}\`. The polling read; no other side effects.
- **\`adopt_draft\`** *(conditional commit)* — \`{action:"adopt_draft"}\` →
  - if ready: \`200 {committed:true, status:"adopted", length}\` (writes content_html)
  - if not ready: \`200 {status:"not_ready"}\` (NOT 409 — per spec)

### 6. \`POST /api/articles/{articleId}/implementation\` — apply keywords/prompts/priorities to draft

Default action: \`full_auto\`.

- **\`suggest_priorities\`** *(read)* — orders pending keywords/prompts by impact.
- **\`suggest_implementation\`** *(read)* — proposes per-keyword/per-prompt placement edits.
- **\`evaluate_implementation\`** *(read)* — scores a proposed edit set.
- **\`evaluate_prompt_implementation\`** — **currently \`501 {error:"not implemented"}\`** (env-blocked; disclosed in spec).
- **\`evaluate_keyword_implementation\`** *(read)* — per-keyword post-edit score.
- **\`implement_all_keywords\`** *(commit, runs 3 loops)* — \`{action:"implement_all_keywords"}\` → \`{committed, loops:3, applied_count}\`.
- **\`full_auto\`** *(commit, default, runs 2 loops)* — \`{}\` → suggest_priorities → suggest_implementation → evaluate → commit. 2-loop cap.

### 7. \`POST /api/articles/{articleId}/repurpose\` — derivative content

**No \`full_auto\`. No commits** — every action is read-only / returns derived content; nothing writes to the article row.

- **\`social_media\`** — \`{action:"social_media", channel?, count?}\` → derived posts.
- **\`email\`** — \`{action:"email", angle?}\` → email copy.
- **\`ad_copy\`** — \`{action:"ad_copy", platform?, length?}\` → ad copy.

---

**Always-verify rule.** For every commit-yes action above, the orchestrator MUST re-read the changed surface via the corresponding read action ([SB:cjgeo->verifyMutation]). Wire-success (\`committed:true\`) is necessary but not sufficient.
`,
    },

    resolveSkillbook: {
      description: "Overview for an executing agent: how to use this skill book to drive the CJGEO REST API end-to-end.",
      howTo: `
This skill book is the wire-level + orchestration layer for the CJGEO content-magic REST API.

**Routing.** Per-action howTos here describe individual API actions; treat \`refApiActions\` as the catalog and the per-action entries (\`setMainKeyword\`, \`keywordFullAuto\`, \`topicFullAuto\`, \`promptFullAuto\`, \`generateDraft\`, \`pollDraft\`, \`adoptDraft\`, etc.) as the orchestration recipes.

**Verify-after-mutate.** After every commit-yes call, route through [SB:cjgeo->verifyMutation] — re-read via \`get_assets\` (asset mutations) or \`get_html\` (content_html mutations). The discipline is non-optional; the orchestration layer owns it.

**Timing belongs here.** Polling cadence, timeout windows, and re-initiate decisions are skill-book concerns ([SB:cjgeo->pollDraft], [SB:cjgeo->reinitiateDraft]). The API only owns the wire shape; this skill book owns \`when\`.

**Read before you act.** Read \`refApiActions\` once at session start so every subsequent action call has the spec in context. Defer to it on any wire-shape ambiguity.
`,
    },

    createArticle: {
      description: "Create a new CJGEO article via POST /api/articles and retain the returned article_id. LOCKED: must dedup-check by title against existing articles BEFORE creating — duplication has become a real operational issue.",
      howTo: `
Bootstraps an article row in [DB:content_magic_articles].

**LOCKED — dedup check BEFORE every create (operator lock 2026-05-26).** CJGEO creates a NEW row on every \`POST /api/articles\` regardless of whether a row with that title already exists. Without an explicit dedup check, batch hydration sweeps and SE re-runs accumulate duplicate articles — observed during the cloudstonedesigns pillar hydration where re-runs left 2-3 copies of the same satellite title in \`content_magic_articles\`. Every orchestrator / SE script / freestyle dispatch that calls \`createArticle\` MUST first check for an existing article with the same title and, if found, REUSE its \`article_id\` instead of creating a new row.

**Dedup check pattern (locked):**
1. Query [DB:content_magic_articles] for rows with \`title = <intended_title>\` (case-insensitive exact match recommended; trim whitespace + collapse whitespace runs before comparing).
2. **If 0 matches** → proceed with \`POST /api/articles\` as below.
3. **If exactly 1 match** → REUSE its \`id\` as \`article_id\`. Log the reuse decision. Do NOT create.
4. **If ≥ 2 matches** → STOP and surface State 1 to the user: "Duplicate articles detected for title <X>: <ids[]> with content_html lengths <lens[]>. Pick one to reuse, or run the dedup procedure first ([SB:cjgeo->deduplicateArticles])." The orchestrator does NOT auto-pick — preferring the longest \`content_html\` is the right default but the dedup decision is operator-owned, not silently inferred.

**Two paths to the dedup query — pick by what's available:**
- **Path A — CJGEO Supabase (preferred for batch / programmatic flows).** Per [SB:cjgeo->connectSupabase]: query CJGEO's own Supabase project (ref \`<REDACTED-PROJECT-REF>\`) for \`select id,title,length(content_html) from content_magic_articles where lower(trim(title)) = lower(trim(<intended>))\`. Service-role key required; never echoed.
- **Path B — CIC on the content-magic list (when human-driven freestyle dispatch).** Open \`https://cjgeoai.com/content-magic\` via CIC, read the visible article table, match by title. Slower; OK for one-off.

**Wire.** \`POST /api/articles\` with \`{title, keyword?, source_url?}\` → \`201 {article_id, imported}\`. \`imported\` is true when \`source_url\` was supplied and content_html was seeded from it. ONLY call after the dedup check above passed with 0 matches.

**Retain article_id.** Every downstream action — keyword, topic, prompt, edit-draft, implementation, repurpose — is keyed on \`{articleId}\` in the URL. Pin it in the orchestrator's state and pass it through every subsequent dispatch.

**ICP / offer toggles are NOT API-exposed.** The UI surface that picks "General Life Scientist (Baseline ICP)" / Offer is not part of the REST API. If the orchestrator needs ICP/offer context, store it as an article-context field via subsequent commit actions (\`edit_title\` / [SB:cjgeo->setMainKeyword] / \`edit_html\` body framing) — do NOT invent unsupported body fields. Document honestly when ICP/offer is required but cannot be set wire-side.

**Surface.** After create OR reuse, the article is at \`/content-magic/{article_id}\` in the UI — used by [SB:cjgeo->surfaceResult] for final CIC review.
`,
    },

    deduplicateArticles: {
      description: "Locate and remove duplicate CJGEO articles by title. Keep the row with the longest content_html; delete the rest. Run when [SB:cjgeo->createArticle] surfaces ≥ 2 matches, or as a periodic housekeeping sweep.",
      howTo: `
**Why this exists.** CJGEO's \`POST /api/articles\` does NOT dedup by title — every call creates a new row. Batch hydration sweeps and SE re-runs accumulate duplicates (observed during the cloudstonedesigns hydration, May 2026). The locked dedup-check in [SB:cjgeo->createArticle] prevents new duplicates; this action cleans up existing ones.

**Procedure:**

1. **List articles + content lengths.** Query [DB:content_magic_articles] via [SB:cjgeo->connectSupabase] (CJGEO Supabase project ref \`<REDACTED-PROJECT-REF>\`):
   \`\`\`sql
   SELECT id, title, status, COALESCE(LENGTH(content_html), 0) AS content_len, updated_at
   FROM content_magic_articles
   WHERE user_id = <operator_user_id>
   ORDER BY LOWER(TRIM(title)), content_len DESC, updated_at DESC;
   \`\`\`
   (Replace \`user_id\` with the operator's actual id if multi-tenant; for single-operator setups omit.)

2. **Group by normalized title.** Compute \`norm = LOWER(TRIM(REGEXP_REPLACE(title, '\\s+', ' ', 'g')))\` per row. Group rows by \`norm\`. Any group with > 1 row is a duplicate set.

3. **Pick the keeper per group (locked rule).** Keep the row with the LARGEST \`content_len\`. Tiebreaker on length tie: newest \`updated_at\`. Surface the chosen keeper id + reasoning per group in the report.

4. **Delete the losers — two paths:**
   - **Path A — Supabase delete (preferred for batch).** From the cjgeo repo with service-role:
     \`\`\`js
     await db.from("content_magic_articles").delete().in("id", losersToDelete);
     \`\`\`
     Verify with a follow-up SELECT that the loser ids are gone. Per [SB:cjgeo->verifyMutation] discipline.
   - **Path B — CJGEO UI delete (for human-eyes-on review).** On \`https://cjgeoai.com/content-magic\`, each article row has a red delete button next to the edit button. Click it per loser (UI confirms with a dialog). Slower; OK for small dedup sets.

5. **Verify.** Re-run the SELECT from step 1. Confirm zero groups now have > 1 row. Log the before/after delta.

**LOCKED — never delete without surfacing first.** Even with the "longest content_html wins" rule, the dedup decision is operator-owned for non-trivial sets. The orchestrator MUST present the proposed keeper + losers to the user (with content lengths and titles) and get explicit \`PROCEED\` before issuing deletes. Auto-deleting silently is a discipline failure even when the rule is correct on paper — content_html length is a proxy for "more work invested", not a guarantee that the longer one is the intended one.

**LOCKED — cascade caveat.** \`content_magic_articles\` has FK references from related tables (drafts, jobs, demo_jobs, etc.). Confirm cascade behavior or null-out related FKs BEFORE deleting — a dangling FK leaves orphan rows that break downstream jobs. Test on one loser first; if delete cascades cleanly, batch the rest.

**Reporting.** End the dedup run with: \`{groups_found, articles_deleted, articles_kept, by_group:[{title, keeper_id, keeper_len, losers:[{id, len}]}]}\`. Attach to the orchestrator's report.
`,
    },

    setMainKeyword: {
      description: "Commit the article's main keyword via POST /api/articles/{id}/keyword action=set_main_keyword. Required before any topic or full-auto downstream workflow.",
      howTo: `
- **Wire.** \`POST /api/articles/{articleId}/keyword\` body \`{action:"set_main_keyword", main_keyword:"<phrase>"}\` → \`200 {committed:true, main_keyword}\`.
- **Why first.** [SB:cjgeo->topicFullAuto] and [SB:cjgeo->keywordFullAuto] both require a committed main_keyword (topic full_auto checks it as a precondition; keyword full_auto verifies it at stage 1). Commit before dispatching either.
- **Verify.** Follow with \`{action:"get_assets"}\` and confirm \`main_keyword\` equals the value posted. Per [SB:cjgeo->verifyMutation].
`,
    },

    keywordFullAuto: {
      description: "Run the 8-stage SEO keyword research chain to completion via POST /api/articles/{id}/keyword with empty body (default action full_auto).",
      howTo: `
- **Role.** Mandatory precursor to [SB:cjgeo->generateDraft]. Must complete before draft kickoff — the precondition gate at [SB:cjgeo->refFullAutoGate] ABORTS \`generate_draft\` if \`keywords[]\` is empty.
- **Wire.** \`POST /api/articles/{articleId}/keyword\` body \`{}\` (omit \`action\` to invoke the default \`full_auto\`) → \`200 {committed:true, summary:{search_count, vertical, horizontal, evaluated, kept}}\`.
- **Chain.** verify main_keyword present → \`search\` → \`research_keyword_vertical\` → \`research_keyword_horizontal\` (first 3 candidates only) → \`evaluate_keyword_relevance\` → commit kept keywords.
- **Preconditions.** Main keyword MUST be committed first via [SB:cjgeo->setMainKeyword] — otherwise stage 1 errors.
- **Verify.** Follow with \`{action:"get_assets"}\` and confirm \`keywords[]\` is populated and \`summary.kept\` matches the asset list length. Per [SB:cjgeo->verifyMutation].

**LOCKED — \`summary.kept = 0\` is a SILENT FAILURE, not success.** The endpoint returns HTTP 200 with \`{committed:true, summary:{kept:0}}\` when DataForSEO returns no data on the seed keyword. Looks identical to success at the wire layer but downstream [SB:cjgeo->refFullAutoGate] aborts on empty \`keywords[]\`. **Treat \`summary.kept === 0\` as a retry-with-different-seed condition, NOT a success.** See [SB:cjgeo->refDataForSEOSeedStrategy] for the head-term fallback pattern.

**\`keywords[]\` payload shape.** Returned objects are \`{keyword, note, include}\` — there is NO numeric \`volume\` or \`difficulty\` field. Selection / prioritization logic must lean on the \`note\` field (LLM-generated intent description), NOT search-volume gates. Older skill book / orchestrator code that filtered by \`kw.volume > N\` is broken against the current payload.
`,
    },

    topicFullAuto: {
      description: "Run the competitor-topic benchmark chain to completion via POST /api/articles/{id}/topic with empty body (default action full_auto).",
      howTo: `
- **Role.** Mandatory precursor to [SB:cjgeo->generateDraft]. Must complete before draft kickoff — the precondition gate at [SB:cjgeo->refFullAutoGate] ABORTS \`generate_draft\` if \`topics[]\` is empty.
- **Wire.** \`POST /api/articles/{articleId}/topic\` body \`{}\` (default \`full_auto\`) → \`200 {committed:true, summary:{...}}\`.
- **Chain.** search competitor URLs (seeded by main_keyword) → \`crawl_url\` per result → \`extract_topics\` → \`evaluate_topics\` → \`save_topics\`.
- **Preconditions.** Main keyword MUST be committed (see [SB:cjgeo->setMainKeyword]) — \`search\`, \`extract_topics\`, and \`full_auto\` all read it.
- **Verify.** \`{action:"get_assets"}\` → confirm \`topics[]\` populated. Per [SB:cjgeo->verifyMutation].

**LOCKED — \`summary.kept = 0\` is a SILENT FAILURE, not success.** Same shape as [SB:cjgeo->keywordFullAuto] — HTTP 200 \`{committed:true, summary:{kept:0}}\` when DataForSEO returns no competitor data on the main_keyword seed. Downstream gate aborts on empty \`topics[]\`. Retry with a different seed per [SB:cjgeo->refDataForSEOSeedStrategy].

**LOCKED — coverage-evaluator skip rule on fresh articles (fixed in cjgeo commit \`f4ad26c\`).** The \`/api/content-magic/topics/evaluate-topic\` LLM prompt asks "is this topic already covered in the article?" — designed for post-draft review. In \`full_auto\` we're HARVESTING topics for a fresh article (article body is ~36 chars of placeholder content), so the evaluator correctly answers "no" for every topic and rejects them all. **Pattern (locked in \`f4ad26c\`): when article body < 200 chars, SKIP the coverage evaluator and accept benchmarked topics directly.** If a topic_full_auto run on a fresh article returns kept=0 despite a head-term seed, suspect a regression here.
`,
    },

    refDataForSEOSeedStrategy: {
      description:
        "Reference (LOCKED): seed-keyword strategy for keywordFullAuto + topicFullAuto when DataForSEO returns empty data (kept=0). Head-term fallback pattern + serial-vs-parallel execution guidance.",
      howTo: `
**Why this exists.** Both [SB:cjgeo->keywordFullAuto] and [SB:cjgeo->topicFullAuto] silently return HTTP 200 with \`summary.kept = 0\` when DataForSEO has no data on the seed keyword. The full_auto chain "succeeded" at the wire level but downstream [SB:cjgeo->refFullAutoGate] aborts \`generate_draft\` on empty \`keywords[]\` / \`topics[]\`. The fix lives in the seed choice + execution cadence, not the API.

### 1. Seed-keyword quality matters more than article topic

**Reliable seeds (DataForSEO returns data):** short head-terms — 1–3 words, common search vocabulary. Examples that worked in batch regenerations:
- "retirement paycheck"
- "annuity fees"
- "annuity comparison"
- "safe money bucket"

**Unreliable seeds (DataForSEO returns kept=0):** long-tail conversational lay-phrases — 5+ words, full-question shape. Examples that returned empty data:
- "should I move money out of the market before retirement"
- "how much monthly income do I need in retirement"
- "what happens to my 401k if I die before retiring"

Lay-phrase seeds match the LLM-aligned section H2s on the consuming site (e.g. bryanblog satellites per [SB:BryanBlog->refStrategy] rule 2), but DataForSEO is keyword-volume-indexed and lacks data on the conversational tail. The article's H1 / page title is rarely a good seed.

### 2. Head-term fallback pattern (LOCKED)

When [SB:cjgeo->keywordFullAuto] or [SB:cjgeo->topicFullAuto] returns \`summary.kept === 0\`:

1. **Do NOT treat as success.** Wire-200 + kept=0 = retry condition.
2. **Re-seed with a shorter head-term.** Drop adjectives, drop question framing, keep the noun phrase. "should I move money out of the market before retirement" → "retirement market timing" or "retirement risk" (try the shortest first).
3. **Re-commit main_keyword** via [SB:cjgeo->setMainKeyword] with the new head-term seed.
4. **Re-run \`full_auto\`.** Verify \`summary.kept > 0\` on retry.
5. **3-strike rule.** If three head-term re-seeds all return kept=0, STOP and surface the article to the user as State 1 — DataForSEO genuinely has no data for this topic neighborhood; the article is probably mis-scoped for SEO-driven generation.

Batch evidence (May 2026 hydration sweep): 13 of 14 articles that hit kept=0 on a lay-phrase seed unlocked on first head-term re-seed. The 14th unlocked on the second re-seed.

### 3. Serial vs parallel execution

**Pattern: SERIAL execution for batch regenerations.** Parallel batches (5+ concurrent full_auto calls) hit empty-data on 14 of 17 articles in the May 2026 sweep. Serial execution + head-term fallbacks unlocked 13 of 14 on first attempt. Whether the underlying cause is rate-limiting by DataForSEO, coincidental sparsity, or a CJGEO-side scheduler issue is undetermined — but **the mitigation pattern is locked: run batch regenerations serially, not in parallel.**

Parallel is fine for one-off article runs (no neighbor articles to contend with). Serial is the rule when regenerating > 3 articles in a session.
`.trim(),
    },

    refSubstrateFixesTimeline: {
      description:
        "Reference: substrate fixes landed during the May 2026 hydration sweep. Localhost-loopback bug class on Vercel serverless, the 3 fixed routes, the 1 still-open route.",
      howTo: `
**Bug class: \`internalPost\` to \`localhost\` on Vercel serverless.** Several per-step API routes used an \`internalPost(url, body)\` helper that pointed at \`http://localhost:3000/...\` for cross-route hops. On Vercel each serverless function runs in its own sandbox — there is no in-process loopback. The helper returned HTTP 500 "fetch failed" in production while running fine locally. Manifests as \`full_auto\` chains aborting silently mid-pipeline with no useful error.

**Fix shape.** A \`getOrigin()\` helper that prefers env-resolvable hosts:
1. \`process.env.SITE_URL\` (explicit prod override)
2. \`process.env.NEXT_PUBLIC_SITE_URL\` (Vercel build env)
3. \`https://\${process.env.VERCEL_URL}\` (Vercel deployment URL)
4. \`http://localhost:3000\` ONLY when \`process.env.NODE_ENV === "development"\`

Replace every \`fetch("http://localhost:3000/api/...")\` with \`fetch(\`\${getOrigin()}/api/...\`)\`.

### Fixed routes (May 2026)

| Route | Fix commit | What broke |
|---|---|---|
| \`app/api/articles/[articleId]/keyword/route.ts\` | \`f41bf6a\` | [SB:cjgeo->keywordFullAuto] returned 500 on Vercel; worked locally. |
| \`app/api/articles/[articleId]/topic/route.ts\` | \`7f2a1b1\` | [SB:cjgeo->topicFullAuto] same class bug. |
| \`/api/content-magic/topics/evaluate-topic\` (LLM prompt) | \`f4ad26c\` | Not the loopback bug — topic-coverage evaluator rejecting all topics on fresh (<200-char) articles. Fix: skip the evaluator when article body < 200 chars. See [SB:cjgeo->topicFullAuto] LOCKED note. |

### Still open (NOT fixed)

- **\`app/api/articles/[articleId]/implementation/route.ts\`** — same \`internalPost\` localhost loopback bug class. Affects [SB:cjgeo->refApiActions §6] (\`implementation/full_auto\` + \`implement_all_keywords\`). Not yet patched as of May 2026. **If implementation full_auto returns 500 on prod, this is why** — apply the same \`getOrigin()\` fix.

### Why this matters for orchestrators

\`full_auto\` chains observe these failures as silent kept=0 / empty-asset returns when the inner cross-route hop dies. The orchestrator's verify-after-mutate gate ([SB:cjgeo->verifyMutation]) catches it; bare wire-success checks do not. If a previously working pipeline starts returning kept=0 on head-term seeds that previously worked, check the substrate first before re-seeding — a regression in the loopback bug class on a new route is the most likely culprit.
`.trim(),
    },

    refFullAgenticMode: {
      description:
        "Reference: full_agentic_mode column + cron state machine on content_magic_articles. Cron-driven end-to-end pipeline for plan-gated tenants. New feature surface (May 2026).",
      howTo: `
**Status.** Shipped May 2026 across cjgeo commits \`3b28ff6\` / \`f395b94\` / \`d309df6\`. Adds a cron-driven full-pipeline mode that walks an article from \`keyword → topic → prompt → edit-draft → done\` without orchestrator intervention. Plan-gated.

### State machine

\`content_magic_articles.full_agentic_mode\` is a string column:
- \`'false'\` (or NULL) — agentic mode OFF. Cron ignores the row.
- \`'keyword'\` — cron's next tick will run [SB:cjgeo->keywordFullAuto].
- \`'topic'\` — next tick runs [SB:cjgeo->topicFullAuto].
- \`'prompt'\` — next tick runs [SB:cjgeo->promptFullAuto].
- \`'edit-draft'\` — next tick kicks off [SB:cjgeo->generateDraft], then polls + adopts.
- \`'done'\` — terminal. Cron skips.

### Cron entry

Cron picks up rows WHERE \`full_agentic_mode != 'false' AND full_agentic_mode != 'done'\` and advances the state machine ONE step per tick, updating the column to the next stage on success. See [SB:cjgeo->inspectCron] for the static-audit procedure.

### Plan gate

Cron only processes rows where \`profiles.subscription_plan IN ('pro', 'test')\`. Free / starter tenants do NOT get agentic mode regardless of the column value — the cron filter drops them.

### UI surface

Inline "+ New article ▾" popover on \`/content-magic\` (\`d309df6\`) with an "Agentic mode" toggle. Setting the toggle stamps \`full_agentic_mode = 'keyword'\` on article creation; cron picks it up on the next tick.

### Operational quirk — Vercel 60s outer-timeout on cron

The cron handler at \`/api/cron/trigger\` is bound by Vercel's 60s function timeout. Each per-step route (keyword, topic, prompt, edit-draft) commits its state advance BEFORE responding, so the state machine progresses correctly. But if a step takes > 60s end-to-end (the topic step is the usual offender — competitor crawl is slow), cron's OUTER GET response is lost to a 504 even though the inner per-step POST already committed. **Symptom:** Vercel logs show 504 on \`/api/cron/trigger\`, but \`content_magic_articles.full_agentic_mode\` still advances. NOT a correctness bug — the state is consistent. Optional improvement: detach per-step POSTs via Next 15's \`after()\` so the cron GET returns 200 immediately while the work continues in the background.

### Orchestrator vs agentic mode

A chaperone-driven orchestrator (e.g. [SB:cursorCJGEO->replicateArticle]) and agentic mode are MUTUALLY EXCLUSIVE on the same article — both would race the state machine. When dispatching from a chaperone, ensure \`full_agentic_mode\` is \`'false'\` on the article (or just don't set it). When using agentic mode, do NOT also dispatch a chaperone for the same article.
`.trim(),
    },

    promptFullAuto: {
      description: "Run the GEO prompt suggestion + commit chain via POST /api/articles/{id}/prompt with empty body (default action full_auto).",
      howTo: `
- **Wire.** \`POST /api/articles/{articleId}/prompt\` body \`{}\` (default \`full_auto\`) → \`200 {committed:true, count}\`.
- **Chain.** \`suggest_prompts\` (bottom-funnel default) → \`save_prompts\`.
- **Scope.** Prompt research and internal-link research are explicitly OUT OF SCOPE for this endpoint.
- **Verify.** \`{action:"get_assets"}\` → confirm \`prompts[]\` length equals returned \`count\`. Per [SB:cjgeo->verifyMutation].
`,
    },

    generateDraft: {
      description: "Kick off async draft generation via POST /api/articles/{id}/edit-draft action=generate_draft. Returns 202; orchestrator owns polling.",
      howTo: `
**Preconditions (LOCKED — do not skip):**
1. [SB:cjgeo->keywordFullAuto] must have completed successfully — the article's \`assets.main_keyword\` is set AND \`assets.keywords[]\` is populated (length > 0).
2. [SB:cjgeo->topicFullAuto] must have completed successfully — the article's \`assets.topics[]\` is populated (length > 0).
3. Both verifications happen via \`POST /api/articles/{id} {action:"get_assets"}\` before draft kickoff (see [SB:cjgeo->refFullAutoGate] for the reusable check).

If either precondition is unmet, ABORT. Do not call \`generate_draft\`. Run the missing step first. ([SB:cjgeo->promptFullAuto] is recommended but not LOCKED — its absence is a "soft" warning, not an abort.)

---

- **Wire.** \`POST /api/articles/{articleId}/edit-draft\` body \`{action:"generate_draft", ...settings}\` → \`202 {article_id, status:"generating", poll, chat_id}\`. \`poll\` is the recommended next call (always [SB:cjgeo->pollDraft]).
- **Settings map (verbatim from spec).**
  - \`mode\`: \`"new_draft"\` | \`"improve_existing_draft"\`
  - \`provide_feedback\`: boolean
  - \`competitor_pages\`: \`[url, ...]\`
  - \`competitor_page_titles\`: \`[title, ...]\`
  - \`context_prompt\`: string
  - \`use_default_context_prompt\`: boolean
  - \`generation_prompt_mode\`: \`"default"\` | \`"custom"\`
  - \`custom_generation_prompt\`: string
  - \`use_custom_template\`: boolean — **LOCKED \`true\` per operator 2026-05-21** (see below).
  - \`page_layout_example_url\`: string
  - \`allow_image_generation\`: boolean — **default \`true\`** per the SE script that drives this skill book.
- **Outline only.** \`generate_draft\` writes to the outline/draft buffer; it does NOT write \`content_html\`. Adoption is a separate commit ([SB:cjgeo->adoptDraft]).
- **Next step.** Hand off to [SB:cjgeo->pollDraft] immediately on 202.

---

### Body-payload example (LOCKED shape — 2026-05-21)

\`\`\`json
{
  "action": "generate_draft",
  "mode": "new_draft",
  "use_default_context_prompt": true,
  "generation_prompt_mode": "auto",
  "use_custom_template": true,
  "allow_image_generation": true
}
\`\`\`

Always pass \`use_custom_template: true\` per operator lock 2026-05-21 — the toggle activates the custom-templates assembly path. (Internal CJGEO routing: \`cjgeo/app/api/articles/[articleId]/edit-draft/route.ts:367\` reads \`body.use_custom_template === true\`.) The operator's screenshot of the CJGEO Content Magic UI showed this toggle OFF on a stale-publish diagnostic; the lock flips it ON for every adopter call.

---

### Default output styling — preserve, do not strip

**CJGEO's default \`generateDraft\` output bundles inline styles + bespoke component classes (\`<style data-draft-styles>\`, \`article-section\`, \`callout\`, \`qr-item\`, \`flow-step\`, \`badge-*\`, \`figure-body\`, \`check-icon\`, \`card\`, \`card-header-*\`, \`qr-grid\`, \`flow-grid\`, \`checklist\`, \`table-wrap\`). Destination adopters must preserve these inline — stripping them strips CJGEO's entire design treatment. See [SB:bosterbio->adoptCJGEOIntoMagentoPage] for the locked preserve-list.**
`,
    },

    pollDraft: {
      description: "Poll draft status every 5 minutes via POST /api/articles/{id}/edit-draft action=pull_status. 20-minute timeout triggers reinitiate. Orchestration owns cadence and timeout — API only owns the wire.",
      howTo: `
This action's contract is OWNED BY THIS SKILL BOOK. The API exposes \`pull_status\` as a stateless read; cadence, timeout, and the re-initiate decision are skill-book policy.

- **Wire.** \`POST /api/articles/{articleId}/edit-draft\` body \`{action:"pull_status"}\` → \`200 {status:"ready"|"not_ready"}\`.
- **Cadence.** Poll **every 5 minutes**. Do NOT poll more aggressively — generation is multi-minute by design and faster polling adds load without benefit.
- **Branch.**
  - \`status === "ready"\` → proceed to [SB:cjgeo->adoptDraft].
  - \`status === "not_ready"\` AND elapsed < 20 min → keep polling.
  - \`status === "not_ready"\` AND elapsed ≥ 20 min → invoke [SB:cjgeo->reinitiateDraft], reset the elapsed clock, and resume polling.
- **No premature adopt.** Never call \`adopt_draft\` while \`pull_status\` reports \`not_ready\` — \`adopt_draft\` returns \`{status:"not_ready"}\` (HTTP 200, not 409) and the orchestrator MUST treat that as a no-op, not as success.
`,
    },

    reinitiateDraft: {
      description: "Re-trigger generation when pollDraft hits the 20-minute timeout. Re-calls generate_draft on the same article. Skill book owns when; API only owns that re-calling re-triggers.",
      howTo: `
- **Wire.** Re-call \`POST /api/articles/{articleId}/edit-draft\` with \`{action:"generate_draft", ...settings}\`:
  - If no draft has been adopted yet: use the SAME settings as the original [SB:cjgeo->generateDraft] call.
  - If a draft exists and the orchestrator wants to push it further: \`{action:"generate_draft", mode:"improve_existing_draft", provide_feedback:true, ...}\`.
- **Skill-book ownership.** *When* to re-initiate (the 20-min timeout per [SB:cjgeo->pollDraft]) is policy here, not in the API. *That* re-calling re-triggers is wire behavior.
- **After re-initiate.** Continue polling via [SB:cjgeo->pollDraft]; do not block further work on the orchestrator side.
`,
    },

    adoptDraft: {
      description: "Adopt the generated draft into content_html via POST /api/articles/{id}/edit-draft action=adopt_draft. Idempotent no-op when not yet ready (returns 200 not_ready, NOT 409).",
      howTo: `
- **Wire.** \`POST /api/articles/{articleId}/edit-draft\` body \`{action:"adopt_draft"}\`:
  - **Ready:** \`200 {committed:true, status:"adopted", length}\` — writes \`content_html\` on [DB:content_magic_articles].
  - **Not ready:** \`200 {status:"not_ready"}\` (NOT 409, per spec). Treat as no-op; resume polling via [SB:cjgeo->pollDraft].
- **Precondition.** Only call after [SB:cjgeo->pollDraft] returns \`ready\`. Belt-and-braces: this action is safe to call early (returns not_ready, no side effect), but doing so wastes a request.
- **Verify.** Follow with \`{action:"get_html"}\` and confirm \`content_html.length\` matches the returned \`length\` (or is non-trivial). Per [SB:cjgeo->verifyMutation].
`,
    },

    refImagePlaceholderHandling: {
      description:
        "Reference (LOCKED): when adopting a CJGEO article into a destination repo, detect <img> placeholders, fill them with gpt-image-2 (per [SB:cjgeo->generateImage]) via the openai_images weapon, store them so the destination serves them, and rewrite \`src\` so every image renders on the live page without 404 / broken-image errors.",
      howTo: `
**Why this exists.** CJGEO drafts ship with \`<img>\` tags whose \`src\` often does NOT resolve from the destination host: CJGEO-internal asset paths, third-party placeholder services, broken external URLs, \`data:\` stubs, or empty \`src\` carrying an authoring hint in \`alt\`. Shipping them unchanged to a destination repo (bryanblog, etc.) breaks the page visually (broken-image icon) and fails the destination's CIC visual-verify gate per the alignment theorem. This step is non-optional on every adoption.

**Where this runs in the pipeline.** Insert as a NEW **Step 2.5** in [SB:BryanBlog->adoptCJGEORawContent], between Step 2 (sanitize) and Step 3 (inject \`id\` on h2s). The composite pipeline becomes: get_html → sanitize → **fillImagePlaceholders (this step)** → anchor ids → deep-dive links → write to \`content/\` → update \`lib/articles.ts\` → rewrite \`page.tsx\` → build → push → CIC visual-verify. Adopters that skip this step ship pages with broken-image icons.

---

### 1. Detect placeholders (LOCKED rule)

After [SB:BryanBlog->adoptCJGEORawContent] Step 2, walk every \`<img>\` in the sanitized HTML. An image is a placeholder if ANY of these hold:

- \`src\` is missing / empty / \`"#"\` / \`"about:blank"\`.
- \`src\` starts with \`data:\` (EXCEPT a small inlined icon under ~4 KB — those pass through).
- \`src\` is a hostname the destination cannot reach (e.g. cjgeoai.com internal paths, \`localhost\`, generic placeholder services like \`via.placeholder.com\`, \`placehold.it\`, \`picsum.photos\`).
- \`src\` returns non-200 on a HEAD fetch from the adopter machine (5s timeout; treat timeout/error/non-2xx as placeholder).
- \`alt\` text carries an explicit generation hint: starts with \`generate:\`, \`AI:\`, or wrapped as \`[image: ...]\` — explicit author intent for AI fill.

If NONE of the above hold AND \`src\` HEADs 200 AND the image is from a host the destination's CSP allows, the image is a real asset — DO NOT regenerate (cheaper + on-brand to keep). Regeneration only fires when at least one condition triggers.

### 2. Generate via gpt-image-2

For each detected placeholder:

1. **Compose the prompt.** Build a one-sentence subject prompt from the surrounding context:
   - Nearest preceding \`<h2>\` text (article section topic).
   - The placeholder's existing \`alt\` (or the body of \`alt\` if it starts with a generation hint).
   - Optional destination-side style suffix from the destination's design guide (e.g. for bryanblog: \`"warm editorial illustration, muted retirement-planning palette, no people faces"\` — sourced from [SB:BryanBlog->refDesign]; callers from other destinations supply their own style suffix).
   The prompt describes the SUBJECT; STYLE comes from the destination's design guide. Do NOT bake the style into the article HTML.
2. **Call the openai_images weapon** per [SB:cjgeo->generateImage]:
   \`\`\`js
   import { generate } from "@/libs/weapon/openai_images";
   const [b64] = await generate({ prompt, size: "1536x1024", quality: "medium" }, userId);
   const buf = Buffer.from(b64, "base64");
   \`\`\`
   - **Default size** \`1536x1024\` (landscape) for in-body article images; \`1024x1024\` for square; \`1024x1536\` for portrait. Pick by the placeholder's aspect intent (\`width\`/\`height\` attrs if present, else default landscape).
   - **Default quality** \`medium\` ($0.042/img). \`high\` ($0.167/img) ONLY if the caller asked. \`low\` ($0.011/img) for thumbnail/icon use.
   - **Model is \`gpt-image-2\`** — do NOT pass \`response_format\` (unsupported; b64_json is always returned). Do NOT pass \`style\` (DALL-E only). Per [SB:cjgeo->generateImage].
3. **Pick a stable filename:** \`<article-slug>-<n>.png\` where \`n\` is the placeholder's 1-based index within the sanitized article. Deterministic so re-runs overwrite cleanly and the cache stays useful.

### 3. Store — two paths, pick by destination

**Path A — destination repo's public assets (PREFERRED for static sites like bryanblog).** Write the PNG to \`public/images/<article-slug>/<filename>\` in the destination repo. Set \`src="/images/<article-slug>/<filename>"\` on the \`<img>\`. The image ships with the build and is served by the same CDN (Vercel/Netlify/etc.) as the page — same origin, no CORS, no cache-control surprises, no extra infra. Commit the PNG alongside the HTML and \`page.tsx\` change so the deploy is atomic (one merge = page + images live together).

**Path B — Supabase Storage public bucket** (preferred when the destination cannot accept binary commits, OR when many articles share an image pool). Upload via the \`supabase_storage\` weapon to a public-read bucket (e.g. \`cjgeo-public-images\`); read the public URL; set \`src\` to that URL.
\`\`\`js
import { writeFile, readPublicUrl } from "@/libs/weapon/supabase_storage";
await writeFile({
  path: \`<article-slug>/<filename>\`,
  content: buf,
  contentType: "image/png",
  bucket: "cjgeo-public-images",
});
const url = readPublicUrl({
  path: \`<article-slug>/<filename>\`,
  bucket: "cjgeo-public-images",
});
\`\`\`
Bucket MUST be configured public-read (verify with a curl HEAD before declaring success). The destination's CSP must allow the Supabase storage host (\`*.supabase.co\`); confirm this BEFORE switching to Path B — a same-origin Path A almost always avoids the CSP fight.

### 4. Rewrite the \`<img>\` tag

After generation + storage, for each placeholder \`<img>\`:

- **\`src\`** → the new path (Path A) or public URL (Path B).
- **\`alt\`** → a human-readable description (the subject prompt without the style suffix). \`alt=""\` is acceptable ONLY for explicitly decorative images; empty alt on a content image silently kills accessibility and is a discipline failure.
- **\`loading="lazy"\`** on every in-body image (decorative or content) — prevents render-blocking and is a free perf win.
- **Explicit \`width\` and \`height\` attributes** matching the generated size (e.g. \`width="1536" height="1024"\`) — prevents CLS (layout shift) and is required by some compliance/lighthouse tools.

### 5. Verify they render — alignment-theorem gate

Per the alignment theorem (per [SB:cjgeo->verifyMutation] + the user-channel verification rule), "the image was generated and uploaded" is NOT the same as "the image renders on the live page". Both checks are required:

1. **Wire check** (necessary, not sufficient). HEAD-fetch each new image URL from the adopter machine:
   - Status 200.
   - \`Content-Type\` matches what was uploaded (e.g. \`image/png\`).
   - \`Content-Length > 0\`.
2. **User-channel visual check** (the actual gate). Open the LIVE destination URL in the user's connected Chrome via CIC. Look at the rendered page:
   - Zero broken-image icons in the article body.
   - Every \`<img>\` in the article body has \`naturalWidth > 0\` (quick smoke via \`Array.from(document.querySelectorAll('article img')).every(i => i.naturalWidth > 0)\`, then visually confirm).
   - DevTools network tab (in the user's CIC session, NOT an agent-side proxy): no 404 / CORS / mixed-content errors on image requests.

A broken-image icon on the live page is an automatic FAIL of [SB:BryanBlog->adoptCJGEORawContent] Step 10's CIC visual verify, no matter how green the local build was. Per the alignment theorem an agent-side preview / headless render / curl that succeeds where the user's connected Chrome shows a broken image is a channel-disagreement to FIX, never noise to route around.

### Anti-patterns

- DO NOT regenerate when the placeholder \`src\` actually resolves. Detect first; regenerate only on a detector hit.
- DO NOT inline generated images as \`data:\` URIs in the HTML — bloats the HTML, hits CSP limits, breaks caching, kills the same-origin cache strategy. Always store as separate files.
- DO NOT commit \`OPENAI_API_KEY\` — it resolves from \`process.env\` or \`profiles.env_vars\` per the \`openai_images\` weapon. Never hardcode; never echo into chaperone-side logs.
- DO NOT use older image models (\`dall-e-3\` / \`dall-e-2\`) — \`gpt-image-2\` is current per [SB:cjgeo->generateImage]; older models have worse text-in-image rendering, inconsistent style, and different size/param semantics.
- DO NOT skip this step "because the article has only one image and it looks like an external URL". Run the detector — placeholder-service URLs (\`via.placeholder.com\` etc.) HEAD 200 but still ship a generic placeholder to users.

### When called from a Cursor cloud agent

When a Cursor cloud agent runs this step (per [SB:cursorCJGEO->replicateArticle] / [SB:BryanBlog->adoptCJGEORawContent]), the same rules apply. \`OPENAI_API_KEY\` is supplied via the same Cursor followup channel as \`CJGEO_API_KEY\` (per [SB:cursorCJGEO->refCredentialHandoff]) — one followup, separate \`KEY=value\` lines. The cursor agent uses Path A by default (commit PNGs to bryanblog's \`public/images/<slug>/\` alongside the HTML and \`page.tsx\` so the Vercel deploy is atomic). The reporting block per [SB:cursorCJGEO->refReportingContract] lists the generated image paths under \`ARTIFACTS\` so the chaperone can verify them post-merge.
`.trim(),
    },

    refFullAutoGate: {
      description: "Reusable precondition gate: verify keywordFullAuto + topicFullAuto have committed before calling generateDraft. ABORT on missing main_keyword / empty keywords[] / empty topics[].",
      howTo: `
**Locked precondition gate for [SB:cjgeo->generateDraft].** Every orchestrator that kicks off draft generation MUST run this check first — recent runs (the bryanblog satellite batch + the Cursor chaperone run) skipped SEO + GEO research and silently lost the value-add of the full_auto chain.

**Check shape (reusable subroutine):**
\`\`\`js
// Verify full_auto preconditions before generate_draft
const assets = await postArticleAction(articleId, "get_assets");
if (!assets.main_keyword) throw new Error("main_keyword missing — run keywordFullAuto first");
if (!assets.keywords?.length) throw new Error("keywords[] empty — run keywordFullAuto first");
if (!assets.topics?.length) throw new Error("topics[] empty — run topicFullAuto first");
\`\`\`

- **Hard gate.** \`main_keyword\` + non-empty \`keywords[]\` + non-empty \`topics[]\` are all required. Missing any one = ABORT (do NOT call \`generate_draft\`); run the missing precursor and re-gate.
- **Soft warning only.** \`prompts[]\` empty is a soft warning per [SB:cjgeo->promptFullAuto] — log it but proceed.
- **Wire.** The single read is \`POST /api/articles/{articleId}\` body \`{action:"get_assets"}\` → \`{main_keyword, keywords[], topics[], prompts[], ...}\` per [SB:cjgeo->refApiActions §1].
- **Where to call.** Immediately before the \`POST /api/articles/{articleId}/edit-draft {action:"generate_draft", ...}\` call in [SB:cjgeo->generateDraft]. SE scripts, chaperone runs, and freestyle orchestrators all share this gate.
`,
    },

    verifyMutation: {
      description: "Discipline: every commit-yes action MUST be verified by re-reading via get_assets (asset mutations) or get_html (content_html mutations). Pre/post diff is recorded in the agent log.",
      howTo: `
**Non-optional gate.** A commit response of \`{committed:true}\` is necessary but not sufficient. The orchestrator MUST re-read the changed surface and confirm the mutation landed.

- **Asset mutations** ([SB:cjgeo->setMainKeyword], [SB:cjgeo->keywordFullAuto], [SB:cjgeo->topicFullAuto], [SB:cjgeo->promptFullAuto], implementation commits) → re-read via \`POST /api/articles/{id}\` \`{action:"get_assets"}\`. Compare pre/post counts and field values.
- **content_html mutations** ([SB:cjgeo->adoptDraft], \`edit_html\`, \`import\`) → re-read via \`POST /api/articles/{id}\` \`{action:"get_html"}\`. Compare pre/post length (and a sample of the body).
- **Log the diff.** Record \`{action, pre, post, delta}\` to the agent's run log so the supervisor / SE judge can audit.
- **Failure semantics.** Missing post-mutation verification counts as a TEST FAILURE for SE runs — equivalent severity to a wire failure.
`,
    },

    observability: {
      description: "Discipline (testing-run only): API responses must surface enriched debug payload (action name, called backend function, state diff, mutation timestamp, correlation ID, sub-call latencies). Missing fields = test failure.",
      howTo: `
**Testing enrichment, not prod contract.** Production responses can be slim (\`{committed:true}\` is fine in prod). During an SE / testing run, the agent MUST gather richer debug data.

- **Required fields per call (testing).**
  - \`action\` — the dispatched action name (echoed back).
  - \`backend_fn\` — the underlying function the route reused from the UI surface.
  - \`state_diff\` — pre/post values of every field the action wrote.
  - \`mutation_ts\` — server-side commit timestamp.
  - \`correlation_id\` — request-scoped id, threaded through any sub-calls.
  - \`sub_call_latencies\` — for chained actions (\`full_auto\`), per-stage timings.
- **How to obtain.** Prefer requesting verbose response shape via a header / setting if the API exposes it. If not, the orchestrator MUST independently compute the diff by reading [SB:cjgeo->verifyMutation] before and after each commit, and record latencies from the orchestrator clock.
- **Failure semantics.** Missing observability fields during a TEST RUN count as a test failure — equivalent severity to missing verification.
`,
    },

    inspectCron: {
      description: "Static inspection: read the CJGEO cron implementation files and verify code handles pending generations, stalled generations, retry logic, timeout windows, and failure resolution.",
      howTo: `
Static-code audit of CJGEO's draft-state cron substrate.

- **Files to read.**
  - \`app/api/cron/trigger/route.js\` — top-level cron entry.
  - \`app/api/internal/article-jobs/*\` — per-job handlers (any route in that directory tree).
  - \`libs/api/articleJobsProcessor.ts\` — processor library; the core state machine.
- **What to verify in the code.**
  - Handles \`status:"generating"\` rows that are still in flight (pending generations).
  - Detects stalled generations (e.g. \`generating\` past a wall-clock threshold) and recovers them.
  - Has explicit retry logic with bounded attempts (mirrors [SB:cjgeo->pollDraft]'s 20-min reinitiate at the substrate layer).
  - Has timeout windows defined as named constants (not magic numbers scattered in branches).
  - Resolves failure terminally (marks \`failed\` with \`error_message\`, doesn't leave rows in an indeterminate state).
- **Output.** A structured static-audit verdict per check above (pass/fail + cited line ranges). Feed into [SB:cjgeo->verifyCron] alongside [SB:cjgeo->inspectRuntime].
`,
    },

    inspectRuntime: {
      description: "Dynamic inspection: fetch runtime cron logs / observable evidence (cloud browser, server logs, or /tmp/cjgeo3000.log for dev) and confirm cron actively manages draft states.",
      howTo: `
Live-evidence audit complementing [SB:cjgeo->inspectCron].

- **Sources.**
  - Cloud browser / hosted server logs for the prod cron handler invocation history.
  - For the dev path: \`/tmp/cjgeo3000.log\` (the conventional dev-server log path for CJGEO).
  - The CJGEO UI's job-status surfaces where exposed.
- **What to confirm.**
  - Cron is invoked on schedule (recent entries within expected interval).
  - Each invocation processes pending jobs (visible state transitions: \`generating → ready → adopted\` or \`generating → failed\`).
  - Stalled rows ARE being recovered (no rows stuck in \`generating\` past the timeout in live data).
  - Failure terminations are reached (no infinite-retry loops).
- **Output.** Structured runtime-audit verdict; combined with [SB:cjgeo->inspectCron] in [SB:cjgeo->verifyCron].
`,
    },

    verifyCron: {
      description: "Combine inspectCron (static) + inspectRuntime (dynamic) into a single verdict: cron behavior matches expectations.",
      howTo: `
- **Inputs.** Output of [SB:cjgeo->inspectCron] (static audit) AND [SB:cjgeo->inspectRuntime] (runtime audit).
- **Verdict shape.** \`{static_pass:bool, runtime_pass:bool, mismatches:[...], verdict:"pass"|"fail"}\`.
- **Pass rule.** Both static and runtime audits must pass independently. A static-pass with runtime-fail (or vice versa) is a FAIL — the substrate is incoherent and must be flagged.
- **Surface.** Attach the verdict to the agent's run log; feed into the final [SB:cjgeo->reviewSummary].
`,
    },

    surfaceResult: {
      description: "End of run: open the final article in CIC at /content-magic/{article_id} and surface the full asset state for user review.",
      howTo: `
- **CIC navigate.** Open the user's CIC tab to \`{BASE_URL}/content-magic/{article_id}\` so the user lands on the finished article on return.
- **Surface payload.** In the user-facing report, summarize:
  - Article \`id\`, title, length of \`content_html\`.
  - Final asset state: \`main_keyword\`, \`keywords[]\` count, \`topics[]\` count, \`prompts[]\` count.
  - Draft adoption status (\`adopted\` with timestamp).
  - Any \`implement_all_keywords\` / implementation full_auto results.
- **Respect locked CIC caveat.** See [SB:cjgeo->driveContentMagicUI] — CIC's renderer freezes on the adopted-draft CONTENT surface. Drive shell navigation, but do NOT attempt to screenshot the rendered draft body via CIC. Use [SB:cjgeo->connectSupabase] or the API \`get_html\` action if a content snapshot is needed for the report.
- **Per the Sage success-claim gate** (cross-role rule in CLAUDE.md): only surface this as DONE after [SB:cjgeo->verifyMutation] passed on every commit-yes action. Otherwise frame as State 1.
`,
    },

    reviewSummary: {
      description: "Closing report shape for an end-to-end CJGEO run: what succeeded, what retried, what mutated, any anomalies.",
      howTo: `
Final report the orchestrator emits at the end of a complete article pipeline run.

- **Successes** — bulleted list of each commit-yes action that landed and was verified ([SB:cjgeo->setMainKeyword], [SB:cjgeo->keywordFullAuto], etc.), with the verification evidence cited.
- **Retries** — any [SB:cjgeo->reinitiateDraft] invocations, with elapsed-time markers and reason.
- **Mutations** — every \`{action, pre, post, delta}\` diff captured per [SB:cjgeo->verifyMutation]. The full ledger, not a sample.
- **Anomalies** — anything observed that did NOT match the locked spec: unexpected status codes, missing observability fields, cron substrate verdicts ([SB:cjgeo->verifyCron]) that failed, any 501s encountered (e.g. \`evaluate_prompt_implementation\`).
- **Final surface** — link/URL where the user reviews the article ([SB:cjgeo->surfaceResult] artifact).
- **Honest scope.** If any step bypassed verification or fell back to a workaround, frame the report as State 1 (per CLAUDE.md end-state contract) — do NOT claim DONE.
`,
    },

    demoMode: {
      description: "Understand and debug the CJGEO Demo Mode pipeline (tick.js state machine, screenshot storage, error patterns).",
      howTo: `
**State machine:** \`libs/demo/tick.js\`
Status flow: \`pending → discovering → creating → screenshotting → emailing → done\`

**Key files:**
- \`libs/demo/tick.js\` — state machine, \`runDemoTick\`, \`runAllDemoTicks\`
- \`libs/demo/screenshot.js\` — \`screenshotUrl\` (Playwright server-side), \`screenshotArticle\` (Thumio via preview route), \`thumioUrl\`
- \`libs/demo/discovery.js\` — three discovery strategies
- \`libs/demo/email.js\` — pitch email sender
- \`app/api/demo/preview/[articleId]/route.js\` — public HTML preview route

**Screenshot storage:**
- Original page: Playwright + \`@sparticuz/chromium\` → JPEG uploaded to \`demo-screenshots\` bucket at \`originals/{jobId}/original_page.jpeg\`. Falls back to Thumio URL if Playwright fails.
- Draft (article): Thumio URL pointing at \`/api/demo/preview/{articleId}\` (NOT direct Supabase Storage URL — Supabase serves HTML as text/plain).
- Both URLs stored on \`demo_jobs\`: \`original_screenshot_url\` and \`draft_screenshot_url\`.

**Retry pattern:**
- \`[retry N]\` prefix encoded in \`error_message\`. \`retryCount(job)\` parses it.
- \`MAX_GENERATION_RETRIES = 1\` (2 total attempts).
- On retry: \`article_id\` cleared so next tick creates fresh article; simpler keyword used (drop example page + design reference screenshot).
- On final failure: \`status = 'failed'\`, \`error_message\` = full message.

**CRITICAL: clear error_message when advancing past creating:**
\`await setStatus(supabase, job.id, "screenshotting", { error_message: null })\`
Without this, stale \`[retry N] attempt N failed\` text shows in the UI even when the job succeeds.

**Phase checklist UI (\`app/(private)/(shell)/demo-mode/[jobId]/page.js\`):**
- \`phaseStatus(done, error, currentPhase, matchPhases, jobDone)\` — when \`jobDone === true\`, ALL phases return "done" regardless of intermediate errors. This prevents stale retry-failure error rows showing on a completed job.
- errorMsg props are suppressed (\`jobDone ? null : s.phase3Error\`) for the same reason.

**Debugging a stuck/failed job:**
1. Check \`demo_jobs.status\`, \`error_message\`, \`article_id\`.
2. If \`status = 'creating'\` with \`article_id\` set: check \`content_magic_articles.outline.status\` — should be one of \`queued/sending/rendering/completed/adopted/failed\`.
3. If \`error_message\` has stale text but \`status = 'done'\`: clear \`error_message\` directly: \`UPDATE demo_jobs SET error_message = NULL WHERE id = '...';\`
4. Trigger a tick manually via \`POST /api/demo/tick\` (requires \`x-cron-secret\` header).
`,
    },
  },
};
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/cjgeo/engineering_decisions.md -->
```markdown
# Engineering Decisions — CJGEO Weapon

## 2026-06-16: v1 API paths, not legacy /api/articles

**Decision:** The weapon uses `/api/v1/articles/...` paths exclusively, NOT the `/api/articles/...` paths documented in `libs/skill_book/cjgeo/index.js refApiActions`.

**Rejected alternative:** Using the `/api/articles/{articleId}` action-based endpoints (get_assets, get_html, set_main_keyword via action field).

**Why:** The `/api/articles/...` routes are LEGACY internal routes without x-api-key auth support on production (they return 404 for the collection endpoint and rely on session-based auth). The `/api/v1/articles/...` routes are the external API surface: they use x-api-key → external_api_keys table lookup, enforce per-user ownership, and have dedicated endpoints per action (not action-field dispatching). Discovered empirically: `/api/articles` POST → 404 on cjgeoai.com; `/api/v1/articles` POST → 201.

**Consequence if violated:** All weapon API calls would 404 or 403 on production. The skill book's `refApiActions` is stale/wrong for external API consumers.

## 2026-06-16: Supabase direct reads for article data

**Decision:** `readArticleDirect` reads from CJGEO's Supabase (ref `<REDACTED-PROJECT-REF>`) using the service role key from `C:/Users/xsj70/cjgeo/.env.local`, because the v1 API doesn't expose read-only get_assets/get_html endpoints.

**Rejected alternative:** Adding get_assets/get_html to the v1 API routes in the CJGEO codebase.

**Why:** The v1 API is intentionally action-oriented (write operations only: create, set keyword, research, generate, adopt). Read operations go through the CJGEO Supabase directly with service role key. The key is loaded from the cjgeo repo's .env.local at runtime (fallback from CJGEO_SUPABASE_SERVICE_ROLE_KEY env var).

**Consequence if violated:** No way to read article assets/content for gate checks or publishing adapters. The publishing adapters depend on content_html reads.

## 2026-06-16: Draft generation works on production only (CONFIRMED)

**Decision:** Use production cjgeoai.com (not localhost:3000) as the base URL for draft generation.

**Rejected alternative:** Running draft generation against the local dev server.

**Why:** CJGEO's draft generation uses a v0 AI engine that fires async after the API call. The local dev server accepts the request but the generation worker fails ("Generation failed" after ~1min). Production (cjgeoai.com on Vercel) successfully processes: queued → rendering → completed. Confirmed 2026-06-16: production draft reached "rendering" status within 2 minutes of the API call.

**Consequence if violated:** Draft generation will always fail on the local dev server, blocking the pipeline at step 6. The weapon's CJGEO_BASE_URL must default to https://cjgeoai.com (production), not localhost.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/cjgeo/index.js -->
```js
/**
 * CJGEO weapon — wraps the CJGEO Content Magic v1 REST API for programmatic
 * full-auto article pipeline execution.
 *
 * Auth: CJGEO_API_KEY env var (x-api-key header on every request).
 * Base URL: CJGEO_BASE_URL env var, defaults to https://cjgeoai.com.
 *
 * v1 API routes (verified from cjgeo codebase 2026-06-16):
 *   POST /api/v1/articles                          — create article
 *   POST /api/v1/articles/{id}/main-keyword        — set main keyword
 *   POST /api/v1/articles/{id}/research/keywords   — keyword research
 *   POST /api/v1/articles/{id}/research/topics     — topic benchmark
 *   POST /api/v1/articles/{id}/draft               — draft generation (fire-and-forget)
 *   GET  /api/v1/articles/{id}/draft/status        — poll draft status
 *   POST /api/v1/articles/{id}/adopt               — adopt draft into content_html
 *
 * NOTE: The skill book cjgeo.refApiActions documents /api/articles/{id} action-based
 * endpoints (get_assets, get_html, etc.) — those are the LEGACY internal routes at
 * /api/articles/[articleId]/route.ts, NOT the v1 API. Both coexist; the v1 routes
 * are the external-API-key path. Assets/HTML reads go through CJGEO Supabase direct
 * when needed (the v1 API doesn't expose read-only get_assets/get_html endpoints).
 */

export const toc = {
  writeArticle:
    "Create a new CJGEO article via the v1 API. Returns article_id.",
  writeMainKeyword:
    "Commit the main keyword on an article. Required before research stages.",
  runKeywordResearch:
    "Run SEO keyword research on an article (5-min chain).",
  runTopicBenchmark:
    "Run competitor topic benchmark on an article (5-min chain).",
  writeDraftGenerate:
    "Kick off async draft generation (fire-and-forget). Poll via readDraftStatus.",
  readDraftStatus:
    "Poll draft generation status (GET). Returns queued/sending/rendering/completed/failed.",
  writeDraftAdopt:
    "Adopt the completed draft into content_html. 409 if draft not completed.",
  readArticleDirect:
    "Read article fields directly from CJGEO Supabase (bypasses API for reads).",
  runFullAutoPipeline:
    "Run the complete no-gating pipeline: create -> keyword -> research -> draft -> poll -> adopt.",
  publishToCjgeoai:
    "cjgeoai.com adapter: verify article is adopted and return the live URL.",
  publishToBosterbio:
    "bosterbio.com adapter: fetch CJGEO content, sanitize for Magento, POST to CMS mutation endpoint.",
  checkCredentials:
    "Check that CJGEO API credentials are configured.",
};

/* ── Internals ─────────────────────────────────────────────────────── */

// Redacted from source — set CJGEO_SUPABASE_URL in .env.local (see .env.example).
const CJGEO_SUPABASE_URL = process.env.CJGEO_SUPABASE_URL;

function loadConfig() {
  const apiKey = process.env.CJGEO_API_KEY;
  const baseUrl = (process.env.CJGEO_BASE_URL || "https://cjgeoai.com").replace(/\/+$/, "");
  if (!apiKey) throw new Error("CJGEO_API_KEY not set");
  return { apiKey, baseUrl };
}

import { readFileSync } from "fs";
import { homedir } from "os";
import { join } from "path";

function loadCjgeoSupabaseKey() {
  const key = process.env.CJGEO_SUPABASE_SERVICE_ROLE_KEY;
  if (key) return key;
  try {
    const envPath = process.env.CJGEO_ENV_FILE || join(homedir(), "cjgeo", ".env.local");
    const envContent = readFileSync(envPath, "utf8");
    const match = envContent.match(/^SUPABASE_SERVICE_ROLE_KEY=(.+)$/m);
    if (match) return match[1].trim();
  } catch { /* ignore */ }
  throw new Error("CJGEO Supabase service role key not available");
}

async function apiPost(path, body = {}) {
  const { apiKey, baseUrl } = loadConfig();
  const url = `${baseUrl}${path}`;
  const res = await fetch(url, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "x-api-key": apiKey,
    },
    body: JSON.stringify(body),
  });
  const text = await res.text();
  let json;
  try { json = JSON.parse(text); } catch { json = { raw: text }; }
  if (!res.ok) {
    const msg = json?.error || `HTTP ${res.status}`;
    throw new Error(`CJGEO API ${path}: ${msg}`);
  }
  return json;
}

async function apiGet(path) {
  const { apiKey, baseUrl } = loadConfig();
  const url = `${baseUrl}${path}`;
  const res = await fetch(url, {
    method: "GET",
    headers: { "x-api-key": apiKey },
  });
  const text = await res.text();
  let json;
  try { json = JSON.parse(text); } catch { json = { raw: text }; }
  if (!res.ok) {
    const msg = json?.error || `HTTP ${res.status}`;
    throw new Error(`CJGEO API GET ${path}: ${msg}`);
  }
  return json;
}

async function supabaseRead(table, select, filter = "") {
  if (!CJGEO_SUPABASE_URL) throw new Error("CJGEO_SUPABASE_URL not set — configure it in .env.local (see .env.example).");
  const key = loadCjgeoSupabaseKey();
  const url = `${CJGEO_SUPABASE_URL}/rest/v1/${table}?select=${encodeURIComponent(select)}${filter ? "&" + filter : ""}`;
  const res = await fetch(url, {
    headers: { apikey: key, Authorization: `Bearer ${key}` },
  });
  if (!res.ok) {
    const text = await res.text();
    throw new Error(`CJGEO Supabase read ${table}: ${text}`);
  }
  return res.json();
}

function sleep(ms) { return new Promise(r => setTimeout(r, ms)); }

/* ── Exports ───────────────────────────────────────────────────────── */

export async function writeArticle({ title, mainKeyword } = {}) {
  if (!title) throw new Error("title is required");
  const body = { title };
  if (mainKeyword) body.main_keyword = mainKeyword;
  const res = await apiPost("/api/v1/articles", body);
  return { article_id: res.article_id || res.id, reused: false };
}

export async function writeMainKeyword({ articleId, mainKeyword } = {}) {
  if (!articleId || !mainKeyword) throw new Error("articleId and mainKeyword required");
  return apiPost(`/api/v1/articles/${articleId}/main-keyword`, { main_keyword: mainKeyword });
}

export async function runKeywordResearch({ articleId, keyword } = {}) {
  if (!articleId) throw new Error("articleId required");
  const body = keyword ? { keyword } : {};
  return apiPost(`/api/v1/articles/${articleId}/research/keywords`, body);
}

export async function runTopicBenchmark({ articleId, keyword } = {}) {
  if (!articleId) throw new Error("articleId required");
  const body = keyword ? { keyword } : {};
  return apiPost(`/api/v1/articles/${articleId}/research/topics`, body);
}

export async function writeDraftGenerate({ articleId, skipResearch = false } = {}) {
  if (!articleId) throw new Error("articleId required");
  return apiPost(`/api/v1/articles/${articleId}/draft`, { skip_research: skipResearch });
}

export async function readDraftStatus({ articleId } = {}) {
  if (!articleId) throw new Error("articleId required");
  return apiGet(`/api/v1/articles/${articleId}/draft/status`);
}

export async function writeDraftAdopt({ articleId } = {}) {
  if (!articleId) throw new Error("articleId required");
  return apiPost(`/api/v1/articles/${articleId}/adopt`, {});
}

/**
 * Read article fields directly from CJGEO Supabase (bypasses API for reads).
 */
export async function readArticleDirect({ articleId, fields = "id,title,status,assets,content_html" } = {}) {
  if (!articleId) throw new Error("articleId required");
  const rows = await supabaseRead("content_magic_articles", fields, `id=eq.${articleId}`);
  if (!rows.length) throw new Error(`Article ${articleId} not found in CJGEO Supabase`);
  return rows[0];
}

/**
 * Run the complete no-gating full-auto pipeline.
 * @param {object} opts
 * @param {string} opts.title - Article title
 * @param {string} opts.mainKeyword - The main keyword/topic
 * @param {number} [opts.pollIntervalMs=60000] - Poll interval (default 1min)
 * @param {number} [opts.pollTimeoutMs=1200000] - Poll timeout (default 20min)
 */
export async function runFullAutoPipeline({ title, mainKeyword, pollIntervalMs = 60_000, pollTimeoutMs = 1_200_000 } = {}) {
  if (!title || !mainKeyword) throw new Error("title and mainKeyword required");

  const stages = [];
  const log = (name, result) => { stages.push({ stage: name, result, ts: new Date().toISOString() }); };

  // 1. Create article
  const created = await writeArticle({ title, mainKeyword });
  log("writeArticle", created);
  const articleId = created.article_id;

  // 2. Set main keyword
  const kw = await writeMainKeyword({ articleId, mainKeyword });
  log("writeMainKeyword", kw);

  // 3. Keyword research
  const kwRes = await runKeywordResearch({ articleId });
  log("runKeywordResearch", kwRes);

  // 4. Topic benchmark
  const topicRes = await runTopicBenchmark({ articleId });
  log("runTopicBenchmark", topicRes);

  // 5. Verify assets via Supabase direct read
  const article = await readArticleDirect({ articleId, fields: "id,assets" });
  const assets = article.assets || {};
  if (!assets.main_keyword) throw new Error("Gate failed: main_keyword missing after research");
  log("gateCheck", { main_keyword: assets.main_keyword, keywords: assets.keywords?.length || 0, topics: assets.topics?.length || 0 });

  // 6. Generate draft (fire-and-forget)
  const gen = await writeDraftGenerate({ articleId });
  log("writeDraftGenerate", gen);

  // 7. Poll until completed
  const pollStart = Date.now();
  let completed = false;
  while (Date.now() - pollStart < pollTimeoutMs) {
    await sleep(pollIntervalMs);
    const status = await readDraftStatus({ articleId });
    log("readDraftStatus", status);
    if (status.status === "completed") { completed = true; break; }
    if (status.status === "failed") {
      return { article_id: articleId, stages, error: `draft generation failed: ${status.error || "unknown"}` };
    }
  }
  if (!completed) {
    return { article_id: articleId, stages, error: "draft poll timeout — reinitiate needed" };
  }

  // 8. Adopt draft
  const adopted = await writeDraftAdopt({ articleId });
  log("writeDraftAdopt", adopted);

  // 9. Verify content via Supabase
  const final = await readArticleDirect({ articleId, fields: "id,title,content_html,status" });
  const contentLength = final.content_html?.length || 0;
  log("verifyContent", { content_length: contentLength, status: final.status });

  return { article_id: articleId, stages, content_length: contentLength };
}

export async function publishToCjgeoai({ articleId } = {}) {
  if (!articleId) throw new Error("articleId required");
  const { baseUrl } = loadConfig();
  const article = await readArticleDirect({ articleId, fields: "id,title,content_html,status" });
  const len = article.content_html?.length || 0;
  if (len < 100) {
    return { published: false, url: null, error: "content_html too short or empty — draft not adopted?" };
  }
  return {
    published: true,
    url: `${baseUrl}/content-magic/${articleId}`,
    content_length: len,
    title: article.title,
  };
}

/**
 * bosterbio.com publishing adapter — fetch adopted content from CJGEO,
 * sanitize for Magento, and POST to the CMS mutation endpoint.
 * @param {object} opts
 * @param {string} opts.articleId - CJGEO article UUID (must have adopted content_html)
 * @param {string} opts.slug - Magento CMS identifier (e.g. "proving-grounds/secondary-antibody-wb")
 * @param {string} [opts.title] - Page title (defaults to article title)
 * @param {string} [opts.mainKeyword] - For meta tags
 */
export async function publishToBosterbio({ articleId, slug, title, mainKeyword } = {}) {
  if (!articleId || !slug) throw new Error("articleId and slug required");

  // 1. Read adopted content_html from CJGEO Supabase
  const article = await readArticleDirect({ articleId, fields: "id,title,content_html,assets" });
  const rawHtml = article.content_html || "";
  if (rawHtml.length < 100) {
    return { published: false, error: "content_html too short — draft not adopted?" };
  }
  const pageTitle = title || article.title;
  const keyword = mainKeyword || article.assets?.main_keyword || "";

  // 2. Sanitize: strip page chrome, preserve CJGEO design system
  let body = rawHtml;
  // Strip outer wrappers that Magento's theme provides
  body = body.replace(/<!DOCTYPE[^>]*>/gi, "");
  body = body.replace(/<html[^>]*>|<\/html>/gi, "");
  body = body.replace(/<head[\s\S]*?<\/head>/gi, "");
  body = body.replace(/<body[^>]*>|<\/body>/gi, "");
  body = body.replace(/<main[^>]*>|<\/main>/gi, "");
  body = body.replace(/<script[\s\S]*?<\/script>/gi, "");
  body = body.replace(/<header[\s\S]*?<\/header>/gi, "");
  body = body.replace(/<footer[\s\S]*?<\/footer>/gi, "");
  body = body.replace(/<nav[\s\S]*?<\/nav>/gi, "");
  body = body.replace(/<!--[\s\S]*?-->/g, "");
  // Decode HTML entities
  body = body.replace(/&mdash;/g, "\u2014");
  body = body.replace(/&nbsp;/g, "\u00A0");
  body = body.replace(/&hellip;/g, "\u2026");
  body = body.replace(/&rsquo;/g, "\u2019");
  // Keep <style data-draft-styles>, <h1>, and all CJGEO component classes intact
  body = body.trim();

  // 3. POST to Magento CMS mutation endpoint
  const magentoPayload = {
    handshake: "<REDACTED-SHARED-SECRET>",
    identifier: slug,
    title: pageTitle,
    content: body,
    content_heading: "",
    meta_title: pageTitle,
    meta_description: `${pageTitle} - Expert guide on ${keyword}`,
    meta_keywords: keyword,
    is_active: true,
    store_id: 1,
    page_layout: "1column",
  };

  const res = await fetch("https://www.bosterbio.com/rest/V1/bapi/cms-page", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(magentoPayload),
  });
  const resText = await res.text();
  let resJson;
  try { resJson = JSON.parse(resText); } catch { resJson = { raw: resText }; }

  if (!res.ok) {
    return { published: false, error: `Magento CMS POST failed: ${res.status} ${resJson?.message || resText.slice(0, 200)}` };
  }

  // Response shape: [page_id, identifier, is_create]
  const pageId = Array.isArray(resJson) ? resJson[0] : null;
  const isCreate = Array.isArray(resJson) ? resJson[2] : null;

  return {
    published: true,
    url: `https://www.bosterbio.com/${slug}`,
    page_id: pageId,
    is_create: isCreate,
    content_length: body.length,
  };
}

export function checkCredentials() {
  const key = process.env.CJGEO_API_KEY;
  const base = process.env.CJGEO_BASE_URL || "https://cjgeoai.com";
  let supabaseOk = false;
  try { loadCjgeoSupabaseKey(); supabaseOk = true; } catch { /* */ }
  return {
    configured: !!key,
    base_url: base,
    key_preview: key ? `${key.slice(0, 8)}...` : null,
    supabase_readable: supabaseOk,
  };
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/openai_images/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/weapon/openai_images`

Rationale layer (per CLAUDE.md §6). Append; supersede in place when a decision
changes.

---

## 2026-06-09 — never pass `response_format` to the images endpoints

**Decision.** `generate`/`edit` never send `response_format`. `b64_json` is always
returned regardless.

**Rejected alternative.** Pass `response_format: "b64_json"` for explicitness (the
intuitive, "correct-looking" call).

**Why.** OpenAI's image endpoints 400 with `unknown_parameter` when
`response_format` is supplied — the parameter isn't in the current schema, yet
b64_json is the only format and is always returned. Sending it for "clarity" breaks
every call.

**Consequence if violated.** All image calls fail with `unknown_parameter`; the
caller suspects their request shape when it's a service-side quirk.

---

## 2026-06-09 — the vision judge uses `gpt-4o`; `gpt-4o-mini` is banned for verification (commit 5f9b4c0)

**Decision.** `judge` runs on full `gpt-4o`. `gpt-4o-mini` is explicitly banned for
the verification path.

**Rejected alternative.** Use `gpt-4o-mini` to cut verification cost.

**Why.** Mini measured a ~49% false-alarm rate on dense UI screenshots (the actual
use case) — insufficient visual reasoning. Verification is a gate; a cheap judge
that hallucinates failures (or passes) defeats its purpose. The cost/accuracy
trade-off was settled at 5f9b4c0 so it isn't re-litigated.

**Consequence if violated.** The visual gate accepts wrong images (or false-alarms
on good ones), undermining the verification system that the rest of GuildOS leans
on.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/openai_images/index.js -->
```js
/**
 * OpenAI Images weapon — image generation and editing.
 *
 * Model: configured via OPENAI_IMAGE_MODEL env var, default "gpt-image-2".
 * To swap models cluster-wide (e.g. when a newer model ships), update the env
 * var in deployment + every locally-running .env.local — no code change.
 *
 * ─── CALLER DISCIPLINE (LOCKED 2026-05-22) ────────────────────────────────
 * Callers MUST be API routes (app/api/**) or skill-book actions
 * (libs/skill_book/**). One-off `.mjs` scripts in `scripts/` are NOT a
 * supported mechanism — they bypass the env-driven model source of truth in
 * practice (people hardcode the model name) and bypass the API-route audit
 * surface. The legacy `scripts/_generate-marketing-infographics.mjs` and
 * `scripts/_design_smoke_v4_gpt_image_2.mjs` are NOT precedent for new
 * one-offs; they're being deprecated. If the dev server is broken, surface
 * that — don't fall back to a script.
 *
 * Credential: OPENAI_API_KEY from profiles.env_vars, fallback to process.env.
 *
 * ─── API surface ──────────────────────────────────────────────────────────
 * Generate: POST https://api.openai.com/v1/images/generations
 *   body (JSON): { model, prompt, size, quality, n }
 *   response:    { data: [{ b64_json }], usage }
 *
 * Edit: POST https://api.openai.com/v1/images/edits  (multipart/form-data)
 *   fields: model, prompt, size, quality, n, image[] (Blob | Buffer)
 *   response: { data: [{ b64_json }], usage }
 *
 * ─── Verified behavior (smoke-tested 2026-04-23) ──────────────────────────
 * - DO NOT pass response_format — causes unknown_parameter error; b64_json is
 *   always returned regardless.
 * - Supported sizes: "1024x1024" | "1024x1536" | "1536x1024" | "auto"
 * - Supported quality: "low" | "medium" | "high"
 * - gpt-image-2 requires org verification on the OpenAI platform before the
 *   model unlocks for API calls.
 */

import { database } from "@/libs/council/database";

export const toc = {
  generate: {
    description:
      "Generate an image — infographic, chart, diagram, poster, illustration, logo, icon, banner, slide background, artwork, or photo — from a text prompt using gpt-image-2 (\"ChatGPT image\"). THE way to create any picture / visual / graphic from words. Returns array of b64_json strings. Via run-action, pass a long/multiline prompt with --input-file <json> and write the PNG(s) with --out <path> (n>1 → -1,-2… suffix).",
    // Binary-output contract honored by run-action's `--out`: the result IS an
    // array of base64 PNG payloads ("from":"$"), each decoded and written to a
    // .png file. This is the metadata the adapter cli.mjs used to carry.
    output: { binary: true, encoding: "base64", contentType: "image/png", ext: "png", from: "$" },
  },
  edit: {
    description:
      "Edit, restyle, redraw, composite, inpaint, or merge existing images (change style, swap / add / remove elements, combine multiple pictures) using gpt-image-2 edits endpoint. Accepts image Buffers + prompt. Returns array of b64_json strings. Via run-action, write the PNG(s) with --out <path>.",
    output: { binary: true, encoding: "base64", contentType: "image/png", ext: "png", from: "$" },
  },
  judge:
    "Contextless vision check: ask gpt-4o (full) whether a given image matches a written claim. Optional baselineUrl adds a second image (the AI-approved design baseline) — pass requires both claim satisfaction AND ≥90% visual similarity (or all key elements present). Returns { verdict, confidence, reasoning, model, baseline_compared }. No chat history, no narrative drift. gpt-4o-mini was retired 2026-04-26 (49% false-alarm rate, see CLAUDE.md WWCD bans).",
};

/**
 * API base URL. Defaults to OpenAI direct. Override with OPENAI_IMAGE_BASE_URL
 * (or OPENAI_BASE_URL) to route through a supported-region OpenAI-compatible
 * proxy — needed where OpenAI geo-blocks the images endpoint by request origin
 * ("Country, region, or territory not supported"). Value should be the full
 * base incl. version segment, e.g. "https://my-proxy.example.com/v1".
 */
function apiBase() {
  return (
    process.env.OPENAI_IMAGE_BASE_URL ||
    process.env.OPENAI_BASE_URL ||
    "https://api.openai.com/v1"
  );
}

/**
 * Read the active image-generation model. Single source of truth — every
 * caller (weapon + scripts) goes through this so a model bump is one env
 * change, not a code-wide grep-and-replace.
 */
export function imageModel() {
  return process.env.OPENAI_IMAGE_MODEL || "gpt-image-2";
}

async function loadKey(userId) {
  if (userId) {
    try {
      const db = await database.init("service");
      const { data } = await db
        .from("profiles")
        .select("env_vars")
        .eq("id", userId)
        .single();
      const k = data?.env_vars?.OPENAI_API_KEY;
      if (k) return k;
    } catch {
      /* fall through */
    }
  }
  return process.env.OPENAI_API_KEY || null;
}

/**
 * Generate images from a text prompt.
 *
 * @param {{ prompt: string, size?: string, quality?: string, n?: number, background?: "transparent"|"opaque"|"auto" }} input
 *   background — gpt-image-2 alpha control. "transparent" returns a true alpha PNG
 *   (use for stickers/badges/logos that overlay other imagery). Omit for default.
 * @param {string} [userId]
 * @returns {Promise<string[]>} Array of base64-encoded PNG strings (one per image)
 */
export async function generate(
  { prompt, size = "1024x1024", quality = "medium", n = 1, background } = {},
  userId
) {
  const key = await loadKey(userId);
  if (!key) throw new Error("OPENAI_API_KEY not configured");

  const body = { model: imageModel(), prompt, size, quality, n };
  if (background) body.background = background;

  const res = await fetch(`${apiBase()}/images/generations`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${key}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(body),
  });

  const json = await res.json();
  if (json.error) throw new Error(`OpenAI Images error: ${json.error.message}`);
  return json.data.map((d) => d.b64_json);
}

/**
 * Edit / restyle one or more images.
 *
 * @param {{ images: (Buffer|Blob)[], prompt: string, size?: string, quality?: string, n?: number }} input
 *   images — array of image Buffers (PNG preferred). Max 10 inputs.
 * @param {string} [userId]
 * @returns {Promise<string[]>} Array of base64-encoded PNG strings
 */
export async function edit(
  { images, prompt, size = "1024x1024", quality = "medium", n = 1 } = {},
  userId
) {
  const key = await loadKey(userId);
  if (!key) throw new Error("OPENAI_API_KEY not configured");
  if (!images?.length) throw new Error("At least one image is required for edit");

  const form = new FormData();
  form.append("model", imageModel());
  form.append("prompt", prompt);
  form.append("size", size);
  form.append("quality", quality);
  form.append("n", String(n));

  for (const img of images) {
    const blob =
      img instanceof Blob ? img : new Blob([img], { type: "image/png" });
    form.append("image[]", blob, "image.png");
  }

  const res = await fetch(`${apiBase()}/images/edits`, {
    method: "POST",
    headers: { Authorization: `Bearer ${key}` },
    body: form,
  });

  const json = await res.json();
  if (json.error) throw new Error(`OpenAI Images edit error: ${json.error.message}`);
  return json.data.map((d) => d.b64_json);
}

/**
 * Contextless image-vs-claim judge. Each call is its own zero-history request to
 * the chat-completions endpoint with a vision message. Used as a "second pair of
 * eyes" gate — the judging model has no narrative about the work, no pressure to
 * defend prior output, no risk of overanchoring on the agent's own description.
 * Strict JSON output for downstream gating.
 *
 * @param {{ imageUrl: string, claim: string, baselineUrl?: string, model?: string }} input
 *   imageUrl    — public URL of the ACTUAL image being verified (the worker's
 *                 deliverable). OpenAI server must be able to fetch it.
 *   claim       — what the image is supposed to show. THE canonical claim
 *                 (typically items.expectation verbatim).
 *   baselineUrl — OPTIONAL. The AI-approved design baseline image from the
 *                 design stage. When present, the judge ALSO checks that the
 *                 actual is ≥90% similar to the baseline OR contains all key
 *                 elements (layout, components, hierarchy). Both checks must
 *                 pass for a 'match' verdict. When absent, behavior is the
 *                 same as before — text-only claim check.
 *   model       — defaults to "gpt-4o" (full). gpt-4o-mini is BANNED for
 *                 verification (CLAUDE.md WWCD bans, 5f9b4c0): 49% false-alarm
 *                 rate on dense screenshots.
 * @param {string} [userId]
 * @returns {Promise<{ verdict: 'match'|'mismatch'|'inconclusive', confidence: number, reasoning: string, model: string, baseline_compared: boolean }>}
 */
export async function judge({ imageUrl, claim, baselineUrl, model = "gpt-4o" } = {}, userId) {
  if (typeof imageUrl !== "string" || !imageUrl.trim()) {
    throw new Error("judge: imageUrl is required");
  }
  if (typeof claim !== "string" || !claim.trim()) {
    throw new Error("judge: claim is required");
  }
  const hasBaseline = typeof baselineUrl === "string" && !!baselineUrl.trim();

  const key = await loadKey(userId);
  if (!key) throw new Error("OPENAI_API_KEY not configured");

  const systemPromptBase = [
    "You are a strict image verification judge.",
    "Your only job is to decide whether the deliverable image satisfies the claim.",
    "Be conservative: if you cannot confirm from the image(s) alone, return 'mismatch' or 'inconclusive'.",
    "Common failure modes to flag as mismatch:",
    "  - stock photo / generic image instead of the claimed subject",
    "  - blank / empty / loading state instead of populated content",
    "  - error page, 404, login screen, dev overlay instead of the claimed page",
    "  - placeholder text or skeleton loaders never resolved",
    "  - URL bar / page title clearly inconsistent with the claim",
  ];
  const systemPromptBaselineExtra = [
    "You will be given TWO images: the ACTUAL deliverable AND a BASELINE design that was AI-approved at the design stage.",
    "Pass requires BOTH:",
    "  (a) the actual image satisfies the claim text, AND",
    "  (b) the actual image is ≥90% visually similar to the baseline OR contains all key elements (layout, components, hierarchy, primary content blocks).",
    "Cosmetic differences (minor spacing, font weight) are tolerated. Missing whole sections, wrong layout, or different page entirely → mismatch.",
  ];
  const systemPromptOutput = [
    "Respond with VALID JSON ONLY, no markdown fences, no preamble:",
    '{ "verdict": "match" | "mismatch" | "inconclusive", "confidence": 0.0-1.0, "reasoning": "<one sentence>" }',
  ];
  const systemPrompt = (hasBaseline
    ? [...systemPromptBase, ...systemPromptBaselineExtra, ...systemPromptOutput]
    : [...systemPromptBase, ...systemPromptOutput]
  ).join("\n");

  const userMessage = hasBaseline
    ? [
        {
          type: "text",
          text: `Claim: ${claim}\n\nFirst image: ACTUAL deliverable.\nSecond image: BASELINE design (AI-approved reference).\nDoes the actual satisfy the claim AND match the baseline?`,
        },
        { type: "image_url", image_url: { url: imageUrl } },
        { type: "image_url", image_url: { url: baselineUrl } },
      ]
    : [
        { type: "text", text: `Claim: ${claim}\n\nDoes the image match this claim?` },
        { type: "image_url", image_url: { url: imageUrl } },
      ];

  const res = await fetch(`${apiBase()}/chat/completions`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${key}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model,
      messages: [
        { role: "system", content: systemPrompt },
        { role: "user", content: userMessage },
      ],
      temperature: 0,
      max_tokens: 200,
      response_format: { type: "json_object" },
    }),
  });

  const json = await res.json();
  if (json.error) throw new Error(`OpenAI judge error: ${json.error.message}`);
  const content = json?.choices?.[0]?.message?.content;
  if (typeof content !== "string") {
    throw new Error("OpenAI judge returned no content");
  }

  let parsed;
  try {
    parsed = JSON.parse(content);
  } catch {
    throw new Error(`OpenAI judge returned non-JSON content: ${content.slice(0, 200)}`);
  }

  const verdict =
    parsed.verdict === "match" || parsed.verdict === "mismatch" || parsed.verdict === "inconclusive"
      ? parsed.verdict
      : "inconclusive";
  const confidence =
    typeof parsed.confidence === "number" && parsed.confidence >= 0 && parsed.confidence <= 1
      ? parsed.confidence
      : 0;
  const reasoning = typeof parsed.reasoning === "string" ? parsed.reasoning : "";

  return { verdict, confidence, reasoning, model, baseline_compared: hasBaseline };
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/supabase_storage/index.js -->
```js
/**
 * Supabase Storage weapon — unified upload/download with path conventions.
 *
 * Auth: Uses SUPABASE_SECRETE_KEY via database service role.
 *
 * Storage convention:
 * - Bucket: GuildOS_Bucket (public)
 * - Path: {channel}/{questId}/{filename} — channel defaults to "general";
 *         cloud agent artifacts use channel="cursor_cloud"
 * - Retention: 30 days (enforced via bucket lifecycle policy or manual cleanup)
 */

export const toc = {
  buildPath: "Build a canonical storage path: {channel}/{questId}/{filename}.",
  writeFile: "Upload a file buffer to Supabase Storage.",
  readPublicUrl: "Get the public URL for a stored file.",
  readFile: "Download a file from Supabase Storage.",
  searchFiles: "Search files in a storage path.",
  deleteFiles: "Delete files from Supabase Storage.",
};
import { database } from "@/libs/council/database";

const DEFAULT_BUCKET = "GuildOS_Bucket";

// ---------------------------------------------------------------------------
// Helpers
// ---------------------------------------------------------------------------

async function getClient() {
  return database.init("service");
}

/**
 * Build a standardized storage path.
 * Convention: {channel}/{questId}/{filename}
 * @param {{ channel?: string, questId?: string, filename: string }} input
 */
export function buildPath({ channel = "general", questId, filename }) {
  if (!filename) throw new Error("filename is required");
  const parts = [channel];
  if (questId) parts.push(questId);
  parts.push(filename);
  return parts.join("/");
}

// ---------------------------------------------------------------------------
// Upload
// ---------------------------------------------------------------------------

/**
 * Upload a file to Supabase Storage.
 * @param {{ path: string, content: Buffer|Blob|string, contentType?: string, bucket?: string, upsert?: boolean }} input
 */
export async function writeFile({ path, content, contentType = "application/octet-stream", bucket = DEFAULT_BUCKET, upsert = true } = {}) {
  if (!path) throw new Error("path is required");
  if (!content) throw new Error("content is required");
  const db = await getClient();
  const { data, error } = await db.storage
    .from(bucket)
    .upload(path, content, { contentType, upsert });
  if (error) throw new Error(`Storage upload failed: ${error.message}`);
  return { path: data?.path || path, fullPath: data?.fullPath || path };
}

/**
 * Get the public URL for a stored file.
 * @param {{ path: string, bucket?: string }} input
 */
export function readPublicUrl({ path, bucket = DEFAULT_BUCKET } = {}) {
  const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL;
  if (!supabaseUrl) throw new Error("NEXT_PUBLIC_SUPABASE_URL not set");
  return {
    url: `${supabaseUrl}/storage/v1/object/public/${bucket}/${path}`,
  };
}

// ---------------------------------------------------------------------------
// Download
// ---------------------------------------------------------------------------

/**
 * Download a file from Supabase Storage.
 * @param {{ path: string, bucket?: string }} input
 */
export async function readFile({ path, bucket = DEFAULT_BUCKET } = {}) {
  if (!path) throw new Error("path is required");
  const db = await getClient();
  const { data, error } = await db.storage
    .from(bucket)
    .download(path);
  if (error) throw new Error(`Storage download failed: ${error.message}`);
  return { blob: data, path };
}

// ---------------------------------------------------------------------------
// List
// ---------------------------------------------------------------------------

/**
 * List files in a storage path.
 * @param {{ path?: string, bucket?: string, limit?: number }} input
 */
export async function searchFiles({ path = "", bucket = DEFAULT_BUCKET, limit = 100 } = {}) {
  const db = await getClient();
  const { data, error } = await db.storage
    .from(bucket)
    .list(path, { limit, sortBy: { column: "created_at", order: "desc" } });
  if (error) throw new Error(`Storage list failed: ${error.message}`);
  return { files: data || [] };
}

// ---------------------------------------------------------------------------
// Delete
// ---------------------------------------------------------------------------

/**
 * Delete files from Supabase Storage.
 * @param {{ paths: string[], bucket?: string }} input
 */
export async function deleteFiles({ paths, bucket = DEFAULT_BUCKET } = {}) {
  if (!Array.isArray(paths) || paths.length === 0) throw new Error("paths array is required");
  const db = await getClient();
  const { error } = await db.storage
    .from(bucket)
    .remove(paths);
  if (error) throw new Error(`Storage delete failed: ${error.message}`);
  return { ok: true, deleted: paths.length };
}

// ---------------------------------------------------------------------------
// Credential check
// ---------------------------------------------------------------------------

export async function checkCredentials() {
  try {
    const url = process.env.NEXT_PUBLIC_SUPABASE_URL;
    const key = process.env.SUPABASE_SECRETE_KEY;
    if (!url) return { ok: false, msg: "Missing NEXT_PUBLIC_SUPABASE_URL" };
    if (!key) return { ok: false, msg: "Missing SUPABASE_SECRETE_KEY" };
    return { ok: true, msg: "Supabase Storage credentials present" };
  } catch (e) {
    return { ok: false, msg: `Error: ${e.message}` };
  }
}
```
<!-- /GUILDOS-FILE -->
