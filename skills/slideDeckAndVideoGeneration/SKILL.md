# slideDeckAndVideoGeneration

<!-- GUILDOS-SKILL name="slideDeckAndVideoGeneration" -->

<!-- GUILDOS-DESC -->
Build branded presentations / slide decks / PPT-replacements AND videos from a single HTML/CSS(/React) substrate. Owns: making slide decks and presentations in HTML (self-contained, real tables, 16:9), turning gpt-image-2 / AI-generated images into slide backgrounds with editable text on top, and generating MP4 video from HTML/React via Remotion. Holds the LOCKED HTML-vs-PowerPoint decision, the deck-generator benchmark, the gpt-image-2 hybrid pattern, the Remotion video recipe, and the HTML→pptx escape hatch. Use for any 'make a deck / presentation / slides / PPT' or 'make a video from HTML' task.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/slideDeckAndVideoGeneration/SKILL.md -->
```markdown
---
name: slideDeckAndVideoGeneration
description: >-
  Build branded presentations / slide decks / PPT-replacements AND videos from a
  single HTML/CSS(/React) substrate. Owns: making slide decks and presentations
  in HTML (self-contained, real tables, 16:9), turning gpt-image-2 /
  AI-generated images into slide backgrounds with editable text on top, and
  generating MP4 video from HTML/React via Remotion. Holds the LOCKED
  HTML-vs-PowerPoint decision, the deck-generator benchmark, the gpt-image-2
  hybrid pattern, the Remotion video recipe, and the HTML→pptx escape hatch. Use
  for any 'make a deck / presentation / slides / PPT' or 'make a video from
  HTML' task.
---

# Slide Deck & Video Generation — from one HTML/React substrate (tactic: HTML/CSS + Remotion)

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/slideDeckAndVideoGeneration/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto slideDeckAndVideoGeneration <action>` (or read the source directly).

## Actions

- **buildDeck** — Build a branded, self-contained HTML deck end-to-end: 16:9 sections, brand tokens, real tables, gpt-image-2 backgrounds, render + overflow-check, inline to a single portable .html.
- **buildVideo** — Build an MP4 with Remotion (React for video): non-interactive setup, a Composition, frame-driven animation, render with bundled ffmpeg/Chromium, reusing the deck's brand tokens/components.
- **exportToPptx** — Escape hatch: convert HTML → editable .pptx, or rebuild as native .pptx via the pptx skill. The converter landscape + limits (pptx is OOXML, not HTML).
- **refArtifacts** — The reference build (MileCell) and the vendored templates: where the known-good deck/video skeletons live for reuse.
- **refBrandTokens** — The CSS-variable brand-token system + the reusable slide-component set, shared between deck and Remotion video so both stay on-brand by construction.
- **refEval** — The five-generator deck benchmark (speed · reliability · cost · quality · gpt-image-2 compat) and what each tool is actually best at.
- **refGotchas** — Environment + tooling gotchas: file:// mangling, CIC hidden-window throttle, ffmpeg/poppler absence, relative-path breakage, slide overflow.
- **refGptImageHybrid** — LOCKED pattern: use gpt-image-2 / AI-generated PNGs as full-bleed slide backgrounds with live, editable HTML text and tables on top.
- **refRemotionSkills** — The official Remotion Agent Skills, vendored at ./remotion-skills/ (SKILL.md + ~36 rules/*.md), plus how to install them into a live Remotion project.
- **refRenderVerify** — How to render + VISUALLY verify each output type: puppeteer (HTML→PNG/PDF + overflow check), PowerPoint COM (pptx→PNG, true fidelity), remotion still (video frames).
- **refStrategy** — LOCKED: HTML/CSS(/React) is the default substrate for decks AND video; the only exception is a client-editable .pptx requirement. The full rationale.
- **resolveSkillbook** — Overview + routing for an executing agent: how to pick deck vs video, the locked substrate decision, and where each recipe lives.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/engineering_decisions.md -->
```markdown
# engineering_decisions — slideDeckAndVideoGeneration

Non-native decisions. Native/obvious choices are omitted.

---

**2026-07-09 · On win-arm64, render HTML overlay → MP4 via Playwright frame-capture + ffmpeg, NOT Remotion.**
- Decision: `overlayCompositeRender.mjs` rasterizes an overlay page frame-by-frame
  with headless Playwright (page exposes `renderFrame(t)` + an `?export=1`
  transparent mode), then composites the PNG sequence over the source video with
  `ffmpeg overlay` (audio copied). This is the arm64 fallback for `buildVideo`.
- Rejected: Remotion's `renderMedia` (the skill's default video path).
- Why: Remotion's renderer is unsupported on win-arm64 (no arm64 headless-shell
  bundle — see reference_remotion_winarm64). Playwright ships arm64 Chromium and
  ffmpeg is present, so frame-capture + composite is the only local path. Because
  the overlay is already frame-driven off a master clock, the same component specs
  work in both engines with no rewrite; only the driver differs
  (`renderFrame(t)` ↔ `useCurrentFrame()`).
- Consequence if violated: `buildVideo` via Remotion silently fails / hangs on the
  operator's arm64 box; agents waste a cycle before falling back.
- Note: `--duration` on the CLI caps only frame *capture*; the output length still
  follows input 0 (the source video). Omit it for a full-length render.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/index.js -->
```js
/**
 * slideDeckAndVideoGeneration skill book — build branded PRESENTATIONS and VIDEO from a single
 * HTML/CSS(/React) substrate, with gpt-image-2 art as first-class backgrounds.
 *
 * ── WHY THIS EXISTS (the decision that forged it, LOCKED 2026-06-05) ─────────
 * We benchmarked five ways to turn an outline into a deck (Anthropic pptx skill /
 * python-pptx, hand-authored HTML, Google Slides API, Marp, Gamma) on identical
 * content (the MileCell proposal). Two were built + visually verified end-to-end;
 * the rest scored on projection. Result (axes: speed · reliability · cost ·
 * quality/fidelity · gpt-image-2 compatibility, /25):
 *     pptx skill (python-pptx) ... 25  — native-editable .pptx, real tables
 *     HTML (hand-authored) ....... 23  — highest design ceiling, best gpt-image-2
 *     Google Slides API .......... 23  (projected) — native Slides, needs OAuth + hosted imgs
 *     Marp ....................... 22  — free, fast, but pptx export is rasterized
 *     Gamma ...................... 17  (projected) — auto-design, paraphrases copy, paid export
 *
 * THE STRATEGIC CALL (LOCKED): **HTML/CSS(/React) is the default substrate for
 * BOTH decks and video.** It wins on the axes that compound for an agent-driven
 * shop:
 *   1. Ecosystem + documentation — the most-documented UI substrate on earth
 *      (reveal.js, GSAP/anime.js, D3/Chart.js, three.js, scroll-telling).
 *   2. LLM-generation affinity — models author and *iterate* HTML/CSS far more
 *      reliably than python-pptx's verbose EMU-coordinate object model. Every
 *      future "tweak slide 12 / restyle the table" is cheaper in HTML.
 *   3. One substrate → deck AND video. Remotion ("React for video") renders the
 *      SAME components to MP4. A deck and its animated/narrated video share one
 *      codebase — you don't rebuild, you re-target.
 *   4. gpt-image-2 art is native: `background-image` / `<img>` + live, editable
 *      text and tables on top. (Proven on the MileCell cover/divider/close.)
 *   5. Highest design ceiling — pixel-perfect, web-grade.
 *
 * THE ONE EXCEPTION (when to NOT use HTML): if the deliverable MUST be a
 * natively client-editable .pptx (the client will open it in PowerPoint and edit
 * text/tables themselves). Then use the Anthropic pptx skill (python-pptx). For a
 * proposal you present/send, the client almost never edits your deck — a gorgeous
 * PDF / web link / short video beats an editable .pptx for winning, and the
 * editability YOU care about (iterating it yourself) is better in HTML anyway.
 *
 * Rejected-as-default (kept as fallbacks): pptx skill → editable-.pptx escape
 * hatch only; Marp → quick diffable drafts; Google Slides API → when it must live
 * in Slides (see [SB:_->googleSlides]); Gamma → throwaway first draft, never for a
 * numbers-exact proposal (it paraphrases copy and restructures tables).
 *
 * ── CHOSEN STACK (LOCKED) ───────────────────────────────────────────────────
 * - Decks: hand-authored single-file HTML/CSS, fixed 16:9 `<section class="slide">`,
 *   brand tokens as CSS variables, real `<table>`s, gpt-image-2 backgrounds,
 *   delivered SELF-CONTAINED (images inlined as base64 → opens offline anywhere).
 * - Video: Remotion (bundles its own ffmpeg + Chromium → zero system setup),
 *   reusing the deck's brand tokens/components. Official Remotion Agent Skills are
 *   vendored at ./remotion-skills/ and pointed to from refRemotionSkills.
 * - Render + VISUAL verify is mandatory: puppeteer (HTML→PNG/PDF), PowerPoint COM
 *   (pptx→PNG, true fidelity), `npx remotion still` (video frames).
 */

export const skillBook = {
  id: "slideDeckAndVideoGeneration",
  title: "Slide Deck & Video Generation — from one HTML/React substrate (tactic: HTML/CSS + Remotion)",
  description:
    "Build branded presentations / slide decks / PPT-replacements AND videos from a single HTML/CSS(/React) substrate. Owns: making slide decks and presentations in HTML (self-contained, real tables, 16:9), turning gpt-image-2 / AI-generated images into slide backgrounds with editable text on top, and generating MP4 video from HTML/React via Remotion. Holds the LOCKED HTML-vs-PowerPoint decision, the deck-generator benchmark, the gpt-image-2 hybrid pattern, the Remotion video recipe (vendored official Remotion skills), and the HTML→pptx escape hatch. Use for any 'make a deck / presentation / slides / PPT' or 'make a video from HTML' task.",
  steps: [],
  toc: {
    resolveSkillbook: {
      description:
        "Overview + routing for an executing agent: how to pick deck vs video, the locked substrate decision, and where each recipe lives.",
      howTo: `
This skill book makes branded **decks** and **videos** from ONE substrate:
HTML/CSS for slides, the same components via Remotion for video.

**Decide first (LOCKED — see refStrategy):**
- Need slides/a deck/a "PPT"? → default to HTML ([SB:slideDeckAndVideoGeneration->buildDeck]).
  Only reach for native-editable .pptx (the Anthropic pptx skill) if the CLIENT
  must edit text/tables in PowerPoint themselves → [SB:slideDeckAndVideoGeneration->exportToPptx].
- Need a video / animated intro / motion graphic? → Remotion ([SB:slideDeckAndVideoGeneration->buildVideo]).
- Need it to live in Google Slides (collaborative)? → [SB:_->googleSlides].

**Routing.**
- The locked HTML-vs-pptx decision + why → [SB:slideDeckAndVideoGeneration->refStrategy] (READ FIRST).
- The five-generator benchmark + scores → [SB:slideDeckAndVideoGeneration->refEval].
- Build an HTML deck end-to-end → [SB:slideDeckAndVideoGeneration->buildDeck].
- Brand tokens + reusable slide components (shared with video) → [SB:slideDeckAndVideoGeneration->refBrandTokens].
- gpt-image-2 / AI art as backgrounds with editable content on top → [SB:slideDeckAndVideoGeneration->refGptImageHybrid] (LOCKED).
- Build a video with Remotion → [SB:slideDeckAndVideoGeneration->buildVideo].
- The vendored official Remotion skills + how to install/use → [SB:slideDeckAndVideoGeneration->refRemotionSkills].
- Convert HTML → editable .pptx (escape hatch) → [SB:slideDeckAndVideoGeneration->exportToPptx].
- Render + VISUALLY verify any output → [SB:slideDeckAndVideoGeneration->refRenderVerify].
- Reference build (MileCell) + vendored templates → [SB:slideDeckAndVideoGeneration->refArtifacts].
- Gotchas that will bite you → [SB:slideDeckAndVideoGeneration->refGotchas] (READ before first run).

**Positive-law note.** Per CLAUDE.md all execution goes through a skill-book howTo
or a weapon. The runnable assets for this book are the vendored templates
(./templates/deck, ./templates/video) and the inline node/puppeteer/PowerPoint-COM
recipes below — invoked directly, not hand-rolled from scratch.
`.trim(),
    },

    refStrategy: {
      description:
        "LOCKED: HTML/CSS(/React) is the default substrate for decks AND video; the only exception is a client-editable .pptx requirement. The full rationale.",
      howTo: `
**LOCKED 2026-06-05 — the substrate decision.**

Default substrate for presentations AND video = **HTML/CSS(/React)**. Five reasons,
in priority order:
1. **LLM affinity (decisive for an agent shop).** Models generate and *iterate*
   HTML/CSS far more reliably than python-pptx's EMU object model. Cheaper, faster,
   lower-error on every future edit. This compounds over hundreds of decks.
2. **One substrate → deck + video.** Remotion renders the same components to MP4.
   The slide components you build today are already video components.
3. **Ecosystem + documentation.** The most-documented UI substrate that exists.
4. **gpt-image-2 native.** AI art as backgrounds with editable text/tables on top
   is trivial in HTML (see refGptImageHybrid).
5. **Highest design ceiling** — pixel-perfect, web-grade.

**The ONE exception → use the Anthropic pptx skill (python-pptx) instead:** the
deliverable must be a **natively client-editable .pptx** (the client opens it in
PowerPoint and edits text/tables). For a proposal you present/send, this rarely
applies — a PDF / web link / video wins the deal, and the editability you care
about (iterating it yourself) is better in HTML.

**Decision rule:** _Choose python-pptx only if "client-editable native .pptx" is a
hard requirement. Otherwise HTML — it wins on every axis plus carries free
video/interactivity optionality._

**The compounding play:** brand tokens as CSS variables + a small set of slide
components (cover / divider / comparison-table / pricing-table / cards / close)
become a reusable asset that gets better each use AND feeds Remotion video off the
same parts. One substrate, on-brand by construction. See refBrandTokens.

**The one risk to engineer around:** HTML has no native slide canvas, so content
overflow/crowding is the real failure mode. Mitigation is non-optional: fixed 16:9
slide containers + an automated overflow check in the render pass (refRenderVerify).
`.trim(),
    },

    refEval: {
      description:
        "The five-generator deck benchmark (speed · reliability · cost · quality · gpt-image-2 compat) and what each tool is actually best at.",
      howTo: `
Benchmarked on identical content (the MileCell 26-slide proposal). Scores /5 per
axis, /25 total. Rows marked (built) were rendered + visually verified; (proj) were
scored on projection.

| Tool | Speed | Reliab | Cost | Quality | gpt-image-2 | Total | Output |
|---|--|--|--|--|--|--|---|
| pptx skill (python-pptx) (built) | 5 | 5 | 5 | 5 | 5 | 25 | native-editable .pptx |
| HTML hand-authored (built) | 4 | 4 | 5 | 5 | 5 | 23 | .html + .pdf (+img pptx) |
| Google Slides API (proj) | 4 | 4 | 5 | 5 | 5 | 23 | native Google Slides |
| Marp (built) | 5 | 4 | 5 | 4 | 4 | 22 | .html/.pdf + img .pptx |
| Gamma (proj) | 5 | 3 | 3 | 4 | 2 | 17 | .pptx/PDF (paid export) |

Best-at:
- **pptx skill** — when you need a real editable PowerPoint with exact numbers/tables.
- **HTML** — highest design ceiling + cleanest gpt-image-2 integration + the video path.
- **Slides API** — when it must live in Google Slides (shareable/collaborative).
- **Marp** — quick, free, diffable drafts; accept rasterized pptx.
- **Gamma** — throwaway first draft only; it paraphrases copy and restructures tables,
  so never for a numbers-exact proposal.

gpt-image-2 specifically: pptx-skill, HTML, and Slides API tie at 5/5 (AI art as
background + content on top). The differentiator is the container: native .pptx vs
web/PDF vs Google Slides. Gamma (won't pin exact backgrounds) and Marp (flattens on
pptx export) are weaker.
`.trim(),
    },

    buildDeck: {
      description:
        "Build a branded, self-contained HTML deck end-to-end: 16:9 sections, brand tokens, real tables, gpt-image-2 backgrounds, render + overflow-check, inline to a single portable .html.",
      howTo: `
**Recipe — HTML deck (the default deck path).**

1. **Structure.** One file. Each slide is a fixed-size 16:9 \`<section class="slide">\`
   (1280×720 or 1600×900 logical). Slides in outline order. Use the vendored
   reference at \`./templates/deck/milecell-example.html\` as the starting skeleton.
2. **Brand tokens.** Define CSS variables once (\`:root{ --navy:#1E3A5F; --teal:#1CA9A6;
   --ink:#333; --risk:#B5483D; ... }\`) and reference everywhere. See refBrandTokens.
3. **Slide types.** Full-navy cover/divider/close (white text); white content slides
   with a thin navy header band. A single muted red (\`--risk\`) ONLY for
   reverse-cost / risk callouts. Subtle hexagon/molecular motif via CSS/SVG.
4. **Tables are real tables.** Comparison + pricing tables = HTML \`<table>\` with a
   navy header row, zebra striping, optional color-coded cells. Keep every number
   verbatim. ≤6 bullets/slide.
5. **gpt-image-2 backgrounds** where you have art → refGptImageHybrid.
6. **Render + VISUAL verify (mandatory).** Per-slide PNGs via puppeteer; run the
   OVERFLOW CHECK; eyeball cover/tables/pricing. See refRenderVerify. Never ship
   on a structural check alone.
7. **Deliver SELF-CONTAINED.** Inline every referenced image as a base64 data URI so
   the deck is a single .html that opens offline anywhere (no server, no broken
   \`../\` paths). Inline transform (no scratch file, run via \`node\` heredoc):

   \`\`\`js
   const fs=require('fs'),path=require('path');
   const base='<deck-folder>';                 // folder the HTML + images live in
   let html=fs.readFileSync(base+'/deck.html','utf8');
   const refs=[...new Set([...html.matchAll(/\\.\\.\\/([\\w.\\-]+\\.png)/g)].map(m=>m[1]))];
   for(const name of refs){
     const b64=fs.readFileSync(path.join(base,name)).toString('base64');
     html=html.split('../'+name).join('data:image/png;base64,'+b64);
   }
   fs.writeFileSync('<out>/Deck.html', html);   // 0 external refs remain
   \`\`\`
   (Verify: 0 remaining \`../*.png\` refs; file opens with backgrounds intact.)

8. **Other outputs from the same file:** PDF via headless Chrome / puppeteer
   \`page.pdf()\` with \`@page{size:1280px 720px;margin:0}\`; an editable .pptx only if
   required → exportToPptx; a video → buildVideo.

Deliver to the path the USER reviews (e.g. their Downloads), not a worktree-only
location.
`.trim(),
    },

    refBrandTokens: {
      description:
        "The CSS-variable brand-token system + the reusable slide-component set, shared between deck and Remotion video so both stay on-brand by construction.",
      howTo: `
**Brand tokens (CSS variables) — single source of truth, shared deck↔video.**

\`\`\`css
:root{
  --navy:#1E3A5F;   /* titles, headers, divider/cover/close backgrounds */
  --teal:#1CA9A6;   /* accents, highlights, key numbers, TOTAL row */
  --ink:#333333;    /* body text on white */
  --paper:#ffffff;  /* content backgrounds */
  --risk:#B5483D;   /* muted red — ONLY for reverse-cost / risk callouts */
  --font: "Inter", system-ui, Arial, sans-serif;
}
\`\`\`

**Reusable slide components** (build once, reuse across decks; mirror them as React
for Remotion so video inherits the same look):
- cover (full-navy, eyebrow + two-tone title + accent rule + subtitle + footer)
- divider (full-navy section break)
- comparison-table (navy header, zebra, color-coded cells, recommendation +
  warning callout cards)
- pricing-table (right-aligned hours/cost, teal TOTAL row)
- cards (2–4 rounded feature cards)
- close (full-navy, four-reasons cards, teal next-step banner)

Rule: tokens live in ONE place; deck CSS and Remotion components both import them.
Editing a token re-skins decks AND videos. This is the compounding asset.
`.trim(),
    },

    refGptImageHybrid: {
      description:
        "LOCKED pattern: use gpt-image-2 / AI-generated PNGs as full-bleed slide backgrounds with live, editable HTML text and tables on top.",
      howTo: `
**LOCKED — the gpt-image-2 hybrid (HTML's killer advantage).**

gpt-image-2 produces gorgeous on-brand art but flat images aren't editable. In HTML
you get both: AI art as the background, real editable content on top.

- **Full-slide background:** \`section.cover{ background:url('../01-title-bg.png')
  center/cover no-repeat; }\` then place \`<h1>\`, tables, bullets as normal HTML on top.
  Best on cover / dividers / close; keep content slides clean white (optionally a
  faint content-bg) so tables stay legible.
- **Asset on a slide:** \`<img>\` with \`object-fit:cover\` inside a sized box.
- **Where the art comes from:** the openai_images weapon (see [SB:_->openai_images]
  and the geo-block workaround in memory) generates the PNGs; drop them next to the
  HTML and reference relatively, then inline for delivery (buildDeck step 7).
- **Why HTML wins here:** pptx-skill and Slides API also score 5/5 on this, but HTML
  needs no hosting (Slides API requires a public image URL) and no object model
  (pptx). Gamma can't pin exact backgrounds; Marp flattens on pptx export.

Proven on the MileCell deck: \`01-title-bg.png\` (cover), \`03-divider-bg.png\`
(dividers), \`04-closing-bg.png\` (close) render as backgrounds with the live deck
title/tables on top. Reference: ./templates/deck/milecell-example.html.
`.trim(),
    },

    buildVideo: {
      description:
        "Build an MP4 with Remotion (React for video): non-interactive setup, a Composition, frame-driven animation, render with bundled ffmpeg/Chromium, reusing the deck's brand tokens/components.",
      howTo: `
**Recipe — Remotion video (the HTML→video path).** On x64 / macOS / Linux,
Remotion bundles its own ffmpeg AND Chromium, so no system setup is needed there.
**PLATFORM CAVEAT (LOCKED 2026-06-05): Remotion does NOT run on Windows ARM64.**
Neither its Chrome Headless Shell nor its native Rust compositor ship a win-arm64
build (verified errors: "Chrome Headless Shell is not available for Windows for
arm64" and "Unsupported architecture on Windows: arm64"). The
\`--browser-executable\` flag fixes the headless-shell download but NOT the
compositor gap. On a win-arm64 box, render on an x64 machine, CI (GitHub Actions),
a cloud agent, or Remotion Lambda — the scaffold + vendored skills are portable and
render fine there. For a quick LOCAL video on win-arm64, capture HTML frames via
puppeteer (its Chromium DOES work here) and stitch with ffmpeg.

1. **Set up non-interactively** (do NOT use \`npm create video\` — it's interactive).
   Start from the vendored scaffold \`./templates/video/\` (package.json, tsconfig,
   src/index.ts, src/Root.tsx, src/<Composition>.tsx). Copy it to a project folder
   and \`npm install\` (Remotion auto-downloads Chromium on first render).
2. **Composition.** Register one \`<Composition id="..." component={...}
   durationInFrames={180} fps={30} width={1920} height={1080} />\` in Root.tsx.
3. **Animate by frame.** Use \`useCurrentFrame\`, \`interpolate\`, \`spring\`,
   \`<Sequence>\`, \`<AbsoluteFill>\`. Load assets from \`public/\` via \`staticFile('x.png')\`
   — drop a gpt-image-2 background in \`public/\` for an on-brand intro.
4. **Reuse the deck's look.** Import the SAME brand tokens / mirror the slide
   components (refBrandTokens) so the video matches the deck.
5. **Render:** \`npx remotion render <CompId> out/video.mp4\` (H.264 1080p).
6. **VISUAL verify (mandatory):** render stills with
   \`npx remotion still <CompId> out/frame-090.png --frame=90\` (a few frames) and
   eyeball that the background art + animated text are present. Then look at the MP4.
7. **Lean on the vendored Remotion skills** for animations/transitions/charts/
   captions/audio/three.js patterns → refRemotionSkills.

Reference build: a 6s MileCell branded intro (gpt-image-2 hex background + spring
title slide-up + teal accent draw) — scaffold at ./templates/video/.
`.trim(),
    },

    refRemotionSkills: {
      description:
        "The official Remotion Agent Skills, vendored at ./remotion-skills/ (SKILL.md + ~36 rules/*.md), plus how to install them into a live Remotion project.",
      howTo: `
**Official Remotion Agent Skills — vendored + how to obtain.**

- **Vendored copy (read these for current best practices):**
  \`libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/SKILL.md\` + \`remotion-skills/rules/*.md\`
  (~36 rule files: animations, text-animations, transitions, sequencing, timing,
  charts, captions/subtitles/SRT, audio + audio-visualization, voiceover, images,
  videos, gifs, lottie, google-fonts/local-fonts, tailwind, three.js (3d),
  html-in-canvas, transparent-videos, maplibre, ffmpeg, calculate-metadata, etc.,
  plus rules/assets/*.tsx example components).
- **Source:** https://github.com/remotion-dev/skills
- **Install into a LIVE Remotion project** (so a render build has the rules in
  context): \`npx skills add remotion-dev/skills\` — or choose "add skills" when
  scaffolding via \`bun create video\`. This is what the "remotion best skill"
  recommendation refers to: Remotion's own best-practices Agent Skill, the
  most-installed video skill.
- **When building a video (buildVideo), consult the matching rule file first** (e.g.
  \`rules/transitions.md\` before adding a scene transition, \`rules/charts-*\` /
  \`rules/text-animations.md\` for animated data or kinetic type). They encode the
  Remotion-specific gotchas that generic React knowledge misses.

Update the vendored copy by re-cloning the repo and re-copying
\`skills/remotion/\` over \`remotion-skills/\`.
`.trim(),
    },

    exportToPptx: {
      description:
        "Escape hatch: convert HTML → editable .pptx, or rebuild as native .pptx via the pptx skill. The converter landscape + limits (pptx is OOXML, not HTML).",
      howTo: `
**HTML → .pptx (the editable-PowerPoint escape hatch).**

Reality check: a .pptx is NOT HTML under the hood — it's Office Open XML (a zip of
DrawingML/XML parts). So "convert HTML to pptx" means re-mapping DOM/CSS onto
PowerPoint shapes. It works, with caveats.

Options, best-first for our case:
1. **Rebuild as native .pptx via the Anthropic pptx skill (python-pptx).** For a
   numbers-exact proposal where the client must edit it, this beats auto-conversion
   — real editable tables, exact figures, full brand control. Score 25/25 in refEval.
   This is the recommended editable-.pptx path.
2. **html2pptx** (Node lib + CLI, \`npm install html2pptx\`; also a Python variant) —
   automatic HTML→pptx: text boxes with styling, lists, positioned elements, SVG →
   native PowerPoint shapes, hyperlinks. Good for mostly-text/box layouts.
   https://github.com/abdelkrimkr/html2pptx
3. **dom-to-pptx** (client-side JS) — converts a DOM element into a fully editable
   slide, preserving gradients, shadows, rounded images, responsive layouts.
   https://github.com/atharva9167j/dom-to-pptx
4. **SaaS / API** — Nutrient DWS HTML-to-PPTX API, GroupDocs, online-convert — when
   you want it hosted/zero-install.

**Limits (all converters):** complex CSS (animations, transforms), JS-generated
dynamic content, and complex flexbox/grid are approximated or dropped. gpt-image-2
backgrounds survive as images (fine) but become non-editable picture fills. If
fidelity of exact tables/numbers matters → prefer option 1 (rebuild), not auto-convert.

A pragmatic alternative to an "editable" pptx: a deck of full-bleed slide images
(render each HTML slide to PNG, drop one per slide into a .pptx). Looks perfect,
not editable — same trade-off as Marp's pptx export.
`.trim(),
    },

    refRenderVerify: {
      description:
        "How to render + VISUALLY verify each output type: puppeteer (HTML→PNG/PDF + overflow check), PowerPoint COM (pptx→PNG, true fidelity), remotion still (video frames).",
      howTo: `
**Render + visual verification (mandatory before any success claim).**

- **HTML → per-slide PNG + PDF:** puppeteer (\`npm i puppeteer\` — bundles Chromium).
  Load the file via \`file://\`, set viewport to slide size, screenshot each
  \`.slide\`. PDF via \`page.pdf()\`. Then **Read the PNGs and look** (cover, every
  table, the pricing total).
- **OVERFLOW CHECK (the HTML failure mode):** in the same puppeteer pass, for each
  \`.slide\` assert \`scrollHeight <= clientHeight\` (and width). Flag any slide whose
  content exceeds the 16:9 box — that's the crowding/clipping bug. Fix CSS, re-render.
- **pptx → PNG (true fidelity, PowerPoint installed):** PowerPoint COM via PowerShell —
  \`$ppt=New-Object -ComObject PowerPoint.Application; $pres=$ppt.Presentations.Open($src,$true,$false,$false); $pres.Export($dir,"PNG",1600,900); $pres.Close(); $ppt.Quit()\`.
  Guaranteed slide order, real PowerPoint rendering. Use this to verify pptx-skill
  decks (don't trust python-pptx structural checks alone).
- **Video → frames + MP4:** \`npx remotion still <Comp> out/frame-N.png --frame=N\` for
  a few frames; confirm bg art + text; then watch the MP4.
- **Showing the user in their browser:** serve the folder over localhost
  (\`python -m http.server <port> --bind <REDACTED-IP>\` from the folder that is the
  parent of the relative image refs) and open \`http://<REDACTED-IP>:<port>/...\`. Do NOT
  rely on CIC \`navigate\` with \`file://\` (it gets mangled to https://) — see refGotchas.

Discipline: a structural/parse check is NOT verification. Look at the pixels in the
user's channel before claiming done.
`.trim(),
    },

    refArtifacts: {
      description:
        "The reference build (MileCell) and the vendored templates: where the known-good deck/video skeletons live for reuse.",
      howTo: `
**Vendored templates (reuse these, don't start from scratch):**
- \`./templates/deck/milecell-example.html\` — known-good HTML deck: 16:9 sections,
  CSS-variable brand tokens, real comparison + pricing tables (TOTAL 628h / $78,500),
  gpt-image-2 backgrounds on cover/divider/close. Best starting skeleton for a new deck.
- \`./templates/video/\` — Remotion scaffold: package.json, tsconfig.json, src/index.ts,
  src/Root.tsx, src/MileCellIntro.tsx (branded intro: hex bg + spring title + teal
  accent draw). Copy → \`npm install\` → \`npx remotion render\`.
- \`./remotion-skills/\` — official Remotion Agent Skills (refRemotionSkills).

**Reference outputs (the benchmark deliverables, on the build machine, not committed):**
- HTML deck: Downloads/milecell-deck/out-html/milecell.html (+ self-contained
  Downloads/MileCell-Proposal-Deck.html with images inlined) + renders/html/slide-*.png.
- pptx-skill deck: Downloads/milecell-deck/out-pptx-skill/ + renders/pptx-skill/.
- Marp deck: Downloads/milecell-deck/out-marp/.
- Remotion intro: Downloads/milecell-deck/out-video-remotion/.

When starting a new project, copy a template out, swap brand tokens + content, render,
overflow-check, deliver self-contained to the user's review path.
`.trim(),
    },

    refGotchas: {
      description:
        "Environment + tooling gotchas: file:// mangling, CIC hidden-window throttle, ffmpeg/poppler absence, relative-path breakage, slide overflow.",
      howTo: `
**Read before first run.**
- **CIC \`navigate\` mangles \`file://\`** → it prepends https:// ("https://file:///C:/...")
  and the page won't load. To show local files in the user's browser, serve over
  localhost and open the http URL (refRenderVerify). puppeteer/headless Chrome handle
  \`file://\` fine — the issue is only CIC's navigate.
- **CIC hidden/occluded window throttles the renderer.** If a CIC tab's
  \`document.visibilityState === 'hidden'\` (window minimized or behind another),
  Chrome throttles it: screenshots time out ("renderer may be frozen") and SPAs never
  paint. Fix: have the user foreground that Chrome window (resize_window alone does NOT
  un-occlude it). Diagnose with a quick \`javascript_tool\` check of visibilityState.
- **Remotion has NO Windows-ARM64 build** (this dev machine is win-arm64). Both its
  Chrome Headless Shell and its native Rust compositor error out
  ("Unsupported architecture on Windows: arm64"). Render Remotion on x64 / CI /
  cloud / Remotion Lambda. For a LOCAL win-arm64 video, capture HTML frames via
  puppeteer (its Chromium works here) and stitch with ffmpeg.
- **ffmpeg is NOT on PATH** — on x64 Remotion bundles its own; a DIY
  puppeteer-frames→ffmpeg pipeline (or any win-arm64 render) needs ffmpeg installed
  (ffmpeg does ship win-arm64 builds).
- **poppler (pdftoppm) is NOT installed** — so the Read tool can't rasterize PDFs.
  Render to PNG via puppeteer or PowerPoint COM instead, then Read the PNGs.
- **Relative \`../bg.png\` breaks when the .html is moved** (and under \`file://\`). Always
  deliver self-contained (inline base64, buildDeck step 7).
- **Slide overflow/crowding** is HTML's one real failure mode (no native slide canvas).
  Always run the overflow check in the render pass (refRenderVerify).
- **Gamma** needs a normal, foregrounded Chrome window (same throttle issue) AND gates
  pptx export behind a paid plan — don't rely on it for a deliverable.
`.trim(),
    },
  },
};
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/overlayCompositeRender.mjs -->
```js
#!/usr/bin/env node
/**
 * overlayCompositeRender — rasterize an HTML/CSS/SVG overlay page to transparent
 * frames with headless Playwright, then composite them over a source video with
 * ffmpeg (original audio preserved). This is the arm64-Windows fallback for
 * slideDeckAndVideoGeneration.buildVideo, where Remotion's renderer is unsupported
 * (see reference_remotion_winarm64 / engineering_decisions.md).
 *
 * The overlay page must expose a global `renderFrame(tSeconds)` and honour an
 * `?export=1` mode (transparent stage, source <video> hidden, controls hidden,
 * rAF loop NOT started — the renderer drives renderFrame(t) per frame).
 *
 * Usage:
 *   node overlayCompositeRender.mjs --url <pageUrl> --video <src.mp4> --out <out.mp4>
 *        [--fps 30] [--w 544] [--h 960] [--duration <sec>] [--keep-frames]
 */
import { chromium } from 'playwright';
import { execFileSync, spawnSync } from 'node:child_process';
import { mkdtempSync, rmSync, mkdirSync } from 'node:fs';
import { tmpdir } from 'node:os';
import { join } from 'node:path';

function arg(name, def=null){ const i=process.argv.indexOf('--'+name); return i>=0 ? process.argv[i+1] : def; }
const has = name => process.argv.includes('--'+name);

const url = arg('url'); const video = arg('video'); const out = arg('out');
const fps = +arg('fps', 30); const W = +arg('w', 544); const H = +arg('h', 960);
if(!url || !video || !out){ console.error('need --url --video --out'); process.exit(1); }

function probeDuration(v){
  const r = execFileSync('ffprobe', ['-v','error','-show_entries','format=duration','-of','default=nw=1:nk=1', v]).toString().trim();
  return parseFloat(r);
}
const duration = arg('duration') ? +arg('duration') : probeDuration(video);
const totalFrames = Math.round(duration*fps);
const framesDir = mkdtempSync(join(tmpdir(), 'overlay-frames-'));
console.log(`[render] ${totalFrames} frames @ ${fps}fps (${duration}s) → ${framesDir}`);

const browser = await chromium.launch({ headless:true });
const page = await browser.newPage({ viewport:{ width:W, height:H }, deviceScaleFactor:1 });
await page.goto(url + (url.includes('?')?'&':'?') + 'export=1', { waitUntil:'networkidle' });
await page.waitForFunction(() => typeof window.renderFrame === 'function');
const stage = page.locator('#stage');

const t0 = Date.now();
for(let f=0; f<totalFrames; f++){
  const t = f/fps;
  await page.evaluate(tt => window.renderFrame(tt), t);
  await stage.screenshot({ path: join(framesDir, `f${String(f).padStart(5,'0')}.png`), omitBackground:true });
  if(f % 300 === 0) console.log(`[render] frame ${f}/${totalFrames} (${((Date.now()-t0)/1000).toFixed(0)}s)`);
}
await browser.close();
console.log(`[render] captured ${totalFrames} frames in ${((Date.now()-t0)/1000).toFixed(0)}s; compositing…`);

const ff = spawnSync('ffmpeg', [
  '-y',
  '-i', video,
  '-framerate', String(fps), '-i', join(framesDir, 'f%05d.png'),
  '-filter_complex', '[0:v][1:v]overlay=0:0:format=auto[v]',
  '-map', '[v]', '-map', '0:a?',
  '-c:v', 'libx264', '-pix_fmt', 'yuv420p', '-crf', '18', '-preset', 'medium',
  '-c:a', 'copy', '-movflags', '+faststart',
  out,
], { stdio:'inherit' });

if(!has('keep-frames')) rmSync(framesDir, { recursive:true, force:true });
if(ff.status !== 0){ console.error('[render] ffmpeg failed'); process.exit(ff.status||1); }
console.log(`[render] done → ${out}`);
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/SKILL.md -->
```markdown
---
name: remotion-best-practices
description: Best practices for Remotion - Video creation in React
metadata:
  tags: remotion, video, react, animation, composition
---

## When to use

Use this skills whenever you are dealing with Remotion code to obtain the domain-specific knowledge.

## New project setup

When in an empty folder or workspace with no existing Remotion project, scaffold one using:

```bash
npx create-video@latest --yes --blank --no-tailwind my-video
```

Replace `my-video` with a suitable project name.

## Designing a video

Animate properties using `useCurrentFrame()` and `interpolate()`. Use Easing to customize the timing of the animation.

```tsx
import { useCurrentFrame, Easing } from "remotion";

export const FadeIn = () => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();

  const opacity = interpolate(frame, [0, 2 * fps], [0, 1], {
    extrapolateRight: "clamp",
    extrapolateLeft: "clamp",
    easing: Easing.bezier(0.16, 1, 0.3, 1),
  });

  return <div style={{ opacity }}>Hello World!</div>;
};
```

CSS transitions or animations are FORBIDDEN - they will not render correctly.  
Tailwind animation class names are FORBIDDEN - they will not render correctly.

Place assets in the `public/` folder at your project root.

Use `staticFile()` to reference files from the `public/` folder.

Add images using the `<Img>` component:

```tsx
import { Img, staticFile } from "remotion";

export const MyComposition = () => {
  return <Img src={staticFile("logo.png")} style={{ width: 100, height: 100 }} />;
};
```

Add videos using the `<Video>` component from `@remotion/media`:

```tsx
import { Video } from "@remotion/media";
import { staticFile } from "remotion";

export const MyComposition = () => {
  return <Video src={staticFile("video.mp4")} style={{ opacity: 0.5 }} />;
};
```

Add audio using the `<Audio>` component from `@remotion/media`:

```tsx
import { Audio } from "@remotion/media";
import { staticFile } from "remotion";

export const MyComposition = () => {
  return <Audio src={staticFile("audio.mp3")} />;
};
```

Assets can be also referenced as remote URLs:

```tsx
import { Video } from "@remotion/media";

export const MyComposition = () => {
  return <Video src="https://remotion.media/video.mp4" />
};
```

To delay content wrap it in `<Sequence>` and use `from`.
To limit the duration of an element, use `durationInFrames` of `<Sequence>`.
`<Sequence>` by default is an absolute fill. For inline content, use `layout="none"`.

```tsx
import { Sequence } from "remotion";

export const Title = () => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();

  const opacity = interpolate(frame, [0, 2 * fps], [0, 1], {
    extrapolateRight: "clamp",
    extrapolateLeft: "clamp",
    easing: Easing.bezier(0.16, 1, 0.3, 1),
  });

  return <div style={{ opacity }}>Title</div>;
};

export const Subtitle = () => {
  return <div>Subtitle</div>;
};

const Main = () => {
  const {fps} = useVideoConfig();

  return (
    <AbsoluteFill>
      <Sequence>
        <Background />
      </Sequence>
      <Sequence from={1 * fps} durationInFrames={2 * fps} layout="none">
        <Title />
      </Sequence>
      <Sequence from={2 * fps} durationInFrames={2 * fps} layout="none">
        <Subtitle />
      </Sequence>
    </AbsoluteFill>
  );
}
```

The width, height, fps, and duration of a video is defined in `src/Root.tsx`:

```tsx
import { Composition } from "remotion";
import { MyComposition } from "./MyComposition";

export const RemotionRoot = () => {
  return (
    <Composition
      id="MyComposition"
      component={MyComposition}
      durationInFrames={100}
      fps={30}
      width={1080}
      height={1080}
    />
  );
};
```

Metadata can also be calculated dynamically:

```tsx
import { Composition, CalculateMetadataFunction } from "remotion";
import { MyComposition, MyCompositionProps } from "./MyComposition";

const calculateMetadata: CalculateMetadataFunction<
  MyCompositionProps
> = async ({ props, abortSignal }) => {
  const data = await fetch(`https://api.example.com/video/${props.videoId}`, {
    signal: abortSignal,
  }).then((res) => res.json());

  return {
    durationInFrames: Math.ceil(data.duration * 30),
    props: {
      ...props,
      videoUrl: data.url,
    },
    width: 1080,
    height: 1080,
  };
};

