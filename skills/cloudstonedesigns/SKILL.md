# cloudstonedesigns

<!-- GUILDOS-SKILL name="cloudstonedesigns" -->

<!-- GUILDOS-DESC -->
Maintain and extend the cloudstonedesigns Next.js architecture site: add portfolio projects, hydrate journey pages with CJGEO, manage the contact form, deploy to Vercel. A Next.js 16 architecture portfolio + lead-gen site built around 3 customer decision journeys (uncertainty, recovery, decision anxiety) with pillar + satellite content, an image-first portfolio grid, and a structured intake form. Read refStrategy + refDesign before any editorial or structural change.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/cloudstonedesigns/SKILL.md -->
```markdown
---
name: cloudstonedesigns
description: >-
  Maintain and extend the cloudstonedesigns Next.js architecture site: add
  portfolio projects, hydrate journey pages with CJGEO, manage the contact
  form, deploy to Vercel. A Next.js 16 architecture portfolio + lead-gen site
  built around 3 customer decision journeys (uncertainty, recovery, decision
  anxiety) with pillar + satellite content, an image-first portfolio grid, and a
  structured intake form. Read refStrategy + refDesign before any editorial or
  structural change.
---

# Cloudstone Designs — Architecture Portfolio + Lead-Gen Site

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/cloudstonedesigns/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto cloudstonedesigns <action>` (or read the source directly).

## Actions

- **addProject** — Add a new portfolio project to the site.
- **adoptCJGEOPage** — Replace native prose on a journey page with CJGEO-generated content. Competitor research + keyword research MUST precede draft generation.
- **deployToVercel** — Deploy to Vercel. DEFERRED — do after CJGEO hydration + real photography.
- **refDesign** — Reference: palette, typography, Tailwind 4 setup, component discipline. Read before any CSS or structural change.
- **refRepoLayout** — Reference: file layout in the cloudstonedesigns repo.
- **refStrategy** — Reference: the strategic decisions that shape every editorial + structural choice. Read this first.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/cloudstonedesigns/index.js -->
```js
/**
 * cloudstonedesigns skill book — knowledge registry for the Cloudstone Designs architecture site.
 *
 * **Project shape.** `cloudstonedesigns` is a Next.js 16 (app router, React 19, TS, Tailwind 4)
 * architecture portfolio + lead-gen site. It is a SEPARATE repo from GuildOS — local only for now.
 * The site is built around 3 customer decision journeys (uncertainty → possibility, recovery from
 * failure, decision anxiety) with pillar + satellite content architecture, a portfolio grid, and
 * a structured contact/intake form.
 *
 * **Content engine.** Native prose fills pages first. CJGEO then replaces each page one-by-one,
 * with competitor research + keyword research BEFORE draft generation (locked prerequisite per
 * refFullAutoGate in the cjgeo skill book).
 *
 * **Design language is informed by momentx.studio:**
 * - Near-black + warm white + stone accent + warm gray — NO bright colors
 * - Photography/imagery does the visual work on portfolio pages
 * - Cormorant Garamond headings (editorial, elegant) + DM Sans body (clean)
 * - 4-column portfolio grid, 16:9 aspect ratio cells
 * - Minimal flat nav (no dropdowns)
 * - Marquee CTA on contact page
 * - Structured intake form pre-qualifies leads (service type, budget, timeline)
 *
 * **Where things live.**
 *  - Repo (local): `C:\Users\xsj70\cloudstonedesigns`
 *  - Repo (GitHub): not yet created
 *  - Live: not yet deployed (Vercel deploy deferred)
 *  - Local dev port: 3005 (`npm run dev:cloudstone`)
 *
 * **Port registry (cross-project):**
 *  - 3002: GuildOS (main app)
 *  - 3004: bryanblog (LOCKED)
 *  - 3005: cloudstonedesigns (LOCKED)
 *
 * **Browser/CIC discipline lives in the `browsercontrol` skill book** — canonical GuildOS-wide
 * owner of every CIC procedure. Any visual-verify step defers to it.
 */

export const skillBook = {
  id: "cloudstonedesigns",
  title: "Cloudstone Designs — Architecture Portfolio + Lead-Gen Site",
  description:
    "Maintain and extend the cloudstonedesigns Next.js architecture site: add portfolio projects, hydrate journey pages with CJGEO, manage the contact form, deploy to Vercel.",
  steps: [],
  toc: {
    refStrategy: {
      description:
        "Reference: the strategic decisions that shape every editorial + structural choice. Read this first.",
      howTo: `
