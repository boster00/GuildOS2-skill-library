# browsercontrol

<!-- GUILDOS-SKILL name="browsercontrol" -->

<!-- GUILDOS-DESC -->
Browser work + VISUAL VERIFICATION, including screenshotting AUTH-GATED / LOGGED-IN pages (Gmail, the GuildOS app, any account the operator is signed into in their MAIN Chrome). Take a screenshot via a Playwright browser logged into the operator's accounts — the way a headless leaf visually verifies a page behind a login. Local Claude uses Claude-in-Chrome MCP; cloud agents use their VM's native Playwright; headless quest LEAVES (which cannot use CIC) drive/screenshot via the leafBrowser weapon — Playwright running system Chrome on the operator's logged-in profile.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/browsercontrol/SKILL.md -->
```markdown
---
name: browsercontrol
description: >-
  Browser work + VISUAL VERIFICATION, including screenshotting AUTH-GATED /
  LOGGED-IN pages (Gmail, the GuildOS app, any account the operator is signed
  into in their MAIN Chrome). Take a screenshot via a Playwright browser logged
  into the operator's accounts — the way a headless leaf visually verifies a page
  behind a login. Local Claude uses Claude-in-Chrome MCP; cloud agents use their
  VM's native Playwright; headless quest LEAVES (which cannot use CIC)
  drive/screenshot via the leafBrowser weapon — Playwright running system Chrome
  on the operator's logged-in profile.
---

# Browser control

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/browsercontrol/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto browsercontrol <action>` (or read the source directly).

## Actions

