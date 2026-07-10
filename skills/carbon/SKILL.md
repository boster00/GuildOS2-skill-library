# carbon

<!-- GUILDOS-SKILL name="carbon" -->

<!-- GUILDOS-DESC -->
Interactive + graphical control of the user's other Windows PC 'Carbon' (Tailscale hostname <REDACTED-HOST>). Tailscale is the DEFAULT transport for every action (works on- and off-LAN); LAN <REDACTED-IP> is fallback only. Actions: connectRemoteDesktop (DEFAULT = RDP over Tailscale; VNC-over-LAN is the no-MS-account alternative), connect (SSH), screenshot (capture Carbon's screen over SSH without an RDP/VNC session), enableVncOverTailscale (open the no-password-MS-account VNC path off-LAN), reachabilityCheck (TCP-probe ports — ICMP is firewalled), facts (canonical hosts/IPs/ports).
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/carbon/SKILL.md -->
```markdown
---
name: carbon
description: >-
  Interactive + graphical control of the user's other Windows PC 'Carbon'
  (Tailscale hostname <REDACTED-HOST>). Tailscale is the DEFAULT transport for every
  action (works on- and off-LAN); LAN <REDACTED-IP> is fallback only. Actions:
  connectRemoteDesktop (DEFAULT = RDP over Tailscale; VNC-over-LAN is the
  no-MS-account alternative), connect (SSH), screenshot (capture Carbon's screen
  over SSH without an RDP/VNC session), enableVncOverTailscale (open the
  no-password-MS-account VNC path off-LAN), reachabilityCheck (TCP-probe ports —
  ICMP is firewalled), facts (canonical hosts/IPs/ports).
---

# Carbon (remote Windows workstation)

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/carbon/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto carbon <action>` (or read the source directly).

## Actions