The site is built on 6 locked strategic decisions.

**1. Decision-journey content architecture.** The site's content engine is NOT a blog or service catalog.
It maps to 3 universal customer decision journeys:
- Scenario 1: "I have a goal but don't know if it's even possible" (uncertainty → possibility)
- Scenario 2: "I already tried something and it didn't work" (frustration → recovery)
- Scenario 3: "I'm close to making a decision but afraid of choosing wrong" (decision anxiety)

Each scenario has a PILLAR page (the journey end-to-end) and SATELLITE pages (one per stage/question).
This mirrors bryanblog's pillar/satellite architecture but serves commercial-intent architecture clients.

**2. LLM-aligned satellite titles.** Titles are phrased the way a real person types into ChatGPT,
Reddit, or Google — NOT as SEO blog posts. "What Do I Do If My Architecture Project Failed?" beats
"Architecture Project Failure Recovery Guide". This captures distributed LLM/search entry points.

**3. Portfolio is the credibility anchor.** Every journey page has a CTA path to the portfolio.
The portfolio is image-first — photography carries the story, not copy. 4-column grid, 16:9 cells,
project narrative on detail pages. Keep portfolio current; stale projects undercut trust.

**4. Lead-gen via structured intake, not generic "contact us".** The contact form pre-qualifies:
service type, budget range, timeline, location. This filters for serious inquiries and sets up
the first consultation productively. CTA copy: "Tell us about your project" — never "Buy now".

**5. Compliance-safe tone.** No cost guarantees. No unrealistic timeline promises. Frame estimates
as "typical ranges — your project may vary." Position the studio as trusted advisor, not vendor.

**6. People page for trust.** A "People" page (separate from About) humanizes the studio.
Clients hire architects they trust as humans, not as services. Keep team bios current.
`.trim(),
    },

    refDesign: {
      description:
        "Reference: palette, typography, Tailwind 4 setup, component discipline. Read before any CSS or structural change.",
      howTo: `
**Framework:** Next.js 16 (app router) · React 19 · TypeScript · Tailwind 4.