- **accessChromeExtensionPage** — Navigate CIC to chrome://extensions/ (reload an extension, toggle dev mode, etc.).
- **captureAuth** — Export a storageState JSON that cloud agents can import as Playwright auth context.
- **leafBrowserControl** — VISUAL VERIFICATION for headless leaves: TAKE A SCREENSHOT via a Playwright browser LOGGED INTO THE OPERATOR'S ACCOUNTS (the logins from their MAIN Chrome) — so a leaf can screenshot auth-gated pages (Gmail, the GuildOS app) that a logged-out browser bounces to a sign-in. Launches Playwright MCP (mcp__playwright__*) driving system Chrome on the operator's profile. TWO auth profiles: `master` (cookies COPIED from the main browser — fine for non-rotating sites like GuildOS/Supabase) and `interactive` (a native login the operator did inside the automation browser, reused DIRECTLY — REQUIRED for Google/Gmail, which reject copied cookies with a passkey wall). If a screenshot comes back as a sign-in page, the session is stale → see updateLogins. NOT for local Claude (that's validateCIC/CIC).
- **updateLogins** — UPDATE / REFRESH the operator's captured browser logins when they go stale or outdated — symptom: a leaf or Playwright screenshot of an auth-gated page (Gmail, the GuildOS app) comes back as a SIGN-IN / account-chooser page instead of the content. Refreshes either the `master` cookie copy (GuildOS/Supabase, non-Google) or the `interactive` native-login profile (Google/Gmail). Operator-gated for the actual sign-in; the agent sets it up and verifies.
- **validateCIC** — HARD PRECONDITION GATE: prove Claude-in-Chrome can really screenshot the GuildOS app before ANY verification-dependent work. Server-up check (cicHealth weapon) → CIC open localhost:3002 → real screenshot → if NOT obtained, STOP and escalate 'CIC not working, needs unblocking' (never dance, never substitute a proxy). Run this whenever a step depends on user-channel CIC visual verification.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/browsercontrol/engineering_decisions.md -->
```markdown
# browsercontrol — Engineering Decisions

> WHY the browser-control routing is shaped this way. Drift-prevention only —
> the procedures live in the skill book `toc`/howTos, the mechanism in
> `libs/weapon/leafBrowser/`.

## 2026-06-15 · Leaf agents get webpage control via `leafBrowser` (Playwright + logged-in-profile copy), NOT CIC and NOT a storageState bundle

**Decision.** A headless quest-execution leaf that must drive a real webpage uses
the `leafBrowser` weapon as its single path: Playwright MCP injected per-invocation
via `claude --print --mcp-config`, pointed at a per-leaf **copy of the operator's
logged-in Chrome profile** (`--user-data-dir <copy> --executable-path <system
Chrome> --headless`). Tagged `[BROWSER]`, gated on `hasMasterProfile()`, wired
through `questTick` execute-dispatch → `claudecli.invoke({mcpConfigPath})` →
`cleanupLeafBrowser`.

**Rejected alternatives.**
1. *Give leaves Claude-in-Chrome (CIC)* — the intuitive choice, since local Claude
   uses CIC. **Impossible:** CIC rides Chrome Native Messaging, binds to ONE
   interactive session, and is undeclarable in any MCP config — a headless
   `claude --print` leaf can never inherit it (verified; see memory
   `reference_cic_not_mcp_leaf_browser`). The dead `runChromeVerification` in
   `libs/adventurer_runtime/server.js` is the cautionary artifact: it spawns a leaf
   with `--allowedTools mcp__Claude_in_Chrome__*` and silently returns null forever.
2. *storageState auth bundle* (`localwork.mountAuthBundle`, `captureAuth`) — replays
   cookies + localStorage into a clean Playwright context. Kept for **cloud Cursor
   agents**, but rejected for leaves: a full-profile copy carries the live logged-in
   session, so auth-walled pages work where a bundle often does not.

**Why.** Only Playwright MCP is config-declarable and injectable per-leaf
(`--mcp-config`); leaves provably inherit MCP servers this way (railway tools work
inside leaves). Full-profile-copy is the only variant that makes auth-walled targets
work headless. Chrome locks a user-data-dir, hence a per-leaf copy (fan-out-safe).

**Consequence if violated.** Wiring leaves to CIC reintroduces a silent no-op (the
`runChromeVerification` failure mode): `[BROWSER]` quests appear to dispatch but
never actually verify anything. Switching to storageState breaks auth-walled targets.

**Stability caveat (state as of 2026-06-15, verify before relying):** the path is
DORMANT until the operator runs `node libs/weapon/leafBrowser/cli.mjs capture` once
(`hasMasterProfile()` is currently `false` on the main box → no-op). It has not yet
been exercised by a real end-to-end leaf run, so "stable" is the design intent, not a
proven fact — confirm one real leaf screenshot of an auth-walled page before treating
it as production-proven. `npx -y @playwright/mcp@latest` fetches latest per leaf
(first run slow + network-dependent). It drives SYSTEM Chrome via `--executable-path`
with Chrome's own `--headless`, so it does NOT depend on Playwright's bundled
Chromium or the absent win-arm64 headless-shell.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/browsercontrol/index.js -->
```js
/**
 * Browser control skill book.
 *
 * ── LOCAL CLAUDE: use Claude-in-Chrome (CIC), always ────────────────────────
 * When Claude runs locally (Guildmaster / PA / Claude Code CLI) and needs to
 * browse, use the Claude-in-Chrome MCP extension tools (`mcp__Claude_in_Chrome__*`).
 * CIC is the ONLY local browser control path. It opens a dedicated tab group per
 * objective, drives the user's real Chrome, and follows an observe→act→observe loop.
 *
 *   - Start with `tabs_context_mcp` (createIfEmpty: true) to get/seed the tab group.
 *   - Open a NEW tab per objective with `tabs_create_mcp` — never reuse a tab
 *     from a previous objective.
 *   - After every action, screenshot and `read_page`/`find` before the next step.
 *     No batch scripts.
 *
 * ── CLOUD CURSOR AGENTS: use the VM's native UI ─────────────────────────────
 * Cursor cloud agents run on a Linux VM with its own headed Chrome and
 * Playwright tooling. They drive the browser natively — do NOT dispatch CIC
 * commands from cloud agents, and do NOT instruct them to reach a local CDP
 * endpoint (port 9222 is not reachable from the cloud). Let them use their
 * built-in controls.
 *
 * ── HEADLESS / leaf / in-process browser work: use leafBrowser (the ONE substrate) ─
 * leafBrowser is the SINGLE owner of all Playwright/Chromium launching in the
 * repo (locked 2026-06-15 — see its engineering_decisions.md). Two surfaces:
 *   1. MCP injection for LEAVES — quest-execution leaves are spawned HEADLESS
 *      (`claude --print` via claudecli.invoke) and CANNOT use CIC (CIC rides
 *      Chrome Native Messaging, binds to ONE interactive session, undeclarable in
 *      MCP config). Their only path to DRIVE / SCREENSHOT a page is
 *      `prepareLeafBrowser`: Playwright MCP via `--mcp-config` driving system Chrome
 *      on the operator's logged-in profile (auth-walled pages render). PROVEN
 *      2026-06-18 — a leaf screenshotted the operator's authenticated Gmail inbox.
 *      Two profiles: a COPY of `master` (cookies from the main browser; non-Google),
 *      or the `interactive` native-login profile reused DIRECTLY (`directProfileDir`
 *      — REQUIRED for Google/Gmail). Logins come from `captureMasterFromChrome`
 *      (auto, master) or `cli.mjs login` (interactive); refresh via `updateLogins`.
 *      See `leafBrowserControl` below.
 *   2. In-process render-to-PNG — `leafBrowser.screenshotUrl` / `launchContext`
 *      is THE way to turn a URL/ref into a verify screenshot with no LLM in the
 *      loop. questPurrview.autoVisualVerify, cicVerify, and poster_render all
 *      delegate here. Isolated (no auth) by default — needs no captured profile —
 *      so it works TODAY. NEVER re-add a local `chromium.launch`, `puppeteer`, or
 *      `html2canvas` in a weapon/leaf; that forks the substrate (the exact drift
 *      that produced 4 parallel launchers + synthetic-canvas "verification").
 */

export const skillBook = {
  id: "browsercontrol",
  title: "Browser control",
  description:
    "Browser work + VISUAL VERIFICATION, including screenshotting AUTH-GATED / LOGGED-IN pages (Gmail, the GuildOS app, any account the operator is signed into in their MAIN Chrome). Take a screenshot via a Playwright browser logged into the operator's accounts — the way a headless leaf visually verifies a page behind a login. Local Claude uses Claude-in-Chrome MCP; cloud agents use their VM's native Playwright; headless quest LEAVES (which cannot use CIC) drive/screenshot via the leafBrowser weapon — Playwright running system Chrome on the operator's logged-in profile. Actions: validateCIC (local CIC precondition gate for user-channel verification), leafBrowserControl (leaf/headless Playwright screenshot of logged-in/auth-walled pages — the visual-verification path for leaves), updateLogins (refresh the operator's captured logins when a session expires or a screenshot shows a sign-in page), captureAuth (export storageState for cloud), accessChromeExtensionPage (open chrome:// URLs via CIC).",
  steps: [],
  toc: {
    validateCIC: {
      description:
        "HARD PRECONDITION GATE: prove Claude-in-Chrome can really screenshot the GuildOS app before ANY verification-dependent work. Server-up check (cicHealth weapon) → CIC open localhost:3002 → real screenshot → if NOT obtained, STOP and escalate 'CIC not working, needs unblocking' (never dance, never substitute a proxy). Run this whenever a step depends on user-channel CIC visual verification.",
      howTo: `
Binary gate: a real CIC screenshot of the running app, or a hard STOP + escalation. **Core principle (stated once): never trust a "navigation succeeded" / "weapon returned ok" return — verify the screenshot is the real app, and a failed CIC is a STOP, not something to work around or substitute a proxy for.**

## Procedure

1. **Server up?** Call the \`cicHealth.checkServer\` weapon ({ url: "http://localhost:3002" }). Any HTTP status (incl. 3xx auth redirect / 401) = UP. If DOWN → not a CIC problem: start the GuildOS dev server (\`npm run dev\` from the main checkout) or escalate "dev server not running", then continue.

2. **Connected browser — auto-bind the local instance, do NOT prompt the user.** \`mcp__Claude_in_Chrome__list_connected_browsers\`.
   - 0 connected → CIC extension not connected → escalate (user must open Chrome with the extension); cannot fix programmatically.
   - ≥1 connected → bind automatically (no AskUserQuestion). Default binding (LOCKED): pick the LOCAL instance (flagged \`isLocal\`/local, e.g. "Main Laptop Browser"); one local → bind it silently; several → most-recently-connected; only ask when there are ZERO local instances.
   - Always (re)pair via \`mcp__Claude_in_Chrome__switch_browser\` and bind the **freshly-connected** instance (broadcasts a Connect prompt; the user clicks Connect in the live one). **Binding the freshly-connected post-restart instance is the fix that works** — a stale pre-restart instance throws "Extension manifest must request permission to access the respective host" and "Failed to deserialize params.clip.scale". Never keep using a stale instance.

3. **Take the real screenshot — navigate → verify → retry.** \`tabs_context_mcp\` (createIfEmpty:true) → \`tabs_create_mcp\` (new tab) → \`navigate\` to \`http://localhost:3002\` (redirects through /opening — any rendered app page validates CIC) → immediately \`screenshot\` (or \`read_page\`) and INSPECT it. \`navigate\` frequently returns success while the tab still shows "New Tab"/blank/the prior URL (the create-then-navigate race) — the recurring "CIC not properly initiated" failure. If the screenshot is not unmistakably the running app (still "New Tab", blank, chrome://newtab, wrong origin), re-issue \`navigate\` to the SAME active tab (do NOT create another) and screenshot again; only a post-verify screenshot counts.

4. **The gate (binary):**
   - **Real screenshot obtained** ("Successfully captured screenshot … ID …" + an actual image of the running app) → CIC HEALTHY. Record the screenshot id. Proceed; downstream user-channel verification uses real CIC screenshots like this one.
   - **Not obtained** — ANY failure ("Extension manifest must request permission to access the respective host", "Failed to deserialize params.clip.scale", "Frame … showing error page", navigate mangling \`chrome://\`→\`https://chrome://\`, blank, timeout) → **STOP. Do not proceed with any verification-dependent step, and do not substitute a proxy** (\`Claude_Preview\`/preview_screenshot, \`cicVerify\` Playwright-headless, curl/HTML, any agent-side render — forbidden per the alignment theorem, \`supervisedEvolution.run\` 1c/1d). **Escalate verbatim:** "CIC is not working and needs unblocking," plus the reset that works:
     - \`chrome://extensions/\` → Claude-in-Chrome → enable Developer mode → **Update**; **Details → Site access → "On all sites"** (or add localhost);
     - fully quit & reopen Chrome;
     - re-pair: \`switch_browser\` and connect the **newly-connected** instance (not the stale one);
     - then re-run \`browsercontrol.validateCIC\`.
   Then HOLD until the user confirms and a re-run obtains a real screenshot.

The screenshot must be of the user's REAL Chrome via \`mcp__Claude_in_Chrome__*\` (their channel), never a sandbox/headless render.
`,
    },
    leafBrowserControl: {
      description:
        "VISUAL VERIFICATION for headless leaves: TAKE A SCREENSHOT via a Playwright browser LOGGED INTO THE OPERATOR'S ACCOUNTS (the logins from their MAIN Chrome) — so a leaf can screenshot auth-gated pages (Gmail, the GuildOS app) that a logged-out browser bounces to a sign-in. Launches Playwright MCP (mcp__playwright__*) driving system Chrome on the operator's profile. TWO auth profiles: `master` (cookies COPIED from the main browser — fine for non-rotating sites like GuildOS/Supabase) and `interactive` (a native login the operator did inside the automation browser, reused DIRECTLY — REQUIRED for Google/Gmail, which reject copied cookies with a passkey wall). If a screenshot comes back as a sign-in page, the session is stale → see updateLogins. NOT for local Claude (that's validateCIC/CIC).",
      howTo: `
Leaf agents (headless quest workers spawned by claudecli.invoke = \`claude --print\`) CANNOT use Claude-in-Chrome: CIC rides Chrome Native Messaging, binds to one interactive session, and is undeclarable in MCP config, so a headless leaf can never inherit it (memory: reference_cic_not_mcp_leaf_browser). Their ONLY way to directly control a webpage is the **leafBrowser** weapon — this is the Plan B.

## Mechanism
\`claude --print --mcp-config <json>\` injects a per-invocation MCP server. \`leafBrowser.prepareLeafBrowser\` writes that JSON declaring \`@playwright/mcp@latest\` (launched via **\`cmd /c npx\` on Windows** — a bare \`npx\` is \`npx.cmd\` and the host spawns it without a shell, so it silently fails to connect and the leaf gets NO tools) pointed at \`--executable-path <system Chrome> --headless\` + a profile. The leaf drives the page with \`mcp__playwright__*\` tools carrying the operator's logins — so auth-walled targets render. **TWO profile modes:**
- **master copy** (default): \`--user-data-dir <per-leaf copy of master>\`, cookies copied from the main Chrome. Good for non-rotating sessions (GuildOS/Supabase). Do NOT use for Google (copied session → passkey re-challenge; copying a rotating session can log the operator out of their main browser — ED 2026-06-16).
- **interactive direct** (\`prepareLeafBrowser({ directProfileDir: '.browser-profiles/interactive', outputDir })\`): points \`--user-data-dir\` at the operator's NATIVE login profile IN PLACE (no copy). REQUIRED for Google/Gmail; single-flight (Chrome locks a user-data-dir). This is what the proven Gmail leaf used.

## Getting the operator's logins into a profile (full menu in updateLogins)
- **master (cookie copy), zero-interaction:** \`leafBrowser.captureMasterFromChrome()\` copies the operator's LIVE Chrome login into the master profile — no manual step. questTick self-arms via \`ensureMasterFromChrome()\` and gates [BROWSER] dispatch on \`hasMasterCookie()\`. The session cookie only copies while Chrome is CLOSED (exclusively locked while running), so it lands on the next Chrome close.
- **interactive (native login), for Google/Gmail:** \`node libs/weapon/leafBrowser/cli.mjs login [url]\` opens a headed window (seeded so the account chooser shows the operator's email); the operator signs in by hand, then closes it GRACEFULLY. Leaves reuse that profile via \`directProfileDir\`. Check state: \`node libs/weapon/leafBrowser/cli.mjs status\`.

## How a leaf gets a browser (automatic, in questTick execute-dispatch)
1. Tag the quest \`[BROWSER]\` in its title or description → \`wantsBrowser\`.
2. questTick gates on \`leafBrowser.hasMasterProfile()\`; if present → \`leafBrowser.prepareLeafBrowser({ workerId })\` copies master→per-leaf profile, writes the mcp-config, returns \`mcpConfigPath\`.
3. \`buildLeafExecutePrompt({ browser:true })\` injects the "AUTHENTICATED BROWSER AVAILABLE" block (use \`mcp__playwright__*\`; verify visually, never via WebFetch).
4. \`claudecli.invoke(prompt, { mcpConfigPath })\` appends \`--mcp-config\`; the leaf runs with Playwright.
5. After the leaf finishes → \`leafBrowser.cleanupLeafBrowser({ workerId })\` removes the profile copy + temp config.

## Public targets without auth (leaf MCP surface)
\`prepareLeafBrowser({ workerId, allowIsolated:true })\` falls back to an ephemeral \`--isolated\` Playwright profile (no login) when no master profile exists — correct for public Storage URLs / localhost / public pages where a real screenshot is wanted but auth is irrelevant.

## In-process screenshots (no LLM) — the OTHER surface, use this for verify
When you just need "URL/ref → PNG" deterministically (visual verification, posters,
QA), do NOT spawn a leaf and do NOT launch Playwright yourself. Call
\`leafBrowser.screenshotUrl({ ref|url, kind?, fullPage?, selector?, authed? })\` →
\`{ ok, buffer|base64 }\`. It resolves public-relative refs (\`proving-grounds/x/index.html\`)
against the dev server with a \`file://\` fallback, renders PDFs via pdf.js, scrolls
to trigger lazy-loads, and FAILS CLOSED on a blank/errored page. This is the path
\`questPurrview.autoVisualVerify\` uses to fill \`verify_screenshot_ref\`. Proven live
2026-06-15. (Need to drive the page yourself? \`leafBrowser.launchContext()\`.)

## Stability — PROVEN 2026-06-18
A real end-to-end leaf run succeeded: a \`claude --print\` leaf with the injected Playwright MCP navigated to Gmail and screenshotted the operator's AUTHENTICATED inbox (<REDACTED-EMAIL>, real senders) → \`public/proving-grounds/leaf-gmail-inbox.png\`. Notes: (a) **Windows npx gotcha — the required fix:** the MCP host spawns the server command directly (no shell); \`npx\` is \`npx.cmd\` on Windows, so a bare \`"npx"\` silently fails and the leaf sees NO mcp__playwright__* tools (the symptom on the first attempt). prepareLeafBrowser now wraps it \`cmd /c npx …\` on win32. (b) \`npx -y @playwright/mcp@latest\` fetches the server per leaf (first run slow + network-dependent). (c) drives SYSTEM Chrome via \`--executable-path\` + Chrome's own \`--headless\` — no Playwright bundled Chromium / win-arm64 headless-shell dependency. (d) Google/Gmail MUST use the \`interactive\` profile via \`directProfileDir\` (the master copy hits the passkey wall).
`,
    },
    updateLogins: {
      description:
        "UPDATE / REFRESH the operator's captured browser logins when they go stale or outdated — symptom: a leaf or Playwright screenshot of an auth-gated page (Gmail, the GuildOS app) comes back as a SIGN-IN / account-chooser page instead of the content. Refreshes either the `master` cookie copy (GuildOS/Supabase, non-Google) or the `interactive` native-login profile (Google/Gmail). Operator-gated for the actual sign-in; the agent sets it up and verifies.",
      howTo: `
A captured login is NOT permanent — cookies expire, refresh tokens rotate, Google re-challenges. When a screenshot of a logged-in page returns a sign-in / account-chooser page, the session is stale; refresh the relevant profile.

## master profile — cookie copy from the main browser (GuildOS/Supabase, non-Google)
\`leafBrowser.captureMasterFromChrome()\` (CLI: \`node libs/weapon/leafBrowser/cli.mjs capture-from-chrome\`) re-copies the operator's LIVE Chrome login. The session **Cookies** file is exclusively locked while Chrome runs, so it only refreshes while Chrome is CLOSED — the call is non-destructive (keeps the prior cookie if it can't grab a new one) and questTick retries it each tick, so it self-refreshes on the next Chrome close.
⚠ Reusing a copied rotating-refresh-token session can LOG THE OPERATOR OUT of their main browser (rotation invalidates the original) — expected, not a bug (ED 2026-06-16). Prefer the interactive profile for important accounts.

## interactive profile — native login (Google/Gmail, anything passkey/rotation-sensitive)
\`node libs/weapon/leafBrowser/cli.mjs login [url]\` re-opens the headed Playwright window (auto-seeded from master so the account chooser shows the operator's email). The operator signs in BY HAND — a CLEAN login goes to the password step, NOT the passkey wall a copied/seeded session triggers. Then close the window GRACEFULLY (never force-kill — that drops the just-completed session, ED 2026-06-16). Leaves then reuse the SAME profile directly (\`prepareLeafBrowser({ directProfileDir })\`), no copy.

## verify the refresh worked
Re-screenshot the target via \`leafBrowser.screenshotUrl\` (in-process) or a leaf via \`prepareLeafBrowser({ directProfileDir })\` — it should now render the content, not a sign-in. (Proven path: the Gmail leaf, \`public/proving-grounds/leaf-gmail-inbox.png\`.)

This is operator-gated: only the user can perform the actual sign-in / passkey. The agent opens the window, pre-fills the email, and verifies the result — it does NOT enter passwords or complete 2FA.
`,
    },
    captureAuth: {
      description: "Export a storageState JSON that cloud agents can import as Playwright auth context.",
      howTo: `
Cloud Cursor agents on Linux VMs drive the browser natively via Playwright but start with a clean session. This script opens a local Chrome, waits for you to log in to whichever service(s) you want the cloud agents to inherit, and writes cookies + localStorage to \`playwright/.auth/user.json\`. Cloud agents then pass that file as Playwright's \`storageState\` to start authenticated.

\`\`\`bash
node scripts/auth-capture.mjs             # log in manually, exports JSON
node scripts/auth-load.mjs                # verify JSON export
\`\`\`

This script is the operator-manual storageState export for CLOUD agents — a distinct job from the headless screenshot substrate (\`leafBrowser\`, which owns all in-repo Chromium launching since 2026-06-15). Local Claude uses Claude-in-Chrome MCP tools for browsing — NOT the captured profile.
`,
    },
    accessChromeExtensionPage: {
      description: "Navigate CIC to chrome://extensions/ (reload an extension, toggle dev mode, etc.).",
      howTo: `
Use the Claude-in-Chrome MCP \`navigate\` tool with URL \`chrome://extensions/\`.

**Critical:** the URL MUST include the trailing slash. \`chrome://extensions\` (no slash) silently fails to load via CIC — the tab navigates, no visible error, but the page never renders. Always use \`chrome://extensions/\`.

Same rule applies to other chrome:// URLs that end in a path segment: \`chrome://settings/\`, \`chrome://flags/\`, \`chrome://inspect/\`. Add the trailing slash when in doubt.

Common follow-ups once on chrome://extensions/:
- Reload an unpacked extension: \`find\` the extension's reload button (circular-arrow icon under its card) and click it.
- Toggle developer mode: top-right switch.
- After reloading, navigate to the target site (or tell the user to refresh their tab) so the new content script loads.

If the user has disabled the extension or uninstalled it, chrome://extensions/ is where you verify — but you cannot programmatically re-enable/re-install; ask the user.
`,
    },
  },
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/leafBrowser/cli.mjs -->
```js
#!/usr/bin/env node
/**
 * leafBrowser CLI — operator helpers for the leaf authenticated-browser profile.
 *
 *   node libs/weapon/leafBrowser/cli.mjs capture   # launch headed Chrome on the master profile to log in
 *   node libs/weapon/leafBrowser/cli.mjs status    # show master/chrome detection
 *
 * `capture` is the ONE manual step: Chrome opens on the master profile, you log
 * into the sites leaves must reach, then close it. Leaves get copies thereafter.
 */
import { spawn } from "child_process";
import { mkdirSync } from "fs";
import { captureMasterProfileCmd, hasMasterProfile, hasMasterCookie, captureMasterFromChrome, openInteractiveLogin, exportStorageState, findChrome } from "./index.js";

const cmd = process.argv[2] || "status";

// Headed, persistent interactive login (for sites that reject copied cookies,
// e.g. Gmail). Auto-seeds the account from the master profile's storageState so
// the page recognizes the operator (Gmail shows their email) BEFORE they log in.
// Run DETACHED — it stays open until the operator closes the window.
//   node libs/weapon/leafBrowser/cli.mjs login [url]
if (cmd === "login") {
  const url = process.argv[3] || "https://mail.google.com/";
  let seedStatePath = null;
  if (hasMasterProfile()) {
    const exp = await exportStorageState({ fromProfile: "master", outPath: ".browser-profiles/auth-seed.json" });
    if (exp.ok) { seedStatePath = exp.path; console.log(`Seeded account from master (${exp.cookieCount} cookies) → ${exp.path}`); }
    else console.log(`Seed export skipped: ${exp.reason}`);
  }
  const r = await openInteractiveLogin({ url, seedStatePath, resetProfile: true });
  console.log(JSON.stringify(r));
  process.exit(r.ok ? 0 : 1);
}

if (cmd === "status") {
  console.log(JSON.stringify({
    hasMasterProfile: hasMasterProfile(),
    hasMasterCookie: hasMasterCookie(),
    chrome: findChrome(),
    capture: captureMasterProfileCmd(),
  }, null, 2));
  process.exit(0);
}

// Non-interactive capture from the operator's LIVE Chrome profile (zero login
// step). Grabs the Cookies DB only when Chrome isn't holding it under its
// exclusive lock — close Chrome first if `hasMasterCookie` stays false.
if (cmd === "capture-from-chrome") {
  const r = captureMasterFromChrome();
  console.log(JSON.stringify(r, null, 2));
  console.log(r.ok ? "ARMED (session cookie captured)." : "NOT armed — Cookies still locked. Close Chrome once and re-run, or it self-arms on the next questTick after Chrome closes.");
  process.exit(r.ok ? 0 : 1);
}

if (cmd === "capture") {
  const info = captureMasterProfileCmd();
  if (!info.chrome) {
    console.error("Chrome not found — install Chrome or set its path. Detected: null");
    process.exit(1);
  }
  mkdirSync(info.masterDir, { recursive: true });
  console.log(`Launching Chrome on master profile:\n  ${info.masterDir}\nLog into the sites leaves need, then CLOSE Chrome to finish.`);
  const child = spawn(info.chrome, [`--user-data-dir=${info.masterDir}`], { stdio: "ignore", detached: false });
  child.on("close", (code) => {
    console.log(`Chrome closed (code=${code}). Master profile ready: ${hasMasterProfile()}`);
    process.exit(0);
  });
  child.on("error", (e) => { console.error("Failed to launch Chrome:", e.message); process.exit(1); });
} else {
  console.error(`Unknown command "${cmd}". Use: capture | status`);
  process.exit(1);
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/leafBrowser/engineering_decisions.md -->
```markdown
# leafBrowser — Engineering Decisions

## 2026-06-15 · leafBrowser is the SINGLE browser substrate; in-process screenshots route through it

**Decision.** leafBrowser owns ALL Playwright/Chromium launching in the repo, via
two surfaces over one auth model:
- `prepareLeafBrowser` — MCP injection for headless `claude --print` LEAVES
  (interactive page-driving by an LLM).
- `launchContext` / `screenshotUrl` — in-process render-to-PNG (no LLM), the
  primitive `questPurrview.autoVisualVerify`, `cicVerify`, and `poster_render`
  now delegate to. `launchContext` is the only `chromium.launch*` call site.

**Rejected alternative.** Leaving each weapon with its own
`await import("playwright"); chromium.launch({headless:true})` (the state before
this date: questPurrview.renderDeliverableScreenshot, cicVerify.screenshotFromUrl,
poster_render.renderPoster each launched independently), and letting the
"planned substrate" (leafBrowser) stay MCP-only.

**Why.** leafBrowser was built with ONLY the MCP-leaf surface, gated on a
captured master profile — so it was dormant (no profile → never called) while the
job that actually recurs is the *unauthenticated, in-process* "URL → verify PNG".
With no in-process surface on leafBrowser, three weapons grew parallel chromium
launchers and, under pressure, leaves hand-rolled `node -e require('playwright')`
and fell back to **html2canvas** (the operator's locked not-real-verification
path). Four+ launchers + a synthetic-canvas escape hatch = exactly the substrate
sprawl a single owner prevents (drifting auth policy, chromium-resolution, and
fail-closed semantics; the browsercontrol skill book even still claimed Chrome was
launched in only one place). Centralizing also makes the auth-profile model
reachable from the in-process path (`authed:true` → persistent context over a
profile copy), so the captured-profile work finally pays off for screenshots, not
just MCP leaves.

**Consequence if violated.** Re-introducing a local `chromium.launch` /
`puppeteer` / `html2canvas` in any weapon forks the substrate again: verify
screenshots stop sharing the fail-closed-on-blank guard and the ref→URL
resolution (the bug that made `proving-grounds/x/index.html` refs un-navigable and
silently empty), and synthetic-canvas "verifications" creep back in. New screenshot
needs MUST call `leafBrowser.screenshotUrl` (or `launchContext` if they drive the
page themselves).

**Scope note.** `CICPolice` (drives a *headed* browser to police CIC) and
`scripts/auth-capture.mjs` (manual storageState export for cloud Cursor agents)
are deliberately NOT folded in — different jobs, not the verify-screenshot
substrate. They are the only remaining direct Playwright launches and are headed /
operator-manual, not headless screenshot producers.

**Proven.** 2026-06-15 — `screenshotUrl({ ref:'proving-grounds/about-us-rewrite/index.html' })`
returned a 304 KB full-page PNG (real render, visually confirmed), including the
dev-server→`file://` fallback for static public refs.

## 2026-06-16 · Master capture = copy the operator's LIVE Chrome profile, non-interactively + self-arming

**Decision.** `captureMasterFromChrome` populates the master by copying the
operator's EXISTING logged-in Chrome profile (auth-only allowlist: `Local State` +
`Network/Cookies` + `Local Storage/leveldb` + `Preferences`) rather than the
interactive headed re-login (`captureMasterProfileCmd` / `cli.mjs capture`).
Dispatch gates on `hasMasterCookie()` (the session Cookies DB) — NOT
`hasMasterProfile()` — and `questTick` calls `ensureMasterFromChrome()` so the
pipeline self-arms with zero operator action.

**Rejected alternatives.** (1) The interactive capture (operator opens headed
Chrome, logs into every site, closes) — rejected because the operator is ALREADY
logged in in their main Chrome and explicitly wanted no manual step. (2) Gating on
`hasMasterProfile()` — rejected: after a partial copy, `Local State` exists so it
returns true while the profile has NO cookie, so a `[BROWSER]` leaf would dispatch
"authenticated" and silently bounce to `/signin`. (3) Force-closing / taskkilling
the operator's Chrome to free the cookie lock — rejected (memory
feedback_browser_control: never taskkill the operator's Chrome).

**Why / the load-bearing constraint.** Chrome's Network service holds
`Default/Network/Cookies` under an EXCLUSIVE OS lock while running — verified three
ways 2026-06-16: `CopyFile` → EBUSY, libuv read-handle → EBUSY, .NET
`FileShare.ReadWrite` → sharing violation; this box is non-admin (no VSS/`esentutl`)
and Chrome 149 refuses `--remote-debugging-port` on the default profile dir (no CDP
fallback). So the session cookie CANNOT be extracted while Chrome is live. The
copy is therefore (a) lock-tolerant (`lockTolerantCopy`: libuv read-handle bypasses
CopyFile's share-mode failure for files Chrome doesn't hold exclusively), (b)
NON-destructive (no `rmSync` wipe — a re-run that can't grab the live-locked cookie
must preserve a previously captured one), and (c) opportunistic (re-attempts only
the cookie each tick; succeeds the moment Chrome is next closed). Everything except
the cookie copies fine while Chrome runs; the cookie self-captures on the next
natural Chrome close — zero operator action.

**Consequence if violated.** Re-adding `rmSync(MASTER_DIR)` to the capture wipes a
good cookie whenever the next opportunistic run hits a live lock (the pipeline
would arm then disarm itself every cron tick). Gating dispatch back on
`hasMasterProfile()` dispatches `[BROWSER]` leaves against a cookieless profile
that just screenshots the sign-in wall (the exact deadlock on quest 988c9792).

**PROVEN end-to-end 2026-06-16.** Operator closed Chrome → `captureMasterFromChrome`
grabbed the now-unlocked `Network/Cookies` → a headless Playwright leaf rendered the
**authenticated** GuildOS `/caravan` (no `/signin` redirect; full dashboard —
`public/proving-grounds/caravan-authed.png`). So ABE cookie decryption works in a
copied profile launched by Playwright with system Chrome on the same user+machine,
and the cron-leaf auth-screenshot deadlock (quest 988c9792) is cleared.

**Google-account caveat (Gmail will NOT work via copied cookies).** Same run, target
`mail.google.com`: Google recognized the account from the copied session ("Hi Sijie,
<REDACTED-EMAIL>") but stopped at "Verifying it's you — complete sign-in using your
passkey" (`public/proving-grounds/gmail-inbox-delivered.png`), even headed + with
anti-automation flags (`--disable-blink-features=AutomationControlled`, no
`--enable-automation`, `navigator.webdriver` spoofed). High-value Google accounts
require a passkey/2FA to authorize a session on a *new browser instance*
(device-bound-session-style hardening), which a profile/cookie copy cannot satisfy
(the passkey is hardware/TPM-bound, non-exportable) and which an agent must not
complete. **Consequence:** do not waste cycles trying to make Gmail (or other
passkey-gated Google surfaces) work through `leafBrowser` **cookie-COPY** reuse — it
is a Google-side wall, not a pipeline defect. The pipeline IS correct for normal
cookie/Supabase-session sites (proven on `/caravan`).

## 2026-06-16 · HAZARD — copying a rotating-refresh-token session can LOG OUT the source browser

**Observed.** After a leaf rendered the authenticated `/caravan` via the cookie-COPY
master (real, screenshotted), within ~the hour BOTH the copy AND the operator's MAIN
Chrome were logged out (`/caravan` → `/signin`). The operator noticed their own
localhost:3002 was signed out.

**Why.** GuildOS auth is Supabase, which **rotates the refresh token** on use. The
copied cookie carries the SAME refresh token as the source. When the leaf's copy
exercised the session and a refresh fired, Supabase rotated the token and invalidated
the prior one — which the operator's main browser still held → the operator got
signed out as collateral. (Plain ~1h access-token expiry can produce the same symptom;
rotation triggered by the copy's use is the likely accelerant.)

**Consequence / guidance.** Cookie-copy auth is **best-effort and can disturb the
source login** for any site with rotating sessions (Supabase, many OAuth sessions).
Do NOT present a cookie-copy render as a durable/shared logged-in state — it is a
point-in-time snapshot that can desync. For durable automation auth, prefer the
**dedicated-interactive-profile** model (log in ONCE in `.browser-profiles/interactive`
and have leaves reuse that profile DIRECTLY) — the session lives only there, so
refreshing/rotating it never touches the operator's main browser. Reserve cookie-copy
for read-mostly, short-lived, non-rotating targets.

## 2026-06-18 · PROVEN — Gmail works headlessly via native interactive login + same-profile reuse

The Gmail path below is now VERIFIED end-to-end, and it resolves the earlier
"Gmail is walled" caveat (which applies ONLY to cookie-COPY):
- **Clean login avoids the passkey wall.** A FRESH interactive login (empty profile,
  no seeded/copied cookies) advances `<REDACTED-EMAIL>` straight to the PASSWORD
  step — NOT the "complete sign-in using your passkey" challenge. The passkey wall
  only fires when Google sees a copied/seeded session it must re-verify on a "new
  device." So do NOT seed the profile when the goal is a real login; seed only to
  show the account chooser.
- **Same-profile headless reuse renders the inbox.** After the operator logged in
  (password) in the headed window on `.browser-profiles/interactive` and closed it
  gracefully, a HEADLESS Playwright launch on that SAME profile dir loaded
  `mail.google.com/mail/u/0/#inbox` with the live inbox (Inbox 368, Compose, labels)
  — `public/proving-grounds/gmail-inbox-delivered.png`. No copy → no device-binding
  re-challenge. This is the durable Gmail-in-a-leaf path; quest 6b5133b8 = complete.

## 2026-06-18 · Leaf-MCP injection: `cmd /c npx` on Windows + `directProfileDir` (PROVEN)

**Decision.** `prepareLeafBrowser`'s mcp-config launches the Playwright MCP server as
`{ command:"cmd", args:["/c","npx",...] }` on win32 (not `{ command:"npx", args:[...] }`),
and accepts `directProfileDir` (use a profile IN PLACE, no copy) + `outputDir`.

**Why — the bug this fixes.** An MCP host (`claude --print --mcp-config`) spawns the
server command DIRECTLY, without a shell. On Windows `npx` is `npx.cmd`; a bare
`"npx"` is not found, the server never connects, and `claude --print` proceeds with
**no `mcp__playwright__*` tools** — silently (no fatal error). First leaf run hit
exactly this: the leaf reported "no Playwright MCP tools connected." Wrapping via
`cmd /c npx` resolves it. **Rejected alternative:** `command:"npx"` (worked on
macOS/Linux, silently broke every Windows leaf — and GuildOS runs on Windows).

**`directProfileDir`.** The default per-leaf COPY of master is wrong for Google
(copy → passkey re-challenge) and for rotating sessions (logs the operator out).
`directProfileDir` points `--user-data-dir` at the operator's native-login profile
(`.browser-profiles/interactive`) in place — single-flight, never copied, never
cleaned up by `cleanupLeafBrowser` (which only removes `LEAF_ROOT/<workerId>`).

**Consequence if violated.** Reverting to `command:"npx"` silently disables browser
for EVERY Windows leaf (the deadlock returns, with no error to diagnose). Copying the
interactive profile instead of using it directly re-triggers the Google passkey wall.

**PROVEN 2026-06-18.** A real `claude --print` leaf with this config navigated to
Gmail and screenshotted the operator's authenticated inbox (<REDACTED-EMAIL>, real
senders) → `public/proving-grounds/leaf-gmail-inbox.png`. This is the end-to-end
leaf-MCP proof the earlier "not yet proven" note (browsercontrol) was waiting on.

## 2026-06-16 · HAZARD — never FORCE-KILL an interactive-login Chrome window

Force-killing the Playwright login window (`Stop-Process -Force` / `taskkill /F`)
skips Chrome's graceful shutdown, which is what flushes the just-completed session
to the profile on disk → the operator's login is LOST (observed: a completed
`<REDACTED-EMAIL>` login vanished after a force-kill, profile showed "Signed
out" next launch). Close via `context.close()` (graceful) or let the operator close
the window. Only force-kill when you are about to WIPE that profile anyway.

Also: Google's account chooser offers no inline "Remove an account" while one of the
listed accounts holds an active session — so to drop a specific account (e.g. a
passkey-walled personal account, keeping a password-login work account) the reliable
move is to WIPE the dedicated profile (`.browser-profiles/interactive`; `master` is
separate and untouched) and have the operator log into only the wanted account. A
passkey-gated Google account cannot be automated headlessly at all; a password-login
(non-passkey) account can.

**The Gmail path (added 2026-06-16): native interactive login, reused directly.**
`openInteractiveLogin()` opens a HEADED persistent Playwright browser on a DEDICATED
profile (`.browser-profiles/interactive`) where the operator logs in BY HAND. Because
the session is MINTED in that browser+profile (not copied in), the device binding is
native → Google does not re-challenge, and it persists for days. **Load-bearing
rule: reuse that profile dir DIRECTLY — NEVER copy it** (a copy re-triggers the exact
passkey wall above; that is the whole reason the cookie-copy master fails for Gmail).
So Google/Gmail leaf targets must launch on `.browser-profiles/interactive` itself,
single-flight (Chrome locks a user-data-dir), NOT via the per-leaf copy path used for
the cookie-copy master. Do not "unify" interactive into the copy flow — it breaks it.
(The Gmail MCP remains a fine alternative when only email *content* is needed, no
browser screenshot.)

## 2026-06-16 · Cloudflare-protected targets (bosterbio): HEADED + ONE fresh context per page

**Discovery (measured this date, verifying the EK1695 / Biotium escalated quests).**
For `www.bosterbio.com` (Cloudflare "Just a moment…" JS challenge):
- `screenshotUrl`/`launchContext` with `headless:true` is **403'd every time** — both
  `isolated` AND `authed` (the bundled-Chromium headless fingerprint is the tell;
  even the operator's real profile + `cf_clearance` cookie does not save it).
- `headless:false` (headed, system Chrome via `executablePath`) **clears the
  challenge** — but only for the **FIRST navigation in a given context**. Cloudflare
  adaptively re-challenges the 2nd+ nav in the same session (observed: page A 200,
  page B/C 403 in one context). So the reliable pattern is **one fresh
  `launchContext` per page** (first-hit-per-context passes), with a poll-until-cleared
  wait (title ≠ "just a moment" AND bodyLen>~400, up to ~25–30 s) before screenshot.

**Rejected alternative.** Batching N bosterbio pages through one headless (or one
shared headed) context — produces a pile of "Just a moment…" challenge screenshots
that *look* like real captures (fail-closed catches blanks, but a 265-char challenge
page is not blank), i.e. silent false verification.

**Consequence if violated.** A verify/QA run that reuses a context or stays headless
will catalog Cloudflare interstitials as "verified" deliverables. Multi-step flows on
bosterbio (add-to-cart → cart) inherently need ≥2 navs → the 2nd gets challenged, so
**cart-submit cannot be auto-driven this way** (verify cart via the cart API, or a
single manual add). Pace + headed + fresh-context-per-page, or don't claim it.

## 2026-06-16 · The master profile authenticates PROD + bosterbio, NOT localhost — verify against prod URLs

**Discovery.** `captureMasterFromChrome` copies the operator's LIVE Chrome cookies, so
the master holds sessions for what they actually browse: `guild-os-ten.vercel.app`
(prod GuildOS), `<REDACTED-HOST>` (prod boster_nexus), `bosterbio.com`,
`cjgeoai.com`, Google. It does **NOT** hold `localhost:3002` / `localhost:3001`
cookies (the operator signs into local dev via CIC / a different context, not their
main Chrome). Verified 2026-06-16: `authed` render of `localhost:3002/caravan` →
302 `/signin`, but `guild-os-ten.vercel.app/caravan` → 200 authed (full dashboard);
same for `<REDACTED-HOST>` `/ooa-dashboard` + `/purchase-orders/open-items`.

**Consequence / guidance.** To visually verify an auth-gated app feature with
leafBrowser, target its **production URL**, not localhost — the cookie is there.
This means a feature must be **DEPLOYED to prod** to be leaf-verifiable; a local-only
build (uncommitted worktree changes, or a dev-only page like the OOA user-drill-down
and the Open-PO per-item status field, both confirmed ABSENT on prod 2026-06-16)
**cannot** be verified this way. The fix for a local-only target is either (a) deploy
it to prod then verify, or (b) capture a localhost login into the master (operator
logs into the local dev server in their main Chrome, then `ensureMasterFromChrome`)
AND have that dev server running. Do not assume `authed:true` reaches localhost.
## 2026-07-03 - Isolated verification launches prefer system Chrome; purrview disables lazy-scroll

**Decision.** `launchContext` uses the operator's system Chrome for isolated
headless screenshots when it can find it, and `questPurrview.autoVisualVerify`
calls `screenshotUrl` with `waitUntil:"domcontentloaded"` and `scrollLazyLoad:false`.

**Rejected alternative.** Keep isolated screenshots on Playwright's bundled
Chromium and let purrview use the screenshot primitive's default
`networkidle`+lazy-scroll behavior.

**Why.** On the Hermes visual-gate acceptance quest, bundled/default rendering
stalled the real `autoVisualVerify` path: the deliverable was a valid public
Supabase HTML page, but the purrview capture never returned. A bounded system
Chrome render of the same URL produced a real PNG in about two seconds. Purrview's
job is to capture enough real pixels for the independent vision judge, not to
crawl an arbitrary page forever.

**Consequence if violated.** Hermes-produced deliverables can reach `purrview`
with valid refs and still never get a `verify_screenshot_ref`, so the review gate
appears broken even though the worker output is correct.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/leafBrowser/index.js -->
```js
/**
 * leafBrowser weapon — THE single browser substrate for GuildOS automation.
 *
 * It exposes TWO surfaces over ONE shared profile/auth model. Both share
 * findChrome() + the isolated-vs-profile-copy decision; nothing else in the repo
 * should launch Chromium/Playwright directly (see engineering_decisions.md
 * 2026-06-15 — consolidation).
 *
 *  SURFACE 1 — MCP injection for LEAVES (prepareLeafBrowser): hand a headless
 *  `claude --print` leaf a live `mcp__playwright__*` browser via
 *  `--mcp-config`. Used when an LLM leaf must DRIVE a page interactively.
 *
 *  SURFACE 2 — in-process render-to-PNG (launchContext / screenshotUrl):
 *  deterministic "navigate a URL → PNG buffer" with no LLM in the loop. This is
 *  what visual-verification (questPurrview.autoVisualVerify), poster rendering
 *  (poster_render), and headless QA (cicVerify) actually need. Before the
 *  2026-06-15 consolidation each of those weapons hand-rolled its own
 *  `chromium.launch`; they now all delegate here.
 *
 * Why this weapon owns both: CIC (Claude-in-Chrome) rides Chrome Native
 * Messaging, binds to ONE interactive session, and is undeclarable in config, so
 * a headless leaf can never inherit it (memory reference_cic_not_mcp_leaf_browser).
 * Playwright is the only headless-capable path; centralizing it here keeps a
 * single auth model and a single chromium-resolution policy.
 *
 * Profile model (full-profile-copy, chosen 2026-06-13): Chrome locks a
 * user-data-dir, so each authenticated leaf gets its OWN copy of the master
 * profile. The copy is selective (auth-bearing files only, cache excluded) to
 * stay small. The master profile is created once, interactively, by the user
 * (captureMaster / cli.mjs capture) — only the user can perform the logins.
 * AUTHENTICATED mode is a no-op until that capture runs; ISOLATED mode (public /
 * localhost / unauthenticated targets — the common case) needs no profile and
 * works immediately.
 */

import { existsSync, mkdirSync, rmSync, cpSync, readFileSync, writeFileSync, statSync } from "fs";
import { join, dirname } from "path";
import { execSync } from "child_process";
import { logWeaponCall } from "../weaponLog/index.js";

export const toc = {
  screenshotUrl:
    "THE single in-process render-to-PNG primitive. Navigate a URL or public-relative ref (proving-grounds/…), scroll to trigger lazy-loads, full-page screenshot → PNG Buffer (or base64). Handles PDFs via pdf.js (multi-page), element shots via `selector`, and fails CLOSED on a blank/errored page. Isolated (no auth) by default; pass authed:true to use the operator's profile copy. Every screenshot job (verify, poster, QA) goes through here — do NOT call chromium.launch elsewhere.",
  launchContext:
    "Low-level: launch ONE Playwright browser context (the only chromium-launch site in the repo) with the isolated-vs-profile-copy auth decision applied. Returns { ok, context, browser, mode, close }. Use when you need to drive the page yourself (e.g. poster_render injecting data); prefer screenshotUrl for plain screenshots.",
  prepareLeafBrowser:
    "Emit an --mcp-config JSON declaring @playwright/mcp with the auth/profile decision applied, for injecting an authenticated browser into a `claude --print` leaf. Default: per-leaf COPY of the master profile. directProfileDir → use that profile IN PLACE, no copy (REQUIRED for Google/Gmail — copying re-triggers the passkey wall; single-flight). outputDir → where the leaf's screenshots land (--output-dir). Returns { ok, mcpConfigPath, profileDir, mode } or { ok:false, reason }.",
  cleanupLeafBrowser: "Remove a leaf's profile copy + mcp-config temp file after the leaf finishes.",
  hasMasterProfile: "True if a captured master profile DIR exists (Default/ or Local State present). NB: not sufficient for auth — use hasMasterCookie for the real dispatch gate.",
  hasMasterCookie: "True only when the master holds a session Cookies DB — the real 'auth armed' gate. A profile without it cannot pass an SSR auth gate (e.g. /caravan). Dispatch gates on this.",
  ensureMasterFromChrome: "Opportunistic self-arming gate: returns true once a cookie is present; otherwise attempts captureMasterFromChrome (succeeds the next moment Chrome isn't holding Cookies under its exclusive lock). Call this from dispatch so the pipeline arms itself with ZERO operator action.",
  findChrome: "Locate the Chrome executable on this machine (registry App Paths → known per-user/Program Files paths).",
  captureMasterProfileCmd: "Return the headed-Chrome command the USER runs once to create + log into the master profile.",
  openInteractiveLogin:
    "Open a HEADED, persistent, anti-automation Playwright browser on a DEDICATED profile (.browser-profiles/<name>, default 'interactive') and navigate to a URL so the OPERATOR can log in BY HAND (passkey/2FA included). Optional seedStatePath (an exportStorageState JSON) pre-seeds the account so the target RECOGNIZES the operator before they touch it (Gmail shows their email in the account chooser, not a blank sign-in); resetProfile wipes a stale prior session first. Because the login completes IN this profile, the device-bound session is native and PERSISTS for days — the path for sites that reject copied cookies (Gmail/Google). Stays open until the operator closes the window. Run detached via `cli.mjs login [url]`. Reuse the SAME profile directly (no copy) to keep the session.",
  exportStorageState:
    "Export a profile's cookies+localStorage as a Playwright storageState JSON — the stable, plaintext, lock-free auth artifact a leaf imports via newContext({storageState}). Reads WITHOUT navigating (no token-refresh/rotation side effect). From the master profile it yields a JSON whose Google cookies make Gmail recognize the operator's account. Use to seed openInteractiveLogin or to hand leaves a portable auth file.",
  captureMasterFromChrome:
    "NON-INTERACTIVE master capture: copy the operator's LIVE Chrome login state (cookies + Local State key + localStorage) into the master slot — zero manual step when they are already logged in. Copies an auth-only allowlist (the full Default profile is ~1 GB) and tolerates per-file locks while Chrome runs. Prefer this over captureMasterProfileCmd when the operator wants no operation. Returns { ok, masterDir, copied, skipped }.",
  navigateAndGetText:
    "Navigate a URL with a Playwright browser (supports directProfileDir for the interactive Google profile) and return the page's innerText. Use for reading Google Drive folder listings, Google Docs, or any auth-walled page where you need TEXT not a screenshot. Pass directProfileDir to use the native Google login profile. Returns { ok, text, url, mode }.",
};

// Repo-root-relative locations (gitignored — see .gitignore .browser-profiles/).
const PROFILES_ROOT = join(process.cwd(), ".browser-profiles");
const MASTER_DIR = join(PROFILES_ROOT, "master");
const LEAF_ROOT = join(PROFILES_ROOT, "leaves");
const MCP_TMP_ROOT = join(PROFILES_ROOT, "mcp-config");

// Cache / non-auth subdirs excluded from the per-leaf copy so a copy is small
// and fast. Auth lives in Cookies / Login Data / Network/ / Web Data /
// Preferences + top-level Local State, all of which ARE copied.
const EXCLUDE_DIR_NAMES = new Set([
  "Cache",
  "Code Cache",
  "GPUCache",
  "GrShaderCache",
  "ShaderCache",
  "DawnCache",
  "DawnGraphiteCache",
  "DawnWebGPUCache",
  "Service Worker",
  "Crashpad",
  "component_crx_cache",
  "extensions_crx_cache",
  "Download Service",
  "optimization_guide_model_store",
  "segmentation_platform",
]);

// Auth-only allowlist for NON-INTERACTIVE capture from the operator's LIVE Chrome
// profile (captureMasterFromChrome). Only the files that actually carry a
// logged-in session — the full Default profile is ~1 GB, so copying it whole (and
// then again per leaf) is wasteful. {p} = the profile sub-dir ("Default").
//   - Local State            → top-level os_crypt key that decrypts Cookies.
//   - Network/Cookies(+wal)   → cookie-based sessions (GuildOS /caravan = Supabase SSR cookie).
//   - Cookies (legacy)        → pre-Network-dir Chrome location, if present.
//   - Preferences             → avoids Chrome's "profile error" repair prompt on launch.
//   - Local Storage/leveldb   → localStorage-based sessions (Supabase browser client, etc.).
const AUTH_CAPTURE_PATHS = [
  "Local State",
  "{p}/Network/Cookies",
  "{p}/Network/Cookies-wal",
  "{p}/Network/Cookies-journal",
  "{p}/Cookies",
  "{p}/Cookies-journal",
  "{p}/Preferences",
  "{p}/Local Storage/leveldb",
];

// Lock / singleton files that must NOT travel into a copy (a stale lock confuses
// the Chrome that later opens the copy; leveldb's LOCK can't be copied while held).
const COPY_SKIP_FILES = new Set(["LOCK", "SingletonLock", "SingletonCookie", "SingletonSocket", "lockfile"]);

/** Copy one path, tolerating Chrome's live write-lock on SQLite files. Windows
 *  CopyFile (used by cpSync/copyFileSync) requests only FILE_SHARE_READ and fails
 *  EBUSY/EPERM against a file Chrome holds open for write (e.g. Network/Cookies).
 *  libuv's read-handle (readFileSync, 'r') shares read+write+delete, so it can read
 *  the same bytes. Dirs (leveldb) go through cpSync; single files fall back to a
 *  byte read→write when CopyFile is blocked. Returns true on success. */
function lockTolerantCopy(src, dest) {
  mkdirSync(dirname(dest), { recursive: true });
  if (statSync(src).isDirectory()) {
    cpSync(src, dest, {
      recursive: true, force: true, errorOnExist: false,
      filter: (s) => !COPY_SKIP_FILES.has(s.split(/[\\/]/).pop()),
    });
    return true;
  }
  try {
    cpSync(src, dest, { force: true, errorOnExist: false });
    return true;
  } catch {
    writeFileSync(dest, readFileSync(src)); // libuv 'r' handle bypasses the write-lock
    return true;
  }
}

export function hasMasterProfile() {
  return existsSync(join(MASTER_DIR, "Default")) || existsSync(join(MASTER_DIR, "Local State"));
}

/** TRUE only when the master holds a session COOKIE — the real "auth is armed"
 *  gate. A profile with Local State + localStorage but NO Cookies DB cannot pass
 *  a server-side (SSR) auth gate like GuildOS /caravan, so dispatch must gate on
 *  THIS, not hasMasterProfile(). (Verified 2026-06-16: cookie-less master → /caravan
 *  302→/signin.) */
export function hasMasterCookie() {
  return (
    existsSync(join(MASTER_DIR, "Default", "Network", "Cookies")) ||
    existsSync(join(MASTER_DIR, "Default", "Cookies"))
  );
}

/** Opportunistic self-arming gate for browser dispatch. Returns true once a
 *  session cookie is present. If it is NOT yet present, attempts a
 *  captureMasterFromChrome() — which succeeds the moment the operator's Chrome
 *  isn't holding Cookies under its exclusive lock (i.e. next time Chrome is
 *  closed). Cheap when already armed (cookie present → no copy). This is what
 *  makes the pipeline require ZERO operator action: it captures the live login
 *  whenever Chrome is next down, with no interactive capture step. */
export function ensureMasterFromChrome() {
  if (hasMasterCookie()) return true;
  try { captureMasterFromChrome(); } catch { /* locked / no chrome — stays unarmed, retry next tick */ }
  return hasMasterCookie();
}

/** Resolve the live Chrome "User Data" dir from the located chrome.exe
 *  (.../Application/chrome.exe → .../User Data), with a LOCALAPPDATA fallback. */
export function chromeUserDataDir() {
  const exe = findChrome();
  if (exe) {
    const ud = join(dirname(dirname(exe)), "User Data"); // Application → Chrome → User Data
    if (existsSync(ud)) return ud;
  }
  const fb = process.env.LOCALAPPDATA && join(process.env.LOCALAPPDATA, "Google/Chrome/User Data");
  return fb && existsSync(fb) ? fb : null;
}

/**
 * Capture the master profile NON-INTERACTIVELY by copying the operator's LIVE
 * Chrome login state into MASTER_DIR. Zero-operation alternative to the headed
 * captureMasterProfileCmd login — correct when the operator is already logged into
 * the target sites in their main Chrome and wants no manual step. Copies only the
 * auth allowlist (AUTH_CAPTURE_PATHS) and tolerates per-file locks (Chrome may be
 * running), so a single locked file degrades to a skip, not a thrown capture.
 * @returns {{ ok:boolean, masterDir:string, copied:string[], skipped:string[], reason?:string }}
 */
export function captureMasterFromChrome({ profile = "Default", userDataDir = null } = {}) {
  const ud = userDataDir || chromeUserDataDir();
  if (!ud) return { ok: false, masterDir: MASTER_DIR, copied: [], skipped: [], reason: "chrome_user_data_not_found" };
  // NON-DESTRUCTIVE (no rmSync wipe): a re-run that can't grab the live-locked
  // Cookies must PRESERVE a previously captured cookie, not destroy it. Per-file
  // skip-if-exists keeps repeated opportunistic calls cheap — only the Cookies
  // file is re-attempted every time (it's the one under Chrome's exclusive lock,
  // so we keep retrying until a Chrome-closed moment lets it through).
  mkdirSync(MASTER_DIR, { recursive: true });
  const copied = [];
  const skipped = [];
  for (const tmpl of AUTH_CAPTURE_PATHS) {
    const rel = tmpl.replace("{p}", profile);
    const isCookie = /Cookies/.test(rel);
    const src = join(ud, rel);
    const dest = join(MASTER_DIR, rel);
    if (existsSync(dest) && !isCookie) { skipped.push(`${rel} (have)`); continue; }
    if (!existsSync(src)) { skipped.push(`${rel} (absent)`); continue; }
    try {
      lockTolerantCopy(src, dest);
      copied.push(rel);
    } catch (e) {
      skipped.push(`${rel} (${e.code || e.message})`);
    }
  }
  const ok = hasMasterCookie();
  logWeaponCall({
    weapon: "leafBrowser", fn: "captureMasterFromChrome",
    input: { userDataDir: ud, profile },
    output: { copied: copied.length, skipped: skipped.length, ok },
    status: ok ? "ok" : "fail",
  });
  return { ok, masterDir: MASTER_DIR, copied, skipped };
}

/** Locate chrome.exe: registry App Paths first (authoritative on Windows), then known paths. */
export function findChrome() {
  if (process.platform === "win32") {
    for (const hive of ["HKCU", "HKLM"]) {
      try {
        const out = execSync(
          `reg query "${hive}\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\App Paths\\chrome.exe" /ve`,
          { encoding: "utf8", stdio: ["ignore", "pipe", "ignore"] },
        );
        const m = out.match(/REG_SZ\s+(.+chrome\.exe)/i);
        if (m && existsSync(m[1].trim())) return m[1].trim();
      } catch { /* try next */ }
    }
  }
  const candidates = [
    process.env.LOCALAPPDATA && join(process.env.LOCALAPPDATA, "Google/Chrome/Application/chrome.exe"),
    "C:/Program Files/Google/Chrome/Application/chrome.exe",
    "C:/Program Files (x86)/Google/Chrome/Application/chrome.exe",
    "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
    "/usr/bin/google-chrome",
  ].filter(Boolean);
  for (const c of candidates) if (existsSync(c)) return c;
  return null;
}

export function captureMasterProfileCmd() {
  const chrome = findChrome();
  return {
    masterDir: MASTER_DIR,
    chrome,
    instructions:
      "Run this ONCE (headed). A fresh Chrome opens on the master profile — log into every site leaves must reach, then close Chrome. Re-run anytime to refresh logins.",
    command: chrome ? `"${chrome}" --user-data-dir="${MASTER_DIR}"` : null,
    cli: "node libs/weapon/leafBrowser/cli.mjs capture",
  };
}

/** Selective recursive copy of the master profile into destDir, skipping cache dirs. */
function copyProfile(srcDir, destDir) {
  mkdirSync(destDir, { recursive: true });
  cpSync(srcDir, destDir, {
    recursive: true,
    force: true,
    errorOnExist: false,
    filter: (src) => {
      const base = src.split(/[\\/]/).pop();
      return !EXCLUDE_DIR_NAMES.has(base);
    },
  });
}

/**
 * Prepare a per-leaf authenticated browser. Copies master→leaf profile and
 * writes an --mcp-config JSON for @playwright/mcp pointed at the copy.
 * @param {{ workerId:string, headless?:boolean }} opts
 * @returns {{ ok:true, mcpConfigPath:string, profileDir:string, serverName:string }
 *          | { ok:false, reason:string }}
 */
export function prepareLeafBrowser({ workerId, headless = true, allowIsolated = false, directProfileDir = null, outputDir = null } = {}) {
  if (!workerId) return fail("workerId required");
  const chrome = findChrome();
  if (!chrome) return fail("chrome_not_found");

  // Profile mode selection:
  //  - directProfileDir given → DIRECT: use that profile IN PLACE (no copy). Needed
  //    for Google/Gmail, where copying re-triggers the device-binding passkey wall
  //    (ED 2026-06-18) — the operator's native interactive login must be reused as-is.
  //    Single-flight (Chrome locks a user-data-dir); never cleaned up by cleanupLeafBrowser.
  //  - master profile present  → AUTHENTICATED: per-leaf copy via --user-data-dir
  //    (carries the operator's logins; needed for auth-walled targets).
  //  - no master, allowIsolated → ISOLATED: ephemeral in-memory profile (--isolated).
  //  - no master, !allowIsolated → refuse (caller must capture a profile first).
  const useDirect = !!(directProfileDir && existsSync(directProfileDir));
  const authed = useDirect || hasMasterProfile();
  if (!authed && !allowIsolated) return fail("no_master_profile");

  const profileDir = useDirect ? directProfileDir : (authed ? join(LEAF_ROOT, workerId) : null);
  const mcpConfigPath = join(MCP_TMP_ROOT, `${workerId}.json`);
  try {
    if (authed && !useDirect) {
      rmSync(profileDir, { recursive: true, force: true });
      copyProfile(MASTER_DIR, profileDir);
    }
    mkdirSync(dirname(mcpConfigPath), { recursive: true });
    const args = [
      "-y",
      "@playwright/mcp@latest",
      ...(headless ? ["--headless"] : []),
      ...(authed ? ["--user-data-dir", profileDir] : ["--isolated"]),
      "--executable-path",
      chrome,
    ];
    if (outputDir) { mkdirSync(outputDir, { recursive: true }); args.push("--output-dir", outputDir); }
    // Windows spawn gotcha: an MCP host spawns the server command directly (no
    // shell), and `npx` on Windows is `npx.cmd` — a bare "npx" fails to launch, the
    // server never connects, and the leaf silently sees NO mcp__playwright__* tools
    // (observed 2026-06-18). Wrap via `cmd /c` on win32 so npx resolves.
    const server = process.platform === "win32"
      ? { command: "cmd", args: ["/c", "npx", ...args] }
      : { command: "npx", args };
    const config = { mcpServers: { playwright: server } };
    writeFileSync(mcpConfigPath, JSON.stringify(config, null, 2));
    const mode = useDirect ? "direct-profile" : authed ? "authenticated" : "isolated";
    const out = { ok: true, mcpConfigPath, profileDir, serverName: "playwright", mode };
    logWeaponCall({ weapon: "leafBrowser", fn: "prepareLeafBrowser", input: { workerId, mode }, output: { profileDir }, status: "ok" });
    return out;
  } catch (e) {
    if (profileDir && !useDirect) { try { rmSync(profileDir, { recursive: true, force: true }); } catch { /* ignore */ } }
    return fail(`prepare_failed: ${e.message}`);
  }
}

export function cleanupLeafBrowser({ workerId } = {}) {
  if (!workerId) return { ok: false, reason: "workerId required" };
  try {
    rmSync(join(LEAF_ROOT, workerId), { recursive: true, force: true });
    rmSync(join(MCP_TMP_ROOT, `${workerId}.json`), { force: true });
    return { ok: true };
  } catch (e) {
    return { ok: false, reason: e.message };
  }
}

function fail(reason) {
  logWeaponCall({ weapon: "leafBrowser", fn: "prepareLeafBrowser", input: {}, output: { reason }, status: "fail" });
  return { ok: false, reason };
}

/**
 * Open a HEADED, persistent Playwright browser on a DEDICATED profile and
 * navigate to `url` so the OPERATOR logs in BY HAND (passkey/2FA). The cookie-copy
 * master fails for Google because the session was minted in a DIFFERENT browser
 * (passkey challenge on a "new" device); logging in HERE binds the session to THIS
 * profile natively, so it PERSISTS across restarts for days. Anti-automation flags
 * (no --enable-automation, navigator.webdriver spoofed, real system Chrome) so
 * Google does not flag the sign-in. Stays open until the operator closes the
 * window — everything is written to the profile dir live, so the login survives.
 * Reuse the SAME profileDir DIRECTLY (never a copy) to keep the session valid.
 * @returns {Promise<{ ok:true, profileDir:string } | { ok:false, reason:string }>}
 */
export async function openInteractiveLogin({ url = "https://mail.google.com/", profileName = "interactive", seedStatePath = null, resetProfile = false } = {}) {
  let chromium;
  try { ({ chromium } = await import("playwright")); }
  catch (e) { return { ok: false, reason: `playwright_unavailable: ${e.message}` }; }
  const chrome = findChrome();
  const profileDir = join(PROFILES_ROOT, profileName);
  // Reset wipes a stale/blank prior session so the seed starts clean (the empty
  // interactive profile otherwise shows a BARE sign-in with no account).
  if (resetProfile) rmSync(profileDir, { recursive: true, force: true });
  mkdirSync(profileDir, { recursive: true });
  let context;
  try {
    context = await chromium.launchPersistentContext(profileDir, {
      headless: false,
      ...(chrome ? { executablePath: chrome } : {}),
      ignoreDefaultArgs: ["--enable-automation"],
      args: ["--disable-blink-features=AutomationControlled", "--no-first-run", "--no-default-browser-check", "--start-maximized"],
      viewport: null,
    });
  } catch (e) {
    return { ok: false, reason: `launch_failed: ${e.message}` };
  }
  // Seed the account from a storageState JSON (exportStorageState output) so the
  // target RECOGNIZES the operator before they touch it — Gmail shows the account
  // chooser with their email instead of a blank sign-in. Cookies write into the
  // persistent profile, so the eventual login they complete is durable here.
  let seeded = 0;
  if (seedStatePath && existsSync(seedStatePath)) {
    try {
      const state = JSON.parse(readFileSync(seedStatePath, "utf8"));
      if (Array.isArray(state.cookies) && state.cookies.length) {
        await context.addCookies(state.cookies);
        seeded = state.cookies.length;
      }
    } catch { /* seeding is best-effort */ }
  }
  await context.addInitScript(() => {
    Object.defineProperty(navigator, "webdriver", { get: () => undefined });
  });
  const page = context.pages()[0] || (await context.newPage());
  await page.goto(url, { waitUntil: "domcontentloaded", timeout: 60000 }).catch(() => {});
  logWeaponCall({ weapon: "leafBrowser", fn: "openInteractiveLogin", input: { url, profileName, seeded }, output: { profileDir }, status: "ok" });
  console.log(`[leafBrowser] interactive browser OPEN on ${profileDir} at ${url} (seeded ${seeded} cookies) — log in by hand; the session persists in that profile. Close the window when done.`);
  await new Promise((resolve) => context.on("close", resolve));
  return { ok: true, profileDir, seeded };
}

/**
 * Export a profile's cookies + localStorage as a Playwright storageState JSON —
 * the stable, plaintext, lock-free auth artifact a leaf imports via
 * `newContext({ storageState })`. Reads from a persistent context WITHOUT
 * navigating (so it never triggers a token refresh/rotation that could disturb
 * the source session). The master profile's copied cookies recognize the
 * operator's Google account, so this JSON makes Gmail show the account chooser.
 * @returns {Promise<{ ok:true, path:string, cookieCount:number } | { ok:false, reason:string }>}
 */
export async function exportStorageState({ fromProfile = "master", outPath = null } = {}) {
  let chromium;
  try { ({ chromium } = await import("playwright")); }
  catch (e) { return { ok: false, reason: `playwright_unavailable: ${e.message}` }; }
  const profileDir = join(PROFILES_ROOT, fromProfile);
  if (!existsSync(profileDir)) return { ok: false, reason: `profile_not_found: ${fromProfile}` };
  const dest = outPath || join(PROFILES_ROOT, `${fromProfile}-state.json`);
  const chrome = findChrome();
  let context;
  try {
    context = await chromium.launchPersistentContext(profileDir, {
      headless: true, ...(chrome ? { executablePath: chrome } : {}),
      ignoreDefaultArgs: ["--enable-automation"],
    });
  } catch (e) { return { ok: false, reason: `launch_failed: ${e.message}` }; }
  let state;
  try { state = await context.storageState(); }
  finally { await context.close(); }
  writeFileSync(dest, JSON.stringify(state));
  logWeaponCall({ weapon: "leafBrowser", fn: "exportStorageState", input: { fromProfile }, output: { dest, cookieCount: state.cookies.length }, status: "ok" });
  return { ok: true, path: dest, cookieCount: state.cookies.length };
}

// ───────────────────────────────────────────────────────────────────────────
// SURFACE 2 — in-process render-to-PNG (the consolidated screenshot substrate).
// Everything that used to do `await import("playwright"); chromium.launch(...)`
// in its own weapon (questPurrview, cicVerify, poster_render, hand-rolled leaf
// fallbacks) now routes through launchContext / screenshotUrl. See ED 2026-06-15.
// ───────────────────────────────────────────────────────────────────────────

/** Resolve a deliverable ref / target to a navigable URL.
 *  - absolute http(s)/file/data → unchanged.
 *  - public-relative ("proving-grounds/x/index.html", "/poster-templates/…") →
 *    resolved against baseUrl (the GuildOS dev server, default :3002).
 *  Returns { url, fileFallback } where fileFallback is a file:// URL into
 *  ./public for static assets when the dev server is down. */
export function resolveTargetUrl(ref, { baseUrl = "http://localhost:3002" } = {}) {
  const raw = String(ref || "").trim();
  if (!raw) return { url: null, fileFallback: null };
  if (/^(https?|file|data):/i.test(raw)) return { url: raw, fileFallback: null };
  const rel = raw.replace(/^\/+/, "");
  const url = `${baseUrl.replace(/\/+$/, "")}/${rel}`;
  const fileFallback = "file:///" + join(process.cwd(), "public", rel).replace(/\\/g, "/");
  return { url, fileFallback };
}

/** Ensure a per-leaf authenticated profile copy exists; returns its dir. */
function ensureAuthProfile(workerId) {
  const profileDir = join(LEAF_ROOT, workerId || "screenshot");
  rmSync(profileDir, { recursive: true, force: true });
  copyProfile(MASTER_DIR, profileDir);
  return profileDir;
}

/**
 * The ONLY place in the repo that launches Chromium. Returns a ready browser
 * context with the auth decision applied.
 *   - authed:true + a captured master → persistent context over a profile copy
 *     (carries the operator's logins), driving SYSTEM Chrome via executablePath.
 *   - otherwise → ephemeral bundled-Chromium context (no login). Correct for
 *     public / localhost / unauthenticated targets (the common case).
 * @returns {Promise<{ ok:true, context, browser, mode:"authenticated"|"isolated", close:()=>Promise<void> }
 *          | { ok:false, reason:string }>}
 */
export async function launchContext({
  authed = false,
  workerId = null,
  headless = true,
  viewport = { width: 1280, height: 1600 },
  deviceScaleFactor = 1,
  directProfileDir = null,
} = {}) {
  let chromium;
  try { ({ chromium } = await import("playwright")); }
  catch (e) { return { ok: false, reason: `playwright_unavailable: ${e.message}` }; }

  // directProfileDir: use a native login profile IN PLACE (no copy) — required for
  // Google/Gmail which reject copied sessions with a passkey wall (ED 2026-06-18).
  if (directProfileDir && existsSync(directProfileDir)) {
    const exe = findChrome();
    try {
      const context = await chromium.launchPersistentContext(directProfileDir, {
        headless,
        viewport,
        deviceScaleFactor,
        ...(exe ? { executablePath: exe } : {}),
      });
      return {
        ok: true, context, browser: context.browser(), mode: "direct-profile",
        close: async () => { try { await context.close(); } catch { /* ignore */ } },
      };
    } catch (e) {
      return { ok: false, reason: `launch_failed_direct: ${e.message}` };
    }
  }

  const useAuth = authed && hasMasterProfile();
  try {
    if (useAuth) {
      const exe = findChrome();
      const profileDir = ensureAuthProfile(workerId);
      const context = await chromium.launchPersistentContext(profileDir, {
        headless,
        viewport,
        deviceScaleFactor,
        ...(exe ? { executablePath: exe } : {}),
      });
      return {
        ok: true, context, browser: context.browser(), mode: "authenticated",
        close: async () => {
          try { await context.close(); } catch { /* ignore */ }
          try { rmSync(profileDir, { recursive: true, force: true }); } catch { /* ignore */ }
        },
      };
    }
    const exe = findChrome();
    const browser = await chromium.launch({
      headless,
      ...(exe ? { executablePath: exe } : {}),
    });
    const context = await browser.newContext({ viewport, deviceScaleFactor });
    return {
      ok: true, context, browser, mode: "isolated",
      close: async () => { try { await browser.close(); } catch { /* ignore */ } },
    };
  } catch (e) {
    return { ok: false, reason: `launch_failed: ${e.message}` };
  }
}

/**
 * Render a target (URL or public-relative ref) to a full-page PNG. THE single
 * screenshot primitive — absorbs the logic the divergent weapons each had:
 * pdf.js multi-page rendering, lazy-load scrolling, fail-closed-on-blank,
 * optional element (`selector`) shots, and dev-server→file:// fallback.
 *
 * @param {{
 *   ref?:string, url?:string, kind?:string,
 *   fullPage?:boolean, selector?:string,
 *   viewport?:{width:number,height:number}, deviceScaleFactor?:number,
 *   waitUntil?:"load"|"domcontentloaded"|"networkidle",
 *   scrollLazyLoad?:boolean, failClosedOnBlank?:boolean,
 *   baseUrl?:string, timeoutMs?:number,
 *   authed?:boolean, workerId?:string, encoding?:"buffer"|"base64",
 *   beforeShot?:string,   // stringified fn body run via page.evaluate before the shot
 * }} opts
 * @returns {Promise<{ ok:true, buffer?:Buffer, base64?:string, mode:string, resolvedUrl:string }
 *          | { ok:false, error:string }>}
 */
export async function screenshotUrl(opts = {}) {
  const {
    ref, url, kind,
    fullPage = true, selector = null,
    viewport = { width: 1280, height: 1600 }, deviceScaleFactor = 1,
    waitUntil = "networkidle",
    scrollLazyLoad = true, failClosedOnBlank = true,
    baseUrl = "http://localhost:3002", timeoutMs = 30000,
    authed = false, workerId = null, encoding = "buffer",
    directProfileDir = null,
  } = opts;

  const target = url || ref;
  if (!target) return { ok: false, error: "ref_or_url_required" };
  const { url: primary, fileFallback } = url ? { url, fileFallback: null } : resolveTargetUrl(ref, { baseUrl });

  const launched = await launchContext({ authed, workerId, viewport, deviceScaleFactor, directProfileDir });
  if (!launched.ok) return { ok: false, error: launched.reason };
  const { context, mode, close } = launched;

  try {
    const page = await context.newPage();
    const isPdf = kind === "document" || /\.pdf(\?|#|$)/i.test(primary || "");

    let buffer;
    if (isPdf) {
      buffer = await renderPdf(page, primary, timeoutMs);
    } else {
      let resolvedUrl = primary;
      let resp = await page.goto(primary, { waitUntil, timeout: timeoutMs }).catch(() => null);
      let contentLen = await pageTextLen(page);
      // dev-server down? fall back to file:// for static public assets.
      if ((!resp || !resp.ok()) && contentLen === 0 && fileFallback) {
        resolvedUrl = fileFallback;
        resp = await page.goto(fileFallback, { waitUntil: "load", timeout: timeoutMs }).catch(() => null);
        contentLen = await pageTextLen(page);
      }
      if (scrollLazyLoad) await autoScroll(page);
      contentLen = await pageTextLen(page);
      if (failClosedOnBlank && (!resp || !resp.ok()) && contentLen === 0) {
        throw new Error(`navigation to ${resolvedUrl} failed (status=${resp ? resp.status() : "no-response"}) and page rendered empty`);
      }
      if (selector) {
        const el = await page.$(selector);
        if (!el) throw new Error(`selector not found: ${selector}`);
        buffer = await el.screenshot({ type: "png" });
      } else {
        buffer = await page.screenshot({ fullPage, type: "png" });
      }
    }

    const out = { ok: true, mode, resolvedUrl: primary };
    if (encoding === "base64") out.base64 = buffer.toString("base64"); else out.buffer = buffer;
    logWeaponCall({ weapon: "leafBrowser", fn: "screenshotUrl", input: { target: primary, mode, authed: mode === "authenticated" }, output: { bytes: buffer.length }, status: "ok" });
    return out;
  } catch (e) {
    logWeaponCall({ weapon: "leafBrowser", fn: "screenshotUrl", input: { target: primary }, output: { error: e.message }, status: "fail" });
    return { ok: false, error: (e.message || String(e)).split("\n")[0] };
  } finally {
    await close();
  }
}

async function pageTextLen(page) {
  return page.evaluate(() => (document.body ? document.body.innerText.trim().length : 0)).catch(() => 0);
}

async function autoScroll(page) {
  await page
    .evaluate(
      () =>
        new Promise((res) => {
          let y = 0;
          const t = setInterval(() => {
            window.scrollBy(0, 500);
            y += 500;
            if (y >= document.body.scrollHeight) { clearInterval(t); res(); }
          }, 120);
        }),
    )
    .catch(() => {});
  await page.waitForTimeout(800);
  await page.evaluate(() => window.scrollTo(0, 0)).catch(() => {});
}

/** pdf.js renders pages 1..N (capped) stacked into one canvas column, then we
 *  screenshot the stack. Headless Chromium downloads PDFs instead of viewing,
 *  so we render them ourselves. */
async function renderPdf(page, ref, timeoutMs) {
  const ver = "3.11.174";
  await page.setContent(
    `<!DOCTYPE html><html><head><script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/${ver}/pdf.min.js"></script></head><body style="margin:0;background:#fff"><div id="pages"></div></body></html>`,
    { waitUntil: "load", timeout: timeoutMs },
  );
  const meta = await page.evaluate(async (pdfUrl) => {
    const L = window.pdfjsLib;
    if (!L) return { ok: false };
    L.GlobalWorkerOptions.workerSrc = "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js";
    const pdf = await L.getDocument(pdfUrl).promise;
    const MAX_PAGES = 15;
    const n = Math.min(pdf.numPages, MAX_PAGES);
    const container = document.getElementById("pages");
    for (let i = 1; i <= n; i++) {
      const p = await pdf.getPage(i);
      const vp = p.getViewport({ scale: 1.5 });
      const c = document.createElement("canvas");
      c.width = vp.width; c.height = vp.height; c.style.display = "block";
      container.appendChild(c);
      await p.render({ canvasContext: c.getContext("2d"), viewport: vp }).promise;
    }
    return { ok: true, total: pdf.numPages, rendered: n };
  }, ref);
  if (!meta?.ok || !meta.rendered) throw new Error("pdf.js failed to render the document");
  const el = await page.$("#pages");
  return el.screenshot();
}

/**
 * Navigate to a URL using a direct Chrome profile and return the page text content.
 * Designed for reading Google Drive / Google Docs / Sheets content using the
 * interactive profile (which holds the operator's Google session).
 *
 * @param {{ url:string, directProfileDir?:string, authed?:boolean, workerId?:string,
 *           timeoutMs?:number, waitUntil?:string, selector?:string }} opts
 * @returns {Promise<{ ok:true, text:string, url:string, mode:string }
 *          | { ok:false, error:string }>}
 */
export async function navigateAndGetText({
  url,
  directProfileDir = null,
  authed = false,
  workerId = null,
  timeoutMs = 30000,
  waitUntil = "networkidle",
  selector = null,
} = {}) {
  if (!url) return { ok: false, error: "url required" };
  const launched = await launchContext({ authed, workerId, directProfileDir });
  if (!launched.ok) return { ok: false, error: launched.reason };
  const { context, mode, close } = launched;
  try {
    const page = await context.newPage();
    await page.goto(url, { waitUntil, timeout: timeoutMs }).catch(() => null);
    await autoScroll(page);
    let text;
    if (selector) {
      const el = await page.$(selector);
      text = el ? await el.innerText() : await page.evaluate(() => document.body?.innerText || "");
    } else {
      text = await page.evaluate(() => document.body?.innerText || "");
    }
    return { ok: true, text: text.trim(), url, mode };
  } catch (e) {
    return { ok: false, error: `navigate_failed: ${e.message}` };
  } finally {
    try { await close(); } catch { /* ignore */ }
  }
}

/**
 * Navigate to a URL and evaluate a JS expression on the page, returning the result.
 * Use for extracting structured data (links, folder IDs, file listings) from pages.
 *
 * @param {{ url:string, jsExpr:string, directProfileDir?:string, authed?:boolean,
 *           workerId?:string, timeoutMs?:number, waitUntil?:string }} opts
 * @returns {Promise<{ ok:true, result:any, url:string, mode:string }
 *          | { ok:false, error:string }>}
 */
export async function navigateAndEvalJs({
  url,
  jsExpr,
  directProfileDir = null,
  authed = false,
  workerId = null,
  timeoutMs = 30000,
  waitUntil = "networkidle",
} = {}) {
  if (!url) return { ok: false, error: "url required" };
  if (!jsExpr) return { ok: false, error: "jsExpr required" };
  const launched = await launchContext({ authed, workerId, directProfileDir });
  if (!launched.ok) return { ok: false, error: launched.reason };
  const { context, mode, close } = launched;
  try {
    const page = await context.newPage();
    await page.goto(url, { waitUntil, timeout: timeoutMs }).catch(() => null);
    await autoScroll(page);
    const result = await page.evaluate((expr) => {
      try { return eval(expr); } catch (e) { return { _evalError: e.message }; }
    }, jsExpr);
    return { ok: true, result, url, mode };
  } catch (e) {
    return { ok: false, error: `navigate_eval_failed: ${e.message}` };
  } finally {
    try { await close(); } catch { /* ignore */ }
  }
}

/** Read a prepared mcp-config (debug/inspection helper). */
export function readMcpConfig(workerId) {
  const p = join(MCP_TMP_ROOT, `${workerId}.json`);
  return existsSync(p) ? JSON.parse(readFileSync(p, "utf8")) : null;
}
```
<!-- /GUILDOS-FILE -->