export const RemotionRoot = () => {
  return (
    <Composition
      id="MyComposition"
      component={MyComposition}
      fps={30}
      width={1080}
      height={1080}
      defaultProps={{ videoId: "abc123" }}
      calculateMetadata={calculateMetadata}
    />
  );
};
```

## Starting preview

Start the Remotion Studio to preview a video:

```bash
npx remotion studio
```

## Optional: one-frame render check

You can render a single frame with the CLI to sanity-check layout, colors, or timing.  
Skip it for trivial edits, pure refactors, or when you already have enough confidence from Studio or prior renders.

```bash
npx remotion still [composition-id] --scale=0.25 --frame=30
```

At 30 fps, `--frame=30` is the one-second mark (`--frame` is zero-based).

## Captions

When dealing with captions or subtitles, load the [./rules/subtitles.md](./rules/subtitles.md) file for more information.

## Using FFmpeg

For some video operations, such as trimming videos or detecting silence, FFmpeg should be used. Load the [./rules/ffmpeg.md](./rules/ffmpeg.md) file for more information.

## Silence detection

When needing to detect and trim silent segments from video or audio files, load the [./rules/silence-detection.md](./rules/silence-detection.md) file.

## Audio visualization

When needing to visualize audio (spectrum bars, waveforms, bass-reactive effects), load the [./rules/audio-visualization.md](./rules/audio-visualization.md) file for more information.

## Sound effects

When needing to use sound effects, load the [./rules/sfx.md](./rules/sfx.md) file for more information.

## 3D content

See [rules/3d.md](rules/3d.md) for 3D content in Remotion using Three.js and React Three Fiber.

## Advanced audio

See [rules/audio.md](rules/audio.md) for advanced audio features like trimming, volume, speed, pitch.

## Dynamic duration, dimensions and data

See [rules/calculate-metadata.md](rules/calculate-metadata.md) for dynamically set composition duration, dimensions, and props.

## Advanced compositions

See [rules/compositions.md](rules/compositions.md) for how to define stills, folders, default props and for how to nest compositions.

## Google Fonts

Is the recommended way to load fonts in Remotion. See [rules/google-fonts.md](rules/google-fonts.md) for how to load Google Fonts.

## Local fonts

See [rules/local-fonts.md](rules/local-fonts.md) for how to load local fonts.

## Getting audio duration

See [rules/get-audio-duration.md](rules/get-audio-duration.md) for getting the duration of an audio file in seconds with Mediabunny.

## Getting video dimensions

See [rules/get-video-dimensions.md](rules/get-video-dimensions.md) for getting the width and height of a video file with Mediabunny.

## Getting video duration

See [rules/get-video-duration.md](rules/get-video-duration.md) for getting the duration of a video file in seconds with Mediabunny.

## GIFs

See [rules/gifs.md](rules/gifs.md) for how to display GIFs synchronized with Remotion's timeline.

## Advanced Images

See [rules/images.md](rules/images.md) for sizing and positioning images, dynamic image paths, and getting image dimensions.

## Light leaks

See [rules/light-leaks.md](rules/light-leaks.md) for light leak overlay effects using `@remotion/light-leaks`.

## Lottie animations

See [rules/lottie.md](rules/lottie.md) for embedding Lottie animations in Remotion.

## HTML in canvas

See [rules/html-in-canvas.md](rules/html-in-canvas.md) if you need to render HTML into a `<canvas>` to apply 2D or WebGL effects via `<HtmlInCanvas>`.

## Measuring DOM nodes

See [rules/measuring-dom-nodes.md](rules/measuring-dom-nodes.md) for measuring DOM element dimensions in Remotion.

## Measuring text

See [rules/measuring-text.md](rules/measuring-text.md) for measuring text dimensions, fitting text to containers, and checking overflow.

## Advanced sequencing

See [rules/sequencing.md](rules/sequencing.md) for more sequencing patterns - delay, trim, limit duration of items.

## TailwindCSS

See [rules/tailwind.md](rules/tailwind.md) for using TailwindCSS in Remotion.

## Text animations

See [rules/text-animations.md](rules/text-animations.md) for typography and text animation patterns.

## Advanced timing

See [rules/timing.md](rules/timing.md) for advanced timing with `interpolate` and Bézier easing, and springs.

## Transitions

See [rules/transitions.md](rules/transitions.md) for scene transition patterns.

## Transparent videos

See [rules/transparent-videos.md](rules/transparent-videos.md) for rendering out a video with transparency.

## Trimming

See [rules/trimming.md](rules/trimming.md) for trimming patterns - cutting the beginning or end of animations.

## Advanced Videos

See [rules/videos.md](rules/videos.md) for advanced knowledge about embedding videos - trimming, volume, speed, looping, pitch.

## Parameterized videos

See [rules/parameters.md](rules/parameters.md) for making a composition parametrizable by adding a Zod schema.

## Maps

For simple maps with little flyovers, consider using static map images.
For complex maps with animated routes or flyovers, load the maps rule: [rules/maplibre.md](rules/maplibre.md)

## Voiceover

See [rules/voiceover.md](rules/voiceover.md) for adding AI-generated voiceover to Remotion compositions using ElevenLabs TTS.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/3d.md -->
```markdown
---
name: 3d
description: 3D content in Remotion using Three.js and React Three Fiber.
metadata:
  tags: 3d, three, threejs
---

# Using Three.js and React Three Fiber in Remotion

Follow React Three Fiber and Three.js best practices.  
Only the following Remotion-specific rules need to be followed:

## Prerequisites

First, the `@remotion/three` package needs to be installed.  
If it is not, use the following command:

```bash
npx remotion add @remotion/three # If project uses npm
bunx remotion add @remotion/three # If project uses bun
yarn remotion add @remotion/three # If project uses yarn
pnpm exec remotion add @remotion/three # If project uses pnpm
```

## Using ThreeCanvas

You MUST wrap 3D content in `<ThreeCanvas>` and include proper lighting.  
`<ThreeCanvas>` MUST have a `width` and `height` prop.

```tsx
import { ThreeCanvas } from "@remotion/three";
import { useVideoConfig } from "remotion";

const { width, height } = useVideoConfig();

<ThreeCanvas width={width} height={height}>
  <ambientLight intensity={0.4} />
  <directionalLight position={[5, 5, 5]} intensity={0.8} />
  <mesh>
    <sphereGeometry args={[1, 32, 32]} />
    <meshStandardMaterial color="red" />
  </mesh>
</ThreeCanvas>;
```

## No animations not driven by `useCurrentFrame()`

Shaders, models etc MUST NOT animate by themselves.  
No animations are allowed unless they are driven by `useCurrentFrame()`.  
Otherwise, it will cause flickering during rendering.

Using `useFrame()` from `@react-three/fiber` is forbidden.

## Animate using `useCurrentFrame()`

Use `useCurrentFrame()` to perform animations.

```tsx
const frame = useCurrentFrame();
const rotationY = frame * 0.02;

<mesh rotation={[0, rotationY, 0]}>
  <boxGeometry args={[2, 2, 2]} />
  <meshStandardMaterial color="#4a9eff" />
</mesh>;
```

## Using `<Sequence>` inside `<ThreeCanvas>`

The `layout` prop of any `<Sequence>` inside a `<ThreeCanvas>` must be set to `none`.

```tsx
import { Sequence } from "remotion";
import { ThreeCanvas } from "@remotion/three";

const { width, height } = useVideoConfig();

<ThreeCanvas width={width} height={height}>
  <Sequence layout="none">
    <mesh>
      <boxGeometry args={[2, 2, 2]} />
      <meshStandardMaterial color="#4a9eff" />
    </mesh>
  </Sequence>
</ThreeCanvas>;
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/assets/charts-bar-chart.tsx -->
```tsx
import {loadFont} from '@remotion/google-fonts/Inter';
import {AbsoluteFill, spring, useCurrentFrame, useVideoConfig} from 'remotion';

const {fontFamily} = loadFont();

const COLOR_BAR = '#D4AF37';
const COLOR_TEXT = '#ffffff';
const COLOR_MUTED = '#888888';
const COLOR_BG = '#0a0a0a';
const COLOR_AXIS = '#333333';

// Ideal composition size: 1280x720

const Title: React.FC<{children: React.ReactNode}> = ({children}) => (
	<div style={{textAlign: 'center', marginBottom: 40}}>
		<div style={{color: COLOR_TEXT, fontSize: 48, fontWeight: 600}}>
			{children}
		</div>
	</div>
);

const YAxis: React.FC<{steps: number[]; height: number}> = ({
	steps,
	height,
}) => (
	<div
		style={{
			display: 'flex',
			flexDirection: 'column',
			justifyContent: 'space-between',
			height,
			paddingRight: 16,
		}}
	>
		{steps
			.slice()
			.reverse()
			.map((step) => (
				<div
					key={step}
					style={{
						color: COLOR_MUTED,
						fontSize: 20,
						textAlign: 'right',
					}}
				>
					{step.toLocaleString()}
				</div>
			))}
	</div>
);

const Bar: React.FC<{
	height: number;
	progress: number;
}> = ({height, progress}) => (
	<div
		style={{
			flex: 1,
			display: 'flex',
			flexDirection: 'column',
			justifyContent: 'flex-end',
		}}
	>
		<div
			style={{
				width: '100%',
				height,
				backgroundColor: COLOR_BAR,
				borderRadius: '8px 8px 0 0',
				opacity: progress,
			}}
		/>
	</div>
);

const XAxis: React.FC<{
	children: React.ReactNode;
	labels: string[];
	height: number;
}> = ({children, labels, height}) => (
	<div style={{flex: 1, display: 'flex', flexDirection: 'column'}}>
		<div
			style={{
				display: 'flex',
				alignItems: 'flex-end',
				gap: 16,
				height,
				borderLeft: `2px solid ${COLOR_AXIS}`,
				borderBottom: `2px solid ${COLOR_AXIS}`,
				paddingLeft: 16,
			}}
		>
			{children}
		</div>
		<div
			style={{
				display: 'flex',
				gap: 16,
				paddingLeft: 16,
				marginTop: 12,
			}}
		>
			{labels.map((label) => (
				<div
					key={label}
					style={{
						flex: 1,
						textAlign: 'center',
						color: COLOR_MUTED,
						fontSize: 20,
					}}
				>
					{label}
				</div>
			))}
		</div>
	</div>
);

