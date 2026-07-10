# BryanBlog

<!-- GUILDOS-SKILL name="BryanBlog" -->

<!-- GUILDOS-DESC -->
Maintain and extend the bryanblog static Next.js content site: add pillars/satellites, recolor, deploy, and (forward-link) replace native articles with CJGEO-generated content. Covers the 7 locked editorial/strategy decisions, the design/palette contract, the CJGEO adoption pipelines (sanitize/anchor/deep-link/diagram-fill and the high-fidelity chrome-preserving variant), the "Mysterious Author" branding lock, and ICP-journey mapping that feeds new pillars.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/BryanBlog/SKILL.md -->
```markdown
---
name: BryanBlog
description: >-
  Maintain and extend the bryanblog static Next.js content site: add
  pillars/satellites, recolor, deploy, and (forward-link) replace native
  articles with CJGEO-generated content. Covers the 7 locked editorial/strategy
  decisions, the design/palette contract, the CJGEO adoption pipelines
  (sanitize/anchor/deep-link/diagram-fill and the high-fidelity chrome-preserving
  variant), the "Mysterious Author" branding lock, and ICP-journey mapping that
  feeds new pillars.
---

# BryanBlog — Bryan's Retirement-Planning Content Site

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/BryanBlog/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto BryanBlog <action>` (or read the source directly).

## Actions

- **addCta** — Add a CTA block to a page. The `<Cta>` component already handles the `not-prose` wrapping; place sparingly.
- **addPillar** — Add a new decision-journey pillar with its satellites. Heavyweight — plan the whole journey before writing.
- **addSatellite** — Add a new satellite article under an existing pillar. The high-frequency operation on bryanblog.
- **adoptCJGEOHighFidelity** — LOCKED canonical pipeline: adopt CJGEO content into a destination repo at HIGH FIDELITY by preserving CJGEO's own hero / chrome / design treatment, NOT stripping it. Per-article scoped <style> block + raw HTML + bypass destination's ArticleShell. Proven 2026-05-26 across all 26 cloudstonedesigns satellites achieving ~90-95% visual parity to CJGEO preview.
- **adoptCJGEORawContent** — Canonical pipeline: CJGEO raw article HTML → bryanblog-aligned page. Sanitize, anchor, deep-link, write to `content/`, rewire the route as an async server component, build, push, CIC-verify.
- **auditUnresearched** — Audit the bryanblog satellite roster against CJGEO assets to classify articles by what's missing (keywords / topics / both / neither). The triage pattern for batch regeneration.
- **deployToVercel** — Deploy or redeploy the site to Vercel. Auto-deploys on push to main; manual paths documented for when the autopath fails.
- **mapICPJourney** — Research action: convert raw ICP insight into scenario-based content journeys. Extract customer scenarios → cluster into journey families → decompose into 8 stages (Trigger / Possibility / Options / Comparison / Risk / Process / Expert Selection / Action) → phrase every topic as a natural-language LLM question → emit pillar + satellite architecture + conversion map. Feeds [SB:BryanBlog->addPillar] / [SB:BryanBlog->addSatellite]. Run BEFORE drafting any pillar.
- **reAdoptCJGEO** — Re-adopt an EXISTING CJGEO article into bryanblog without creating a new article — pulls content_html from the existing article_id and re-runs sanitize / anchor / fill / verify. The fidelity-uplift pattern.
- **recolor** — Change the palette. The whole site reskin lives in one `@theme` block.
- **refBranding** — Reference (LOCKED): bryanblog's branding constraint — the author surface is 'The Mysterious Author', never 'Bryan'. Every CJGEO context_prompt must include this; every adoption must grep for leaks before push.
- **refDesign** — Reference: palette, typography, framework choices, and the locked `not-prose` rule that makes nested components work inside `.prose-article`. Read before any structural CSS change.
- **refRepoLayout** — Reference: where things live in the bryanblog repo. Quick map so a fresh agent doesn't have to glob.
- **refStrategy** — Reference: the 7 strategic decisions that shape every editorial + structural choice on bryanblog. Read this first when adding pages, rewriting prose, or changing structure.
- **replaceArticleWithCJGEO** — Forward-reference: replace a native article's prose with CJGEO-generated content. The next phase of the project. Drives via the cjgeo skill book.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/BryanBlog/index.js -->
```js
/**
 * BryanBlog skill book — knowledge registry for the <REDACTED-HOST> static content site.
 *
 * **Project shape.** `bryanblog` is a Next.js 15 (app router, React 19, TS, Tailwind 4) static
 * content site for a retirement-planning advisor (Bryan). It is a SEPARATE repo + Vercel project
 * from GuildOS — owned by `boster00` (<REDACTED-EMAIL>). The site is decision-journey pillars
 * (e.g. "Will my money last in retirement?") with satellite articles that branch off each section,
 * so any landing page — pillar or satellite — gives a reader a clear path forward.
 *
 * **Current state (locked).** The site is live with NATIVE prose. The roadmap is to replace
 * each article one-by-one with CJGEO-generated content. See `replaceArticleWithCJGEO` here +
 * the `cjgeo` skill book for the hydration pipeline.
 *
 * **Browser/CIC discipline lives in the `browsercontrol` skill book** — the canonical
 * GuildOS-wide owner of every CIC procedure (validateCIC, screenshot/navigate loops,
 * cloud-vs-local browser policy). Any visual-verify step in the actions below defers to it;
 * do not restate browser rules here.
 *
 * **Where things live.**
 *  - Repo (local): `C:\Users\xsj70\bryanblog`
 *  - Repo (GitHub): https://github.com/boster00/bryanblog
 *  - Live: https://<REDACTED-HOST>/
 *  - Vercel project: `bryanblog` under `cjs-projects-963e3b09` (account `boster00`)
 *  - Local dev port: 3004 (LOCKED — `npm run dev -- -p 3004`; port 3005 is cloudstonedesigns)
 *
 * **Design language is ported from `seres-immigration`** (Astro 5, at
 * `C:\Users\xsj70\seres-immigration`, live at https://seresimmigration.com/en/). The article
 * idioms (warm-paper bg, drop-cap, h2 auto-underline, sticky desktop TOC on pillars,
 * related-articles block on satellites) were lifted from there. Bespoke seres widgets
 * (folder tabs, comparison panels) were intentionally NOT ported — defer to a follow-up if
 * a future article actually needs them.
 */

export const skillBook = {
  id: "BryanBlog",
  title: "BryanBlog — Bryan's Retirement-Planning Content Site",
  description:
    "Maintain and extend the bryanblog static Next.js content site: add pillars/satellites, recolor, deploy, and (forward-link) replace native articles with CJGEO-generated content.",
  steps: [],
  toc: {
    refStrategy: {
      description:
        "Reference: the 7 strategic decisions that shape every editorial + structural choice on bryanblog. Read this first when adding pages, rewriting prose, or changing structure.",
      howTo: `
The site is built on seven locked strategic decisions. Don't re-litigate them per-article; honor them.

**1. Decision-journey pillars.** Each pillar page maps to ONE consumer decision journey (e.g.
"Will my money last in retirement?"). The pillar covers the journey end-to-end; satellites branch off
each pillar section to go deeper. A reader landing on ANY page — pillar or satellite — should have a
clear path forward. The pillar's 7 H2 sections each become a launch pad to a satellite.

**2. LLM-aligned section headers.** Section H2s are phrased the way a layperson would type into
ChatGPT/Claude/Google, NOT the way an industry expert would label them. The technical concept goes in
a subheader or in the body. Example:

  H2: "How much should I keep safe versus invested?"
  Sub: "Growth assets, safe-money bucket, market volatility, and downside protection."

This is the rule that captures LLM/search intent at distributed entry points.

