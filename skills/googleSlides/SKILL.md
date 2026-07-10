# googleSlides

<!-- GUILDOS-SKILL name="googleSlides" -->

<!-- GUILDOS-DESC -->
Slides-NATIVE fallback (HTML is the default deck substrate — [SB:slideDeckAndVideoGeneration]). Use ONLY when the deliverable must be a real, collaborative/shareable Google Slides file. Builds/edits branded decks via the Slides API presentations.batchUpdate: slides, text, tables, images-by-URL, backgrounds, brand colors. Covers OAuth setup and the LOCKED finding that API image insert bypasses the UI 'cannot upload images' restriction.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/googleSlides/SKILL.md -->
```markdown
---
name: googleSlides
description: >-
  Slides-NATIVE fallback (HTML is the default deck substrate —
  [SB:slideDeckAndVideoGeneration]). Use ONLY when the deliverable must be a
  real, collaborative/shareable Google Slides file. Builds/edits branded decks
  via the Slides API presentations.batchUpdate: slides, text, tables,
  images-by-URL, backgrounds, brand colors. Covers OAuth setup and the LOCKED
  finding that API image insert bypasses the UI 'cannot upload images'
  restriction.
---

# Google Slides — Programmatic Deck Builder (Slides REST API)

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/googleSlides/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto googleSlides <action>` (or read the source directly).

## Actions

- **addTextBox** — batchUpdate shapes to add a TEXT_BOX (createShape) at a position/size in EMU, fill it with text (insertText), and style the text (updateTextStyle — font, size, bold, color). Includes the EMU/PT unit rules.
- **applyBrandColors** — How brand colors work in the Slides API: rgbColor floats 0-1 (not 0-255, not hex), where they apply (text foreground, shape fill, table cell fill, page background), and the MileCell navy/teal palette pre-converted.
- **buildBrandedDeck** — The end-to-end recipe: read deck → plan per-slide requests → batch in chunks → verify. Ordering rules (createShape/createTable before insertText that targets them within the same batch), batch-size guidance, idempotency.
- **createDeck** — Create a new Google Slides presentation, or target an existing one by ID. Two creation paths (Slides API presentations.create, or Drive create_file). How to extract the presentationId from a Slides URL.
- **createSlide** — batchUpdate request shape to add a slide with a predefined layout (BLANK / TITLE / TITLE_AND_BODY / SECTION_HEADER etc.) at a chosen index, with a stable objectId you assign.
- **createTable** — batchUpdate request to create a table (rows × columns) on a slide, then fill cells with insertText (cellLocation), and color header rows / cells via updateTableCellProperties.
- **insertImageByUrl** — batchUpdate createImage request to insert an image from a PUBLIC URL onto a slide at a position/size. This is the path that BYPASSES the UI upload restriction (see refImageUploadBypass).
- **refAuth** — Auth setup (LOCKED): exactly what creds exist on this machine, what's missing, and the two paths to a presentations-scoped bearer token (OAuth refresh-token mint, or one-time installed-app consent). Includes the token-exchange wire shape.
- **refGotchas** — Reference: the gotchas that will bite a first-time Slides API deck builder — fields masks, all-or-nothing batches, objectId rules, EMU/PT/rgbColor units, image public-URL + aspect ratio, private-Drive image failure, non-idempotent re-runs, page-size geometry.
- **refImageUploadBypass** — LOCKED finding: the Slides API createImage path BYPASSES the UI 'You do not have access to upload images' restriction, because the API takes a public URL and Google fetches it server-side — there is no client-side upload to block.
- **refMileCellDeck** — Reference: the specific MileCell sales-proposal deck — target presentation URL, brand palette, the two content source files, slide inventory, and the slides that need generated images / screenshots inserted by URL.
- **refToolSurvey** — Reference: the 2026-06-04 survey of programmatic Google Slides tools (Slides API, md2googleslides + forks, Drive MCP, Apps Script, Presenton/gslides/FlashDocs) and why the raw Slides REST API won for our branded-deck case.
- **refVerify** — Verify-after-build discipline: GET the presentation to confirm slide/element counts and ids, AND open the live deck URL in CIC for a real visual check. Wire-success is necessary but not sufficient.
- **resolveSkillbook** — Overview for an executing agent: how to use this skill book to build a branded Google Slides deck end-to-end via the Slides REST API.
- **setBackground** — updatePageProperties request to set a slide's background to a solid brand color OR to a full-bleed stretched picture (by public URL). Used for navy divider slides and image backgrounds.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/googleSlides/index.js -->
```js
/**
 * googleSlides skill book — programmatically build branded decks that must live
 * NATIVELY in Google Slides, via the official Slides REST API (presentations.batchUpdate).
 *
 * ── WHEN THIS BOOK (vs the HTML default) ────────────────────────────────────
 * The DEFAULT deck substrate is HTML/CSS(/React) ([SB:slideDeckAndVideoGeneration],
 * LOCKED 2026-06-05) — best LLM-affinity and one substrate fans out to deck + video.
 * This book is the NARROW fallback for the case where the deliverable must be a
 * real, collaborative/shareable Google Slides file the recipient edits in Slides
 * (not an exported artifact). Reach for it ONLY then.
 *
 * ── WHY THIS EXISTS (the decision that forged it) ───────────────────────────
 * Within that Slides-native lane, hand-building a deck by clicking through the
 * Slides web UI is slow, brittle, and — on the MileCell account — partially
 * BLOCKED: the account throws "You do not have access to upload images" on the UI
 * image-upload path. A third-party Slides add-on inserted images fine because it
 * used the Slides *API* image path, not the UI uploader. This skill book
 * consolidates the best Slides-native method (the Slides API by raw fetch, zero
 * deps) and the LOCKED finding that the API path BYPASSES that UI upload restriction.
 *
 * ── CHOSEN METHOD (LOCKED 2026-06-04) ───────────────────────────────────────
 * Official Google Slides REST API, called by raw `fetch` against
 *   https://slides.googleapis.com/v1/presentations/{id}:batchUpdate
 * with a bearer access token minted from a Google OAuth refresh token (same
 * token-exchange pattern the merchant-center weapon already uses —
 * [SB:googleSlides->refAuth]). NO npm dependency required; the `googleapis`
 * Node SDK is OPTIONAL sugar, not installed in this repo.
 *
 * Rejected alternatives (surveyed 2026-06-04, see [SB:googleSlides->refToolSurvey]):
 *   - md2googleslides (googleworkspace) — ARCHIVED read-only 2025-11-18, 3 yrs
 *     unmaintained, doesn't run as-is. Forks (wescpy, Om3rr) exist but add a
 *     toolchain we'd have to vet; markdown loses the brand-color/background/
 *     table control we need. Not worth it for one branded deck.
 *   - Drive MCP (server 56b2b5db…) — CAN create an EMPTY presentation
 *     (create_file mimeType application/vnd.google-apps.presentation) but has
 *     NO batchUpdate / slide-editing tool. Useless for authoring content.
 *   - Apps Script — would require a bound script + manual auth; heavier than the
 *     REST API for a one-shot deck.
 *   - Presenton / gslides-maker / FlashDocs — third-party SaaS or thin wrappers;
 *     no advantage over calling the API directly, and add an external dependency.
 *
 * The API gives full control: createSlide, insertText, createTable, createImage
 * (by public URL), page background solid-color + stretched-picture-fill, brand
 * RGB colors on shapes/text/table cells. Everything we need for a branded deck.
 */

export const skillBook = {
  id: "googleSlides",
  title: "Google Slides — Programmatic Deck Builder (Slides REST API)",
  description:
    "Slides-NATIVE fallback (HTML is the default deck substrate — [SB:slideDeckAndVideoGeneration]). Use ONLY when the deliverable must be a real, collaborative/shareable Google Slides file. Builds/edits branded decks via the Slides API presentations.batchUpdate: slides, text, tables, images-by-URL, backgrounds, brand colors. Covers OAuth setup and the LOCKED finding that API image insert bypasses the UI 'cannot upload images' restriction.",
  steps: [],
  toc: {
    resolveSkillbook: {
      description:
        "Overview for an executing agent: how to use this skill book to build a branded Google Slides deck end-to-end via the Slides REST API.",
      howTo: `