- **connect** — SSH into Carbon. DEFAULT = Tailscale (works off-LAN); `ssh carbon` LAN alias is the fallback. Non-interactive one-shot commands use BatchMode so they never silently fall back to a password.
- **connectRemoteDesktop** — Open a graphical remote-control session to Carbon. DEFAULT = RDP over Tailscale (mstsc /v:<REDACTED-IP>) — works on- and off-LAN. The no-MS-account alternative is VNC over LAN (RealVNC Viewer → <REDACTED-IP>:5900). Picks Tailscale unless explicitly told LAN.
- **enableVncOverTailscale** — Make the no-MS-account VNC path work OFF-LAN. Carbon answers VNC :5900 on the LAN NIC (<REDACTED-IP>) but NOT on the tailnet (<REDACTED-IP>) — almost certainly a Windows Firewall scope. Broaden the :5900 inbound rule to the Tailscale interface so RealVNC works from anywhere.
- **facts** — Canonical hosts, IPs, ports, and credential model for Carbon. Single source of truth — read before hand-typing any address.
- **reachabilityCheck** — Decide whether Carbon is up and which transport to use. ICMP ping is firewalled on Carbon — a failed ping is NOT a down signal. Probe real TCP ports (Tailscale first), and confirm <REDACTED-HOST> is on the tailnet.
- **screenshot** — Capture Carbon's current screen over SSH (no RDP/VNC viewer needed) and pull it back to this workstation. Uses Carbon's take-screenshot.ps1 scheduled task, which must run in the interactive session (/it).
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/carbon/index.js -->
```js
/**
 * Carbon (remote Windows workstation) skill book.
 *
 * "Carbon" is the user's other Windows PC — Tailscale hostname `<REDACTED-HOST>`,
 * the same box the bosterbio skill book uses as a whitelisted ProxyJump hop.
 * This book owns INTERACTIVE / GRAPHICAL control of Carbon (remote desktop +
 * SSH + screen capture), not the bosterbio SSH-tunnel use.
 *
 * Access layers, in priority order:
 *   1. Tailscale (<REDACTED-IP> / `<REDACTED-HOST>`) — DEFAULT. Works on-LAN AND
 *      off-LAN, no port forwards, encrypted (WireGuard). This is the canonical
 *      target for every action below.
 *   2. LAN (<REDACTED-IP>) — fallback only, when on the same Wi-Fi as Carbon.
 *
 * Why Tailscale is the default (not LAN):
 *   - It works from anywhere with internet; LAN only works at home.
 *   - It is the same robustness call the bosterbio skill book already made when
 *     it DEPRECATED its LAN hop in favour of the Tailscale ProxyJump.
 *   - LAN is a strict subset of when Tailscale works, so defaulting to LAN buys
 *     nothing and silently breaks the moment you're off the home network.
 *   The ONLY thing LAN currently gives you that Tailscale doesn't is VNC:5900
 *   (see `enableVncOverTailscale` — Carbon answers VNC on the LAN NIC but not on
 *   the tailnet yet). Fix that and there is no remaining reason to touch LAN.
 *
 * Current device facts (verified 2026-06-08):
 *   - Windows 11 workstation, user `xsj70`
 *   - Tailscale hostname `<REDACTED-HOST>`, IPv4 <REDACTED-IP> (stable, on the JetRails
 *     whitelist — see bosterbio skill book)
 *   - LAN IPv4 <REDACTED-IP> (DHCP — reserve in router for stability)
 *   - OpenSSH server on :22 (this workstation's pubkey is in authorized_keys)
 *   - RDP (mstsc) on :3389 — reachable over BOTH Tailscale and LAN
 *   - RealVNC server on :5900 — reachable over LAN ONLY (see enableVncOverTailscale)
 *   - ICMP ping is firewalled on both interfaces — a failed ping does NOT mean
 *     Carbon is down. Probe a real TCP port instead (Test-NetConnection ... -Port).
 */

export const skillBook = {
  id: "carbon",
  title: "Carbon (remote Windows workstation)",
  description:
    "Interactive + graphical control of the user's other Windows PC 'Carbon' (Tailscale hostname <REDACTED-HOST>). Tailscale is the DEFAULT transport for every action (works on- and off-LAN); LAN <REDACTED-IP> is fallback only. Actions: connectRemoteDesktop (DEFAULT = RDP over Tailscale; VNC-over-LAN is the no-MS-account alternative), connect (SSH), screenshot (capture Carbon's screen over SSH without an RDP/VNC session), enableVncOverTailscale (open the no-password-MS-account VNC path off-LAN), reachabilityCheck (TCP-probe ports — ICMP is firewalled), facts (canonical hosts/IPs/ports).",
  steps: [],
  toc: {
    connectRemoteDesktop: {
      description:
        "Open a graphical remote-control session to Carbon. DEFAULT = RDP over Tailscale (mstsc /v:<REDACTED-IP>) — works on- and off-LAN. The no-MS-account alternative is VNC over LAN (RealVNC Viewer → <REDACTED-IP>:5900). Picks Tailscale unless explicitly told LAN.",
      howTo: `
End state: a remote-desktop window onto Carbon is open on this workstation. Default target is **Tailscale**, so this works whether or not you're on Carbon's LAN.

## Decision: which of the two paths

