# raspberryPiLocal

<!-- GUILDOS-SKILL name="raspberryPiLocal" -->

<!-- GUILDOS-DESC -->
Access and admin procedures for the household Raspberry Pi over LAN and Tailscale. Actions: setupSshAccess (one-time SSH+key install), setupTailscale (join the tailnet for off-LAN access), setupRemoteDesktop (RealVNC over Tailscale), installClaudeCode (Node 18 + pure-JS Claude Code 1.0.x, the only path on 32-bit armhf), connect (`ssh pi`), connectRemoteDesktop (launch VNC viewer), formularyKeys (canonical credential key names).
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/raspberryPiLocal/SKILL.md -->
```markdown
---
name: raspberryPiLocal
description: >-
  Access and admin procedures for the household Raspberry Pi over LAN and
  Tailscale. Actions: setupSshAccess (one-time SSH+key install), setupTailscale
  (join the tailnet for off-LAN access), setupRemoteDesktop (RealVNC over
  Tailscale), installClaudeCode (Node 18 + pure-JS Claude Code 1.0.x, the only
  path on 32-bit armhf), connect (`ssh pi`), connectRemoteDesktop (launch VNC
  viewer), formularyKeys (canonical credential key names).
---

# Raspberry Pi (local LAN + Tailscale)

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/raspberryPiLocal/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto raspberryPiLocal <action>` (or read the source directly).

## Actions

- **connect** — Log in to the Pi from this Windows workstation via SSH — over LAN or Tailscale.
- **connectRemoteDesktop** — Launch RealVNC Viewer pointing at the Pi — defaults to Tailscale if available, falls back to LAN.
- **formularyKeys** — Canonical Formulary key names that future Pi-control weapons should read. The agent surfaces these to the user once with their (non-secret) values prefilled; secret values (PI_PASSWORD) the user types into the Council Hall → Formulary UI themselves — they never appear in chat or in any agent-side store.
- **installClaudeCode** — Install Claude Code on the Pi as the orchestrator runtime for always-on agent work (dailies, lifesci_intel, scheduled /loop tasks). Pinned to Node 18.20.8 armv6l + Claude Code 1.0.128 — the ONLY combination that runs on 32-bit Raspbian; Node 20+ and Claude Code 2.x both fail on this hardware. Output: `claude` on PATH at /usr/local/bin/claude.
- **setupRemoteDesktop** — Enable RealVNC graphical remote desktop on the Pi and connect from this Windows workstation. Use Tailscale (when up) for off-LAN access, or LAN IP for local. Note: Chrome Remote Desktop is NOT available — no armhf build; RealVNC is the equivalent on this hardware.
- **setupSshAccess** — One-time setup: enable SSH on the Pi, install this PC's public key as an authorized_key for the Pi user, and add a `pi` Host entry to ~/.ssh/config so `ssh pi` resolves without flags. End state: passwordless key-only login from this Windows box over LAN.
- **setupTailscale** — Install Tailscale on the Pi and join your tailnet so you can SSH / VNC / reach the Pi from anywhere — no port forwards, no DDNS, no LAN dependency. Adds Tailscale SSH and a stable MagicDNS hostname `raspberrypi.<tailnet>.ts.net`.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/raspberryPiLocal/index.js -->
```js
/**
 * Raspberry Pi (local LAN + Tailscale) skill book.
 *
 * Procedures for accessing the household Raspberry Pi via three layers:
 *   1. SSH over LAN (`ssh pi` shortcut, mDNS hostname `raspberrypi.local`)
 *   2. SSH + ICMP + anything else over the Tailscale tailnet (works off-LAN)
 *   3. RealVNC graphical remote desktop (via either LAN or Tailscale)
 *
 * Current device:
 *   - Pi 4, 32-bit Raspbian Bullseye (armv7l / armhf)
 *   - LXDE desktop running under lightdm, RealVNC server on :5900
 *   - Tailscale 1.98.3 installed, hostname advertised as `raspberrypi`
 *   - Local LAN IP: <REDACTED-IP> (DHCP — reserve in router for stability)
 *   - User account: `winston` (sudo group, key-authed from this workstation)
 *
 * Why NOT Chrome Remote Desktop: no armhf build of the host package exists,
 * and Google Chrome itself has no ARM Linux build. RealVNC over Tailscale is
 * the equivalent on this hardware; no OS reinstall required.
 *
 * Formulary keys consumed by future Pi-control weapons (see `formularyKeys`
 * action). Secret values (PI_PASSWORD) are typed by the user into the
 * Council Hall → Formulary UI; the agent never sees them.
 */

export const skillBook = {
  id: "raspberryPiLocal",
  title: "Raspberry Pi (local LAN + Tailscale)",
  description:
    "Access and admin procedures for the household Raspberry Pi over LAN and Tailscale. Actions: setupSshAccess (one-time SSH+key install), setupTailscale (join the tailnet for off-LAN access), setupRemoteDesktop (RealVNC over Tailscale), installClaudeCode (Node 18 + pure-JS Claude Code 1.0.x, the only path on 32-bit armhf), connect (`ssh pi`), connectRemoteDesktop (launch VNC viewer), formularyKeys (canonical credential key names).",
  steps: [],
  toc: {
    setupSshAccess: {
      description:
        "One-time setup: enable SSH on the Pi, install this PC's public key as an authorized_key for the Pi user, and add a `pi` Host entry to ~/.ssh/config so `ssh pi` resolves without flags. End state: passwordless key-only login from this Windows box over LAN.",
      howTo: `
