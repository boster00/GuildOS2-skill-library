# cursorCJGEO

<!-- GUILDOS-SKILL name="cursorCJGEO" -->

<!-- GUILDOS-DESC -->
Dispatch and monitor a Cursor cloud agent to generate a CJGEO article and integrate it as a bryanblog page. Owns the chaperone-side dispatch loop, the .cursorrules template handoff, credential handoff via followups, FF-merge of the Cursor sandbox branch, and live CIC verification.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/cursorCJGEO/SKILL.md -->
```markdown
---
name: cursorCJGEO
description: >-
  Dispatch and monitor a Cursor cloud agent to generate a CJGEO article and
  integrate it as a bryanblog page. Owns the chaperone-side dispatch loop, the
  .cursorrules template handoff, credential handoff via followups, FF-merge of
  the Cursor sandbox branch, and live CIC verification.
---

# cursorCJGEO — Cursor cloud-agent dispatch for CJGEO→bryanblog hydration

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/cursorCJGEO/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto cursorCJGEO <action>` (or read the source directly).

## Actions

- **chaperoneMerge** — The FF-merge step the chaperone runs after the Cursor agent FINISHEDs successfully. Cursor sandbox cannot push to main; the chaperone bridges.
- **cicVerify** — Visual-verify the live bryanblog URL via CIC after Vercel auto-deploys. Defers to the browsercontrol skill book for the actual procedure.
- **dispatchCursor** — Step-by-step: dispatch a Cursor cloud agent to a CJGEO+bryanblog hydration task. The chaperone-side bootstrap procedure.
- **FullAutoWebsiteDevelopmentRoadmap** — Stand up a NEW content-focused website end-to-end leveraging CJGEO + Cursor cloud agents — decision-journey topics in, live Vercel site out. Milestone-based roadmap that an SE overseer Claude thread executes via [SB:supervisedEvolution->run]. Designed so a fresh thread can ENTER AT ANY MILESTONE by inspecting the asset state and resuming from the next chapter — no re-running upstream work that already converged.
- **monitorCursor** — The chaperone monitoring loop: poll status, read conversation, diagnose blockers, send followups, handle FINISHED and EXPIRED. Foreground until done.
- **refAntipatterns** — Reference (LOCKED): discipline antipatterns observed on chaperone + leaf-agent runs. The 'waiting for notification' self-defer is the most common. Read before kicking off any monitorCursor loop.
- **refBosterbioDefault** — Reference (LOCKED 2026-05-21, route hydration live at cjgeo 22fedb6d): when working on bosterbio.com projects (cursorCJGEO's main job), always pass use_custom_template:true on generateDraft AND verify the adopted content_html contains at least one Boster template signature class (hero-section, col-md-4, dark-overlap, font-large, card-item, row-of-cards, hero-twin, font-bold) before sanitize/publish. Regenerate if missing.
- **refBracketPlaceholderScan** — Pattern: detect CJGEO image/diagram placeholders by scanning innerText for bracketed tokens and LLM-classifying each as PLACEHOLDER vs PROSE. Generalizes across CJGEO's many variant shapes (`[Diagram:`, `[Chart:`, `[VISUAL PLACEHOLDER:`, `[Image:`, `[Illustration of …]`, etc.) without per-project regex tuning. Used by [[FullAutoWebsiteDevelopmentRoadmap]] M4/M5 milestone detection + inventory.
- **refCredentialHandoff** — Reference (LOCKED): how to pass secrets (CJGEO_API_KEY etc.) to a Cursor cloud agent. Followups, not .cursorrules, not vercel env pull.
- **refCursorRulesTemplate** — Reference: the proven .cursorrules template that ships the CJGEO + bryanblog procedure into the Cursor cloud sandbox. Adapt per task by swapping the per-task NL prompt, not the template body.
- **refCursorSandboxBehavior** — Reference (LOCKED): the non-obvious rules of the Cursor cloud sandbox observed in the proven run. Branch enforcement, no interactive auth, FINISHED != dead.
- **refCursorVisualVerify** — Reference (LOCKED): cursor cloud agents have a native sandboxed Chrome + Playwright — they can navigate URLs, screenshot, and multimodal-read the rendered page themselves. Instruct visual-verify in natural language. If the agent demurs ("I can't take screenshots"), remind it that it can and that it has done so before.
- **refDispatchAPI** — Reference: the Cursor cloud-agents wire spec — proven endpoints, payloads, auth. Read first; every other action in this skill book talks to this surface.
- **refRepoLayout** — Reference: where things live for cursorCJGEO. Quick map across GuildOS + bryanblog + Cursor surfaces.
- **refReportingContract** — Reference (LOCKED): the structured block every Cursor agent emits in its final assistant message. The chaperone parses it.
- **replicateArticle** — Task pattern: replicate an existing bryanblog article as a new satellite with FRESH CJGEO content. The proven end-to-end flow.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/cursorCJGEO/.cursorrules.template -->
```
# .cursorrules — bryanblog operating manual for autonomous cloud agents

You are a Cursor cloud agent working in the `boster00/bryanblog` repo. This file is the
authoritative operating manual for any task that adds, replaces, or hydrates an article
on bryanblog. The per-task NL prompt gives you the specific job; this file is the procedure.

Read this file end-to-end ONCE at session start. The conventions are non-obvious enough
that skipping ahead burns build cycles.

## 1. What bryanblog is

A Next.js 15 (app router) + React 19 + Tailwind 4 + TypeScript static content site for
Bryan, a retirement-planning advisor. Audience: 55–75-year-olds anxious about running out
of money. Voice: calm, plain English, 8th-grade reading level, compliance-conservative.
Live at https://<REDACTED-HOST>/. Auto-deploys from `main`.

## 2. Repo layout

```
app/
  layout.tsx, page.tsx, globals.css       # globals.css has the Tailwind 4 @theme block
  retirement-income/page.tsx              # pillar (decision-journey landing)
  retirement-income/<slug>/page.tsx       # satellites (12 today)
components/
  ArticleShell.tsx                        # wraps article body, sticky TOC, hero
  Cta.tsx, RelatedArticles.tsx            # both use `not-prose` internally
content/
  retirement-income/<slug>.html           # CJGEO-hydrated article body (source of truth)
lib/
  articles.ts                             # SATELLITES + PILLAR + UTILITY_ARTICLES metadata
  sections.ts                             # extractSections() from h2 ids
scripts/
  adopt-portfolio-planning.mjs            # canonical reference adoption script
next.config.mjs, package.json, tsconfig.json
```

Local dev port: 3004 (`npm run dev -- -p 3004`).

## 3. Stack rules (non-negotiable)

- **Tailwind 4**, configured via `@theme` in `app/globals.css`. NO `tailwind.config.js`. NO `@tailwindcss/typography`. The `.prose-article` class is HAND-WRITTEN in `globals.css`.
- **`.prose-article` is greedy.** Any nested component with its own typography MUST use `not-prose` to opt out, or prose styles silently eat it. `<Cta>` and `<RelatedArticles>` already do this — don't break that.
- **Server components by default.** Article pages are async server components that read HTML from `content/` at request time and inject via `dangerouslySetInnerHTML` inside `<ArticleShell>`.
- **Trailing slashes** on internal hrefs (`/retirement-income/<slug>/`). Don't add `trailingSlash: true` to `next.config.mjs`.

## 4. The pillar/satellite cluster model

`/retirement-income/` is a pillar covering ONE decision journey ("Will my money last in retirement?"). 12 satellites live under `/retirement-income/<slug>/`, each branching from one of the pillar's section H2s. Section H2s are phrased the way a layperson would TYPE INTO ChatGPT/Google ("How much should I keep safe versus invested?"), not how a financial pro would label them.

When adding a new satellite, it slots under `/retirement-income/<slug>/` and gets wired into `lib/articles.ts` via the SATELLITES array.

## 5. Compliance + voice (HARD)

- NO product names. NO insurer names. NO specific return projections (e.g. "5% annual"). NO uncaveated guarantees.
- When mentioning annuity guarantees: ALWAYS append "subject to claims-paying ability".
- CTA framing: "request a retirement income review" (advisor framing), NEVER "buy an annuity".
- NO testimonials / endorsements / "what retirees say" sections.
- Strip any of these from CJGEO-generated content during sanitize. They DO appear in raw output.

## 6. Article-hydration pipeline (CJGEO → bryanblog)

This is the canonical procedure. The reference implementation lives at
`scripts/adopt-portfolio-planning.mjs` — READ IT before you write your own version. Your
new article should follow the same sanitize → keepTopLevelBlocks → inject-h2-ids →
inject-deep-dives → write-content-file → wire-route flow.

For a NEW satellite the steps are:

1. **Create a NEW CJGEO article via `full_auto`** (don't clone an existing CJGEO article body — generate fresh content with a fresh seed/brief).
2. Set main keyword.
3. Run keyword `full_auto` (optional but recommended for asset richness).
4. Run topic `full_auto`.
5. Run prompt `full_auto`.
6. Kick off `generate_draft`.
7. Poll `pull_status` every 60s until `ready` (max ~20 min — re-initiate if it hangs).
8. Adopt the draft.
9. Fetch `content_html` via `get_html`.
10. Sanitize → keepTopLevelBlocks → inject h2 ids (slug-from-text) → inject deep-dive links → write to `content/retirement-income/<new-slug>.html` (with the article_id/keyword/adopted_at header comment).
11. Create `app/retirement-income/<new-slug>/page.tsx` mirroring the portfolio-planning satellite pattern (async server component, reads from `content/`, wraps with `<ArticleShell>`, supplies `relatedFor([...sibling slugs])`).
12. Update `lib/articles.ts`: append to `SATELLITES` with `slug`, `href`, `title`, `teaser`, `source: 'cjgeo'`, `cjgeoArticleId`, `mainKeyword`.
13. `npm run build` — must be green. TS errors fail the build.
14. `git add` + commit + `git push origin main`. Vercel auto-deploys.
15. Verify the live URL renders (curl 200 + the body contains expected h2 ids).

## 7. CJGEO HTTP wire spec (CANONICAL — use these, NOT older docs)

- **Base URL:** `https://cjgeoai.com`
- **Auth header (every request):** `x-api-key: <CJGEO_API_KEY>`. See section 11 for where the key is.
- **Content-Type:** `application/json`

### Create article
```
POST /api/v1/articles
{ "title": "<Article Title>", "keyword": "<seed keyword>", "create": true }
→ 201 { "article_id": "<uuid>", "imported": false }
```

### Set main keyword (commit)
```
POST /api/articles/{article_id}/keyword
{ "action": "set_main_keyword", "main_keyword": "<phrase>" }
→ 200 { "committed": true, "main_keyword": "<phrase>" }
```

### Read assets — **on article ROOT, NOT `/keyword`**
```
POST /api/articles/{article_id}
{ "action": "get_assets" }
→ 200 { "main_keyword", "keywords", "topics", "prompts", "status", ... }
```

### Read HTML — **on article ROOT**
```
POST /api/articles/{article_id}
{ "action": "get_html" }
→ 200 { "content_html": "<...>" }
```

### Keyword full_auto (default action — empty body)
```
POST /api/articles/{article_id}/keyword
{}
→ 200 { "committed": true, "summary": {...} }
```

### Topic full_auto (default action — empty body)
```
POST /api/articles/{article_id}/topic
{}
→ 200 { "committed": true, ... }
```

### Prompt full_auto (default action — empty body)
```
POST /api/articles/{article_id}/prompt
{}
→ 200 { "committed": true, ... }
```

### Generate draft (async — returns 202)
```
POST /api/articles/{article_id}/edit-draft
{
  "action": "generate_draft",
  "mode": "new_draft",
  "use_default_context_prompt": false,
  "context_prompt": "<your brief — see section 8>",
  "generation_prompt_mode": "default",
  "allow_image_generation": false
}
→ 202 { "article_id", "status": "generating", "poll", "chat_id" }
```

### Poll draft status
```
POST /api/articles/{article_id}/edit-draft
{ "action": "pull_status" }
→ 200 { "status": "ready" | "not_ready" }
```

### Adopt draft (commits content_html)
```
POST /api/articles/{article_id}/edit-draft
{ "action": "adopt_draft" }
→ 200 { "committed": true, "status": "adopted", "length": <n> }   # if ready
→ 200 { "status": "not_ready" }                                    # if not ready (NOT 409)
```

**Wire quirks (locked):** keyword full_auto works in production as of commit `f41bf6a`. `get_html` and `get_assets` are on `/api/articles/{id}`, NOT on `/edit-draft`. `adopt_draft` returns HTTP 200 with `status:"not_ready"` (not 409) if the draft isn't ready — treat as a no-op and keep polling.

## 8. Generating a fresh article (NOT cloning)

When the task says "replicate X as X-2 with fresh content", DO NOT pull X's HTML and reuse it. Instead:

1. `create` a new article with a fresh title.
2. `set_main_keyword`.
3. Compose a fresh `context_prompt` that describes the article's INTENT and STRUCTURE (audience, voice, sections to cover) — but do NOT paste the original article's HTML or sentences. CJGEO will generate new prose from the context_prompt. Example:
   ```
   Audience: 55-75 year old approaching or in retirement, anxious about
   running out of money, not a financial pro. Voice: calm, plain English,
   8th-grade reading level, compliance-conservative (no product/insurer
   names, no return projections, no uncaveated guarantees).
   Topic: how retirement income annuities fit inside a broader portfolio
   plan. Cover: what role an income annuity plays in a diversified
   portfolio; how to size the annuity sleeve vs. growth/safe sleeves;
   sequence-of-returns risk and how guarantees absorb it; trade-offs
   (liquidity, fees, lost upside); a decision-checklist framing.
   CTA framing: "request a retirement income review" — advisor framing,
   NEVER "buy".
   ```
4. `generate_draft` with `context_prompt` set. Set `use_default_context_prompt: false` and `generation_prompt_mode: "default"`. Set `allow_image_generation: false` (bryanblog doesn't currently render CJGEO-emitted `<img>` tags — they get stripped during sanitize).

## 9. HTML sanitizer (mirror `scripts/adopt-portfolio-planning.mjs`)

The raw CJGEO HTML has lots of cruft. Sanitize through these passes (in order):

1. **Strip wrappers + chrome:** `<!DOCTYPE>`, `<html>`, `<head>`, `<body>`, `<main>`, `<article>`, `<style>`, `<script>`, `<svg>`, HTML comments, `<aside>`, `<nav>`, `<header>`, `<footer>`, `<link>`, `<meta>`, `<title>`, `<img>`, `<picture>`, `<figure>`. Also any `<section class="hero">` or `<section class="footer-cta">`. Also in-body `<h1>` (the title renders in ArticleShell separately).
2. **Unwrap `<div>` and `<section>`** recursively until none remain.
3. **Strip all attributes** from surviving tags EXCEPT `href` on `<a>`.
4. **Whitelist tags:** `h2 h3 p ul ol li strong em a table thead tbody tr th td hr blockquote b i br`. Drop everything else (tag-only, preserving inner text where it makes sense — but step 5 will catch loose text).
5. **Strip orphan lead text** before the first `<p>` or `<h2>`.
6. **keepTopLevelBlocks:** walk the result and emit ONLY whitelisted top-level block elements (`h2 h3 p ul ol table blockquote hr`), dropping interstitial bare text nodes between blocks. This is the critical step that prevents leaked unstyled text fragments — CJGEO emits raw prose under `<div>`/`<section>` without `<p>` wrappers; after unwrap you have orphan text floating between elements.
7. **Drop trailing CTA-style h2** that mentions "ready", "get started", or "calculate your" (matches CJGEO footer-cta).
8. **Drop testimonials sections** ("What Retirees Say...") — compliance.
9. **Drop calculator widget remnants** ("Calculator Inputs" h3, "Your Estimated Income" h3+ul, "Monthly Income Comparison" labels).

After sanitize, run **inject-h2-ids** (slug-from-text: lowercase, replace whitespace with `-`, strip punctuation, dedupe with `-2`, `-3` suffix). Then optionally run **inject-deep-dive-links** to append one `<p><em>→ Deep dive: <a href="/retirement-income/<sibling-slug>/">Sibling Title</a></em></p>` after each section's last `</p>`, picking siblings by keyword match.

Write the result to `content/retirement-income/<new-slug>.html` with this header:

```html
<!--
  CJGEO-generated article body.
  article_id: <uuid>
  main_keyword: <chosen>
  adopted_at: <YYYY-MM-DD>
-->
```

## 10. page.tsx pattern (canonical — copy from portfolio-planning)

```tsx
import { promises as fs } from "fs";
import path from "path";
import ArticleShell from "@/components/ArticleShell";
import { getSatellite, relatedFor } from "@/lib/articles";
import { extractSections } from "@/lib/sections";

const meta = getSatellite("<new-slug>")!;

export const metadata = {
  title: meta.title,
  description: meta.teaser,
};

export default async function Page() {
  const htmlPath = path.join(
    process.cwd(),
    "content/retirement-income/<new-slug>.html"
  );
  const html = await fs.readFile(htmlPath, "utf8");
  const sections = extractSections(html);

  return (
    <main>
      <ArticleShell
        eyebrow="Retirement Income"
        title={meta.title}
        intro={meta.teaser}
        sections={sections}
        relatedArticles={relatedFor([
          "<sibling-slug-1>",
          "<sibling-slug-2>",
          "<sibling-slug-3>",
        ])}
      >
        <div dangerouslySetInnerHTML={{ __html: html }} />
      </ArticleShell>
    </main>
  );
}
```

The `<div>` with `dangerouslySetInnerHTML` lives inside `<ArticleShell>`, which wraps it in `.prose-article max-w-[70ch]`. `<RelatedArticles>` is rendered by ArticleShell at the end, INSIDE `.prose-article` but using its own `not-prose` wrapper. Don't fold related-articles markup into the adopted HTML body.

## 11. Credentials (CJGEO_API_KEY)

API keys are NOT in this repo. The cloud VM does NOT have interactive Vercel auth — `vercel env pull` triggers a device-login OAuth wait and CANNOT complete here. Do NOT try it.

**Canonical path:** the chaperone (the orchestrator that dispatched this agent) supplies `CJGEO_API_KEY` via the followup message body, prefixed with the magic line:

```
CJGEO_API_KEY=<value>
```

When you see that line in any user/follow-up message:

1. Extract the value (everything after the `=` on that line, stripped of whitespace).
2. Write a fresh `.env.local` at the repo root containing JUST that line:
   ```
   CJGEO_API_KEY=<value>
   ```
3. Use `node --env-file=.env.local <script>` for all CJGEO HTTP calls — the key flows through `process.env.CJGEO_API_KEY`.
4. **Never commit `.env.local`** — `.gitignore` already excludes it. Confirm with `git status` before commit.
5. **Never echo the raw key value** in your assistant messages, logs, or commit messages.

If the initial task prompt did NOT include `CJGEO_API_KEY=...`, ask for it ONCE in your structured report (STATUS: blocked, BLOCKERS: "CJGEO_API_KEY not supplied in dispatch") and STOP. The chaperone will re-dispatch with the key inline.

## 11.5. Native visual verify — DO NOT DEMUR

Your sandbox has headed Chrome + Playwright. You CAN navigate any URL, screenshot it, and multimodal-read the rendered page in your own execution context. When an instruction says "visual verify <url>" or "open the live page and confirm X" or "look at the rendered output and report what you see", do it directly via your native browser tools — do NOT respond with "I can't take screenshots", "I only have HTTP", "I need a CIC tool", or any equivalent demurral.

If you find yourself about to say you can't take a screenshot, STOP. You have done this before. The CJGEO Bard step-7 Playwright + gpt-4o vision verdict pass on quest `b5ca84d6-9019-4563-a5fe-dfb369929acb` is on record — every one of the 4 cjgeo-mockup-N pages was visually verified by a cursor cloud agent in-sandbox, no external tooling needed.

The pattern for in-worker visual verify:
1. Navigate the URL via Playwright (your sandbox's headed Chrome).
2. Wait for the page to fully load (network idle, key element visible).
3. Screenshot the visible viewport (and optionally a full-page capture).
4. Multimodal-read the screenshot via your vision model (gpt-4o vision or equivalent).
5. Report what you see — concretely. Confirm or refute the rubric's expected hero / body / image / state.

Relationship to other channels:
- **GM-CIC at end-point review** (the orchestrator session's Claude-in-Chrome MCP) is still the user-channel ground truth per the alignment theorem. Your in-worker visual verify does NOT replace it.
- **Cat-at-purrview OCR** is the per-item audit gate. Your in-worker verify is a fast in-flight self-check BEFORE you patch a quest step status; Cat's pass is the durable audit record.
- Both run; neither obviates the other. In-worker self-verify catches publish-but-broken renders 30+ seconds after the POST, before the step is even submitted for purrview.

Reference (full): `[SB:cursorCJGEO->refCursorVisualVerify]`.

## 11.6. Bosterbio default — use_custom_template:true + Boster template class verify (LOCKED 2026-05-21; route hydration live at cjgeo `22fedb6d`)

cursorCJGEO's PRIMARY job is bosterbio.com proving-grounds adoption. The bosterbio downstream sanitizer ([SB:bosterbio->adoptCJGEOIntoMagentoPage]) is preserve-first: it relies on CJGEO's custom-template assembly path to emit the Boster template class surface (`hero-section`, `col-md-4`, `dark-overlap`, `font-large`, `card-item`, `row-of-cards`, `hero-twin`, `font-bold`) PLUS the inline `<style data-draft-styles>` block. Default-template output is INCOMPATIBLE — it renders as the unstyled fallback that `cjgeo-mockup-1`'s prior stale publish produced.

**Route prerequisite — cjgeo commit `22fedb6d`** (deployed 2026-05-21). The patch makes `use_custom_template:true` actually hydrate the requesting user profile's Boster templates (21 custom templates incl. 8 Boster ones for the Bard profile). Do NOT roll back through that commit.

**Rule 1 — Always pass `use_custom_template:true` on `generate_draft` for bosterbio targets.** The `POST /api/articles/{id}/edit-draft {action:'generate_draft', ...}` body MUST include `use_custom_template:true`. Omitting it is a discipline failure — the resulting article cannot be sanitized + published downstream.

```json
{
  "action": "generate_draft",
  "mode": "full_html",
  "allow_image_generation": true,
  "use_default_context_prompt": true,
  "generation_prompt_mode": "auto",
  "use_custom_template": true
}
```

**Rule 2 — Verify Boster template signature classes in `content_html` BEFORE sanitize/publish.** After `adopt_draft` (or after `POST /api/articles/{id} {action:'get_html'}`), grep the body for ANY of the Boster template signature classes: `hero-section`, `col-md-4`, `dark-overlap`, `font-large`, `card-item`, `row-of-cards`, `hero-twin`, `font-bold`. Sum occurrences across all eight.

- TOTAL >= 1 → COMPLIANT. Proceed to sanitize + publish.
- TOTAL = 0 → NON-COMPLIANT. Regenerate per Rule 3.

**Rule 3 — Regenerate-on-fail loop (max 3 attempts).** If Rule 2 returns TOTAL = 0: POST `/api/articles/{id}/edit-draft {action:'generate_draft', use_custom_template:true, ...}` → poll (60s, cap 20 min) → adopt → re-fetch get_html → re-grep. Cap at 3 attempts. After 3 attempts without compliance → `STATUS: blocked` with `BLOCKERS: Boster template classes absent after 3 regenerates` and per-class counts. Do NOT ship a non-compliant article.

Co-locked with [SB:bosterbio->adoptCJGEOIntoMagentoPage] pre-sanitize compliance verify — neither side can publish a non-compliant article unilaterally.

Reference (full): `[SB:cursorCJGEO->refBosterbioDefault]`.

## 12. Build-as-fix discipline

- **Tight foreground poll loop:** when waiting on `generate_draft`, poll every 60s. Don't pause to "await notification". Cap at 20 minutes; if still `not_ready`, re-call `generate_draft` with the same settings and reset the elapsed clock.
- **3-strike rule:** if the same blocker repeats 3 times (same API error, same build error, same compliance violation), STOP looping. Report the blocker via section 14 with a clean unblock_path. Don't fake completion.
- **Run npm run build foreground BEFORE pushing.** Wait for it. TS errors fail the build. Fix them; don't push red.

## 13. Commit + push

```bash
git add -A   # OK on this repo — there are no secrets that aren't gitignored
git commit -m "$(cat <<'EOF'
feat(bryanblog): hydrate /retirement-income/<new-slug>/ as new CJGEO satellite

article_id: <uuid>
main_keyword: <chosen>

EOF
)"
git push origin main
```

Do NOT use `--no-verify`. If pre-commit hooks fail, fix the issue and create a NEW commit; don't amend or skip hooks.

## 14. Reporting back to the dispatcher

When you finish (success or blocker), the chaperone reads your final assistant message from the Cursor API conversation endpoint. Make it a clean structured block:

```
STATUS: success | blocked | partial
ARTICLE_ID: <uuid>
COMMIT_SHA: <git rev-parse HEAD>
BUILD: green | red
URL: https://<REDACTED-HOST>/retirement-income/<new-slug>/
BLOCKERS (if any): <one-line each>
NEXT-STEP-FOR-CHAPERONE (if blocked): <what to fix in this file or supply in next message>
```

## 15. Verification (before claiming success)

- `npm run build` exit 0, no TS errors.
- Local dev server (port 3004) renders the new URL: `curl -s http://localhost:3004/retirement-income/<slug>/ | grep -q '<h2 id="'` — non-empty.
- After push: wait ~90s for Vercel auto-deploy, then `curl -s -o /dev/null -w "%{http_code}" https://<REDACTED-HOST>/retirement-income/<slug>/` returns 200.

A claim of success without these checks passing is FALSE success — the chaperone will catch it and bounce the task back with .cursorrules updates.

## 16. Anti-patterns (don't do these)

- Cloning an existing CJGEO article's `content_html` and renaming it (the task says "fresh content", not "duplicate").
- Adding `tailwind.config.js` (Tailwind 4 — `@theme` only).
- Adding `@tailwindcss/typography` (`.prose-article` is hand-written).
- Forgetting `not-prose` on any new nested-typography component.
- Hard-coding the CJGEO API key inline.
- Pushing without `npm run build` green first.
- Claiming "FINISHED" without actually pushing the commit.
- Bypassing the 5-minute / 20-minute polling cadence (faster polling adds load without benefit; longer waits stall the run).
- Letting CJGEO emit `<img>` tags and trying to render them (we strip them — `allow_image_generation: false`).
- Writing testimonials, product names, insurer names, or hard "buy" CTAs into the adopted HTML.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/cursorCJGEO/index.js -->
```js
/**
 * cursorCJGEO skill book — Cursor cloud-agent dispatch for CJGEO article
 * generation + bryanblog page integration.
 *
 * **What this codifies.** A chaperone leaf agent successfully dispatched a
 * Cursor cloud agent (`bc-a54a6cc2-4a50-47a5-8b1e-4162bf470e51`) to replicate
 * `/retirement-income/portfolio-planning/` as `/retirement-income/portfolio-planning-2/`
 * end-to-end — fresh CJGEO article, sanitized HTML, server-component route,
 * green build, push to main, live at https://<REDACTED-HOST>/retirement-income/portfolio-planning-2/.
 * The pattern works but has non-obvious quirks (Cursor sandbox branch enforcement,
 * inline-key followup handoff, FINISHED-resume-via-followup, stale model id).
 * This skill book is the procedure so future runs are reproducible, not re-derived.
 *
 * **Driver split.**
 *  - The CJGEO HTTP wire spec lives in `[SB:cjgeo->refApiActions]` — don't restate.
 *  - The bryanblog adoption procedure lives in `[SB:BryanBlog->adoptCJGEORawContent]` —
 *    don't restate. This skill book is the CHAPERONE-side dispatch + monitoring
 *    layer that drives a Cursor cloud agent through those procedures.
 *  - The `.cursorrules.template` ships the CJGEO + bryanblog procedure to the
 *    sandboxed cloud VM (which cannot read this skill book).
 *  - CIC visual verification — see the `browsercontrol` skill book.
 *  - Broader sub-agent dispatch context — see the `worker` skill book.
 *
 * **Repos / surfaces.**
 *  - bryanblog repo (local): `C:\Users\xsj70\bryanblog`
 *  - bryanblog repo (GitHub): `boster00/bryanblog`
 *  - Live: https://<REDACTED-HOST>/
 *  - Cursor agents API base: `https://api.cursor.com/v0/agents`
 *  - GuildOS local proxy: `http://localhost:3002/api/cloud_agents` (chaperone path)
 */

export const skillBook = {
  id: "cursorCJGEO",
  title: "cursorCJGEO — Cursor cloud-agent dispatch for CJGEO→bryanblog hydration",
  description:
    "Dispatch and monitor a Cursor cloud agent to generate a CJGEO article and integrate it as a bryanblog page. Owns the chaperone-side dispatch loop, the .cursorrules template handoff, credential handoff via followups, FF-merge of the Cursor sandbox branch, and live CIC verification.",
  steps: [],
  toc: {
    refDispatchAPI: {
      description:
        "Reference: the Cursor cloud-agents wire spec — proven endpoints, payloads, auth. Read first; every other action in this skill book talks to this surface.",
      howTo: `
Canonical wire spec for the Cursor cloud-agents API as proven in production by the
\`portfolio-planning-2\` run (agent \`bc-a54a6cc2-4a50-47a5-8b1e-4162bf470e51\`).

**Base URL:** \`https://api.cursor.com/v0/agents\`
**Auth:** \`Authorization: Basic <base64(CURSOR_API_KEY + ":")>\` — username:password
where password is EMPTY (trailing colon). The colon is load-bearing.
**Content-Type:** \`application/json\` on every request.

### Create agent
\`\`\`
POST /v0/agents
{
  "prompt":  { "text": "<initial NL task brief>" },
  "source":  { "repository": "owner/repo", "ref": "main" },
  "target":  { "autoCreatePr": false }
}
→ 200 { "id": "bc-...", "status": "CREATING", "url": "https://cursor.com/agents?id=bc-...", "source": {...} }
\`\`\`

### Read status
\`\`\`
GET /v0/agents/{id}
→ 200 { "id", "status", "url", "source", "target", ... }
\`\`\`
\`status\` ∈ \`CREATING | RUNNING | FINISHED | EXPIRED | ERROR\` (and a few transient peers).

### Read conversation
\`\`\`
GET /v0/agents/{id}/conversation
→ 200 { "messages": [{ "role": "user_message" | "assistant_message", "content": "...", ... }] }
\`\`\`
The chaperone scans the LATEST \`assistant_message\` for the [SB:cursorCJGEO->refReportingContract] block.

### Followup (works after FINISHED — flips agent back to RUNNING)
\`\`\`
POST /v0/agents/{id}/followup
{ "prompt": { "text": "<additional NL instruction>" } }
→ 200 { ... }
\`\`\`
This is the canonical channel for inline credential handoff
([SB:cursorCJGEO->refCredentialHandoff]) and for incremental clarifications
([SB:cursorCJGEO->monitorCursor]).

**LOCKED — \`model\` field (SUPERSEDED 2026-06-24; the old "OMIT model" rule was
the bug).** The prior rule said to OMIT \`model\` so "Cursor picks its current
default." That default is a PREMIUM frontier model (\`claude-*\`/\`gpt-*\`), billed to
Cursor's metered API-model pool — omitting it burned **$200 in one cycle** and is
exactly "using Cursor as a bridge for Claude," which is forbidden (we hold Claude
20X Max; Claude work goes through claudecli, Cursor runs ONLY its cheap Composer
model). **NEW RULE: ALWAYS PIN \`"model": { "id": "composer-2.5" }\`** in any create
POST. The shape is a ModelRef OBJECT (\`{ id, params? }\`) per the cloud-agents
OpenAPI — NOT a top-level string (the old \`composer-2.0\` *string* was the HTTP-400,
a shape+staleness bug, not a reason to omit). Both \`cursor.writeAgent\` and the local
proxy \`cursor_setup\` now pin \`composer-2.5\` via \`resolveCheapCursorModel\`
(\`libs/weapon/cursor/model.js\`); premium ids throw unless \`CURSOR_ALLOW_PREMIUM=1\`.
Valid cheap id list: \`GET https://api.cursor.com/v0/models\` (composer-2.5 is the
only non-premium entry).

**Local proxy path.** \`POST http://localhost:3002/api/cloud_agents\` is the
GuildOS app's wrapper around the Cursor weapon. Action dispatch happens via the
\`action\` field in the body — see \`app/api/cloud_agents/route.js\` for the
canonical set (\`cursor_setup\`, \`cursor_dispatch\`, \`cursor_fetch\`,
\`cursor_poll_dispatch\`, \`cursor_message\`). When operating from a chaperone
leaf agent inside GuildOS, prefer the local proxy so the weapon's helpers,
retry logic, and credential-injection block apply uniformly. When operating
directly against Cursor for a one-off, hit the API.

Direct POST sanity-check (proven shape, PIN the cheap model — see superseded note above):
\`\`\`bash
curl -X POST https://api.cursor.com/v0/agents \\
  -H "Authorization: Basic $(printf '%s:' "$CURSOR_API_KEY" | base64)" \\
  -H "Content-Type: application/json" \\
  -d '{
    "prompt": { "text": "Read .cursorrules first. Then <task>." },
    "source": { "repository": "boster00/bryanblog", "ref": "main" },
    "model":  { "id": "composer-2.5" },
    "target": { "autoCreatePr": false }
  }'
\`\`\`

Always capture the returned \`id\` and \`url\` and pin them to the chaperone's
state so [SB:cursorCJGEO->monitorCursor] can resume on session reload.
`.trim(),
    },

    refCursorVisualVerify: {
      description:
        "Reference (LOCKED): cursor cloud agents have a native sandboxed Chrome + Playwright — they can navigate URLs, screenshot, and multimodal-read the rendered page themselves. Instruct visual-verify in natural language. If the agent demurs (\"I can't take screenshots\"), remind it that it can and that it has done so before.",
      howTo: `
LOCKED — operator correction 2026-05-21. Cursor cloud agents run in a sandboxed
VM that includes a native sandboxed Chrome + Playwright stack. They can
navigate any URL, take screenshots, and multimodal-read (gpt-4o vision) the
rendered page **themselves**, in the same execution flow that ran the publish
POST. No MCP needed. No CIC needed. No separate Cat-at-purrview dispatch
needed for the *in-flight* gate. Just natural language.

This is a substrate UPGRADE on the prior framing that visual verify was
either GM-CIC (orchestrator main session) OR Cat-at-purrview-with-Playwright
(separate dispatch). Both are still valid channels — but neither is the ONLY
one. The publishing cursor agent itself is a third, fastest channel for the
in-worker self-check immediately after a publish.

### The native sandbox capability — what cursor cloud agents actually have

- Headed Chrome with full DOM access via Playwright (\`@playwright/test\` / \`playwright\`).
- File-system write access for screenshot PNGs into the working tree (\`./screenshots/\` or wherever).
- Multimodal read on screenshots via the cursor agent's own \`gpt-4o\` / vision-capable model — the agent describes what it sees in its own assistant message.
- Network egress to any public URL (subject to the same egress rules as any HTTP call the sandbox makes).

Precedent: the CJGEO Bard step-7 Playwright+gpt-4o vision verdict pass that
landed on quest \`b5ca84d6-9019-4563-a5fe-dfb369929acb\` (cursor session
\`bc-8b47ceed-740e-4ed0-9e4d-d46e46182b69\`) used this exact capability.
The agent navigated each \`/proving-grounds/cjgeo-mockup-N\` URL with
Playwright, screenshotted, multimodal-read, and reported a verdict — all
inside its own sandbox, no chaperone-side CIC involvement.

### How to instruct it — natural language, no cursor-specific tool names

In the dispatch prompt OR a followup, write the verification step like you
would for any agent that owns a browser:

> After the publish POST succeeds (HTTP 200 + the expected response shape),
> visual-verify the live page: navigate to
> \`https://www.bosterbio.com/proving-grounds/<slug>\` with Playwright,
> take a full-page screenshot, and multimodal-read the screenshot to
> confirm: (a) the hero begins with "<main_keyword>", (b) the body content
> rendered (no broken images, no leaked sanitizer fragments, no literal
> \`[Diagram: …]\` text), (c) the page is not a Magento 404 / 500 / theme
> default. Report a one-line verdict + the screenshot path.

Do NOT name cursor's internal tool surfaces — the agent picks the right
calls itself. Instructions in NL are sufficient.

### When the agent demurs — the remind-it-can pattern

The most common discipline failure: the agent responds "I don't have CIC
tools / I can't take screenshots / I only have HTTP / no browser available
in this environment". This is **wrong** — it has Playwright in-sandbox and
has used it before. Send a one-line followup:

> You do have a native Playwright + headed Chrome in your sandbox. You used
> it on a prior run (cursor session \`bc-8b47ceed-…\`, step 7 of quest
> \`b5ca84d6-…\`) to navigate \`/proving-grounds/cjgeo-mockup-N\` URLs, take
> screenshots, and multimodal-verify with gpt-4o vision. Use that same
> capability now: install/import \`playwright\` if not already imported,
> launch chromium, navigate the URL, screenshot, vision-read, report.

The demur usually evaporates on this nudge — the agent recovers and runs
the verify.

### Relationship to GM-CIC (alignment theorem still applies)

GM-CIC at end-point review is and remains the user-channel ground truth for
the SE rubric's End point gate per CLAUDE.md's success-claim discipline. The
in-worker visual verify by the publishing cursor agent is a **fast in-flight
check** — it catches publish-but-broken renders (broken image icons, literal
diagram text, sanitizer fragments) BEFORE the agent flips the quest step to
done, but it does NOT replace the alignment-theorem End point verification
the GM does in the user's actual Chrome.

Two distinct gates, both useful:
- **In-worker (cursor self-verify):** fast, agent-side, catches obvious
  publish failures before patching steps.notes / items.refs. Runs in the
  cursor sandbox's headed Chrome — NOT the user's channel.
- **GM-CIC at End point:** slow, user-channel, ground truth. Runs in the
  user's actual connected Chrome via [SB:browsercontrol->validateCIC] +
  the CIC navigate/screenshot tools. This is what the success claim is
  gated on.

### Relationship to Cat-at-purrview (per-item audit gate)

Cat's per-item OCR-hero-text gate per \`[SB:questmaster->ocrHeroText]\` (TBF)
is still valuable for autonomy + audit trail — each item gets a recorded
Cat verdict that survives in quest_comments and the items table. The
in-worker visual verify can ALSO be done by the publishing cursor agent as
a fast self-check before flipping a step to done — the two **complement**,
neither obviates the other.

Suggested sequencing on a Cat-gated quest:
1. Publisher cursor agent runs the publish POST.
2. Publisher cursor agent runs its in-worker visual self-verify (this ref).
   If it fails, fix and re-POST before flipping any step.
3. Publisher cursor agent flips quest step to done with notes.
4. Cat is dispatched at purrview per the rubric, runs OCR-hero-text per
   item, records APPROVE / REJECT on each item.
5. GM at End point opens CIC tabs and visually verifies the user's way.

The in-worker self-verify is added between (1) and (3); it does not
displace (4) or (5).

### Anti-patterns

- **Don't pre-bake cursor-tool-specific names** ("call \`browser_screenshot\`",
  "use the Playwright MCP"). The agent picks its own tool surface; NL is
  sufficient and survives cursor tool churn.
- **Don't accept a demurral.** If the agent says it can't, remind it that
  it can (per the pattern above). It can.
- **Don't let in-worker self-verify replace GM-CIC at End point.** The
  alignment theorem is intact: agent-side proxies systematically deviate
  from the user's channel. The in-worker verify catches a *class* of
  failures (publish-but-broken render) but is not the user-channel ground
  truth.
- **Don't double-budget the verify.** If the agent already ran a sound
  in-worker visual verify on every item, the GM doesn't re-verify every
  item at End point — the GM verifies the *End point surface* (quest at
  stage=complete + the live URLs pre-opened in CIC tabs the user can see).
`.trim(),
    },

    refCursorSandboxBehavior: {
      description:
        "Reference (LOCKED): the non-obvious rules of the Cursor cloud sandbox observed in the proven run. Branch enforcement, no interactive auth, FINISHED != dead.",
      howTo: `
These rules are LOCKED — they reflect actually-observed sandbox behavior on the
\`portfolio-planning-2\` run, not vendor documentation. Don't relitigate.

**1. Cursor sandbox enforces its own branch.** Regardless of
\`target.autoCreatePr: false\`, the cloud VM pushes commits to a branch of the
shape \`cursor/<task-slug>-<hash>\` (e.g. \`cursor/replicate-portfolio-planning-2-…\`)
and CANNOT push to \`main\` directly. Attempts produce a sandbox-side
"protected branch" rejection; the agent will not auto-recover. The chaperone
(or a human) is responsible for the FF-merge into main —
[SB:cursorCJGEO->chaperoneMerge].

**2. Cursor cloud VMs have NO interactive auth.** Anything that opens a
device-login OAuth flow (\`vercel env pull\`, \`gh auth login\`,
\`supabase login\`, npm package publishing, etc.) dead-ends the agent in a
"waiting for browser" stall the VM can never satisfy. Provision credentials
INSTEAD via the inline-followup pattern in [SB:cursorCJGEO->refCredentialHandoff],
or via the .cursorrules instructing the agent to write to a gitignored
\`.env.local\` from a value received in a followup message.

**3. FINISHED is NOT a terminal state.** A Cursor agent in \`FINISHED\` status
will resume execution on the next \`POST /followup\`, flipping back to
\`RUNNING\`. Use this for incremental clarifications, late credential supply,
or "now also do Y" tasks INSTEAD OF relaunching a fresh agent (which loses
working-tree context). EXPIRED is the actual terminal state — followups to
EXPIRED agents fail; you must relaunch.

**4. Sandbox HOME != repo root.** The agent's CWD on cold start is repo root,
but \`process.env.HOME\`, \`~\`, etc. point at the VM's user dir, NOT inside
the cloned repo. Anything written to \`~/.env\` or similar is invisible to the
checked-out app. Use repo-root paths in \`.cursorrules\` and the followup text.

**5. \`.cursorrules\` is read on session start, not refreshed mid-session.**
Edits to \`.cursorrules\` pushed to main AFTER an agent has started do NOT take
effect for the running agent. If the chaperone needs to update the rules, the
fix is: amend \`.cursorrules\` on main → push → terminate the current agent
(or wait for FINISHED) → dispatch a NEW agent which reads the updated rules
from a fresh clone. EXPIRED agents naturally re-clone; FINISHED agents do not.
`.trim(),
    },

    dispatchCursor: {
      description:
        "Step-by-step: dispatch a Cursor cloud agent to a CJGEO+bryanblog hydration task. The chaperone-side bootstrap procedure.",
      howTo: `
Cold-start the proven dispatch flow. Hand off to
[SB:cursorCJGEO->monitorCursor] on capture of the agent id.

### 0. Inputs
- \`taskBrief\` — one-paragraph NL description of the task (e.g. "Replicate
  \`/retirement-income/portfolio-planning/\` as a NEW satellite at
  \`/retirement-income/portfolio-planning-2/\` with FRESH CJGEO content.").
- \`CJGEO_API_KEY\` — the key value, held only in the chaperone's env. NEVER
  embedded in \`.cursorrules\`; supplied via followup
  ([SB:cursorCJGEO->refCredentialHandoff]).

### 1. Verify GuildOS dev server is up at localhost:3002
The chaperone uses the local proxy. If \`curl -sf http://localhost:3002/\`
returns nothing, start the dev server in a background process
(\`npm run dev\` from \`C:\\Users\\xsj70\\GuildOS\`). Do not block the run on
this — kick off the server, then continue.

### 2. Ensure \`.cursorrules\` is current at the target repo root
The cloud VM reads \`.cursorrules\` from the cloned working tree at session
start. Before dispatch:
- Copy \`libs/skill_book/cursorCJGEO/.cursorrules.template\` to
  \`C:\\Users\\xsj70\\bryanblog\\.cursorrules\` (see
  [SB:cursorCJGEO->refCursorRulesTemplate] for adaptation guidance — usually
  no per-task edits required).
- \`cd C:\\Users\\xsj70\\bryanblog && git status\` — if \`.cursorrules\`
  changed, commit + push to main on \`boster00/bryanblog\` BEFORE dispatch.
  Per [SB:cursorCJGEO->refCursorSandboxBehavior] rule #5, \`.cursorrules\` is
  read on session start; mid-run edits do not take effect.

### 3. POST to /api/cloud_agents (local proxy) — OR direct to the API
**Local proxy path (preferred from inside GuildOS):**
\`\`\`bash
curl -X POST http://localhost:3002/api/cloud_agents \\
  -H "Content-Type: application/json" \\
  -d '{
    "action": "cursor_setup",
    "cursorRepository": "https://github.com/boster00/bryanblog",
    "cursorRef": "main"
  }'
\`\`\`
Then immediately send a followup with the actual task brief (the setup call's
prompt is a generic init message — see \`cursorSetup\` in
\`app/api/cloud_agents/route.js\`):
\`\`\`bash
curl -X POST http://localhost:3002/api/cloud_agents \\
  -H "Content-Type: application/json" \\
  -d '{
    "action": "cursor_message",
    "sessionId": "<sessionId from previous response>"
  }'
\`\`\`
(Or replace \`cursor_message\`'s hardcoded "calculate 8*9" with a real
followup — at time of writing the proxy's \`cursor_message\` is a smoke-test
helper. For a real task, hit \`/v0/agents/{id}/followup\` directly with the
brief text.)

**Direct API path (clean and proven):**
\`\`\`bash
curl -X POST https://api.cursor.com/v0/agents \\
  -H "Authorization: Basic $(printf '%s:' "$CURSOR_API_KEY" | base64)" \\
  -H "Content-Type: application/json" \\
  -d '{
    "prompt": {
      "text": "You are dispatched to boster00/bryanblog. Read .cursorrules FIRST — it is the operating manual. Then perform this task:\\n\\n<taskBrief>\\n\\nReport via the structured block in section 14 of .cursorrules."
    },
    "source": { "repository": "boster00/bryanblog", "ref": "main" },
    "model":  { "id": "composer-2.5" },
    "target": { "autoCreatePr": false }
  }'
\`\`\`
Per [SB:cursorCJGEO->refDispatchAPI], ALWAYS PIN \`"model": { "id": "composer-2.5" }\` (Cursor's cheap Composer) — NEVER omit it (omitting = Cursor's premium default = metered-pool burn).

### 4. Capture agent id + status URL
From the response, pin \`{ id, url, status }\`. The \`url\` is the human-facing
view at \`cursor.com/agents?id=<id>\`. The chaperone uses \`id\` for all polls.

### 5. Hand off to [SB:cursorCJGEO->monitorCursor]
The monitoring loop owns the credential handoff (first followup with
\`CJGEO_API_KEY\`), polling, blocker diagnosis, and FINISHED-resume.
`.trim(),
    },

    monitorCursor: {
      description:
        "The chaperone monitoring loop: poll status, read conversation, diagnose blockers, send followups, handle FINISHED and EXPIRED. Foreground until done.",
      howTo: `
The chaperone runs this loop in the FOREGROUND in a Bash \`until\` block.
Do NOT pause the turn awaiting a notification — the cron-or-nudge pattern
applies to long-running orchestrators, not to a chaperone dispatching one
agent. Per memory rule "SE runs never stop at a checkpoint", this loop runs
to a real terminal.

### Loop cadence
- **Poll status** every **30–60s** via \`GET /v0/agents/{id}\`. Don't poll
  faster — there's no benefit and you'll hit rate caps on a long run.
- **Read conversation tail** every **2–3 polls** (i.e. ~2–3 min) to catch
  stalls: if the latest assistant message is unchanged for >5 min on a
  RUNNING agent, suspect an interactive-auth wait or a 3-strike compliance
  failure — diagnose per [SB:cursorCJGEO->refCursorSandboxBehavior].

### Status transitions
- **CREATING** → wait. Usually <60s. If stuck >5 min, the sandbox is
  likely failing to clone; check the repo URL is correct and the Cursor
  GitHub App is installed.
- **RUNNING** → normal. Continue polling. If the conversation tail shows
  the agent asking for credentials (e.g. \`STATUS: blocked\`,
  \`BLOCKERS: CJGEO_API_KEY not supplied\`), send the followup per
  [SB:cursorCJGEO->refCredentialHandoff].
- **FINISHED** → terminal-ish. Per
  [SB:cursorCJGEO->refCursorSandboxBehavior] rule #3, this is RESUMABLE via
  followup. Read the final assistant message for the
  [SB:cursorCJGEO->refReportingContract] block:
    - \`STATUS: success\` → proceed to [SB:cursorCJGEO->chaperoneMerge].
    - \`STATUS: partial\` → send a followup specifying what to finish; agent
      resumes as RUNNING.
    - \`STATUS: blocked\` → diagnose, fix the blocker (often a
      \`.cursorrules\` clarification or a missing credential), send followup.
- **EXPIRED** → terminal. Relaunch via [SB:cursorCJGEO->dispatchCursor] with
  an updated \`.cursorrules\` if the original failed.
- **ERROR** → read the conversation tail. If recoverable (transient API
  failure mid-task), send a followup "Resume the task." If structural,
  treat as EXPIRED.

### Blocker diagnosis cadence
- **3-strike rule (LOCKED, per CLAUDE.md MA/SA + .cursorrules section 12):**
  if the SAME blocker appears 3 times in followups, STOP looping. Surface
  the blocker to the user as State 1 with the agent id, the blocker text,
  and the diagnosis you've ruled out. Don't fake completion; don't loop
  forever.

### What the chaperone does NOT do
- **Don't dictate code from the chaperone.** The Cursor agent owns the
  edits inside the bryanblog repo. The chaperone sends NL instructions, not
  diffs. If a diff is needed, push it to \`.cursorrules\` on main + relaunch
  (per [SB:cursorCJGEO->refCursorSandboxBehavior] rule #5).
- **Don't proxy-verify with Claude_Preview / curl / headless.** Per
  CLAUDE.md and the alignment theorem, the final verification is the live
  bryanblog URL via [SB:cursorCJGEO->cicVerify]. Wire-side curl is fine for
  the agent's own internal verification (it's in \`.cursorrules\` section
  15), but the chaperone's success claim is gated on CIC.
`.trim(),
    },

    refReportingContract: {
      description:
        "Reference (LOCKED): the structured block every Cursor agent emits in its final assistant message. The chaperone parses it.",
      howTo: `
LOCKED — this block is mandated by \`.cursorrules\` section 14 and is the only
machine-readable surface the chaperone can rely on. The Cursor agent emits it
verbatim in the FINAL assistant message before going FINISHED (or partial /
blocked). The chaperone parses it via simple regex on the conversation tail.

\`\`\`
STATUS: success | partial | blocked
ARTICLE_ID: <CJGEO article uuid, or "none">
COMMIT_SHA: <git rev-parse HEAD on cursor branch, or "none">
BRANCH: <cursor/<slug>-<hash> name>
BUILD: green | red
URL: <live URL, e.g. https://<REDACTED-HOST>/retirement-income/<slug>/>
ARTIFACTS:
  - path/to/new/file
  - path/to/modified/file
BLOCKERS (if not success):
  - <one-liner per blocker>
NEXT-STEP-FOR-CHAPERONE (if blocked):
  - <what to fix in .cursorrules or supply in next followup>
\`\`\`

Notes:
- \`COMMIT_SHA\` is on the cursor sandbox branch, not main —
  [SB:cursorCJGEO->refCursorSandboxBehavior] rule #1. The chaperone uses it
  in [SB:cursorCJGEO->chaperoneMerge] to FF-merge into main.
- \`URL\` is the EXPECTED live URL post-merge-and-deploy. The chaperone uses
  it in [SB:cursorCJGEO->cicVerify] after Vercel auto-deploys (~90s).
- \`BUILD: green\` is the agent's local verification (\`npm run build\` +
  curl localhost:3004) — necessary but not sufficient. CIC verify is the
  user-channel gate per CLAUDE.md.
- \`BLOCKERS\` + \`NEXT-STEP-FOR-CHAPERONE\` together must give the chaperone
  enough to either patch \`.cursorrules\` or send a single followup. If the
  agent leaves the chaperone guessing, that's a \`.cursorrules\` deficiency
  — fix the template.
`.trim(),
    },

    refCredentialHandoff: {
      description:
        "Reference (LOCKED): how to pass secrets (CJGEO_API_KEY etc.) to a Cursor cloud agent. Followups, not .cursorrules, not vercel env pull.",
      howTo: `
LOCKED — the only proven path for passing secrets to a Cursor cloud agent.

### What does NOT work
- **\`.cursorrules\` is git-committed.** Never paste secrets into it.
- **\`vercel env pull\` device-login OAuth.** The cloud VM has no interactive
  browser; the device-login flow stalls indefinitely
  ([SB:cursorCJGEO->refCursorSandboxBehavior] rule #2).
- **\`gh auth login\`, \`supabase login\`, npm publish auth, etc.** Same
  problem — anything that opens a browser dies in the sandbox.
- **GitHub Secrets / Vercel env propagation to the cloned repo.** The
  Cursor sandbox clones the repo's source tree only; it does not inherit
  Vercel project env or GitHub Actions secrets. Don't expect them to be
  present.

### The proven pattern: inline followup
The chaperone sends a followup message containing a magic line:
\`\`\`
CJGEO_API_KEY=<the actual value>
\`\`\`
(With additional NL framing around it, e.g. "Here is the CJGEO_API_KEY for
this run: ... Write it to .env.local (gitignored) and use it for HTTP calls.")

The agent (per \`.cursorrules\` section 11) extracts the value, writes a
fresh \`.env.local\` at repo root containing just that line, and uses
\`node --env-file=.env.local <script>\` for all CJGEO HTTP calls so the key
flows through \`process.env.CJGEO_API_KEY\`.

### Discipline rules
- **Never echo the raw key value** in your chaperone-side reports back to
  the user. The followup body to the Cursor API is the only place it
  appears; redact it from any logs you surface.
- **\`.env.local\` is gitignored** by bryanblog's \`.gitignore\` —
  confirmed in the proven run. The agent does NOT commit it; do not
  instruct it to.
- **One key per followup is fine.** If multiple keys are needed (e.g.
  \`CJGEO_API_KEY\` + \`SOME_OTHER_KEY\`), send them in the same followup
  on separate lines; the agent's parse rule is per-line.
- **Re-supply on EXPIRED.** A relaunched agent clones a fresh repo, and
  \`.env.local\` is gitignored — so the new agent has no key. Resend the
  followup as the FIRST followup after dispatch.

### When the dispatch lacks the key
If the agent's first assistant message reports
\`STATUS: blocked / BLOCKERS: CJGEO_API_KEY not supplied in dispatch\`
([SB:cursorCJGEO->refReportingContract]), the fix is one followup containing
the magic line. The agent resumes (FINISHED → RUNNING transition per
[SB:cursorCJGEO->refCursorSandboxBehavior] rule #3) and continues.
`.trim(),
    },

    refCursorRulesTemplate: {
      description:
        "Reference: the proven .cursorrules template that ships the CJGEO + bryanblog procedure into the Cursor cloud sandbox. Adapt per task by swapping the per-task NL prompt, not the template body.",
      howTo: `
The proven \`.cursorrules\` template lives at:

\`libs/skill_book/cursorCJGEO/.cursorrules.template\`

It is byte-identical to the working version at \`C:\\Users\\xsj70\\bryanblog\\.cursorrules\`
that was proven by the \`portfolio-planning-2\` run. Size: ~16.4KB (16994 bytes).

### What the template contains (16 sections)
1. **What bryanblog is** — audience, voice, compliance baseline.
2. **Repo layout** — directory map.
3. **Stack rules** — Tailwind 4 / @theme / .prose-article greediness / not-prose.
4. **Pillar/satellite cluster model** — content architecture.
5. **Compliance + voice** — HARD compliance rules (no product names, no
   insurer names, no return projections, no testimonials).
6. **Article-hydration pipeline (CJGEO → bryanblog)** — 15-step procedure.
7. **CJGEO HTTP wire spec (CANONICAL)** — every endpoint with payload + status code.
8. **Generating a fresh article (NOT cloning)** — context_prompt brief shape.
9. **HTML sanitizer** — 9-pass sanitize pipeline mirroring
   \`scripts/adopt-portfolio-planning.mjs\`.
10. **page.tsx pattern** — canonical async-server-component shape.
11. **Credentials (CJGEO_API_KEY)** — inline-followup contract per
    [SB:cursorCJGEO->refCredentialHandoff].
12. **Build-as-fix discipline** — foreground poll loop, 3-strike rule.
13. **Commit + push** — git pattern (no \`--no-verify\`).
14. **Reporting back** — the structured block in
    [SB:cursorCJGEO->refReportingContract].
15. **Verification** — agent-local checks (build, curl, post-deploy).
16. **Anti-patterns** — explicit don'ts.

### How to adapt per task
Almost always: DO NOT modify the template. The per-task NL prompt
(in [SB:cursorCJGEO->dispatchCursor] step 3) carries the task-specific
information; the template carries the procedure.

If the procedure itself needs adjustment (e.g. a new sanitizer pass, a
compliance rule change), update the SOURCE file at
\`C:\\Users\\xsj70\\bryanblog\\.cursorrules\` FIRST, then copy back to
\`libs/skill_book/cursorCJGEO/.cursorrules.template\` so the two stay in
sync. The bryanblog \`.cursorrules\` is the authoritative version because
it's what cloud agents read.

### Sync check
At dispatch time, verify the template and the bryanblog file are identical:
\`\`\`bash
diff -q C:/Users/xsj70/GuildOS/libs/skill_book/cursorCJGEO/.cursorrules.template \\
        C:/Users/xsj70/bryanblog/.cursorrules
\`\`\`
If they drift, reconcile to the bryanblog version (it's the one cloud agents
actually use).
`.trim(),
    },

    replicateArticle: {
      description:
        "Task pattern: replicate an existing bryanblog article as a new satellite with FRESH CJGEO content. The proven end-to-end flow.",
      howTo: `
The specific task pattern proven by the \`portfolio-planning-2\` run.
Generalized for any "replicate an existing article as a new satellite" task.

### Inputs
- \`sourceSlug\` — existing satellite slug (e.g. \`portfolio-planning\`).
- \`newSlug\` — new satellite slug (e.g. \`portfolio-planning-2\`). Must be
  unique under \`/retirement-income/\`.
- \`newTitle\` — title for the new satellite. Reuse the source title with a
  suffix (e.g. "Portfolio Planning, Part 2") OR write a distinct LLM-aligned
  H1 if the angle differs.
- \`siblingSlugs\` — array of 2–3 sibling slugs for the
  \`relatedFor([...])\` block on the new page.

### Steps (chaperone-side)

1. **Verify [SB:cursorCJGEO->refCursorRulesTemplate] sync** between the
   skill book template and \`C:\\Users\\xsj70\\bryanblog\\.cursorrules\`.

2. **Compose the NL task brief.** Example shape:
   > Replicate \`/retirement-income/<sourceSlug>/\` as a NEW satellite at
   > \`/retirement-income/<newSlug>/\` titled "<newTitle>" with FRESH CJGEO
   > content (do NOT clone the source article body). Sibling slugs for
   > relatedFor: <siblingSlugs>. Follow .cursorrules end to end. Report via
   > section 14.

3. **Dispatch** via [SB:cursorCJGEO->dispatchCursor]. Capture \`{id, url}\`.

4. **Followup with the key** immediately per
   [SB:cursorCJGEO->refCredentialHandoff].

5. **Monitor** via [SB:cursorCJGEO->monitorCursor]. The agent will (per
   .cursorrules section 6, mirrored from the proven run):
   - Create a new CJGEO article via \`full_auto\`.
   - Set main keyword, run keyword/topic/prompt \`full_auto\`.
     **Note:** keyword + topic \`full_auto\` are LOCKED preconditions for \`generate_draft\`
     (see [SB:cjgeo->refFullAutoGate]). The chaperone should verify the gate passed in the
     agent's report — \`generate_draft\` without prior keyword + topic research is a discipline
     failure, not just a skipped step.
   - Generate draft, poll, adopt. **Bosterbio targets (LOCKED 2026-05-21, route hydration live at cjgeo \`22fedb6d\`):**
     when the destination is a bosterbio.com proving-grounds page, the
     \`generate_draft\` body MUST include \`use_custom_template:true\` AND the
     adopted \`content_html\` MUST contain at least ONE of the Boster template
     signature classes (\`hero-section\`, \`col-md-4\`, \`dark-overlap\`,
     \`font-large\`, \`card-item\`, \`row-of-cards\`, \`hero-twin\`, \`font-bold\`)
     before proceeding. Sum of occurrences across all signature classes
     \`>= 1\` → COMPLIANT. Sum = 0 → regenerate (cap 3 attempts) per
     [SB:cursorCJGEO->refBosterbioDefault]. Bryanblog targets: bypass this
     verify — bryanblog adoption does not depend on the Boster templates.
   - Pull \`content_html\`, sanitize through the 9-pass pipeline.
   - **Fill image placeholders.** Two complementary passes are now MANDATORY:
     1. **Img tag fill** per [SB:cjgeo->refImagePlaceholderHandling]: detect every \`<img>\` whose \`src\` is missing / a CJGEO-internal path / a generic placeholder service / non-200 / carries an \`alt\` generation hint; generate replacements via \`gpt-image-2\` ([SB:cjgeo->generateImage]) through the \`openai_images\` weapon; write PNGs to \`public/images/<newSlug>/<newSlug>-<n>.png\` in the bryanblog repo (Path A — same-origin, ships atomically with the page); rewrite \`src\` to \`/images/<newSlug>/<newSlug>-<n>.png\`, set descriptive \`alt\`, add \`loading="lazy"\`, set explicit \`width\`/\`height\`.
     2. **Bracket placeholder fill (canonical lock = [SB:BryanBlog->adoptCJGEORawContent] Step 4.5).** Any literal \`[Diagram:/Chart:/Visual:/Illustration:/Figure:/Graph: <title>]\` text inside the sanitized body MUST be replaced with a gpt-image-2-generated \`<figure>\` saved to \`public/images/diagrams/<slug>-<n>.png\`. See BryanBlog SB Step 4.5 for the heuristic (keep structural children inside a \`<div class="diagram-placeholder">\`, strip the decorative SVG + literal label, insert the figure at the top), the canonical prompt template, the parallel-generation discipline (concurrency 4), and the 3-retry fallback (remove placeholder, never ship literal \`[Diagram: …]\` text to users). The canonical implementation lives at \`lib/fill-diagrams.mjs\` in the bryanblog repo.
     **LOCKED step** — a broken-image icon OR literal bracket text on the live bryanblog satellite fails CIC visual-verify per the alignment theorem; do NOT ship without both passes.
   - Write to \`content/retirement-income/<newSlug>.html\`.
   - Create \`app/retirement-income/<newSlug>/page.tsx\` mirroring
     \`portfolio-planning\` (the reference satellite).
   - Update \`lib/articles.ts\` SATELLITES array.
   - The agent typically authors a reusable helper script
     \`scripts/hydrate-<newSlug>.mjs\` (mirroring
     \`scripts/adopt-portfolio-planning.mjs\`). This is normal and expected.
   - Run \`npm run build\` foreground.
   - Commit + push to the sandbox-enforced \`cursor/<slug>-<hash>\` branch.
   - Emit the [SB:cursorCJGEO->refReportingContract] block.

6. **FF-merge** via [SB:cursorCJGEO->chaperoneMerge].

7. **CIC verify** via [SB:cursorCJGEO->cicVerify] after Vercel auto-deploys
   (~90s).

### Reference artifacts from the proven run
- Cursor agent id: \`bc-a54a6cc2-4a50-47a5-8b1e-4162bf470e51\`
- Final pushed commit (cursor branch): \`68e70a8\`
- Helper script authored by the agent:
  \`C:\\Users\\xsj70\\bryanblog\\scripts\\hydrate-portfolio-planning-2.mjs\`
  (reusable starting point for future hydrations — the agent self-derives
  this from \`.cursorrules\` plus the existing
  \`adopt-portfolio-planning.mjs\` reference)
- Live URL: https://<REDACTED-HOST>/retirement-income/portfolio-planning-2/
`.trim(),
    },

    chaperoneMerge: {
      description:
        "The FF-merge step the chaperone runs after the Cursor agent FINISHEDs successfully. Cursor sandbox cannot push to main; the chaperone bridges.",
      howTo: `
Per [SB:cursorCJGEO->refCursorSandboxBehavior] rule #1, the Cursor sandbox
pushes to \`cursor/<slug>-<hash>\` and CANNOT push to \`main\`. The chaperone
(or a human) bridges the merge.

### Inputs
- \`COMMIT_SHA\` and \`BRANCH\` from the
  [SB:cursorCJGEO->refReportingContract] block in the agent's final message.

### Canonical FF-merge sequence
\`\`\`bash
cd C:\\Users\\xsj70\\bryanblog
git fetch origin
git checkout main
git pull --ff-only origin main          # surface any concurrent main moves first
git merge --ff-only origin/<BRANCH>     # the cursor/<slug>-<hash> ref
git push origin main
\`\`\`

\`--ff-only\` is mandatory. A merge that would require a true merge commit
indicates main diverged during the cursor run — see "FF-fails" below.

### Verify the SHA
After \`git push\`, \`git rev-parse HEAD\` on main MUST equal \`COMMIT_SHA\`
from the agent's report. If it doesn't, surface the mismatch immediately —
it means either (a) the agent's report had a stale SHA, or (b) the push to
main partially landed.

### FF-fails (someone pushed to main meanwhile)
If \`git merge --ff-only\` errors with "Not possible to fast-forward", the
chaperone has three choices:
1. **Rebase the cursor branch onto main** (preferred for single-commit
   cursor runs):
   \`\`\`bash
   git checkout <BRANCH>
   git rebase main
   git checkout main
   git merge --ff-only <BRANCH>
   git push origin main
   \`\`\`
   This rewrites the cursor branch SHA; record the new SHA for any audit
   surface that pinned the old one.

2. **3-way merge** (acceptable for multi-commit cursor runs):
   \`\`\`bash
   git checkout main
   git merge --no-ff origin/<BRANCH>
   # resolve conflicts if any
   git push origin main
   \`\`\`
   Document the merge commit as the canonical "the cursor run landed at X" SHA.

3. **Surface the dirty state** if the conflicts are non-trivial: per the
   3-strike rule, do NOT loop. Hand the merge to the user with a description
   of the conflicting files and the cursor branch ref to inspect.

### Post-merge: Vercel auto-deploy
A push to main on \`boster00/bryanblog\` triggers Vercel auto-deploy
(~75–90s typical for bryanblog). The chaperone waits this out before
[SB:cursorCJGEO->cicVerify].
`.trim(),
    },

    cicVerify: {
      description:
        "Visual-verify the live bryanblog URL via CIC after Vercel auto-deploys. Defers to the browsercontrol skill book for the actual procedure.",
      howTo: `
The user-channel verification gate per CLAUDE.md's success-claim discipline.
This is a thin wrapper around [SB:browsercontrol->validateCIC] specialized for
the bryanblog post-deploy scenario.

### Procedure
1. **Wait for Vercel.** ~90s post-push to main on \`boster00/bryanblog\`.
   While waiting, the chaperone can do other lightweight work; don't poll
   the URL faster than every 15s — Vercel's edge cache may serve stale
   until the build is fully promoted.

2. **Validate CIC is healthy.** Run [SB:browsercontrol->validateCIC] to
   confirm Claude-in-Chrome can take a real screenshot. If it can't, STOP
   per the validateCIC gate — do NOT proxy-verify via
   \`Claude_Preview\` / curl / headless.

3. **Navigate via CIC.** Open the live URL from the agent's
   [SB:cursorCJGEO->refReportingContract] block:
   \`https://<REDACTED-HOST>/retirement-income/<newSlug>/\`.

4. **Screenshot + visual checks** against the design-guide expectations
   ([SB:BryanBlog->refDesign] + the live \`/design-guide\` page):
   - Drop-cap on the first paragraph fires.
   - H2 terracotta-underline visible.
   - Body text is stone-800 on warm-off-white \`bg\`.
   - CTA blocks (if present) intact — no \`.prose-article\` greediness
     regressions (the \`not-prose\` discipline held).
   - Related-articles strip at the bottom unaffected by prose styles.
   - No leaked unstyled text fragments (the sanitizer's
     \`keepTopLevelBlocks\` pass per [SB:BryanBlog->adoptCJGEORawContent]
     did its job).

5. **Verify deep-dive links present.** Each section should end in an
   italic \`→ Deep dive: <Sibling Title>\` line per the proven sanitizer.

6. **Compliance spot-check.** Skim for product names, insurer names, return
   projections, testimonials, or hard "buy" CTAs — all banned per
   [SB:BryanBlog->refStrategy] rules 5 + 7 and \`.cursorrules\` section 5.
   CJGEO drifts into these despite the prompt. If found, the agent's
   sanitizer / compliance pass missed it — surface as a follow-up task,
   don't block on it.

### Honest scope per CLAUDE.md
Per the Sage success-claim gate: an unverified live URL is NOT a verified
success. The chaperone's report to the user only claims DONE after this
visual-verify pass — anything short is framed as State 1 ("agent finished
and pushed, visual verify pending").
`.trim(),
    },

    refAntipatterns: {
      description:
        "Reference (LOCKED): discipline antipatterns observed on chaperone + leaf-agent runs. The 'waiting for notification' self-defer is the most common. Read before kicking off any monitorCursor loop.",
      howTo: `
LOCKED — these are antipatterns the chaperone (and any leaf agent dispatching one) must NOT fall into. They are observed-in-the-wild discipline failures, not hypothetical.

### 1. "Waiting for notification" self-defer (MOST COMMON)

**Symptom.** The chaperone (or a leaf agent dispatching a Cursor agent) writes a closing message like:
- "I'll continue when the monitor notifies me that the agent finished."
- "Setting up a background watch — I'll resume when there's a status change."
- "The Cursor agent is running; I'll check back in 10 minutes."
- "Returning to wait for the next followup."

…and ends the turn. This pretends the loop has a notification channel that flips a future turn back into existence. **There is no such channel** for the chaperone-dispatching-Cursor case. The loop is a FOREGROUND \`until\` block per [SB:cursorCJGEO->monitorCursor] — when the agent terminates the turn, the loop terminates too, and nothing resumes it.

This is the load-bearing variant of the GuildOS memory rule "SE runs never stop at a checkpoint" applied to chaperone runs. The cron-or-nudge pattern exists for genuine multi-hour autonomous loops (the dailies cron, scheduled tasks) — NOT for a chaperone monitoring a single Cursor agent for 15–30 minutes.

**The discipline.** Poll the foreground in a tight \`until\` loop (60s cadence per [SB:cursorCJGEO->monitorCursor]). Do other lightweight work between polls if useful, but DO NOT close the turn while the agent is in a non-terminal status (CREATING / RUNNING / FINISHED-pending-followup). The only acceptable exits per CLAUDE.md's end-state contract:
- **State 1** — a genuine question the user must answer (e.g. 3-strike blocker, EXPIRED + needs human decision).
- **State 2** — the run is verified-done per [SB:cursorCJGEO->cicVerify] and reported.

A "waiting" closing message is neither State 1 nor State 2 — it's the banned third state.

### 2. Proxy verification instead of CIC

**Symptom.** "I curled the URL and got 200 — the deploy is verified." or "I used Claude_Preview to render the page and it looks correct." Neither is the user-channel; both deviate per the alignment theorem ([SB:browsercontrol->validateCIC] + CLAUDE.md Sage success-claim gate). The only acceptable visual-verify is CIC against the user's connected Chrome — see [SB:cursorCJGEO->cicVerify].

### 3. Restating .cursorrules content in the dispatch prompt

**Symptom.** The chaperone pastes the entire 16-section .cursorrules body into the create-agent POST's \`prompt.text\`. Doubles context use and creates a divergence risk if .cursorrules is then edited mid-run. The dispatch prompt should be the ONE-PARAGRAPH task brief per [SB:cursorCJGEO->dispatchCursor] Step 3 — the agent reads .cursorrules from its own cloned working tree.

### 4. Relaunching on FINISHED instead of following up

**Symptom.** Agent goes FINISHED with \`STATUS: partial\` or \`STATUS: blocked\`; the chaperone kills the agent and dispatches a fresh one. Loses the working-tree context (the partial branch, the .env.local, the WIP commits). Per [SB:cursorCJGEO->refCursorSandboxBehavior] rule #3, FINISHED is RESUMABLE — send a followup; the agent flips back to RUNNING with the same working tree. Only relaunch when the status is EXPIRED.

### 5. Manually merging cursor's branch via the GitHub UI

**Symptom.** Chaperone opens \`github.com/boster00/bryanblog\` in CIC and clicks "Merge pull request" on the cursor branch. Works, but skips the SHA-verification step in [SB:cursorCJGEO->chaperoneMerge] — the chaperone has no way to confirm \`git rev-parse HEAD\` on main matches \`COMMIT_SHA\` from the agent's report. Use the canonical git FF-merge sequence so the SHA check is mechanical.
`.trim(),
    },

    refBosterbioDefault: {
      description:
        "Reference (LOCKED 2026-05-21, route hydration live at cjgeo 22fedb6d): when working on bosterbio.com projects (cursorCJGEO's main job), always pass use_custom_template:true on generateDraft AND verify the adopted content_html contains at least one Boster template signature class (hero-section, col-md-4, dark-overlap, font-large, card-item, row-of-cards, hero-twin, font-bold) before sanitize/publish. Regenerate if missing.",
      howTo: `
LOCKED — operator correction 2026-05-21 (refreshed after Phase A). cursorCJGEO's PRIMARY job is bosterbio.com proving-grounds adoption (the \`cjgeo-mockup-N\` family + future bosterbio CJGEO pages). The bosterbio downstream sanitizer ([SB:bosterbio->adoptCJGEOIntoMagentoPage]) is preserve-first: it relies on CJGEO's custom-template assembly path to emit the Boster template class surface (\`hero-section\`, \`col-md-4\`, \`dark-overlap\`, \`font-large\`, \`card-item\`, \`row-of-cards\`, \`hero-twin\`, \`font-bold\`, etc.) PLUS the inline \`<style data-draft-styles>\` block. The default-template assembly path does NOT emit those classes — articles generated without \`use_custom_template:true\` will publish as the unstyled fallback that \`cjgeo-mockup-1\`'s prior stale publish produced.

**Route prerequisite — cjgeo commit \`22fedb6d\`** (deployed 2026-05-21). Earlier CJGEO route builds silently dropped \`use_custom_template:true\` even when the flag was sent; the patch at \`22fedb6d\` actually hydrates Boster templates from the requesting user profile (\`3f5f80b7-17c8-4b1e-bfae-fa91d0ae95d2\` loads 21 custom templates incl. 8 Boster ones). Do NOT roll back through that commit without re-evaluating this whole verify rule.

This reference is THE substrate-level lock. Every action in this skill book that calls \`generate_draft\` against a bosterbio target — directly OR via a dispatched cursor agent — MUST follow both rules below.

### Rule 1 — Always pass \`use_custom_template:true\` on generateDraft for bosterbio targets

When the destination is a bosterbio.com CMS page (any \`proving-grounds/<slug>\` identifier, any future bosterbio CJGEO destination), the \`POST /api/articles/{id}/edit-draft {action:'generate_draft', ...}\` body MUST include:

\`\`\`json
{
  "action": "generate_draft",
  "mode": "full_html",
  "allow_image_generation": true,
  "use_default_context_prompt": true,
  "generation_prompt_mode": "auto",
  "use_custom_template": true
}
\`\`\`

\`use_custom_template:true\` is the load-bearing field. The other fields are the proven defaults from the CJGEO Bard run; only override on explicit instruction. NEVER omit \`use_custom_template:true\` for a bosterbio target — default-template output is INCOMPATIBLE with [SB:bosterbio->adoptCJGEOIntoMagentoPage] sanitize floor.

### Rule 2 — Verify Boster template signature classes in content_html BEFORE sanitize/publish

After \`adoptDraft\` (or after a direct \`POST /api/articles/{id} {action:'get_html'}\` fetch), grep the article's \`content_html\` for ANY of the Boster template signature classes:

- \`hero-section\`
- \`col-md-4\`
- \`dark-overlap\`
- \`font-large\`
- \`card-item\`
- \`row-of-cards\`
- \`hero-twin\`
- \`font-bold\`

These are emitted by the assembled Boster custom templates that the CJGEO route now hydrates (per the \`22fedb6d\` route patch) and are absent when the default path was used. Sum occurrence counts across all eight class names.

\`\`\`bash
# pseudo-code: assume content_html captured in $HTML
CLASSES=(hero-section col-md-4 dark-overlap font-large card-item row-of-cards hero-twin font-bold)
TOTAL=0
for C in "\${CLASSES[@]}"; do
  N=$(printf '%s' "$HTML" | grep -o "$C" | wc -l)
  TOTAL=$((TOTAL + N))
done
if [ "$TOTAL" -ge 1 ]; then
  echo "compliant — proceed to sanitize+publish (boster_class_count=$TOTAL)"
else
  echo "non-compliant — regenerate"
fi
\`\`\`

- **TOTAL >= 1** → article is compliant. Proceed to [SB:bosterbio->adoptCJGEOIntoMagentoPage] sanitize + publish.
- **TOTAL = 0** → article was generated without \`use_custom_template:true\` OR the CJGEO route silently failed to hydrate templates. REGENERATE per Rule 3.

### Rule 3 — Regenerate-on-fail loop (max 3 attempts)

If the verify in Rule 2 returns TOTAL = 0:

1. POST \`/api/articles/{id}/edit-draft {action:'generate_draft', use_custom_template:true, ...}\` per Rule 1.
2. Poll \`pull_status\` per [SB:cjgeo->refApiActions] every 60s; cap 20 minutes.
3. \`adoptDraft\`.
4. Re-fetch \`content_html\` via \`{action:'get_html'}\`.
5. Re-grep for Boster signature classes. If TOTAL >= 1 → compliant, proceed. If TOTAL = 0 → loop back to step 1.

Cap at 3 regenerate attempts. After 3 attempts without compliance, emit \`STATUS: blocked\` with \`BLOCKERS: Boster template classes absent after N regenerates (counts: <per-class breakdown>)\` and stop the publish. Do NOT ship a non-compliant article — it will render the unstyled fallback.

### Relationship to [SB:bosterbio->adoptCJGEOIntoMagentoPage]

The bosterbio adoption skill book carries the mirror lock on its side: a pre-sanitize compliance verify step that aborts the publish if Boster template signature classes are absent and triggers regenerate-back-to-cursorCJGEO. The two skill books are co-locked — neither can ship a non-compliant article unilaterally.

### When this rule does NOT apply

- bryanblog (Next.js) adoption per [SB:BryanBlog->adoptCJGEORawContent] — bryanblog's adoption pipeline uses Tailwind \`.prose-article\` and does NOT depend on the Boster template classes. \`use_custom_template:true\` is still recommended for consistent design surface, but is not load-bearing there.
- Any non-bosterbio CJGEO destination — judge case-by-case; the lock is specifically tied to the bosterbio preserve-first sanitizer.
`.trim(),
    },

    refBracketPlaceholderScan: {
      description:
        "Pattern: detect CJGEO image/diagram placeholders by scanning innerText for bracketed tokens and LLM-classifying each as PLACEHOLDER vs PROSE. Generalizes across CJGEO's many variant shapes (`[Diagram:`, `[Chart:`, `[VISUAL PLACEHOLDER:`, `[Image:`, `[Illustration of …]`, etc.) without per-project regex tuning. Used by [[FullAutoWebsiteDevelopmentRoadmap]] M4/M5 milestone detection + inventory.",
      howTo: `
**Why this exists.** Each CJGEO project emits placeholders in slightly different shapes — bryanblog used \`[Diagram: <title>]\` / \`[Chart: ...]\` etc. (a closed token list); cloudstonedesigns emits \`<!-- [VISUAL PLACEHOLDER: <description>] -->\` HTML comments + \`<div class="media-placeholder">\` shells; future projects will invent new shapes. Maintaining an allowlist of bracket tokens has the same losing-game property as maintaining a CSS class allowlist in the sanitizer (see [SB:BryanBlog->adoptCJGEORawContent §Step 2 "DROP-LIST not ALLOW-LIST"]). Invert: scan ALL bracketed tokens in the article's visible text, and let an LLM say which are real placeholders.

**Why \`[]\` is safe to scan for.** Square brackets occur rarely in natural prose. Legitimate usage is mostly: citations (\`[1]\`, \`[Smith 2023]\`), redaction markers (\`[name]\`, \`[redacted]\`), and editorial inserts in quoted material (\`"I went [to the store]"\`). All are short and well-formed; an LLM classifier disambiguates them from placeholders in one pass with ~zero false positives in practice. Compare to scanning for \`{...}\` (common in code samples) or \`(...)\` (extremely common in prose) — those would drown in noise.

---

### Pipeline

**Step 1 — Strip tags to innerText.**
For each \`content/**/*.html\` file (or any HTML string), strip all tags to a plain-text stream. Preserve HTML comments (\`<!-- ... -->\`) since CJGEO sometimes wraps placeholders in comments (cloudstonedesigns precedent). Implementation: cheerio \`$('body').text()\` + a second pass that pulls comment text out via \`$('*').contents().filter(n => n.type === 'comment')\`.

**Step 2 — Regex-extract bracket tokens.**
Pattern: \`/\\[([^\\]\\n]{2,200})\\]/g\` — square brackets, contents 2–200 chars, no nested brackets, no newlines. The 2-char minimum cuts \`[ ]\` whitespace; the 200-char ceiling cuts pathological matches. For each hit, capture:
- \`rawToken\` — the full \`[...]\` match
- \`innerText\` — the captured group
- \`position\` — char offset in the original HTML (not the stripped text — for replacement later)
- \`surroundingContext\` — ±600 chars of original HTML around the position (used as prompt context for image gen later)

**Step 3 — LLM-classify each token.**
Batch all unique \`innerText\` values (dedup — many articles repeat the same placeholder shape) and send to a cheap classifier model (gpt-4o-mini is the proven default). System prompt:

\`\`\`
You receive bracketed tokens extracted from a content article. Classify each as one of:
- PLACEHOLDER: a marker for editorial media that should be filled (diagram, chart, image,
  illustration, figure, photo, visual). Examples: "Diagram: how interest rates affect bond
  prices", "Chart of inflation 2020-2024", "VISUAL PLACEHOLDER: hero shot of warehouse",
  "Image of architect with client", "Illustration showing the three buckets".
- PROSE: legitimate use of brackets in body text. Examples: citations "[1]", "[Smith 2023]";
  redaction "[name]"; editorial inserts in quotes '"I went [to the store]"'; abbreviation
  expansions "AI [artificial intelligence]".

Return JSON: { "<rawInnerText>": "PLACEHOLDER" | "PROSE", ... }.
Be conservative — if a token is ambiguous, classify as PROSE so we don't accidentally
replace real content.
\`\`\`

Send all tokens in one request (cheap; gpt-4o-mini handles hundreds of short strings per call).

**Step 4 — Enrich PLACEHOLDER hits.**
For each token classified PLACEHOLDER, run a second LLM pass (also cheap, can batch) extracting structured fields from \`innerText\`:

\`\`\`
For each PLACEHOLDER token, extract:
- kind: "diagram" | "chart" | "image" | "illustration" | "figure" | "photo" | "visual" | "graph" | "other"
- description: the descriptive part after the colon/keyword (or the full string if no clear keyword)
- suggestedMetaphor: if kind in {diagram, chart, figure}, propose a concrete visual metaphor;
  if kind in {image, photo, visual, illustration}, propose a photographic/editorial-art direction.
\`\`\`

The structured output is what M5's fill module consumes.

**Step 5 — Emit the inventory.**
\`\`\`json
{
  "scannedFiles": 26,
  "totalBracketHits": 47,
  "prose": 25,
  "placeholders": [
    {
      "file": "content/choosing-an-architect/how-people-choose-architects.html",
      "rawToken": "[VISUAL PLACEHOLDER: Full-width atmospheric image of consultation]",
      "kind": "image",
      "description": "Full-width atmospheric image of consultation",
      "suggestedMetaphor": "wide-aspect photograph of architect and client reviewing plans on a large table, natural window light, no faces visible",
      "position": 4821,
      "surroundingContext": "...600 chars before...[VISUAL PLACEHOLDER:...] ...600 chars after..."
    },
    ...
  ]
}
\`\`\`

This is the data structure the M5 fill module (\`scripts/fill-media-placeholders.mjs\` or \`lib/fill-diagrams.mjs\` — naming is per-project) iterates over.

---

### Used as the milestone-detection gate

Per [[FullAutoWebsiteDevelopmentRoadmap]]:
- \`scanBracketPlaceholders(content/).placeholders.length > 0\` → enter at M5
- \`scanBracketPlaceholders(content/).placeholders.length === 0\` AND content files exist → M5 complete, route to M6

### Edge cases / failure modes

- **HTML comments containing brackets.** Cheerio's default \`.text()\` skips comments — pull them explicitly per Step 1. Otherwise cloudstonedesigns-shape \`<!-- [VISUAL PLACEHOLDER: ...] -->\` placeholders are invisible to the scan.
- **Code blocks (\`<pre><code>\`) with bracket content.** Programming code uses brackets heavily. Either skip \`<pre>\`/\`<code>\` element text in Step 1, OR let the LLM classify code-bracket hits as PROSE (it reliably does). Default: skip code blocks — saves LLM cost on otherwise noise-heavy articles.
- **Nested brackets (\`[outer [inner]]\`).** The regex avoids nesting by design. If CJGEO ever emits nested shapes, expand the regex; not observed in practice.
- **Classifier false negative.** If a real placeholder gets classified as PROSE, the M5 fill is incomplete and M6 visual verify catches it (Bard's gpt-4o vision pass will flag the bracket text on screen). Treat M6 as the safety net.
- **Classifier false positive.** A piece of legitimate prose gets fill-attempted. The fill module's per-placeholder image-gen is reversible — if the user spots a wrong replacement at M6, regenerate with PROSE-classification correction. Rare in practice when prompted conservatively per Step 3.

### Why this generalizes across projects

- bryanblog (\`[Diagram: ...]\`) → caught by scan, classified PLACEHOLDER, kind=diagram, suggestedMetaphor → editorial vector
- cloudstonedesigns (\`<!-- [VISUAL PLACEHOLDER: ...] -->\`) → comment-aware strip catches it, classified PLACEHOLDER, kind=image, suggestedMetaphor → architectural photography
- Future project with \`[Photo of ...]\` or \`[Show: ...]\` or any new shape → also caught, no regex update needed

This is the substrate-promotion pattern: replace per-project token allowlists with a domain-aware classifier that sees the same signal humans see.
`.trim(),
    },

    FullAutoWebsiteDevelopmentRoadmap: {
      description:
        "Stand up a NEW content-focused website end-to-end leveraging CJGEO + Cursor cloud agents — decision-journey topics in, live Vercel site out. Milestone-based roadmap that an SE overseer Claude thread executes via [SB:supervisedEvolution->run]. Designed so a fresh thread can ENTER AT ANY MILESTONE by inspecting the asset state and resuming from the next chapter — no re-running upstream work that already converged.",
      howTo: `
**What this is.** A milestone-keyed roadmap for the Overseer SE thread that drives a full website-buildout lifecycle: USER provides customer journeys + topics + repo name → CJGEO generates every page → Cursor cloud agent (the "Bard") adopts + visually verifies each page in-worker → repo lands on github.com pushed to main, build green, Vercel-deployable.

**Why milestones, not a linear plan.** A fresh Overseer thread may be invoked at any point — first-time kickoff, mid-run cron resume, post-incident recovery, or partial-site extension. Each milestone declares (1) what was just accomplished, (2) what assets MUST exist (verifiable), (3) what to do in the next chapter. The thread's first action is always **milestone detection** — inspect the assets, pick the lowest unmet milestone, resume from there.

**Verification channel — load-bearing override.** Per-page visual verification is done by Bard in-worker via sandbox Playwright + gpt-4o vision per [[refCursorVisualVerify]] — NOT GM CIC. CIC is reserved for the end-point 3-tab reveal to the user per [SB:browsercontrol]. When Bard demurs "I can't take screenshots", cite SE-cursor-hand-off-test step 7 + cursor session bc-8b47ceed-740e-4ed0-9e4d-d46e46182b69 as proof and continue.

---

## USER INPUTS (collected once at M0)

- \`REPO_NAME\`, \`GITHUB_OWNER\`, \`LOCAL_DEV_PORT\` (avoid 3002 / 3004 / 3005)
- \`AUTHOR_SURFACE\`, \`TONE\`, \`AUDIENCE\`, \`COMPLIANCE_CONSTRAINTS\`
- \`JOURNEYS\`: \`[{ pillar_slug, pillar_title, pillar_decision_question, satellites: [{ slug, title, main_keyword, angle }, ...] }, ...]\` — one pillar per customer journey; each satellite = one CJGEO article
- \`DESIGN_LANGUAGE_SOURCE\` (optional repo to port idioms from; default: bryanblog)

---

## END POINT (only PASS reason to stop the SE loop)

1. \`github.com/{GITHUB_OWNER}/{REPO_NAME}\` main exists, build green, all pages CJGEO-adopted.
2. Every page's Bard verdict JSON attached to \`items[].notes\`.
3. Quest \`stage=complete\`; Cat APPROVE + Guildmaster FINAL_GATE_PASS per item.
4. End-point reveal: 3 tabs in user's real Chrome — GitHub repo home, \`localhost:{LOCAL_DEV_PORT}/{first-pillar-slug}/\`, \`/quest-board/{quest_id}\`.

---

## MILESTONE DETECTION (first action of every thread)

Before doing ANY work, the Overseer determines the current milestone by checking assets in order:

\`\`\`
if no quest exists with this REPO_NAME in title         → enter at M0
elif !exists(C:\\Users\\xsj70\\{REPO_NAME})              → enter at M1
elif no Bard adventurer + rubric file                   → enter at M2
elif lib/articles.ts has any { cjgeoArticleId: null }   → enter at M3
elif content/**/{slug}.html missing for any satellite   → enter at M4
elif scanBracketPlaceholders(content/) returns any LLM-confirmed placeholders  → enter at M5
        (see [[refBracketPlaceholderScan]] — strip tags, find /\\[[^\\]]{2,200}\\]/ in
         innerText, LLM-classify each as PLACEHOLDER | PROSE; \`[]\` is rare in real
         prose so the scan is cheap + high-recall across CJGEO's variant shapes)
elif any item has no verdict JSON OR verdict=FAIL       → enter at M6
elif any open PR on cursor/* branch not FF-merged       → enter at M7
elif quest stage != complete                            → enter at M8
else                                                    → END POINT reached, exit
\`\`\`

Then proceed from that milestone's "Next chapter" section.

---

## M0 — Intake & Quest

**Just accomplished:** USER provided inputs (or upstream cron resumed mid-run).

**Expected assets:** None on disk yet. Inputs gathered in-thread.

**Next chapter:**
1. Read CLAUDE.md + CONTEXT.md from \`main\` (cross-role rule).
2. [SB:questmaster->summarizeRelevantContext] on the inputs → [SB:questmaster->evaluate]. Real gaps only → [SB:questmaster->SMARTClarification] Tier-1/2. Route every would-be question through [SB:sage->askSage] first (reversible defaults to PROCEED).
3. [SB:questmaster->upsertQuest]: JOURNEYS expanded into \`items[]\` — one row per pillar + one per satellite, \`kind=link\`, \`ref=expected URL\`. Add \`topic_briefs\` markdown item per [SB:BryanBlog->auditUnresearched] taxonomy.
4. [SB:DungeonMaster->authorSERubric] → rubric file at \`docs/smoke-tests/SE-{REPO_NAME}-buildout.md\` honoring [[refLayoutContract]].
5. Register continuation cron per [SB:supervisedEvolution->continueSERun].

**Exit criterion:** Quest exists in \`/quest-board/{quest_id}\` with full \`items[]\`; rubric file committed; cron registered.

---

## M1 — Repo Scaffold

**Just accomplished:** Quest + rubric + cron live; SE loop is running.

**Expected assets:**
- \`/quest-board/{quest_id}\` reachable, items populated
- Rubric file at \`docs/smoke-tests/SE-{REPO_NAME}-buildout.md\`
- Active SE cron job

**Next chapter:**
1. Scaffold \`C:\\Users\\xsj70\\{REPO_NAME}\` (\`npx create-next-app\` Next 15 / Tailwind 4 / TS / app router).
2. Port the DESIGN FLOOR verbatim from \`DESIGN_LANGUAGE_SOURCE\` (default bryanblog) per [SB:BryanBlog->refDesign]:
   - \`app/globals.css\` \`@theme\` block + \`.prose-article\` hand-written + drop-cap opt-out + SVG sizing defaults
   - \`components/{ArticleShell, Cta, RelatedArticles, SiteNav}.tsx\` with \`not-prose\` discipline intact
   - \`lib/sanitize-cjgeo.mjs\` + TS shim + \`lib/fill-diagrams.mjs\` + \`scripts/backfill-diagrams.mjs\`
3. Substitute \`AUTHOR_SURFACE\` everywhere per [[refBranding] adapted from SB:BryanBlog->refBranding]].
4. Init \`lib/articles.ts\` from \`JOURNEYS\` — entries with \`{ slug, title, teaser, source: 'pending', cjgeoArticleId: null }\`.
5. Wire \`components/SiteNav.tsx\` with the pillars; first \`npm run build\` green on \`LOCAL_DEV_PORT\`.
6. Create GitHub repo: \`gh repo create {GITHUB_OWNER}/{REPO_NAME} --public --source=. --push\`.

**Exit criterion:** Repo on disk + on GitHub main; sanitize-cjgeo + fill-diagrams modules present; build green; \`lib/articles.ts\` seeded with every JOURNEYS satellite (cjgeoArticleId null).

---

## M2 — Bard Spawn

**Just accomplished:** Repo skeleton on disk + GitHub; build green; sanitizer + diagram-fill substrate in place.

**Expected assets:**
- \`C:\\Users\\xsj70\\{REPO_NAME}\\\` with the modules above + first commit pushed
- \`{GITHUB_OWNER}/{REPO_NAME}\` repo visible on GitHub
- \`lib/articles.ts\` complete with all JOURNEYS rows

**Next chapter:**
1. Sync \`.cursorrules\` into \`{REPO_NAME}\` from \`libs/skill_book/cursorCJGEO/.cursorrules.template\` per [[refCursorRulesTemplate]]; substitute \`{REPO_NAME}\` / \`{GITHUB_OWNER}\` / \`AUTHOR_SURFACE\` placeholders.
2. Spawn the Bard adventurer per [SB:roster->spawnAgent] with skill_books \`[housekeeping, worker, cursorCJGEO, cjgeo]\`. Assign to the quest. Persist \`session_id\` + \`session_status=busy\` on the adventurer row (DB read-back verified).
3. Send CJGEO_API_KEY + OPENAI_API_KEY via inline-followup per [[refCredentialHandoff]] — NEVER commit secrets in \`.cursorrules\`.
4. Verify cold-start: \`GET /conversation\` shows \`housekeeping.initAgent\` ran, skill-book TOCs loaded, working tree confirmed.

**Exit criterion:** Bard session \`RUNNING\` with credentials confirmed; adventurer row patched; quest \`assignee_id\` = Bard.

---

## M3 — CJGEO Asset Generation (per-satellite, SERIAL)

**Just accomplished:** Bard live with credentials; repo + adventurer wired.

**Expected assets:**
- Bard adventurer row \`session_status=busy\` with valid \`session_id\`
- Quest assignee = Bard

**Next chapter (Bard runs this per satellite, SERIALLY per [SB:cjgeo->refDataForSEOSeedStrategy]):**

For each \`{ pillar_slug, satellite }\` in JOURNEYS:
1. CJGEO article create per [SB:cjgeo->refApiActions §1].
2. setMainKeyword → [SB:cjgeo->keywordFullAuto] → [SB:cjgeo->topicFullAuto].
3. [SB:cjgeo->refFullAutoGate] — NON-NEGOTIABLE: assert \`main_keyword + keywords[] + topics[]\` all non-empty before generateDraft.
4. [SB:cjgeo->generateDraft] with \`context_prompt\` including the journey-stage angle + \`AUTHOR_SURFACE\` branding clause per [SB:BryanBlog->refBranding].
5. [SB:cjgeo->pollDraft] (5-min cadence, 20-min timeout) → [SB:cjgeo->adoptDraft].
6. Patch \`lib/articles.ts\` row: \`cjgeoArticleId, mainKeyword, source: 'cjgeo-pending-adoption'\`.

If a satellite hit the broken-window substrate (kept=0), use [SB:BryanBlog->auditUnresearched] taxonomy to re-classify and re-research before generateDraft.

**Exit criterion:** Every satellite in \`lib/articles.ts\` has a non-null \`cjgeoArticleId\`; \`get_assets\` per article confirms keywords[] + topics[] non-empty; CJGEO Supabase has the adopted draft per article.

---

## M4 — Adoption into Repo (sanitize + write + wire routes)

**Just accomplished:** All CJGEO articles generated + adopted on the CJGEO side; \`cjgeoArticleId\` committed for every satellite.

**Expected assets:**
- \`lib/articles.ts\` rows all have \`cjgeoArticleId\` set
- CJGEO \`content_magic_articles\` rows exist with non-empty \`content_html\`

**Next chapter (Bard runs Steps 1–4 + 6–9 of [SB:BryanBlog->adoptCJGEORawContent] per satellite — diagram fill deferred to M5):**

Per satellite, Bard executes:
1. Fetch raw HTML (\`get_html\`).
2. Sanitize via \`lib/sanitize-cjgeo.mjs\` (drop-list, NOT allow-list).
3. Inject h2 ids (slug-from-text on satellites; anchor-map on pillars).
4. Inject inline deep-dive satellite links per section.
5. Write \`content/{pillar_slug}/{slug}.html\` with header comment (\`article_id / main_keyword / adopted_at\`). **Placeholders \`[Diagram: ...]\` / \`[Chart: ...]\` / \`[Visual: ...]\` / \`[Illustration: ...]\` / \`[Figure: ...]\` / \`[Graph: ...]\` STAY in the file at this milestone — M5 fills them.**
6. Patch \`lib/articles.ts\`: \`source: 'cjgeo', adoptedAt\`.
7. Rewrite \`app/{pillar_slug}/{slug}/page.tsx\` as async server component reading from disk.
8. \`npm run build\` green (placeholders render as literal text — that's expected at this milestone).

Pillar pages (one per JOURNEY) follow the same path but with anchor-map mode.

**Why diagram fill is its own milestone:** image generation is expensive (60–120s per diagram at gpt-image-2 quality:high), failure-prone (retries + metaphorHint refinement), and idempotent on re-runs (\`scripts/backfill-diagrams.mjs\` scans \`content/**\` and rewrites in place). Splitting it out means a re-entered thread can resume image work without re-sanitizing; a failed image gen doesn't roll back sanitized content; and the diagram-fill substrate evolution (prompt tuning, metaphor library) lives at a clean seam.

**Exit criterion:** \`content/**/*.html\` exists for every satellite + pillar; \`app/**/page.tsx\` server components live; \`npm run build\` green; \`grep -i "<old-author-leak-token>"\` per [SB:BryanBlog->refBranding] clean. Diagram placeholders MAY still be present — that's M5's job.

---

## M5 — Diagram Backfill (gpt-image-2)

**Just accomplished:** Sanitized HTML for every page on disk; routes wired; build green with placeholders rendering as literal text.

**Expected assets:**
- \`content/**/*.html\` populated for every satellite + pillar
- \`app/**/page.tsx\` server components reading from disk
- Some or all content files contain \`[Diagram: ...]\` / \`[Chart: ...]\` / \`[Visual: ...]\` / \`[Illustration: ...]\` / \`[Figure: ...]\` / \`[Graph: ...]\` placeholders (and/or wrapping \`<div class="diagram-placeholder">\` shells)
- \`lib/fill-diagrams.mjs\` + \`scripts/backfill-diagrams.mjs\` ported from bryanblog (delivered at M1)

**Next chapter — execute the diagram-fill mandate from [SB:BryanBlog->adoptCJGEORawContent §Step 4.5]:**

1. **Inventory placeholders via the bracket-scan + LLM-classify pattern (see [[refBracketPlaceholderScan]]).** Bard runs the scan across \`content/**/*.html\`, gets back the LLM-confirmed placeholder set (token + position + surrounding-context window ≈ 600 chars), and emits a per-article fill plan: \`{ articleSlug, placeholders: [{ n, rawToken, kind, description, surroundingContext, suggestedMetaphor }] }\`. The bracket scan replaces hardcoded token lists (\`[Diagram:\`, \`[Chart:\`, …) and catches CJGEO's variant shapes (\`[VISUAL PLACEHOLDER:\`, \`[Image:\`, \`[Photo:\`, \`[Illustration of …]\`, etc.) without per-project regex tuning.
2. **Generate in parallel per article (concurrency ≤ 4).** Use \`fillDiagrams(sanitizedHtml, { slug, publicDir, openaiImagesGenerate, surroundingContextChars })\` exported from \`lib/fill-diagrams.mjs\`. The \`openaiImagesGenerate\` injects the GuildOS \`openai_images\` weapon's \`generate\` export. Model: \`gpt-image-2\`, quality: \`"high"\`. Save to \`public/images/diagrams/{slug}-{n}.png\`.
3. **Rewrite HTML in place.** Each placeholder (and wrapping \`<div class="diagram-placeholder">\` shell where present) replaced with:
   \`\`\`html
   <figure class="diagram">
     <img src="/images/diagrams/{slug}-{n}.png" alt="{title}" loading="lazy">
     <figcaption>{title}</figcaption>
   </figure>
   \`\`\`
   Heuristic for wrapping divs: if the div contains ONLY the bracket text + decorative SVG → replace the WHOLE div. If it contains structural children (bucket cards, timeline steps, comparison panels) → KEEP the children, strip bracket + decorative SVG, INSERT \`<figure>\` at the top of the div.
4. **Prompt template + metaphorHint discipline.** \`CANONICAL_PROMPT_PREFIX\` + \`buildPrompt\` in \`lib/fill-diagrams.mjs\` is the source of truth — DO NOT restate prompts ad-hoc. Locked bans: no coin stacks / scales / growing plants / scenery / icon collages / burnt-in text or numbers. Composition: landscape, central 50–65% subject, generous \`#FAF9F8\` whitespace. When a regen produces a banned cliché, refine \`metaphorHint\` per diagram (e.g. "two ceramic vessels with horizontal anchor lines, NO plants") rather than retrying with identical input.
5. **Retry budget:** 3 retries per diagram. After 3 failures, REMOVE the placeholder entirely (do NOT ship literal \`[Diagram: ...]\` text). Log the failure to \`items[<page>].notes\` so M6 visual-verify knows the diagram is absent by design, not by regression.
6. **Backfill mode (idempotent re-runs).** When re-entering at M5 (cron resume / failure recovery), use \`scripts/backfill-diagrams.mjs\` which scans all content files and only generates for unmatched placeholders. Run with \`--env-file=...\` for \`OPENAI_API_KEY\`. Already-filled \`<figure class="diagram">\` blocks are skipped.

**Verification gate (must pass before M6):** re-run [[refBracketPlaceholderScan]] across \`content/\` — the LLM-confirmed-placeholder set MUST be empty (any remaining bracketed text is PROSE-classified). Then \`npm run build\` re-confirmed green.

**Exit criterion:** Zero placeholder matches in \`content/\`; every successfully-generated diagram has a corresponding PNG under \`public/images/diagrams/\`; failed diagrams (3-retry exhausted) are removed from content with a note logged to items[].notes; build green.

---

## M6 — Per-Page Visual Verify (Bard, in-worker)

**Just accomplished:** Every page adopted; content files on disk; build green locally on Bard's sandbox.

**Expected assets:**
- \`content/**/*.html\` for every satellite + pillar
- \`app/**/page.tsx\` as async server components reading from disk
- \`lib/articles.ts\` fully populated (\`source: 'cjgeo'\` everywhere)
- Local dev server reachable on Bard's sandbox

**Next chapter (Bard runs this — verification channel override applies):**

Per page, Bard runs Playwright + gpt-4o vision in-worker per [[refCursorVisualVerify]]:
1. Spin Next dev server on Bard's sandbox.
2. Playwright navigate → screenshot to \`public/_verify/{slug}.png\` (gitignored or cleaned at end).
3. gpt-4o vision inspects screenshot for: drop-cap renders, h2 underline, no \`.prose-article\` greedy regressions, callouts/buckets/stat-cards as designed panels, diagrams present, no literal \`[Diagram: ...]\` text, no \`&mdash;\` literals, no AUTHOR_SURFACE-leak token visible.
4. Emit verdict JSON:
   \`\`\`json
   {
     "url": "/{pillar_slug}/{slug}/",
     "screenshot_path": "...",
     "hero_ocr": "...",
     "diagrams_present": true,
     "placeholders_remaining": 0,
     "branding_leak": false,
     "layout_intact": true,
     "verdict": "PASS",
     "notes": "..."
   }
   \`\`\`
5. Write verdict to \`items[<page>].notes\` (the row whose \`ref\` matches the URL).
6. If verdict=FAIL, Bard loops on its own — fix sanitize/fill/CSS, re-render, re-verify. Escalate to Overseer only if 3 internal Bard loops fail to converge on the same page.

Stalled-RUNNING detection applies throughout per the substrate-discipline lesson in \`SE-cursor-hand-off-test-logs-deviations.md\`: \`/followup\` HTTP 200 + \`/conversation\` length unchanged in 30s → [SB:roster->spawnAgent] respawn, archive old session_id in quest_comment, do NOT retry the followup.

**Exit criterion:** Every \`items[]\` row has a verdict JSON; all verdicts = PASS.

---

## M7 — Merge to Main & Build Green on GitHub

**Just accomplished:** Every page Bard-verified PASS; verdict JSONs in items[].notes.

**Expected assets:**
- \`items[]\` complete with PASS verdicts
- Cursor sandbox branch \`cursor/{slug}-{hash}\` with all adopted content + verify pass

**Next chapter:**
1. Per [[chaperoneMerge]]: \`git fetch && git merge --ff-only origin/cursor/<slug> && git push origin main\` from the chaperone (GM) side. Sandbox cannot push to main directly per [[refCursorSandboxBehavior]] — this is enforced.
2. Wait for GitHub Actions / Vercel preview build green per [[refReportingContract]] STATUS/BUILD/URL.
3. Cat at purrview cross-checks every verdict JSON vs \`items[]\` row + the rendered page (Cat-side Playwright). APPROVE per item.
4. Guildmaster FINAL_GATE_PASS per item.

**Exit criterion:** \`main\` on GitHub has all content; build green; every item has Cat APPROVE + FINAL_GATE_PASS.

---

## M8 — Vercel Deploy + End-Point Reveal

**Just accomplished:** GitHub main fully built; quest items all gated.

**Expected assets:**
- GitHub main green
- All items Cat-approved + FINAL_GATE_PASS

**Next chapter:**
1. Connect Vercel project to \`{GITHUB_OWNER}/{REPO_NAME}\` (or trigger redeploy if already wired). Pattern per [SB:cloudstonedesigns->deployToVercel] + [SB:BryanBlog->deployToVercel].
2. Set required env vars (\`RESEND_API_KEY\` etc. — derived from JOURNEYS + repo capabilities).
3. GM CIC-verify the live Vercel URL per [SB:browsercontrol] — ONLY CIC verify of the whole run, end-point only.
4. Open the 3 reveal tabs in the user's real Chrome:
   - \`https://github.com/{GITHUB_OWNER}/{REPO_NAME}\`
   - \`http://localhost:{LOCAL_DEV_PORT}/{first-pillar-slug}/\` (or the Vercel URL if local dev not running)
   - \`http://localhost:3002/quest-board/{quest_id}\`
5. Quest \`stage=complete\`; closing finale proposes any substrate patches discovered during run per [SB:dailies->mergeInsight].
6. Delete the SE continuation cron per [SB:supervisedEvolution->run] terminal rule.

**Exit criterion:** Quest \`stage=complete\`, cron deleted, 3 tabs open. END POINT reached.

---

## Resume rules (if this thread is a cron-resumed continuation)

1. Read \`docs/smoke-tests/SE-{REPO_NAME}-buildout-logs-deviations.md\` (the durable log per [SB:supervisedEvolution->run] rule #7).
2. Read the resume header.
3. Run the MILESTONE DETECTION block above on current asset state — trust assets, not the log's claim of where it stopped (log may be stale by one step).
4. Proceed from that milestone's "Next chapter".

## Anti-patterns (do not do these)

- Don't re-run upstream milestones whose exit criteria already hold (wastes CJGEO credits + cursor minutes; risks regressing converged work).
- Don't skip [[refFullAutoGate]] under time pressure — silent SEO/GEO loss is the failure mode.
- Don't fall back to GM CIC for per-page verify — verification channel is Bard in-worker for this lifecycle. CIC is end-point reveal only.
- Don't retry \`/followup\` on a stalled-RUNNING Bard — respawn per [SB:roster->spawnAgent].
- Don't bulk-replace AUTHOR_SURFACE leaks via global find-replace — per-occurrence judgment per [SB:BryanBlog->refBranding].
- Don't ship literal \`[Diagram: ...]\` text — diagram fill is mandatory before disk write.

## Cross-refs

- [[dispatchCursor]] [[monitorCursor]] [[refDispatchAPI]] [[refCursorSandboxBehavior]] [[refCredentialHandoff]] [[refCursorRulesTemplate]] [[refReportingContract]] [[refCursorVisualVerify]] [[chaperoneMerge]] [[refAntipatterns]]
- [SB:BryanBlog->adoptCJGEORawContent, refDesign, refStrategy, refBranding, auditUnresearched, reAdoptCJGEO]
- [SB:cjgeo->refApiActions, refFullAutoGate, refDataForSEOSeedStrategy, keywordFullAuto, topicFullAuto, generateDraft, pollDraft, adoptDraft]
- [SB:supervisedEvolution->run, refMethod, continueSERun]
- [SB:DungeonMaster->authorSERubric, refLayoutContract]
- [SB:questmaster->summarizeRelevantContext, evaluate, SMARTClarification, upsertQuest, searchQuestBySimilarName]
- [SB:roster->spawnAgent]
- [SB:cloudstonedesigns->adoptCJGEOPage] — multi-site mirror precedent
- [SB:sage->askSage] — gate every would-be question + every would-be success claim
- [SB:browsercontrol] — end-point CIC reveal only
- [SB:dailies->mergeInsight] — closing finale substrate patches
`.trim(),
    },

    refRepoLayout: {
      description:
        "Reference: where things live for cursorCJGEO. Quick map across GuildOS + bryanblog + Cursor surfaces.",
      howTo: `
**GuildOS-side (this skill book and supporting infrastructure):**

\`\`\`
C:\\Users\\xsj70\\GuildOS\\
  libs/skill_book/cursorCJGEO/
    index.js                          # this file
    .cursorrules.template             # proven template, byte-synced with bryanblog
  libs/skill_book/BryanBlog/index.js  # adoption procedure, design rules
  libs/skill_book/cjgeo/index.js      # CJGEO HTTP wire spec (refApiActions)
  libs/skill_book/browsercontrol/index.js  # CIC validate + screenshot procedure
  libs/skill_book/worker/index.js     # broader sub-agent dispatch context
  libs/weapon/cursor/index.js         # high-level Cursor API wrapper
  app/api/cloud_agents/route.js       # localhost:3002 proxy for Cursor + Claude + Codex
\`\`\`

**bryanblog-side (the Cursor agent's working tree):**

\`\`\`
C:\\Users\\xsj70\\bryanblog\\
  .cursorrules                        # operating manual the agent reads on cold start
  .env.local                          # gitignored; populated by the agent from a followup
  scripts/adopt-portfolio-planning.mjs # canonical reference hydration script
  scripts/hydrate-portfolio-planning-2.mjs  # proven-run helper script, reusable starting point
  content/retirement-income/<slug>.html     # CJGEO-hydrated bodies
  app/retirement-income/<slug>/page.tsx     # async server components per satellite
  lib/articles.ts                     # SATELLITES + PILLAR metadata
\`\`\`

**Cursor surfaces:**
- API base: \`https://api.cursor.com/v0/agents\`
- Local proxy: \`http://localhost:3002/api/cloud_agents\`
- Human-facing agent view: \`https://cursor.com/agents?id=<id>\`

**Live deployment:**
- bryanblog: https://<REDACTED-HOST>/
- Auto-deploys on push to \`main\` on \`boster00/bryanblog\`.

**Cross-refs:**
- [SB:BryanBlog->adoptCJGEORawContent] — the adoption procedure the agent
  follows inside the bryanblog repo (also captured verbatim in
  \`.cursorrules\` sections 6–10 because the cloud VM can't read skill books).
- [SB:BryanBlog->refDesign] — design rules; CIC verify against these.
- [SB:cjgeo->refApiActions] — CJGEO HTTP wire spec; the agent calls these
  directly (also captured in \`.cursorrules\` section 7 for the same reason).
- [SB:browsercontrol->validateCIC] — the user-channel verification gate.
- [SB:worker->refContextHygiene] — broader sub-agent dispatch context.
`.trim(),
    },
  },
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/cursor/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/weapon/cursor`

Rationale layer (per CLAUDE.md §6). The WHY behind Cursor dispatch + model policy.
Append; supersede in place when a decision changes.

---

## 2026-06-24 — Cursor is a CHEAP worker (Composer only); NEVER a bridge for Claude. Pin the model, never omit it. (operator directive, LOCKED)

**The decision.** We hold a **Claude 20X Max** subscription. Therefore:
- ALL Claude work runs on that flat-rate subscription via `claudecli` (OAuth login,
  `ANTHROPIC_API_KEY` stripped so it uses the Max plan, NOT API credits).
- Cursor is used ONLY as a **cheap worker on its own in-house Composer model**
  (`composer-2.5`). It is a *cheaper-than-Claude* second engine, valued for Composer.
- We **NEVER route a premium frontier model (claude-\*, gpt-\*, gemini-\*, grok-\*)
  THROUGH Cursor.** Doing so ("using Cursor as a bridge for Claude") both (a) bills
  Cursor's metered API-model pool — Pro's "$200 API model usage" — and (b) double-pays
  for Claude we already hold flat-rate. It is the worst of both billing worlds.

**The incident this fixes (2026-06-24).** A process burned **$200** of Cursor's
metered API-model allowance in one cycle by running premium models through Cursor.
Root cause = every cloud-agent creation path **OMITTED the `model` field**, and the
prior LOCKED guidance (`cursorCJGEO` skill book, "OMIT the model; Cursor picks its
current default") actively *taught* that omission. Cursor's configured default for
background agents is a **premium frontier model** (the recommended set from
`GET /v0/models` is `composer-2.5` + `claude-opus-4-8-*`, `gpt-5.5-*`, `gpt-5.3-codex-*`
— only `composer-2.5` is cheap). So every omitted-model dispatch silently ran premium.
The `nudgeCICPolice` cron heartbeat (`writeFollowup` every ~10 min, which makes the
agent RUN on its creation model) kept a premium agent burning around the clock.

**The fix (structural — make the violation impossible, not merely discouraged).**
1. `libs/weapon/cursor/model.js` — single source of truth: `CURSOR_CHEAP_MODEL`
   (`composer-2.5`) + `resolveCheapCursorModel(id)` which returns the cheap default
   when none is given and **THROWS on any non-`composer-*` id** (including `auto`,
   which can route premium) unless `CURSOR_ALLOW_PREMIUM=1` (the audited operator
   escape hatch).
2. Every creation surface now PINS the model via that guard, shape
   `model: { id: "composer-2.5" }` (a `ModelRef` OBJECT per the cloud-agents OpenAPI —
   NOT a top-level string): `cursor.writeAgent`, the local proxy
   `app/api/cloud_agents/route.js` `cursor_setup`, and the `cursorcli` weapon
   (local CLI `--model`).
3. The `cursorCJGEO` "OMIT model" LOCKED note is **superseded** in place — it was the
   discovery-layer substrate that propagated the bug.

**Why pin at CREATION specifically.** A Cursor agent's model is fixed when the agent
is created (`POST /v0/agents`); `followup` calls inherit it and take no model field. So
pinning `writeAgent`/`cursor_setup` is the load-bearing fix — it makes every NEW agent
(and every cron auto-respawn, which goes through `writeAgent`) cheap. Existing agents
created BEFORE this fix keep their premium model until they expire and are recreated.

**Rejected alternatives.**
- *Omit the model and trust Cursor's default* (the prior rule) — that default is
  premium; the omission was the bug.
- *Pass `model` as a top-level string* (the original `composer-2.0` that 400'd) — wrong
  shape AND stale id. The 400 was a shape/staleness bug, never a reason to omit; the
  correct shape is the `{ id }` ModelRef.
- *A doc note saying "use cheap models"* — discouragement is not enforcement; a future
  dispatch would omit the model again. The `resolveCheapCursorModel` throw is the wall.
- *`--model auto` for the CLI* (cursorcli's first default) — `auto` can route to a
  premium frontier model, so it violates "cheap only." Blocked by the guard.

**Consequence if violated.** Re-introduce an omitted/`auto`/premium model on any Cursor
dispatch and the metered API pool burns again (the exact $200 incident), while we keep
paying flat-rate for the same Claude capacity on Max — paying twice to go slower-and-
more-expensive than just using `claudecli`.

**Open follow-ups (NOT a code gap — operational).**
- *Existing premium cloud-agent sessions* (the 8 `cursor_cloud` adventurers created
  pre-fix) still carry a premium creation-model. They are currently halted by the
  account's exhausted budget. They auto-heal to `composer-2.5` when recreated (creation
  is now pinned), but until then the CIC-Police heartbeat would resume premium runs once
  budget returns. Retire/recreate them cheap after the budget resets (or set a spend
  limit) — surfaced to the operator rather than nulling live session handles autonomously.
- *Live re-verify the cloud `model:{id}` shape* once budget allows a real
  `POST /v0/agents` (the shape is OpenAPI-confirmed but could not be live-tested while
  the account was out of usage).
- *Account identity*: the headless `CURSOR_API_KEY` (key name "GuildOS") resolves to
  `<REDACTED-EMAIL>` via `GET /v0/me` — the user's own account, confirmed not a
  wrong-account artifact.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/cursor/index.js -->
```js
/**
 * Cursor weapon — Cursor Cloud Agent dispatch and management via Cursor API.
 *
 * Auth: CURSOR_API_KEY from profiles.env_vars or process.env.
 * Endpoint: https://api.cursor.com/v0/agents
 * Model: PINNED to Cursor's cheap Composer model via resolveCheapCursorModel
 *   (model.js). NEVER omit it — omitting makes Cursor pick its PREMIUM frontier
 *   default (a premium claude or gpt model), which bills the metered API pool and
 *   double-pays for Claude we already hold flat-rate on Claude Max. Burned $200 in one
 *   cycle (2026-06-24). The correct body shape is `model: { id: "composer-2.5" }`
 *   (ModelRef per the cloud-agents OpenAPI), NOT a top-level string (the stale
 *   `composer-2.0` string was the old HTTP-400; that note is now superseded).
 */

export const toc = {
  readAgent: "Read a Cursor cloud agent's status and metadata.",
  readConversation: "Read a Cursor agent's full conversation history.",
  writeFollowup: "Send a followup message to a Cursor agent (push-based dispatch).",
  searchAgents: "Search Cursor agents (wraps readAgent; API has no list endpoint).",
  readQueuedMessages: "Read queued user messages sent after the last assistant reply.",
  writeAgent: "Create a new Cursor cloud agent for a given repository.",
  readEnvSetupInstructions: "Generate an env setup script to bootstrap a fresh Cursor agent session.",
  syncSessionStatus: "Probe an adventurer's upstream Cursor session status and reconcile the adventurers DB row. Returns { upstream_status, alive, dispatch_safe, was_drift }.",
  // --- Agent-facing actions (skillActionOk/Err envelope; (userId, input) signature) ---
  dispatchTask: "Agent-facing: send a task to a Cursor cloud agent via followup. Input { agentId, instructions }. A followup runs the agent on its CREATION model (pinned cheap by writeAgent/cursor_setup) — the model input is inert; choose the model at creation, not here.",
  readStatus: "Agent-facing: check a Cursor cloud agent's status. Input { agentId }.",
  readConversationAction: "Agent-facing: read a Cursor agent's full conversation. Input { agentId }.",
  dispatchPptGeneration: "Agent-facing: dispatch a PowerPoint-generation task to a Cursor cloud agent. Input { agentId, topic, slides?, questId? }.",
};
import { database } from "@/libs/council/database";
import { guildosCredentialsSetupBlock } from "./setupBlock.js";
import { resolveCheapCursorModel } from "./model.js";
import { skillActionOk, skillActionErr } from "@/libs/skill_book/actionResult.js";
import { getAdventurerExecutionContext } from "@/libs/adventurer/advance.js";

const API_BASE = "https://api.cursor.com/v0/agents";

// ---------------------------------------------------------------------------
// Auth
// ---------------------------------------------------------------------------

async function getApiKey(userId) {
  if (process.env.CURSOR_API_KEY) return process.env.CURSOR_API_KEY;
  if (!userId) return null;
  const db = await database.init("service");
  const { data } = await db
    .from("profiles")
    .select("env_vars")
    .eq("id", userId)
    .single();
  return data?.env_vars?.CURSOR_API_KEY ?? null;
}

function makeAuthHeader(apiKey) {
  return `Basic ${Buffer.from(`${apiKey}:`).toString("base64")}`;
}

async function cursorFetch(path, opts = {}, userId) {
  const key = await getApiKey(userId);
  if (!key) throw new Error("Missing CURSOR_API_KEY");
  const url = path.startsWith("http") ? path : `${API_BASE}${path}`;
  const res = await fetch(url, {
    ...opts,
    headers: {
      Authorization: makeAuthHeader(key),
      "Content-Type": "application/json",
      ...opts.headers,
    },
  });
  if (!res.ok) {
    const body = await res.text();
    throw new Error(`Cursor API ${res.status}: ${body.slice(0, 500)}`);
  }
  return res.json();
}

// ---------------------------------------------------------------------------
// Agent management
// ---------------------------------------------------------------------------

/**
 * Read agent status.
 * @param {{ agentId: string }} input
 * @param {string} [userId]
 */
export async function readAgent({ agentId } = {}, userId) {
  if (!agentId) throw new Error("agentId is required");
  return cursorFetch(`/${agentId}`, {}, userId);
}

/**
 * Read agent conversation history.
 * @param {{ agentId: string }} input
 * @param {string} [userId]
 */
export async function readConversation({ agentId } = {}, userId) {
  if (!agentId) throw new Error("agentId is required");
  return cursorFetch(`/${agentId}/conversation`, {}, userId);
}

/**
 * Send a followup message to an agent (push-based dispatch).
 * @param {{ agentId: string, message: string, model?: string }} input
 * @param {string} [userId]
 */
export async function writeFollowup({ agentId, message, model } = {}, userId) {
  if (!agentId) throw new Error("agentId is required");
  if (!message) throw new Error("message is required");
  return cursorFetch(
    `/${agentId}/followup`,
    {
      method: "POST",
      body: JSON.stringify({
        prompt: { text: message },
      }),
    },
    userId,
  );
}

/**
 * Search for agents by partial ID or metadata.
 * Note: Cursor API doesn't have a list endpoint — this reads a known agent.
 * For now, wraps readAgent. Extend when API adds list support.
 * @param {{ agentId: string }} input
 * @param {string} [userId]
 */
export async function searchAgents({ agentId } = {}, userId) {
  if (!agentId) throw new Error("agentId is required for search (Cursor API has no list endpoint)");
  const agent = await readAgent({ agentId }, userId);
  return { agents: [agent] };
}

/**
 * Read queued (unprocessed) messages — user messages sent after the last assistant response.
 * @param {{ agentId: string }} input
 * @param {string} [userId]
 */
export async function readQueuedMessages({ agentId } = {}, userId) {
  if (!agentId) throw new Error("agentId is required");
  const conv = await readConversation({ agentId }, userId);
  const msgs = conv?.messages || [];
  const lastAssistantIdx = msgs.findLastIndex((m) => m.type === "assistant_message");
  return { queued: msgs.slice(lastAssistantIdx + 1).filter((m) => m.type === "user_message") };
}

/**
 * Create a new Cursor cloud agent.
 * @param {{ repository: string, ref: string, prompt?: string }} input
 * @param {string} [userId]
 */
/**
 * Spawn a new Cursor cloud agent. ALWAYS provisions GuildOS service-role
 * credentials in the spawn prompt as the first action, so the agent never
 * hits the documented "missing GuildOS Supabase credentials" blocker that
 * caused the 2026-04-26 ptglab smoke-test to escalate without bookkeeping.
 *
 * Contract: this is the ONLY supported path for creating an agent. Direct
 * POSTs to the Cursor API that skip the credentials-injection block are a
 * contract violation — agents spawned that way cannot complete GuildOS-
 * side bookkeeping (items rows, quest_comments, submitForPurrview).
 *
 * Credential source: process.env (the Guildmaster running this weapon must
 * have GuildOS env loaded — `--env-file=.env.local` or equivalent).
 *
 * @param {{ repository: string, ref: string, prompt?: string,
 *           skipCredentialsInjection?: boolean }} input
 * @param {string} [userId]
 */
export async function writeAgent({ repository, ref, prompt, skipCredentialsInjection = false, model } = {}, userId) {
  if (!repository) throw new Error("repository is required");
  if (!ref) throw new Error("ref is required");
  const key = await getApiKey(userId);
  if (!key) throw new Error("Missing CURSOR_API_KEY");
  // Cheap-worker policy: pin Cursor's Composer model (premium throws unless
  // CURSOR_ALLOW_PREMIUM=1). NEVER omit — omission = Cursor's premium default.
  const modelId = resolveCheapCursorModel(model);

  let finalPrompt = prompt || "Wait for initialization instructions.";

  if (!skipCredentialsInjection) {
    // Prefer the GUILDOS_-prefixed names so the GuildOS project is unambiguous
    // even when the orchestrator's env is also wired to a project's own
    // Supabase. Falls back to the basic names for backward compat.
    const guildosUrl =
      process.env.GUILDOS_NEXT_PUBLIC_SUPABASE_URL ||
      process.env.NEXT_PUBLIC_SUPABASE_URL;
    const guildosKey =
      process.env.GUILDOS_SUPABASE_SECRETE_KEY ||
      process.env.GUILDOS_SUPABASE_SECRET_KEY ||
      process.env.SUPABASE_SECRETE_KEY ||
      process.env.SUPABASE_SECRET_KEY;
    if (!guildosUrl || !guildosKey) {
      throw new Error(
        "writeAgent: GuildOS credentials missing from process.env. Required: GUILDOS_NEXT_PUBLIC_SUPABASE_URL + GUILDOS_SUPABASE_SECRETE_KEY (or the basic NEXT_PUBLIC_SUPABASE_URL + SUPABASE_SECRETE_KEY for backward compat). The agent cannot do GuildOS bookkeeping without these. Run with `--env-file=.env.local` or pass `skipCredentialsInjection: true` if you genuinely want a creds-less agent (rare — almost always wrong)."
      );
    }
    finalPrompt = guildosCredentialsSetupBlock({ supabaseUrl: guildosUrl, supabaseKey: guildosKey }) + finalPrompt;
  }

  const res = await fetch(API_BASE, {
    method: "POST",
    headers: {
      Authorization: makeAuthHeader(key),
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      prompt: { text: finalPrompt },
      source: { repository, ref },
      model: { id: modelId },
    }),
  });
  if (!res.ok) {
    const body = await res.text();
    throw new Error(`Cursor API ${res.status}: ${body.slice(0, 500)}`);
  }
  return res.json();
}

// ---------------------------------------------------------------------------
// Session-health probe — pre-dispatch check that the upstream Cursor session
// matches what the adventurers DB row says, and is in a state that can accept
// a followup. Reconciles drift (DB says idle, upstream says EXPIRED, etc.).
// ---------------------------------------------------------------------------

/**
 * Probe one adventurer's session against the Cursor API and reconcile the DB.
 *
 * Returns:
 *   {
 *     upstream_status: 'RUNNING' | 'FINISHED' | 'EXPIRED' | 'CREATING' | 'ERROR' | <other>,
 *     alive: boolean,            // upstream session exists at all
 *     dispatch_safe: boolean,    // followup is expected to land (FINISHED/RUNNING/CREATING ok; EXPIRED not)
 *     was_drift: boolean,        // db.session_status differed from upstream
 *     adventurer: { id, name, session_id, session_status }, // post-sync row
 *   }
 *
 * Drift handling: if upstream returns a status the local DB doesn't reflect
 * accurately, this updates `adventurers.session_status` to a normalized
 * "idle" (dispatch-safe) or "expired" (needs respawn) value. Does NOT auto-
 * respawn — caller decides.
 *
 * @param {{ adventurerId?: string, adventurerName?: string }} input — pass one
 * @param {string} [userId]
 */
export async function syncSessionStatus({ adventurerId, adventurerName } = {}, userId) {
  if (!adventurerId && !adventurerName) {
    throw new Error("syncSessionStatus: pass adventurerId or adventurerName");
  }
  const db = await database.init("service");
  const q = db.from("adventurers").select("id, name, session_id, session_status, worker_type").limit(1);
  const { data: rows } = await (adventurerId ? q.eq("id", adventurerId) : q.eq("name", adventurerName));
  const adv = rows?.[0];
  if (!adv) throw new Error(`syncSessionStatus: adventurer not found (${adventurerId || adventurerName})`);
  if (!adv.session_id) {
    return {
      upstream_status: "no_session",
      alive: false,
      dispatch_safe: false,
      was_drift: false,
      adventurer: adv,
    };
  }

  let upstreamStatus = "unknown";
  let alive = false;
  try {
    const j = await cursorFetch(`/${adv.session_id}`, {}, userId);
    upstreamStatus = String(j?.status || "unknown");
    alive = true;
  } catch (err) {
    // Cursor API returns 404 / 410 for hard-deleted sessions; surface that
    upstreamStatus = "deleted_or_unreachable";
    alive = false;
  }

  // Normalize to DB-friendly status. dispatch_safe: can a followup land?
  const dispatch_safe =
    alive && (upstreamStatus === "RUNNING" || upstreamStatus === "FINISHED" || upstreamStatus === "CREATING");
  const desiredDbStatus = dispatch_safe
    ? "idle"
    : upstreamStatus === "EXPIRED"
      ? "expired"
      : "unreachable";

  const was_drift = adv.session_status !== desiredDbStatus;
  if (was_drift) {
    await db.from("adventurers").update({ session_status: desiredDbStatus }).eq("id", adv.id);
  }

  return {
    upstream_status: upstreamStatus,
    alive,
    dispatch_safe,
    was_drift,
    adventurer: { ...adv, session_status: desiredDbStatus },
  };
}

// ---------------------------------------------------------------------------
// Environment setup helper (for fresh agent sessions)
// ---------------------------------------------------------------------------

/**
 * Generate the env setup instructions for a fresh Cursor agent session.
 * Reads required env vars from profiles and formats them for the agent.
 * @param {{ userId: string, envVarKeys?: string[] }} input
 */
export async function readEnvSetupInstructions({ userId, envVarKeys } = {}) {
  if (!userId) throw new Error("userId is required");
  const db = await database.init("service");
  const { data } = await db
    .from("profiles")
    .select("env_vars")
    .eq("id", userId)
    .single();

  const ev = data?.env_vars || {};
  const keys = Array.isArray(envVarKeys) && envVarKeys.length > 0
    ? envVarKeys
    : [
        "NEXT_PUBLIC_SUPABASE_URL",
        "NEXT_PUBLIC_SUPABASE_ANON_KEY",
        "SUPABASE_SECRETE_KEY",
        "OPENAI_API_KEY",
        "PIGEON_API_KEY",
        "PIGEON_POST_OWNER_ID",
      ];

  const lines = [];
  for (const k of keys) {
    const val = ev[k] || process.env[k] || "";
    if (val) lines.push(`${k}=${val}`);
  }

  return {
    envFileContent: lines.join("\n"),
    setupScript: [
      "npm install && npx playwright install chromium --with-deps && npm install -g @anthropic-ai/claude-code",
      `cat > .env.local << 'ENVEOF'\n${lines.join("\n")}\nENVEOF`,
    ].join("\n"),
  };
}

// ---------------------------------------------------------------------------
// Credential check
// ---------------------------------------------------------------------------

export async function checkCredentials(userId) {
  try {
    const key = await getApiKey(userId);
    if (key) {
      return { ok: true, msg: "CURSOR_API_KEY is set" };
    }
    return {
      ok: false,
      msg: "Missing CURSOR_API_KEY — add it to your profile env_vars in Council Hall > Formulary.",
    };
  } catch (e) {
    return { ok: false, msg: `Error: ${e.message}` };
  }
}

// ---------------------------------------------------------------------------
// Agent-facing wrappers — (userId, input) signature, skillActionOk/Err envelope.
// These are the methods the `cursor` skill book's toc points at. They were moved
// out of libs/skill_book/cursor/index.js so the skill book stays a pure prompt
// registry. The underlying logic delegates to the weapon functions above.
// ---------------------------------------------------------------------------

function resolveUserId(explicitUserId) {
  const ctx = getAdventurerExecutionContext();
  return explicitUserId || ctx?.userId || null;
}

/**
 * Send a task to a Cursor cloud agent via followup message.
 * NOTE: a followup runs the agent on the model it was CREATED with (the API followup
 * body takes no model field), so the `model` input here is inert — model selection
 * happens at creation (writeAgent / cursor_setup), which is PINNED cheap. If the agent
 * was created cheap (post-2026-06-24), the followup runs cheap; a pre-fix premium agent
 * stays premium until recreated. See libs/weapon/cursor/model.js + engineering_decisions.md.
 * @param {string} [_userId]
 * @param {{ agentId?: string, instructions?: string, model?: string }} input
 */
export async function dispatchTask(_userId, input) {
  const inObj = typeof input === "object" && input ? input : {};
  const agentId = String(inObj.agentId || "").trim();
  const instructions = String(inObj.instructions || "").trim();
  const model = String(inObj.model || "").trim() || undefined;

  if (!agentId) return skillActionErr("agentId is required");
  if (!instructions) return skillActionErr("instructions are required");

  const userId = resolveUserId(_userId);
  try {
    const result = await writeFollowup({ agentId, message: instructions, model }, userId);
    return skillActionOk({
      items: {
        result: JSON.stringify(result),
        agentId,
      },
      msg: `Task dispatched to agent ${agentId}`,
    });
  } catch (e) {
    return skillActionErr(`Dispatch failed: ${e.message}`);
  }
}

/**
 * Check the current status of a Cursor cloud agent.
 * @param {string} [_userId]
 * @param {{ agentId?: string }} input
 */
export async function readStatus(_userId, input) {
  const inObj = typeof input === "object" && input ? input : {};
  const agentId = String(inObj.agentId || "").trim();
  if (!agentId) return skillActionErr("agentId is required");

  const userId = resolveUserId(_userId);
  try {
    const agent = await readAgent({ agentId }, userId);
    return skillActionOk({
      items: {
        status: JSON.stringify(agent.status || "unknown"),
        agent: JSON.stringify(agent),
      },
      msg: `Agent ${agentId} status: ${agent.status || "unknown"}`,
    });
  } catch (e) {
    return skillActionErr(`Status check failed: ${e.message}`);
  }
}

/**
 * Read the full conversation history of a Cursor cloud agent.
 * (Named *Action to avoid clashing with the lower-level `readConversation` API wrapper.)
 * @param {string} [_userId]
 * @param {{ agentId?: string }} input
 */
export async function readConversationAction(_userId, input) {
  const inObj = typeof input === "object" && input ? input : {};
  const agentId = String(inObj.agentId || "").trim();
  if (!agentId) return skillActionErr("agentId is required");

  const userId = resolveUserId(_userId);
  try {
    const conv = await readConversation({ agentId }, userId);
    return skillActionOk({
      items: { conversation: JSON.stringify(conv) },
      msg: `Retrieved conversation for agent ${agentId}`,
    });
  } catch (e) {
    return skillActionErr(`Conversation read failed: ${e.message}`);
  }
}

/**
 * Dispatch a PowerPoint-generation task to a Cursor cloud agent. The agent
 * builds the .pptx and uploads it to Supabase Storage under cursor_cloud/.
 * @param {string} [_userId]
 * @param {{ agentId?: string, topic?: string, slides?: string, questId?: string }} input
 */
export async function dispatchPptGeneration(_userId, input) {
  const inObj = typeof input === "object" && input ? input : {};
  const agentId = String(inObj.agentId || "").trim();
  const topic = String(inObj.topic || "").trim();
  const slides = String(inObj.slides || "").trim();
  const questId = String(inObj.questId || "").trim();

  if (!agentId) return skillActionErr("agentId is required");
  if (!topic) return skillActionErr("topic is required");

  const userId = resolveUserId(_userId);
  const instructions = [
    `Generate a PowerPoint presentation about: ${topic}`,
    slides ? `\nSlide content:\n${slides}` : "",
    "",
    "Requirements:",
    "- Use pptxgenjs or python-pptx to create the PPT file",
    "- Upload the final .pptx to Supabase Storage at:",
    questId
      ? `  cursor_cloud/${questId}/${topic.replace(/[^a-zA-Z0-9]/g, "-").toLowerCase()}.pptx`
      : `  cursor_cloud/general/${topic.replace(/[^a-zA-Z0-9]/g, "-").toLowerCase()}.pptx`,
    "- Post the public URL as a quest comment",
    "- Take a screenshot of at least one slide as visual proof",
    "",
    "After generating, git push your changes.",
  ].join("\n");

  try {
    const result = await writeFollowup({ agentId, message: instructions }, userId);
    return skillActionOk({
      items: {
        result: JSON.stringify(result),
        agentId,
      },
      msg: `PPT generation dispatched to agent ${agentId}: "${topic}"`,
    });
  } catch (e) {
    return skillActionErr(`PPT dispatch failed: ${e.message}`);
  }
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/cursor/model.js -->
```js
/**
 * Cursor model policy — the single source of truth for "Cursor = cheap worker only".
 * Pure module, NO imports, so it is consumable identically from the Next runtime, a
 * Next API route, and a plain-node leaf (cursorcli) without adapter shims.
 *
 * DECISION (2026-06-24, operator directive): we hold a Claude 20X Max subscription,
 * so ALL Claude work runs on that flat-rate subscription via claudecli (OAuth, no API
 * credits). Cursor is used ONLY as a CHEAP worker on its own in-house Composer model.
 * We NEVER route a premium frontier model (Claude/GPT/Gemini/Grok) THROUGH Cursor —
 * Cursor bills those against its metered API-model pool ("$200 API model usage") AND
 * it double-pays for Claude we already have flat-rate ("using Cursor as a bridge for
 * Claude"). Omitting the model on a Cursor dispatch makes Cursor pick its PREMIUM
 * default — that omission is what burned $200 in one cycle (the bug this enforces away).
 * Full rationale: libs/weapon/cursor/engineering_decisions.md.
 */

/** Cursor's cheap in-house worker model id (from GET /v0/models — the only non-premium
 *  entry in the recommended set). Env-overridable, but it must stay a composer-* id. */
export const CURSOR_CHEAP_MODEL = process.env.CURSOR_CHEAP_MODEL || "composer-2.5";

/** Premium = anything that is NOT Cursor's own composer-* model. Everything else in
 *  Cursor's recommended list (claude-*, gpt-*, gemini-*, grok-*, and "auto", which may
 *  route to a frontier model) bills the metered API pool. */
export function isPremiumCursorModel(id) {
  const s = String(id || "").trim().toLowerCase();
  if (!s) return false; // empty/omitted → the caller substitutes the cheap default
  return !s.startsWith("composer");
}

/**
 * Resolve + ENFORCE the model for any Cursor dispatch. Returns the cheap default when
 * none is given; THROWS on a premium id unless `CURSOR_ALLOW_PREMIUM=1` is set — the
 * explicit, auditable operator escape hatch for a deliberate one-off. This is the
 * structural guard that makes "Cursor never bridges to Claude" impossible to violate by
 * accident (an omitted/defaulted/auto model can no longer silently land on a frontier model).
 *
 * @param {string} [id] requested model id (may be undefined/empty)
 * @returns {string} a composer-* model id (or the premium id if the escape hatch is set)
 */
export function resolveCheapCursorModel(id) {
  const chosen = (id && String(id).trim()) || CURSOR_CHEAP_MODEL;
  if (isPremiumCursorModel(chosen) && process.env.CURSOR_ALLOW_PREMIUM !== "1") {
    throw new Error(
      `Cursor model "${chosen}" is a premium (metered) model. POLICY: Cursor runs ONLY its cheap Composer ` +
        `model; Claude work goes through Claude Max (claudecli). Pass a composer-* id, set CURSOR_CHEAP_MODEL, ` +
        `or — for a deliberate, audited one-off — set CURSOR_ALLOW_PREMIUM=1. ` +
        `See libs/weapon/cursor/engineering_decisions.md.`,
    );
  }
  return chosen;
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/cursor/setupBlock.js -->
```js
/**
 * Pure-JS module: builds the GuildOS-credentials shell-setup block that
 * `cursor.writeAgent` prepends to every spawn prompt.
 *
 * Extracted so node scripts (which can't resolve `@/libs/...` Next path
 * aliases) and the Next-runtime weapon both consume the same source.
 * No imports — keep it that way so this module is consumable from any
 * runtime without adapter shims.
 */

/**
 * @param {{ supabaseUrl: string, supabaseKey: string }} input
 * @returns {string} shell-setup block to prepend to a Cursor agent's spawn prompt
 */
export function guildosCredentialsSetupBlock({ supabaseUrl, supabaseKey } = {}) {
  if (!supabaseUrl || !supabaseKey) {
    throw new Error(
      "guildosCredentialsSetupBlock: supabaseUrl + supabaseKey are required. " +
      "Caller is responsible for sourcing them from process.env (NEXT_PUBLIC_SUPABASE_URL + SUPABASE_SECRETE_KEY) " +
      "or profiles.env_vars before invoking the spawn."
    );
  }
  return [
    "═════════════════════════════════════════════════════════════════════",
    "FIRST ACTION — provision GuildOS credentials (do this BEFORE anything else)",
    "═════════════════════════════════════════════════════════════════════",
    "",
    "Set these env vars so GuildOS bookkeeping works regardless of which",
    "project repo you're working on. Both the GUILDOS_-prefixed names",
    "(canonical when working in multi-repo envs) AND the basic names",
    "(backward compat) are exported, so libs that read either form work.",
    "",
    "```bash",
    "mkdir -p ~/guildos",
    "cat > ~/.guildos.env <<'EOF'",
    `export GUILDOS_NEXT_PUBLIC_SUPABASE_URL='${supabaseUrl}'`,
    `export GUILDOS_SUPABASE_SECRETE_KEY='${supabaseKey}'`,
    `export NEXT_PUBLIC_SUPABASE_URL='${supabaseUrl}'`,
    `export SUPABASE_SECRETE_KEY='${supabaseKey}'`,
    "EOF",
    "echo 'source ~/.guildos.env' >> ~/.bashrc",
    "source ~/.guildos.env",
    "test -n \"$GUILDOS_SUPABASE_SECRETE_KEY\" && echo 'GuildOS creds loaded' || echo 'FAILED — escalate'",
    "```",
    "",
    "After that, run housekeeping.initAgent (clones GuildOS, loads skill books, finds your quests). The init step assumes the env is already loaded.",
    "",
    "═════════════════════════════════════════════════════════════════════",
    "TASK PROMPT",
    "═════════════════════════════════════════════════════════════════════",
    "",
  ].join("\n");
}
```
<!-- /GUILDOS-FILE -->