**Tailwind 4 specifics:**
- Configured via \`@theme\` block in \`app/globals.css\` — NO \`tailwind.config.js\`.
- No typography plugin. Custom \`.prose-article\` class hand-written in globals.css.

**Palette (informed by momentx.studio — restraint is the point):**
| Token        | Hex       | Role                                        |
|--------------|-----------|---------------------------------------------|
| primary      | \`#0D0D0C\` | near-black warm — headings, body text       |
| accent       | \`#8B7355\` | cloudstone warm — CTAs, H2 underline, links |
| bg           | \`#FAFAF8\` | warm white — page background                |
| surface      | \`#F0EDE8\` | light stone — cards, callout backgrounds    |
| neutral-mid  | \`#A8A6A1\` | warm gray — borders, meta text, dividers    |
| accent-tint  | \`#EDE5DA\` | pale stone — soft callout backgrounds       |

**Typography:**
- Body sans: **DM Sans** (Google Fonts, variable, weights 300–600)
- Headings serif: **Cormorant Garamond** (Google Fonts, variable opsz, weights 300–700)

**The \`not-prose\` rule (LOCKED — same discipline as bryanblog).**
\`.prose-article\` is greedy. Any component nested inside it must use \`not-prose\` to opt out.
Known components that require \`not-prose\`: \`<Cta>\`, \`<RelatedArticles>\`.
Drop-cap opt-out for structural boxes: \`.prose-article :is(.callout, .bucket, .stat-card, .takeaway) p::first-letter\`.

**Portfolio grid:** 4-col desktop / 2-col tablet / 1-col mobile. 16:9 aspect-ratio cells.

**URL structure:**
- \`/\` — home
- \`/portfolio/\` · \`/portfolio/<slug>/\`
- \`/about/\` · \`/people/\` · \`/contact/\`
- \`/starting-your-project/\` · \`/starting-your-project/<slug>/\` (9 satellites)
- \`/project-rescue/\` · \`/project-rescue/<slug>/\` (9 satellites)
- \`/choosing-an-architect/\` · \`/choosing-an-architect/<slug>/\` (8 satellites)
`.trim(),
    },

    addProject: {
      description: "Add a new portfolio project to the site.",
      howTo: `
1. Edit \`lib/projects.ts\` — append a Project entry (slug, title, category, year, location, teaser, imageSrc).
2. Add images to \`public/images/projects/<slug>/\` (hero 16:9 minimum).
3. Write 3-4 paragraphs of professional project narrative in the detail page.
   Use real architecture vocabulary: program, envelope, materiality, section, parti, threshold.
4. Build + CIC-verify.
`.trim(),
    },

    adoptCJGEOPage: {
      description:
        "Replace native prose on a journey page with CJGEO-generated content. Competitor research + keyword research MUST precede draft generation.",
      howTo: `
Mirrors [SB:BryanBlog->adoptCJGEORawContent] for architecture journey pages.

**LOCKED prerequisite:** main_keyword set + keywords[] non-empty + topics[] non-empty BEFORE generate_draft.

**Competitor research (cloudstonedesigns-specific):** Before keyword commit, identify 3-5 competitor
pages ranking for the target query. Feed competitive signals into topicFullAuto context so CJGEO
goes deeper than surface-level content.

**Context prompt required in every generateDraft call:**
- Target audience: anxious homeowner / developer / business owner evaluating architecture services
- Journey scenario and stage (Scenario 1/2/3, stage name)
- Tone: calm, expert, trustworthy — never salesy
- Studio name: "Cloudstone Designs"

**Pipeline:**
1. Create CJGEO article → keywordFullAuto → topicFullAuto → generateDraft → pollDraft → adoptDraft
2. get_html → sanitizeCJGEO (copy bryanblog's sanitize-cjgeo.mjs to lib/)
3. Write to \`content/<pillar-slug>/<satellite-slug>.html\`
4. Update lib/articles.ts (source: 'cjgeo', cjgeoArticleId, mainKeyword, adoptedAt)
5. Rewrite page.tsx as async server component reading from content file
6. Build + push + CIC-verify
`.trim(),
    },

    deployToVercel: {
      description: "Deploy to Vercel. DEFERRED — do after CJGEO hydration + real photography.",
      howTo: `
**Status: DEFERRED.** Preconditions before deploying:
- At least one full pillar + all satellites CJGEO-hydrated
- Portfolio has real project photography (not placeholders)
- Contact form wired to Resend (not console.log)

When ready:
1. \`gh repo create boster00/cloudstonedesigns --public\`
2. \`git remote add origin https://github.com/boster00/cloudstonedesigns && git push -u origin main\`
3. Connect Vercel, set RESEND_API_KEY env var
4. CIC-verify live URL
`.trim(),
    },

    refRepoLayout: {
      description: "Reference: file layout in the cloudstonedesigns repo.",
      howTo: `
\`C:\\Users\\xsj70\\cloudstonedesigns\\\`

app/ — all routes (home, portfolio, about, people, contact, 3 pillar dirs each with [slug])
components/ — SiteNav, Footer, ProjectCard, ProjectGrid, ArticleShell, Cta, RelatedArticles,
              ContactForm, MarqueeText, PeopleCard
lib/ — projects.ts (6 placeholder projects), articles.ts (3 pillars, 26 satellites)
content/ — (created at CJGEO adoption time) <pillar-slug>/<satellite-slug>.html
app/api/contact/route.ts — POST handler (honeypot + validation, Resend TODO)
package.json — "dev:cloudstone": "next dev -p 3005"

Port registry: 3002=GuildOS · 3004=bryanblog · 3005=cloudstonedesigns
`.trim(),
    },
  },
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->
