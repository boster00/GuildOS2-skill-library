# imageGeneration

<!-- GUILDOS-SKILL name="imageGeneration" -->

<!-- GUILDOS-DESC -->
Generate images and visual assets. Default AI model is gpt-image-2 (via the openai_images weapon) — will be upgraded to more advanced models over time. Also renders brand-locked, deterministic posters via the poster_render weapon (sprite sheets, illustrations, posters). Two paths: creative/variety output (sprite sheets, illustrations, photo art) via openai_images, and brand-locked, byte-stable deliverables via poster_render (Playwright HTML/CSS → PNG, ~1.4 s/render vs ~135 s for AI gen). Design reading (Figma) lives in the figma skill book, not here.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/imageGeneration/SKILL.md -->
```markdown
---
name: imageGeneration
description: >-
  Generate images and visual assets. Default AI model is gpt-image-2 (via the
  openai_images weapon) — will be upgraded to more advanced models over time.
  Also renders brand-locked, deterministic posters via the poster_render weapon
  (sprite sheets, illustrations, posters). Two paths: creative/variety output
  (sprite sheets, illustrations, photo art) via openai_images, and brand-locked,
  byte-stable deliverables via poster_render (Playwright HTML/CSS → PNG,
  ~1.4 s/render vs ~135 s for AI gen). Design reading (Figma) lives in the
  figma skill book, not here.
---

# Image Generation

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/imageGeneration/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto imageGeneration <action>` (or read the source directly).

## Actions

- **generateSpriteSheet** — Generate a chibi character sprite sheet (1280×720, 8×4 grid, 4 animations) using the locked standard prompt via gpt-image-2. Pass character='pig' for the GuildOS pig agent sheet.
- **refPosterTemplateAuthoring** — LOCKED contract for authoring a new poster template under public/poster-templates/<slug>/.
- **renderPoster** — Brand-locked, deterministic PNG render from an HTML/CSS template under
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/imageGeneration/index.js -->
```js
/**
 * Image Generation skill book — pure text-prompt registry for producing visual
 * assets. Two paths, picked by need:
 *   - Creative / variety output (sprite sheets, illustrations, photo art) →
 *     openai_images weapon. Default model: gpt-image-2 (the current best
 *     available — swap to a more advanced model here when one ships).
 *   - Brand-locked, byte-stable deliverable → poster_render weapon
 *     (Playwright HTML/CSS → PNG). ~1.4 s/render vs ~135 s for AI gen.
 * Design *reading* (Figma) lives in the `figma` skill book + weapon, not here.
 * No JS logic in this book.
 */

export const skillBook = {
  id: "imageGeneration",
  title: "Image Generation",
  description:
    "Generate images and visual assets. Default AI model is gpt-image-2 (via the openai_images " +
    "weapon) — will be upgraded to more advanced models over time. Also renders brand-locked, " +
    "deterministic posters via the poster_render weapon (sprite sheets, illustrations, posters).",

  toc: {
    renderPoster: {
      description:
        "Brand-locked, deterministic PNG render from an HTML/CSS template under " +
        "public/poster-templates/<slug>/. Calls the poster_render weapon (Playwright " +
        "headless Chromium). For run-stable outputs where AI-gen jitter is unacceptable. " +
        "Read refPosterTemplateAuthoring before adding a new template.",
      input: {
        template: "string slug — directory name under public/poster-templates/",
        data: "JSON-serializable payload the template's window.renderPoster() consumes",
        outPath: "absolute filesystem path to write the PNG",
        viewport: "{w,h} — default 1024x1536; matches .poster-canvas dimensions in the template",
      },
      output: { path: "string", bytes: "number", viewport: "{w,h}", elapsed_ms: "number" },
    },
    generateSpriteSheet: {
      description: "Generate a chibi character sprite sheet (1280×720, 8×4 grid, 4 animations) using the locked standard prompt via gpt-image-2. Pass character='pig' for the GuildOS pig agent sheet.",
    },
    refPosterTemplateAuthoring: {
      description:
        "LOCKED contract for authoring a new poster template under public/poster-templates/<slug>/. " +
        "Required exports: window.renderPoster(data) function and .poster-canvas.poster-ready " +
        "ready-marker class. Includes brand-color CSS-var convention, screenshot-target rules, " +
        "and the canonical wb-decision-card reference template.",
    },
  },

  generateSpriteSheet: `