export const MyAnimation = () => {
	const frame = useCurrentFrame();
	const {fps, height} = useVideoConfig();

	const data = [
		{month: 'Jan', price: 2039},
		{month: 'Mar', price: 2160},
		{month: 'May', price: 2327},
		{month: 'Jul', price: 2426},
		{month: 'Sep', price: 2634},
		{month: 'Nov', price: 2672},
	];

	const minPrice = 2000;
	const maxPrice = 2800;
	const priceRange = maxPrice - minPrice;
	const chartHeight = height - 280;
	const yAxisSteps = [2000, 2400, 2800];

	return (
		<AbsoluteFill
			style={{
				backgroundColor: COLOR_BG,
				padding: 60,
				display: 'flex',
				flexDirection: 'column',
				fontFamily,
			}}
		>
			<Title>Gold Price 2024</Title>

			<div style={{display: 'flex', flex: 1}}>
				<YAxis steps={yAxisSteps} height={chartHeight} />
				<XAxis height={chartHeight} labels={data.map((d) => d.month)}>
					{data.map((item, i) => {
						const progress = spring({
							frame: frame - i * 5 - 10,
							fps,
							config: {damping: 18, stiffness: 80},
						});

						const barHeight =
							((item.price - minPrice) / priceRange) * chartHeight * progress;

						return (
							<Bar key={item.month} height={barHeight} progress={progress} />
						);
					})}
				</XAxis>
			</div>
		</AbsoluteFill>
	);
};
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/assets/text-animations-typewriter.tsx -->
```tsx
import {
	AbsoluteFill,
	interpolate,
	useCurrentFrame,
	useVideoConfig,
} from 'remotion';

const COLOR_BG = '#ffffff';
const COLOR_TEXT = '#000000';
const FULL_TEXT = 'From prompt to motion graphics. This is Remotion.';
const PAUSE_AFTER = 'From prompt to motion graphics.';
const FONT_SIZE = 72;
const FONT_WEIGHT = 700;
const CHAR_FRAMES = 2;
const CURSOR_BLINK_FRAMES = 16;
const PAUSE_SECONDS = 1;

// Ideal composition size: 1280x720

const getTypedText = ({
	frame,
	fullText,
	pauseAfter,
	charFrames,
	pauseFrames,
}: {
	frame: number;
	fullText: string;
	pauseAfter: string;
	charFrames: number;
	pauseFrames: number;
}): string => {
	const pauseIndex = fullText.indexOf(pauseAfter);
	const preLen =
		pauseIndex >= 0 ? pauseIndex + pauseAfter.length : fullText.length;

	let typedChars = 0;
	if (frame < preLen * charFrames) {
		typedChars = Math.floor(frame / charFrames);
	} else if (frame < preLen * charFrames + pauseFrames) {
		typedChars = preLen;
	} else {
		const postPhase = frame - preLen * charFrames - pauseFrames;
		typedChars = Math.min(
			fullText.length,
			preLen + Math.floor(postPhase / charFrames),
		);
	}
	return fullText.slice(0, typedChars);
};

const Cursor: React.FC<{
	frame: number;
	blinkFrames: number;
	symbol?: string;
}> = ({frame, blinkFrames, symbol = '\u258C'}) => {
	const opacity = interpolate(
		frame % blinkFrames,
		[0, blinkFrames / 2, blinkFrames],
		[1, 0, 1],
		{extrapolateLeft: 'clamp', extrapolateRight: 'clamp'},
	);

	return <span style={{opacity}}>{symbol}</span>;
};

export const MyAnimation = () => {
	const frame = useCurrentFrame();
	const {fps} = useVideoConfig();

	const pauseFrames = Math.round(fps * PAUSE_SECONDS);

	const typedText = getTypedText({
		frame,
		fullText: FULL_TEXT,
		pauseAfter: PAUSE_AFTER,
		charFrames: CHAR_FRAMES,
		pauseFrames,
	});

	return (
		<AbsoluteFill
			style={{
				backgroundColor: COLOR_BG,
			}}
		>
			<div
				style={{
					color: COLOR_TEXT,
					fontSize: FONT_SIZE,
					fontWeight: FONT_WEIGHT,
					fontFamily: 'sans-serif',
				}}
			>
				<span>{typedText}</span>
				<Cursor frame={frame} blinkFrames={CURSOR_BLINK_FRAMES} />
			</div>
		</AbsoluteFill>
	);
};
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/assets/text-animations-word-highlight.tsx -->
```tsx
import {loadFont} from '@remotion/google-fonts/Inter';
import React from 'react';
import {AbsoluteFill, spring, useCurrentFrame, useVideoConfig} from 'remotion';

/*
 * Highlight a word in a sentence with a spring-animated wipe effect.
 */

// Ideal composition size: 1280x720

const COLOR_BG = '#ffffff';
const COLOR_TEXT = '#000000';
const COLOR_HIGHLIGHT = '#A7C7E7';
const FULL_TEXT = 'This is Remotion.';
const HIGHLIGHT_WORD = 'Remotion';
const FONT_SIZE = 72;
const FONT_WEIGHT = 700;
const HIGHLIGHT_START_FRAME = 30;
const HIGHLIGHT_WIPE_DURATION = 18;

const {fontFamily} = loadFont();

const Highlight: React.FC<{
	word: string;
	color: string;
	delay: number;
	durationInFrames: number;
}> = ({word, color, delay, durationInFrames}) => {
	const frame = useCurrentFrame();
	const {fps} = useVideoConfig();

	const highlightProgress = spring({
		fps,
		frame,
		config: {damping: 200},
		delay,
		durationInFrames,
	});
	const scaleX = Math.max(0, Math.min(1, highlightProgress));

	return (
		<span style={{position: 'relative', display: 'inline-block'}}>
			<span
				style={{
					position: 'absolute',
					left: 0,
					right: 0,
					top: '50%',
					height: '1.05em',
					transform: `translateY(-50%) scaleX(${scaleX})`,
					transformOrigin: 'left center',
					backgroundColor: color,
					borderRadius: '0.18em',
					zIndex: 0,
				}}
			/>
			<span style={{position: 'relative', zIndex: 1}}>{word}</span>
		</span>
	);
};

export const MyAnimation = () => {
	const highlightIndex = FULL_TEXT.indexOf(HIGHLIGHT_WORD);
	const hasHighlight = highlightIndex >= 0;
	const preText = hasHighlight ? FULL_TEXT.slice(0, highlightIndex) : FULL_TEXT;
	const postText = hasHighlight
		? FULL_TEXT.slice(highlightIndex + HIGHLIGHT_WORD.length)
		: '';

	return (
		<AbsoluteFill
			style={{
				backgroundColor: COLOR_BG,
				alignItems: 'center',
				justifyContent: 'center',
				fontFamily,
			}}
		>
			<div
				style={{
					color: COLOR_TEXT,
					fontSize: FONT_SIZE,
					fontWeight: FONT_WEIGHT,
				}}
			>
				{hasHighlight ? (
					<>
						<span>{preText}</span>
						<Highlight
							word={HIGHLIGHT_WORD}
							color={COLOR_HIGHLIGHT}
							delay={HIGHLIGHT_START_FRAME}
							durationInFrames={HIGHLIGHT_WIPE_DURATION}
						/>
						<span>{postText}</span>
					</>
				) : (
					<span>{FULL_TEXT}</span>
				)}
			</div>
		</AbsoluteFill>
	);
};
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/audio-visualization.md -->
```markdown
---
name: audio-visualization
description: Audio visualization patterns - spectrum bars, waveforms, bass-reactive effects
metadata:
  tags: audio, visualization, spectrum, waveform, bass, music, audiogram, frequency
---

# Audio Visualization in Remotion

## Prerequisites

```bash
npx remotion add @remotion/media-utils
```

## Loading Audio Data

Use `useWindowedAudioData()` (https://www.remotion.dev/docs/use-windowed-audio-data) to load audio data:

```tsx
import { useWindowedAudioData } from "@remotion/media-utils";
import { staticFile, useCurrentFrame, useVideoConfig } from "remotion";

const frame = useCurrentFrame();
const { fps } = useVideoConfig();

const { audioData, dataOffsetInSeconds } = useWindowedAudioData({
  src: staticFile("podcast.wav"),
  frame,
  fps,
  windowInSeconds: 30,
});
```

## Spectrum Bar Visualization

Use `visualizeAudio()` (https://www.remotion.dev/docs/visualize-audio) to get frequency data for bar charts:

```tsx
import { useWindowedAudioData, visualizeAudio } from "@remotion/media-utils";
import { staticFile, useCurrentFrame, useVideoConfig } from "remotion";

const frame = useCurrentFrame();
const { fps } = useVideoConfig();

const { audioData, dataOffsetInSeconds } = useWindowedAudioData({
  src: staticFile("music.mp3"),
  frame,
  fps,
  windowInSeconds: 30,
});

if (!audioData) {
  return null;
}

const frequencies = visualizeAudio({
  fps,
  frame,
  audioData,
  numberOfSamples: 256,
  optimizeFor: "speed",
  dataOffsetInSeconds,
});

return (
  <div style={{ display: "flex", alignItems: "flex-end", height: 200 }}>
    {frequencies.map((v, i) => (
      <div
        key={i}
        style={{
          flex: 1,
          height: `${v * 100}%`,
          backgroundColor: "#0b84f3",
          margin: "0 1px",
        }}
      />
    ))}
  </div>
);
```

- `numberOfSamples` must be power of 2 (32, 64, 128, 256, 512, 1024)
- Values range 0-1; left of array = bass, right = highs
- Use `optimizeFor: "speed"` for Lambda or high sample counts

**Important:** When passing `audioData` to child components, also pass the `frame` from the parent. Do not call `useCurrentFrame()` in each child - this causes discontinuous visualization when children are inside `<Sequence>` with offsets.

## Waveform Visualization

Use `visualizeAudioWaveform()` (https://www.remotion.dev/docs/media-utils/visualize-audio-waveform) with `createSmoothSvgPath()` (https://www.remotion.dev/docs/media-utils/create-smooth-svg-path) for oscilloscope-style displays:

```tsx
import {
  createSmoothSvgPath,
  useWindowedAudioData,
  visualizeAudioWaveform,
} from "@remotion/media-utils";
import { staticFile, useCurrentFrame, useVideoConfig } from "remotion";

const frame = useCurrentFrame();
const { width, fps } = useVideoConfig();
const HEIGHT = 200;

const { audioData, dataOffsetInSeconds } = useWindowedAudioData({
  src: staticFile("voice.wav"),
  frame,
  fps,
  windowInSeconds: 30,
});

if (!audioData) {
  return null;
}

const waveform = visualizeAudioWaveform({
  fps,
  frame,
  audioData,
  numberOfSamples: 256,
  windowInSeconds: 0.5,
  dataOffsetInSeconds,
});

const path = createSmoothSvgPath({
  points: waveform.map((y, i) => ({
    x: (i / (waveform.length - 1)) * width,
    y: HEIGHT / 2 + (y * HEIGHT) / 2,
  })),
});

return (
  <svg width={width} height={HEIGHT}>
    <path d={path} fill="none" stroke="#0b84f3" strokeWidth={2} />
  </svg>
);
```

## Bass-Reactive Effects

Extract low frequencies for beat-reactive animations:

```tsx
const frequencies = visualizeAudio({
  fps,
  frame,
  audioData,
  numberOfSamples: 128,
  optimizeFor: "speed",
  dataOffsetInSeconds,
});

const lowFrequencies = frequencies.slice(0, 32);
const bassIntensity =
  lowFrequencies.reduce((sum, v) => sum + v, 0) / lowFrequencies.length;

const scale = 1 + bassIntensity * 0.5;
const opacity = Math.min(0.6, bassIntensity * 0.8);
```

## Volume-Based Waveform

Use `getWaveformPortion()` (https://www.remotion.dev/docs/get-waveform-portion) when you need simplified volume data instead of frequency spectrum:

```tsx
import { getWaveformPortion } from "@remotion/media-utils";
import { useCurrentFrame, useVideoConfig } from "remotion";

const frame = useCurrentFrame();
const { fps } = useVideoConfig();
const currentTimeInSeconds = frame / fps;

const waveform = getWaveformPortion({
  audioData,
  startTimeInSeconds: currentTimeInSeconds,
  durationInSeconds: 5,
  numberOfSamples: 50,
});

// Returns array of { index, amplitude } objects (amplitude: 0-1)
waveform.map((bar) => (
  <div key={bar.index} style={{ height: bar.amplitude * 100 }} />
));
```

## Postprocessing

Low frequencies naturally dominate. Apply logarithmic scaling for visual balance:

```tsx
const minDb = -100;
const maxDb = -30;

const scaled = frequencies.map((value) => {
  const db = 20 * Math.log10(value);
  return (db - minDb) / (maxDb - minDb);
});
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/audio.md -->
```markdown
---
name: audio
description: Using audio and sound in Remotion - importing, trimming, volume, speed, pitch
metadata:
  tags: audio, media, trim, volume, speed, loop, pitch, mute, sound, sfx
---

# Using audio in Remotion

## Prerequisites

First, the @remotion/media package needs to be installed.
If it is not installed, use the following command:

```bash
npx remotion add @remotion/media
```

## Importing Audio

Use `<Audio>` from `@remotion/media` to add audio to your composition.

```tsx
import { Audio } from "@remotion/media";
import { staticFile } from "remotion";

export const MyComposition = () => {
  return <Audio src={staticFile("audio.mp3")} />;
};
```

Remote URLs are also supported:

```tsx
<Audio src="https://remotion.media/audio.mp3" />
```

By default, audio plays from the start, at full volume and full length.
Multiple audio tracks can be layered by adding multiple `<Audio>` components.

## Trimming

Use `trimBefore` and `trimAfter` to remove portions of the audio. Values are in frames.

```tsx
const { fps } = useVideoConfig();

return (
  <Audio
    src={staticFile("audio.mp3")}
    trimBefore={2 * fps} // Skip the first 2 seconds
    trimAfter={10 * fps} // End at the 10 second mark
  />
);
```

The audio still starts playing at the beginning of the composition - only the specified portion is played.

## Delaying

Wrap the audio in a `<Sequence>` to delay when it starts:

```tsx
import { Sequence, staticFile } from "remotion";
import { Audio } from "@remotion/media";

const { fps } = useVideoConfig();

return (
  <Sequence from={1 * fps}>
    <Audio src={staticFile("audio.mp3")} />
  </Sequence>
);
```

The audio will start playing after 1 second.

## Volume

Set a static volume (0 to 1):

```tsx
<Audio src={staticFile("audio.mp3")} volume={0.5} />
```

Or use a callback for dynamic volume based on the current frame:

```tsx
import { interpolate } from "remotion";

const { fps } = useVideoConfig();

return (
  <Audio
    src={staticFile("audio.mp3")}
    volume={(f) =>
      interpolate(f, [0, 1 * fps], [0, 1], { extrapolateRight: "clamp" })
    }
  />
);
```

The value of `f` starts at 0 when the audio begins to play, not the composition frame.

## Muting

Use `muted` to silence the audio. It can be set dynamically:

```tsx
const frame = useCurrentFrame();
const { fps } = useVideoConfig();

return (
  <Audio
    src={staticFile("audio.mp3")}
    muted={frame >= 2 * fps && frame <= 4 * fps} // Mute between 2s and 4s
  />
);
```

## Speed

Use `playbackRate` to change the playback speed:

```tsx
<Audio src={staticFile("audio.mp3")} playbackRate={2} /> {/* 2x speed */}
<Audio src={staticFile("audio.mp3")} playbackRate={0.5} /> {/* Half speed */}
```

Reverse playback is not supported.

## Looping

Use `loop` to loop the audio indefinitely:

```tsx
<Audio src={staticFile("audio.mp3")} loop />
```

Use `loopVolumeCurveBehavior` to control how the frame count behaves when looping:

- `"repeat"`: Frame count resets to 0 each loop (default)
- `"extend"`: Frame count continues incrementing

```tsx
<Audio
  src={staticFile("audio.mp3")}
  loop
  loopVolumeCurveBehavior="extend"
  volume={(f) => interpolate(f, [0, 300], [1, 0])} // Fade out over multiple loops
/>
```

## Pitch

Use `toneFrequency` to adjust the pitch without affecting speed. Values range from 0.01 to 2:

```tsx
<Audio
  src={staticFile("audio.mp3")}
  toneFrequency={1.5} // Higher pitch
/>
<Audio
  src={staticFile("audio.mp3")}
  toneFrequency={0.8} // Lower pitch
/>
```

Pitch shifting only works during server-side rendering, not in the Remotion Studio preview or in the `<Player />`.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/calculate-metadata.md -->
```markdown
---
name: calculate-metadata
description: Dynamically set composition duration, dimensions, and props
metadata:
  tags: calculateMetadata, duration, dimensions, props, dynamic
---

# Using calculateMetadata

Use `calculateMetadata` on a `<Composition>` to dynamically set duration, dimensions, and transform props before rendering.

```tsx
<Composition
  id="MyComp"
  component={MyComponent}
  durationInFrames={300}
  fps={30}
  width={1920}
  height={1080}
  defaultProps={{ videoSrc: "https://remotion.media/video.mp4" }}
  calculateMetadata={calculateMetadata}
/>
```

## Setting duration based on a video

Use the [`getVideoDuration`](./get-video-duration.md) and [`getVideoDimensions`](./get-video-dimensions.md) skills to get the video duration and dimensions:

```tsx
import { CalculateMetadataFunction } from "remotion";
import { getVideoDuration } from "./get-video-duration";

const calculateMetadata: CalculateMetadataFunction<Props> = async ({
  props,
}) => {
  const durationInSeconds = await getVideoDuration(props.videoSrc);

  return {
    durationInFrames: Math.ceil(durationInSeconds * 30),
  };
};
```

## Matching dimensions of a video

Use the [`getVideoDimensions`](./get-video-dimensions.md) skill to get the video dimensions:

```tsx
import { CalculateMetadataFunction } from "remotion";
import { getVideoDuration } from "./get-video-duration";
import { getVideoDimensions } from "./get-video-dimensions";

const calculateMetadata: CalculateMetadataFunction<Props> = async ({
  props,
}) => {
  const dimensions = await getVideoDimensions(props.videoSrc);

  return {
    width: dimensions.width,
    height: dimensions.height,
  };
};
```

## Setting duration based on multiple videos

```tsx
const calculateMetadata: CalculateMetadataFunction<Props> = async ({
  props,
}) => {
  const metadataPromises = props.videos.map((video) =>
    getVideoDuration(video.src),
  );
  const allMetadata = await Promise.all(metadataPromises);

  const totalDuration = allMetadata.reduce(
    (sum, durationInSeconds) => sum + durationInSeconds,
    0,
  );

  return {
    durationInFrames: Math.ceil(totalDuration * 30),
  };
};
```

## Setting a default outName

Set the default output filename based on props:

```tsx
const calculateMetadata: CalculateMetadataFunction<Props> = async ({
  props,
}) => {
  return {
    defaultOutName: `video-${props.id}`, // .mp4 is added automatically
  };
};
```

## Transforming props

Fetch data or transform props before rendering:

```tsx
const calculateMetadata: CalculateMetadataFunction<Props> = async ({
  props,
  abortSignal,
}) => {
  const response = await fetch(props.dataUrl, { signal: abortSignal });
  const data = await response.json();

  return {
    props: {
      ...props,
      fetchedData: data,
    },
  };
};
```

The `abortSignal` cancels stale requests when props change in the Studio.

## Return value

All fields are optional. Returned values override the `<Composition>` props:

- `durationInFrames`: Number of frames
- `width`: Composition width in pixels
- `height`: Composition height in pixels
- `fps`: Frames per second
- `props`: Transformed props passed to the component
- `defaultOutName`: Default output filename
- `defaultCodec`: Default codec for rendering
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/compositions.md -->
```markdown
---
name: compositions
description: Defining compositions, stills, folders, default props and dynamic metadata
metadata:
  tags: composition, still, folder, props, metadata
---

A `<Composition>` defines the component, width, height, fps and duration of a renderable video.

## Default Props

Pass `defaultProps` to provide initial values for your component.  
Values must be JSON-serializable (`Date`, `Map`, `Set`, and `staticFile()` are supported).

```tsx
import { Composition } from "remotion";
import { MyComposition, MyCompositionProps } from "./MyComposition";

export const RemotionRoot = () => {
  return (
    <Composition
      id="MyComposition"
      component={MyComposition}
      durationInFrames={100}
      fps={30}
      width={1080}
      height={1080}
      defaultProps={
        {
          title: "Hello World",
          color: "#ff0000",
        } satisfies MyCompositionProps
      }
    />
  );
};
```

Use `type` declarations for props rather than `interface` to ensure `defaultProps` type safety.

## Folders

Use `<Folder>` to organize compositions in the sidebar.  
Folder names can only contain letters, numbers, and hyphens.

```tsx
import { Composition, Folder } from "remotion";

export const RemotionRoot = () => {
  return (
    <>
      <Folder name="Marketing">
        <Composition id="Promo" /* ... */ />
        <Composition id="Ad" /* ... */ />
      </Folder>
      <Folder name="Social">
        <Folder name="Instagram">
          <Composition id="Story" /* ... */ />
          <Composition id="Reel" /* ... */ />
        </Folder>
      </Folder>
    </>
  );
};
```

## Stills

Use `<Still>` for single-frame images. It does not require `durationInFrames` or `fps`.

```tsx
import { Still } from "remotion";
import { Thumbnail } from "./Thumbnail";

export const RemotionRoot = () => {
  return (
    <Still id="Thumbnail" component={Thumbnail} width={1280} height={720} />
  );
};
```

## Calculate Metadata

Use `calculateMetadata` to make dimensions, duration, or props dynamic based on data.

```tsx
import { Composition, CalculateMetadataFunction } from "remotion";
import { MyComposition, MyCompositionProps } from "./MyComposition";

const calculateMetadata: CalculateMetadataFunction<
  MyCompositionProps
> = async ({ props, abortSignal }) => {
  const data = await fetch(`https://api.example.com/video/${props.videoId}`, {
    signal: abortSignal,
  }).then((res) => res.json());

  return {
    durationInFrames: Math.ceil(data.duration * 30),
    props: {
      ...props,
      videoUrl: data.url,
    },
  };
};

export const RemotionRoot = () => {
  return (
    <Composition
      id="MyComposition"
      component={MyComposition}
      fps={30}
      width={1080}
      height={1080}
      defaultProps={{ videoId: "abc123" }}
      calculateMetadata={calculateMetadata}
    />
  );
};
```

The function can return `props`, `durationInFrames`, `width`, `height`, `fps`, and codec-related defaults. It runs once before rendering begins.

## Nesting compositions within another

To add a composition within another composition, you can use the `<Sequence>` component with a `width` and `height` prop to specify the size of the composition.

```tsx
<AbsoluteFill>
  <Sequence width={COMPOSITION_WIDTH} height={COMPOSITION_HEIGHT}>
    <CompositionComponent />
  </Sequence>
</AbsoluteFill>
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/display-captions.md -->
```markdown
---
name: display-captions
description: Displaying captions in Remotion with TikTok-style pages and word highlighting
metadata:
  tags: captions, subtitles, display, tiktok, highlight
---

# Displaying captions in Remotion

This guide explains how to display captions in Remotion, assuming you already have captions in the [`Caption`](https://www.remotion.dev/docs/captions/caption) format.

## Prerequisites

Read [Transcribing audio](transcribe-captions.md) for how to generate captions.

First, the [`@remotion/captions`](https://www.remotion.dev/docs/captions) package needs to be installed.
If it is not installed, use the following command:

```bash
npx remotion add @remotion/captions
```

## Fetching captions

First, fetch your captions JSON file. Use [`useDelayRender()`](https://www.remotion.dev/docs/use-delay-render) to hold the render until the captions are loaded:

```tsx
import { useState, useEffect, useCallback } from "react";
import { AbsoluteFill, staticFile, useDelayRender } from "remotion";
import type { Caption } from "@remotion/captions";

export const MyComponent: React.FC = () => {
  const [captions, setCaptions] = useState<Caption[] | null>(null);
  const { delayRender, continueRender, cancelRender } = useDelayRender();
  const [handle] = useState(() => delayRender());

  const fetchCaptions = useCallback(async () => {
    try {
      // Assuming captions.json is in the public/ folder.
      const response = await fetch(staticFile("captions123.json"));
      const data = await response.json();
      setCaptions(data);
      continueRender(handle);
    } catch (e) {
      cancelRender(e);
    }
  }, [continueRender, cancelRender, handle]);

  useEffect(() => {
    fetchCaptions();
  }, [fetchCaptions]);

  if (!captions) {
    return null;
  }

  return <AbsoluteFill>{/* Render captions here */}</AbsoluteFill>;
};
```

## Creating pages

Use `createTikTokStyleCaptions()` to group captions into pages. The `combineTokensWithinMilliseconds` option controls how many words appear at once:

```tsx
import { useMemo } from "react";
import { createTikTokStyleCaptions } from "@remotion/captions";
import type { Caption } from "@remotion/captions";

// How often captions should switch (in milliseconds)
// Higher values = more words per page
// Lower values = fewer words (more word-by-word)
const SWITCH_CAPTIONS_EVERY_MS = 1200;

const { pages } = useMemo(() => {
  return createTikTokStyleCaptions({
    captions,
    combineTokensWithinMilliseconds: SWITCH_CAPTIONS_EVERY_MS,
  });
}, [captions]);
```

## Rendering with Sequences

Map over the pages and render each one in a `<Sequence>`. Calculate the start frame and duration from the page timing:

```tsx
import { Sequence, useVideoConfig, AbsoluteFill } from "remotion";
import type { TikTokPage } from "@remotion/captions";

const CaptionedContent: React.FC = () => {
  const { fps } = useVideoConfig();

  return (
    <AbsoluteFill>
      {pages.map((page, index) => {
        const nextPage = pages[index + 1] ?? null;
        const startFrame = (page.startMs / 1000) * fps;
        const endFrame = Math.min(
          nextPage ? (nextPage.startMs / 1000) * fps : Infinity,
          startFrame + (SWITCH_CAPTIONS_EVERY_MS / 1000) * fps,
        );
        const durationInFrames = endFrame - startFrame;

        if (durationInFrames <= 0) {
          return null;
        }

        return (
          <Sequence
            key={index}
            from={startFrame}
            durationInFrames={durationInFrames}
          >
            <CaptionPage page={page} />
          </Sequence>
        );
      })}
    </AbsoluteFill>
  );
};
```

## White-space preservation

The captions are whitespace sensitive. You should include spaces in the `text` field before each word. Use `whiteSpace: "pre"` to preserve the whitespace in the captions.

## Separate component for captions

Put captioning logic in a separate component.  
Make a new file for it.

## Word highlighting

A caption page contains `tokens` which you can use to highlight the currently spoken word:

```tsx
import { AbsoluteFill, useCurrentFrame, useVideoConfig } from "remotion";
import type { TikTokPage } from "@remotion/captions";

const HIGHLIGHT_COLOR = "#39E508";

const CaptionPage: React.FC<{ page: TikTokPage }> = ({ page }) => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();

  // Current time relative to the start of the sequence
  const currentTimeMs = (frame / fps) * 1000;
  // Convert to absolute time by adding the page start
  const absoluteTimeMs = page.startMs + currentTimeMs;

  return (
    <AbsoluteFill style={{ justifyContent: "center", alignItems: "center" }}>
      <div style={{ fontSize: 80, fontWeight: "bold", whiteSpace: "pre" }}>
        {page.tokens.map((token) => {
          const isActive =
            token.fromMs <= absoluteTimeMs && token.toMs > absoluteTimeMs;

          return (
            <span
              key={token.fromMs}
              style={{ color: isActive ? HIGHLIGHT_COLOR : "white" }}
            >
              {token.text}
            </span>
          );
        })}
      </div>
    </AbsoluteFill>
  );
};
```

## Display captions alongside video content

By default, put the captions alongside the video content, so the captions are in sync.  
For each video, make a new captions JSON file.

```tsx
<AbsoluteFill>
  <Video src={staticFile("video.mp4")} />
  <CaptionPage page={page} />