| Path | Command target | Auth needed | Works off-LAN? |
|---|---|---|---|
| **RDP over Tailscale** (DEFAULT) | \`mstsc /v:<REDACTED-IP>\` | Carbon's **Microsoft-account** email + password | **Yes** |
| **VNC over LAN** (no MS account) | RealVNC Viewer → \`<REDACTED-IP>:5900\` | Standalone **VNC password** | No (LAN only until \`enableVncOverTailscale\`) |

Default to **RDP over Tailscale**. Reach for VNC only when the user wants to avoid the Microsoft-account login AND is on the home LAN — or after \`enableVncOverTailscale\` has been run.

## Why RDP asks for a Microsoft account (so you can explain it)

RDP logs you into an actual Windows user session ON Carbon, so it authenticates against Carbon's Windows login — which is a Microsoft account, not a local account. Use the **full MS-account email** as the username (NOT \`xsj70\`) with that account's password. This is inherent to RDP; there's no way around it short of using VNC (which has its own separate password).

## 1. Launch RDP over Tailscale (DEFAULT)

\`\`\`powershell
Start-Process mstsc -ArgumentList '/v:<REDACTED-IP>'
Start-Sleep 2
Get-Process mstsc -ErrorAction SilentlyContinue | Select-Object Id, MainWindowTitle | Format-Table -AutoSize
\`\`\`

A "Remote Desktop Connection" window opens, then a Windows Security credential prompt. **The agent must NOT type the password** (entering passwords into login fields is a prohibited action) — surface that the window is up and have the user enter the MS-account email + password. A blank \`MainWindowTitle\` after launch = it's sitting on that credential prompt (expected), not an error.

LAN fallback (only if explicitly on Carbon's Wi-Fi and Tailscale is down): \`Start-Process mstsc -ArgumentList '/v:<REDACTED-IP>'\`.

## 2. Alternative — VNC over LAN (no Microsoft account)

\`\`\`powershell
$vnc = "C:\\Program Files\\RealVNC\\VNC Viewer\\vncviewer.exe"
Start-Process $vnc -ArgumentList '<REDACTED-IP>'   # add ':5900' if a non-default port
\`\`\`

Window title becomes \`<REDACTED-IP> - RealVNC Viewer\`; it prompts for the **VNC password** (separate from the MS account — that's the whole point of this path). If \`vncviewer.exe\` isn't found: \`winget install --id RealVNC.VNCViewer --silent --accept-package-agreements --accept-source-agreements\`. To make this path work OFF-LAN over Tailscale, run \`enableVncOverTailscale\` first.

## 3. Verify (don't claim "connected" blind)

- mstsc/vncviewer launching ≠ a live session — both still need the user's password.
- To prove the window is up: \`Get-Process mstsc\` / \`Get-Process vncviewer\` (PID + title).
- To prove a session actually ESTABLISHED (after the user authenticates): \`Get-NetTCPConnection -RemoteAddress <REDACTED-IP> -RemotePort 3389\` should show \`Established\`.
- For a visual confirm of Carbon's desktop, use the \`screenshot\` action (SSH capture) — it doesn't depend on the RDP/VNC viewer being focused.
`,
    },

    connect: {
      description:
        "SSH into Carbon. DEFAULT = Tailscale (works off-LAN); `ssh carbon` LAN alias is the fallback. Non-interactive one-shot commands use BatchMode so they never silently fall back to a password.",
      howTo: `
\`\`\`powershell
# DEFAULT — Tailscale (works from anywhere), local key auth:
ssh -o BatchMode=yes xsj70@<REDACTED-IP> "hostname; whoami"

# LAN fallback alias (only on Carbon's Wi-Fi):
ssh -o BatchMode=yes carbon "hostname"
\`\`\`

Both authenticate with this workstation's \`~/.ssh/id_ed25519\` (already in Carbon's authorized_keys). \`BatchMode=yes\` disables the password prompt, so a success genuinely proves key auth.

Failure-mode lookup:
- \`Connection timed out\` to <REDACTED-IP> — Tailscale is down on this client or on <REDACTED-HOST>. Open the Tailscale tray / \`tailscale status | findstr <REDACTED-HOST>\`; fall back to \`ssh carbon\` on the LAN.
- \`Permission denied (publickey)\` — this client's key isn't in Carbon's authorized_keys (rotated/removed). Re-install it.
- Both ICMP pings fail but SSH works — that's normal here; ping is firewalled on Carbon. Never use ping as the reachability test (see \`reachabilityCheck\`).
`,
    },

    screenshot: {
      description:
        "Capture Carbon's current screen over SSH (no RDP/VNC viewer needed) and pull it back to this workstation. Uses Carbon's take-screenshot.ps1 scheduled task, which must run in the interactive session (/it).",
      howTo: `
End state: a fresh PNG of Carbon's screen at \`C:\\Users\\xsj70\\screenshot-carbon.png\` on THIS box. Lets you visually verify Carbon without an interactive RDP/VNC window.

\`\`\`powershell
# 1. Trigger Carbon's screenshot scheduled task (writes C:\\Users\\xsj70\\screenshot.png on Carbon).
#    Tailscale first; swap to 'carbon' for LAN.
ssh -o BatchMode=yes xsj70@<REDACTED-IP> "schtasks /run /tn TakeScreenshot"
Start-Sleep 2
# 2. Copy it back (always OVERWRITE the same filename — never create screenshot-1, -2, ...).
scp xsj70@<REDACTED-IP>:screenshot.png C:/Users/xsj70/screenshot-carbon.png
\`\`\`

Notes:
- The \`TakeScreenshot\` task must be registered with the \`/it\` (interactive) flag or it captures a black session-0 frame. The capture script lives on Carbon at \`C:\\Users\\xsj70\\take-screenshot.ps1\`.
- **Rule: one screenshot file, always overwritten** — both on Carbon (\`screenshot.png\`) and locally (\`screenshot-carbon.png\`). No clutter.
- If the monitor was turned off (Carbon's \`monitor-off.ps1\`), SSH may drop from NIC power-saving; turn it back on with \`monitor-on.ps1\` over SSH and retry.
`,
    },

    enableVncOverTailscale: {
      description:
        "Make the no-MS-account VNC path work OFF-LAN. Carbon answers VNC :5900 on the LAN NIC (<REDACTED-IP>) but NOT on the tailnet (<REDACTED-IP>) — almost certainly a Windows Firewall scope. Broaden the :5900 inbound rule to the Tailscale interface so RealVNC works from anywhere.",
      howTo: `
## The symptom (verified 2026-06-08)

\`Test-NetConnection <REDACTED-IP> -Port 5900\` → **True**, but \`Test-NetConnection <REDACTED-IP> -Port 5900\` → **False**. RDP/3389 answers on both; only VNC is LAN-scoped. So today the no-Microsoft-account VNC path only works on the home Wi-Fi.

## Most likely cause

The RealVNC server listens on \`<REDACTED-IP>:5900\` (all interfaces), but Windows Defender Firewall's inbound rule for 5900 is scoped to the **local subnet / Private profile**, so packets arriving on the Tailscale adapter (source <REDACTED-IP>/10) are dropped. Confirm on Carbon over SSH:

\`\`\`powershell
ssh -o BatchMode=yes xsj70@<REDACTED-IP> "powershell -Command \\"Get-NetTCPConnection -LocalPort 5900 -State Listen | Select LocalAddress,LocalPort; Get-NetFirewallRule -DisplayName '*VNC*' | Get-NetFirewallAddressFilter | Select RemoteAddress\\""
\`\`\`

- \`LocalAddress <REDACTED-IP>\` confirms the server isn't the limiter → it's the firewall.
- A \`RemoteAddress\` of \`LocalSubnet\` (not \`Any\`) on the VNC rule confirms the scope.

## The fix — allow the Tailscale interface to reach :5900

Run on Carbon over SSH (one-time). Scope to the tailnet CIDR, NOT \`Any\` — don't expose VNC to the whole internet:

\`\`\`powershell
ssh -o BatchMode=yes xsj70@<REDACTED-IP> "powershell -Command \\"New-NetFirewallRule -DisplayName 'VNC over Tailscale' -Direction Inbound -Action Allow -Protocol TCP -LocalPort 5900 -RemoteAddress <REDACTED-IP>/10\\""
\`\`\`

Then re-probe from this workstation:

\`\`\`powershell
(Test-NetConnection <REDACTED-IP> -Port 5900 -WarningAction SilentlyContinue).TcpTestSucceeded   # expect True
\`\`\`

Once True, \`connectRemoteDesktop\` can use VNC over Tailscale (\`vncviewer.exe <REDACTED-IP>:5900\`) — the no-MS-account path, now working off-LAN — and there's no remaining reason to use the LAN IP for anything.

Security note: relying on the Tailscale ACL + a <REDACTED-IP>/10-scoped rule keeps :5900 off the public internet. Don't widen the rule to \`Any\`.
`,
    },

    reachabilityCheck: {
      description:
        "Decide whether Carbon is up and which transport to use. ICMP ping is firewalled on Carbon — a failed ping is NOT a down signal. Probe real TCP ports (Tailscale first), and confirm <REDACTED-HOST> is on the tailnet.",
      howTo: `
\`\`\`powershell
# Tailscale node present + online?
tailscale status | findstr <REDACTED-HOST>          # expect: <REDACTED-IP>  <REDACTED-HOST>  ... active

# TCP probes (this is the real up-check — NOT ping):
"RDP  TS : " + (Test-NetConnection <REDACTED-IP> -Port 3389 -WarningAction SilentlyContinue).TcpTestSucceeded
"RDP  LAN: " + (Test-NetConnection <REDACTED-IP>  -Port 3389 -WarningAction SilentlyContinue).TcpTestSucceeded
"VNC  LAN: " + (Test-NetConnection <REDACTED-IP>  -Port 5900 -WarningAction SilentlyContinue).TcpTestSucceeded
"SSH  LAN: " + (Test-NetConnection <REDACTED-IP>  -Port 22   -WarningAction SilentlyContinue).TcpTestSucceeded
\`\`\`

Interpretation:
- \`RDP TS = True\` → use Tailscale for remote desktop (the default). Done.
- \`RDP TS = False\` but \`<REDACTED-HOST>\` shows \`active\` → transient; retry. If <REDACTED-HOST> is missing/offline → Tailscale path unavailable, fall back to LAN.
- All LAN probes True but TS probes False → you're on Carbon's Wi-Fi but Tailscale is down on one end; either is fine, but prefer fixing Tailscale so off-LAN keeps working.
- ICMP ping failing while these TCP probes pass is EXPECTED — ping is firewalled. Never report "Carbon is down" off a failed ping.
`,
    },

    facts: {
      description: "Canonical hosts, IPs, ports, and credential model for Carbon. Single source of truth — read before hand-typing any address.",
      howTo: `
| Thing | Value | Notes |
|---|---|---|
| Tailscale hostname | \`<REDACTED-HOST>\` | DEFAULT transport target |
| Tailscale IPv4 | \`<REDACTED-IP>\` | stable; also JetRails-whitelisted (bosterbio book) |
| LAN IPv4 | \`<REDACTED-IP>\` | fallback only; DHCP (reserve in router) |
| SSH user | \`xsj70\` | key \`~/.ssh/id_ed25519\`, alias \`ssh carbon\` (LAN) |
| SSH port | \`22\` | OpenSSH server, key already authorized |
| RDP port | \`3389\` | reachable over BOTH TS and LAN |
| RDP login | **full Microsoft-account email** + MS password | NOT \`xsj70\`; agent must not type the password |
| VNC port | \`5900\` | RealVNC; LAN-only until \`enableVncOverTailscale\` |
| VNC login | standalone **VNC password** | separate from the MS account |
| ICMP ping | firewalled on both NICs | use TCP probes, never ping, for up-checks |

Paths on Carbon: \`C:\\Users\\xsj70\\bosternexus\\\` (boster_nexus repo), \`C:\\Users\\xsj70\\take-screenshot.ps1\`, \`monitor-off.ps1\`, \`monitor-on.ps1\`.

Boundary with the **bosterbio** skill book: that book uses Carbon only as a Tailscale ProxyJump hop (\`ssh -J xsj70@<REDACTED-IP> ...\`) to reach the JetRails-whitelisted production server. THIS book owns controlling Carbon itself (remote desktop, SSH-to-Carbon, screen capture). Don't duplicate the ProxyJump recipe here.
`,
    },
  },
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->
