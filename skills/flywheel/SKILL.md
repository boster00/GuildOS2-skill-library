# flywheel

<!-- GUILDOS-SKILL name="flywheel" -->

<!-- GUILDOS-DESC -->
Control Flywheel-hosted WordPress sites. No public API/MCP exists — platform settings are CIC-UI-only, WordPress internals are WP-CLI-over-SSH. Covers the dashboard URL/click map for CIC (navigationMap), site notification recipients (notificationEmails), SSH/SFTP credentials get/set + keys (sshCredentials), backups/rollbacks/staging + privacy gating (backupsAndStaging), and creating a WordPress administrator (newWpAdmin). Companion flywheel weapon holds the deep-links and WP-CLI/SSH wrappers.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/flywheel/SKILL.md -->
```markdown
---
name: flywheel
description: >-
  Control Flywheel-hosted WordPress sites. No public API/MCP exists — platform
  settings are CIC-UI-only, WordPress internals are WP-CLI-over-SSH. Covers the
  dashboard URL/click map for CIC (navigationMap), site notification recipients
  (notificationEmails), SSH/SFTP credentials get/set + keys (sshCredentials),
  backups/rollbacks/staging + privacy gating (backupsAndStaging), and creating a
  WordPress administrator (newWpAdmin). Companion flywheel weapon holds the
  deep-links and WP-CLI/SSH wrappers.
---

# Flywheel (WordPress hosting)

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/flywheel/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto flywheel <action>` (or read the source directly).

## Actions

- **backupsAndStaging** — Capability #3 — backups, rollbacks, staging sites, and security-gating staging. All UI-only via CIC. Backups tab (download/restore + manual), Staging tab (create/go-live/reset), and Privacy Mode for gating staging behind no-index/visibility.
- **navigationMap** — The canonical Flywheel dashboard URL + click map so CIC can navigate directly instead of hunting. Covers the site tabs (Overview/Plugins/Performance/Stats/Backups/Staging/Advanced) and account pages (profile/security/ssh).
- **newWpAdmin** — Capability #4 — create a new WordPress administrator. Flywheel has no UI button for this; do it inside WordPress. Scriptable path: WP-CLI over the SSH gateway (flywheel weapon createWpAdmin). Manual path: WP-ADMIN one-click SSO → Users → Add New.
- **notificationEmails** — Capability #1 — control which email addresses receive a site's notification emails. In Flywheel this is the per-site Collaborators list (Overview tab); there is no separate per-notification toggle. Plus the SendGrid 'Email Deliverability' setup on Advanced governs outbound WP mail.
- **sshCredentials** — Capability #2 — get and set SSH/SFTP credentials. GET: the per-site SSH command lives on Advanced; SFTP host/user on /profile. SET: the password is the Flywheel account password (/profile/security); headless access is via SSH public keys (/profile/ssh).
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/flywheel/index.js -->
```js
/**
 * Flywheel skill book.
 *
 * How to control Flywheel-hosted WordPress sites (app.getflywheel.com), given
 * that NO public Flywheel management API and NO MCP server exist. Control is a
 * HYBRID of (a) CIC-driven UI clicks for platform settings and (b) WP-CLI over
 * the Flywheel SSH gateway for WordPress-level changes.
 *
 * Companion weapon: `libs/weapon/flywheel` — deterministic deep-links
 * (dashboardUrls), the SSH/WP-CLI wrappers (wpCli, createWpAdmin), and the
 * connection facts (getConnectionInfo, listKnownSites).
 *
 * --- API / MCP VERDICT (verified 2026-06-03 via CIC) ---------------------
 * The legacy Flywheel platform has no customer API and no API-token UI.
 * Swept exhaustively while logged in as CJ (account "cjxia", org "sijiexia1"):
 *   - Top nav MANAGE → only Organizations + Blueprints.
 *   - Account dropdown → only Account / Billing / Security / Help / Sign out.
 *   - /profile tabs → ACCOUNT, SECURITY, SSH. None expose tokens/integrations.
 *   - Dashboard makes no public `/api/...` calls a customer can reuse.
 * WP Engine's API (wpengineapi.com) is provisioned only inside the
 * my.wpengine.com portal; this account is not migrated there → unavailable.
 * Conclusion: drive the UI through CIC; use SSH/WP-CLI for WordPress internals.
 * -------------------------------------------------------------------------
 *
 * Account snapshot (2026-06-03): 1 site — "Kyinno Bio Live Site"
 *   slug `kyinno-bio-live-site`, org `sijiexia1`, domain www.kyinno.com,
 *   staging <REDACTED-HOST>, Cloud Platform, WP 6.9.4 (upgrade
 *   to 7.0 pending; Flywheel auto-upgrades starting 2026-06-08), PHP 8.2.
 *   Collaborators: Sijie Xia (OWNER, <REDACTED-EMAIL>), CJ Xia
 *   (<REDACTED-EMAIL>), Paul Ryazanov, zhang tao, Rienzi Mosqueda.
 */

export const skillBook = {
  id: "flywheel",
  title: "Flywheel (WordPress hosting)",
  description:
    "Control Flywheel-hosted WordPress sites. No public API/MCP exists — platform settings are CIC-UI-only, WordPress internals are WP-CLI-over-SSH. Actions: navigationMap (the dashboard URL/click map for CIC), notificationEmails (capability #1 — who receives site emails), sshCredentials (capability #2 — get/set SSH & SFTP access + keys), backupsAndStaging (capability #3 — backups, rollbacks, staging, privacy gating), newWpAdmin (capability #4 — create a WordPress administrator).",
  steps: [],
  toc: {
    navigationMap: {
      description:
        "The canonical Flywheel dashboard URL + click map so CIC can navigate directly instead of hunting. Covers the site tabs (Overview/Plugins/Performance/Stats/Backups/Staging/Advanced) and account pages (profile/security/ssh).",
      howTo: `