</AbsoluteFill>
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/ffmpeg.md -->
```markdown
---
name: ffmpeg
description: Using FFmpeg and FFprobe in Remotion
metadata:
  tags: ffmpeg, ffprobe, video, trimming
---

## FFmpeg in Remotion

`ffmpeg` and `ffprobe` do not need to be installed. They are available via the `npx remotion ffmpeg` and `npx remotion ffprobe`:

```bash
npx remotion ffmpeg -i input.mp4 output.mp3
npx remotion ffprobe input.mp4
```

### Trimming videos

You have 2 options for trimming videos:

1. **Preferred**: Use the `trimBefore` and `trimAfter` props of the `<Video>` component. This is non-destructive, requires no re-encoding, and you can change the trim at any time.

```tsx
import {Video} from '@remotion/media';

<Video src={staticFile('video.mp4')} trimBefore={5 * fps} trimAfter={10 * fps} />;
```

2. Use the FFmpeg command line. You MUST re-encode the video to avoid frozen frames at the start of the video. Only use this if you need a standalone trimmed file (e.g. for upload or external use).

```bash
# Re-encodes from the exact frame
npx remotion ffmpeg -ss 00:00:05 -i public/input.mp4 -to 00:00:10 -c:v libx264 -c:a aac public/output.mp4
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/get-audio-duration.md -->
```markdown
---
name: get-audio-duration
description: Getting the duration of an audio file in seconds with Mediabunny
metadata:
  tags: duration, audio, length, time, seconds, mp3, wav
---

# Getting audio duration with Mediabunny

Mediabunny can extract the duration of an audio file. It works in browser, Node.js, and Bun environments.

## Getting audio duration

```tsx title="get-audio-duration.ts"
import { Input, ALL_FORMATS, UrlSource } from "mediabunny";

export const getAudioDuration = async (src: string) => {
  const input = new Input({
    formats: ALL_FORMATS,
    source: new UrlSource(src, {
      getRetryDelay: () => null,
    }),
  });

  const durationInSeconds = await input.computeDuration();
  return durationInSeconds;
};
```

## Usage

```tsx
const duration = await getAudioDuration("https://remotion.media/audio.mp3");
console.log(duration); // e.g. 180.5 (seconds)
```

## Using with staticFile in Remotion

Make sure to wrap the file path in `staticFile()`:

```tsx
import { staticFile } from "remotion";

const duration = await getAudioDuration(staticFile("audio.mp3"));
```

## In Node.js and Bun

Use `FileSource` instead of `UrlSource`:

```tsx
import { Input, ALL_FORMATS, FileSource } from "mediabunny";

const input = new Input({
  formats: ALL_FORMATS,
  source: new FileSource(file), // File object from input or drag-drop
});
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/get-video-dimensions.md -->
```markdown
---
name: get-video-dimensions
description: Getting the width and height of a video file with Mediabunny
metadata:
  tags: dimensions, width, height, resolution, size, video
---

# Getting video dimensions with Mediabunny

Mediabunny can extract the width and height of a video file. It works in browser, Node.js, and Bun environments.

## Getting video dimensions

```tsx
import { Input, ALL_FORMATS, UrlSource } from "mediabunny";

export const getVideoDimensions = async (src: string) => {
  const input = new Input({
    formats: ALL_FORMATS,
    source: new UrlSource(src, {
      getRetryDelay: () => null,
    }),
  });

  const videoTrack = await input.getPrimaryVideoTrack();
  if (!videoTrack) {
    throw new Error("No video track found");
  }

  return {
    width: videoTrack.displayWidth,
    height: videoTrack.displayHeight,
  };
};
```

## Usage

```tsx
const dimensions = await getVideoDimensions("https://remotion.media/video.mp4");
console.log(dimensions.width); // e.g. 1920
console.log(dimensions.height); // e.g. 1080
```

## Using with local files

For local files, use `FileSource` instead of `UrlSource`:

```tsx
import { Input, ALL_FORMATS, FileSource } from "mediabunny";

const input = new Input({
  formats: ALL_FORMATS,
  source: new FileSource(file), // File object from input or drag-drop
});

const videoTrack = await input.getPrimaryVideoTrack();
const width = videoTrack.displayWidth;
const height = videoTrack.displayHeight;
```

## Using with staticFile in Remotion

```tsx
import { staticFile } from "remotion";

const dimensions = await getVideoDimensions(staticFile("video.mp4"));
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/get-video-duration.md -->
```markdown
---
name: get-video-duration
description: Getting the duration of a video file in seconds with Mediabunny
metadata:
  tags: duration, video, length, time, seconds
---

# Getting video duration with Mediabunny

Mediabunny can extract the duration of a video file. It works in browser, Node.js, and Bun environments.

## Getting video duration

```tsx
import { Input, ALL_FORMATS, UrlSource } from "mediabunny";

export const getVideoDuration = async (src: string) => {
  const input = new Input({
    formats: ALL_FORMATS,
    source: new UrlSource(src, {
      getRetryDelay: () => null,
    }),
  });

  const durationInSeconds = await input.computeDuration();
  return durationInSeconds;
};
```

## Usage

```tsx
const duration = await getVideoDuration("https://remotion.media/video.mp4");
console.log(duration); // e.g. 10.5 (seconds)
```

## Video files from the public/ directory

Make sure to wrap the file path in `staticFile()`:

```tsx
import { staticFile } from "remotion";

const duration = await getVideoDuration(staticFile("video.mp4"));
```

## In Node.js and Bun

Use `FileSource` instead of `UrlSource`:

```tsx
import { Input, ALL_FORMATS, FileSource } from "mediabunny";

const input = new Input({
  formats: ALL_FORMATS,
  source: new FileSource(file), // File object from input or drag-drop
});

const durationInSeconds = await input.computeDuration();
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/gifs.md -->
```markdown
---
name: gif
description: Displaying GIFs, APNG, AVIF and WebP in Remotion
metadata:
  tags: gif, animation, images, animated, apng, avif, webp
---

# Using Animated images in Remotion

## Basic usage

Use `<AnimatedImage>` to display a GIF, APNG, AVIF or WebP image synchronized with Remotion's timeline:

```tsx
import { AnimatedImage, staticFile } from "remotion";

export const MyComposition = () => {
  return (
    <AnimatedImage src={staticFile("animation.gif")} width={500} height={500} />
  );
};
```

Remote URLs are also supported (must have CORS enabled):

```tsx
<AnimatedImage
  src="https://example.com/animation.gif"
  width={500}
  height={500}
/>
```

## Sizing and fit

Control how the image fills its container with the `fit` prop:

```tsx
// Stretch to fill (default)
<AnimatedImage src={staticFile("animation.gif")} width={500} height={300} fit="fill" />

// Maintain aspect ratio, fit inside container
<AnimatedImage src={staticFile("animation.gif")} width={500} height={300} fit="contain" />

// Fill container, crop if needed
<AnimatedImage src={staticFile("animation.gif")} width={500} height={300} fit="cover" />
```

## Playback speed

Use `playbackRate` to control the animation speed:

```tsx
<AnimatedImage src={staticFile("animation.gif")} width={500} height={500} playbackRate={2} /> {/* 2x speed */}
<AnimatedImage src={staticFile("animation.gif")} width={500} height={500} playbackRate={0.5} /> {/* Half speed */}
```

## Looping behavior

Control what happens when the animation finishes:

```tsx
// Loop indefinitely (default)
<AnimatedImage src={staticFile("animation.gif")} width={500} height={500} loopBehavior="loop" />

// Play once, show final frame
<AnimatedImage src={staticFile("animation.gif")} width={500} height={500} loopBehavior="pause-after-finish" />

// Play once, then clear canvas
<AnimatedImage src={staticFile("animation.gif")} width={500} height={500} loopBehavior="clear-after-finish" />
```

## Styling

Use the `style` prop for additional CSS (use `width` and `height` props for sizing):

```tsx
<AnimatedImage
  src={staticFile("animation.gif")}
  width={500}
  height={500}
  style={{
    borderRadius: 20,
    position: "absolute",
    top: 100,
    left: 50,
  }}
/>
```

## Getting GIF duration

Use `getGifDurationInSeconds()` from `@remotion/gif` to get the duration of a GIF.

```bash
npx remotion add @remotion/gif
```

```tsx
import { getGifDurationInSeconds } from "@remotion/gif";
import { staticFile } from "remotion";

const duration = await getGifDurationInSeconds(staticFile("animation.gif"));
console.log(duration); // e.g. 2.5
```

This is useful for setting the composition duration to match the GIF:

```tsx
import { getGifDurationInSeconds } from "@remotion/gif";
import { staticFile, CalculateMetadataFunction } from "remotion";

const calculateMetadata: CalculateMetadataFunction = async () => {
  const duration = await getGifDurationInSeconds(staticFile("animation.gif"));
  return {
    durationInFrames: Math.ceil(duration * 30),
  };
};
```

## Alternative

If `<AnimatedImage>` does not work (only supported in Chrome and Firefox), you can use `<Gif>` from `@remotion/gif` instead.

```bash
npx remotion add @remotion/gif # If project uses npm
bunx remotion add @remotion/gif # If project uses bun
yarn remotion add @remotion/gif # If project uses yarn
pnpm exec remotion add @remotion/gif # If project uses pnpm
```

```tsx
import { Gif } from "@remotion/gif";
import { staticFile } from "remotion";

export const MyComposition = () => {
  return <Gif src={staticFile("animation.gif")} width={500} height={500} />;
};
```

The `<Gif>` component has the same props as `<AnimatedImage>` but only supports GIF files.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/google-fonts.md -->
```markdown
---
name: fonts
description: Loading Google Fonts and local fonts in Remotion
metadata:
  tags: fonts, google-fonts, typography, text
---

# Using fonts in Remotion

## Google Fonts with @remotion/google-fonts

The recommended way to use Google Fonts. It's type-safe and automatically blocks rendering until the font is ready.

### Prerequisites

First, the @remotion/google-fonts package needs to be installed.
If it is not installed, use the following command:

```bash
npx remotion add @remotion/google-fonts # If project uses npm
bunx remotion add @remotion/google-fonts # If project uses bun
yarn remotion add @remotion/google-fonts # If project uses yarn
pnpm exec remotion add @remotion/google-fonts # If project uses pnpm
```

```tsx
import { loadFont } from "@remotion/google-fonts/Lobster";

const { fontFamily } = loadFont();

export const MyComposition = () => {
  return <div style={{ fontFamily }}>Hello World</div>;
};
```

Preferrably, specify only needed weights and subsets to reduce file size:

```tsx
import { loadFont } from "@remotion/google-fonts/Roboto";

const { fontFamily } = loadFont("normal", {
  weights: ["400", "700"],
  subsets: ["latin"],
});
```

## Using in components

Call `loadFont()` at the top level of your component or in a separate file that's imported early:

```tsx
import { loadFont } from "@remotion/google-fonts/Montserrat";

const { fontFamily } = loadFont("normal", {
  weights: ["400", "700"],
  subsets: ["latin"],
});

export const Title: React.FC<{ text: string }> = ({ text }) => {
  return (
    <h1
      style={{
        fontFamily,
        fontSize: 80,
        fontWeight: "bold",
      }}
    >
      {text}
    </h1>
  );
};
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/html-in-canvas.md -->
```markdown
# Using `<HtmlInCanvas>` in Remotion

Renders children into a `<canvas>` so you can post-process them with the Canvas 2D API or WebGL.

Only works in Chrome 149+ with the `chrome://flags/#canvas-draw-element` flag enabled.  
Give the user a notice.

## Nesting

Do not nest `<HtmlInCanvas>` inside another `<HtmlInCanvas>`. Remotion throws:

```
<HtmlInCanvas> effects cannot be nested together. Chrome will only display the outer effect. Consider merging the effects into one if you can.
```

## Enabling WebGL during renders

If you make use of WebGL during renders, you need to enable it:

From the CLI:

```bash
npx remotion render --gl=angle
```

Set it as the default for Studio and CLI (advised):

```ts
import { Config } from "@remotion/cli/config";

Config.setChromiumOpenGlRenderer("angle");
```

## Basic usage

By default, draws to canvas with no effect applied:

```tsx
import { HtmlInCanvas } from "remotion";

export const MyComp = () => {
  return (
    <HtmlInCanvas width={1280} height={720}>
      <div style={{ fontSize: 80 }}>Hello</div>
    </HtmlInCanvas>
  );
};
```

## 2D effect with `onPaint`

`onPaint` runs whenever the content updates. Call `ctx.drawElementImage(elementImage, 0, 0)` to draw the captured DOM, and assign the returned transform to `element.style.transform` so DOM selection still aligns with the painted output.

```tsx
import {
  AbsoluteFill,
  HtmlInCanvas,
  type HtmlInCanvasOnPaint,
  useCurrentFrame,
  useVideoConfig,
} from "remotion";
import { useCallback } from "react";

export const Blur = () => {
  const frame = useCurrentFrame();
  const { width, height, fps } = useVideoConfig();

  const onPaint: HtmlInCanvasOnPaint = useCallback(
    ({ canvas, element, elementImage }) => {
      const ctx = canvas.getContext("2d");
      if (!ctx) throw new Error("Failed to acquire 2D context");

      const blurPx = 4 + 18 * (0.5 + 0.5 * Math.sin((frame / fps) * Math.PI));

      ctx.reset();
      ctx.filter = `blur(${blurPx}px)`;
      const transform = ctx.drawElementImage(elementImage, 0, 0);
      element.style.transform = transform.toString();
    },
    [frame, fps],
  );

  return (
    <HtmlInCanvas width={width} height={height} onPaint={onPaint}>
      <AbsoluteFill style={{ justifyContent: "center", alignItems: "center", fontSize: 120 }}>
        <h1>Hello</h1>
      </AbsoluteFill>
    </HtmlInCanvas>
  );
};
```

## WebGL effects

For WebGL, set up the context, program, and texture in `onInit` and return a cleanup function. Inside `onPaint`, upload the captured DOM with `gl.texElementImage2D(...)` and draw.

```tsx
const onInit: HtmlInCanvasOnInit = useCallback(({ canvas }) => {
  const gl = canvas.getContext("webgl2", { alpha: true, premultipliedAlpha: true });
  if (!gl) {
    throw new Error(
      "WebGL2 unavailable. Try rendering with the --gl=angle option. See https://remotion.dev/docs/gl-options.",
    );
  }
  gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, true);
  // compile program, create texture, set up VAO...
  return () => {
    // delete program, texture, buffers...
  };
}, []);

const onPaint: HtmlInCanvasOnPaint = useCallback(({ elementImage }) => {
  gl.texElementImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, elementImage);
  gl.drawArrays(gl.TRIANGLES, 0, 6);
}, []);
```

For a fully working minimal example, see https://github.com/remotion-dev/remotion/blob/main/packages/docs/components/demos/HtmlInCanvasDocsDemoWebGL.tsx.

## Async `onPaint`

`onPaint` may be `async`. Remotion holds the frame open via `delayRender()` until the promise resolves. Useful for multi-pass effects with `createImageBitmap`.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/images.md -->
```markdown
## Sizing and positioning

Use the `style` prop to control size and position:

```tsx
<Img
  src={staticFile("photo.png")}
  style={{
    width: 500,
    height: 300,
    position: "absolute",
    top: 100,
    left: 50,
    objectFit: "cover",
  }}
/>
```

## Dynamic image paths

Use template literals for dynamic file references:

```tsx
import { Img, staticFile, useCurrentFrame } from "remotion";

const frame = useCurrentFrame();

// Image sequence
<Img src={staticFile(`frames/frame${frame}.png`)} />

// Selecting based on props
<Img src={staticFile(`avatars/${props.userId}.png`)} />

// Conditional images
<Img src={staticFile(`icons/${isActive ? "active" : "inactive"}.svg`)} />
```

This pattern is useful for:

- Image sequences (frame-by-frame animations)
- User-specific avatars or profile images
- Theme-based icons
- State-dependent graphics

## Getting image dimensions

Use `getImageDimensions()` to get the dimensions of an image:

```tsx
import { getImageDimensions, staticFile } from "remotion";

const { width, height } = await getImageDimensions(staticFile("photo.png"));
```

This is useful for calculating aspect ratios or sizing compositions:

```tsx
import {
  getImageDimensions,
  staticFile,
  CalculateMetadataFunction,
} from "remotion";

const calculateMetadata: CalculateMetadataFunction = async () => {
  const { width, height } = await getImageDimensions(staticFile("photo.png"));
  return {
    width,
    height,
  };
};
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/import-srt-captions.md -->
```markdown
---
name: import-srt-captions
description: Importing .srt subtitle files into Remotion using @remotion/captions
metadata:
  tags: captions, subtitles, srt, import, parse
---

# Importing .srt subtitles into Remotion

If you have an existing `.srt` subtitle file, you can import it into Remotion using `parseSrt()` from `@remotion/captions`.

If you don't have a .srt file, read [Transcribing audio](transcribe-captions.md) for how to generate captions instead.

## Prerequisites

First, the @remotion/captions package needs to be installed.
If it is not installed, use the following command:

```bash
npx remotion add @remotion/captions # If project uses npm
bunx remotion add @remotion/captions # If project uses bun
yarn remotion add @remotion/captions # If project uses yarn
pnpm exec remotion add @remotion/captions # If project uses pnpm
```

## Reading an .srt file

Use `staticFile()` to reference an `.srt` file in your `public` folder, then fetch and parse it:

```tsx
import { useState, useEffect, useCallback } from "react";
import { AbsoluteFill, staticFile, useDelayRender } from "remotion";
import { parseSrt } from "@remotion/captions";
import type { Caption } from "@remotion/captions";

export const MyComponent: React.FC = () => {
  const [captions, setCaptions] = useState<Caption[] | null>(null);
  const { delayRender, continueRender, cancelRender } = useDelayRender();
  const [handle] = useState(() => delayRender());

  const fetchCaptions = useCallback(async () => {
    try {
      const response = await fetch(staticFile("subtitles.srt"));
      const text = await response.text();
      const { captions: parsed } = parseSrt({ input: text });
      setCaptions(parsed);
      continueRender(handle);
    } catch (e) {
      cancelRender(e);
    }
  }, [continueRender, cancelRender, handle]);

  useEffect(() => {
    fetchCaptions();
  }, [fetchCaptions]);

  if (!captions) {
    return null;
  }

  return <AbsoluteFill>{/* Use captions here */}</AbsoluteFill>;
};
```

Remote URLs are also supported - you can `fetch()` a remote file via URL instead of using `staticFile()`.

## Using imported captions

Once parsed, the captions are in the `Caption` format and can be used with all `@remotion/captions` utilities.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/light-leaks.md -->
```markdown
---
name: light-leaks
description: Light leak overlay effects for Remotion using @remotion/light-leaks.
metadata:
  tags: light-leaks, overlays, effects, transitions
---

## Light Leaks

This only works from Remotion 4.0.415 and up. Use `npx remotion versions` to check your Remotion version and `npx remotion upgrade` to upgrade your Remotion version.

`<LightLeak>` from `@remotion/light-leaks` renders a WebGL-based light leak effect. It reveals during the first half of its duration and retracts during the second half.

Typically used inside a `<TransitionSeries.Overlay>` to play over the cut point between two scenes. See the **transitions** rule for `<TransitionSeries>` and overlay usage.

## Prerequisites

```bash
npx remotion add @remotion/light-leaks
```

## Basic usage with TransitionSeries

```tsx
import { TransitionSeries } from "@remotion/transitions";
import { LightLeak } from "@remotion/light-leaks";

<TransitionSeries>
  <TransitionSeries.Sequence durationInFrames={60}>
    <SceneA />
  </TransitionSeries.Sequence>
  <TransitionSeries.Overlay durationInFrames={30}>
    <LightLeak />
  </TransitionSeries.Overlay>
  <TransitionSeries.Sequence durationInFrames={60}>
    <SceneB />
  </TransitionSeries.Sequence>
</TransitionSeries>;
```

## Props

- `durationInFrames?` — defaults to the parent sequence/composition duration. The effect reveals during the first half and retracts during the second half.
- `seed?` — determines the shape of the light leak pattern. Different seeds produce different patterns. Default: `0`.
- `hueShift?` — rotates the hue in degrees (`0`–`360`). Default: `0` (yellow-to-orange). `120` = green, `240` = blue.

## Customizing the look

```tsx
import { LightLeak } from "@remotion/light-leaks";

// Blue-tinted light leak with a different pattern
<LightLeak seed={5} hueShift={240} />;

// Green-tinted light leak
<LightLeak seed={2} hueShift={120} />;
```

## Standalone usage

`<LightLeak>` can also be used outside of `<TransitionSeries>`, for example as a decorative overlay in any composition:

```tsx
import { AbsoluteFill } from "remotion";
import { LightLeak } from "@remotion/light-leaks";

const MyComp: React.FC = () => (
  <AbsoluteFill>
    <MyContent />
    <LightLeak durationInFrames={60} seed={3} />
  </AbsoluteFill>
);
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/local-fonts.md -->
```markdown
For local font files, use the `@remotion/fonts` package.

### Prerequisites

First, install @remotion/fonts:

```bash
npx remotion add @remotion/fonts # If project uses npm
bunx remotion add @remotion/fonts # If project uses bun
yarn remotion add @remotion/fonts # If project uses yarn
pnpm exec remotion add @remotion/fonts # If project uses pnpm
```

### Loading a local font

Place your font file in the `public/` folder and use `loadFont()`:

```tsx
import { loadFont } from "@remotion/fonts";
import { staticFile } from "remotion";

await loadFont({
  family: "MyFont",
  url: staticFile("MyFont-Regular.woff2"),
});

export const MyComposition = () => {
  return <div style={{ fontFamily: "MyFont" }}>Hello World</div>;
};
```

### Loading multiple weights

Load each weight separately with the same family name:

```tsx
import { loadFont } from "@remotion/fonts";
import { staticFile } from "remotion";

await Promise.all([
  loadFont({
    family: "Inter",
    url: staticFile("Inter-Regular.woff2"),
    weight: "400",
  }),
  loadFont({
    family: "Inter",
    url: staticFile("Inter-Bold.woff2"),
    weight: "700",
  }),
]);
```

### Available options

```tsx
loadFont({
  family: "MyFont", // Required: name to use in CSS
  url: staticFile("font.woff2"), // Required: font file URL
  format: "woff2", // Optional: auto-detected from extension
  weight: "400", // Optional: font weight
  style: "normal", // Optional: normal or italic
  display: "block", // Optional: font-display behavior
});
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/lottie.md -->
```markdown
---
name: lottie
description: Embedding Lottie animations in Remotion.
metadata:
  category: Animation
---

# Using Lottie Animations in Remotion

## Prerequisites

First, the @remotion/lottie package needs to be installed.  
If it is not, use the following command:

```bash
npx remotion add @remotion/lottie # If project uses npm
bunx remotion add @remotion/lottie # If project uses bun
yarn remotion add @remotion/lottie # If project uses yarn
pnpm exec remotion add @remotion/lottie # If project uses pnpm
```

## Displaying a Lottie file

To import a Lottie animation:

- Fetch the Lottie asset
- Wrap the loading process in `delayRender()` and `continueRender()`
- Save the animation data in a state
- Render the Lottie animation using the `Lottie` component from the `@remotion/lottie` package

```tsx
import { Lottie, LottieAnimationData } from "@remotion/lottie";
import { useEffect, useState } from "react";
import { cancelRender, continueRender, delayRender } from "remotion";

export const MyAnimation = () => {
  const [handle] = useState(() => delayRender("Loading Lottie animation"));

  const [animationData, setAnimationData] =
    useState<LottieAnimationData | null>(null);

  useEffect(() => {
    fetch("https://assets4.lottiefiles.com/packages/lf20_zyquagfl.json")
      .then((data) => data.json())
      .then((json) => {
        setAnimationData(json);
        continueRender(handle);
      })
      .catch((err) => {
        cancelRender(err);
      });
  }, [handle]);

  if (!animationData) {
    return null;
  }

  return <Lottie animationData={animationData} />;
};
```

## Styling and animating

Lottie supports the `style` prop to allow styles and animations:

```tsx
return (
  <Lottie animationData={animationData} style={{ width: 400, height: 400 }} />
);
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/maplibre.md -->
```markdown
---
name: maps
description: Make deterministic Remotion map animations with MapLibre GL JS and Turf. Use for animated routes, flyovers, map markers, labels, and camera movement.
metadata:
  tags: map, map animation, maplibre, turf, geojson, route animation
---

Use MapLibre GL JS for rendering maps in Remotion. Use Turf for geospatial operations such as great-circle routes, distances, slicing lines, and positions along routes.

## Core rules

- Prefer `@turf/turf` for geospatial work. Do not hand-roll distance, great-circle, route slicing, or coordinate interpolation unless the user explicitly needs a custom non-geodesic effect.
- Use GeoJSON sources and MapLibre layers for lines, markers, and labels. Avoid DOM `Marker` elements unless the user specifically asks for HTML markers.
- Disable non-deterministic map behavior: `interactive: false`, `fadeDuration: 0`.
- Use `delayRender()` / `continueRender()` around map loading and per-frame map updates.
- Before continuing the initial render, add sources/layers, apply the frame-0 camera with `jumpTo()`, then wait for `idle`.
- Do not add a `mapInstance.remove()` cleanup function; it can interfere with Remotion's render lifecycle.
- Use standard MapLibre style JSON URLs and layer/source APIs.
- Do not install `@types/maplibre-gl`; MapLibre ships its own types.

Coordinates in MapLibre, Turf, and GeoJSON are `[longitude, latitude]`.

```ts
const zurich: [number, number] = [8.5417, 47.3769];
const newYork: [number, number] = [-74.006, 40.7128];
```

## Prerequisites

Install MapLibre and Turf with the project's package manager.

```bash
npm i maplibre-gl @turf/turf
```

```bash
bun i maplibre-gl @turf/turf
```

```bash
yarn add maplibre-gl @turf/turf
```

```bash
pnpm i maplibre-gl @turf/turf
```

Import the MapLibre CSS once in the component or an app-level stylesheet:

```ts
import 'maplibre-gl/dist/maplibre-gl.css';
```

## Basic map example