**Default deck substrate is HTML ([SB:slideDeckAndVideoGeneration]); use THIS book
only when the deck must live natively in Google Slides (collaborative/shareable).**

This skill book is the wire-level + procedure layer for building branded Google
Slides decks WITHOUT clicking through the web UI.

**The one-paragraph version.** Mint a bearer token from a Google OAuth refresh
token that carries the \`presentations\` scope ([SB:googleSlides->refAuth]). Then
POST batches of \`requests\` to
\`https://slides.googleapis.com/v1/presentations/{presentationId}:batchUpdate\`.
Each request is one mutation: createSlide, createShape(TEXT_BOX)+insertText,
createTable, createImage(url), updatePageProperties(background),
updateShapeProperties / updateTextStyle (brand colors). Build the whole deck as
a sequence of batches; verify by reading the deck back
([SB:googleSlides->refVerify]) and by CIC-opening the live URL.

**Routing.**
- Auth + token mint → [SB:googleSlides->refAuth] (READ FIRST — nothing works without a scoped token).
- The single most important fact (image upload bypass) → [SB:googleSlides->refImageUploadBypass] (LOCKED).
- Create / open the deck file → [SB:googleSlides->createDeck].
- Per-element request shapes → [SB:googleSlides->createSlide], [SB:googleSlides->addTextBox], [SB:googleSlides->createTable], [SB:googleSlides->insertImageByUrl], [SB:googleSlides->setBackground], [SB:googleSlides->applyBrandColors].
- The full builder recipe (one deck, many batches) → [SB:googleSlides->buildBrandedDeck].
- The MileCell deck specifics (brand palette, content source files) → [SB:googleSlides->refMileCellDeck].
- Tool survey + why we picked the API → [SB:googleSlides->refToolSurvey].
- Gotchas that will bite you → [SB:googleSlides->refGotchas] (READ before first run).

**Read before you act.** Read \`refAuth\`, \`refImageUploadBypass\`, and
\`refGotchas\` once at session start so every subsequent batch has the constraints
in context.

**Positive-law note.** Per CLAUDE.md, all execution goes through a skill-book
howTo or a weapon. The runnable code for this book is the per-action request
shapes below, invoked via in-app imports or a CLI in THIS tree
(\`libs/skill_book/googleSlides/cli.mjs\` when added). Do NOT scatter ad-hoc
\`.mjs\`/\`.py\` deck-builder scripts elsewhere — extend this book or forge a
\`libs/weapon/googleSlides/\` weapon via blacksmith if a reusable runtime is
needed.
`,
    },

    refImageUploadBypass: {
      description:
        "LOCKED finding: the Slides API createImage path BYPASSES the UI 'You do not have access to upload images' restriction, because the API takes a public URL and Google fetches it server-side — there is no client-side upload to block.",
      howTo: `
**LOCKED (2026-06-04) — Slides API image insert BYPASSES the UI upload block.**

**The problem we hit.** On the MileCell Google account, the Slides web UI throws
"You do not have access to upload images" when you try Insert → Image → Upload.
That is an *upload* permission gate on the account — it blocks pushing image
BYTES from the browser into Google's storage.

**Why the API sidesteps it.** \`presentations.batchUpdate\` \`createImage\` (and the
\`stretchedPictureFill\` background path) do NOT upload bytes from the client. You
pass a **publicly accessible image URL**; Google's servers fetch the image ONCE
at insertion time and store a copy inside the presentation
(\`Image.source_url\`). There is no client upload step, so the account's
upload-permission gate never engages. The third-party add-on that "inserted
images fine via its API scope" worked for exactly this reason — it used this
same API path, not the UI uploader.

**Corroboration (inverse failure mode).** Public Google issue trackers show the
OPPOSITE complaint: the UI "Insert by URL" works while the *API* rejects
**private Drive** images ("Access to the provided image was forbidden",
issuetracker 150933939). That proves the API path is gated ONLY on the image URL
being publicly reachable — NOT on the account's upload permission. So:
  - Account-can't-upload (our case) → API insert WORKS (URL is public, no upload).
  - URL-is-private-Drive → API insert FAILS (regardless of upload permission).

**Operational rule.** Host every deck image at a PUBLIC URL first, then insert
via the API. We already have Supabase Storage public URLs and gpt-image-2
generation:
  1. Generate the image (gpt-image-2 via the openai_images weapon — see
     [SB:cjgeo->generateImage]).
  2. Upload to a PUBLIC bucket via the \`supabase_storage\` weapon; take the public
     URL (must be a clean https URL ≤ 2 KB, ending in a real image content-type).
  3. Insert via [SB:googleSlides->insertImageByUrl] or [SB:googleSlides->setBackground].

**Do NOT** waste time fighting the UI uploader, asking the user to "enable
uploads", or trying CIC drag-and-drop. The API path is the fix and it is already
unblocked for this account.

**Sources.**
- CreateImageRequest reference: https://developers.google.com/workspace/slides/api/reference/rest/v1/presentations/request (URL must be publicly accessible, ≤ 2 KB; fetched once at insertion, copy stored).
- Add-image guide: https://developers.google.com/workspace/slides/api/guides/add-image
- Private-Drive-forbidden (inverse case): https://issuetracker.google.com/issues/150933939 and https://github.com/googleapis/google-api-python-client/issues/2215
`,
    },

    refAuth: {
      description:
        "Auth setup (LOCKED): exactly what creds exist on this machine, what's missing, and the two paths to a presentations-scoped bearer token (OAuth refresh-token mint, or one-time installed-app consent). Includes the token-exchange wire shape.",
      howTo: `
**READ FIRST. No batchUpdate call works without a token carrying the
\`https://www.googleapis.com/auth/presentations\` scope.**

### What we have on THIS machine (audited 2026-06-04)

In \`C:/Users/xsj70/GuildOS/.env.local\` (and mirrored in profiles.env_vars via
[SB:_->getGoogleCredentials] at \`libs/council/profileEnvVars.js\`):
  - \`GOOGLE_ID\`            — OAuth client id (the GuildOS Google OAuth app).
  - \`GOOGLE_SECRET\`        — OAuth client secret for that app.
  - \`GOOGLE_REFRESH_TOKEN\` — refresh token from the NextAuth Google **login**
    flow. **Scope is almost certainly just \`openid email profile\`** — NOT
    Slides/Drive. Treat as INSUFFICIENT for Slides until proven otherwise.
  - \`GOOGLE_GMAIL_REFRESH_TOKEN\` — Gmail token, scoped \`gmail.readonly\` +
    \`gmail.modify\` only (see [SB:gmail->*] header). NOT Slides.
  - (No \`GOOGLE_SERVICE_ACCOUNT\` for Slides; the SA pattern in
    [SB:_->google_merchant_center] is content-API scoped, not Slides, and a SA
    can't edit a presentation owned by a human account unless that file is
    shared TO the service account or domain-wide delegation is configured.)

\`gcloud\` CLI is installed (v568) but has **no credentialed account** and no ADC
(\`gcloud auth list\` → "No credentialed accounts"; default project
\`boster-cbi\`). So gcloud ADC is NOT a ready path either.

### What's MISSING

A refresh token (or access token) minted under the GuildOS OAuth client
**WITH the \`presentations\` scope** (and, if you want the deck owned by the
acting user, \`drive.file\`). None of the stored tokens carry it. This is the one
gap to close.

### Path A — reuse the OAuth client, do a one-time consent for the Slides scope (RECOMMENDED)

We already own the OAuth app (\`GOOGLE_ID\`/\`GOOGLE_SECRET\`). Mint a Slides-scoped
refresh token once, store it (e.g. as \`GOOGLE_SLIDES_REFRESH_TOKEN\` in
profiles.env_vars / .env.local), then reuse forever via the token exchange
below.

One-time consent (installed-app / loopback flow):
  1. Ensure the OAuth client has \`http://localhost\` (or a loopback port) as an
     authorized redirect URI in Google Cloud Console → APIs & Services →
     Credentials. Ensure the **Google Slides API** (and Drive API if using
     \`drive.file\`) is ENABLED for the project the client belongs to.
  2. Open the consent URL in the user's browser (CIC ok), authorize as the
     MileCell/deck-owner account:
     \`\`\`
     https://accounts.google.com/o/oauth2/v2/auth?
       client_id=<GOOGLE_ID>
       &redirect_uri=http://localhost
       &response_type=code
       &access_type=offline
       &prompt=consent
       &scope=https://www.googleapis.com/auth/presentations%20https://www.googleapis.com/auth/drive.file
     \`\`\`
  3. Grab the \`code\` from the redirect URL, exchange it for tokens (one POST):
     \`\`\`
     POST https://oauth2.googleapis.com/token   (application/x-www-form-urlencoded)
       code=<code>
       client_id=<GOOGLE_ID>
       client_secret=<GOOGLE_SECRET>
       redirect_uri=http://localhost
       grant_type=authorization_code
     \`\`\`
     The response includes \`refresh_token\` (store it) and a short-lived
     \`access_token\`.

### The reusable token mint (refresh_token → access_token) — wire shape

This is the SAME pattern [SB:_->google_merchant_center] uses, minus the JWT (we
have a real refresh token, not a service account). Zero npm deps:
\`\`\`js
async function mintSlidesAccessToken({ clientId, clientSecret, refreshToken }) {
  const res = await fetch("https://oauth2.googleapis.com/token", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      client_id: clientId,
      client_secret: clientSecret,
      refresh_token: refreshToken,
      grant_type: "refresh_token",
    }),
  });
  if (!res.ok) throw new Error(\`token mint failed (\${res.status}): \${await res.text()}\`);
  const { access_token } = await res.json();      // ~1h lifetime
  return access_token;                            // -> Authorization: Bearer <access_token>
}
\`\`\`
Pull creds via \`getGoogleCredentials(userId)\` (\`libs/council/profileEnvVars.js\`),
but note that helper currently returns Gmail-or-generic refresh token; for Slides
add a dedicated \`GOOGLE_SLIDES_REFRESH_TOKEN\` and read it explicitly so you don't
accidentally use the login-scoped token.

### Path B — service account (only if a SA is provisioned + the deck is shared to it)

Mint via JWT exactly like [SB:_->google_merchant_center]'s \`getAccessToken\`, but
\`scope=https://www.googleapis.com/auth/presentations\`. The SA can only edit decks
it OWNS or that are explicitly shared to its \`client_email\` (or via domain-wide
delegation). For a deck living in a personal Google account, share the file to
the SA email first. Heavier than Path A for a one-off; prefer A.

**Never echo \`GOOGLE_SECRET\` / refresh tokens / access tokens to stdout.** Mint
in-process and pass as a Bearer header only.
`,
    },

    createDeck: {
      description:
        "Create a new Google Slides presentation, or target an existing one by ID. Two creation paths (Slides API presentations.create, or Drive create_file). How to extract the presentationId from a Slides URL.",
      howTo: `
**Targeting an existing deck (our MileCell case).** The presentation already
exists. Extract the \`presentationId\` from the URL — it is the path segment
between \`/d/\` and \`/edit\`:
  \`https://docs.google.com/presentation/d/<PRESENTATION_ID>/edit\`
For MileCell: \`1WKTP6eKmb5CUFEemznJqjcTTLRJfgiJlfIcU1xFn26E\`. All batchUpdate
calls hit \`/v1/presentations/<PRESENTATION_ID>:batchUpdate\`.

**Reading the current structure first (ALWAYS do this before mutating an
existing deck).** GET the presentation to learn slide objectIds, existing
placeholder shapes, and the page size (needed to compute full-bleed image
geometry — see [SB:googleSlides->refGotchas]):
  \`\`\`
  GET https://slides.googleapis.com/v1/presentations/<PRESENTATION_ID>
      Authorization: Bearer <access_token>
  \`\`\`
Returns \`{ presentationId, pageSize:{width,height}, slides:[{objectId, pageElements:[…]}], layouts, masters }\`. Pin \`pageSize\` (usually 9144000 × 5143500 EMU = 10"×5.63" widescreen 16:9).

**Creating a NEW deck — two paths:**
- **Slides API (preferred, gives you the id directly):**
  \`\`\`
  POST https://slides.googleapis.com/v1/presentations
      Authorization: Bearer <access_token>
      { "title": "MileCell Digital Commerce Platform — Proposal" }
  \`\`\`
  Returns the full presentation incl. \`presentationId\` and a default first slide.
- **Drive MCP (only if you specifically want it in a Drive folder via the
  connected Drive server):** \`create_file\` with
  \`contentMimeType: "application/vnd.google-apps.presentation"\`, \`title: "…"\`.
  This makes an EMPTY deck and returns the file id (= presentationId). **The
  Drive MCP CANNOT edit slides** — switch to the Slides API for all content.

**Default-slide cleanup.** \`presentations.create\` seeds one blank slide. If you
build your own title slide, delete the seeded one with a \`deleteObject\` request
referencing its objectId (read it from the create response). Same for the
MileCell deck's "blank slide 1" per the prompt-pack FINAL CLEANUP note.
`,
    },

    createSlide: {
      description:
        "batchUpdate request shape to add a slide with a predefined layout (BLANK / TITLE / TITLE_AND_BODY / SECTION_HEADER etc.) at a chosen index, with a stable objectId you assign.",
      howTo: `
**\`createSlide\` request** — keys: \`objectId\`, \`insertionIndex\`,
\`slideLayoutReference: { predefinedLayout }\`.
- \`objectId\` — OPTIONAL but ASSIGN IT (id rules → [SB:googleSlides->refGotchas]).
  Assigning your own id lets you target the slide in the SAME batch for
  createShape/createImage without a round-trip to read back a server-generated id.
- \`insertionIndex\` — 0-based position. Omit to append at the end.
- \`predefinedLayout\` — common values: \`BLANK\` (build everything yourself —
  recommended for full brand control), \`TITLE\`, \`TITLE_AND_BODY\`,
  \`SECTION_HEADER\` (for divider slides like MileCell slide 8/14),
  \`TITLE_AND_TWO_COLUMNS\`, \`CAPTION_ONLY\`. With non-BLANK layouts the slide gets
  placeholder shapes; to fill them, read the created slide's pageElements to find
  the placeholder objectIds, then insertText into them. For maximum control over
  a branded deck, prefer \`BLANK\` + your own createShape text boxes.

**Branded-deck tip.** Use \`BLANK\` everywhere and lay out your own text boxes; it
avoids fighting layout-master placeholder styling and gives pixel control over
the navy/teal palette ([SB:googleSlides->applyBrandColors]).
`,
    },

    addTextBox: {
      description:
        "batchUpdate shapes to add a TEXT_BOX (createShape) at a position/size in EMU, fill it with text (insertText), and style the text (updateTextStyle — font, size, bold, color). Includes the EMU/PT unit rules.",
      howTo: `
**Three requests, in order, in the SAME batch (createShape → insertText → style):**

1. **createShape** — \`shapeType: "TEXT_BOX"\`, \`elementProperties: { pageObjectId,
   size:{width,height}, transform:{scaleX,scaleY,translateX,translateY,unit} }\`,
   plus your assigned \`objectId\`. \`translateX/Y\` = top-left corner offset from the
   page origin.
2. **insertText** — \`{ objectId, insertionIndex: 0, text }\` against the shape's id.
3. **updateTextStyle** — \`{ objectId, textRange:{type:"ALL"}, style, fields }\`.
   Style props: \`fontFamily\`, \`fontSize:{magnitude,unit:"PT"}\`, \`bold\`,
   \`foregroundColor.opaqueColor.rgbColor\` (color shapes →
   [SB:googleSlides->applyBrandColors]).

**Paragraph alignment / spacing** uses \`updateParagraphStyle\` with
\`{ alignment: "CENTER" | "START" | "END", ... }\` + a \`textRange\` and the same
\`fields\` mask discipline.

**Units, the \`fields\` mask, and same-batch ordering are LOCKED in
[SB:googleSlides->refGotchas] — read it once; not repeated here.**
`,
    },

    createTable: {
      description:
        "batchUpdate request to create a table (rows × columns) on a slide, then fill cells with insertText (cellLocation), and color header rows / cells via updateTableCellProperties.",
      howTo: `
**1. createTable** — \`{ objectId, elementProperties:{ pageObjectId, size,
   transform }, rows, columns }\`. Creates an empty grid; the API auto-sizes
   columns evenly across the given width.

**2. insertText into each cell** — \`{ objectId, cellLocation:{ rowIndex,
   columnIndex }, insertionIndex: 0, text }\` (0-based row/col). One insertText per
   cell; build them all in the same batch.

**3. Style a header row / cells** — \`updateTableCellProperties\` with a
   \`tableRange: { location:{rowIndex,columnIndex}, rowSpan, columnSpan }\` and
   \`tableCellProperties.tableCellBackgroundFill.solidFill.color\` for the fill (e.g.
   navy header row); then \`updateTextStyle\` the header cells to white
   (\`foregroundColor\` rgbColor 1,1,1). Colors → [SB:googleSlides->applyBrandColors];
   units/fields-mask → [SB:googleSlides->refGotchas].

**Gotcha.** Column widths default to equal. To set explicit widths use
\`updateTableColumnProperties\` with a \`columnIndices\` list + \`tableColumnProperties.columnWidth\`.
Row height grows with content automatically. For the dense MileCell RFQ table
(slide 9, ~14 rows), keep font small (9–11 PT) and consider splitting across two
slides as the outline notes (slides 10–11).
`,
    },

    insertImageByUrl: {
      description:
        "batchUpdate createImage request to insert an image from a PUBLIC URL onto a slide at a position/size. This is the path that BYPASSES the UI upload restriction (see refImageUploadBypass).",
      howTo: `
**This is the upload-bypass path — see [SB:googleSlides->refImageUploadBypass]
(LOCKED).** The URL must be PUBLIC, ≤ 2 KB, PNG/JPEG/GIF, < 50 MB, ≤ 25 MP.
Google fetches it server-side once at insertion.

**\`createImage\` request** — \`{ objectId, url, elementProperties:{ pageObjectId,
size, transform } }\` (same EMU geometry shape as createShape →
[SB:googleSlides->refGotchas]).
- Maintain aspect ratio yourself in \`size\` (the API will stretch to the box; it
  does NOT auto-fit). Compute height = width × (imgH / imgW).
- After insert, \`Image.source_url\` holds the URL you passed. The displayed image
  is a stored COPY — if the source URL later 404s, the slide keeps working.

**Image sourcing pipeline (for generated assets):**
  1. Generate via gpt-image-2 ([SB:cjgeo->generateImage]).
  2. Upload to a PUBLIC Supabase bucket via the \`supabase_storage\` weapon; get
     the public URL (\`readPublicUrl\`).
  3. createImage with that URL.

**Screenshots (e.g. MileCell slide 13 v0 hepatocyte demo):** capture via CIC,
upload the PNG to the public bucket, then createImage by URL — do NOT try to
upload it through the Slides UI (blocked on this account).

**Gotcha — private Drive URLs FAIL.** A \`drive.google.com\` link to a
non-public file returns "Access to the provided image was forbidden". Either
make the Drive file public (anyone-with-link viewer) and use a direct-download
form, or host on Supabase. Supabase public URLs are the reliable path here.
`,
    },

    setBackground: {
      description:
        "updatePageProperties request to set a slide's background to a solid brand color OR to a full-bleed stretched picture (by public URL). Used for navy divider slides and image backgrounds.",
      howTo: `
Both use one **\`updatePageProperties\`** request — \`{ objectId, pageProperties:{
pageBackgroundFill }, fields }\`:

- **Solid color** (e.g. MileCell navy divider slides 8, 14, 26) →
  \`pageBackgroundFill.solidFill.color.rgbColor\`; \`fields:
  "pageBackgroundFill.solidFill.color"\`.
- **Full-bleed image** → \`pageBackgroundFill.stretchedPictureFill.contentUrl\`
  (public URL); \`fields: "pageBackgroundFill.stretchedPictureFill.contentUrl"\`.
  \`contentUrl\` follows the SAME public-URL rule as createImage and BYPASSES the UI
  upload block ([SB:googleSlides->refImageUploadBypass]). The background stretches
  to the full page; size the source image to the deck aspect ratio (16:9) to avoid
  distortion.

- For a navy slide with white text, set the background first, then add white text
  boxes ([SB:googleSlides->addTextBox] with foregroundColor 1,1,1).

The \`fields\` mask is mandatory (→ [SB:googleSlides->refGotchas]) — without it the
update is rejected.
`,
    },

    applyBrandColors: {
      description:
        "How brand colors work in the Slides API: rgbColor floats 0-1 (not 0-255, not hex), where they apply (text foreground, shape fill, table cell fill, page background), and the MileCell navy/teal palette pre-converted.",
      howTo: `
**Color model.** rgbColor channels are FLOATS in [0,1] (LOCKED in
[SB:googleSlides->refGotchas] item 4 — not repeated here). There is also
\`themeColor\` (e.g. "DARK1","LIGHT2","ACCENT1") referencing the master's palette;
for a fully controlled brand look use explicit \`rgbColor\` rather than theme colors
so you don't inherit unexpected master styling.

**Where color applies (all wrapped in \`{ "rgbColor": {...} }\`):**
  - Text:           \`updateTextStyle\` → \`style.foregroundColor.opaqueColor.rgbColor\`
  - Text highlight: \`updateTextStyle\` → \`style.backgroundColor.opaqueColor.rgbColor\`
  - Shape fill:     \`updateShapeProperties\` → \`shapeProperties.shapeBackgroundFill.solidFill.color.rgbColor\`
  - Shape outline:  \`updateShapeProperties\` → \`shapeProperties.outline.outlineFill.solidFill.color.rgbColor\`
  - Table cell:     \`updateTableCellProperties\` → \`tableCellProperties.tableCellBackgroundFill.solidFill.color.rgbColor\`
  - Page bg:        \`updatePageProperties\` → \`pageProperties.pageBackgroundFill.solidFill.color.rgbColor\`
  Each REQUIRES a \`fields\` mask naming the exact dotted path.

**MileCell brand palette (from milecell-bio.com / the prompt pack), pre-converted:**
  - Deep navy  \`#1e3a5f\`  → { red: 0.118, green: 0.227, blue: 0.373 }   (headings, divider backgrounds, table header fill)
  - White      \`#ffffff\`  → { red: 1, green: 1, blue: 1 }               (text on navy)
  - Teal/aqua accent \`#2dd4bf\` (illustrative — confirm exact hex from the site) → { red: 0.176, green: 0.831, blue: 0.749 }   (accents, underlines, chips)
  - Amber accent (slide 7 "get it wrong") \`#f59e0b\` → { red: 0.961, green: 0.620, blue: 0.043 }
  - Body text dark \`#1f2937\` → { red: 0.122, green: 0.161, blue: 0.220 }
  Confirm exact teal hex by sampling milecell-bio.com before final polish;
  adjust the float triple accordingly (hex→float = each pair /255).

**Brand-consistency recipe per slide:** navy headings, dark-gray body, teal
accent shapes/underlines (createShape RECTANGLE/LINE filled teal), white
backgrounds for content slides, full-navy backgrounds for section dividers
(slides 8, 14, 26) with white text. Hexagon/molecular motif is best delivered as
a generated PNG inserted via [SB:googleSlides->insertImageByUrl] rather than
drawn with native shapes.
`,
    },

    buildBrandedDeck: {
      description:
        "The end-to-end recipe: read deck → plan per-slide requests → batch in chunks → verify. Ordering rules (createShape/createTable before insertText that targets them within the same batch), batch-size guidance, idempotency.",
      howTo: `
**The full builder loop for a branded deck (e.g. MileCell, 26 slides):**

0. **Auth.** Mint a presentations-scoped access token ([SB:googleSlides->refAuth]).
1. **Read.** GET the presentation; pin \`presentationId\`, \`pageSize\`, existing
   slide objectIds ([SB:googleSlides->createDeck]).
2. **Plan per slide.** For each slide, assemble an ORDERED request list. Order
   within a batch matters — a request can reference an object created EARLIER in
   the SAME batch:
     a. \`createSlide\` (assign objectId) — or target an existing slide.
     b. \`updatePageProperties\` for the background (do early so text sits on top).
     c. \`createShape\` / \`createTable\` / \`createImage\` (assign objectIds).
     d. \`insertText\` into shapes/cells created in (c).
     e. \`updateTextStyle\` / \`updateParagraphStyle\` / \`updateShapeProperties\` /
        \`updateTableCellProperties\` for colors/fonts ([SB:googleSlides->applyBrandColors]).
   Object ids MUST be assigned in the create request to be referenceable later in
   the same batch (you cannot reference a server-generated id mid-batch).
3. **Batch in chunks.** One batchUpdate can carry many requests, but keep
   batches per-slide or per-few-slides (~50–150 requests) so a single bad request
   doesn't roll back the whole deck (batchUpdate is ALL-OR-NOTHING — any invalid
   request fails the entire batch and applies nothing). Smaller batches = easier
   to locate the offending request from the error.
     \`\`\`
     POST https://slides.googleapis.com/v1/presentations/<ID>:batchUpdate
         Authorization: Bearer <token>
         { "requests": [ ...ordered requests for this chunk... ] }
     \`\`\`
   The response returns \`replies[]\` aligned to your requests (createSlide/
   createShape replies include the resulting objectId — useful when you let the
   server assign ids).
4. **Verify** ([SB:googleSlides->refVerify]) — read back + CIC visual check.

**Idempotency / re-runs.** batchUpdate is NOT idempotent — re-running the same
createShape batch ADDS duplicate shapes. For a re-buildable pipeline either (a)
delete + recreate target slides each run (\`deleteObject\` on the slide objectId,
then recreate), or (b) make object ids deterministic and \`deleteObject\` them
first. Don't blindly re-run an append batch against a deck that already has the
content.

**Token lifetime.** Access tokens last ~1h. For a long build, mint once at the
start; if you hit a 401 mid-build, re-mint and continue from the next chunk.

**Content source for MileCell.** Slide copy is fully written in
\`C:/Users/xsj70/GuildOS/milecell-slide-prompt-pack.md\` (plain-language,
brand-baked) and \`milecell-proposal-outline.md\` (structure + tables + the RFQ
matrix + pricing tables). Treat the prompt-pack as the per-slide content of
record; the outline holds the exact table rows for slides 9/23/24. See
[SB:googleSlides->refMileCellDeck].
`,
    },

    refVerify: {
      description:
        "Verify-after-build discipline: GET the presentation to confirm slide/element counts and ids, AND open the live deck URL in CIC for a real visual check. Wire-success is necessary but not sufficient.",
      howTo: `
**Two-layer verify (both required before claiming the deck is done):**

1. **Structural (programmatic).** GET
   \`https://slides.googleapis.com/v1/presentations/<ID>\` and confirm:
     - slide count matches the plan,
     - each expected slide objectId exists,
     - each slide's \`pageElements\` contains the shapes/tables/images you created
       (count + types),
     - images show a non-empty \`image.contentUrl\` (Google fetched & stored the
       copy — confirms the upload-bypass worked).
   batchUpdate replies returning 200 with replies[] is necessary but NOT
   sufficient — a 200 can still produce a visually broken slide (overlapping text
   boxes, off-page transforms, distorted images).

2. **Visual (USER channel, LOCKED per success-gate).** Open the live deck in CIC:
   \`https://docs.google.com/presentation/d/<ID>/edit\` (or .../present), screenshot
   the real Chrome tab via \`mcp__computer-use__screenshot\` / CIC computer
   screenshot — NOT html2canvas, NOT a synthetic render. Eyeball: brand colors
   correct, text fits its box, images rendered (no broken-image placeholder),
   tables readable, navy dividers full-bleed. Only after this passes is the deck
   SHIP-able.

**Where to review (state it to the user):** the live Slides URL above. That is
the artifact path; the worktree/API response is not.
`,
    },

    refToolSurvey: {
      description:
        "Reference: the 2026-06-04 survey of programmatic Google Slides tools (Slides API, md2googleslides + forks, Drive MCP, Apps Script, Presenton/gslides/FlashDocs) and why the raw Slides REST API won for our branded-deck case.",
      howTo: `
**Surveyed 2026-06-04. Verdict: official Slides REST API (raw fetch), no deps.**

| Tool | Create slides | Text | Tables | Image by URL | Background | Brand RGB | Auth | Health | Verdict |
|---|---|---|---|---|---|---|---|---|---|
| **Slides REST API** (this book) | ✅ | ✅ | ✅ | ✅ (public URL, bypasses UI block) | ✅ solid + picture | ✅ rgbColor floats | OAuth presentations scope | Official, fully maintained | **CHOSEN** |
| googleapis Node SDK | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | same OAuth | Maintained, but adds a dep not installed here | Optional sugar over the same API |
| md2googleslides (googleworkspace) | ✅ (markdown) | ✅ | partial | ✅ (it calls the API) | limited | limited (markdown styling only) | OAuth | **ARCHIVED read-only 2025-11-18, 3 yrs unmaintained, broken as-is** | Rejected |
| md2googleslides forks (wescpy / Om3rr "markdown-to-googleslides", incl. MCP) | ✅ | ✅ | partial | ✅ | limited | limited | OAuth | Community forks, Node 22+, unvetted | Rejected (loses brand control; extra toolchain) |
| **Drive MCP** (server 56b2b5db…) | ⬚ empty deck only (create_file presentation mime) | ❌ | ❌ | ❌ | ❌ | ❌ | the connected MCP creds | Maintained but Drive-only | Rejected for authoring (no batchUpdate tool) |
| Apps Script (SlidesApp) | ✅ | ✅ | ✅ | ✅ | ✅ | bound-script auth | Maintained | Rejected (heavier setup for one-shot) |
| Presenton / gslides-maker / FlashDocs | varies | varies | varies | varies | varies | varies | external SaaS / wrappers | varies | Rejected (external dep, no edge over direct API) |

**Why the raw REST API (not the Node SDK):** \`googleapis\` is not installed in
this repo (checked package.json + node_modules). The API is plain JSON over
HTTPS; the token mint is the same fetch-based pattern the merchant-center weapon
already uses. Zero new deps, full control over brand colors / backgrounds /
image-by-URL — which is exactly what a branded deck needs and what md2gslides
gives up. If a richer runtime is later wanted, \`npm i googleapis\` and the same
request shapes pass straight through \`slides.presentations.batchUpdate({...})\`.

**Sources.**
- Slides API overview: https://developers.google.com/workspace/slides/api/guides/overview
- batchUpdate reference: https://developers.google.com/workspace/slides/api/reference/rest/v1/presentations/batchUpdate
- Scopes: https://developers.google.com/workspace/slides/api/scopes
- md2googleslides (archived): https://github.com/googleworkspace/md2googleslides ; fork https://github.com/Om3rr/markdown-to-googleslides
- 2025 dev guide (corroborates image/format limits): https://www.flashdocs.com/post/google-slides-api-comprehensive-guide-for-developers
`,
    },

    refGotchas: {
      description:
        "Reference: the gotchas that will bite a first-time Slides API deck builder — fields masks, all-or-nothing batches, objectId rules, EMU/PT/rgbColor units, image public-URL + aspect ratio, private-Drive image failure, non-idempotent re-runs, page-size geometry.",
      howTo: `
**Read before the first batch. Each of these has burned someone.**

1. **\`fields\` mask is MANDATORY on every \`update*\` request.** Omit it → request
   rejected. List exactly the dotted paths you set (e.g.
   \`"shapeBackgroundFill.solidFill.color"\`). A path in the mask but absent from
   the body CLEARS that property.

2. **batchUpdate is ALL-OR-NOTHING.** One invalid request fails the WHOLE batch
   and applies nothing. Keep batches small (~per slide) so the error points at a
   narrow set. The error message names the offending request index.

3. **objectId rules.** 5–50 chars, \`[a-zA-Z0-9_-]\`, must start with a letter or
   underscore, UNIQUE per presentation. Assign ids on create so you can target
   the object later IN THE SAME BATCH (you cannot reference a server-assigned id
   mid-batch). Reusing an existing id on create → error.

4. **Units.** Geometry (size/transform) → EMU (914400/in, 12700/pt) or PT. Font
   size → PT only. Colors → rgbColor floats 0–1 (hex pair ÷ 255), NOT 0–255, NOT
   hex strings. Mixing these up is the #1 silent-looking-but-wrong bug.

5. **Images: public URL only, ≤ 2 KB, PNG/JPEG/GIF, < 50 MB, ≤ 25 MP.** Google
   fetches server-side at insertion (this is the UI-upload BYPASS —
   [SB:googleSlides->refImageUploadBypass]). The API does NOT preserve aspect
   ratio — compute height = width × imgH/imgW yourself or the image stretches.

6. **Private Drive images FAIL** ("Access to the provided image was forbidden")
   even with the right OAuth scope, because the fetch is anonymous-ish. Host on
   a PUBLIC bucket (Supabase) — don't fight Drive sharing.

7. **Re-runs are NOT idempotent.** Re-running an append/create batch DUPLICATES
   elements. Delete (deleteObject) + recreate, or use deterministic ids you
   delete first.

8. **Page geometry.** Read \`pageSize\` (widescreen default 9144000 × 5143500 EMU).
   Full-bleed = translate (0,0), size = full pageSize. Elements with off-page
   transforms render invisibly but succeed at the wire — verify visually.

9. **Drive MCP can't author.** The connected Drive server (56b2b5db…) has no
   batchUpdate tool; it can only create an empty presentation / read metadata.
   Don't waste a turn looking for a slides-edit tool there.

10. **Token scope mismatch is silent until the call.** A 403 \`PERMISSION_DENIED\`
    or \`insufficient authentication scopes\` on batchUpdate almost always means the
    token lacks \`presentations\` (e.g. you used the NextAuth login token). Re-mint
    with the right scope ([SB:googleSlides->refAuth]).

11. **Enable the API.** The Slides API must be ENABLED in the Cloud project the
    OAuth client belongs to, else token-valid calls 403. Drive API too if using
    \`drive.file\`.
`,
    },

    refMileCellDeck: {
      description:
        "Reference: the specific MileCell sales-proposal deck — target presentation URL, brand palette, the two content source files, slide inventory, and the slides that need generated images / screenshots inserted by URL.",
      howTo: `
**Target deck.**
\`https://docs.google.com/presentation/d/1WKTP6eKmb5CUFEemznJqjcTTLRJfgiJlfIcU1xFn26E/edit\`
→ presentationId \`1WKTP6eKmb5CUFEemznJqjcTTLRJfgiJlfIcU1xFn26E\`.

**Audience / tone.** Wayne Vaz (CCO) + Nianwei Lin (CEO) — commercial buyers;
revenue/pipeline/ROI language, plain (no internal codenames). 26 slides; delete
the blank slide 1 on final cleanup.

**Content sources (records of truth), brand palette, image slides:**
  - Per-slide copy → \`C:/Users/xsj70/GuildOS/milecell-slide-prompt-pack.md\`;
    structure + exact table rows (RFQ matrix, pricing tiers) →
    \`C:/Users/xsj70/GuildOS/milecell-proposal-outline.md\`.
  - Brand palette → [SB:googleSlides->applyBrandColors] (navy #1e3a5f, white, teal,
    amber). Match milecell-bio.com.
  - Slides needing images via the upload-bypass path: the v0 hepatocyte demo-page
    screenshot (capture via CIC) and any generated motif/chart graphics (gpt-image-2,
    [SB:cjgeo->generateImage]). Each → public Supabase bucket → createImage /
    setBackground by URL. All other slides are native text boxes + tables + solid
    backgrounds.

**Build via [SB:googleSlides->buildBrandedDeck]; verify via
[SB:googleSlides->refVerify] (CIC on the live URL).**
`,
    },
  },
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->