Flywheel is a React SPA at https://app.getflywheel.com. Deep-links work for
most pages, but a few tabs only mount via in-app click (noted below). Use the
\`flywheel\` weapon's \`dashboardUrls({ site })\` to build these programmatically.

## Site URL shape
\`https://app.getflywheel.com/<org>/<site-slug>\` (e.g. \`/sijiexia1/kyinno-bio-live-site\`).

| Page | URL suffix | What's there |
|---|---|---|
| Overview | (none) | Collaborators list (notification recipients), privacy mode, IP, screenshot |
| Plugins | \`/plugins\` | Plugin list / updates |
| Performance | \`/performance\` | Perf metrics |
| Stats | \`/stats\` | Traffic stats |
| Backups | \`/backups\` | Daily auto-backups, ADD NEW BACKUPS, per-backup Download / Restore |
| Staging | \`/staging_site\` | **NOTE: path is \`staging_site\`, not \`staging\`.** Go Live, Reset staging, Staging WP-Admin, staging privacy mode |
| Advanced | \`/advanced\` | SSH command, staging toggle, Force HTTPS, CDN, WP_DEBUG, multisite, **Email Deliverability (SendGrid)**, PHP/DB, redirects, export logs |
| WP-Admin (live) | — | "WP-ADMIN" button = one-click SSO into wp-admin |
| Staging WP-Admin | — | "STAGING WP-ADMIN" button on the staging tab |

## Account pages (not site-scoped)
| Page | URL | What's there |
|---|---|---|
| Account/profile | \`/profile\` | Username, name, email, **SFTP Access block** (<REDACTED-HOST>:22, user, "your Flywheel password") |
| Security | \`/profile/security\` | **Change password** (this IS the SFTP/SSH password), enable 2FA |
| SSH keys | \`/profile/ssh\` | **ADD SSH KEY** — register public keys for headless SSH/WP-CLI |
| Sites list | \`/sites\` | All sites, CSV export, filter |
| Organizations | \`/organizations\` | Org/billing grouping (none configured) |

## CIC gotchas (observed 2026-06-03)
- Opening the account dropdown can freeze \`Page.captureScreenshot\` (renderer busy). Prefer \`find\`/\`read_page\` for hrefs, or navigate by URL, instead of screenshotting an open menu.
- \`/account\` is a 404 — the real path is \`/profile\`.
- Direct-navigating a tab whose route differs (e.g. \`/staging\`) bounces to \`/sites\`; use \`/staging_site\` or click the in-app tab.
`,
    },

    notificationEmails: {
      description:
        "Capability #1 — control which email addresses receive a site's notification emails. In Flywheel this is the per-site Collaborators list (Overview tab); there is no separate per-notification toggle. Plus the SendGrid 'Email Deliverability' setup on Advanced governs outbound WP mail.",
      howTo: `
Flywheel has **no granular "notification preferences" page**. Two levers:

## A. Who gets platform notification emails → the site's Collaborators
Site notification/alert emails (and dashboard access) go to the site's
**Collaborators**. Manage them on the **Overview** tab:
\`https://app.getflywheel.com/<org>/<site>\`

- Each collaborator is listed with their email and a **Remove** link.
- **ADD COLLABORATORS** adds a new email (they get an invite + thereafter the
  site's notification emails).
- The **OWNER** (here Sijie Xia / <REDACTED-EMAIL>) receives billing/critical
  notices and cannot be removed without transferring ownership.

To change who is notified: add the desired address as a collaborator and
**Remove** anyone who should stop receiving them. This is the only Flywheel-side
control over the recipient set. CIC steps:
1. Navigate to the Overview URL above.
2. \`find\` "ADD COLLABORATORS" → click → type the email → confirm; or
3. \`find\` the "Remove" link next to the address to drop → click → confirm.

## B. Outbound WordPress email (from the site to its users) → SendGrid
**Advanced** tab → **Email Deliverability** → **SET UP EMAIL** authenticates
the site's outbound mail through SendGrid (improves deliverability of WP-sent
mail like password resets / form notifications). This governs mail the site
*sends*, not who receives Flywheel platform alerts.

NOTE: if "notification emails" means a specific WordPress plugin's notifications
(e.g. form/WooCommerce emails), that recipient list lives inside wp-admin, not
Flywheel — reach it via the WP-ADMIN button or WP-CLI.
`,
    },

    sshCredentials: {
      description:
        "Capability #2 — get and set SSH/SFTP credentials. GET: the per-site SSH command lives on Advanced; SFTP host/user on /profile. SET: the password is the Flywheel account password (/profile/security); headless access is via SSH public keys (/profile/ssh).",
      howTo: `
Use the \`flywheel\` weapon's \`getConnectionInfo()\` for the facts, and
\`sshCommand({ site })\` / \`sshTarget({ site })\` to build connection strings.

## GET — read the credentials
- **Per-site SSH command** (Advanced tab):
  \`ssh <org>+<site>@<REDACTED-HOST>\`
  e.g. \`ssh sijiexia1+kyinno-bio-live-site@<REDACTED-HOST>\`
  (Advanced tab also has COPY COMMAND / OPEN IN TERMINAL buttons.)
- **SFTP** (/profile, "SFTP Access"): host \`<REDACTED-HOST>\`, port 22,
  username = your account username (\`cjxia\`).
- **Password**: there is no separate SFTP/SSH password — it is your **Flywheel
  account password**.

## SET — change the credentials
- **Password** → /profile/security → **CHANGE** (verifies current password).
  This rotates SFTP + gateway-password auth for every site at once.
  ⚠ Irreversible-ish / account-wide → confirm with the user before changing.
- **SSH public key (recommended for headless / WP-CLI)** → /profile/ssh →
  **ADD SSH KEY** → paste the contents of \`~/.ssh/id_ed25519.pub\` (Windows:
  \`$env:USERPROFILE\\.ssh\\id_ed25519.pub\`). Once registered, the system
  \`ssh\` (BatchMode) used by the \`ssh\`/\`flywheel\` weapons authenticates with
  no password — this is what unlocks \`wpCli\` / \`createWpAdmin\`.
  A key **is** registered as of 2026-06-03: name "GuildOS nexus (xsj70)",
  this workstation's \`~/.ssh/id_ed25519\` (fingerprint ad:e8:39:84:78:d2:…).
  The WP-CLI path below is live and verified. Gateway is slow (~11s/connect) —
  use ConnectTimeout≥20 and don't treat a single timeout as failure.

## Verify a key works (after adding)
\`\`\`
ssh -o BatchMode=yes <org>+<site>@<REDACTED-HOST> "wp option get siteurl"
\`\`\`
BatchMode=yes means success proves key auth (not a silent password fallback).
`,
    },

    backupsAndStaging: {
      description:
        "Capability #3 — backups, rollbacks, staging sites, and security-gating staging. All UI-only via CIC. Backups tab (download/restore + manual), Staging tab (create/go-live/reset), and Privacy Mode for gating staging behind no-index/visibility.",
      howTo: `
All on the site's own tabs — UI-only, drive via CIC.

## Backups & rollbacks — Backups tab (\`/<org>/<site>/backups\`)
- Flywheel takes **daily automatic backups** (each lists posts/pages/plugins/
  comments + WP version, with **Download** and **Restore**).
- **ADD NEW BACKUPS** creates a manual restore point on demand (do this before
  risky changes).
- **Rollback** = click **Restore** on the target backup → confirm. Restore
  overwrites the live site with that snapshot → confirm with the user first
  (it's destructive to current state, though itself recoverable via a fresh
  backup taken just before).

## Staging — Staging tab (\`/<org>/<site>/staging_site\`)
- **Turn staging on**: Advanced tab → Site Options → **Staging** toggle (or
  the Overview "Staging" badge). Staging domain here is
  \`<REDACTED-HOST>\`.
- **Reset staging** = **RESET STAGING** (recreates the staging copy from live;
  current staging is lost → confirm).
- **Go live** = **LAUNCH STAGING CHANGES** — pushes staging → production. It
  asks what to include (files/DB) and confirms before pushing. Irreversible to
  production → ALWAYS take a fresh live backup first and confirm with the user.
- **Staging WP-Admin**: the **STAGING WP-ADMIN** button SSOs into the staging
  site's wp-admin.

## Security-gating staging — Privacy Mode
Staging is **not** password-protected by default, and its Privacy Mode is
**off** by default ("This may result in SEO penalties and unwanted visitor
activity"). To gate it:
- On the Staging tab (and Overview) there is a **Privacy Mode** control. Turn it
  **on** to hide the staging copy from visitors and search engines.
- Privacy Mode is the Flywheel mechanism for both staging and production
  visibility gating; there is no separate htpasswd UI on the Cloud Platform —
  Privacy Mode is the equivalent. (For true HTTP-auth on a path, do it in
  wp-admin / via a plugin or .htaccess over SSH.)
- Re-enable Privacy Mode on staging whenever it's left exposed.
`,
    },

    newWpAdmin: {
      description:
        "Capability #4 — create a new WordPress administrator. Flywheel has no UI button for this; do it inside WordPress. Scriptable path: WP-CLI over the SSH gateway (flywheel weapon createWpAdmin). Manual path: WP-ADMIN one-click SSO → Users → Add New.",
      howTo: `
Flywheel does **not** expose "create WP admin" in its own dashboard — the WP
user table lives inside WordPress. Two paths:

## A. Scriptable — WP-CLI over SSH (preferred, programmatic)
Prerequisite is DONE — an SSH key ("GuildOS nexus (xsj70)") is registered and
the path is verified working (2026-06-03). Use the weapon:
\`\`\`js
import { createWpAdmin } from "@/libs/weapon/flywheel";
await createWpAdmin({
  site: "kyinno-bio-live-site",          // org defaults to the recorded one
  username: "newadmin",
  email: "<REDACTED-EMAIL>",
  // password optional — omit to let WP-CLI generate + print one
});
\`\`\`
Under the hood: \`ssh <org>+<site>@<REDACTED-HOST> "wp user create <user> <email> --role=administrator --porcelain"\`
(\`--porcelain\` returns just the new user ID). Verify:
\`wpCli({ site, args: "user list --role=administrator --fields=ID,user_login,user_email" })\`.

Promote an existing user instead:
\`wpCli({ site, args: "user set-role <login-or-id> administrator" })\`.

## B. Manual — via the dashboard (no SSH key needed)
1. Site Overview → **WP-ADMIN** button = one-click SSO into wp-admin (no WP
   password needed).
2. In wp-admin: **Users → Add New** → fill username + email → set Role =
   **Administrator** → **Add New User**.
   (CIC can drive this if SSO lands you in wp-admin; the WP-ADMIN button opens
   a new tab — grab its tabId via tabs_context before acting.)

Creating an admin is reversible (delete/demote the user), so under the YOLO/
reversible policy just do it when asked — no escalation needed. Sending the new
admin their credentials by email IS an external send → confirm that part.
`,
    },
  },
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/flywheel/index.js -->
```js
/**
 * Flywheel weapon — control Flywheel-hosted WordPress sites.
 *
 * IMPORTANT — there is NO public Flywheel management API and NO MCP server.
 * The legacy Flywheel platform (app.getflywheel.com) exposes zero customer-
 * facing REST/GraphQL endpoints and no API-token UI (verified 2026-06-03 by
 * exhaustively sweeping MANAGE, the account dropdown, and every /profile tab —
 * ACCOUNT / SECURITY / SSH — none offer tokens or integrations). WP Engine's
 * API (wpengineapi.com) only serves accounts provisioned in the my.wpengine.com
 * portal; this Flywheel account is not migrated there, so it is unavailable.
 *
 * Therefore control is split two ways (HYBRID):
 *   1. Platform settings (backups, staging, notifications, SSH-key mgmt,
 *      privacy gating) — UI-ONLY, driven through CIC. This weapon supplies the
 *      deterministic deep-links; the `flywheel` skill book holds the click map.
 *   2. WordPress-level control (new admin user, plugins, users, content) —
 *      SCRIPTABLE over SSH/WP-CLI through the Flywheel SSH gateway. This weapon
 *      wraps that path via the existing `ssh` weapon.
 *
 * Prerequisite for the SSH/WP-CLI path: an SSH public key must be registered at
 * https://app.getflywheel.com/profile/ssh (none was present as of 2026-06-03 —
 * `addSshKey` howTo in the skill book covers it). The gateway authenticates by
 * key; the system `ssh` (BatchMode) the `ssh` weapon uses then works headless.
 */

import { executeCommand } from "@/libs/weapon/ssh";

export const toc = {
  dashboardUrls: "Build the deterministic Flywheel dashboard deep-links for a site (overview/advanced/backups/staging/wp-admin) + account pages, for CIC navigation.",
  sshCommand: "Return the raw `ssh <org>+<site>@<REDACTED-HOST>` connection command for a site.",
  sshTarget: "Return the {user, host} pair for the Flywheel SSH gateway, for passing to the ssh weapon.",
  wpCli: "Run an arbitrary WP-CLI command on a site over the Flywheel SSH gateway (requires a registered SSH key).",
  createWpAdmin: "Create a new WordPress administrator on a site via WP-CLI over SSH (capability #4).",
  getConnectionInfo: "Return the known SSH/SFTP connection facts (gateway host, SFTP host/port, account username).",
  listKnownSites: "Return the locally-recorded site registry (org slug, domains, staging domain).",
  checkCredentials: "Report whether system ssh is available + the gateway reachability hint.",
};

// ---------------------------------------------------------------------------
// Constants — public infrastructure, safe to hardcode. NO passwords here:
// the SFTP/SSH password is the Flywheel account password (set via the SECURITY
// tab, never stored by the agent); WP-CLI auth is by registered SSH key.
// ---------------------------------------------------------------------------

const DASHBOARD = "https://app.getflywheel.com";
const SSH_GATEWAY = "<REDACTED-HOST>"; // per-site gateway: <org>+<site>@<REDACTED-HOST>
const SFTP_HOST = "<REDACTED-HOST>";
const SFTP_PORT = 22;

/**
 * Locally-recorded sites. The org *slug* (used in dashboard URLs) can differ
 * from the SFTP account username — record both. Add new sites here as they are
 * onboarded; this is a convenience cache, not a source of truth (the dashboard
 * is). Verified 2026-06-03.
 */
const KNOWN = {
  account: { username: "cjxia", orgSlug: "sijiexia1" },
  sites: {
    "kyinno-bio-live-site": {
      org: "sijiexia1",
      domain: "www.kyinno.com",
      staging: "<REDACTED-HOST>",
      platform: "cloud",
    },
  },
};

// ---------------------------------------------------------------------------
// Deep-link builder (for CIC navigation)
// ---------------------------------------------------------------------------

/**
 * Build the Flywheel dashboard deep-links for a site, plus the account pages.
 * @param {{ org?: string, site: string }} input
 *   org defaults to the recorded org for a known site, else KNOWN.account.orgSlug.
 * @returns {Record<string, string>}
 */
export function dashboardUrls({ org, site } = {}) {
  if (!site) throw new Error("site (slug) is required");
  const o = org || KNOWN.sites[site]?.org || KNOWN.account.orgSlug;
  const base = `${DASHBOARD}/${o}/${site}`;
  return {
    overview: base,
    plugins: `${base}/plugins`,
    performance: `${base}/performance`,
    stats: `${base}/stats`,
    backups: `${base}/backups`,
    staging: `${base}/staging_site`, // NOTE: tab path is /staging_site, not /staging
    advanced: `${base}/advanced`, // SSH command, staging toggle, email deliverability, multisite, DB
    // Account-level (not site-scoped):
    account: `${DASHBOARD}/profile`,
    security: `${DASHBOARD}/profile/security`, // change account password (== SFTP/SSH password) + 2FA
    sshKeys: `${DASHBOARD}/profile/ssh`, // add/remove SSH public keys
    sites: `${DASHBOARD}/sites`,
    organizations: `${DASHBOARD}/organizations`,
  };
}

// ---------------------------------------------------------------------------
// SSH / WP-CLI path
// ---------------------------------------------------------------------------

/** The gateway login string: `<org>+<site>@<REDACTED-HOST>`. */
export function sshCommand({ org, site } = {}) {
  const { user, host } = sshTarget({ org, site });
  return `ssh ${user}@${host}`;
}

/** {user, host} for the Flywheel SSH gateway — feed to the ssh weapon. */
export function sshTarget({ org, site } = {}) {
  if (!site) throw new Error("site (slug) is required");
  const o = org || KNOWN.sites[site]?.org || KNOWN.account.orgSlug;
  return { user: `${o}+${site}`, host: SSH_GATEWAY };
}

/**
 * Run a WP-CLI command on a site over the Flywheel SSH gateway.
 * Requires a registered SSH key (see skill book `addSshKey`).
 * @param {{ org?: string, site: string, args: string }} input
 *   args: the WP-CLI args *after* `wp`, e.g. "plugin list --status=active".
 * @param {string} [userId]
 */
export async function wpCli({ org, site, args } = {}, userId) {
  if (!args) throw new Error("args (the WP-CLI command after `wp`) is required");
  const { user, host } = sshTarget({ org, site });
  return executeCommand({ host, user, command: `wp ${args}` }, userId);
}

/**
 * Create a new WordPress administrator on a site (capability #4).
 * If password is omitted, WP-CLI generates one and prints it (parse stdout).
 * @param {{ org?: string, site: string, username: string, email: string, password?: string }} input
 * @param {string} [userId]
 */
export async function createWpAdmin({ org, site, username, email, password } = {}, userId) {
  if (!username || !email) throw new Error("username and email are required");
  // Quote email; pass --role=administrator. --porcelain prints only the new user ID.
  let args = `user create ${username} ${email} --role=administrator`;
  if (password) args += ` --user_pass='${password}'`;
  args += " --porcelain";
  return wpCli({ org, site, args }, userId);
}

// ---------------------------------------------------------------------------
// Facts
// ---------------------------------------------------------------------------

/** Known SSH/SFTP connection facts. The password is the Flywheel account password (not stored). */
export function getConnectionInfo() {
  return {
    sshGateway: SSH_GATEWAY,
    sshPerSiteFormat: `ssh <org>+<site>@${SSH_GATEWAY}`,
    sftpHost: SFTP_HOST,
    sftpPort: SFTP_PORT,
    sftpUsername: KNOWN.account.username,
    authNote:
      "SFTP/SSH password == your Flywheel account password (set at /profile/security). " +
      "Headless WP-CLI uses an SSH public key registered at /profile/ssh instead.",
  };
}

/** The locally-recorded site registry. */
export function listKnownSites() {
  return KNOWN;
}

// ---------------------------------------------------------------------------
// Credential check
// ---------------------------------------------------------------------------

export async function checkCredentials(userId) {
  try {
    const sites = Object.keys(KNOWN.sites).length;
    // Best-effort: is a key registered for the gateway? We can't know without a
    // site, so just confirm system ssh exists (the ssh weapon enforces the rest).
    const ssh = await executeCommand({ host: "<REDACTED-HOST>", user: "noop", command: "true", timeout: 6000 }, userId).catch(
      (e) => ({ ok: false, stderr: e.message }),
    );
    const keyHint = ssh?.ok
      ? "gateway reachable"
      : "no SSH key registered yet OR gateway unreachable (UI control still works via CIC)";
    return {
      ok: true,
      msg: `Flywheel: UI control via CIC (no API). ${sites} known site(s). SSH/WP-CLI: ${keyHint}.`,
    };
  } catch (e) {
    return { ok: false, msg: `Flywheel weapon error: ${e.message}` };
  }
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/ssh/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/weapon/ssh`