```tsx
import {useEffect, useRef, useState} from 'react';
import {AbsoluteFill, useDelayRender, useVideoConfig} from 'remotion';
import maplibregl from 'maplibre-gl';
import 'maplibre-gl/dist/maplibre-gl.css';

const zurich: [number, number] = [8.5417, 47.3769];

export const MyComposition = () => {
	const containerRef = useRef<HTMLDivElement>(null);
	const {delayRender, continueRender} = useDelayRender();
	const {width, height} = useVideoConfig();
	const [loadingHandle] = useState(() => delayRender('Loading map'));

	useEffect(() => {
		if (!containerRef.current) {
			return;
		}

		const mapInstance = new maplibregl.Map({
			container: containerRef.current,
			style: 'https://demotiles.maplibre.org/style.json',
			center: zurich,
			zoom: 7,
			interactive: false,
			attributionControl: false,
			fadeDuration: 0,
			canvasContextAttributes: {
				preserveDrawingBuffer: true,
			},
		});

		mapInstance.on('load', () => {
			mapInstance.jumpTo({center: zurich, zoom: 7});
			mapInstance.once('idle', () => {
				continueRender(loadingHandle);
			});
		});
	}, [continueRender, loadingHandle]);

	return (
		<AbsoluteFill>
			<div ref={containerRef} style={{width, height, position: 'absolute'}} />
		</AbsoluteFill>
	);
};
```

Animated examples should keep the loaded map in React state and skip per-frame updates until that state is set.

## Animated flight route example

This example shows the recommended pattern for route animations:

- Turf creates the route and markers.
- Turf slices the route for line reveal animation.
- The camera has a separate route from the target route.
- MapLibre's `calculateCameraOptionsFromTo()` is used for camera movement.
- Frame 0 is prepared before `continueRender()`.

```tsx
import * as turf from '@turf/turf';
import {useEffect, useRef, useState} from 'react';
import {
	AbsoluteFill,
	Easing,
	interpolate,
	useCurrentFrame,
	useDelayRender,
	useVideoConfig,
} from 'remotion';
import maplibregl, {type GeoJSONSource, type Map} from 'maplibre-gl';
import 'maplibre-gl/dist/maplibre-gl.css';

const zurich: [number, number] = [8.5417, 47.3769];
const newYork: [number, number] = [-74.006, 40.7128];

const greatCircleLine = (from: [number, number], to: [number, number]) => {
	const route = turf.greatCircle(from, to, {npoints: 100});

	if (route.geometry.type === 'LineString') {
		return turf.lineString(route.geometry.coordinates);
	}

	// Great-circle routes crossing the antimeridian can become MultiLineString.
	// Keep the example valid by choosing the longest segment.
	const longestSegment = route.geometry.coordinates.reduce((longest, segment) => {
		return segment.length > longest.length ? segment : longest;
	});

	return turf.lineString(longestSegment);
};

const targetRoute = greatCircleLine(zurich, newYork);
const targetRouteDistance = turf.length(targetRoute);

const cameraRoute = greatCircleLine(zurich, newYork);
const cameraRouteDistance = turf.length(cameraRoute);

const cityMarkers = turf.featureCollection([
	turf.point(zurich, {name: 'Zurich'}),
	turf.point(newYork, {name: 'New York'}),
]);

const clampProgress = (progress: number) => Math.min(1, Math.max(0, progress));

const distanceAlong = (totalDistance: number, progress: number) => {
	// Keep the route non-empty at progress 0; Turf can error on zero-length slices.
	return Math.max(0.001, totalDistance * clampProgress(progress));
};

const getPartialTargetRoute = (progress: number) => {
	return turf.lineSliceAlong(
		targetRoute,
		0,
		distanceAlong(targetRouteDistance, progress),
	);
};

const getCameraOptions = (
	map: Map,
	progress: number,
	cameraAltitudeMeters: number,
	cameraLatitudeOffset: number,
) => {
	const target = turf.along(
		targetRoute,
		distanceAlong(targetRouteDistance, progress),
	).geometry.coordinates;
	const camera = turf.along(
		cameraRoute,
		distanceAlong(cameraRouteDistance, progress),
	).geometry.coordinates;

	return map.calculateCameraOptionsFromTo(
		new maplibregl.LngLat(camera[0], camera[1] - cameraLatitudeOffset),
		cameraAltitudeMeters,
		new maplibregl.LngLat(target[0], target[1]),
	);
};

export const MyComposition = () => {
	const containerRef = useRef<HTMLDivElement>(null);
	const frame = useCurrentFrame();
	const {delayRender, continueRender} = useDelayRender();
	const {durationInFrames, height, width} = useVideoConfig();
	const [map, setMap] = useState<Map | null>(null);
	const [loadingHandle] = useState(() => delayRender('Loading MapLibre map'));

	useEffect(() => {
		if (!containerRef.current) {
			return;
		}

		const mapInstance = new maplibregl.Map({
			container: containerRef.current,
			style: 'https://demotiles.maplibre.org/style.json',
			center: zurich,
			zoom: 7,
			interactive: false,
			attributionControl: false,
			fadeDuration: 0,
			canvasContextAttributes: {
				preserveDrawingBuffer: true,
			},
		});

		mapInstance.on('load', () => {
			mapInstance.addSource('trace', {
				type: 'geojson',
				data: getPartialTargetRoute(0),
			});

			mapInstance.addLayer({
				id: 'trace-line',
				type: 'line',
				source: 'trace',
				layout: {
					'line-cap': 'round',
					'line-join': 'round',
				},
				paint: {
					'line-color': '#111111',
					'line-width': 7,
				},
			});

			mapInstance.addSource('city-markers', {
				type: 'geojson',
				data: cityMarkers,
			});

			mapInstance.addLayer({
				id: 'city-marker-dots',
				type: 'circle',
				source: 'city-markers',
				paint: {
					'circle-color': '#f03b20',
					'circle-radius': 12,
					'circle-stroke-color': '#ffffff',
					'circle-stroke-width': 4,
				},
			});

			mapInstance.addLayer({
				id: 'city-marker-labels',
				type: 'symbol',
				source: 'city-markers',
				layout: {
					'text-allow-overlap': true,
					'text-anchor': 'top',
					'text-field': ['get', 'name'],
					'text-offset': [0, 0.9],
					'text-size': 28,
				},
				paint: {
					'text-color': '#111111',
					'text-halo-color': '#ffffff',
					'text-halo-width': 3,
				},
			});

			mapInstance.jumpTo(getCameraOptions(mapInstance, 0, 180000, 1.1));
			mapInstance.once('idle', () => {
				setMap(mapInstance);
				continueRender(loadingHandle);
			});
		});
	}, [continueRender, loadingHandle]);

	useEffect(() => {
		if (!map) {
			return;
		}

		const handle = delayRender('Rendering MapLibre frame');
		const timelineProgress = interpolate(frame, [0, durationInFrames - 1], [0, 1], {
			extrapolateLeft: 'clamp',
			extrapolateRight: 'clamp',
		});
		const travelProgress = interpolate(timelineProgress, [0.2, 0.82], [0, 1], {
			extrapolateLeft: 'clamp',
			extrapolateRight: 'clamp',
			easing: Easing.inOut(Easing.cubic),
		});
		const cameraAltitudeMeters = interpolate(
			timelineProgress,
			[0, 0.28, 0.74, 1],
			[180000, 2200000, 2200000, 180000],
			{
				extrapolateLeft: 'clamp',
				extrapolateRight: 'clamp',
				easing: Easing.inOut(Easing.cubic),
			},
		);
		const cameraLatitudeOffset = interpolate(
			timelineProgress,
			[0, 0.28, 0.74, 1],
			[1.1, 8, 8, 1.1],
			{
				extrapolateLeft: 'clamp',
				extrapolateRight: 'clamp',
				easing: Easing.inOut(Easing.cubic),
			},
		);
		const trace = map.getSource('trace') as GeoJSONSource | undefined;

		trace?.setData(getPartialTargetRoute(travelProgress));
		map.jumpTo(
			getCameraOptions(
				map,
				travelProgress,
				cameraAltitudeMeters,
				cameraLatitudeOffset,
			),
		);

		map.once('idle', () => continueRender(handle));
		// Force an idle event even if the camera parameters are unchanged from the previous frame.
		map.triggerRepaint();
	}, [continueRender, delayRender, durationInFrames, frame, map]);

	return (
		<AbsoluteFill style={{backgroundColor: '#e8eef3'}}>
			<div ref={containerRef} style={{height, position: 'absolute', width}} />
		</AbsoluteFill>
	);
};
```

## Camera guidance

Use MapLibre's camera helper for camera movement:

```ts
map.calculateCameraOptionsFromTo(cameraLngLat, cameraAltitudeMeters, targetLngLat);
```

A good pattern is to keep two concepts separate:

- `targetRoute`: where the animated line is and where the camera looks.
- `cameraRoute`: where the camera moves.

Then use Turf to read positions from both routes for the same progress value:

```ts
const target = turf.along(targetRoute, targetDistance * progress).geometry.coordinates;
const camera = turf.along(cameraRoute, cameraDistance * progress).geometry.coordinates;

map.jumpTo(
	map.calculateCameraOptionsFromTo(
		new maplibregl.LngLat(camera[0], camera[1]),
		cameraAltitudeMeters,
		new maplibregl.LngLat(target[0], target[1]),
	),
);
```

For zoom-out / travel / zoom-in animations, animate travel progress separately from camera altitude. Camera altitude is measured in meters. This avoids heavy custom camera math.

## Lines

Use GeoJSON sources for lines. Unless the user asks, do not add glow effects or extra decorative points.

For geodesic flight routes, use Turf:

```ts
const line = greatCircleLine(start, end);
const distance = turf.length(line);
const partialLine = turf.lineSliceAlong(
	line,
	0,
	// Keep the route non-empty at progress 0.
	Math.max(0.001, distance * progress),
);
```

For a visually straight line on the map, use a simple GeoJSON `LineString` between the two points instead of `greatCircle()`.

## Markers and labels

Use map-native GeoJSON layers for markers and labels:

```tsx
mapInstance.addSource('markers', {
	type: 'geojson',
	data: turf.featureCollection([
		turf.point([-118.2437, 34.0522], {name: 'Los Angeles'}),
	]),
});

mapInstance.addLayer({
	id: 'marker-dots',
	type: 'circle',
	source: 'markers',
	paint: {
		'circle-color': '#f03b20',
		'circle-radius': 12,
		'circle-stroke-color': '#ffffff',
		'circle-stroke-width': 4,
	},
});

mapInstance.addLayer({
	id: 'marker-labels',
	type: 'symbol',
	source: 'markers',
	layout: {
		'text-allow-overlap': true,
		'text-anchor': 'top',
		'text-field': ['get', 'name'],
		'text-offset': [0, 0.9],
		'text-size': 28,
	},
	paint: {
		'text-color': '#111111',
		'text-halo-color': '#ffffff',
		'text-halo-width': 3,
	},
});
```

Make marker sizes and label font sizes large enough for the composition resolution.

## Styles

Default to the stock MapLibre demo style:

```ts
style: 'https://demotiles.maplibre.org/style.json'
```

If the user requests another style, use any valid MapLibre style JSON URL.

## Rendering

For WebGL map renders, prefer single concurrency and ANGLE:

```bash
bunx remotion render [composition-id] out/video.mp4 --gl=angle --concurrency=1
```

Use the equivalent package runner for the project. In npm projects, use `npx`; in Bun projects, use `bunx`.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/measuring-dom-nodes.md -->
```markdown
---
name: measuring-dom-nodes
description: Measuring DOM element dimensions in Remotion
metadata:
  tags: measure, layout, dimensions, getBoundingClientRect, scale
---

# Measuring DOM nodes in Remotion

Remotion applies a `scale()` transform to the video container, which affects values from `getBoundingClientRect()`. Use `useCurrentScale()` to get correct measurements.

## Measuring element dimensions

```tsx
import { useCurrentScale } from "remotion";
import { useRef, useEffect, useState } from "react";

export const MyComponent = () => {
  const ref = useRef<HTMLDivElement>(null);
  const scale = useCurrentScale();
  const [dimensions, setDimensions] = useState({ width: 0, height: 0 });

  useEffect(() => {
    if (!ref.current) return;
    const rect = ref.current.getBoundingClientRect();
    setDimensions({
      width: rect.width / scale,
      height: rect.height / scale,
    });
  }, [scale]);

  return <div ref={ref}>Content to measure</div>;
};
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/measuring-text.md -->
```markdown
---
name: measuring-text
description: Measuring text dimensions, fitting text to containers, and checking overflow
metadata:
  tags: measure, text, layout, dimensions, fitText, fillTextBox
---

# Measuring text in Remotion

## Prerequisites

Install @remotion/layout-utils if it is not already installed:

```bash
npx remotion add @remotion/layout-utils
```

## Measuring text dimensions

Use `measureText()` to calculate the width and height of text:

```tsx
import { measureText } from "@remotion/layout-utils";

const { width, height } = measureText({
  text: "Hello World",
  fontFamily: "Arial",
  fontSize: 32,
  fontWeight: "bold",
});
```

Results are cached - duplicate calls return the cached result.

## Fitting text to a width

Use `fitText()` to find the optimal font size for a container:

```tsx
import { fitText } from "@remotion/layout-utils";

const { fontSize } = fitText({
  text: "Hello World",
  withinWidth: 600,
  fontFamily: "Inter",
  fontWeight: "bold",
});

return (
  <div
    style={{
      fontSize: Math.min(fontSize, 80), // Cap at 80px
      fontFamily: "Inter",
      fontWeight: "bold",
    }}
  >
    Hello World
  </div>
);
```

## Checking text overflow

Use `fillTextBox()` to check if text exceeds a box:

```tsx
import { fillTextBox } from "@remotion/layout-utils";

const box = fillTextBox({ maxBoxWidth: 400, maxLines: 3 });

const words = ["Hello", "World", "This", "is", "a", "test"];
for (const word of words) {
  const { exceedsBox } = box.add({
    text: word + " ",
    fontFamily: "Arial",
    fontSize: 24,
  });
  if (exceedsBox) {
    // Text would overflow, handle accordingly
    break;
  }
}
```

## Best practices

**Load fonts first:** Only call measurement functions after fonts are loaded.

```tsx
import { loadFont } from "@remotion/google-fonts/Inter";

const { fontFamily, waitUntilDone } = loadFont("normal", {
  weights: ["400"],
  subsets: ["latin"],
});

waitUntilDone().then(() => {
  // Now safe to measure
  const { width } = measureText({
    text: "Hello",
    fontFamily,
    fontSize: 32,
  });
});
```

**Use validateFontIsLoaded:** Catch font loading issues early:

```tsx
measureText({
  text: "Hello",
  fontFamily: "MyCustomFont",
  fontSize: 32,
  validateFontIsLoaded: true, // Throws if font not loaded
});
```

**Match font properties:** Use the same properties for measurement and rendering:

```tsx
const fontStyle = {
  fontFamily: "Inter",
  fontSize: 32,
  fontWeight: "bold" as const,
  letterSpacing: "0.5px",
};

const { width } = measureText({
  text: "Hello",
  ...fontStyle,
});

return <div style={fontStyle}>Hello</div>;
```

**Avoid padding and border:** Use `outline` instead of `border` to prevent layout differences:

```tsx
<div style={{ outline: "2px solid red" }}>Text</div>
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/parameters.md -->
```markdown
---
name: parameters
description: Make a video parametrizable by adding a Zod schema
metadata:
  tags: parameters, zod, schema
---

To make a video parametrizable, a Zod schema can be added to a composition.

First, `zod` must be installed .

Search the project for lockfiles and run the correct command depending on the package manager:

If `package-lock.json` is found, use the following command:

```bash
npm i zod
```

If `bun.lockb` is found, use the following command:

```bash
bun i zod
```

If `yarn.lock` is found, use the following command:

```bash
yarn add zod
```

If `pnpm-lock.yaml` is found, use the following command:

```bash
pnpm i zod
```

Then, a Zod schema can be defined alongside the component:

```tsx title="src/MyComposition.tsx"
import { z } from "zod";

export const MyCompositionSchema = z.object({
  title: z.string(),
});

const MyComponent: React.FC<z.infer<typeof MyCompositionSchema>> = () => {
  return (
    <div>
      <h1>{props.title}</h1>
    </div>
  );
};
```

In the root file, the schema can be passed to the composition:

```tsx title="src/Root.tsx"
import { Composition } from "remotion";
import { MycComponent, MyCompositionSchema } from "./MyComposition";

export const RemotionRoot = () => {
  return (
    <Composition
      id="MyComposition"
      component={MyComponent}
      durationInFrames={100}
      fps={30}
      width={1080}
      height={1080}
      defaultProps={{ title: "Hello World" }}
      schema={MyCompositionSchema}
    />
  );
};
```

Now, the user can edit the parameter visually in the sidebar.

All schemas that are supported by Zod are supported by Remotion.

Remotion requires that the top-level type is a z.object(), because the collection of props of a React component is always an object.

## Color picker

For adding a color picker, use `zColor()` from `@remotion/zod-types`.

If it is not installed, use the following command:

```bash
npx remotion add @remotion/zod-types # If project uses npm
bunx remotion add @remotion/zod-types # If project uses bun
yarn remotion add @remotion/zod-types # If project uses yarn
pnpm exec remotion add @remotion/zod-types # If project uses pnpm
```

Then import `zColor` from `@remotion/zod-types`:

```tsx
import { zColor } from "@remotion/zod-types";
```

Then use it in the schema:

```tsx
export const MyCompositionSchema = z.object({
  color: zColor(),
});
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/sequencing.md -->
```markdown
---
name: sequencing
description: Sequencing patterns for Remotion - delay, trim, limit duration of items
metadata:
  tags: sequence, series, timing, delay, trim
---

Use `<Sequence>` to delay when an element appears in the timeline.

```tsx
import { Sequence } from "remotion";

export const Title = () => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();

  const opacity = interpolate(frame, [0, 2 * fps], [0, 1], {
    extrapolateRight: "clamp",
    extrapolateLeft: "clamp",
    easing: Easing.bezier(0.16, 1, 0.3, 1),
  });

  return <div style={{ opacity }}>Title</div>;
};

export const Subtitle = () => {
  return <div>Subtitle</div>;
};

const Main = () => {
  const {fps} = useVideoConfig();

  return (
    <AbsoluteFill>
      <Sequence>
        <Background />
      </Sequence>
      <Sequence from={1 * fps} durationInFrames={2 * fps} layout="none">
        <Title />
      </Sequence>
      <Sequence from={2 * fps} durationInFrames={2 * fps} layout="none">
        <Subtitle />
      </Sequence>
    </AbsoluteFill>
  );
}
```

This will by default wrap the component in an absolute fill element.  
If the items should not be wrapped, use the `layout` prop:

```tsx
<Sequence layout="none">
  <Title />
</Sequence>
```

## Premounting

This loads the component in the timeline before it is actually played.  
Always premount any `<Sequence>`!

```tsx
<Sequence premountFor={1 * fps}>
  <Title />
</Sequence>
```

## Series

Use `<Series>` when elements should play one after another without overlap.

```tsx
import { Series } from "remotion";

<Series>
  <Series.Sequence durationInFrames={45}>
    <Intro />
  </Series.Sequence>
  <Series.Sequence durationInFrames={60}>
    <MainContent />
  </Series.Sequence>
  <Series.Sequence durationInFrames={30}>
    <Outro />
  </Series.Sequence>
</Series>;
```

Same as with `<Sequence>`, the items will be wrapped in an absolute fill element by default when using `<Series.Sequence>`, unless the `layout` prop is set to `none`.

### Series with overlaps

Use negative offset for overlapping sequences:

```tsx
<Series>
  <Series.Sequence durationInFrames={60}>
    <SceneA />
  </Series.Sequence>
  <Series.Sequence offset={-15} durationInFrames={60}>
    {/* Starts 15 frames before SceneA ends */}
    <SceneB />
  </Series.Sequence>
</Series>
```

## Frame References Inside Sequences

Inside a Sequence, `useCurrentFrame()` returns the local frame (starting from 0):

```tsx
<Sequence from={60} durationInFrames={30}>
  <MyComponent />
  {/* Inside MyComponent, useCurrentFrame() returns 0-29, not 60-89 */}
</Sequence>
```

## Nested Sequences

Sequences can be nested for complex timing:

```tsx
<Sequence from={0} durationInFrames={120}>
  <Background />
  <Sequence from={15} durationInFrames={90} layout="none">
    <Title />
  </Sequence>
  <Sequence from={45} durationInFrames={60} layout="none">
    <Subtitle />
  </Sequence>
</Sequence>
```

## Nesting compositions within another

To add a composition within another composition, you can use the `<Sequence>` component with a `width` and `height` prop to specify the size of the composition.

```tsx
<AbsoluteFill>
  <Sequence width={COMPOSITION_WIDTH} height={COMPOSITION_HEIGHT}>
    <CompositionComponent />
  </Sequence>
</AbsoluteFill>
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/sfx.md -->
```markdown
---
name: sfx
description: Including sound effects
metadata:
  tags: sfx, sound, effect, audio
---

To include a sound effect, use the `<Audio>` tag:

```tsx
import { Audio } from "@remotion/sfx";

<Audio src={"https://remotion.media/whoosh.wav"} />;
```

The following sound effects are available:

- `https://remotion.media/whoosh.wav`
- `https://remotion.media/whip.wav`
- `https://remotion.media/page-turn.wav`
- `https://remotion.media/switch.wav`
- `https://remotion.media/mouse-click.wav`
- `https://remotion.media/shutter-modern.wav`
- `https://remotion.media/shutter-old.wav`
- `https://remotion.media/ding.wav`
- `https://remotion.media/bruh.wav`
- `https://remotion.media/vine-boom.wav`
- `https://remotion.media/windows-xp-error.wav`

For more sound effects, search the internet. A good resource is https://github.com/kapishdima/soundcn/tree/main/assets.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/silence-detection.md -->
```markdown
---
name: silence-detection
description: Adaptive silence detection for video/audio files using FFmpeg loudnorm and silencedetect
metadata:
  tags: silence, detection, trimming, ffmpeg, loudnorm, audio
---

# Adaptive Silence Detection

Detect silent segments in video or audio files.

Requires FFmpeg — see [ffmpeg.md](./ffmpeg.md) for how to invoke it in Remotion projects.

## Step 1: Measure loudness with `loudnorm`

Use the `loudnorm` filter in JSON mode to get the EBU R128 integrated loudness and gating threshold for each file:

```bash
npx remotion ffmpeg -i public/video.mov -map 0:a -af loudnorm=print_format=json -f null /dev/null
```

As output you will get:
- `input_i`: Integrated loudness (dB) — the overall perceived volume
- `input_thresh`: EBU R128 gating threshold (dB) — the level below which audio is considered too quiet to count toward loudness measurement

## Step 2: Detect silences using adaptive threshold

Pass the `input_thresh` value from step 1 as the `noise` parameter to `silencedetect`:

```bash
npx remotion ffmpeg -i public/video.mov -map 0:a -af "silencedetect=noise=${THRESH}dB:d=0.5" -f null /dev/null
```

Parameters:
- `noise`: The threshold below which audio is considered silent. Use `input_thresh` from step 1.
- `d`: Minimum silence duration in seconds. `0.5` is a good default.

## Interpreting the output

The filter outputs pairs of `silence_start` and `silence_end` timestamps:

```
[silencedetect] silence_start: 0
[silencedetect] silence_end: 2.241021 | silence_duration: 2.241021
[silencedetect] silence_start: 38.77425
[silencedetect] silence_end: 39.619604 | silence_duration: 0.845354
```

## Identifying leading and trailing silence

- **Leading silence**: Consecutive silence segments starting at or near 0. If the first `silence_start` is > 0.5s, there is no leading silence.
- **Trailing silence**: The last silence segment that extends to (or near) the end of the file. Compare the last `silence_end` with the file's total duration.

When multiple silences are nearly contiguous at the start or end (gap < 0.2s), treat them as a single leading/trailing silence block.

## Using with Remotion's `<Video>` component

Apply the detected trim points using `trimBefore` and `trimAfter` (values are in frames):

```tsx
import { Video } from "@remotion/media";
import { staticFile, useVideoConfig } from "remotion";

const { fps } = useVideoConfig();

<Video
  src={staticFile("video.mov")}
  trimBefore={Math.floor(leadingEnd * fps)}
  trimAfter={Math.ceil(trailingStart * fps)}
/>
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/subtitles.md -->
```markdown
---
name: subtitles
description: subtitles and caption rules
metadata:
  tags: subtitles, captions, remotion, json
---

All captions must be processed in JSON. The captions must use the `Caption` type which is the following:

```ts
import type { Caption } from "@remotion/captions";
```

This is the definition:

```ts
type Caption = {
  text: string;
  startMs: number;
  endMs: number;
  timestampMs: number | null;
  confidence: number | null;
};
```

## Generating captions

To transcribe video and audio files to generate captions, load the [./transcribe-captions.md](./transcribe-captions.md) file for more instructions.

## Displaying captions

To display captions in your video, load the [./display-captions.md](./display-captions.md) file for more instructions.

## Importing captions

To import captions from a .srt file, load the [./import-srt-captions.md](./import-srt-captions.md) file for more instructions.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/tailwind.md -->
```markdown
---
name: tailwind
description: Using TailwindCSS in Remotion.
metadata:
---

You can and should use TailwindCSS in Remotion, if TailwindCSS is installed in the project.

Don't use `transition-*` or `animate-*` classes - always animate using the `useCurrentFrame()` hook.

Tailwind must be installed and enabled first in a Remotion project - fetch https://www.remotion.dev/docs/tailwind using WebFetch for instructions.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/text-animations.md -->
```markdown
---
name: text-animations
description: Typography and text animation patterns for Remotion.
metadata:
  tags: typography, text, typewriter, highlighter ken
---

## Text animations

Based on `useCurrentFrame()`, reduce the string character by character to create a typewriter effect.

## Typewriter Effect

See [Typewriter](assets/text-animations-typewriter.tsx) for an advanced example with a blinking cursor and a pause after the first sentence.

Always use string slicing for typewriter effects. Never use per-character opacity.

## Word Highlighting

See [Word Highlight](assets/text-animations-word-highlight.tsx) for an example for how a word highlight is animated, like with a highlighter pen.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/timing.md -->
```markdown
---
name: timing
description: Interpolation and timing in Remotion—prefer interpolate with Bézier easing; springs as a specialized option
metadata:
  tags: easing, bezier, interpolation, spring, timing
---

Drive motion with `interpolate()` over explicit frame range. To customize timing, use **`Easing.bezier`**. The four parameters are the same as CSS `cubic-bezier(x1, y1, x2, y2)`.

A simple linear interpolation is done using the `interpolate` function.