Generate a sprite sheet using the \`openai_images\` weapon (\`@/libs/weapon/openai_images\`).

**Model:** gpt-image-2
**Size:** 1536x1024 (landscape — closest supported size to 1280×720)
**Quality:** high
**DO NOT** pass \`response_format\` — unsupported, always returns b64_json.

**Invocation:**
\`\`\`js
import { generate } from "@/libs/weapon/openai_images";
const [b64] = await generate(
  { prompt: PROMPT, size: "1536x1024", quality: "high" },
  userId
);
const buffer = Buffer.from(b64, "base64");
\`\`\`

**Canvas / grid spec (locked — applies to all characters):**
- Canvas: 1280×720 px
- Grid: 8 columns × 4 rows
- Cell: 160×180 px
- Sprite crop area: 128×128 px, offset x+16 y+24 inside each cell
- Labels below each pose, format: [pose, frame n, x, y, w, h]

**Row layout:**
| Row | Animation | Frames |
|-----|-----------|--------|
| 1   | idle (subtle breathing/bobbing) | 4 |
| 2   | working (waving sword/tool) | 8 |
| 3   | attention (jumping, one hand raised, O-mouth) | 6 |
| 4   | walk cycle | 8 |

**Exact frame labels (corrected y offsets — y+24 inside each 180px cell):**
\`\`\`
idle, frame 1, 16, 24, 128, 128      idle, frame 2, 176, 24, 128, 128
idle, frame 3, 336, 24, 128, 128     idle, frame 4, 496, 24, 128, 128

working, frame 1, 16, 204, 128, 128  working, frame 2, 176, 204, 128, 128
working, frame 3, 336, 204, 128, 128 working, frame 4, 496, 204, 128, 128
working, frame 5, 656, 204, 128, 128 working, frame 6, 816, 204, 128, 128
working, frame 7, 976, 204, 128, 128 working, frame 8, 1136, 204, 128, 128

attention, frame 1, 16, 384, 128, 128  attention, frame 2, 176, 384, 128, 128
attention, frame 3, 336, 384, 128, 128 attention, frame 4, 496, 384, 128, 128
attention, frame 5, 656, 384, 128, 128 attention, frame 6, 816, 384, 128, 128

walk, frame 1, 16, 564, 128, 128  walk, frame 2, 176, 564, 128, 128
walk, frame 3, 336, 564, 128, 128 walk, frame 4, 496, 564, 128, 128
walk, frame 5, 656, 564, 128, 128 walk, frame 6, 816, 564, 128, 128
walk, frame 7, 976, 564, 128, 128 walk, frame 8, 1136, 564, 128, 128
\`\`\`

---

**Character prompts — pick the right one:**

**Pig (GuildOS pig agent — use this for "piggy"):**
\`\`\`
Create a sprite sheet for a round chibi cowboy pig character on a white background.

Character design: chubby round pig face with pink snout, wearing a brown cowboy hat with a gold sheriff star, red bandana around neck, brown leather vest, blue jeans, brown cowboy boots. Soft pastel chibi art style, clean 2D illustration, low visual noise.

Canvas size 1280x720 pixels. Arrange frames in a clean 8-column by 4-row grid. Each cell is 160x180 pixels. Inside each cell, place a 128x128 character pose area with consistent scale and centered alignment. Leave label text below each pose in small readable font.

Row 1 (IDLE STANDING): 4 frames, subtle breathing/bobbing animation.
Row 2 (WORKING - WAVING SWORD): 8 frames, character waves a silver sword in a full swing arc.
Row 3 (RAISING HAND FOR ATTENTION): 6 frames, character jumps with one hand raised, O-shaped mouth, sparkle effects.
Row 4 (WALKING): 8 frames, simple side-view walk cycle.

Under each frame, add labels in this exact format [pose, frame n, x, y, w, h]:
idle, frame 1, 16, 24, 128, 128 | idle, frame 2, 176, 24, 128, 128 | idle, frame 3, 336, 24, 128, 128 | idle, frame 4, 496, 24, 128, 128
working, frame 1, 16, 204, 128, 128 | working, frame 2, 176, 204, 128, 128 | working, frame 3, 336, 204, 128, 128 | working, frame 4, 496, 204, 128, 128 | working, frame 5, 656, 204, 128, 128 | working, frame 6, 816, 204, 128, 128 | working, frame 7, 976, 204, 128, 128 | working, frame 8, 1136, 204, 128, 128
attention, frame 1, 16, 384, 128, 128 | attention, frame 2, 176, 384, 128, 128 | attention, frame 3, 336, 384, 128, 128 | attention, frame 4, 496, 384, 128, 128 | attention, frame 5, 656, 384, 128, 128 | attention, frame 6, 816, 384, 128, 128
walk, frame 1, 16, 564, 128, 128 | walk, frame 2, 176, 564, 128, 128 | walk, frame 3, 336, 564, 128, 128 | walk, frame 4, 496, 564, 128, 128 | walk, frame 5, 656, 564, 128, 128 | walk, frame 6, 816, 564, 128, 128 | walk, frame 7, 976, 564, 128, 128 | walk, frame 8, 1136, 564, 128, 128
\`\`\`

**Generic placeholder (featureless chibi body — for new character templates):**
\`\`\`
Create a transparent-background sprite sheet template for a featureless round chibi character body, no accessories, no animal features, no clothing, simple placeholder body only. Canvas size 1280x720 pixels. 8-column by 4-row grid, each cell 160x180 px, 128x128 crop area per pose, x+16 y+24 offset. Soft chibi fantasy art, pastel colors, low visual noise. Row 1: idle 4 frames. Row 2: working 8 frames. Row 3: attention 6 frames. Row 4: walk 8 frames. Label each frame below as [pose, frame n, x, y, w, h] using the corrected y values: idle y=24, working y=204, attention y=384, walk y=564.
\`\`\`

**After generating:**
1. Save buffer to Supabase Storage — bucket: \`assets\`, path: \`sprite-sheets/<character>-<timestamp>.png\`.
2. Store public URL in quest inventory under key \`sprite_sheet_url\`.
`.trim(),

  renderPoster: `
Render a brand-locked, deterministic PNG via the \`poster_render\` weapon (\`@/libs/weapon/poster_render\`).

**When to use:** when output stability + brand control matter more than visual variety. AI image-gen jitters every run; this path produces byte-identical PNGs for byte-identical inputs.

**Invocation (from an API route or another skill-book action — NOT from a script):**
\`\`\`js
import { renderPoster } from "@/libs/weapon/poster_render";
const result = await renderPoster({
  template: "wb-decision-card",    // slug under public/poster-templates/
  data: {                           // JSON payload your template consumes
    gene: "BDNF",
    proteinName: "Brain-Derived Neurotrophic Factor",
    uniprot: "P23560",
    tiles: [{ title: "Quick Facts", body: "27 kDa precursor ...", accent: "ok" }, ...]
  },
  outPath: "/abs/path/to/output.png",
  viewport: { w: 1024, h: 1536 }   // optional; default matches .poster-canvas
});
// → { path, bytes, viewport, elapsed_ms, template, source_url }
\`\`\`

**Reference template (canonical):** \`public/poster-templates/wb-decision-card/\` — the brand-locked Western-Blot decision card. Wired by \`app/api/pg/generate-poster/route.js\` with per-gene PRESETS for BDNF + CD68.

**End-to-end render time:** ~1.2–1.5 s on dev box (cold Chromium launch + ~50 ms screenshot).

**Caller discipline:** routes (\`app/api/**\`) or skill-book actions only. Do NOT call from \`scripts/\` — see memory \`feedback_skill_books_weapons_first.md\` LOCKED 2026-05-22.
`.trim(),

  refPosterTemplateAuthoring: `
LOCKED contract for adding a new poster template under \`public/poster-templates/<slug>/\`. Read this BEFORE writing a new template. Pattern proven on \`wb-decision-card\` (Iter 20, 2026-05-22).

## Required structure

\`\`\`
public/poster-templates/<slug>/
  index.html   # single-file template; CSS + JS inlined; brand assets co-located
  assets/      # optional — logos, fonts, sub-images referenced by index.html
\`\`\`

## Required contract — your index.html MUST do these three things

### 1. Define \`.poster-canvas\` as the screenshot target
The weapon takes a screenshot of this element. Size it to your intended PNG dimensions exactly. Margins around it inside the document body are ignored.

\`\`\`html
<div class="poster-canvas" id="canvas">
  ...all visual content...
</div>
\`\`\`

### 2. Expose \`window.renderPoster(data)\` as the data entry point
The weapon calls this via \`page.evaluate\` after \`load\`. URL-hash data is NOT a supported entry point in headless mode (race conditions with hashchange listener attachment).

\`\`\`js
window.renderPoster = function(data){
  // populate DOM from data
  document.getElementById('hGene').textContent = data.gene;
  // ...etc
  // signal readiness:
  document.getElementById('canvas').classList.add('poster-ready');
};
\`\`\`

### 3. Toggle \`.poster-canvas.poster-ready\` when done
The weapon waits for this exact selector before screenshotting. If your render does async work (font load, image decode), only add \`poster-ready\` AFTER the work finishes.

## Brand-control conventions

- **Colors:** CSS custom properties at \`:root\` — \`--ink\`, \`--paper\`, \`--amber\`, \`--gold\`, \`--green\`, \`--red\`, \`--hair\`, \`--mid\`. Future templates re-use the same names; brand changes are one edit.
- **Typography:** title in serif (Playfair/Garamond), labels in Inter caps + tracked. Load brand fonts from \`/assets/fonts/\` if available; otherwise system fallback.
- **Layout:** outer canvas border + inner outline mark the print-safe area. Use \`grid-template-columns: 1fr 1fr\` for 2-column tile grids.
- **Accent classes:** \`accent: "warn"\` (amber border + red badge) and \`accent: "ok"\` (green badge) are the canonical tile accents — re-use these names.

## Optional but useful

- **Manual preview via URL hash:** add a tiny script that reads \`#data=<base64-JSON>\` and calls \`window.renderPoster(...)\` so designers can iterate in a normal browser tab without round-tripping through the weapon.
- **Console-error capture:** the weapon listens to \`console\` + \`pageerror\` and logs failures. Throw early and loud in your renderPoster — silent failures hang the wait.

## Wiring into a route

\`\`\`js
// app/api/pg/<your-feature>/route.js
import { renderPoster } from "@/libs/weapon/poster_render";
// In POST handler:
const result = await renderPoster({
  template: "<your-slug>",
  data: payload,
  outPath: path.join(OUT_DIR, filename),
  viewport: { w: 1024, h: 1536 },
});
\`\`\`

The middleware allow-list at \`libs/council/auth/middleware.js\` must include \`/poster-templates\` — Playwright requests get redirected to \`/signin\` otherwise.

## Reference template

\`public/poster-templates/wb-decision-card/index.html\` — 8-tile decision card, full brand-color palette, accent-class examples, hash-based preview. Copy it as the starting point for any new poster type.
`.trim(),
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