Rationale layer (per CLAUDE.md §6). Append; supersede in place when a decision
changes.

---

## 2026-06-09 — SSH hardcodes `BatchMode=yes` + `StrictHostKeyChecking=accept-new`

**Decision.** Every SSH invocation passes `-o BatchMode=yes` (never prompt for a
password) and `-o StrictHostKeyChecking=accept-new` (trust-on-first-use,
auto-populating known_hosts).

**Rejected alternative.** Allow interactive password prompts and/or strict host-key
verification (the ssh defaults).

**Why.** The weapon is agent-invoked automation. An interactive prompt hangs the
agent indefinitely; strict host-key checking blocks every first-time connection.
The operating assumption is that passwordless keys are pre-configured by the
operator, so BatchMode + accept-new is the correct automation semantic.

**Consequence if violated.** First connection to a new host hangs on the unknown-host
prompt, or any host needing a password never connects — both as silent agent stalls.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/ssh/index.js -->
```js
/**
 * SSH weapon — execute commands on remote machines.
 *
 * Uses system SSH (no npm deps). Requires passwordless SSH keys to be set up.
 * Known hosts are configured in profiles.env_vars as SSH_HOSTS JSON.
 */

export const toc = {
  executeCommand: "Execute a shell command on a remote host via SSH.",
  searchHosts: "Search configured SSH hosts.",
  readRemoteFile: "Read a file from a remote host over SSH.",
};
import { execFile } from "node:child_process";
import { promisify } from "node:util";
import { database } from "@/libs/council/database";

const execFileAsync = promisify(execFile);
const DEFAULT_TIMEOUT = 30_000;

// ---------------------------------------------------------------------------
// Host registry
// ---------------------------------------------------------------------------

/**
 * Known host presets. Additional hosts can be added via SSH_HOSTS env var.
 */
// Host/user for boster_production are redacted from source — set BOSTER_PROD_HOST /
// BOSTER_PROD_USER (and optionally BOSTER_PROD_PORT) in .env.local. The preset is
// omitted from the registry when its host env is unset; use SSH_HOSTS or a raw host.
const KNOWN_HOSTS = {
  carbon: { host: "carbon", user: null, description: "Carbon SSH server (local network)" },
  ...(process.env.BOSTER_PROD_HOST
    ? {
        boster_production: {
          host: process.env.BOSTER_PROD_HOST,
          user: process.env.BOSTER_PROD_USER || undefined,
          port: Number(process.env.BOSTER_PROD_PORT) || 2223,
          description: "Boster production server — Magento 2, MariaDB",
        },
      }
    : {}),
};

/**
 * Resolve a host alias or raw host string to connection details.
 * @param {string} hostOrAlias
 * @param {string} [userId]
 */
async function resolveHost(hostOrAlias, userId) {
  // Check presets
  const preset = KNOWN_HOSTS[hostOrAlias];
  if (preset) return preset;

  // Check user-defined hosts from env_vars
  if (userId) {
    try {
      const db = await database.init("service");
      const { data } = await db.from("profiles").select("env_vars").eq("id", userId).single();
      const hostsJson = data?.env_vars?.SSH_HOSTS;
      if (hostsJson) {
        const hosts = JSON.parse(hostsJson);
        if (hosts[hostOrAlias]) return hosts[hostOrAlias];
      }
    } catch { /* no custom hosts */ }
  }

  // Raw host string
  return { host: hostOrAlias, user: null, description: null };
}

// ---------------------------------------------------------------------------
// Execute
// ---------------------------------------------------------------------------

/**
 * Execute a command on a remote host via SSH.
 * @param {{ host: string, command: string, user?: string, timeout?: number }} input
 * @param {string} [userId]
 * @returns {Promise<{ ok: boolean, stdout: string, stderr: string, exitCode: number|null }>}
 */
export async function executeCommand({ host, command, user, timeout } = {}, userId) {
  if (!host) throw new Error("host is required");
  if (!command) throw new Error("command is required");

  const resolved = await resolveHost(host, userId);
  const sshUser = user || resolved.user;
  const target = sshUser ? `${sshUser}@${resolved.host}` : resolved.host;
  const timeoutMs = timeout || DEFAULT_TIMEOUT;

  const args = [
    "-o", "ConnectTimeout=10",
    "-o", "StrictHostKeyChecking=accept-new",
    "-o", "BatchMode=yes",
  ];
  if (resolved.port) {
    args.push("-p", String(resolved.port));
  }
  args.push(target, command);

  try {
    const { stdout, stderr } = await execFileAsync("ssh", args, {
      timeout: timeoutMs,
      maxBuffer: 5 * 1024 * 1024,
    });
    return { ok: true, stdout: stdout.slice(0, 10000), stderr: stderr.slice(0, 2000), exitCode: 0 };
  } catch (e) {
    return {
      ok: false,
      stdout: (e.stdout || "").slice(0, 10000),
      stderr: (e.stderr || e.message || "").slice(0, 2000),
      exitCode: e.code ?? null,
    };
  }
}

/**
 * Search available SSH hosts (presets + user-defined).
 * @param {object} [input]
 * @param {string} [userId]
 */
export async function searchHosts(input = {}, userId) {
  const hosts = { ...KNOWN_HOSTS };

  if (userId) {
    try {
      const db = await database.init("service");
      const { data } = await db.from("profiles").select("env_vars").eq("id", userId).single();
      const hostsJson = data?.env_vars?.SSH_HOSTS;
      if (hostsJson) {
        const custom = JSON.parse(hostsJson);
        Object.assign(hosts, custom);
      }
    } catch { /* no custom hosts */ }
  }

  return {
    hosts: Object.entries(hosts).map(([alias, info]) => ({
      alias,
      host: info.host,
      user: info.user,
      description: info.description,
    })),
  };
}

/**
 * Read a file from a remote host.
 * @param {{ host: string, path: string, user?: string, tail?: number }} input
 * @param {string} [userId]
 */
export async function readRemoteFile({ host, path, user, tail } = {}, userId) {
  if (!path) throw new Error("path is required");
  const cmd = tail ? `tail -n ${tail} ${path}` : `cat ${path}`;
  return executeCommand({ host, command: cmd, user }, userId);
}

// ---------------------------------------------------------------------------
// Credential check
// ---------------------------------------------------------------------------

export async function checkCredentials(userId) {
  try {
    // Check if ssh is available
    const { stdout } = await execFileAsync("ssh", ["-V"], { timeout: 5000 }).catch((e) => ({
      stdout: "",
      stderr: e.stderr || e.message,
    }));
    const hosts = await searchHosts({}, userId);
    return {
      ok: true,
      msg: `SSH available. ${hosts.hosts.length} known hosts configured.`,
    };
  } catch (e) {
    return { ok: false, msg: `SSH not available: ${e.message}` };
  }
}
```
<!-- /GUILDOS-FILE -->