```ts title="Going from 0 to 1 over 100 frames"
import { interpolate } from "remotion";

const opacity = interpolate(frame, [0, 100], [0, 1]);
```

By default, the values are not clamped, so the value can go outside the range [0, 1].  
Here is how they can be clamped:

```ts title="Going from 0 to 1 over 100 frames with extrapolation"
const opacity = interpolate(frame, [0, 100], [0, 1], {
  extrapolateRight: "clamp",
  extrapolateLeft: "clamp",
});
```

## Bézier easing

Use `Easing.bezier(x1, y1, x2, y2)` inside the `interpolate` options object. The curve is identical in spirit to CSS animations and transitions, which helps when you are stealing timing from the web or from a designer’s spec.

```ts
import { interpolate, Easing } from "remotion";

const opacity = interpolate(frame, [0, 60], [0, 1], {
  easing: Easing.bezier(0.16, 1, 0.3, 1),
  extrapolateLeft: "clamp",
  extrapolateRight: "clamp",
});
```

### Examples (copy-paste curves)

**1. Crisp UI entrance (strong ease-out, no overshoot)** — slows nicely into the rest value; similar to many system “deceleration” curves.

```tsx
const enter = interpolate(frame, [0, 45], [0, 1], {
  easing: Easing.bezier(0.16, 1, 0.3, 1),
  extrapolateLeft: "clamp",
  extrapolateRight: "clamp",
});
```

**2. Editorial / slow fade (balanced ease-in-out)** — symmetric acceleration and deceleration over a hold-friendly move.

```tsx
const progress = interpolate(frame, [0, 90], [0, 1], {
  easing: Easing.bezier(0.45, 0, 0.55, 1),
  extrapolateLeft: "clamp",
  extrapolateRight: "clamp",
});
```

**3. Playful overshoot (control point y > 1)** — a little past the target then settles; use sparingly for emphasis.

```tsx
const pop = interpolate(frame, [0, 30], [0, 1], {
  easing: Easing.bezier(0.34, 1.56, 0.64, 1),
  extrapolateLeft: "clamp",
  extrapolateRight: "clamp",
});
```

## Preset easings (`Easing.in` / `Easing.out` / named curves)

Easing can be added to the `interpolate` function without a custom cubic:

```ts
import { interpolate, Easing } from "remotion";

const value1 = interpolate(frame, [0, 100], [0, 1], {
  easing: Easing.inOut(Easing.cubic),
  extrapolateLeft: "clamp",
  extrapolateRight: "clamp",
});
```

The default easing is `Easing.linear`.  
Convexities:

- `Easing.in` — starting slow and accelerating
- `Easing.out` — starting fast and slowing down
- `Easing.inOut`

Named curves (from most linear to most curved):

- `Easing.quad`
- `Easing.cubic` (good default when you do not need a custom cubic)
- `Easing.sin`
- `Easing.exp`
- `Easing.circle`

### Easing direction for enter/exit animations

Use `Easing.out` for enter animations (starts fast, decelerates into place) and `Easing.in` for exit animations (starts slow, accelerates away). This feels natural because elements arrive with momentum and leave with gravity. When you need a specific curve from design, prefer a single `Easing.bezier(...)` instead of stacking presets.

## Composing interpolations

When multiple properties share the same timing (e.g. a slide-in panel and a video shift), avoid duplicating the full interpolation for each property. Instead, create a single normalized progress value (0 to 1) and derive each property from it:

```tsx
const slideIn = interpolate(
  frame,
  [slideInStart, slideInStart + slideInDuration],
  [0, 1],
  {
    easing: Easing.bezier(0.22, 1, 0.36, 1),
    extrapolateLeft: "clamp",
    extrapolateRight: "clamp",
  },
);
const slideOut = interpolate(
  frame,
  [slideOutStart, slideOutStart + slideOutDuration],
  [0, 1],
  { easing: Easing.in(Easing.cubic), extrapolateLeft: "clamp", extrapolateRight: "clamp" },
);
const progress = slideIn - slideOut;

// Derive multiple properties from the same progress
const overlayX = interpolate(progress, [0, 1], [100, 0]);
const videoX = interpolate(progress, [0, 1], [0, -20]);
const opacity = interpolate(progress, [0, 1], [0, 1]);
```

The key idea: separate **timing** (when and how fast) from **mapping** (what values to animate between).
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/transcribe-captions.md -->
```markdown
---
name: transcribe-captions
description: Transcribing audio to generate captions in Remotion
metadata:
  tags: captions, transcribe, whisper, audio, speech-to-text
---

# Transcribing audio

To transcribe audio to generate captions in Remotion, you can use the [`transcribe()`](https://www.remotion.dev/docs/install-whisper-cpp/transcribe) function from the [`@remotion/install-whisper-cpp`](https://www.remotion.dev/docs/install-whisper-cpp) package.

## Prerequisites

First, the @remotion/install-whisper-cpp package needs to be installed.
If it is not installed, use the following command:

```bash
npx remotion add @remotion/install-whisper-cpp
```

## Transcribing

Make a Node.js script to download Whisper.cpp and a model, and transcribe the audio.

```ts
import path from "path";
import {
  downloadWhisperModel,
  installWhisperCpp,
  transcribe,
  toCaptions,
} from "@remotion/install-whisper-cpp";
import fs from "fs";

const to = path.join(process.cwd(), "whisper.cpp");

await installWhisperCpp({
  to,
  version: "1.5.5",
});

await downloadWhisperModel({
  model: "medium.en",
  folder: to,
});

// Convert the audio to a 16KHz wav file first if needed:
// import {execSync} from 'child_process';
// execSync('ffmpeg -i /path/to/audio.mp4 -ar 16000 /path/to/audio.wav -y');

const whisperCppOutput = await transcribe({
  model: "medium.en",
  whisperPath: to,
  whisperCppVersion: "1.5.5",
  inputPath: "/path/to/audio123.wav",
  tokenLevelTimestamps: true,
});

// Optional: Apply our recommended postprocessing
const { captions } = toCaptions({
  whisperCppOutput,
});

// Write it to the public/ folder so it can be fetched from Remotion
fs.writeFileSync("captions123.json", JSON.stringify(captions, null, 2));
```

Transcribe each clip individually and create multiple JSON files.

See [Displaying captions](display-captions.md) for how to display the captions in Remotion.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/transitions.md -->
```markdown
---
name: transitions
description: Scene transitions and overlays for Remotion using TransitionSeries.
metadata:
  tags: transitions, overlays, fade, slide, wipe, scenes
---

## TransitionSeries

`<TransitionSeries>` arranges scenes and supports two ways to enhance the cut point between them:

- **Transitions** (`<TransitionSeries.Transition>`) — crossfade, slide, wipe, etc. between two scenes. Shortens the timeline because both scenes play simultaneously during the transition.
- **Overlays** (`<TransitionSeries.Overlay>`) — render an effect (e.g. a light leak) on top of the cut point without shortening the timeline.

Children are absolutely positioned.

## Prerequisites

```bash
npx remotion add @remotion/transitions
```

## Transition example

```tsx
import { TransitionSeries, linearTiming } from "@remotion/transitions";
import { fade } from "@remotion/transitions/fade";

<TransitionSeries>
  <TransitionSeries.Sequence durationInFrames={60}>
    <SceneA />
  </TransitionSeries.Sequence>
  <TransitionSeries.Transition
    presentation={fade()}
    timing={linearTiming({ durationInFrames: 15 })}
  />
  <TransitionSeries.Sequence durationInFrames={60}>
    <SceneB />
  </TransitionSeries.Sequence>
</TransitionSeries>;
```

## Overlay example

Any React component can be used as an overlay. For a ready-made effect, see the **light-leaks** rule.

```tsx
import { TransitionSeries } from "@remotion/transitions";
import { LightLeak } from "@remotion/light-leaks";

<TransitionSeries>
  <TransitionSeries.Sequence durationInFrames={60}>
    <SceneA />
  </TransitionSeries.Sequence>
  <TransitionSeries.Overlay durationInFrames={20}>
    <LightLeak />
  </TransitionSeries.Overlay>
  <TransitionSeries.Sequence durationInFrames={60}>
    <SceneB />
  </TransitionSeries.Sequence>
</TransitionSeries>;
```

## Mixing transitions and overlays

Transitions and overlays can coexist in the same `<TransitionSeries>`, but an overlay cannot be adjacent to a transition or another overlay.

```tsx
import { TransitionSeries, linearTiming } from "@remotion/transitions";
import { fade } from "@remotion/transitions/fade";
import { LightLeak } from "@remotion/light-leaks";

<TransitionSeries>
  <TransitionSeries.Sequence durationInFrames={60}>
    <SceneA />
  </TransitionSeries.Sequence>
  <TransitionSeries.Overlay durationInFrames={30}>
    <LightLeak />
  </TransitionSeries.Overlay>
  <TransitionSeries.Sequence durationInFrames={60}>
    <SceneB />
  </TransitionSeries.Sequence>
  <TransitionSeries.Transition
    presentation={fade()}
    timing={linearTiming({ durationInFrames: 15 })}
  />
  <TransitionSeries.Sequence durationInFrames={60}>
    <SceneC />
  </TransitionSeries.Sequence>
</TransitionSeries>;
```

## Transition props

`<TransitionSeries.Transition>` requires:

- `presentation` — the visual effect (e.g. `fade()`, `slide()`, `wipe()`).
- `timing` — controls speed and easing (e.g. `linearTiming()`, `springTiming()`).

## Overlay props

`<TransitionSeries.Overlay>` accepts:

- `durationInFrames` — how long the overlay is visible (positive integer).
- `offset?` — shifts the overlay relative to the cut point center. Positive = later, negative = earlier. Default: `0`.

## Available transition types

Import transitions from their respective modules:

```tsx
import { fade } from "@remotion/transitions/fade";
import { slide } from "@remotion/transitions/slide";
import { wipe } from "@remotion/transitions/wipe";
import { flip } from "@remotion/transitions/flip";
import { clockWipe } from "@remotion/transitions/clock-wipe";
```

## Slide transition with direction

```tsx
import { slide } from "@remotion/transitions/slide";

<TransitionSeries.Transition
  presentation={slide({ direction: "from-left" })}
  timing={linearTiming({ durationInFrames: 20 })}
/>;
```

Directions: `"from-left"`, `"from-right"`, `"from-top"`, `"from-bottom"`

## Timing options

```tsx
import { linearTiming, springTiming } from "@remotion/transitions";

// Linear timing - constant speed
linearTiming({ durationInFrames: 20 });

// Spring timing - organic motion
springTiming({ config: { damping: 200 }, durationInFrames: 25 });
```

## Duration calculation

Transitions overlap adjacent scenes, so the total composition length is **shorter** than the sum of all sequence durations. Overlays do **not** affect the total duration.

For example, with two 60-frame sequences and a 15-frame transition:

- Without transitions: `60 + 60 = 120` frames
- With transition: `60 + 60 - 15 = 105` frames

Adding an overlay between two other sequences does not change the total.

### Getting the duration of a transition

Use the `getDurationInFrames()` method on the timing object:

```tsx
import { linearTiming, springTiming } from "@remotion/transitions";

const linearDuration = linearTiming({
  durationInFrames: 20,
}).getDurationInFrames({ fps: 30 });
// Returns 20

const springDuration = springTiming({
  config: { damping: 200 },
}).getDurationInFrames({ fps: 30 });
// Returns calculated duration based on spring physics
```

For `springTiming` without an explicit `durationInFrames`, the duration depends on `fps` because it calculates when the spring animation settles.

### Calculating total composition duration

```tsx
import { linearTiming } from "@remotion/transitions";

const scene1Duration = 60;
const scene2Duration = 60;
const scene3Duration = 60;

const timing1 = linearTiming({ durationInFrames: 15 });
const timing2 = linearTiming({ durationInFrames: 20 });

const transition1Duration = timing1.getDurationInFrames({ fps: 30 });
const transition2Duration = timing2.getDurationInFrames({ fps: 30 });

const totalDuration =
  scene1Duration +
  scene2Duration +
  scene3Duration -
  transition1Duration -
  transition2Duration;
// 60 + 60 + 60 - 15 - 20 = 145 frames
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/transparent-videos.md -->
```markdown
---
name: transparent-videos
description: Rendering transparent videos in Remotion
metadata:
  tags: transparent, alpha, codec, vp9, prores, webm
---

# Rendering Transparent Videos

Remotion can render transparent videos in two ways: as a ProRes video or as a WebM video.

## Transparent ProRes

Ideal for when importing into video editing software.

**CLI:**

```bash
npx remotion render --image-format=png --pixel-format=yuva444p10le --codec=prores --prores-profile=4444 MyComp out.mov
```

**Default in Studio** (restart Studio after changing):

```ts
// remotion.config.ts
import { Config } from "@remotion/cli/config";

Config.setVideoImageFormat("png");
Config.setPixelFormat("yuva444p10le");
Config.setCodec("prores");
Config.setProResProfile("4444");
```

**Setting it as the default export settings for a composition** (using `calculateMetadata`):

```tsx
import { CalculateMetadataFunction } from "remotion";

const calculateMetadata: CalculateMetadataFunction<Props> = async ({
  props,
}) => {
  return {
    defaultCodec: "prores",
    defaultVideoImageFormat: "png",
    defaultPixelFormat: "yuva444p10le",
    defaultProResProfile: "4444",
  };
};

<Composition
  id="my-video"
  component={MyVideo}
  durationInFrames={150}
  fps={30}
  width={1920}
  height={1080}
  calculateMetadata={calculateMetadata}
/>;
```

## Transparent WebM (VP9)

Ideal for when playing in a browser.

**CLI:**

```bash
npx remotion render --image-format=png --pixel-format=yuva420p --codec=vp9 MyComp out.webm
```

**Default in Studio** (restart Studio after changing):

```ts
// remotion.config.ts
import { Config } from "@remotion/cli/config";

Config.setVideoImageFormat("png");
Config.setPixelFormat("yuva420p");
Config.setCodec("vp9");
```

**Setting it as the default export settings for a composition** (using `calculateMetadata`):

```tsx
import { CalculateMetadataFunction } from "remotion";

const calculateMetadata: CalculateMetadataFunction<Props> = async ({
  props,
}) => {
  return {
    defaultCodec: "vp8",
    defaultVideoImageFormat: "png",
    defaultPixelFormat: "yuva420p",
  };
};

<Composition
  id="my-video"
  component={MyVideo}
  durationInFrames={150}
  fps={30}
  width={1920}
  height={1080}
  calculateMetadata={calculateMetadata}
/>;
```
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/trimming.md -->
```markdown
---
name: trimming
description: Trimming patterns for Remotion - cut the beginning or end of animations
metadata:
  tags: sequence, trim, clip, cut, offset
---

Use `<Sequence>` with a negative `from` value to trim the start of an animation.

## Trim the Beginning

A negative `from` value shifts time backwards, making the animation start partway through:

```tsx
import { Sequence, useVideoConfig } from "remotion";

const fps = useVideoConfig();

<Sequence from={-0.5 * fps}>
  <MyAnimation />
</Sequence>;
```

The animation appears 15 frames into its progress - the first 15 frames are trimmed off.
Inside `<MyAnimation>`, `useCurrentFrame()` starts at 15 instead of 0.

## Trim the End

Use `durationInFrames` to unmount content after a specified duration:

```tsx
<Sequence durationInFrames={1.5 * fps}>
  <MyAnimation />
</Sequence>
```

The animation plays for 45 frames, then the component unmounts.

## Trim and Delay

Nest sequences to both trim the beginning and delay when it appears:

```tsx
<Sequence from={30}>
  <Sequence from={-15}>
    <MyAnimation />
  </Sequence>
</Sequence>
```

The inner sequence trims 15 frames from the start, and the outer sequence delays the result by 30 frames.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/videos.md -->
```markdown
---
name: videos
description: Embedding videos in Remotion - trimming, volume, speed, looping, pitch
metadata:
  tags: video, media, trim, volume, speed, loop, pitch
---

# Using videos in Remotion

## Prerequisites

First, the @remotion/media package needs to be installed.  
If it is not, use the following command:

```bash
npx remotion add @remotion/media # If project uses npm
bunx remotion add @remotion/media # If project uses bun
yarn remotion add @remotion/media # If project uses yarn
pnpm exec remotion add @remotion/media # If project uses pnpm
```

Use `<Video>` from `@remotion/media` to embed videos into your composition.

```tsx
import { Video } from "@remotion/media";
import { staticFile } from "remotion";

export const MyComposition = () => {
  return <Video src={staticFile("video.mp4")} />;
};
```

Remote URLs are also supported:

```tsx
<Video src="https://remotion.media/video.mp4" />
```

## Trimming

Use `trimBefore` and `trimAfter` to remove portions of the video. Values are in seconds.

```tsx
const { fps } = useVideoConfig();

return (
  <Video
    src={staticFile("video.mp4")}
    trimBefore={2 * fps} // Skip the first 2 seconds
    trimAfter={10 * fps} // End at the 10 second mark
  />
);
```

## Delaying

Wrap the video in a `<Sequence>` to delay when it appears:

```tsx
import { Sequence, staticFile } from "remotion";
import { Video } from "@remotion/media";

const { fps } = useVideoConfig();

return (
  <Sequence from={1 * fps}>
    <Video src={staticFile("video.mp4")} />
  </Sequence>
);
```

The video will appear after 1 second.

## Sizing and Position

Use the `style` prop to control size and position:

```tsx
<Video
  src={staticFile("video.mp4")}
  style={{
    width: 500,
    height: 300,
    position: "absolute",
    top: 100,
    left: 50,
    objectFit: "cover",
  }}
/>
```

## Volume

Set a static volume (0 to 1):

```tsx
<Video src={staticFile("video.mp4")} volume={0.5} />
```

Or use a callback for dynamic volume based on the current frame:

```tsx
import { interpolate } from "remotion";

const { fps } = useVideoConfig();

return (
  <Video
    src={staticFile("video.mp4")}
    volume={(f) =>
      interpolate(f, [0, 1 * fps], [0, 1], { extrapolateRight: "clamp" })
    }
  />
);
```

Use `muted` to silence the video entirely:

```tsx
<Video src={staticFile("video.mp4")} muted />
```

## Speed

Use `playbackRate` to change the playback speed:

```tsx
<Video src={staticFile("video.mp4")} playbackRate={2} /> {/* 2x speed */}
<Video src={staticFile("video.mp4")} playbackRate={0.5} /> {/* Half speed */}
```

Reverse playback is not supported.

## Looping

Use `loop` to loop the video indefinitely:

```tsx
<Video src={staticFile("video.mp4")} loop />
```

Use `loopVolumeCurveBehavior` to control how the frame count behaves when looping:

- `"repeat"`: Frame count resets to 0 each loop (for `volume` callback)
- `"extend"`: Frame count continues incrementing

```tsx
<Video
  src={staticFile("video.mp4")}
  loop
  loopVolumeCurveBehavior="extend"
  volume={(f) => interpolate(f, [0, 300], [1, 0])} // Fade out over multiple loops
/>
```

## Pitch

Use `toneFrequency` to adjust the pitch without affecting speed. Values range from 0.01 to 2:

```tsx
<Video
  src={staticFile("video.mp4")}
  toneFrequency={1.5} // Higher pitch
/>
<Video
  src={staticFile("video.mp4")}
  toneFrequency={0.8} // Lower pitch
/>
```

Pitch shifting only works during server-side rendering, not in the Remotion Studio preview or in the `<Player />`.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/remotion-skills/rules/voiceover.md -->
```markdown
---
name: voiceover
description: Adding AI-generated voiceover to Remotion compositions using TTS
metadata:
  tags: voiceover, audio, elevenlabs, tts, speech, calculateMetadata, dynamic duration
---

# Adding AI voiceover to a Remotion composition

Use ElevenLabs TTS to generate speech audio per scene, then use [`calculateMetadata`](./calculate-metadata) to dynamically size the composition to match the audio.

## Prerequisites

By default this guide uses **ElevenLabs** as the TTS provider (`ELEVENLABS_API_KEY` environment variable). Users may substitute any TTS service that can produce an audio file.

If the user has not specified a TTS provider, recommend ElevenLabs and ask for their API key.

Ensure the environment variable is available when running the generation script:

```bash
node --strip-types generate-voiceover.ts
```

## Generating audio with ElevenLabs

Create a script that reads the config, calls the ElevenLabs API for each scene, and writes MP3 files to the `public/` directory so Remotion can access them via `staticFile()`.

The core API call for a single scene:

```ts title="generate-voiceover.ts"
const response = await fetch(
  `https://api.elevenlabs.io/v1/text-to-speech/${voiceId}`,
  {
    method: "POST",
    headers: {
      "xi-api-key": process.env.ELEVENLABS_API_KEY!,
      "Content-Type": "application/json",
      Accept: "audio/mpeg",
    },
    body: JSON.stringify({
      text: "Welcome to the show.",
      model_id: "eleven_multilingual_v2",
      voice_settings: {
        stability: 0.5,
        similarity_boost: 0.75,
        style: 0.3,
      },
    }),
  },
);

const audioBuffer = Buffer.from(await response.arrayBuffer());
writeFileSync(`public/voiceover/${compositionId}/${scene.id}.mp3`, audioBuffer);
```

## Dynamic composition duration with calculateMetadata

Use [`calculateMetadata`](./calculate-metadata.md) to measure the [audio durations](./get-audio-duration.md) and set the composition length accordingly.

```tsx
import { CalculateMetadataFunction, staticFile } from "remotion";
import { getAudioDuration } from "./get-audio-duration";

const FPS = 30;

const SCENE_AUDIO_FILES = [
  "voiceover/my-comp/scene-01-intro.mp3",
  "voiceover/my-comp/scene-02-main.mp3",
  "voiceover/my-comp/scene-03-outro.mp3",
];