**3. Evergreen pages, not blog posts.** These articles are structured, thematic, systematic. They
live as evergreen pages with stable URLs under \`/<pillar-slug>/<satellite-slug>/\`. Blogs (a separate
route, NOT YET BUILT) are reserved for one-off topics that don't fit the pillar/satellite system.

**4. Entry points everywhere.** Every section of the pillar can serve as a landing page for LLM/search
traffic via its satellite. Distributed entry points is the goal, not the side effect — it captures
distinct intents.

**5. Compliance-conservative tone.** No product names. No insurer names. No specific return
projections. No uncaveated guarantees. Whenever an annuity guarantee is mentioned, append
"subject to claims-paying ability".

**6. Audience-first language.** 8th-grade reading level. Calm and mature. Never salesy. The reader is
a 55–75-year-old anxious about running out of money — not a financial pro. Don't write past them; don't
talk down to them either.

**7. Soft CTA framing.** Never "buy an annuity". Always "request a retirement income review" —
positioning Bryan as advisor, not salesman. CTAs are sparing; they lose weight if everywhere.

**End-state contract.** Every article — pillar or satellite — must end ready-to-act on. A reader who
lands cold must (a) understand the concept, (b) know what their next decision is, (c) have an obvious,
non-pushy next click (related article OR the soft CTA).
`.trim(),
    },

    refDesign: {
      description:
        "Reference: palette, typography, framework choices, and the locked \`not-prose\` rule that makes nested components work inside \`.prose-article\`. Read before any structural CSS change.",
      howTo: `
**Framework:** Next.js 15.4.8 (app router) · React 19 · TypeScript · Tailwind 4.

**Tailwind 4 specifics:**
- Configured via the \`@theme\` block in \`app/globals.css\` — **no \`tailwind.config.js\`**.
- No \`@tailwindcss/typography\` plugin. No DaisyUI.
- The \`.prose-article\` class is **hand-written** in \`globals.css\` (not the typography plugin).

**Palette (Thrivent-inspired, warmer):**
| Token            | Hex       | Role                                  |
|------------------|-----------|---------------------------------------|
| primary          | \`#1C4259\` | slate teal-navy — headings, links     |
| accent           | \`#8C3A2E\` | muted oxblood/terracotta — CTAs, h2 underline |
| bg               | \`#FAF9F8\` | warm off-white — page background      |
| accent-tint      | \`#F4D9C9\` | terracotta wash — soft callouts       |
| neutral-mid      | \`#B5A99A\` | warm taupe — borders, dividers        |

Body text: \`stone-800\` (Tailwind default scale).

**Typography:**
- Body sans: **Inter** (Google Fonts, weights 400–700).
- Headings serif: **Fraunces** (Google Fonts, variable \`opsz\` 9–144, weights 400–700, \`tracking-tight\`).

**The \`not-prose\` rule (LOCKED — caught in CIC visual verify).**
The \`.prose-article\` class is greedy. Any component nested under it that has its own typography
needs \`not-prose\` to opt out, OR the prose styles will silently eat it. Two known failure modes
observed during build:
- \`<Cta>\` buttons rendered invisibly because they inherited prose link styling.
- \`<RelatedArticles>\` rendered with spurious bullets because the prose \`ul\` rules applied.
Both components now wrap their content with \`not-prose\`. **Any new component you add inside
\`.prose-article\` MUST do the same** unless you've deliberately checked it inherits prose correctly.
Always CIC-verify after structural changes — don't trust "build green" alone. CIC procedure lives
in the \`browsercontrol\` skill book.

**Idioms ported from seres-immigration:**
- Warm-paper background (the \`bg\` token).
- Drop-cap on the first paragraph of every article.
- H2 with auto terracotta underline.
- Sticky desktop TOC on pillar pages.
- Related-articles block on every satellite.

**Idioms intentionally NOT ported (defer until needed):**
- Folder-tab navigation (seres had this for visa categories).
- Comparison panels (seres had this for visa-type side-by-sides).

**URL structure:**
- \`/\` — home
- \`/retirement-income/\` — pillar
- \`/retirement-income/<slug>/\` — 11 satellites
- \`/about/\`, \`/contact/\`, \`/privacy/\` — stubs (non-content utility pages)

When adding new pillars: \`/<pillar-slug>/\` with satellites at \`/<pillar-slug>/<satellite-slug>/\`.

**Trailing-slash quirk (known, NOT FIXED).** Next.js default issues 308 redirects from \`/foo/\` to
\`/foo\`. Internal links here are written with trailing slashes, so clicking shows one 308 hop. Harmless.
To eliminate, either drop trailing slashes from hrefs in \`lib/articles.ts\` OR set
\`trailingSlash: true\` in \`next.config.mjs\`. Choose intentionally; don't accidentally do both.

**Drop-cap opt-out inside structural boxes (LOCKED — commit \`6cae6f9\`).** The default rule
\`.prose-article p:first-of-type::first-letter\` adds a Fraunces drop-cap to the first paragraph of
every prose article. It LEAKS into the first paragraph of nested structural boxes (callouts, buckets,
stat cards, takeaway boxes, diagram wrappers) — a giant drop-cap inside a small callout is a visible
discipline failure. Codified opt-out in \`app/globals.css\`:

\`\`\`css
.prose-article :is(.callout, .bucket, .stat-card, .takeaway, .diagram-placeholder) p::first-letter {
  initial-letter: unset;
  font-size: inherit;
}
\`\`\`

Any NEW structural-box class added to the sanitizer's preserved set MUST be added to this opt-out
selector OR have a parent that already is. CIC-verify a callout with text in it after any
\`.prose-article\` CSS edit.

**Inline SVG default sizing (LOCKED — commit \`5d923d2\`).** CJGEO emits checkmark / icon / decorative
SVGs without explicit width/height. Browsers default unsized inline \`<svg>\` to FILL the container,
turning a 16-px checkmark into a 600-px page-wide banner. The codified default in \`app/globals.css\`:

\`\`\`css
.prose-article svg { width: 1.5rem; height: 1.5rem; }
\`\`\`

Per-context overrides (diagram \`<figure>\`, hero illustrations) explicitly set larger sizes via a
more specific selector. If a new SVG context renders too small / too large, add a specific selector
under \`.prose-article\` rather than removing the default — the default catches the long tail of
unattributed icons that CJGEO emits.
`.trim(),
    },

    addSatellite: {
      description:
        "Add a new satellite article under an existing pillar. The high-frequency operation on bryanblog.",
      howTo: `
Adds one satellite under an existing pillar (e.g. \`/retirement-income/<new-slug>/\`).

**Steps:**
1. **Edit \`lib/articles.ts\`** — append a new entry to the satellite array for the parent pillar
   with \`slug\`, \`title\`, \`teaser\`. Keep the slug short, hyphenated, and descriptive of the
   satellite's intent (e.g. \`how-much-safe-money-do-i-need\`).
2. **Create the page file** at \`app/<pillar-slug>/<new-slug>/page.tsx\`. Mirror an existing
   satellite's structure:
   - \`import { getSatellite } from "@/lib/articles"\`
   - \`import { ArticleShell } from "@/components/ArticleShell"\`
   - \`import { RelatedArticles } from "@/components/RelatedArticles"\`
   - \`export const metadata = { title, description }\` — title MUST match the LLM-aligned phrasing.
   - 4–6 \`<section id="...">\` blocks, each with an \`<h2>\` using LLM-aligned phrasing
     (per \`refStrategy\` rule 2).
3. **Wire the inbound link.** On the relevant pillar section, add an inline
   \`<Link href="/<pillar-slug>/<new-slug>/">\` reference so the satellite is reachable from the
   pillar's journey. Distributed entry points (rule 4) — don't orphan the satellite.
4. **Build:** from the repo root, \`npm run build\`. Verify green; type errors fail the build.
5. **Commit + push** to \`main\` on the \`boster00/bryanblog\` GitHub remote. Vercel auto-deploys
   on push to main.
6. **CIC-verify the live URL** after Vercel finishes (procedure: \`browsercontrol\` skill book).
   Per \`refDesign\`'s \`not-prose\` rule: visual verify is non-negotiable for any new component nesting.
`.trim(),
    },

    mapICPJourney: {
      description:
        "Research action: convert raw ICP insight into scenario-based content journeys. Extract customer scenarios → cluster into journey families → decompose into 8 stages (Trigger / Possibility / Options / Comparison / Risk / Process / Expert Selection / Action) → phrase every topic as a natural-language LLM question → emit pillar + satellite architecture + conversion map. Feeds [SB:BryanBlog->addPillar] / [SB:BryanBlog->addSatellite]. Run BEFORE drafting any pillar.",
      howTo: `
**Core principle.** A good content journey starts from *"what is happening in the customer's life
that makes them ask for help?"* — NOT from *"what does the company sell?"* Do NOT organize content
by Bryan's offer categories (retirement income, annuities, planning). Start from the customer's
lived scenario; the offer appears later as the natural expert-solution path.

**Why this exists.** bryanblog's strategy (refStrategy rule 1) is decision-journey pillars, not
service pages. This action is how the journey is actually mapped: raw ICP insight → scenarios →
journey families → stages → LLM-typable questions → pillar + satellite architecture →
website-conversion plan. Run this BEFORE [SB:BryanBlog->addPillar] for any pillar where the
journey isn't already locked.

This is RESEARCH, not strategy. Inductive only — cast a wide net, do not gate scenarios on
"too niche" / "too sales-y" / "compliance-concerning". Editorial curation happens later at
[SB:BryanBlog->addPillar] gate-time.

---

### Step 1 — Input intake (LOCKED — run this FIRST, before any research)

Output TWO lists verbatim to the operator. Do not proceed to Step 2 until the operator confirms or
supplies missing pending items. Missing items are acceptable — mark NOT-AVAILABLE and proceed
inductively.

**List A — Existing answers from context** (pre-fill from session memory + skill book context).
For bryanblog the standing answers are:

1. **Offer description.** A "retirement income review" — a 30-minute advisor conversation with no
   pitch / no obligation. Bryan positions as advisor, not salesman (per refStrategy rule 7). No
   product names, no insurer names, no return projections in any surface (per refStrategy rule 5).
2. **ICP description.** 55–75-year-old US consumer, anxious about running out of money in
   retirement. 8th-grade reading level. Calm + mature. Not a financial pro (per refStrategy rule 6).
   Author surface is "The Mysterious Author" — never "Bryan" (per refBranding).
3. **Target forums.** Default: Reddit. Recommended subs for this ICP — r/retirement,
   r/financialindependence, r/personalfinance, r/Bogleheads, r/Medicare, r/SocialSecurity.
4. **Customer emails / access method.** NOT-AVAILABLE — pending operator input.
5. **Seed keywords.** Pre-fillable from the live pillar + 11 live satellites under
   \`/retirement-income/\` (read \`lib/articles.ts\` in the bryanblog repo): "retirement income",
   "will my money last in retirement", "how much safe money", "social security at 62 vs 70",
   "sequence of returns risk", etc. Operator can override / extend.
6. **Questions the user wants to cover.** Currently scoped by the 7 H2s on the
   \`/retirement-income/\` pillar; operator confirms whether new pillars are in-scope this run.
7. **Social media defaults.** LinkedIn (financial-planner / retiree-adjacent voices), Facebook
   groups (where the 55–75 audience actually lives — search "retirement planning" / "early
   retirement" groups), YouTube comment sections on retirement-planning channels. Instagram is
   weak for this ICP; deprioritize unless operator confirms a creator they want surveyed.

**List B — Pending questions for the operator** (optional inputs that sharpen research; any
NOT-AVAILABLE just means we proceed inductively):

1. **Offer description** — any refinement / sub-offer beyond the standing "retirement income review"?
2. **ICP description** — any segment splits Bryan wants surfaced (pre-retiree vs in-retirement,
   widowed, business-owner exits, federal employees, etc.)?
3. **Target forums** — additional non-Reddit forums (Bogleheads.org, ER.org,
   AARP community, Morningstar boards)?
4. **Customer emails (or access method)** — inbox export, Gmail MCP query / label, IMAP creds, or
   "Bryan stores call notes in <location>". Source of verbatim consumer language + objections.
5. **Seed keywords** — beyond the auto-extracted ones, any high-priority terms to seed the
   DataForSEO related-keyword research (per [SB:cjgeo->keywordFullAuto])?
6. **Questions the user wants to cover** — any decisions Bryan explicitly wants covered or
   excluded (e.g. include Medicare-cost planning, exclude estate / long-term-care)?
7. **Social media accounts** — specific LinkedIn / YouTube creators / Facebook groups whose audience
   represents the ICP and whose comments / discussions should be mined?

Operator responses become Step 2's research substrate.

---

### Step 2 — Convert raw ICP insight into customer SCENARIOS

A scenario is NOT a demographic, an offer category, or a keyword. A scenario is a real-world
situation that creates urgency, confusion, risk, or decision pressure.

Use the canonical scenario sentence:

> The customer is trying to **[achieve outcome]**, but they are unsure about **[main uncertainty]**,
> worried about **[main risk]**, and need to decide **[next decision]**.

For each scenario extract:

| Field | Question to answer |
|---|---|
| Trigger | What caused the customer to start thinking about this now? |
| Desired outcome | What do they want to make happen? |
| Fear | What are they afraid will go wrong? |
| Confusion | What do they not understand? |
| Stakes | What happens if they choose wrong or delay? |
| Decision | What choice are they trying to make next? |
| Readiness | Early-stage, comparing options, or ready to act? |

Source material for scenarios (in priority order): customer emails / call notes (List B #4) →
forum threads (List B #3) → social comments (List B #7) → seed-keyword search-intent inference
(List B #5). Target 8–15 scenarios. Fewer = under-served; more = noise.

---

### Step 3 — Cluster scenarios into JOURNEY FAMILIES

Group scenarios by decision journey, NOT by Bryan's offer.

**The test:** would a customer naturally describe this journey in their own words?

**Good journey framing:** "How do I figure out whether this goal is possible?" · "How do I avoid
making a costly mistake?" · "How do I compare my options?" · "How do I prepare before I talk to an
expert?" · "How do I know which path is right for my situation?" · "How do I move from
uncertainty to action?"

**Bad journey framing:** Service Category A / Package Type B / Product Line C / Professional
Deliverable D.

Each journey family has:

| Field | Description |
|---|---|
| Journey name | Plain-language label based on the customer's problem |
| Core question | The main question the journey answers |
| Scenario cluster | Which customer scenarios belong here |
| Emotional driver | Fear, hope, urgency, frustration, status, security, etc. |
| Decision driver | The choice the customer needs to make |
| Conversion path | The natural next action after education |

---

### Step 4 — Decompose each journey into the 8 STAGES (LOCKED stage model)

Every journey decomposes into the same 8 stages. This is a locked model — do not invent new
stages; map every customer question to one of these.

**Stage 1 — Trigger / "Is this relevant to me?"**
Content goal: validate the situation and name the problem in the customer's own language.
LLM-typable patterns: "What should I do if [situation]?" · "Is it possible to [desired outcome]?"
· "Why is [problem] happening?" · "Do I need help with [situation]?"

**Stage 2 — Possibility / "Can I do this?"**
Content goal: explain feasibility, requirements, constraints, disqualifiers.
Patterns: "Can I [achieve outcome] if [constraint]?" · "What are the requirements for [goal]?" ·
"How do I know if I qualify for [path]?" · "What makes someone eligible or ineligible for [option]?"

**Stage 3 — Option Framing / "What are my choices?"**
Content goal: map the major options in plain language.
Patterns: "What are my options if I want to [goal]?" · "What is the best way to [outcome]?" ·
"Should I choose [path A] or [path B]?" · "What alternatives do I have if [constraint]?"

**Stage 4 — Comparison / "Which path fits me?"**
Content goal: match options to situation-specific criteria.
Patterns: "Which option is better for someone in my situation?" · "How do I choose between [A]
and [B]?" · "What is the difference between [A] and [B]?" · "What should I consider before
choosing [path]?"

**Stage 5 — Risk / "What could go wrong?"**
Content goal: explain risks, tradeoffs, common mistakes, red flags.
Patterns: "What are the common mistakes when trying to [goal]?" · "What can go wrong with [path]?"
· "How do I avoid delays or rejection?" · "What should I check before I commit?"

**Stage 6 — Process / "What happens next?"**
Content goal: practical roadmap — steps, timeline, documents, preparation, responsibilities.
Patterns: "What is the process for [goal]?" · "What steps do I need to take to [outcome]?" ·
"How long does [process] usually take?" · "What documents or information do I need before starting?"

**Stage 7 — Expert Selection / "Who should help me?"**
Content goal: what to look for in an expert, what questions to ask, how to evaluate fit.
Patterns: "How do I choose the right expert for [situation]?" · "What questions should I ask
before hiring someone?" · "How do I know if someone is qualified to help with [goal]?" · "What
should I bring to the first consultation?"

**Stage 8 — Action / "How do I start?"**
Content goal: convert interest into a consultation, assessment, review, estimate, checklist, or
intake. For bryanblog this is the "retirement income review" CTA (per refStrategy rule 7).
Patterns: "What should I do first if I want to [goal]?" · "How do I prepare for a consultation
about [situation]?" · "What information should I collect before asking for help?" · "Can someone
review my situation and tell me my options?"

---

### Step 5 — Turn each stage into TOPIC CANDIDATES (LLM-question phrasing)

For each (journey, stage, scenario) triple, generate one or more topic candidates by combining
**customer situation + customer question + stage intent**.

**Phrase as natural-language questions, NOT professional labels.**

| ❌ Bad (SEO / editorial label) | ✅ Good (LLM-typable question) |
|---|---|
| Eligibility Requirements Overview | Can I do this if my situation is complicated? |
| Comprehensive Service Guide | What happens if I made a mistake earlier? |
| Professional Process Explanation | How do I know which option is safer? |
| Advanced Compliance Considerations | What should I prepare before talking to someone? |

Each topic gets:

| Field | Description |
|---|---|
| Topic title | Plain-language question |
| Journey | Which journey it belongs to |
| Stage | One of the 8 stages |
| Customer scenario | Which scenario generated it |
| Search/LLM phrasing | Natural question the customer would type |
| Intent | Awareness / consideration / evaluation / conversion |
| Page role | Pillar section · satellite article · FAQ · checklist · comparison page · CTA page |
| Next step | What the page should naturally lead to |

---

### Step 6 — Choose the PILLAR PAGE for each journey

Each journey gets ONE pillar page. The pillar is a guided decision page, NOT a service page.

**Pillar title formulas:**
- "How Do I [Achieve Desired Outcome] Without [Main Fear or Risk]?"
- "Can I [Achieve Outcome]? A Plain-English Guide to Your Options"
- "What Should I Do If [Customer Situation]?"
- "How to Plan for [Goal] When You're Not Sure Where to Start"
- "How to Choose the Right Path for [Scenario]"

Pillar structure mirrors the 8 stages — one section per relevant stage. Skip a stage only if it
genuinely doesn't apply to that journey (rare; usually all 8 apply).

---

### Step 7 — Assign SATELLITES to pillar sections

The pillar section gives the simple answer. The satellite gives the deeper answer.

\`\`\`
Journey Name
  Main pillar: [broad customer question]
    Stage: [journey stage]
      Pillar section: [plain-language section question]
      Satellite: [deeper LLM-style question]
      Satellite: [comparison question]
      Satellite: [mistake / risk question]
\`\`\`

**Satellite-title formulas:**
- "What are my options if [scenario]?"
- "Can I still [goal] if [constraint]?"
- "How long does [process] take?"
- "What are the biggest mistakes to avoid?"
- "What is the difference between [option A] and [option B]?"
- "How do I know whether [path] is right for me?"
- "What should I prepare before my first consultation?"

---

### Step 8 — Convert each journey into WEBSITE FUNCTION

A journey page doesn't just educate — it moves the customer forward. Assign every page a function:

| Page role | Function |
|---|---|
| Pillar page | Help visitor understand the whole decision landscape |
| Satellite article | Answer a specific high-intent question |
| Comparison page | Help visitor choose between paths |
| Risk page | Help visitor avoid mistakes |
| Checklist page | Prepare visitor for action |
| Intake page | Capture the right information for expert review |
| CTA module | Convert the journey into a next step |

**CTA formulas (use these, not generic "Contact us" / "Learn more" / "Get started"):**
- Request a situation review
- Check your options
- Prepare for your first consultation
- Get a feasibility review
- Ask an expert to review your case
- Start with a planning call
- Submit your situation for review

For bryanblog the standing CTA is "Request a retirement income review" (per refStrategy rule 7) —
match the situation when wording it per-page, but the soft-advisor framing is locked.

---

### Step 9 — Final output package (LOCKED format)

Write a single markdown report at \`docs/icp-journey-map-<YYYY-MM-DD>.md\` in the bryanblog repo
(NOT GuildOS — research lives next to the articles it informs). Commit + push so the historical
sequence of ICP-mapping runs is preserved.

Required sections (in this exact order):

**A. Customer scenario extraction**

| Scenario | Trigger | Desired outcome | Fear | Confusion | Decision needed | Readiness stage |

**B. Journey families**

| Journey | Core customer question | Scenarios included | Emotional driver | Decision driver | Conversion path |

**C. Journey-stage map**

| Journey | Stage | Customer question | What the content must answer | Funnel intent |

**D. LLM-targeted topic map**

| Journey | Stage | Topic title (natural-language question) | Scenario source | Page role | Next step |

**E. Pillar and satellite architecture** (tree format, one block per journey)

\`\`\`
Journey Name
  Main pillar: [broad customer question]
    Stage: Trigger
      Section: [plain-language question]
      Satellite: [deeper LLM-style question]
    Stage: Possibility
      Section: [plain-language question]
      Satellite: [deeper LLM-style question]
    Stage: Options
      Section: [plain-language question]
      Satellite: [deeper LLM-style question]
    Stage: Comparison
      Section: [plain-language question]
      Satellite: [deeper LLM-style question]
    Stage: Risk
      Section: [plain-language question]
      Satellite: [deeper LLM-style question]
    Stage: Process
      Section: [plain-language question]
      Satellite: [deeper LLM-style question]
    Stage: Expert Selection
      Section: [plain-language question]
      Satellite: [deeper LLM-style question]
    Stage: Action
      Section: [plain-language question]
      Satellite: [deeper LLM-style question]
\`\`\`

**F. Website conversion map**

| Journey | Pillar page CTA | Satellite CTA | Intake information to collect | Follow-up page or flow |

---

### Quality control rules (run before declaring the report done)

1. **Would the customer say this?** If a topic sounds like internal company language, rewrite.
2. **Does this start from the customer's situation?** If it starts from the offer, rewrite.
3. **Does each topic belong to a journey stage?** If not, it's random — drop or remap.
4. **Does each page move the customer forward?** If it only informs but doesn't reduce risk /
   clarify a decision / prepare for action, improve it.
5. **Is the CTA the next logical step?** Early-stage pages don't push hard; later-stage pages
   invite review / consultation / assessment / intake.
6. **Does the topic sound like something someone would ask an LLM?** If it sounds like a
   traditional SEO blog title, rewrite into natural language.

---

### Discipline notes (bryanblog-specific)

- **Compliance constraints still apply to scenario / topic phrasing.** No product names, no
  insurer names, no return projections, no uncaveated guarantees — even in research artifacts
  (per refStrategy rule 5 + refBranding). Customer-language verbatim quotes are an exception when
  preserved as quotes.
- **"Bryan" never appears in any topic title, pillar title, or CTA phrasing.** The author surface
  is "The Mysterious Author" (per refBranding) — research artifacts honor the same rule.
- **DataForSEO seed-keyword expansion** happens via [SB:cjgeo->keywordFullAuto] when the topic map
  is ready for SEO research — NOT during scenario extraction. Keep this action's output pre-SEO.
- **Visual-verify N/A.** This action produces a markdown report, not a rendered page. The Sage
  success-claim gate (CLAUDE.md) is satisfied when the markdown file is opened and spot-checked by
  the operator at the path stated in the report.

---

### One-sentence summary

Take raw ICP research, extract real customer scenarios, cluster those scenarios into decision
journeys, break each journey into the 8 stages, turn each stage into natural-language questions
people would actually ask an LLM, and map those questions into pillar pages, satellite pages,
checklists, comparisons, and conversion flows.
`.trim(),
    },

    addPillar: {
      description:
        "Add a new decision-journey pillar with its satellites. Heavyweight — plan the whole journey before writing.",
      howTo: `
A pillar is a strategic addition, not a tactical one. Plan the journey before writing.

**Pre-work (do this BEFORE touching files):**
1. Identify the ONE consumer decision journey the pillar covers (per \`refStrategy\` rule 1).
   Examples: "Will my money last?", "Should I take Social Security at 62 or 70?", "How much
   should I have in safe money?"
2. Draft the 7 LLM-aligned section H2s mapping to that journey (per \`refStrategy\` rule 2).
   This is the spine — each H2 will become a satellite's launch pad.
3. Define 8–12 satellite slugs in advance with their angles. You don't have to write all of
   them on day one, but you should know what they'll be so the pillar internal-links land on
   real targets.

**Implementation steps:**
1. **Extend \`lib/articles.ts\`** — add a \`pillars\` entry (or extend whatever data module
   shape exists) with the new pillar's \`slug\`, \`title\`, and its satellites array.
2. **Create \`app/<new-pillar-slug>/page.tsx\`** using \`ArticleShell\` with 7
   \`<section id="...">\` blocks for the 7 H2s. Each section ends with an inline link to its
   corresponding satellite.
3. **Create each satellite** under \`app/<new-pillar-slug>/<satellite-slug>/page.tsx\`
   following the \`addSatellite\` pattern.
4. **Update \`components/SiteNav.tsx\`** to add the new pillar to the top-level nav.
5. **Build, commit, push.** Vercel auto-deploys.
6. **CIC-verify** the pillar + a sample of 2 satellites after deploy (procedure: \`browsercontrol\` skill book).

**Honest scope check.** If you can't commit to 8+ satellites within the next quarter, the pillar
will look stub-thin and undercut the "decision journey" promise. Defer the pillar until the
satellite roster is real.
`.trim(),
    },

    recolor: {
      description: "Change the palette. The whole site reskin lives in one \`@theme\` block.",
      howTo: `
The palette lives in ONE place: the \`@theme\` block in \`app/globals.css\`. Recoloring the site
is a 5-token swap.

**Steps:**
1. Edit \`app/globals.css\`. In the \`@theme\` block, replace the 5 color tokens
   (\`primary\`, \`accent\`, \`bg\`, \`accent-tint\`, \`neutral-mid\`).
2. **Keep semantic names stable.** Don't rename tokens or every \`bg-primary\`-style class
   across the site breaks.
3. **Verify contrast** (WCAG AA minimum):
   - Body text on \`bg\` — target ≥ 7:1 (AAA preferred for body prose).
   - Headings (\`primary\`) on \`bg\` — ≥ 4.5:1.
   - CTAs (\`accent\`) on \`bg\` — ≥ 4.5:1.
4. **Build + push.** Vercel auto-deploys.
5. **CIC-verify** the home page, one pillar, and one satellite after deploy (procedure: \`browsercontrol\` skill book). Look specifically at:
   - Drop-cap visibility.
   - H2 terracotta-underline visibility.
   - CTA button contrast.
   - Footer/nav contrast.
`.trim(),
    },

    addCta: {
      description:
        "Add a CTA block to a page. The \`<Cta>\` component already handles the \`not-prose\` wrapping; place sparingly.",
      howTo: `
Use the existing component:

\`\`\`tsx
<Cta
  heading="Request a retirement income review"
  body="A 30-minute conversation, no pitch, no obligation."
  button={{ label: "Schedule the review", href: "/contact/" }}
/>
\`\`\`

**The component already wraps its content with \`not-prose\`** so it works inside
\`.prose-article\` without inheriting prose link styles (per \`refDesign\` rule).

**Placement discipline.**
- Per \`refStrategy\` rule 7: framing is "request a retirement income review", never "buy".
- CTAs lose weight if everywhere. One mid-article and one end-of-article is the usual maximum;
  one end-of-article is often enough.
- The CTA button \`href\` is the soft conversion path — currently \`/contact/\`. Don't deep-link
  to a third-party scheduler or product page from inside the article body.
`.trim(),
    },

    deployToVercel: {
      description:
        "Deploy or redeploy the site to Vercel. Auto-deploys on push to main; manual paths documented for when the autopath fails.",
      howTo: `
**Primary path: push to main.**
\`git push origin main\` on the \`boster00/bryanblog\` GitHub remote triggers an auto-deploy in
the Vercel \`bryanblog\` project (under team/account \`cjs-projects-963e3b09\` / \`boster00\`).

**Manual redeploy (when push didn't trigger or you want to force):**
- Dashboard: log into vercel.com as \`boster00\`, find the \`bryanblog\` project under
  \`cjs-projects-963e3b09\`, click "Redeploy" on the latest deployment.
- CLI alternative: from the repo root,
  \`\`\`bash
  cd C:/Users/xsj70/bryanblog
  vercel --prod --yes
  \`\`\`
  (Requires \`vercel\` CLI logged in as \`boster00\`.)

**Post-deploy verify.**
- Live URL: https://<REDACTED-HOST>/
- Always CIC-verify after a structural change — Vercel reports build success but does not verify
  rendering. CIC procedure lives in the \`browsercontrol\` skill book.
`.trim(),
    },

    replaceArticleWithCJGEO: {
      description:
        "Forward-reference: replace a native article's prose with CJGEO-generated content. The next phase of the project. Drives via the cjgeo skill book.",
      howTo: `
This is the hand-off path from native prose to CJGEO-generated content. Each article is replaced
one at a time — never bulk-replace, because the CIC visual-verify step is per-article.

**Driver skill book:** \`cjgeo\` (see \`libs/skill_book/cjgeo/index.js\`). This action is the
bryanblog-side recipe; cjgeo owns the wire-level pipeline.

**Steps:**
1. **Pick the target article** and identify its main keyword (the LLM-aligned H1 phrasing is
   usually a good starting point — e.g. "How much money do I need to retire?").
2. **Keyword research** — drive the CJGEO Content Magic keyword tool per \`[SB:cjgeo->setMainKeyword]\`
   + \`[SB:cjgeo->keywordFullAuto]\`. Verify search volumes. Commit ONE main keyword per article.
3. **Generate the article** — drive \`[SB:cjgeo->topicFullAuto]\`, \`[SB:cjgeo->promptFullAuto]\`,
   \`[SB:cjgeo->generateDraft]\`, \`[SB:cjgeo->pollDraft]\` (5-min cadence, 20-min timeout),
   \`[SB:cjgeo->adoptDraft]\`. The adopted draft body is \`content_html\` on
   \`content_magic_articles\` in CJGEO's Supabase.
4. **Pull the HTML.** Use \`[SB:cjgeo->connectSupabase]\` (from the \`C:/Users/xsj70/cjgeo\` repo)
   to fetch \`content_html\` for the article id. CIC's renderer freezes on CJGEO's draft surface
   (see \`[SB:cjgeo->driveContentMagicUI]\`), so don't try to scrape it via CIC.
5. **Replace the page prose in bryanblog.** Open the target page at
   \`app/<pillar-slug>/<satellite-slug>/page.tsx\` and replace the article's JSX prose body with the
   CJGEO content, converted to the page's existing component shape.
   - **Keep the \`<section id="...">\` anchors** intact so the pillar's inline anchor links
     keep working.
   - **Keep inter-article \`<Link>\` references** to other satellites intact — these are the
     distributed-entry-point web (\`refStrategy\` rule 4).
   - Strip any CJGEO-side markup that conflicts with bryanblog's component shapes (e.g. inline
     style attributes that fight Tailwind).
6. **Mark the article's source.** In \`lib/articles.ts\`, set \`source: 'cjgeo'\` (vs \`'native'\`)
   on the article's entry. This is how the bryanblog roster tracks which articles have been
   hydrated by CJGEO. Add the field if it doesn't exist yet on the schema.
7. **Build + push.** Vercel auto-deploys.
8. **CIC-verify the live URL** after deploy (procedure: \`browsercontrol\` skill book). Per the
   Sage success-claim gate (CLAUDE.md cross-role rule): an unverified CJGEO replacement is NOT
   done — the layout-shape risk is real (CJGEO doesn't know about bryanblog's \`.prose-article\`
   greediness or the \`not-prose\` rule).

**Compliance re-check.** CJGEO content can drift into product names, return projections, or hard
CTAs. Re-read \`refStrategy\` rules 5 + 7 against the generated content BEFORE merging. If the
generated copy violates compliance, edit it down before push — don't ship and patch.
`.trim(),
    },

    adoptCJGEORawContent: {
      description:
        "Canonical pipeline: CJGEO raw article HTML → bryanblog-aligned page. Sanitize, anchor, deep-link, write to \`content/\`, rewire the route as an async server component, build, push, CIC-verify.",
      howTo: `
This is the codified end-to-end procedure to adopt CJGEO-generated content into bryanblog. It was
proven informally across the pilot, batch, and pillar hydrations and is now the single canonical
howTo. Prefer this over the older \`replaceArticleWithCJGEO\` recipe — that entry stays for
strategic context (compliance re-check, distributed entry points) but does NOT cover the
sanitize/anchor/route mechanics.

**Driver skill books.**
- CJGEO HTTP wire spec — see \`[SB:cjgeo->refApiActions]\`. Don't restate the wire spec here.
- Browser/CIC visual verify — see the \`browsercontrol\` skill book.

**Locked precondition for draft kickoff.** Before calling \`[SB:cjgeo->generateDraft]\`, the article MUST have \`main_keyword\` set AND non-empty \`keywords[]\` (via \`[SB:cjgeo->keywordFullAuto]\`) AND non-empty \`topics[]\` (via \`[SB:cjgeo->topicFullAuto]\`). The reusable check lives at \`[SB:cjgeo->refFullAutoGate]\` — call it immediately before \`generate_draft\`. Earlier satellite batch runs silently skipped SEO/GEO research and lost the value-add; this gate exists to prevent that.

**Authority.** \`/design-guide\` on the live bryanblog site is the authoritative reference for
bryanblog's visual language. Any sanitizer or template choice that conflicts with what
\`/design-guide\` shows is a discipline failure — flag it and reconcile to /design-guide before
shipping.

---

### Inputs

- \`articleId\` — CJGEO article uuid
- \`slug\` — target bryanblog satellite slug (e.g. \`how-much-safe-money-do-i-need\`)
- \`urlPath\` — full URL path on bryanblog (e.g. \`/retirement-income/<slug>/\`)
- \`pagePath\` — route file path (e.g. \`app/retirement-income/<slug>/page.tsx\`)
- \`anchorMap\` (optional) — \`{ <expected-id>: <human-readable-h2-text-fragment> }\` dictionary used
  for the pillar where TOC anchors must be preserved 1:1. Omit on satellites where slug-from-text
  is fine.
- \`satelliteLinks\` (optional) — \`{ <section-anchor-id>: [{ slug, title }, ...] }\` map of related
  satellites to inject as inline deep-dive links under each section.

---

### Pre-flight

- \`/design-guide\` exists on the live site. Read it FIRST (\`browsercontrol\` SB) and treat it as the
  visual contract. Anything in the sanitized HTML that contradicts it must be reconciled before push.
- API key: \`CJGEO_API_KEY\` in \`C:\\Users\\xsj70\\GuildOS\\.env.local\`. All CJGEO calls use the
  \`x-api-key\` header.
- API endpoint canonicals (corrected wire spec — use these, NOT older skill book entries):
  - Create: \`POST /api/v1/articles { title, keyword, create: true }\` → 201 \`{ article_id }\`
  - Set main keyword: \`POST /api/articles/{id}/keyword { action: "set_main_keyword", main_keyword }\`
  - Keyword full_auto (fixed in cjgeo commit \`f41bf6a\`): \`POST /api/articles/{id}/keyword {}\` → 200 \`{ committed, summary }\`
  - **Get assets:** \`POST /api/articles/{id} { action: "get_assets" }\` — on the article ROOT, NOT \`/keyword\` (verified quirk from pillar hydration)
  - Generate draft: \`POST /api/articles/{id}/edit-draft { action: "generate_draft", mode, allow_image_generation, context_prompt, use_default_context_prompt, generation_prompt_mode }\`
  - Poll: \`POST /api/articles/{id}/edit-draft { action: "pull_status" }\`
  - Adopt: \`POST /api/articles/{id}/edit-draft { action: "adopt_draft" }\`
  - Get HTML: \`POST /api/articles/{id} { action: "get_html" }\` → \`{ content_html }\`

---

### Adoption pipeline

**Step 1 — Fetch raw HTML.**
\`POST /api/articles/{articleId} { action: "get_html" }\` with \`x-api-key\`. Capture \`content_html\`.

**Step 2 — Sanitize via the canonical module \`lib/sanitize-cjgeo.mjs\`.**

**Authoritative API:** \`import { sanitizeCJGEO } from "@/lib/sanitize-cjgeo"\` (TS shim at
\`lib/sanitize-cjgeo.d.ts\`). Signature:
\`\`\`ts
sanitizeCJGEO(rawHtml: string, options?: {
  anchorMap?: Array<{ id: string; keywords: string[] }>;   // locked pillar anchors
  satelliteLinks?: Record<string, Array<{ href: string; title: string }>>;  // deep-dive injection
  strictAnchors?: boolean;
}): { html: string; headings: Array<{id, text}>; wordCount: number }
\`\`\`

**Pipeline (run by \`sanitizeCJGEO\` in order):**
1. \`stripScaffoldingAndChrome\` — drop DOCTYPE/html/head/body/main/article, style/script/comments,
   link/meta/title, header/footer/nav, img/picture, in-body h1, and CSS-only FAQ \`<input>\`/\`<label>\`
   accordion shells (we keep \`<details>\` instead, which works without CSS). Drop blocks
   (div/section/aside) whose class matches DROP_BLOCK_CLASSES (hero, site-header, site-footer,
   toc, article-meta, byline, author-bio, cta-block, cta-box, etc.). Drop bare \`<aside>\`.
2. \`unwrapSections\` — remove \`<section>\` open + close tags (keep contents).
3. \`cleanAttributesAndUnwrap\` — innermost-first pass on \`<div>\`/\`<span>\`. **Permissive mode:**
   keep ALL non-drop-list classes (drop only role/aria/data/style/id). This lets the long tail of
   BEM-style class names (stat-card__label, bucket--safe, compare-card__heading,
   timeline__item, callout--warn, def-grid, pull-quote, etc.) flow through and pick up CSS in
   \`app/globals.css\`. Then attr-clean headings, \`<a>\` (href-only), \`<ul>\`/\`<ol>\`/\`<li>\` (keep
   class), table tags (keep class), \`<details>\`/\`<summary>\` (keep class + open). SVG attrs
   filtered to geometry-safe allowlist (viewBox, fill, xmlns, aria-hidden, cx/cy/r/rx/ry, d,
   x/y/x1/y1/x2/y2, points, stroke + variants, width/height, transform, opacity, gradient
   primitives).
4. \`dropNonAllowedTags\` — defensive: any tag not in ALLOWED_TAGS gets removed.
5. \`injectH2Ids\` — anchor-map mode (pillar, locked) or slug-from-text mode (satellites). The
   satellite slug-from-text path de-duplicates collisions with \`-2\`, \`-3\` suffixes.
6. \`injectSatelliteLinks\` — per-section inline \`<p><em>→ Deep dive: <a>...</a></em></p>\`.

**Class-preservation strategy: DROP-LIST, not allow-list.** CJGEO emits hundreds of BEM-style
class names — trying to enumerate every "preserved" class is a losing game. So the sanitizer
preserves EVERY class name by default and only drops classes (and their blocks) that are known
page-chrome. The trade-off is honest fidelity to the source structure at the cost of carrying
extra (unstyled-but-harmless) class names through to the DOM. CSS in
\`app/globals.css\` styles the ones that materially affect rendering.

**ALLOWED_TAGS:** h2 h3 h4 h5 h6 p blockquote pre hr br ul ol li dl dt dd table thead tbody tfoot
tr th td caption strong em b i a code mark abbr small sub sup u span time details summary figure
figcaption div section svg g circle path rect line polyline polygon ellipse text tspan defs
linearGradient radialGradient stop use title.

**DROP_BLOCK_CLASSES** (block + contents dropped — this is THE list to maintain): all variants of
hero/hero-* (incl. hero__* BEM), article-hero/article-hero__*, article-header, article-deck,
article-meta/article-meta__*, article-label, article-title, article-subtitle, article-category,
meta-bar/meta-item/meta-author/meta-divider, byline/byline-*, author-avatar/author-bio/author-bio__*/
author-box/author-info/author-name, reading-time, site-header/site-footer/site-name/site-tagline/
site-wrap/footer-*, toc/toc-*/toc__*, sidebar/article-sidebar, cta-section/cta-block/cta-block__*/
cta-box/cta-box__*/cta-btn/cta-button/cta-heading/cta-fine.

Known quirk: CJGEO sometimes emits a full HTML document (DOCTYPE + html + head + body), other
times just a fragment. The sanitizer MUST handle both shapes — \`stripScaffoldingAndChrome\` does.

**Step 3 — Inject \`id\` on h2s.** Two modes:

*Default — slug-from-text.* For each \`<h2>\`, derive \`id\` from text: lowercase, replace whitespace
with \`-\`, strip punctuation, cap at ~60 chars. Idempotent on re-runs.

*Anchor-map mode (pillar).* When \`anchorMap\` is supplied, fuzzy-match each \`<h2>\`'s text against
the expected text fragments and assign the corresponding required id. Abort + report if:
- count of h2s ≠ count of entries in \`anchorMap\`, OR
- any \`<h2>\` fails to map to a required id.

This is non-negotiable for the pillar because the sticky TOC and inline pillar→satellite anchor
links break silently if an id drifts.

**Step 4 — Inject inline deep-dive links (optional, by section).**

For each section, after the last \`<p>\` BEFORE the next \`<h2>\` (or end of body), append one
\`<p>\` per related satellite:

\`\`\`html
<p><em>→ Deep dive: <a href="/retirement-income/<satellite-slug>/">Satellite Title</a></em></p>
\`\`\`

Sections with multiple satellites get multiple \`<p>\` lines in order. The italic + arrow framing
is a locked idiom — matches the design language without competing with body prose. Per
\`refStrategy\` rule 4, this is how the pillar journey radiates into satellites at every section.

**Step 4.5 — Fill diagrams with gpt-image-2 (LOCKED — MANDATORY).**

Any content placeholder of the form \`[Diagram: <title>]\`, \`[Chart: <title>]\`, \`[Visual: <title>]\`,
\`[Illustration: <title>]\`, \`[Figure: <title>]\`, or \`[Graph: <title>]\` MUST be replaced with a
gpt-image-2-generated editorial illustration BEFORE the content file is written to disk. Both the
literal text placeholder AND any wrapping \`<div class="diagram-placeholder">\` shell are processed
in the same pass. The image is saved to \`public/images/diagrams/<slug>-<n>.png\` and referenced
inline via \`<figure class="diagram"><img src="/images/diagrams/<slug>-<n>.png" alt="<title>" loading="lazy"><figcaption>{title}</figcaption></figure>\`.

If image generation fails after 3 retries: log the failure, REMOVE the placeholder entirely (don't
ship literal \`[Diagram: ...]\` text to users), and continue with the rest of the adoption.

**Heuristic for the wrapping \`<div class="diagram-placeholder">\`:**
- If the div contains ONLY the bracket text + a decorative SVG (no other structural children
  with classes like \`.bucket\`, \`.timeline\`, \`.compare-card\`, \`<table>\`, etc.) → replace the
  WHOLE div with the new \`<figure>\`.
- If the div contains structural children (bucket cards, timeline steps, comparison panels, etc.)
  → KEEP the structural children, strip the bracket text and decorative SVG, and INSERT the
  \`<figure>\` at the top of the div.

**Canonical prompt template — SOURCE OF TRUTH is \`lib/fill-diagrams.mjs\` in the bryanblog
repo (the \`CANONICAL_PROMPT_PREFIX\` constant + \`buildPrompt\` function). Do NOT restate the full
prompt here; read the module directly and update it there if you need to refine.**

**Why the template is locked the way it is (audit 2026-05-20).** The original short
"Style: clean editorial vector …" prompt consistently produced gpt-image-2 output with finance
clichés (money buckets, growing money plants, balance scales of money, coin stacks with $ signs),
burnt-in text/numbers (Social Security ages "62/66-67/70", bar-chart axis labels), scenery
backgrounds (Greek temples, cityscapes, lighthouses), and bottom-strip icon collages. The refined
template adds explicit STRICTLY-FORBIDDEN bans (no plants in vessels, no scales, no coin stacks,
no scenery, no icon strips, no collage), a strengthened NO-TEXT directive, a density rule
("ONE concept clearly, 2-4 simple shapes with breathing space"), and a composition rule
(landscape with central 50-65% subject + generous solid #FAF9F8 whitespace). The five live diagrams
under \`public/images/diagrams/\` (monthly-income-needs-1..3, retirement-paycheck-1..2) are the
locked reference examples of the refined style — consult them before tweaking the prompt.

**Cliché note:** even with explicit bans, gpt-image-2 will sometimes regenerate the very cliché
you forbid (growing plants and balance scales were the recurring offenders on the audit). The
workaround that worked: add a \`metaphorHint\` per-diagram pointing at a specific abstract-geometry
alternative (e.g. "two ceramic vessels with horizontal anchor lines, NO plants, NO water") instead
of leaving the model to invent its own metaphor. When a regen still produces a cliché after one
retry, refine the metaphorHint rather than retrying with identical input.

**Canonical implementation:** \`lib/fill-diagrams.mjs\` in the bryanblog repo. Exposes
\`fillDiagrams(sanitizedHtml, { slug, publicDir, openaiImagesGenerate, surroundingContextChars })\`.
Generates images IN PARALLEL (concurrency up to 4 per article — gpt-image-2 at quality:"high"
takes 60–120s each; sequential would be too slow). Uses \`openai_images\` weapon's \`generate\`
export from GuildOS \`libs/weapon/openai_images\` (or any compatible adapter injected via the
\`openaiImagesGenerate\` option).

**Backfill script:** \`scripts/backfill-diagrams.mjs\` scans every article in
\`content/retirement-income/*.html\`, \`content/about/index.html\`, \`content/contact/intro.html\`
and rewrites in place. Run with \`--env-file=C:/Users/xsj70/cjgeo/.env.local\` for \`OPENAI_API_KEY\`.

**Verification gate:** after backfill, grep all content files for the six placeholder prefixes —
zero matches required before commit. Then CIC-verify the live pages.

**Step 5 — Write to \`content/<path>/<slug>.html\`.**

This file is the source of truth for the article body. Prepend a header comment:

\`\`\`html
<!--
  CJGEO-generated article body.
  article_id: <uuid>
  main_keyword: <chosen>
  adopted_at: <YYYY-MM-DD>
-->
\`\`\`

For \`/retirement-income/<slug>/\` satellites this lands at \`content/retirement-income/<slug>.html\`.

**Step 6 — Update \`lib/articles.ts\`.** Set the article's entry:
- \`source: 'cjgeo'\`
- \`cjgeoArticleId: '<uuid>'\`
- \`mainKeyword: '<chosen>'\`
- \`adoptedAt: '<ISO date>'\` (extend the type cleanly if the field doesn't exist yet)

**Step 7 — Rewrite \`page.tsx\` as an async server component.** Read the HTML from disk at
request time; render it via \`dangerouslySetInnerHTML\` INSIDE \`ArticleShell\` but OUTSIDE the
CTA + RelatedArticles components (so they keep the \`not-prose\` discipline from \`refDesign\`).

\`\`\`tsx
import { promises as fs } from "fs";
import path from "path";
import ArticleShell from "@/components/ArticleShell";
import RelatedArticles from "@/components/RelatedArticles";
import { getSatellite, relatedFor } from "@/lib/articles";

export default async function Page() {
  const html = await fs.readFile(
    path.join(process.cwd(), "content/retirement-income/<slug>.html"),
    "utf8"
  );
  // Derive sections from h2 ids via regex — feed ArticleShell's TOC.
  const sections = Array.from(html.matchAll(/<h2 id="([^"]+)">([^<]+)<\\/h2>/g))
    .map(([, id, text]) => ({ id, label: text }));
  const meta = getSatellite("<slug>")!;
  const related = relatedFor(["<sibling-slug-1>", "<sibling-slug-2>"]);
  return (
    <ArticleShell eyebrow={meta.eyebrow} title={meta.title} sections={sections}>
      <div className="prose-article" dangerouslySetInnerHTML={{ __html: html }} />
      <RelatedArticles articles={related} />
    </ArticleShell>
  );
}
\`\`\`

CTA blocks (pillar §7, the income-review-checklist satellite) stay as \`<Cta>\` JSX OUTSIDE the
\`dangerouslySetInnerHTML\` — that's how \`not-prose\` stays intact. Don't fold CTAs into the
adopted HTML body or they'll inherit prose styling and render invisibly (\`refDesign\` rule).

**Step 8 — Build + verify.**
- \`npm run build\` green (no TS errors, no missing imports).
- Spin dev server on port 3004. Curl the URL: 200 + body contains every expected anchor id.

**Step 9 — Commit + push.**
- Commit, push to \`boster00/bryanblog\` main.
- Vercel auto-deploys (~75s).

**Step 10 — Visual verify via the \`browsercontrol\` skill book.** Open the live URL via CIC and
confirm renders match \`/design-guide\` expectations:
- Drop-cap on first paragraph (\`.prose-article p:first-of-type::first-letter\` fires)
- h2 terracotta underline visible
- Terracotta inline link color
- Related-articles strip at the bottom unaffected by prose greediness
- CTA blocks (where present) intact — white text on terracotta, no prose inheritance
- No \`.prose-article\` greediness regressions (the \`not-prose\` escape hatch on \`<Cta>\` and
  \`<RelatedArticles>\` MUST remain — per \`refDesign\`'s locked rule)

**Step 10a — Side-by-side fidelity verify (preserved structural elements).** When the article uses
any of the preserved classes (callouts, comparison-wrap, income-layers, checklist, ind badges),
do a side-by-side visual diff vs the CJGEO preview:
- CJGEO render: \`https://cjgeoai.com/content-magic/{articleId}/preview\` via CIC navigation.
  (The \`/preview\` route is the public render that does NOT freeze, unlike the editor surface —
  see \`[SB:cjgeo->captureArticleScreenshot]\`. Use it for the CJGEO side.)
- bryanblog render: the live URL via CIC navigation.
- For each preserved class present in the article, confirm: callout boxes render as panels (not
  bare paragraphs); the comparison table has a colored header row + cell badges; income-layers
  shows the stacked-card visual with slate-navy gradient labels; checklist items show inline
  checkmarks; ind badges are colored pill labels.
- If a structural element looks bare/unstyled, the CSS allowlist in \`app/globals.css\` is missing
  a rule — add it under \`.prose-article .<class-name>\`, re-deploy, re-verify.

Per the Sage success-claim gate (CLAUDE.md cross-role rule), an adoption without this visual-verify
step is NOT done. The layout-shape risk is real — CJGEO doesn't know about bryanblog's
\`.prose-article\` greediness or the \`not-prose\` rule.

---

### Known quirks (observed across pilot, batch, pillar hydrations)

- CJGEO emits styled wrappers + sometimes full HTML documents — sanitizer must handle both.
- CJGEO doesn't reliably honor target word counts (asks for 500-700, returns 1000-1600+). Accept it
  — depth is fine, doesn't block adoption.
- CJGEO sometimes folds explicit "intro" instructions into the first section heading. Not blocking;
  the sanitizer's h1-strip + orphan-text-before-first-h2 strip cleans it up.
- \`keyword full_auto\` payload returns \`keywords[]\` without numeric volume/difficulty — fall back
  to the \`note\` field (intent description) when volume is needed for prioritization. (Owner doc:
  [SB:cjgeo->keywordFullAuto].)
- **HTML-entity double-escape bug (LOCKED — fixed in commit \`e0c88a0\`).** Source HTML from CJGEO
  sometimes contains pre-escaped entities (\`&mdash;\`, \`&nbsp;\`, \`&hellip;\`, \`&rsquo;\`, \`&amp;\`).
  The sanitizer's text-escape pass would then double-escape, e.g. \`&mdash;\` → \`&amp;mdash;\`, which
  renders as literal text "&mdash;" to users. Fix: \`decodeCommonEntities(text)\` runs BEFORE any
  \`escapeHtmlText()\` pass in \`lib/sanitize-cjgeo.mjs\`. If a future caption / blockquote shows literal
  \`&xxx;\` text on the live page, the decode pass is being skipped — fix the sanitizer, don't
  string-replace in the content file.
- **Entity-in-slug known issue (NOT YET FIXED).** \`injectH2Ids\` slug-from-text mode does NOT decode
  entities before slugifying. An h2 containing \`&mdash;\` in the source produces an id like
  \`how-to-plan-mdash-step-by-step\` (the literal \`mdash\` leaks through). Manual workaround: re-write
  the h2 source HTML to use the literal character ('—') before sanitize. Proper fix: pipe heading
  text through \`decodeCommonEntities()\` inside \`injectH2Ids\` before the slug regex pass. Low
  priority — the pillar uses anchor-map mode which avoids the slug pass entirely.

---

### Compliance re-check

Before push, re-read \`refStrategy\` rules 5 + 7 against the adopted content. CJGEO drifts into
product names, return projections, or hard CTAs more often than expected. Edit down BEFORE push —
don't ship and patch.
`.trim(),
    },

    refBranding: {
      description:
        "Reference (LOCKED): bryanblog's branding constraint — the author surface is 'The Mysterious Author', never 'Bryan'. Every CJGEO context_prompt must include this; every adoption must grep for leaks before push.",
      howTo: `
**The lock.** bryanblog's published author surface is "The Mysterious Author". The owner's first name
"Bryan" must NEVER appear in body copy, author bylines, captions, image alt text, or metadata title /
description. The site is intentionally pseudonymous for the published persona; the owner identity
stays off-channel.

**Why this is fragile.** CJGEO's article generator does not know about this constraint by default —
the model freely uses "Bryan" if it's anywhere in the page slug, the keyword research, or the
context_prompt. Adoptions WITHOUT explicit prompt framing leak "Bryan" into the body roughly 30–40% of the
time. The framing alone reduces it to ~5%; the post-sanitize grep catches the residue.

### Where to enforce

**1. CJGEO context_prompt — MANDATORY clause.** Every [SB:cjgeo->generateDraft] call for bryanblog
content MUST include explicit branding framing in \`context_prompt\`. Canonical phrasing:

> "Author byline is 'The Mysterious Author' — DO NOT use the name 'Bryan' anywhere in the article body,
> headings, captions, or author surface. The author persona is intentionally pseudonymous."

Include this verbatim in the context_prompt for both [SB:BryanBlog->adoptCJGEORawContent] (fresh
articles) and [SB:BryanBlog->reAdoptCJGEO] (re-runs). The chaperone-driven flow per
[SB:cursorCJGEO->replicateArticle] also threads this through.

**2. Post-sanitize grep.** After [SB:BryanBlog->adoptCJGEORawContent] Step 2 (sanitize) and BEFORE
Step 5 (write to \`content/\`), grep the sanitized HTML for the literal token "Bryan":

\`\`\`bash
grep -i "bryan" content/<path>/<slug>.html
\`\`\`

Expected output: empty. If non-empty, MANUALLY rewrite each occurrence to "The Mysterious Author" (or
remove the surrounding clause if rewriting is awkward). Only push after the grep is clean. The
sanitizer itself does NOT auto-rewrite — false positives are too easy (place names, surnames in
quoted research, etc.); a human / agent decision is required per occurrence.

**3. Vercel-deploy CIC verify (Step 10 of [SB:BryanBlog->adoptCJGEORawContent]).** Final visual
verify includes scrolling the article body in CIC and confirming "Bryan" does not appear. The CSS does
not catch text content; only the visual pass does.

### Anti-patterns

- DO NOT inject branding via a global find-replace pass on the sanitized HTML. Context matters — a
  legitimate quoted reference to a different "Bryan" (an author cited in research, a place name like
  "Bryan, Texas") should NOT be rewritten. Per-occurrence judgment is the rule.
- DO NOT put the branding rule ONLY in \`.cursorrules\` and skip the context_prompt — \`.cursorrules\`
  is the chaperone-side discipline manual; CJGEO does not read it. The framing must be in CJGEO's own
  prompt path.
- DO NOT relax the "Bryan" ban for SEO reasons. The pseudonymity is a strategic choice per the site
  owner; it overrides the SEO yield from using a real name.
`.trim(),
    },

    auditUnresearched: {
      description:
        "Audit the bryanblog satellite roster against CJGEO assets to classify articles by what's missing (keywords / topics / both / neither). The triage pattern for batch regeneration.",
      howTo: `
**Why this exists.** When an orchestrator needs to regenerate / re-research a roster of articles, the
first decision is "which articles need WHAT?". Some articles have committed keywords but no topics
(a partial pipeline run); some have neither; some have both but lost depth. The audit classifies each
article into one of four states so the next step is mechanical, not eyeballed.

### Taxonomy (LOCKED — four states)

For each \`{articleId, kw_count, topic_count}\` triple, classify by:

| State | Condition | Next action |
|---|---|---|
| \`OK\` | \`kw_count > 0 AND topic_count > 0\` | No research needed. May still need draft regeneration if body is stale. |
| \`MISSING_TOPIC\` | \`kw_count > 0 AND topic_count === 0\` | Run [SB:cjgeo->topicFullAuto] only. Keywords already committed. |
| \`MISSING_KW\` | \`kw_count === 0 AND topic_count > 0\` | Run [SB:cjgeo->keywordFullAuto] only. Rare — usually means a previous topic-only run. |
| \`MISSING_BOTH\` | \`kw_count === 0 AND topic_count === 0\` | Run [SB:cjgeo->keywordFullAuto] THEN [SB:cjgeo->topicFullAuto]. Most common after a fresh article create. |

### How to gather counts

Pull from CJGEO's Supabase per [SB:cjgeo->connectSupabase]. For each bryanblog satellite with a
\`cjgeoArticleId\` in \`lib/articles.ts\`:

\`\`\`js
const { data } = await db.from("content_magic_articles")
  .select("id, title, outline")
  .in("id", articleIds);
// outline.keywords?.length, outline.topics?.length give the counts
\`\`\`

Or via the API: \`POST /api/articles/{id} {action:"get_assets"}\` returns
\`{main_keyword, keywords[], topics[], prompts[]}\` per [SB:cjgeo->refApiActions §1].

### Output shape

Produce a structured table the orchestrator can iterate:

\`\`\`
[
  { slug: 'how-much-safe-money-do-i-need', articleId: '...', state: 'OK', kw: 12, topics: 8 },
  { slug: 'should-i-take-social-security-at-62', articleId: '...', state: 'MISSING_BOTH', kw: 0, topics: 0 },
  ...
]
\`\`\`

Then dispatch the next-action per state, SERIALLY per [SB:cjgeo->refDataForSEOSeedStrategy] §3. Do
NOT parallel-batch the regenerations — the DataForSEO empty-data risk is higher in parallel.

### When to run the audit

- Before any batch hydration sweep (the high-yield trigger).
- After a substrate fix landed (e.g. the May 2026 \`f41bf6a\` / \`7f2a1b1\` fixes) — articles that
  silently kept=0 during the broken window need re-classification.
- Periodically (~monthly) as a health-check during the CJGEO-replacement-rollout phase.
`.trim(),
    },

    reAdoptCJGEO: {
      description:
        "Re-adopt an EXISTING CJGEO article into bryanblog without creating a new article — pulls content_html from the existing article_id and re-runs sanitize / anchor / fill / verify. The fidelity-uplift pattern.",
      howTo: `
**When to use this.** An article was previously adopted into bryanblog, but the sanitizer / CSS /
diagram fill / branding pass has improved since the original adoption and you want to re-pull the
fidelity uplift WITHOUT generating a fresh CJGEO article (which would lose the existing keyword /
topic research and consume credits unnecessarily).

**Proven on commit \`d8626a4\`** as the fidelity-uplift sweep across the May 2026 satellite roster —
14 satellites re-adopted from existing CJGEO sources after the sanitizer drop-list rewrite + diagram
fill + drop-cap opt-out + SVG sizing + entity-decode fixes landed.

### When NOT to use this

- The CJGEO article is genuinely stale (research is months old, keywords no longer reflect site
  intent, body angle has drifted). Use [SB:cursorCJGEO->replicateArticle] for a fresh generation.
- The article was generated under a broken substrate window (e.g. before the \`f41bf6a\` /
  \`7f2a1b1\` fixes) and has empty keywords/topics — re-research first per
  [SB:BryanBlog->auditUnresearched], then [SB:BryanBlog->adoptCJGEORawContent].

### Steps

1. **Look up the existing \`cjgeoArticleId\`** from \`lib/articles.ts\` for the target slug.
2. **Pull current \`content_html\`** via \`POST /api/articles/{articleId} {action:"get_html"}\` per
   [SB:cjgeo->refApiActions §1]. NO new article is created; no \`generate_draft\` is called.
3. **Run the existing sanitize / anchor / inject pipeline** per
   [SB:BryanBlog->adoptCJGEORawContent] Steps 2–4.5. The improvements in the current sanitizer
   (drop-list, drop-cap opt-out, SVG sizing, entity decode, branding grep per
   [SB:BryanBlog->refBranding]) all apply.
4. **Overwrite \`content/<path>/<slug>.html\`** in place. Update the header comment's \`adopted_at\`
   to the current date. Leave \`article_id\` and \`main_keyword\` unchanged.
5. **Skip the \`lib/articles.ts\` source-of-truth edit** — \`source: 'cjgeo'\` is already set;
   \`cjgeoArticleId\` is unchanged. No other fields need rewriting.
6. **Build + push + CIC visual-verify** per [SB:BryanBlog->adoptCJGEORawContent] Steps 8–10a.
   The visual-verify is non-optional — that's the whole point of the re-adoption.

### What you save vs full regeneration

- **No CJGEO credit spend.** \`get_html\` is a read; no \`generate_draft\` call.
- **Keyword/topic research preserved.** No risk of DataForSEO empty-data on a re-seed.
- **Compliance / branding leaks already triaged** if the article was previously through the
  post-sanitize grep — re-grep anyway per [SB:BryanBlog->refBranding] (small risk of regressions
  if the sanitizer's branding-impact behavior changed).

### What you do NOT pick up by re-adopting

- New competitor topics / keywords (research is frozen in the source CJGEO article).
- LLM model upgrades on CJGEO's side (the content_html was written by the model active at the
  ORIGINAL generation time).

If those matter, the regeneration path per [SB:cursorCJGEO->replicateArticle] is correct, not this.
`.trim(),
    },

    adoptCJGEOHighFidelity: {
      description:
        "LOCKED canonical pipeline: adopt CJGEO content into a destination repo at HIGH FIDELITY by preserving CJGEO's own hero / chrome / design treatment, NOT stripping it. Per-article scoped <style> block + raw HTML + bypass destination's ArticleShell. Proven 2026-05-26 across all 26 cloudstonedesigns satellites achieving ~90-95% visual parity to CJGEO preview.",
      howTo: `
This is the high-fidelity alternative to [SB:BryanBlog->adoptCJGEORawContent]. Where that action STRIPS CJGEO's hero / dark hero bands / illustrated visuals / comparison panels and renders the body in bryanblog's own ArticleShell (a strict-brand strategy), THIS action PRESERVES the full CJGEO design treatment per-article — same eyebrows-with-rules + big tall serif titles + subtitle decks + meta rows + dark hero bands + multi-card grids + multi-step flow with circular nodes — so the destination page looks like CJGEO's editorial preview almost verbatim.

**When to use which strategy.**
- Use **[SB:BryanBlog->adoptCJGEORawContent]** when the destination has a strong, locked ArticleShell that the brand insists on (e.g. bryanblog's drop-cap + h2 underline + sticky TOC + footer-related-articles must wrap every article). The body inherits the destination's design language; CJGEO chrome is discarded.
- Use **THIS action** when the destination has a generic / loose chrome and you want CJGEO's editorial polish to come through (e.g. cloudstonedesigns — momentx-inspired warm-stone palette, minimal Tailwind 4 design system, no editorial chrome of its own). The body brings its own scoped chrome; the destination only wraps it in nav + post-article CTA + RelatedArticles.
- **Both strategies preserve cloudstonedesigns / bryanblog brand palette tokens.** CJGEO's parchment palette is NOT injected globally; it's scoped per-article. The destination's nav, footer, site-header, and other chrome are untouched.

**Proven 2026-05-26 across cloudstonedesigns** (26 satellites, 3 pillars). Two leaf-agent iterations on different pages confirmed 92-95% parity to CJGEO preview with the infrastructure alone (no per-page CSS authoring required for most articles).

---

### Infrastructure to build once per destination repo

**1. API route: \`app/api/cjgeo-readopt/[id]/route.ts\`** (or equivalent). Accepts \`?raw=1\` query param. Two responsibilities:

(a) Fetch raw \`content_html\` from CJGEO via \`POST {CJGEO_BASE}/api/articles/{id} {action:"get_html"}\` with the \`x-api-key\` header (per [SB:cjgeo->refApiActions §1]).

(b) Process and write the content file. In **raw mode**, the route runs an inline CSS scoper, NOT \`sanitizeCJGEO\`:

\`\`\`ts
// Extract <style data-draft-styles> block from raw content_html.
const styleMatch = raw.match(/<style[^>]*>([\\s\\S]*?)<\\/style>/i);
const styleSrc = styleMatch ? styleMatch[1] : "";
const htmlNoStyle = raw.replace(/<style[^>]*>[\\s\\S]*?<\\/style>/gi, "");

// Scope every selector under \`.article-<slug>\`:
//  - drop site-chrome (html, body, :root, *, .site-header, .site-footer, .site-nav, .site-wrap)
//  - replace :host / :host > .editorContent with the scope class
//  - prefix every remaining selector with \`.article-<slug>\`
//  - keep @media / @supports / @keyframes / @font-face / @import intact (recurse for @media + @supports)
const scopeCls = \`article-\${slug}\`;
const scopePrefix = \`.\${scopeCls}\`;
function scopeCss(css) { /* see cloudstonedesigns route for the parser */ }
const scopedStyle = \`<style data-cjgeo-scoped="\${scopeCls}">\${scopeCss(styleSrc)}</style>\`;

// Wrap body so the scoped rules match.
const finalHtml = \`\${scopedStyle}\\n<div class="\${scopeCls}">\${htmlNoStyle}</div>\`;
// Write to content/<pillarSlug>/<slug>.html with header comment.
\`\`\`

The scoper is the LOAD-BEARING primitive. Without it the page either (a) leaks CJGEO chrome into the destination site or (b) loses CJGEO chrome entirely. With it, the article is isolated.

**2. Destination's route file: \`app/<pillar>/[slug]/page.tsx\`** (or framework equivalent). Add a \`cjgeo\` source branch that bypasses the destination's ArticleShell:

\`\`\`tsx
if (bodyHtml && satellite.source === 'cjgeo') {
  return (
    <div className="cjgeo-article">
      <div className="prose-article cjgeo-full" dangerouslySetInnerHTML={{ __html: bodyHtml }} />
      <div className="max-w-7xl mx-auto px-6">
        <Cta ... />
        <RelatedArticles articles={related} />
      </div>
    </div>
  );
}
// else fall back to ArticleShell-based rendering for native articles.
\`\`\`

Render body full-width (no max-width constraint on .prose-article when .cjgeo-full is present). Cta + RelatedArticles still render after in destination brand.

**3. Cross-article shared patterns in globals.css** scoped under \`.prose-article.cjgeo-full\`. Optional but useful when the same pattern appears across many articles (callout, pill, comparison-table, dark-band). Most patterns are already captured in the per-article scoped block — only add to globals when DRY makes sense.

---

### Per-article adoption procedure (the standard flow)

For each article:

1. **Pull + scope:** \`curl http://localhost:<port>/api/cjgeo-readopt/<articleId>?raw=1\`. This writes a content file with a per-article scoped \`<style>\` block at the top and the body wrapped in \`<div class="article-<slug>">\`.

2. **Visual verify via CIC** (per [SB:browsercontrol]):
   - Open the CJGEO preview at \`https://cjgeoai.com/content-magic/<articleId>/preview\`. Screenshot full-page.
   - Open the destination CSD page at \`http://localhost:<port>/<pillar>/<slug>\`. Screenshot full-page.
   - Compare side-by-side.

3. **For most articles, no further work is needed.** The scoped \`<style>\` block carries CJGEO's design treatment intact. Expected baseline: ~85-95% fidelity to CJGEO preview, just from steps 1+2.

4. **If gaps exist, the residual pattern is almost always one of:**

   **(a) Globals.css specificity battle.** The destination's \`globals.css\` ships generic \`.prose-article .tag / .pill / .badge / .indicator-chip\` chrome with the SAME specificity \`(0,2,0)\` as the per-article scoped \`.article-<slug> .tag\` rule. When CSS bundling order favors globals, the destination chrome wins and the per-article styling is lost on chip/pill/badge classes.

   **Fix-1 (preferred):** add a reset block in globals.css that explicitly UNSETS the destination chrome inside a \`.cjgeo-full\` scope:
   \`\`\`css
   .prose-article.cjgeo-full .tag,
   .prose-article.cjgeo-full .pill,
   .prose-article.cjgeo-full .indicator-chip,
   .prose-article.cjgeo-full .badge,
   .prose-article.cjgeo-full .brand {
     all: unset; /* let per-article scoped CSS reassert */
   }
   \`\`\`
   Specificity \`(0,3,1)\` wins. Per-article scoped rules then take effect normally.

   **Fix-2 (per-article hotfix):** when the CSS bundle is stale or you need an immediate fix, bump the scoped \`.tag\` (or other affected) rules in the article's \`<style>\` block to use \`!important\`. The article scope keeps the bang contained.

   **(b) Stale CSS bundle / Next dev HMR.** Next.js dev mode is flaky on CSS hot-reload. If your CSS edits don't show up after reload, force a rebuild: touch the file with a small comment change, or hard-reload (Ctrl+Shift+R). NOT a permanent issue — production build picks up everything.

   **(c) Missing media / images.** CJGEO articles often include \`[Hero visual: ...]\` / \`[Visual placeholder: ...]\` text blocks. If the destination wants real images instead of placeholder boxes, run a diagram-fill pass per [SB:cjgeo->refImagePlaceholderHandling] (gpt-image-2 via the openai_images weapon).

   **(d) Site-chrome leaks.** If you notice the destination's typography/layout looking different on the article AFTER adoption (e.g. body font changed site-wide), the scoper failed to drop site-chrome rules. Audit the scoper's drop-list — \`html\`, \`body\`, \`:root\`, \`*\`, \`*::before\`, \`*::after\`, \`.site-header\`, \`.site-footer\`, \`.site-nav\`, \`.site-wrap\` must all be dropped from the scoped style.

5. **Verify gate (per Sage success-claim, CLAUDE.md):** open the LIVE destination URL in user CIC, screenshot, eyeball against the CJGEO preview screenshot. Same hero structure, same eyebrows, same section markers, same component density, same color semantics for pills/callouts/severity badges. Pixel-perfect is NOT the bar — **editorial weight is.** Wrap points and minor padding deltas don't count.

---

### Bulk adoption (>5 articles)

When adopting many articles in sequence:

1. Loop the \`/api/cjgeo-readopt/<id>?raw=1\` curl for every article (serially or in small parallel batches; the route is idempotent and the dev server handles it).
2. Spot-check 2-3 representative articles via CIC (one with simple body, one with complex grid + comparison panels, one with dark hero bands). If those pass, the rest are almost certainly fine — the scoper is generic.
3. Dispatch a leaf agent per problem article ONLY if a spot-check surfaces a real gap. Per-article agents are rarely needed once infrastructure works.

---

### Leaf-agent prompt template (LOCKED — proven 2026-05-26 across multiple iterations)

When the spot-check finds a residual gap on one article that you want a leaf agent to close, dispatch with this prompt shape:

> You are achieving high-fidelity visual match between a CJGEO preview page and a [destination] page. The adoption infrastructure has been built — the article's content_html is ingested in raw mode and scoped per-article. Your job is to compare the two pages and fix residual gaps until ~90% fidelity.
>
> **CJGEO preview:** \`<url>\`
> **Destination page:** \`<url>\`
> **Article slug:** \`<slug>\`
>
> Infrastructure in place: \`/api/cjgeo-readopt/<id>?raw=1\` fetches raw content_html, extracts the \`<style>\` block, scopes every selector under \`.article-<slug>\`, drops site-chrome rules, preserves \`@media\`/\`@keyframes\`. Each content file has a per-article scoped \`<style data-cjgeo-scoped="article-<slug>">\` block. Pillar route files have a \`cjgeo\` source branch that bypasses ArticleShell.
>
> **Steps:** screenshot both via CIC → compare → identify gaps → if needed re-pull via \`curl <route>?raw=1\` → for residual gaps add CSS to the per-article \`<style>\` block OR globals.css → reload + re-screenshot → iterate.
>
> **Stop at ~90% similarity.** Same hero structure, same section layouts, same density. Pixel-perfect is not the bar.
>
> **Constraints:** CIC only for browser; no .mjs script files; no git commits; do NOT change destination brand tokens; HMR is flaky — re-edit the file to force rebuild.
>
> **Report:** final fidelity verdict, gaps closed, gaps remaining, files modified, screenshot IDs for final pair, any generalizable pattern for the skill book.

---

### Anti-patterns (do NOT do these)

- **Do NOT** strip the CJGEO \`<style>\` block without scoping it first. The styles ARE the high-fidelity treatment. If you strip them and try to recreate via globals.css, you're authoring ~24 KB of CSS per article by hand — defeats the purpose.

- **Do NOT** inject the CJGEO \`<style>\` block unmodified at the article level. Its selectors target \`:host\`, \`html\`, \`body\`, etc. and will either silently fail (Shadow DOM-only selectors) or leak globally (chrome rules). Always run the scoper.

- **Do NOT** dispatch a leaf agent per article on the assumption "every page needs a custom CSS pass." Two iterations on cloudstonedesigns confirmed: most articles need ZERO per-page CSS once the infrastructure is right. Save agents for the actual outliers.

- **Do NOT** mix the strategies on one site. Either ALL articles use the destination's ArticleShell (per [SB:BryanBlog->adoptCJGEORawContent]) or ALL articles preserve CJGEO chrome (per THIS action). Mixing produces visual inconsistency across articles.

- **Do NOT** swap the destination's brand color tokens to CJGEO's tokens. Map them: keep destination \`--color-accent\`, \`--color-bg\`, etc. The per-article scope handles CJGEO's color refs via inline values inside the scoped \`<style>\`.

---

### Locked references

- Working implementation: \`C:\\\\Users\\\\xsj70\\\\cloudstonedesigns\\\\app\\\\api\\\\cjgeo-readopt\\\\[id]\\\\route.ts\` — the scoper + raw-mode handler.
- Working pillar route file: \`C:\\\\Users\\\\xsj70\\\\cloudstonedesigns\\\\app\\\\starting-your-project\\\\[slug]\\\\page.tsx\` — the cjgeo source branch + ArticleShell bypass.
- Cross-article shared CSS: \`C:\\\\Users\\\\xsj70\\\\cloudstonedesigns\\\\app\\\\globals.css\` — ~530 lines under \`.prose-article.cjgeo-full\`.
- Commit: \`99594a1\` (cloudstonedesigns/master) — the build that achieved ~92-95% fidelity across all 26 articles.

The 8 generalizable patterns (cemented from the first leaf agent's report 2026-05-26):

1. Per-article scoped \`<style>\` block is the load-bearing primitive.
2. Drop site-chrome rules during scoping (\`html\`, \`body\`, \`:root\`, \`*\`, \`.site-header\`, etc.).
3. Preserve \`@media\` and \`@keyframes\` unchanged (don't try to prefix them).
4. Bypass the destination's ArticleShell in the route file when \`source === 'cjgeo'\`.
5. Cross-article shared patterns belong in \`globals.css\` under \`.prose-article.cjgeo-full\` — not in every article file.
6. Do NOT touch destination brand tokens.
7. Verification = CIC navigate both URLs → screenshot at same scroll position → eyeball.
8. The 90% bar is *editorial weight*, not pixels.
`.trim(),
    },

    refRepoLayout: {
      description:
        "Reference: where things live in the bryanblog repo. Quick map so a fresh agent doesn't have to glob.",
      howTo: `
\`C:\\Users\\xsj70\\bryanblog\\\`

\`\`\`
app/
  layout.tsx                       # root layout — Inter + Fraunces font loaders, SiteNav, footer
  page.tsx                         # home
  globals.css                      # Tailwind 4 @theme block, .prose-article hand-written
  about/page.tsx                   # stub
  contact/page.tsx                 # stub
  privacy/page.tsx                 # stub
  retirement-income/
    page.tsx                       # pillar — 7 H2 sections
    <satellite-slug>/page.tsx      # 11 satellites under this pillar
components/
  ArticleShell.tsx                 # wraps article body with .prose-article + sticky TOC (pillar)
  Cta.tsx                          # CTA block — uses not-prose internally
  RelatedArticles.tsx              # related-articles strip — uses not-prose internally
  SiteNav.tsx                      # top nav — pillar links live here
lib/
  articles.ts                      # data module: pillars + satellites with slug/title/teaser
                                   # (extend with \`source: 'cjgeo' | 'native'\` when CJGEO hydration starts)
next.config.mjs                    # default config; trailingSlash NOT set (see refDesign quirk)
package.json                       # Next 15.4.8, React 19, Tailwind 4, TypeScript
tsconfig.json                      # default Next TS config
public/                            # static assets — favicon, og images, etc.
README.md                          # bryanblog-side notes (this skill book is the GuildOS-side ref)
\`\`\`

**Repo metadata:**
- Local: \`C:\\Users\\xsj70\\bryanblog\`
- GitHub: \`boster00/bryanblog\` — https://github.com/boster00/bryanblog
- Vercel: \`bryanblog\` project under \`cjs-projects-963e3b09\` (account \`boster00\`)
- Live: https://<REDACTED-HOST>/
- Local dev port: 3004 (\`npm run dev -- -p 3004\` from the repo root)
- Owner email: <REDACTED-EMAIL>

**Related context:**
- Design-language source repo: \`C:\\Users\\xsj70\\seres-immigration\` (Astro 5).
- Content-hydration partner: CJGEO (\`C:\\Users\\xsj70\\cjgeo\` + the \`cjgeo\` skill book).
- Browser/CIC discipline (validateCIC, screenshot/navigate, cloud-vs-local browser policy):
  the \`browsercontrol\` skill book — canonical GuildOS-wide owner.
`.trim(),
    },
  },
};

export default skillBook;
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