End state: \`ssh pi\` from this Windows PC logs in to the Pi with no password, using \`$env:USERPROFILE\\.ssh\\id_ed25519\`.

## 1. Enable SSH on the Pi (one of)

- **Pi has keyboard+monitor:** \`sudo systemctl enable --now ssh\` (or \`sudo raspi-config\` → Interface Options → SSH → Enable).
- **Headless, SD card on this PC:** create an empty file named \`ssh\` (no extension) on the boot partition. SSH auto-enables on first boot.
- **Raspberry Pi Imager (recommended for fresh installs):** use the gear-icon advanced settings to preset hostname, username, password, Wi-Fi, **and** paste the contents of \`$env:USERPROFILE\\.ssh\\id_ed25519.pub\` as an authorized SSH public key. If this is done, steps 3 + 4 below are already finished.

## 2. Resolve the Pi from PowerShell

\`\`\`powershell
ping -n 2 raspberrypi.local
\`\`\`

Any reply (IPv4 or IPv6) = mDNS is working. If it fails: check the router DHCP client list for the Pi's IP and use that instead of \`raspberrypi.local\`. Reserve the IP in DHCP so the name stays stable.

## 3. Ensure a local SSH keypair exists

\`\`\`powershell
if (-not (Test-Path $env:USERPROFILE\\.ssh\\id_ed25519.pub)) { ssh-keygen -t ed25519 -f $env:USERPROFILE\\.ssh\\id_ed25519 -N '""' }
Get-Content $env:USERPROFILE\\.ssh\\id_ed25519.pub
\`\`\`

## 4. Install the public key on the Pi (one-time, requires password)

The agent CANNOT drive the interactive Pi password prompt — the user runs this one-liner in their own PowerShell:

\`\`\`powershell
type $env:USERPROFILE\\.ssh\\id_ed25519.pub | ssh <user>@raspberrypi.local "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
\`\`\`

If no password is set on the Pi yet, set one first on the Pi console: \`sudo passwd <user>\`. There is no remote path around this — SSH requires a password or a pre-installed key to authenticate. Never ask the user to paste the Pi password into chat (it lands in the transcript); have them run the one-liner themselves.

## 5. Verify key auth works (agent, non-interactive)

\`\`\`powershell
ssh -o BatchMode=yes -o ConnectTimeout=5 <user>@raspberrypi.local "echo CONNECTED; hostname; uname -srm"
\`\`\`

\`BatchMode=yes\` disables password prompts, so success proves key auth — not a silent fallback to password.

## 6. Add the \`pi\` shortcut to ~/.ssh/config

\`\`\`powershell
$cfg = "$env:USERPROFILE\\.ssh\\config"
if (-not (Test-Path $cfg)) { New-Item -ItemType File $cfg | Out-Null }
if (-not (Select-String -Path $cfg -Pattern '^Host pi$' -Quiet)) {
  Add-Content -Path $cfg -Value "\`nHost pi\`n    HostName raspberrypi.local\`n    User <user>\`n    IdentityFile ~/.ssh/id_ed25519" -Encoding utf8
}
ssh -o BatchMode=yes pi "echo OK from \\\`$(hostname)"
\`\`\`

## 7. Harden (recommended, after key auth is confirmed)

On the Pi:

\`\`\`bash
sudo sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
\`\`\`

Do NOT do this before step 5 succeeds — disabling password auth without a working key locks you out remotely (recovery then requires keyboard+monitor or SD-card surgery).

## Notes

- mDNS (\`*.local\`) needs Bonjour on Windows. Built into Windows 10/11; if it fails, iTunes/Bonjour Print Services installs it.
- IPv6 reply is fine — OpenSSH on Windows follows whichever AAAA/A record resolves.
- If the Pi user is not the same as the Windows user, the \`User\` line in ~/.ssh/config is what makes \`ssh pi\` work without \`<user>@\`.
`,
    },

    setupTailscale: {
      description:
        "Install Tailscale on the Pi and join your tailnet so you can SSH / VNC / reach the Pi from anywhere — no port forwards, no DDNS, no LAN dependency. Adds Tailscale SSH and a stable MagicDNS hostname `raspberrypi.<tailnet>.ts.net`.",
      howTo: `
End state: from any Tailscale-connected device, \`ssh winston@raspberrypi.<tailnet>.ts.net\` (or by tailnet IP) reaches the Pi. The Pi appears in your Tailscale admin console as node "raspberrypi".

## Prerequisite

SSH access to the Pi already works (see \`setupSshAccess\`). The user has a Tailscale account; if not, they create one at https://login.tailscale.com (free Personal plan covers 100 devices).

## 1. Install Tailscale on the Pi (agent, via SSH)

\`\`\`powershell
ssh -o BatchMode=yes pi "curl -fsSL https://tailscale.com/install.sh | sudo sh"
\`\`\`

The install script adds Tailscale's apt repo, installs the \`tailscale\` package, and enables the \`tailscaled\` systemd service. Works on armhf (the Pi's 32-bit userspace) — the package set is \`tailscale_<version>_armhf.deb\`.

## 2. Start \`tailscale up\` and capture the auth URL

\`tailscale up\` blocks waiting for browser auth and prints the URL to stderr. Run it backgrounded with output to a file, then read the file:

\`\`\`powershell
ssh -o BatchMode=yes pi "sudo pkill -f 'tailscale up' 2>/dev/null; sleep 1; sudo sh -c 'nohup tailscale up --ssh --hostname=raspberrypi >/tmp/tsup.log 2>&1 &'; sleep 6; sudo cat /tmp/tsup.log"
\`\`\`

Flags worth knowing:
- \`--ssh\` — enables **Tailscale SSH**: the Pi authenticates incoming SSH using tailnet identity instead of \`authorized_keys\`. The standard OpenSSH path keeps working too. Major win — no per-device key rotation.
- \`--hostname=raspberrypi\` — locks the tailnet node name. Without this Tailscale picks a name from the OS hostname, which can drift.
- (Optional later) \`--advertise-exit-node\`, \`--advertise-routes=<lan-cidr>\` to use the Pi as an exit node or a subnet router. Don't enable on first run.

## 3. Surface the auth URL to the user

The log file will contain a line like:

\`\`\`
To authenticate, visit:

    https://login.tailscale.com/a/XXXXXXXXXXXX
\`\`\`

**The agent cannot click this for the user.** Surface it verbatim and wait. The URL expires (~10 min) — if stale, re-run step 2 to issue a new one. The first device on a tailnet may also prompt for tailnet creation in the same flow.

## 4. Verify the Pi joined the tailnet

After the user clicks "Connect" in the browser:

\`\`\`powershell
ssh -o BatchMode=yes pi "sudo tailscale status | head -5; echo ---; sudo tailscale ip -4"
\`\`\`

Expect a 100.x.y.z IPv4 (and an fd7a:115c::… IPv6). The MagicDNS name is \`raspberrypi.<your-tailnet>.ts.net\` — visible in the Tailscale admin at https://login.tailscale.com/admin/machines.

## 5. Add a tailnet-routed SSH alias

\`\`\`powershell
$cfg = "$env:USERPROFILE\\.ssh\\config"
Add-Content -Path $cfg -Value "\`nHost pi-ts\`n    HostName raspberrypi.<your-tailnet>.ts.net\`n    User winston\`n    IdentityFile ~/.ssh/id_ed25519" -Encoding utf8
ssh -o BatchMode=yes pi-ts "echo OK from \\\`$(hostname); curl -s ifconfig.me"
\`\`\`

\`pi\` keeps targeting the LAN; \`pi-ts\` targets the tailnet (works off-LAN, e.g. from a coffee shop).

## Notes & gotchas

- **MagicDNS suffix.** Personal tailnets get a name like \`tail<abcd>.ts.net\`. Find yours in the admin console DNS panel — it's the same suffix on every node.
- **Tailscale SSH vs OpenSSH.** With \`--ssh\`, Tailscale runs its own SSH server on port 22 alongside (NOT instead of) sshd. ACLs in the admin console control who from the tailnet can SSH in; the local user must still exist.
- **No port forwards required.** Tailscale punches NAT via DERP relays — works behind double-NAT, CGNAT, Starlink, etc.
- **Daemon survives reboot** (systemd unit \`tailscaled.service\` enabled by the installer). \`tailscale up\` is one-shot; re-running with the same hostname is idempotent.
- **Removing the Pi from the tailnet** is done in the admin console (Machines → … → Remove). On the Pi: \`sudo tailscale logout\` then \`sudo tailscale down\`.
`,
    },

    setupRemoteDesktop: {
      description:
        "Enable RealVNC graphical remote desktop on the Pi and connect from this Windows workstation. Use Tailscale (when up) for off-LAN access, or LAN IP for local. Note: Chrome Remote Desktop is NOT available — no armhf build; RealVNC is the equivalent on this hardware.",
      howTo: `
End state: RealVNC Viewer on Windows opens the Pi's LXDE desktop. Connection target = LAN IP or Tailscale MagicDNS name on port 5900. Login = the Pi system user (winston) + Pi password.

## Why NOT Chrome Remote Desktop

The \`chrome-remote-desktop\` host package is published only for \`amd64\` and \`arm64\` — there is no \`armhf\` build, and 32-bit Raspbian Bullseye runs an armhf userspace. Google Chrome itself also has no Linux/ARM build at all (only Chromium). On 64-bit Pi OS (arm64 / Bookworm) CRD does work; the only path to CRD on this device is reflashing to 64-bit Pi OS. RealVNC over Tailscale gives the same outcome (graphical remote control, encrypted, NAT-traversal) without reinstalling.

## Prerequisites

- Desktop environment running on the Pi (LXDE under lightdm — preinstalled on Pi OS Desktop). Verify: \`systemctl is-active lightdm\` returns \`active\`.
- \`realvnc-vnc-server\` package installed (preinstalled on Pi OS Desktop). Verify: \`dpkg -l realvnc-vnc-server\`.
- Pi password set for the login user — RealVNC's default auth scheme delegates to PAM.

## 1. Enable the RealVNC service on the Pi (agent, via SSH)

\`\`\`powershell
ssh -o BatchMode=yes pi "sudo raspi-config nonint do_vnc 0; systemctl is-enabled vncserver-x11-serviced; systemctl is-active vncserver-x11-serviced; sudo ss -tlnp | grep ':5900'"
\`\`\`

\`do_vnc 0\` = enable (raspi-config inverts boolean: 0=on, 1=off). Expect:
- \`enabled\` + \`active\`
- A listener on \`<REDACTED-IP>:5900\` owned by \`vncserver-x11-c\`.

## 2. Install RealVNC Viewer on Windows

\`\`\`powershell
winget install --id RealVNC.VNCViewer --silent --accept-package-agreements --accept-source-agreements
\`\`\`

Installs to \`C:\\Program Files\\RealVNC\\VNC Viewer\\vncviewer.exe\`. (Open-source alternatives like TightVNC also work — the Pi's RealVNC server speaks the RFB protocol.)

## 3. Launch a session

\`\`\`powershell
$vnc = "C:\\Program Files\\RealVNC\\VNC Viewer\\vncviewer.exe"
# Over LAN (use the Pi's reserved DHCP IP)
Start-Process $vnc -ArgumentList "<REDACTED-IP>:5900"
# Over Tailscale (works off-LAN, once setupTailscale is done)
Start-Process $vnc -ArgumentList "raspberrypi.<tailnet>.ts.net:5900"
\`\`\`

A window opens. First connect prompts an identity-check warning — accept it (and optionally save the host fingerprint). Log in with \`winston\` + the Pi password.

## 4. Security posture

- **LAN target (<REDACTED-IP>):** anyone on the LAN can reach :5900 and try to log in. Acceptable for trusted home Wi-Fi; for guest networks tighten with a firewall rule (\`sudo ufw allow from <REDACTED-IP>/24 to any port 5900\`) or rely on Tailscale-only access (next bullet).
- **Tailscale target:** preferred. Don't expose :5900 publicly. Optionally tighten further with Tailscale ACLs so only your own devices can hit port 5900.
- Default RealVNC encryption is "Prefer on" — sessions are TLS-equivalent encrypted in transit. Don't disable.

## 5. Optional: change VNC port or resolution

- **Port:** edit \`/etc/vnc/config.d/common.custom\` (\`Port=5901\`) and \`sudo systemctl restart vncserver-x11-serviced\`.
- **Headless resolution** (no monitor attached, default 720p is small): \`sudo raspi-config\` → Display Options → VNC Resolution. Or edit \`/boot/config.txt\`: uncomment \`hdmi_force_hotplug=1\` and set \`framebuffer_width\` / \`framebuffer_height\`.

## Troubleshooting

- **"Cannot currently show the desktop"** in the viewer = no X session is running. Pi OS Lite has none; switch to Pi OS Desktop or install \`raspberrypi-ui-mods lightdm\`.
- **Black/blank screen but connected** = display unplugged + framebuffer not forced. See "headless resolution" above.
- **Authentication failed (RealVNC sign-in dialog)** = Pi user has no password or PAM is misconfigured. Set with \`sudo passwd winston\` on the Pi.
`,
    },

    installClaudeCode: {
      description:
        "Install Claude Code on the Pi as the orchestrator runtime for always-on agent work (dailies, lifesci_intel, scheduled /loop tasks). Pinned to Node 18.20.8 armv6l + Claude Code 1.0.128 — the ONLY combination that runs on 32-bit Raspbian; Node 20+ and Claude Code 2.x both fail on this hardware. Output: `claude` on PATH at /usr/local/bin/claude.",
      howTo: `
End state: \`ssh pi\` then \`claude --version\` returns \`1.0.128 (Claude Code)\`. Authenticated via \`ANTHROPIC_API_KEY\` env var so non-interactive systemd-timed runs work.

## Why this exact pin (the hard-learned facts)

The 32-bit armhf userspace forces three non-obvious choices:

- **Node 20+ fails on Bullseye.** Official armv7l builds were dropped at Node 18; unofficial-builds publishes Node 20 armv6l, but those are compiled against GLIBCXX_3.4.29/30 (gcc 11/12) and Bullseye ships libstdc++ at GLIBCXX_3.4.28 (gcc 10). \`node -v\` errors with "version \`GLIBCXX_3.4.30' not found". Upgrading libstdc++ via backports is a rabbit hole — just use Node 18.
- **Node from \`apt\` is too old.** Raspbian Bullseye ships Node 12.22 — below Claude Code's Node 18 minimum.
- **NodeSource dropped armhf.** \`deb.nodesource.com/setup_20.x\` errors with "Unsupported architecture: armhf. Only amd64, arm64 are supported." Useless on this Pi.
- **unofficial-builds publishes armv6l, not armv7l, for Node 18+.** That's fine: armv7-A chips (Pi 2/3/4 in 32-bit mode) execute armv6 binaries natively — small perf trade vs. NEON-optimized armv7 but a non-issue for an orchestrator that's mostly waiting on API calls.
- **Claude Code 2.x is native-binary only.** Starting at v2.0.0, \`@anthropic-ai/claude-code\` ships a Rust(?)-compiled binary loaded via platform-specific optional npm deps. Supported: \`darwin-{arm64,x64}\`, \`linux-{x64,arm64}\`, \`win32-{x64,arm64}\`, \`freebsd-{x64,arm64}\`. NO linux-arm (armhf) build. Postinstall errors \`Unsupported platform: linux arm\`. Pinning to \`@1.0.128\` (last 1.x release, pure-JS) is the workaround until either Anthropic ships an armhf native binary or the Pi is reflashed to 64-bit Pi OS.

## Procedure

\`\`\`powershell
# 1. Install Node 18.20.8 armv6l from unofficial-builds.
ssh -o BatchMode=yes pi "set -e; sudo rm -f /usr/local/bin/{node,npm,npx,corepack}; cd /tmp; V=v18.20.8; curl -fsSLO https://unofficial-builds.nodejs.org/download/release/\\\${V}/node-\\\${V}-linux-armv6l.tar.xz; sudo tar -xJf node-\\\${V}-linux-armv6l.tar.xz -C /usr/local --strip-components=1 --exclude CHANGELOG.md --exclude LICENSE --exclude README.md; rm node-\\\${V}-linux-armv6l.tar.xz; /usr/local/bin/node -v; /usr/local/bin/npm -v"

# 2. Install pure-JS Claude Code (pin to 1.0.128, NOT @latest).
ssh -o BatchMode=yes pi "sudo /usr/local/bin/npm install -g @anthropic-ai/claude-code@1.0.128 2>&1 | tail -5; /usr/local/bin/claude --version"
\`\`\`

Expect: \`1.0.128 (Claude Code)\`. If you see \`Error: claude native binary not installed\`, you accidentally installed @latest — uninstall and reinstall at 1.0.128.

## Authentication

For interactive sessions (SSH into Pi, run \`claude\`), the CLI opens a browser-based login flow on first run.

For non-interactive use (systemd timers, scheduled jobs), set \`ANTHROPIC_API_KEY\`:

\`\`\`bash
# On the Pi, as winston:
mkdir -p ~/.config/anthropic
chmod 700 ~/.config/anthropic
echo 'export ANTHROPIC_API_KEY=sk-ant-...' > ~/.config/anthropic/env
chmod 600 ~/.config/anthropic/env
# Source it in scheduled jobs:
echo '[ -r ~/.config/anthropic/env ] && . ~/.config/anthropic/env' >> ~/.bashrc
\`\`\`

For systemd units running as \`winston\`, add \`EnvironmentFile=%h/.config/anthropic/env\` to the unit file.

Do NOT put \`ANTHROPIC_API_KEY\` in the GuildOS Formulary — the Formulary is a per-user GuildOS-app store, not the Pi's environment. The Pi's Claude Code reads the OS env, not Supabase.

## Smoke test

\`\`\`powershell
ssh -o BatchMode=yes pi 'export ANTHROPIC_API_KEY=$(cat ~/.config/anthropic/env | sed "s/.*=//"); echo "what year is it?" | /usr/local/bin/claude -p --output-format text'
\`\`\`

Expect a short text response. If you get an auth error, the key path is wrong. If you get a model/tool error, the install is fine but the request is malformed — try \`claude -p "hello"\` instead.

## Upgrade path to v2.x (later)

When Anthropic ships an armhf native binary OR when the Pi is reflashed to 64-bit Pi OS (Bookworm arm64), drop the version pin:

\`\`\`powershell
ssh -o BatchMode=yes pi "sudo /usr/local/bin/npm install -g @anthropic-ai/claude-code@latest && claude --version"
\`\`\`

For 64-bit Pi OS, Node 20+ also becomes available via NodeSource (\`arm64\` is supported) — that's a cleaner stack overall, at the cost of a one-time SD-card reflash.

## Sister install: Codex CLI

\`@openai/codex\` follows the same pattern as Claude Code 2.x — native binary with no armhf build. Same workaround applies: pin to the last pure-JS release if one exists, or wait for OS reflash. Install command parallel to Claude Code; verify with \`codex --version\`.
`,
    },

    connect: {
      description: "Log in to the Pi from this Windows workstation via SSH — over LAN or Tailscale.",
      howTo: `
\`\`\`powershell
ssh pi        # LAN — uses raspberrypi.local + IdentityFile from ~/.ssh/config
ssh pi-ts     # Tailscale — works off-LAN once setupTailscale is done
\`\`\`

One-shot remote command (capturable for the agent, never falls back to password):

\`\`\`powershell
ssh -o BatchMode=yes pi "<command>"
\`\`\`

Failure mode lookup:
- \`Could not resolve hostname raspberrypi.local\` — Pi offline, on a different network, or its mDNS responder (avahi-daemon) stopped. Use \`pi-ts\` instead, or fall back to the DHCP-reserved IP (\`<REDACTED-IP>\`).
- \`Permission denied (publickey)\` — authorized_keys entry was removed/rotated. Re-run \`setupSshAccess\` step 4.
- \`pi-ts\`: \`Connection timed out\` — the Pi is offline OR Tailscale on the Pi is logged out. SSH to it on the LAN and \`sudo tailscale status\`.
`,
    },

    connectRemoteDesktop: {
      description: "Launch RealVNC Viewer pointing at the Pi — defaults to Tailscale if available, falls back to LAN.",
      howTo: `
\`\`\`powershell
$vnc = "C:\\Program Files\\RealVNC\\VNC Viewer\\vncviewer.exe"
# Choose ONE target:
Start-Process $vnc -ArgumentList "<REDACTED-IP>:5900"                          # LAN
Start-Process $vnc -ArgumentList "raspberrypi.<tailnet>.ts.net:5900"         # Tailscale (off-LAN)
\`\`\`

A viewer window opens; log in with \`winston\` + Pi password. The Tailscale form works from anywhere; the LAN form only from home Wi-Fi.

If VNC Viewer isn't installed: \`winget install --id RealVNC.VNCViewer --silent --accept-package-agreements --accept-source-agreements\`.
`,
    },

    formularyKeys: {
      description:
        "Canonical Formulary key names that future Pi-control weapons should read. The agent surfaces these to the user once with their (non-secret) values prefilled; secret values (PI_PASSWORD) the user types into the Council Hall → Formulary UI themselves — they never appear in chat or in any agent-side store.",
      howTo: `
The Formulary (Council Hall → Formulary, \`/council-hall/formulary\`) stores per-user named env-vars. Values are write-only — never displayed back. Future Pi-control weapons read these via the standard env-var resolution path (profile values override \`process.env\`).

## Canonical key schema

| Key | Example value | Secret? | Source |
|---|---|---|---|
| \`PI_HOST\` | \`raspberrypi.local\` | no | mDNS hostname on LAN |
| \`PI_LAN_IP\` | \`<REDACTED-IP>\` | no | DHCP-assigned (reserve in router) |
| \`PI_USER\` | \`winston\` | no | Pi system user |
| \`PI_PASSWORD\` | (user-typed) | **YES** | Pi system user's password — for VNC + sudo |
| \`PI_TS_HOST\` | \`raspberrypi.<tailnet>.ts.net\` | no | MagicDNS — fill after \`setupTailscale\` |
| \`PI_TS_IP\` | \`100.x.y.z\` | no | Tailscale IPv4 — fill after \`setupTailscale\` |
| \`PI_VNC_PORT\` | \`5900\` | no | RealVNC default; change only if customized |

The SSH private key path is NOT a Formulary entry — it lives on disk at \`$env:USERPROFILE\\.ssh\\id_ed25519\` and is referenced by \`~/.ssh/config\`. Saving a path here would just duplicate state.

## How the agent should save them

Do **not** ask the user to paste \`PI_PASSWORD\` into chat — that puts it in the session transcript.

Two acceptable paths:

**A. User does it (preferred for the password):** open https://localhost:3002/council-hall/formulary in the user's browser. They click "Brew a new seal" for each key, type the value, and submit. Agent can pre-populate the non-secret keys via path B and leave \`PI_PASSWORD\` for them.

**B. Agent drives the API (non-secrets):** the user is signed in to GuildOS in their browser. Have them paste this into the browser DevTools console (so the session cookie is attached) — it sets all non-secret keys in one PATCH:

\`\`\`javascript
fetch('/api/council/profile/env-vars', {
  method: 'PATCH',
  credentials: 'include',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    set: {
      PI_HOST: 'raspberrypi.local',
      PI_LAN_IP: '<REDACTED-IP>',
      PI_USER: 'winston',
      PI_VNC_PORT: '5900',
      // Add after setupTailscale completes:
      // PI_TS_HOST: 'raspberrypi.<tailnet>.ts.net',
      // PI_TS_IP:   '100.x.y.z',
    }
  })
}).then(r => r.json()).then(console.log)
\`\`\`

Then the user adds \`PI_PASSWORD\` via the UI ("Brew a new seal" → name \`PI_PASSWORD\` → essence = the password → Commit to shelf).

## Verification

After the user runs the snippet + brews PI_PASSWORD, GET the keys list (no values returned — that's the point):

\`\`\`javascript
fetch('/api/council/profile/env-vars', { credentials: 'include' }).then(r => r.json()).then(console.log)
// → { keys: ["PI_HOST","PI_LAN_IP","PI_PASSWORD","PI_USER","PI_VNC_PORT", ...] }
\`\`\`
`,
    },
  },
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->