<!-- GUILDOS-FILE: libs/weapon/poster_render/index.js -->
```js
/**
 * Poster Render weapon — deterministic PNG generation from an HTML/CSS template.
 *
 * Use case: brand-locked, run-stable image deliverables (decision cards,
 * one-page guides, social previews) where AI image-gen jitter is unacceptable.
 * Data goes in as JSON, exact same bytes come out for the same input.
 *
 * Pattern:
 *   - The TEMPLATE is a static HTML file in /public/poster-templates/<slug>/.
 *     It reads its `data` from the URL hash (`#data=<base64-encoded JSON>`)
 *     and renders client-side. Brand assets (logo, fonts, colors) live
 *     inside the template — agents never see them.
 *   - The WEAPON launches headless Chromium via Playwright, navigates to
 *     `http://localhost:<port>/poster-templates/<slug>/#data=<...>`, waits
 *     for a sentinel selector (`.poster-ready`), screenshots the
 *     `.poster-canvas` element, saves PNG.
 *
 * Browser launch goes through the single substrate (leafBrowser.launchContext)
 * — the only chromium-launch site in the repo. This weapon keeps its own
 * page-driving (data injection, sentinel wait, element shot); it just no longer
 * launches Chromium itself. See leafBrowser engineering_decisions.md 2026-06-15.
 *
 * ─── API surface ──────────────────────────────────────────────────────────
 * renderPoster({ template, data, outPath, port?, waitMs? })
 *   template — slug under /public/poster-templates/  (e.g. "wb-decision-card")
 *   data     — JSON-serializable payload the template reads from #data hash
 *   outPath  — absolute path to write the PNG
 *   port     — dev-server port (default 3002)
 *   waitMs   — extra ms to wait after `.poster-ready` (default 200)
 *
 * Returns { path, bytes, viewport: {w,h}, elapsed_ms, template, source_url }.
 *
 * ─── Caller discipline ────────────────────────────────────────────────────
 * Call from API routes (app/api/**) or skill-book actions. Do NOT call from
 * scripts in scripts/ — see memory feedback_skill_books_weapons_first.md
 * LOCKED 2026-05-22.
 */

import fs from "node:fs/promises";
import path from "node:path";
import { launchContext } from "../leafBrowser/index.js";

export const toc = {
  renderPoster:
    "Render a deterministic PNG from a static HTML/CSS template under /public/poster-templates/<slug>/. Data injected via URL hash. Headless Chromium via Playwright. Brand-locked, run-stable, no AI in the hot path.",
};

/**
 * Find the .poster-canvas element in the template page, screenshot it as PNG,
 * write to outPath.
 *
 * @param {{template:string, data:object, outPath:string, port?:number, waitMs?:number, viewport?:{w:number,h:number}}} input
 * @returns {Promise<{path:string, bytes:number, viewport:{w:number,h:number}, elapsed_ms:number, template:string, source_url:string}>}
 */
export async function renderPoster({
  template,
  data,
  outPath,
  port = 3002,
  waitMs = 200,
  viewport = { w: 1024, h: 1536 },
}) {
  if (!template || typeof template !== "string") throw new Error("template (slug) required");
  if (!outPath || typeof outPath !== "string") throw new Error("outPath required");
  if (data == null) throw new Error("data required");

  const t0 = Date.now();
  // Use explicit index.html — Next.js static serving under /public doesn't
  // auto-index trailing-slash paths (returns 404 for /poster-templates/foo/).
  // Data is injected via page.evaluate after load — URL hashes are flaky in
  // headless contexts (race between hash listener attachment and document
  // parse, browser-specific stripping).
  const sourceUrl = `http://localhost:${port}/poster-templates/${template}/index.html`;

  const launched = await launchContext({
    viewport: { width: viewport.w, height: viewport.h },
    deviceScaleFactor: 2, // sharper text on the resulting PNG
  });
  if (!launched.ok) throw new Error(`poster_render: browser launch failed — ${launched.reason}`);
  const { context, close } = launched;
  try {
    const page = await context.newPage();
    page.on("console", (msg) => {
      if (msg.type() === "error") {
        // eslint-disable-next-line no-console
        console.error("[poster_render template console.error]", msg.text());
      }
    });
    page.on("pageerror", (err) => {
      // eslint-disable-next-line no-console
      console.error("[poster_render template pageerror]", err.message);
    });
    // Use 'load' (not 'networkidle') — Next.js dev keeps a HMR WebSocket open
    // that prevents networkidle from ever firing.
    await page.goto(sourceUrl, { waitUntil: "load", timeout: 20000 });
    // Inject the data payload directly — calls the template's window.renderPoster
    // function with the JSON payload. Templates MUST expose
    // `window.renderPoster(data)` and toggle `.poster-canvas.poster-ready` when done.
    await page.evaluate((payload) => {
      if (typeof window.renderPoster !== "function") {
        throw new Error("template must expose window.renderPoster(data)");
      }
      window.renderPoster(payload);
    }, data);
    // Templates signal "I'm done laying out, fonts loaded, assets in" by
    // exposing a `.poster-ready` class on the canvas element.
    // state:'attached' — element-in-DOM is sufficient; we don't need visibility
    // checks (canvas may be larger than viewport, which marks it 'not visible'
    // by Playwright's default).
    await page.waitForSelector(".poster-canvas.poster-ready", { state: "attached", timeout: 15000 });
    if (waitMs > 0) await page.waitForTimeout(waitMs);

    const canvas = await page.$(".poster-canvas");
    if (!canvas) throw new Error("template did not expose .poster-canvas element");
    const buffer = await canvas.screenshot({ type: "png", omitBackground: false });

    await fs.mkdir(path.dirname(outPath), { recursive: true });
    await fs.writeFile(outPath, buffer);

    return {
      path: outPath,
      bytes: buffer.length,
      viewport,
      elapsed_ms: Date.now() - t0,
      template,
      source_url: sourceUrl,
    };
  } finally {
    await close();
  }
}
```
<!-- /GUILDOS-FILE -->