export const calculateMetadata: CalculateMetadataFunction<Props> = async ({
  props,
}) => {
  const durations = await Promise.all(
    SCENE_AUDIO_FILES.map((file) => getAudioDuration(staticFile(file))),
  );

  const sceneDurations = durations.map((durationInSeconds) => {
    return durationInSeconds * FPS;
  });

  return {
    durationInFrames: Math.ceil(sceneDurations.reduce((sum, d) => sum + d, 0)),
  };
};
```

The computed `sceneDurations` are passed into the component via a `voiceover` prop so the component knows how long each scene should be.

If the composition uses [`<TransitionSeries>`](./transitions.md), subtract the overlap from total duration: [./transitions.md#calculating-total-composition-duration](./transitions.md#calculating-total-composition-duration)

## Rendering audio in the component

See [audio.md](./audio.md) for more information on how to render audio in the component.

## Delaying audio start

See [audio.md#delaying](./audio.md#delaying) for more information on how to delay the audio start.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/templates/deck/milecell-example.html -->
```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>MileCell Digital Commerce Platform &amp; Growth Roadmap</title>
<style>
  /* ============================================================
     BRAND TOKENS
     ============================================================ */
  :root{
    --navy:#1E3A5F;
    --navy-deep:#16304e;
    --navy-line:#2c4d75;
    --teal:#1CA9A6;
    --teal-bright:#2BB3A3;
    --teal-soft:#e7f6f5;
    --ink:#333333;
    --ink-soft:#5a6675;
    --paper:#ffffff;
    --paper-soft:#f6f9fb;
    --red:#b9534b;          /* single muted warning red */
    --red-soft:#fbeeec;
    --border:#dfe6ee;
    --shadow:0 10px 30px rgba(30,58,95,.10);
  }

  *{box-sizing:border-box;margin:0;padding:0;}
  html,body{background:#0b1622;-webkit-font-smoothing:antialiased;text-rendering:optimizeLegibility;}
  body{
    font-family:"Inter",-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,Helvetica,Arial,sans-serif;
    color:var(--ink);
    font-size:18px;
    line-height:1.5;
  }

  /* ============================================================
     SLIDE FRAME — 1280 x 720 logical (16:9)
     ============================================================ */
  .deck{display:flex;flex-direction:column;align-items:center;gap:26px;padding:26px 0;}
  .slide{
    position:relative;
    width:1280px;
    height:720px;
    background:var(--paper);
    overflow:hidden;
    box-shadow:var(--shadow);
  }

  /* ---- CONTENT SLIDE: white body + thin navy header band ---- */
  .slide.content{
    background:var(--paper) url("../02-content-bg.png") center/cover no-repeat;
  }
  .hdr{
    position:relative;
    height:84px;
    background:var(--navy);
    display:flex;align-items:center;
    padding:0 56px;
    color:#fff;
    border-bottom:3px solid var(--teal);
    z-index:3;
  }
  .hdr .kicker{
    font-size:13px;letter-spacing:.16em;text-transform:uppercase;
    color:var(--teal-bright);font-weight:600;margin-bottom:2px;
  }
  .hdr h2{font-size:27px;font-weight:700;line-height:1.12;letter-spacing:-.01em;}
  .hdr .pageno{
    position:absolute;right:56px;top:50%;transform:translateY(-50%);
    font-size:13px;color:#9db4cf;font-weight:500;letter-spacing:.04em;
  }
  .hdr .brandmark{
    position:absolute;right:120px;top:50%;transform:translateY(-50%);
    font-size:13px;font-weight:700;letter-spacing:.18em;color:#cfe0f0;
  }
  .body{
    position:relative;z-index:2;
    padding:34px 56px 48px;
    height:calc(720px - 84px);
  }
  .lead{font-size:21px;color:var(--navy);font-weight:600;line-height:1.32;margin-bottom:20px;letter-spacing:-.005em;}
  .lead .accent{color:var(--teal);}

  /* generic helpers */
  .muted{color:var(--ink-soft);}
  .teal{color:var(--teal);}
  .navytxt{color:var(--navy);}
  .b{font-weight:700;}
  .sm{font-size:15px;}
  .xs{font-size:13px;}

  /* bullet list */
  ul.bul{list-style:none;display:flex;flex-direction:column;gap:13px;margin-top:6px;}
  ul.bul li{position:relative;padding-left:30px;font-size:18px;line-height:1.42;color:var(--ink);}
  ul.bul li::before{
    content:"";position:absolute;left:4px;top:9px;width:11px;height:11px;
    background:var(--teal);
    clip-path:polygon(25% 0,75% 0,100% 50%,75% 100%,25% 100%,0 50%);
  }
  ul.bul li .b{color:var(--navy);}

  /* ============================================================
     TABLES
     ============================================================ */
  table.tbl{border-collapse:collapse;width:100%;font-size:15px;background:#fff;box-shadow:var(--shadow);border-radius:8px;overflow:hidden;}
  table.tbl thead th{
    background:var(--navy);color:#fff;font-weight:600;text-align:left;
    padding:12px 14px;font-size:13.5px;letter-spacing:.02em;border-right:1px solid var(--navy-line);
  }
  table.tbl thead th:last-child{border-right:none;}
  table.tbl tbody td{padding:11px 14px;border-bottom:1px solid var(--border);border-right:1px solid #eef2f6;color:var(--ink);vertical-align:middle;}
  table.tbl tbody td:last-child{border-right:none;}
  table.tbl tbody tr:nth-child(even){background:var(--paper-soft);}
  table.tbl tbody tr:last-child td{border-bottom:none;}
  table.tbl td.lbl{font-weight:600;color:var(--navy);}
  table.tbl .yes{color:var(--teal);font-weight:700;}
  table.tbl .no{color:var(--red);font-weight:700;}
  table.tbl .mid{color:#b8852f;font-weight:700;}
  /* pricing total row */
  table.tbl tr.total td{background:var(--navy)!important;color:#fff;font-weight:700;font-size:16px;border-bottom:none;}
  table.tbl tr.total td .teal{color:var(--teal-bright);}
  table.tbl td.num{text-align:right;font-variant-numeric:tabular-nums;white-space:nowrap;}
  table.tbl th.num{text-align:right;}
  /* risk pill */
  .pill{display:inline-block;padding:2px 10px;border-radius:11px;font-size:12px;font-weight:700;letter-spacing:.02em;}
  .pill.low{background:var(--teal-soft);color:#0d7a77;}
  .pill.med{background:#fdf3e0;color:#9a6a16;}
  .pill.high{background:var(--red-soft);color:var(--red);}

  /* ============================================================
     CARDS
     ============================================================ */
  .cards{display:grid;gap:24px;}
  .cards.two{grid-template-columns:1fr 1fr;}
  .cards.three{grid-template-columns:1fr 1fr 1fr;}
  .card{
    background:#fff;border:1px solid var(--border);border-radius:12px;
    padding:24px 24px 22px;box-shadow:var(--shadow);position:relative;overflow:hidden;
  }
  .card .tag{font-size:12px;letter-spacing:.14em;text-transform:uppercase;font-weight:700;color:var(--teal);margin-bottom:4px;}
  .card h3{font-size:20px;color:var(--navy);font-weight:700;margin-bottom:10px;line-height:1.18;}
  .card.featured{border-color:var(--teal);border-width:2px;}
  .card.featured::after{content:"RECOMMENDED";position:absolute;top:14px;right:-34px;transform:rotate(45deg);background:var(--teal);color:#fff;font-size:10px;font-weight:700;letter-spacing:.12em;padding:4px 40px;}
  .card p{font-size:15.5px;line-height:1.46;color:var(--ink);}

  /* footer differentiator strip */
  .diffstrip{display:grid;grid-template-columns:repeat(4,1fr);gap:14px;margin-top:auto;}
  .diffstrip .d{background:var(--navy);color:#fff;border-radius:9px;padding:14px 16px;font-size:13.5px;line-height:1.32;font-weight:500;border-left:3px solid var(--teal);}
  .diffstrip .d .b{color:var(--teal-bright);display:block;font-size:12px;letter-spacing:.06em;text-transform:uppercase;margin-bottom:3px;}

  /* ============================================================
     CALLOUTS
     ============================================================ */
  .callout{border-radius:10px;padding:14px 18px;font-size:15.5px;line-height:1.4;}
  .callout.rec{background:var(--teal-soft);border-left:4px solid var(--teal);color:#0c5e5b;}
  .callout.rec .b{color:var(--navy);}
  .callout.warn{background:var(--red-soft);border-left:4px solid var(--red);color:#7d3a34;}
  .callout.warn .b{color:var(--red);}
  .callout.stat{background:var(--navy);color:#fff;border-left:4px solid var(--teal);}
  .callout.stat .big{color:var(--teal-bright);font-weight:700;}
  .callout.note{background:var(--paper-soft);border-left:4px solid var(--navy-line);color:var(--ink-soft);font-size:14px;font-style:italic;}

  /* ============================================================
     FULL-NAVY SLIDES (cover / divider / close) w/ gpt-image-2 bg
     ============================================================ */
  .slide.cover{
    background:var(--navy) url("../01-title-bg.png") center/cover no-repeat;
    color:#fff;display:flex;flex-direction:column;justify-content:center;
    padding:0 96px;
  }
  .slide.divider{
    background:var(--navy) url("../03-divider-bg.png") center/cover no-repeat;
    color:#fff;display:flex;flex-direction:column;justify-content:center;
    padding:0 110px;
  }
  .slide.close{
    background:var(--navy) url("../04-closing-bg.png") center/cover no-repeat;
    color:#fff;display:flex;flex-direction:column;justify-content:center;
    padding:0 96px;
  }
  /* subtle scrim so live HTML text stays legible over the art */
  .slide.cover::before,.slide.close::before{
    content:"";position:absolute;inset:0;
    background:linear-gradient(90deg,rgba(20,40,66,.86) 0%,rgba(20,40,66,.55) 50%,rgba(20,40,66,.18) 100%);
    z-index:1;
  }
  .slide.divider::before{
    content:"";position:absolute;inset:0;
    background:linear-gradient(180deg,rgba(20,40,66,.34) 0%,rgba(20,40,66,.62) 100%);
    z-index:1;
  }
  .cover-inner,.divider-inner,.close-inner{position:relative;z-index:2;}

  .cover .eyebrow{font-size:14px;letter-spacing:.28em;text-transform:uppercase;color:var(--teal-bright);font-weight:600;margin-bottom:22px;}
  .cover h1{font-size:56px;line-height:1.06;font-weight:800;letter-spacing:-.02em;max-width:880px;}
  .cover h1 .teal{color:var(--teal-bright);}
  .cover .sub{font-size:23px;color:#d7e4f1;margin-top:24px;max-width:760px;line-height:1.38;font-weight:400;}
  .cover .foot{position:absolute;left:96px;bottom:54px;font-size:15px;color:#a9c0d8;letter-spacing:.03em;z-index:2;}
  .cover .rule{width:96px;height:4px;background:var(--teal);margin:26px 0 0;border-radius:2px;}

  .divider .dnum{font-size:15px;letter-spacing:.3em;color:var(--teal-bright);font-weight:700;margin-bottom:18px;text-transform:uppercase;}
  .divider h1{font-size:48px;line-height:1.1;font-weight:800;letter-spacing:-.015em;max-width:920px;}
  .divider .dsub{font-size:21px;color:#cfe0f0;margin-top:20px;max-width:820px;line-height:1.4;font-weight:400;}
  .divider .rule{width:90px;height:4px;background:var(--teal);margin-top:24px;border-radius:2px;}

  .close h1{font-size:50px;line-height:1.08;font-weight:800;letter-spacing:-.02em;max-width:840px;}
  .close h1 .teal{color:var(--teal-bright);}
  .close .reasons{display:grid;grid-template-columns:1fr 1fr;gap:14px 30px;margin-top:30px;max-width:840px;}
  .close .reasons .r{position:relative;padding-left:30px;font-size:17px;color:#e2ecf6;line-height:1.34;}
  .close .reasons .r::before{content:"";position:absolute;left:0;top:6px;width:13px;height:13px;background:var(--teal);clip-path:polygon(25% 0,75% 0,100% 50%,75% 100%,25% 100%,0 50%);}
  .close .nextstep{margin-top:30px;font-size:19px;color:#fff;max-width:820px;line-height:1.4;}
  .close .nextstep .b{color:var(--teal-bright);}
  .close .sign{margin-top:24px;font-size:17px;color:#bcd0e6;font-style:italic;}

  /* number stat blocks */
  .statgrid{display:grid;gap:18px;}
  .statgrid.three{grid-template-columns:1fr 1fr 1fr;}
  .statgrid.four{grid-template-columns:repeat(4,1fr);}
  .stat{background:#fff;border:1px solid var(--border);border-top:4px solid var(--teal);border-radius:10px;padding:18px 18px 16px;box-shadow:var(--shadow);}
  .stat .num{font-size:34px;font-weight:800;color:var(--navy);line-height:1;letter-spacing:-.02em;}
  .stat .num .u{font-size:18px;color:var(--teal);font-weight:700;}
  .stat .lab{font-size:13.5px;color:var(--ink-soft);margin-top:8px;line-height:1.32;}

  /* ladder (slide 5) */
  .ladder{display:flex;flex-direction:column;gap:8px;}
  .ladder .step{display:flex;align-items:center;gap:14px;background:#fff;border:1px solid var(--border);border-radius:8px;padding:10px 16px;box-shadow:0 4px 12px rgba(30,58,95,.06);}
  .ladder .step .n{flex:none;width:30px;height:30px;border-radius:50%;background:var(--paper-soft);color:var(--ink-soft);font-weight:700;display:flex;align-items:center;justify-content:center;font-size:14px;border:1px solid var(--border);}
  .ladder .step .t{font-size:15px;color:var(--ink);line-height:1.32;}
  .ladder .step.top{background:var(--navy);border-color:var(--navy);}
  .ladder .step.top .n{background:var(--teal);color:#fff;border-color:var(--teal);}
  .ladder .step.top .t{color:#fff;font-weight:600;}
  .ladder .step.top .t .b{color:var(--teal-bright);}

  /* journey chips (slide 15) */
  .journey{display:flex;align-items:center;gap:0;flex-wrap:wrap;margin:6px 0 18px;}
  .journey .j{background:var(--navy);color:#fff;font-size:14px;font-weight:600;padding:9px 16px;border-radius:6px;position:relative;}
  .journey .arr{color:var(--teal);font-weight:700;padding:0 8px;font-size:16px;}

  /* feature menu tiers (slide 21) */
  .tiers{display:grid;grid-template-columns:1fr 1fr 1fr;gap:18px;}
  .tier{background:#fff;border:1px solid var(--border);border-radius:11px;overflow:hidden;box-shadow:var(--shadow);}
  .tier .th{background:var(--navy);color:#fff;padding:11px 16px;font-weight:700;font-size:15px;}
  .tier .th .s{display:block;font-size:11.5px;color:var(--teal-bright);font-weight:600;letter-spacing:.04em;margin-top:2px;}
  .tier ul{list-style:none;padding:14px 16px;display:flex;flex-direction:column;gap:9px;}
  .tier ul li{font-size:13.5px;line-height:1.32;color:var(--ink);position:relative;padding-left:18px;}
  .tier ul li::before{content:"";position:absolute;left:0;top:6px;width:8px;height:8px;background:var(--teal);clip-path:polygon(25% 0,75% 0,100% 50%,75% 100%,25% 100%,0 50%);}

  /* roi chips (slide 24) */
  .chips{display:flex;flex-wrap:wrap;gap:10px;margin-top:6px;}
  .chips .c{background:var(--teal-soft);color:#0c5e5b;border:1px solid #b8e2e0;border-radius:20px;padding:7px 16px;font-size:14px;font-weight:600;}

  /* two-col generic */
  .cols2{display:grid;grid-template-columns:1fr 1fr;gap:30px;align-items:start;}
  .cols2.wide-left{grid-template-columns:1.15fr .85fr;}

  /* mockup frame (slide 13) */
  .mock{background:#fff;border:1px solid var(--border);border-radius:12px;box-shadow:var(--shadow);overflow:hidden;}
  .mock .bar{background:var(--navy);height:42px;display:flex;align-items:center;padding:0 16px;gap:7px;}
  .mock .bar .dot{width:10px;height:10px;border-radius:50%;background:#3a5a82;}
  .mock .bar .ttl{color:#cfe0f0;font-size:12px;margin-left:10px;letter-spacing:.04em;}
  .mock .mbody{padding:18px 20px;}
  .mock .hero{background:linear-gradient(120deg,var(--navy),var(--teal));border-radius:8px;color:#fff;padding:18px;}
  .mock .hero .h{font-size:18px;font-weight:700;}
  .mock .hero .badges{margin-top:10px;display:flex;gap:8px;}
  .mock .hero .badge{background:rgba(255,255,255,.18);border-radius:5px;font-size:11px;padding:4px 9px;}
  .mock .specrow{display:flex;justify-content:space-between;font-size:13px;padding:7px 0;border-bottom:1px solid var(--border);color:var(--ink);}
  .mock .specrow .k{color:var(--ink-soft);}
  .mock .ctas{display:flex;gap:10px;margin-top:14px;}
  .mock .cta{flex:1;text-align:center;border-radius:7px;font-size:13px;font-weight:700;padding:9px;}
  .mock .cta.p{background:var(--teal);color:#fff;}
  .mock .cta.s{background:#fff;color:var(--navy);border:1.5px solid var(--navy);}

  /* small footnote */
  .src{font-size:12px;color:var(--ink-soft);}

  /* divider corner hexagon motif (extra, for content slides w/o image dependency) */
  .hexmotif{position:absolute;right:-40px;bottom:-40px;width:220px;height:220px;opacity:.06;z-index:0;pointer-events:none;}
</style>
</head>
<body>
<div class="deck">

<!-- ============================================================
     SLIDE 1 — COVER (gpt-image-2 background: 01-title-bg.png)
     ============================================================ -->
<section class="slide cover" id="slide-01">
  <div class="cover-inner">
    <div class="eyebrow">MileCell Biotechnology</div>
    <h1>MileCell Digital Commerce Platform <span class="teal">&amp; Growth Roadmap</span></h1>
    <div class="rule"></div>
    <div class="sub">From a custom-coded site to a self-manageable, CRM-integrated revenue engine.</div>
  </div>
  <div class="foot">Prepared for MileCell Biotechnology &nbsp;|&nbsp; June 2026</div>
</section>

<!-- ============================================================
     SLIDE 2 — Two Ways to Win
     ============================================================ -->
<section class="slide content" id="slide-02">
  <div class="hdr">
    <div><div class="kicker">Frame</div><h2>Two Ways to Win</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">02 / 26</div>
  </div>
  <div class="body" style="display:flex;flex-direction:column;">
    <div class="cards two" style="margin-bottom:22px;">
      <div class="card">
        <div class="tag">Tier 1 — The Platform</div>
        <h3>Everything in your RFQ</h3>
        <p>A modular, self-manageable, CRM-integrated B2B/B2C commerce site. <span class="b navytxt">Fixed scope. No upfront payment — pay on acceptance.</span></p>
      </div>
      <div class="card featured">
        <div class="tag">Tier 2 — The Growth Engine</div>
        <h3>Optional, recommended</h3>
        <p>A milestone-driven monthly build + maintenance that turns the platform into <span class="b navytxt">measurable pipeline</span> — not just a site that sits there.</p>
      </div>
    </div>
    <div class="diffstrip">
      <div class="d"><span class="b">Proven stack</span>Built this exact stack for a reagent company</div>
      <div class="d"><span class="b">Your science</span>We speak hepatocytes, PBMCs, ADME</div>
      <div class="d"><span class="b">HubSpot</span>HubSpot + automation, daily</div>
      <div class="d"><span class="b">Operators</span>Operators with your stakes</div>
    </div>
  </div>
</section>

<!-- ============================================================
     SLIDE 3 — DIVIDER (gpt-image-2 background: 03-divider-bg.png)
     ============================================================ -->
<section class="slide divider" id="slide-03">
  <div class="divider-inner">
    <div class="dnum">Section 1</div>
    <h1>Decisions to Lock First</h1>
    <div class="dsub">The few choices that are expensive to reverse later.</div>
    <div class="rule"></div>
  </div>
</section>

<!-- ============================================================
     SLIDE 4 — Platform decision (comparison table)
     ============================================================ -->
<section class="slide content" id="slide-04">
  <div class="hdr">
    <div><div class="kicker">Decisions to Lock First</div><h2>A CMS Choice, Not a Coding Choice</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">04 / 26</div>
  </div>
  <div class="body">
    <div class="lead">You're choosing a <span class="accent">platform you control</span> — with a deep developer ecosystem, never locked to one vendor.</div>
    <table class="tbl" style="margin-bottom:16px;">
      <thead><tr>
        <th>Platform</th><th>Open ecosystem</th><th>Native B2B (PO / quote / account pricing)</th><th>API quality</th><th>Self-manageable CMS</th>
      </tr></thead>
      <tbody>
        <tr><td class="lbl">Adobe Commerce (Magento)</td><td class="yes">Yes</td><td class="yes">Yes</td><td class="yes">High</td><td class="mid">Moderate</td></tr>
        <tr><td class="lbl">BigCommerce B2B</td><td class="yes">Yes</td><td class="yes">Yes</td><td class="yes">High</td><td class="yes">Strong</td></tr>
        <tr><td class="lbl">Shopify Plus B2B</td><td class="mid">Partial</td><td class="mid">Partial</td><td class="yes">High</td><td class="yes">Strong</td></tr>
        <tr><td class="lbl">Webflow + Commerce</td><td class="no">Limited</td><td class="no">No</td><td class="mid">Moderate</td><td class="yes">Strong</td></tr>
      </tbody>
    </table>
    <div class="cols2" style="gap:18px;">
      <div class="callout rec"><span class="b">Recommendation:</span> an open, B2B-native, strong-API platform. We've already tamed this class for life sciences — we run Boster on Magento and know COA / lot / PO / quote patterns.</div>
      <div class="callout warn"><span class="b">Warning:</span> pick wrong here and you migrate again in 18 months.</div>
    </div>
  </div>
</section>

<!-- ============================================================
     SLIDE 5 — CRM + transaction gap (ladder)
     ============================================================ -->
<section class="slide content" id="slide-05">
  <div class="hdr">
    <div><div class="kicker">Decisions to Lock First</div><h2>The CRM &amp; Transaction Gap</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">05 / 26</div>
  </div>
  <div class="body">
    <div class="lead">You already chose HubSpot — <span class="accent">good call</span>, and we work in it daily.</div>
    <div class="cols2 wide-left" style="gap:30px;">
      <div class="ladder">
        <div class="step"><div class="n">1</div><div class="t">Better than a spreadsheet</div></div>
        <div class="step"><div class="n">2</div><div class="t">We see each customer's history — but hate manual entry</div></div>
        <div class="step"><div class="n">3</div><div class="t">We work around the gaps with Excel exports</div></div>
        <div class="step"><div class="n">4</div><div class="t">The vendor quotes us for custom dev — never again</div></div>
        <div class="step top"><div class="n">5</div><div class="t"><span class="b">Custom automation + system integrations:</span> store, CRM and shipping work together automatically — tested and maintained.</div></div>
      </div>
      <div style="display:flex;flex-direction:column;gap:16px;">
        <div class="callout warn"><span class="b">Open question:</span> what handles transactions &amp; shipping / fulfillment? HubSpot is records + marketing — not an order / inventory system.</div>
        <div class="callout stat"><span class="big">+129% leads, +36% more deals</span> in year one — HubSpot customers (HubSpot).</div>
      </div>
    </div>
  </div>
</section>

<!-- ============================================================
     SLIDE 6 — Hosting & ownership
     ============================================================ -->
<section class="slide content" id="slide-06">
  <div class="hdr">
    <div><div class="kicker">Decisions to Lock First</div><h2>Hosting &amp; Ownership</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">06 / 26</div>
  </div>
  <div class="body">
    <div class="lead">US-hosted — your requirement. Every account in <span class="accent">YOUR</span> name. We set up; you hold the keys.</div>
    <div class="cols2" style="gap:30px;">
      <ul class="bul">
        <li><span class="b">In your name:</span> hosting, payment gateway, email, analytics</li>
        <li><span class="b">Recommended stack:</span> platform host + CDN + HubSpot + Stripe + email service</li>
        <li>We configure and hand over — no hidden ownership, no lock-in</li>
        <li>US data residency satisfied by design</li>
      </ul>
      <div class="callout rec" style="align-self:center;font-size:18px;line-height:1.45;">
        <span class="b">"You could replace us tomorrow and everything keeps running."</span>
      </div>
    </div>
  </div>
</section>

<!-- ============================================================
     SLIDE 7 — Product data model
     ============================================================ -->
<section class="slide content" id="slide-07">
  <div class="hdr">
    <div><div class="kicker">Decisions to Lock First</div><h2>Product Data Model</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">07 / 26</div>
  </div>
  <div class="body">
    <div class="lead">Set the catalog structure <span class="accent">once</span> — get it right and everything downstream just works.</div>
    <div class="cols2" style="gap:30px;">
      <ul class="bul">
        <li><span class="b">Structure:</span> Species × Cell type × Application × Format</li>
        <li><span class="b">Plus fields:</span> COA-by-lot, datasheets, manuals</li>
        <li>Right once → filtering, citations, search, bulk upload, future ERP sync all just work</li>
        <li>We bring a life-science catalog template</li>
      </ul>
      <div style="display:flex;flex-direction:column;gap:14px;">
        <div class="callout rec"><span class="b">Get it right once</span> → filtering · citations · search · bulk upload · ERP sync.</div>
        <div class="callout warn"><span class="b">Get it wrong</span> → re-tag the whole catalog later.</div>
      </div>
    </div>
  </div>
</section>

<!-- ============================================================
     SLIDE 8 — DIVIDER (gpt-image-2 background: 03-divider-bg.png)
     ============================================================ -->
<section class="slide divider" id="slide-08">
  <div class="divider-inner">
    <div class="dnum">Section 2</div>
    <h1>Everything Else in Your RFQ — Covered</h1>
    <div class="dsub">Well-understood, low-risk modules — here's the honest coverage.</div>
    <div class="rule"></div>
  </div>
</section>

<!-- ============================================================
     SLIDE 9 — RFQ coverage table
     ============================================================ -->
<section class="slide content" id="slide-09">
  <div class="hdr">
    <div><div class="kicker">Everything Else in the RFQ</div><h2>RFQ Coverage — The Honest Read</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">09 / 26</div>
  </div>
  <div class="body" style="padding-top:24px;">
    <table class="tbl">
      <thead><tr><th style="width:34%;">Item</th><th style="width:13%;">Risk</th><th style="width:16%;">Complexity</th><th>Comment</th></tr></thead>
      <tbody>
        <tr><td class="lbl">Modular CMS + templates</td><td><span class="pill low">Low</span></td><td><span class="pill med">Med</span></td><td>Native</td></tr>
        <tr><td class="lbl">Catalog + filtering</td><td><span class="pill low">Low</span></td><td><span class="pill med">Med</span></td><td>&nbsp;</td></tr>
        <tr><td class="lbl">Scientific citations / Bioz</td><td><span class="pill med">Med</span></td><td><span class="pill med">Med</span></td><td>&nbsp;</td></tr>
        <tr><td class="lbl">B2B e-commerce quote-to-order</td><td><span class="pill med">Med</span></td><td><span class="pill high">High</span></td><td>The core</td></tr>
        <tr><td class="lbl">Payments — card / PO / wire / check</td><td><span class="pill med">Med</span></td><td><span class="pill high">High</span></td><td>&nbsp;</td></tr>
        <tr><td class="lbl">Customer portal</td><td><span class="pill low">Low</span></td><td><span class="pill med">Med</span></td><td>&nbsp;</td></tr>
        <tr><td class="lbl">HubSpot CRM</td><td><span class="pill low">Low</span></td><td><span class="pill med">Med</span></td><td>&nbsp;</td></tr>
        <tr><td class="lbl">Marketing automation</td><td><span class="pill low">Low</span></td><td><span class="pill low">Low</span></td><td>&nbsp;</td></tr>
        <tr><td class="lbl">Resource hub + COA-by-lot search</td><td><span class="pill med">Med</span></td><td><span class="pill med">Med</span></td><td>&nbsp;</td></tr>
        <tr><td class="lbl">SEO / analytics / speed</td><td><span class="pill low">Low</span></td><td><span class="pill med">Med</span></td><td>&nbsp;</td></tr>
      </tbody>
    </table>
  </div>
</section>

<!-- ============================================================
     SLIDE 10 — Who you're working with
     ============================================================ -->
<section class="slide content" id="slide-10">
  <div class="hdr">
    <div><div class="kicker">Who We Are / Proof</div><h2>Who You're Working With</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">10 / 26</div>
  </div>
  <div class="body">
    <div class="lead">Not a vendor at arm's length — an <span class="accent">operator with the same stakes</span>.</div>
    <div class="cards three">
      <div class="card">
        <div class="tag">Same stakes</div>
        <h3>A marketing leader &amp; business owner</h3>
        <p>We carry the same P&amp;L pressure you do — decisions are made like owners, not contractors.</p>
      </div>
      <div class="card">
        <div class="tag">Build &amp; operate</div>
        <h3>We run the lead engine</h3>
        <p>We build AND operate it — not just hand over a pretty site and walk away.</p>
      </div>
      <div class="card">
        <div class="tag">Your science</div>
        <h3>We speak it fluently</h3>
        <p>Hepatocytes, PBMCs, ADME / DMPK, cell &amp; gene therapy.</p>
      </div>
    </div>
  </div>
</section>

<!-- ============================================================
     SLIDE 11 — Your RFQ told us you mean business
     ============================================================ -->
<section class="slide content" id="slide-11">
  <div class="hdr">
    <div><div class="kicker">Who We Are / Proof</div><h2>Your RFQ Told Us You Mean Business</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">11 / 26</div>
  </div>
  <div class="body">
    <div class="lead">You scoped the things only people who've <span class="accent">hit these walls</span> know to scope.</div>
    <div class="cols2" style="gap:30px;">
      <ul class="bul">
        <li>PO / offline-approval payments</li>
        <li>Scientific citations</li>
        <li>Modular-over-custom</li>
        <li>Full CRM lifecycle</li>
        <li>COA-by-lot</li>
      </ul>
      <div style="display:flex;flex-direction:column;gap:14px;">
        <div class="callout rec">New leadership, real funding, booths at <span class="b">SOT · BIO · ISSX · AACR · ASGCT</span>.</div>
        <div class="callout note" style="font-style:normal;color:var(--navy);font-size:16px;">"The work of people who hit these walls and learned. Glad to talk to operators."</div>
      </div>
    </div>
  </div>
</section>

<!-- ============================================================
     SLIDE 12 — Proof: same engine, reagent company
     ============================================================ -->
<section class="slide content" id="slide-12">
  <div class="hdr">
    <div><div class="kicker">Who We Are / Proof</div><h2>The Same Engine, for a Reagent Company</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">12 / 26</div>
  </div>
  <div class="body">
    <div class="lead">Boster Bio — sells antibodies &amp; assays to <span class="accent">the same researchers</span> you serve.</div>
    <div class="statgrid four" style="margin-bottom:18px;">
      <div class="stat"><div class="num">↑</div><div class="lab">Search rankings up after our content program</div></div>
      <div class="stat"><div class="num">2×</div><div class="lab">Return on Google Ads spend</div></div>
      <div class="stat"><div class="num">$1000s<span class="u">/mo</span></div><div class="lab">Wasted ad spend eliminated</div></div>
      <div class="stat"><div class="num">Auto</div><div class="lab">Site → quote → order, with FedEx tracking</div></div>
    </div>
    <ul class="bul">
      <li>Offline / PO orders tied back to the sourcing ad</li>
      <li>Plus two more clients with documented traffic growth</li>
    </ul>
  </div>
</section>

<!-- ============================================================
     SLIDE 13 — Proof: we already built one of your pages (mockup)
     ============================================================ -->
<section class="slide content" id="slide-13">
  <div class="hdr">
    <div><div class="kicker">Who We Are / Proof</div><h2>We Already Built One of Your Pages</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">13 / 26</div>
  </div>
  <div class="body">
    <div class="cols2 wide-left" style="gap:34px;align-items:center;">
      <div class="mock">
        <div class="bar"><span class="dot"></span><span class="dot"></span><span class="dot"></span><span class="ttl">milecell.com / cryopreserved-human-hepatocytes</span></div>
        <div class="mbody">
          <div class="hero">
            <div class="h">Cryopreserved Human Hepatocytes</div>
            <div class="badges"><span class="badge">Cited in 14 publications</span><span class="badge">COA by lot</span></div>
          </div>
          <div class="specrow"><span class="k">Viability</span><span>&gt; 80% post-thaw</span></div>
          <div class="specrow"><span class="k">Format</span><span>Plateable / Suspension</span></div>
          <div class="specrow"><span class="k">Donor data</span><span>Age · sex · ethnicity</span></div>
          <div class="ctas"><div class="cta p">Request Quote</div><div class="cta s">Download COA</div></div>
        </div>
      </div>
      <div>
        <div class="lead" style="margin-bottom:14px;">A MileCell-branded product page, already built.</div>
        <ul class="bul">
          <li>Hero, specs table, citation badges</li>
          <li>Request Quote / Download COA actions</li>
          <li><span class="b">Built in hours — more detailed than the category leader</span></li>
        </ul>
        <div class="callout rec" style="margin-top:16px;"><span class="b">Imagine the whole catalog.</span></div>
      </div>
    </div>
  </div>
</section>

<!-- ============================================================
     SLIDE 14 — Vision
     ============================================================ -->
<section class="slide content" id="slide-14">
  <div class="hdr">
    <div><div class="kicker">The Growth Engine</div><h2>Where Your US Commercial Engine Could Be</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">14 / 26</div>
  </div>
  <div class="body">
    <div class="lead">In <span class="accent">12–18 months</span>: a repeatable system that turns researchers who search online into qualified leads and orders.</div>
    <ul class="bul">
      <li>Every dollar tied to pipeline you can show your board</li>
      <li>Ad spend that optimizes itself</li>
      <li>A content library that rivals BioIVT</li>
      <li>Reps freed from manual data entry</li>
    </ul>
    <div class="callout stat" style="margin-top:22px;font-size:18px;">"You industrialized inventory and quoting before — <span class="big">this industrializes demand.</span>"</div>
  </div>
</section>

<!-- ============================================================
     SLIDE 15 — Journey + competitor gap (table)
     ============================================================ -->
<section class="slide content" id="slide-15">
  <div class="hdr">
    <div><div class="kicker">The Growth Engine</div><h2>The Researcher Journey &amp; the Competitor Gap</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">15 / 26</div>
  </div>
  <div class="body" style="padding-top:24px;">
    <div class="journey">
      <span class="j">Educate</span><span class="arr">›</span>
      <span class="j">Discover</span><span class="arr">›</span>
      <span class="j">Validate</span><span class="arr">›</span>
      <span class="j">Quote / Sample</span><span class="arr">›</span>
      <span class="j">Purchase</span><span class="arr">›</span>
      <span class="j">Use</span><span class="arr">›</span>
      <span class="j">Reorder</span>
    </div>
    <table class="tbl">
      <thead><tr><th style="width:22%;">Stage</th><th style="width:39%;">BioIVT (dominates)</th><th>MileCell today</th></tr></thead>
      <tbody>
        <tr><td class="lbl">Educate / Discover / Validate</td><td>Decision guides, live inventory alerts, donor screening, COA depth</td><td class="no">Near-empty today</td></tr>
        <tr><td class="lbl" colspan="3" style="background:var(--teal-soft);color:#0c5e5b;">White space even BioIVT is weak on ↓</td></tr>
        <tr><td class="lbl">Citation badges</td><td colspan="2">On-page citation badges</td></tr>
        <tr><td class="lbl">Donor table</td><td colspan="2">A public, searchable donor table</td></tr>
        <tr><td class="lbl">Content cluster</td><td colspan="2">A hepatocyte selection-guide content cluster</td></tr>
        <tr><td class="lbl">Inventory</td><td colspan="2">Live ready-to-ship alerts</td></tr>
      </tbody>
    </table>
  </div>
</section>

<!-- ============================================================
     SLIDE 16 — Growth: Content & SEO
     ============================================================ -->
<section class="slide content" id="slide-16">
  <div class="hdr">
    <div><div class="kicker">The Growth Engine</div><h2>Content &amp; SEO — Including AI Search</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">16 / 26</div>
  </div>
  <div class="body">
    <div class="lead">Get found on Google <span class="accent">and inside AI answers</span> (ChatGPT).</div>
    <div class="statgrid three" style="margin-bottom:20px;">
      <div class="stat"><div class="num">+20<span class="u">% / +30%</span></div><div class="lab">Organic traffic &amp; product-page conversions in 6 months — a life-science reagent company</div></div>
      <div class="stat"><div class="num">#38<span class="u">→ #7</span></div><div class="lab">"Antibody discovery" moved in 8 months — a peer</div></div>
      <div class="stat"><div class="num">+527<span class="u">%</span></div><div class="lab">AI-referred visits grew in early 2025 — early-mover edge</div></div>
    </div>
    <div class="callout rec"><span class="b">The early-mover edge is real:</span> AI answer engines are surfacing reagent buyers now — before your competitors optimize for them.</div>
  </div>
</section>

<!-- ============================================================
     SLIDE 17 — Growth: Ads + attribution
     ============================================================ -->
<section class="slide content" id="slide-17">
  <div class="hdr">
    <div><div class="kicker">The Growth Engine</div><h2>Ads + Attribution</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">17 / 26</div>
  </div>
  <div class="body">
    <div class="lead">Cut wasted clicks; feed closed / PO orders back into Google so it <span class="accent">optimizes for real buyers</span>.</div>
    <div class="statgrid three" style="margin-bottom:20px;">
      <div class="stat"><div class="num">+42<span class="u">%</span></div><div class="lab">Qualified-lead share from offline-conversion feedback, in ~2 months</div></div>
      <div class="stat"><div class="num">~37<span class="u">% ↓</span></div><div class="lab">Lower cost per qualified opportunity</div></div>
      <div class="stat"><div class="num">10–25<span class="u">%</span></div><div class="lab">Of spend recovered via negative-keyword cleanup</div></div>
    </div>
    <div class="callout stat"><span class="big">Boster: 2× ROAS</span>, thousands per month saved.</div>
  </div>
</section>

<!-- ============================================================
     SLIDE 18 — Growth: Quote-to-cash automation
     ============================================================ -->
<section class="slide content" id="slide-18">
  <div class="hdr">
    <div><div class="kicker">The Growth Engine</div><h2>Quote-to-Cash Automation</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">18 / 26</div>
  </div>
  <div class="body">
    <div class="lead">Automate quote → approval → order → shipment; <span class="accent">win back stalled carts and quotes</span>.</div>
    <div class="statgrid three" style="margin-bottom:20px;">
      <div class="stat"><div class="num">~28<span class="u">% ↓</span></div><div class="lab">Sales cycle, &amp; approval wait cut up to 95% — quote automation (Aberdeen)</div></div>
      <div class="stat"><div class="num">~70<span class="u">%</span></div><div class="lab">Of carts abandoned; automated reminders recover 10–14%</div></div>
      <div class="stat"><div class="num">+37<span class="u">%</span></div><div class="lab">A 3-email sequence recovers more than a single reminder</div></div>
    </div>
    <div class="callout rec">The same site → quote → order automation we run for Boster, applied to your catalog.</div>
  </div>
</section>

<!-- ============================================================
     SLIDE 19 — Growth: Free the non-coders
     ============================================================ -->
<section class="slide content" id="slide-19">
  <div class="hdr">
    <div><div class="kicker">The Growth Engine</div><h2>Free the Non-Coders</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">19 / 26</div>
  </div>
  <div class="body">
    <div class="lead">Update pages, launch campaigns, manage products yourselves — in <span class="accent">hours, not dev-ticket weeks</span>.</div>
    <div class="cards three" style="margin-bottom:18px;">
      <div class="card"><div class="tag">Edit</div><h3>Visual page editor</h3><p>Drag-and-drop layout changes — no developer in the loop.</p></div>
      <div class="card"><div class="tag">Manage</div><h3>Content management</h3><p>A CMS your team actually owns and runs.</p></div>
      <div class="card"><div class="tag">Catalog</div><h3>AI-assisted PIM</h3><p>AI-assisted product information management.</p></div>
    </div>
    <div class="callout rec"><span class="b">Directly fixes your RFQ's #1 pain:</span> needing a developer for every change. (Works on any platform.)</div>
  </div>
</section>

<!-- ============================================================
     SLIDE 20 — Growth: Outbound + partners
     ============================================================ -->
<section class="slide content" id="slide-20">
  <div class="hdr">
    <div><div class="kicker">The Growth Engine</div><h2>Outbound + Partners</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">20 / 26</div>
  </div>
  <div class="body">
    <div class="lead">Inbound, plus a steady outbound motion to reach more labs <span class="accent">without a big sales team</span>.</div>
    <div class="cols2" style="gap:30px;">
      <ul class="bul">
        <li>Targeted email outreach</li>
        <li>Distributor / partner enablement</li>
        <li>The market-development motion you already know</li>
      </ul>
      <div class="callout stat" style="align-self:center;"><span class="big">50% more sales-ready leads at 33% lower cost</span> — strong lead nurturing (Forrester).</div>
    </div>
  </div>
</section>

<!-- ============================================================
     SLIDE 21 — Advanced feature menu (3 tiers)
     ============================================================ -->
<section class="slide content" id="slide-21">
  <div class="hdr">
    <div><div class="kicker">The Growth Engine</div><h2>Advanced Feature Menu</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">21 / 26</div>
  </div>
  <div class="body" style="padding-top:26px;">
    <div class="tiers">
      <div class="tier">
        <div class="th">Tier 1<span class="s">High impact, quick</span></div>
        <ul>
          <li>Faceted product finder</li>
          <li>Find-my-cell wizard</li>
          <li>Public donor / lot table w/ downloadable COA</li>
          <li>Cited-in-publications badge</li>
          <li>Hepatocyte selection-guide content</li>
        </ul>
      </div>
      <div class="tier">
        <div class="th">Tier 2<span class="s">Differentiating</span></div>
        <ul>
          <li>Live ready-to-ship inventory + alerts</li>
          <li>Product configurator</li>
          <li>Donor-matched cell sets</li>
          <li>Account-based pricing on login</li>
          <li>Loyalty + referral + first-order discount</li>
        </ul>
      </div>
      <div class="tier">
        <div class="th">Tier 3<span class="s">Big pharma</span></div>
        <ul>
          <li>Punchout to procurement (Coupa, SAP Ariba)</li>
          <li>Bulk order upload</li>
          <li>Full customer portal</li>
          <li>Structured scientific resource library</li>
        </ul>
      </div>
    </div>
  </div>
</section>

<!-- ============================================================
     SLIDE 22 — Build vs. operate
     ============================================================ -->
<section class="slide content" id="slide-22">
  <div class="hdr">
    <div><div class="kicker">The Growth Engine</div><h2>Build vs. Operate</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">22 / 26</div>
  </div>
  <div class="body">
    <div class="lead">Building it is step one. <span class="accent">Running it is where the return comes from.</span></div>
    <ul class="bul">
      <li>A platform sitting idle doesn't pay back</li>
      <li>Return comes from operating it monthly: content, automations, ad optimization, reporting</li>
      <li>That's why growth is an ongoing monthly engagement</li>
      <li>In-house hires take years to build proven systems — we bring systems already tested</li>
    </ul>
  </div>
</section>

<!-- ============================================================
     SLIDE 23 — Pricing Tier 1 (TOTAL 628 hrs / $78,500)
     ============================================================ -->
<section class="slide content" id="slide-23">
  <div class="hdr">
    <div><div class="kicker">Pricing · Timeline · Close</div><h2>Tier 1 — The Platform</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">23 / 26</div>
  </div>
  <div class="body" style="padding:18px 56px 20px;">
    <table class="tbl" style="font-size:13.5px;">
      <thead><tr><th>Module</th><th class="num" style="width:14%;">Hours</th><th class="num" style="width:16%;">Cost</th></tr></thead>
      <tbody>
        <tr><td class="lbl">Platform setup + CMS</td><td class="num">60</td><td class="num">$7,500</td></tr>
        <tr><td class="lbl">Catalog + filtering</td><td class="num">48</td><td class="num">$6,000</td></tr>
        <tr><td class="lbl">Scientific citations (Bioz)</td><td class="num">24</td><td class="num">$3,000</td></tr>
        <tr><td class="lbl">B2B e-commerce (quote-to-order; card / PO / offline)</td><td class="num">120</td><td class="num">$15,000</td></tr>
        <tr><td class="lbl">Customer account &amp; portal</td><td class="num">64</td><td class="num">$8,000</td></tr>
        <tr><td class="lbl">HubSpot CRM integration</td><td class="num">48</td><td class="num">$6,000</td></tr>
        <tr><td class="lbl">Marketing automation</td><td class="num">40</td><td class="num">$5,000</td></tr>
        <tr><td class="lbl">Resource hub + COA-by-lot search</td><td class="num">56</td><td class="num">$7,000</td></tr>
        <tr><td class="lbl">Security &amp; access</td><td class="num">24</td><td class="num">$3,000</td></tr>
        <tr><td class="lbl">Integrations (payments / email / analytics)</td><td class="num">40</td><td class="num">$5,000</td></tr>
        <tr><td class="lbl">SEO / analytics / speed / mobile</td><td class="num">40</td><td class="num">$5,000</td></tr>
        <tr><td class="lbl">PM / QA / training / docs</td><td class="num">64</td><td class="num">$8,000</td></tr>
        <tr class="total"><td>TOTAL</td><td class="num"><span class="teal">628 hours</span></td><td class="num"><span class="teal">$78,500</span></td></tr>
      </tbody>
    </table>
    <div style="display:flex;justify-content:space-between;align-items:center;margin-top:12px;gap:20px;">
      <div class="src">No payment upfront · milestone-based acceptance · hosting / third-party fees billed to your accounts.</div>
      <div class="src" style="font-style:italic;text-align:right;">Rate is a placeholder baseline — adjust $/hr or hours and the total recomputes.</div>
    </div>
  </div>
</section>

<!-- ============================================================
     SLIDE 24 — Pricing Tier 2 (~$6,500/month)
     ============================================================ -->
<section class="slide content" id="slide-24">
  <div class="hdr">
    <div><div class="kicker">Pricing · Timeline · Close</div><h2>Tier 2 — The Growth Engine</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">24 / 26</div>
  </div>
  <div class="body">
    <div class="cols2" style="gap:34px;align-items:center;">
      <div>
        <table class="tbl">
          <thead><tr><th>Engagement</th><th class="num">Investment</th></tr></thead>
          <tbody>
            <tr><td class="lbl">Milestone-based monthly build + maintenance</td><td class="num"><span class="teal b" style="font-size:18px;">~$6,500</span><span class="muted">/mo</span></td></tr>
            <tr><td colspan="2" class="sm muted">Each month ships one growth module and maintains the rest.</td></tr>
          </tbody>
        </table>
        <div class="callout warn" style="margin-top:18px;"><span class="b">Optional but recommended</span> — the platform underdelivers without it.</div>
      </div>
      <div>
        <div class="lead" style="font-size:18px;">What the monthly engine returns:</div>
        <div class="chips">
          <span class="c">More organic traffic</span>
          <span class="c">2× ad return</span>
          <span class="c">Recovered carts</span>
          <span class="c">Faster quotes</span>
          <span class="c">Attributed pipeline</span>
        </div>
      </div>
    </div>
  </div>
</section>

<!-- ============================================================
     SLIDE 25 — Timeline & team
     ============================================================ -->
<section class="slide content" id="slide-25">
  <div class="hdr">
    <div><div class="kicker">Pricing · Timeline · Close</div><h2>Timeline &amp; Team</h2></div>
    <div class="brandmark">MILECELL</div><div class="pageno">25 / 26</div>
  </div>
  <div class="body">
    <div class="cards two" style="margin-bottom:22px;">
      <div class="card">
        <div class="tag">Phase 1 — Core</div>
        <h3>Site + catalog + CRM</h3>
        <p><span class="b navytxt">4–8 weeks</span> (your target), delivered in milestones.</p>
      </div>
      <div class="card featured">
        <div class="tag">Phase 2 — Growth</div>
        <h3>The growth engine</h3>
        <p>Rolling — <span class="b navytxt">one module per month</span>.</p>
      </div>
    </div>
    <div class="lead" style="font-size:18px;margin-bottom:12px;">Your team:</div>
    <div class="diffstrip" style="grid-template-columns:repeat(3,1fr);">
      <div class="d"><span class="b">Project Manager</span>Your main contact</div>
      <div class="d"><span class="b">Technical Lead</span>Build &amp; integrations</div>
      <div class="d"><span class="b">Business / Marketing Consultant</span>Growth &amp; strategy</div>
    </div>
  </div>
</section>

<!-- ============================================================
     SLIDE 26 — CLOSE (gpt-image-2 background: 04-closing-bg.png)
     ============================================================ -->
<section class="slide close" id="slide-26">
  <div class="close-inner">
    <h1>Let's lock the big decisions <span class="teal">together.</span></h1>
    <div class="reasons">
      <div class="r">Built this exact engine for a reagent company</div>
      <div class="r">We speak your science</div>
      <div class="r">We live in HubSpot &amp; automation</div>
      <div class="r">Operators with your stakes</div>
    </div>
    <div class="nextstep"><span class="b">Next step:</span> a 45-minute working session to settle the four upfront decisions — platform, CRM / transactions, hosting, catalog structure.</div>
    <div class="sign">"Glad to be working with people who mean business."</div>
  </div>
</section>

</div>
</body>
</html>
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/templates/video/package.json -->
```json
{
  "name": "milecell-intro-remotion",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "description": "Remotion branded intro for the MileCell sales-proposal deck",
  "scripts": {
    "dev": "remotion studio",
    "render": "remotion render MileCellIntro out/MileCell-intro.mp4",
    "still": "remotion still MileCellIntro"
  },
  "dependencies": {
    "@remotion/cli": "^4.0.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "remotion": "^4.0.0"
  },
  "devDependencies": {
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "typescript": "^5.4.0"
  }
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/templates/video/src/MileCellIntro.tsx -->
```tsx
import {
  AbsoluteFill,
  Img,
  Sequence,
  interpolate,
  spring,
  staticFile,
  useCurrentFrame,
  useVideoConfig,
} from "remotion";

// ---- Brand tokens ----
const NAVY = "#1E3A5F";
const TEAL = "#1CA9A6";
const WHITE = "#FFFFFF";
const FONT =
  '"Helvetica Neue", Helvetica, Arial, "Segoe UI", system-ui, sans-serif';

export const MileCellIntro: React.FC = () => {
  const frame = useCurrentFrame();
  const { fps, durationInFrames } = useVideoConfig();

  // Slow, subtle full-bleed zoom on the AI-generated hex background: 1.0 -> 1.06
  const bgScale = interpolate(frame, [0, durationInFrames], [1.0, 1.06], {
    extrapolateRight: "clamp",
  });

  // Gentle fade-to-navy over the last ~15 frames
  const fadeToNavy = interpolate(
    frame,
    [durationInFrames - 15, durationInFrames],
    [0, 1],
    { extrapolateLeft: "clamp", extrapolateRight: "clamp" }
  );

  // A soft navy scrim over the photo so text stays legible throughout
  const scrimOpacity = 0.55;

  return (
    <AbsoluteFill style={{ backgroundColor: NAVY }}>
      {/* Background image with slow zoom */}
      <AbsoluteFill
        style={{
          transform: `scale(${bgScale})`,
          transformOrigin: "center center",
        }}
      >
        <Img
          src={staticFile("01-title-bg.png")}
          style={{
            width: "100%",
            height: "100%",
            objectFit: "cover",
          }}
        />
      </AbsoluteFill>

      {/* Navy scrim + bottom gradient for legibility */}
      <AbsoluteFill
        style={{
          background: `linear-gradient(180deg, rgba(30,58,95,${
            scrimOpacity - 0.1
          }) 0%, rgba(30,58,95,${scrimOpacity}) 55%, rgba(30,58,95,${
            scrimOpacity + 0.2
          }) 100%)`,
        }}
      />

      {/* Centered text block */}
      <AbsoluteFill
        style={{
          justifyContent: "center",
          alignItems: "center",
          fontFamily: FONT,
          padding: "0 160px",
          textAlign: "center",
        }}
      >
        {/* Eyebrow */}
        <Eyebrow frame={frame} fps={fps} />

        {/* Title */}
        <Title frame={frame} fps={fps} />

        {/* Accent rule */}
        <AccentRule frame={frame} fps={fps} />

        {/* Subtitle */}
        <Subtitle frame={frame} fps={fps} />
      </AbsoluteFill>

      {/* Footer */}
      <Footer frame={frame} fps={fps} />

      {/* Fade to navy at the end */}
      <AbsoluteFill
        style={{
          backgroundColor: NAVY,
          opacity: fadeToNavy,
          pointerEvents: "none",
        }}
      />
    </AbsoluteFill>
  );
};

// ---------- Pieces ----------

const Eyebrow: React.FC<{ frame: number; fps: number }> = ({ frame }) => {
  const start = 10;
  const opacity = interpolate(frame, [start, start + 18], [0, 1], {
    extrapolateLeft: "clamp",
    extrapolateRight: "clamp",
  });
  return (
    <div
      style={{
        opacity,
        color: TEAL,
        fontSize: 30,
        fontWeight: 600,
        letterSpacing: 8,
        textTransform: "uppercase",
        marginBottom: 28,
      }}
    >
      MileCell Biotechnology
    </div>
  );
};

const Title: React.FC<{ frame: number; fps: number }> = ({ frame, fps }) => {
  const start = 24;
  const local = Math.max(0, frame - start);
  const s = spring({
    frame: local,
    fps,
    config: { damping: 200, stiffness: 90, mass: 1 },
  });
  const translateY = interpolate(s, [0, 1], [40, 0]);
  const opacity = interpolate(local, [0, 16], [0, 1], {
    extrapolateLeft: "clamp",
    extrapolateRight: "clamp",
  });
  return (
    <div
      style={{
        opacity,
        transform: `translateY(${translateY}px)`,
        color: WHITE,
        fontSize: 76,
        fontWeight: 700,
        lineHeight: 1.12,
        maxWidth: 1400,
      }}
    >
      MileCell Digital Commerce Platform{" "}
      <span style={{ color: TEAL }}>&amp; Growth Roadmap</span>
    </div>
  );
};

const AccentRule: React.FC<{ frame: number; fps: number }> = ({ frame }) => {
  const start = 56;
  const width = interpolate(frame, [start, start + 22], [0, 180], {
    extrapolateLeft: "clamp",
    extrapolateRight: "clamp",
  });
  return (
    <div
      style={{
        height: 5,
        width,
        backgroundColor: TEAL,
        borderRadius: 4,
        margin: "36px 0",
      }}
    />
  );
};

const Subtitle: React.FC<{ frame: number; fps: number }> = ({ frame }) => {
  const start = 72;
  const opacity = interpolate(frame, [start, start + 22], [0, 1], {
    extrapolateLeft: "clamp",
    extrapolateRight: "clamp",
  });
  const translateY = interpolate(frame, [start, start + 22], [24, 0], {
    extrapolateLeft: "clamp",
    extrapolateRight: "clamp",
  });
  return (
    <div
      style={{
        opacity,
        transform: `translateY(${translateY}px)`,
        color: "rgba(255,255,255,0.88)",
        fontSize: 34,
        fontWeight: 400,
        lineHeight: 1.4,
        maxWidth: 1150,
      }}
    >
      From a custom-coded site to a self-manageable, CRM-integrated revenue
      engine.
    </div>
  );
};

const Footer: React.FC<{ frame: number; fps: number }> = ({ frame }) => {
  return (
    <Sequence from={104}>
      <FooterInner frame={frame} />
    </Sequence>
  );
};

const FooterInner: React.FC<{ frame: number }> = ({ frame }) => {
  const start = 104;
  const opacity = interpolate(frame, [start, start + 20], [0, 1], {
    extrapolateLeft: "clamp",
    extrapolateRight: "clamp",
  });
  return (
    <AbsoluteFill
      style={{
        justifyContent: "flex-end",
        alignItems: "center",
        paddingBottom: 64,
        fontFamily: FONT,
      }}
    >
      <div
        style={{
          opacity,
          color: "rgba(255,255,255,0.72)",
          fontSize: 24,
          fontWeight: 500,
          letterSpacing: 1.5,
        }}
      >
        Prepared for MileCell Biotechnology&nbsp;&nbsp;|&nbsp;&nbsp;June 2026
      </div>
    </AbsoluteFill>
  );
};
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/templates/video/src/Root.tsx -->
```tsx
import { Composition } from "remotion";
import { MileCellIntro } from "./MileCellIntro";

export const RemotionRoot: React.FC = () => {
  return (
    <Composition
      id="MileCellIntro"
      component={MileCellIntro}
      durationInFrames={180}
      fps={30}
      width={1920}
      height={1080}
    />
  );
};
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/templates/video/src/index.ts -->
```ts
import { registerRoot } from "remotion";
import { RemotionRoot } from "./Root";

registerRoot(RemotionRoot);
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/slideDeckAndVideoGeneration/templates/video/tsconfig.json -->
```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "lib": ["DOM", "DOM.Iterable", "ESNext"],
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true,
    "noEmit": true,
    "isolatedModules": true,
    "resolveJsonModule": true
  },
  "include": ["src"]
}
```
<!-- /GUILDOS-FILE -->
