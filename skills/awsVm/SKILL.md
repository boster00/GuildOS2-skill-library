# awsVm

<!-- GUILDOS-SKILL name="awsVm" -->

<!-- GUILDOS-DESC -->
SSH + graphical remote control of the operator's AWS-hosted Windows VM '<REDACTED-HOST>' (<REDACTED-INSTANCE-ID>, us-east-2, public IP <REDACTED-IP>). Headless: alias `ssh guildos-aws-vm` (key auth, User Administrator, remote shell = PowerShell 7). GUI: Amazon DCV browser desktop at https://<REDACTED-IP>:8443 driven via Claude-in-Chrome. Use to run remote commands, drive the desktop, provision on-demand cloud worker VMs, or repair a broken SSH path. Launching/terminating instances is weapon/aws territory.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/awsVm/SKILL.md -->
```markdown
---
name: awsVm
description: >-
  SSH + graphical remote control of the operator's AWS-hosted Windows VM
  '<REDACTED-HOST>' (<REDACTED-INSTANCE-ID>, us-east-2, public IP <REDACTED-IP>).
  Headless: alias `ssh guildos-aws-vm` (key auth, User Administrator, remote
  shell = PowerShell 7). GUI: Amazon DCV browser desktop at
  https://<REDACTED-IP>:8443 driven via Claude-in-Chrome. Use to run remote
  commands, drive the desktop, provision on-demand cloud worker VMs, or repair a
  broken SSH path. Launching/terminating instances is weapon/aws territory.
---

# AWS VM (<REDACTED-HOST> Windows desktop)

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/awsVm/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto awsVm <action>` (or read the source directly).

## Actions

- **connect** — SSH into the AWS VM via the `guildos-aws-vm` alias. Key-based, non-interactive; includes the canonical smoke test that proves the path end-to-end.
- **dcvDesktop** — Graphically control the VM's Windows desktop through the Amazon DCV browser client, driven via Claude-in-Chrome (CIC). Use this ONLY when you need the GUI (see a window, click a native app); for headless commands use connect/runCommand over SSH. Two operator-gated steps (cert bypass + password) are unavoidable — the agent cannot cross either.
- **facts** — Canonical ids, endpoints, ports, and credential locations for the AWS VM. Single source of truth — read before hand-typing any id or address.
- **provisionWorker** — Spin up a FULL GuildOS worker + team desktop VM ON DEMAND, and stop/terminate it. Each teammate gets a cloud workstation that leverages AWS's multi-Gbps bandwidth (their local internet is the bottleneck). Auto-stops when idle so it bills like a laptop-you-only-run-when-working, not a 24/7 server. Zero persistent fleet — launch per need.
- **runCommand** — Run a one-shot remote command on the AWS VM over SSH. Remote shell is PowerShell 7 (pwsh) — write PowerShell syntax, quote for it, and use BatchMode so automation never hangs on a prompt.
- **troubleshoot** — Repair playbook when `ssh guildos-aws-vm` fails: operator IP changed (SG lockout), instance IP changed (stop/start), sshd dead, or key rejected. Includes the SSM out-of-band channel that fixes everything without a reboot.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/awsVm/engineering_decisions.md -->
```markdown
# awsVm — engineering decisions

## 2026-07-03 · Daily transport = plain SSH over the public IP with a /32-scoped SG rule, NOT SSM Session Manager

- **Decision:** `ssh guildos-aws-vm` connects directly to the instance's public IP on port 22, allowed by a security-group rule scoped to the operator's `/32`. SSM (`AWS-StartSSHSession` ProxyCommand) was evaluated and rejected as the daily path.
- **Rejected alternative:** SSM port-forward / ProxyCommand wrapped in the ssh alias (zero open inbound ports).
- **Why:** the operator's contract is *plain `ssh guildos-aws-vm` from this Windows box* — an SSM ProxyCommand makes every ssh invocation depend on formulary AWS creds being resolvable plus the session-manager-plugin binary, which breaks the alias in exactly the contexts (fresh shells, scp, tools that shell out to ssh) where it must "just work". The instance's existing exposure posture is already public-IP + /32-scoped ports (3389/8443 since launch), so one more /32 port adds no new class of exposure.
- **Consequence-if-violated:** converting the alias to SSM-only will make ssh/scp fail whenever the MCP/formulary chain is down and reintroduce a cred dependency on every hop; deleting the /32 rule instead of *updating* it (see next ED) locks SSH out entirely.

## 2026-07-03 · SSM instance profile `guildos-ec2-ssm` attached anyway — as the out-of-band repair channel

- **Decision:** even though daily transport is plain SSH, the instance keeps an IAM instance profile with `AmazonSSMManagedInstanceCore`, and the troubleshoot action routes all repairs (dead sshd, key re-plant, lockout) through SSM RunCommand.
- **Rejected alternative:** no instance profile (the instance's original state), repairing via reboot/user-data or the EC2 serial console.
- **Why:** with SSH broken there is otherwise NO non-disruptive way into a Windows instance — user-data edits require a stop/start (rotates the public IP and kills the operator's desktop session), and the DCV/RDP path shares the same SG-lockout failure mode as SSH. SSM is the only channel that survives all three failure classes without touching the instance's run state. This is also how SSH was bootstrapped in the first place (sshd install + key plant ran over SSM RunCommand).
- **Consequence-if-violated:** detaching the profile "because nothing uses SSM day-to-day" turns the next sshd failure or IP rotation into a forced reboot of the operator's live desktop.

## 2026-07-03 · SSH key = workstation `id_ed25519` in `administrators_authorized_keys`, NOT the EC2 key pair `.pem`

- **Decision:** the alias authenticates with the operator workstation's standard `~/.ssh/id_ed25519`; its pubkey is planted in `C:\ProgramData\ssh\administrators_authorized_keys` on the VM (with the strict `icacls` ACL Windows OpenSSH requires).
- **Rejected alternative:** pointing `IdentityFile` at `~/.ssh/<REDACTED-HOST>-key.pem` (the EC2 key pair created at launch).
- **Why:** the EC2 key pair is not wired into sshd at all on Windows AMIs — its only function is RSA-decrypting the Administrator RDP password, so using it as `IdentityFile` fails auth while *looking* plausible. One standard workstation key across hosts also matches the carbon / raspberryPiLocal precedent.
- **Consequence-if-violated:** an agent "fixing" the config to use the instance's own key pair breaks auth and sends debugging down the wrong path (the key exists and matches the instance, but sshd has never heard of it).

## 2026-07-03 · GuildOS runs as a Scheduled Task (SYSTEM), never a `Start-Process` launched over SSH

- **Decision:** the GuildOS dev server on the VM is a Windows Scheduled Task (`GuildOS-dev`, runs as SYSTEM, trigger AtStartup) executing `run-dev.ps1`. It is NOT launched by `Start-Process` from an interactive SSH command.
- **Rejected alternative:** `ssh guildos-aws-vm "Start-Process npm run dev:app"` (the obvious "just start it" one-liner).
- **Why:** Windows OpenSSH terminates the session's child-process tree when the SSH connection closes — a `Start-Process`-detached node server LOOKS alive (returns a PID, serves 200 during the session) but is dead seconds after the ssh call returns. Verified directly: first-session probe = 200, immediate fresh-session probe = `node_procs=0`, connection refused. Only a Task-Scheduler-owned process outlives the session (and survives reboot).
- **Consequence-if-violated:** "started it over ssh, it 200'd" is a FALSE positive — always confirm durability from a SECOND, separate ssh session (or via the scheduled task), never from the same session that launched it.

## 2026-07-03 · Cron / questTick / caravan FORCED OFF on this VM — env vars set in-process, not just in `.env.local`

- **Decision:** `run-dev.ps1` sets `CRON_ACTIVE=false`, `GUILDOS_DISABLE_QUESTTICK=true`, `GUILDOS_CARAVAN_DISPATCH=false` as real process-env vars before `npm run dev:app`, and runs `dev:app` (Next only) — never `dev` (which also spawns the cron via `concurrently`).
- **Rejected alternative:** trusting the copied `.env.local`, or running the full `npm run dev`.
- **Why:** `@next/env` never overrides a var already present in `process.env`, so setting these in-process GUARANTEES they win regardless of what the copied `.env.local` says (first-win/last-win ambiguity across dotenv versions makes file-append unreliable). This host shares the live Supabase quest bus with Carbon and the operator's workstation; a second cron loop here would double-dispatch/lease quests and corrupt the live pipeline. Belt-and-suspenders: `dev:app` doesn't launch cron at all.
- **Consequence-if-violated:** running `npm run dev` (or flipping cron on without a distinct `LOCAL_GUILD_NAME` scoping) turns this VM into a rogue worker racing the real workers for the same quests — a live-data hazard, not a local-only bug.

## 2026-07-03 · On-demand workers via bootstrap-on-launch + SSM secrets + auto-stop — NOT a baked golden AMI, NOT a persistent fleet

- **Decision:** teammates get full GuildOS worker desktops spun up ON DEMAND (zero persistent fleet). The reusable primitive is a Launch Template whose user-data runs `libs/weapon/aws/worker-provision.ps1`, which pulls `.env.local` from an SSM SecureString at launch and registers an auto-stop-when-idle task. Instance size is chosen AT LAUNCH.
- **Rejected alternatives:** (a) a pre-baked golden AMI as the primary primitive; (b) baking `.env.local` into the image; (c) an always-on fleet sized up front.
- **Why:** the operator is cost-conscious (explicitly compared a 32GB always-on box to buying a laptop) and wants EASE, not a standing fleet. Bootstrap-on-launch keeps ONE script as the source of truth (edit once, every future VM updates; no stale image, no sysprep dance) — a golden AMI is an optional speed optimization layered on later, not the foundation. Secrets-at-launch via SSM keeps the launch artifact safe to reuse/share and lets a single rotation propagate. Auto-stop makes billing track actual usage (a 6-hr/day worker ≈ ¼ the cost of always-on), which is the whole reason cloud workers beat buying laptops. Size-at-launch means the current undersized t3.medium never constrains the fleet.
- **Consequence-if-violated:** baking secrets into an AMI turns every clone into a credential-leak surface and forces a re-image on every rotation; standing up an always-on fleet burns money the operator explicitly didn't want; a golden AMI as the ONLY source of truth drifts silently from the provisioning reality.

## 2026-07-03 · GUI control = Amazon DCV via CIC; SSH is headless-only (Windows session isolation)

- **Decision:** two distinct remote-control channels, used for different jobs. SSH (`connect`/`runCommand`) for all headless/scripted work; Amazon DCV browser desktop (8443) driven via Claude-in-Chrome (`dcvDesktop`) for anything graphical.
- **Rejected alternative:** trying to drive the GUI over SSH (e.g. `Start-Process notepad` and expecting it on the desktop), or standing up a separate VNC/RDP-automation path.
- **Why:** a process launched over Windows OpenSSH runs in a non-interactive session, so its windows never render on the interactive DCV/console desktop (verified: an SSH-created Desktop *file* appears after F5, but an SSH-launched *app window* does not). The GUI genuinely has to be driven through the interactive session, and DCV-in-a-browser is the only channel CIC can reach. Keeping the two channels separate (SSH headless, DCV graphical) avoids chasing invisible windows.
- **Consequence-if-violated:** automating GUI steps over SSH silently no-ops (the window exists in a hidden session) — debugging burns time on "why didn't the app open?" when the real answer is session isolation.

## 2026-07-03 · DCV login is operator-gated at TWO walls the agent cannot cross

- **Decision:** the `dcvDesktop` flow explicitly stops and hands off at (1) the self-signed-cert interstitial and (2) the password field; the agent only navigates, types the non-secret username, and drives the desktop *after* login.
- **Rejected alternative:** assuming CIC can fully automate DCV login end-to-end.
- **Why:** (1) Chrome's cert-error interstitial is a browser-privileged page CIC cannot attach to — click/read/screenshot all fail; only a human (or a remembered per-profile bypass) gets past it. (2) Typing a password into a login field is a hard prohibited agent action even on the operator's own box. Both were hit live 2026-07-03. Documenting them as *expected* operator steps (not failures) is what keeps a future agent from thrashing against them.
- **Consequence-if-violated:** an agent that treats the cert page as a transient error will retry-loop forever; one that types the password violates the safety rule. The correct move is the documented hand-off.

## 2026-07-03 · DCV 8443 open to <REDACTED-IP>/0; the VM's team is the PRIMARY user

- **Decision:** SG opens 8443 (DCV) to `<REDACTED-IP>/0`; RDP 3389 stays per-/32 (operator + team member `<REDACTED-IP>`). The operator's TEAM is the primary user of this VM (operator is secondary).
- **Rejected alternative:** per-/32 whitelisting DCV too (rejected by the operator — team connects from varying networks), and — the actual mistake made and reversed this day — treating the VM as unused and revoking `<REDACTED-IP>`.
- **Why:** explicit operator decision to trade network-level restriction for team convenience on the browser-desktop port; the Administrator password is accepted as the sole barrier there. `<REDACTED-IP>` is a live teammate — an idle-looking box (no RDP logins, empty desktop) is NOT evidence it's unused when it's provisioned for a team that hasn't connected yet.
- **Consequence-if-violated:** re-locking 8443 to a whitelist cuts the team off; revoking <REDACTED-IP> (done once, restored) locks out a teammate. Since 8443 is internet-facing, a weak Administrator password is the whole exposure — a stronger one is the standing recommendation.

## 2026-07-03 · Remote DefaultShell = PowerShell **7** (`pwsh.exe`), installed just for this — not cmd.exe, not Windows PowerShell 5.1

- **Decision:** setup installed PowerShell 7.4 on the VM and wrote the `HKLM:\SOFTWARE\OpenSSH DefaultShell` registry value → `C:\Program Files\PowerShell\7\pwsh.exe`. A one-line AllUsers profile (`C:\Program Files\PowerShell\7\profile.ps1`) defines `function uptime { Get-Uptime }`.
- **Rejected alternatives:** cmd.exe (the Windows OpenSSH native default) and the preinstalled Windows PowerShell 5.1.
- **Why:** the operator-approved smoke test is verbatim `ssh guildos-aws-vm "hostname && whoami && uptime"`. PS 5.1 cannot parse `&&` at all (hard parse error), and cmd.exe has `&&` but forces cmd syntax on every other recipe in this book. Only pwsh 7 gives both `&&` chaining and PowerShell semantics. `uptime` exists on no Windows shell — the profile function supplies it (profiles DO load for `pwsh -c` one-shots; that's relied on, so never add `-NoProfile` to the sshd invocation).
- **Consequence-if-violated:** flipping DefaultShell back to powershell.exe/cmd.exe breaks the canonical smoke test itself (`&&` parse error, or `uptime` unrecognized) and every PowerShell-syntax recipe in runCommand; deleting the profile.ps1 breaks only `uptime` — a confusing partial failure.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/awsVm/index.js -->
```js
/**
 * awsVm — the operator's Windows VM on AWS ("<REDACTED-HOST>").
 *
 * Per-machine access skill book, same genre as `carbon` and `raspberryPiLocal`:
 * this book owns HOW to reach and operate the AWS-hosted Windows desktop VM.
 * Launch/teardown of instances stays in `weapon/aws` (launchWindowsDesktop /
 * terminateInstance); account-level AWS discipline stays in the `aws` skill
 * book. Don't duplicate either here.
 *
 * Canonical facts (verified 2026-07-03):
 *   - Instance: <REDACTED-INSTANCE-ID>, Name tag `<REDACTED-HOST>`, t3.medium,
 *     Windows Server 2022, us-east-2, created by `weapon/aws.launchWindowsDesktop`.
 *   - Public IP: <REDACTED-IP> (NOT an Elastic IP — changes on stop/start; see
 *     troubleshoot). Private IP <REDACTED-IP>.
 *   - Security group: sg-00800404871e8dc0f (`<REDACTED-HOST>-sg`). Every ingress
 *     rule is /32-scoped to the operator's IP: 22 (SSH), 3389 (RDP), 8443 (DCV).
 *     No <REDACTED-IP>/0 anywhere — keep it that way.
 *   - SSH alias: `guildos-aws-vm` in ~/.ssh/config → User Administrator,
 *     IdentityFile ~/.ssh/id_ed25519 (the workstation's standard key, planted
 *     in the VM's administrators_authorized_keys). The EC2 key pair
 *     `<REDACTED-HOST>-key` (~/.ssh/<REDACTED-HOST>-key.pem) is NOT the SSH key —
 *     it only decrypts the Windows Administrator password (see facts).
 *   - Out-of-band channel: SSM (instance profile `guildos-ec2-ssm`,
 *     AmazonSSMManagedInstanceCore). Used for repair when SSH is broken —
 *     never for day-to-day commands.
 *   - Remote shell over SSH = PowerShell 7 (pwsh.exe as the OpenSSH
 *     DefaultShell registry value). Write remote one-shots in PowerShell
 *     syntax, not cmd/bash; `&&` chaining works (PS7, not PS5.1).
 */

export const skillBook = {
  id: "awsVm",
  title: "AWS VM (<REDACTED-HOST> Windows desktop)",
  description:
    "SSH + graphical remote control of the operator's AWS-hosted Windows VM '<REDACTED-HOST>' (<REDACTED-INSTANCE-ID>, us-east-2, public IP <REDACTED-IP>). Headless: alias `ssh guildos-aws-vm` (key auth, User Administrator, remote shell = PowerShell 7). GUI: Amazon DCV browser desktop at https://<REDACTED-IP>:8443 driven via Claude-in-Chrome. Actions: connect (SSH in / smoke test), runCommand (one-shot remote PowerShell + stdin-pipe pattern + session-isolation caveat), dcvDesktop (CIC graphic control through the DCV browser client — two operator-gated steps: cert bypass + password), provisionWorker (spin up a full GuildOS worker+desktop VM on demand, auto-stop-when-idle, size chosen at launch — the repeatable per-teammate cloud-workstation process), troubleshoot (IP changed / SG lockout / sshd dead — includes the SSM out-of-band repair path), facts (canonical ids/ports/keys/credentials). Launching or terminating instances is weapon/aws territory, not this book.",
  steps: [],
  toc: {
    connect: {
      description:
        "SSH into the AWS VM via the `guildos-aws-vm` alias. Key-based, non-interactive; includes the canonical smoke test that proves the path end-to-end.",
      howTo: `
End state: an authenticated SSH session (or one-shot command) on the VM as \`Administrator\`, no password or host-key prompt.

\`\`\`powershell
# Canonical smoke test (operator-approved wording — run verbatim):
ssh guildos-aws-vm "hostname && whoami && uptime"
\`\`\`

PASS = completes non-interactively in <10s, exit 0, prints hostname \`EC2AMAZ-469NL1I\`. Verified output (2026-07-03, ran twice back-to-back):

\`\`\`
EC2AMAZ-469NL1I
ec2amaz-469nl1i\\administrator
<Get-Uptime table — days/hours since last boot>
\`\`\`

Notes:
- The alias lives in \`~/.ssh/config\` (Host guildos-aws-vm → HostName <REDACTED-IP>, User Administrator, IdentityFile ~/.ssh/id_ed25519, IdentitiesOnly yes). IP changes are fixed by editing ONE line there — see troubleshoot.
- Remote shell is **PowerShell 7** (\`pwsh.exe\` as DefaultShell — NOT Windows PowerShell 5.1, which cannot parse \`&&\`). \`uptime\` resolves via a one-line function in the AllUsers pwsh profile (\`C:\\Program Files\\PowerShell\\7\\profile.ps1\` → \`Get-Uptime\`) — on a fresh Windows box it would NOT exist (see engineering_decisions.md).
- For interactive work just \`ssh guildos-aws-vm\` — you land in a PowerShell prompt.
- Use \`-o BatchMode=yes\` for automation so a broken key fails fast instead of hanging on a password prompt it can never answer.
`,
      input: { command: "string — optional one-shot command; omit for interactive session" },
      output: { exitCode: "number", stdout: "string" },
    },

    runCommand: {
      description:
        "Run a one-shot remote command on the AWS VM over SSH. Remote shell is PowerShell 7 (pwsh) — write PowerShell syntax, quote for it, and use BatchMode so automation never hangs on a prompt.",
      howTo: `
\`\`\`powershell
# Template — from this workstation:
ssh -o BatchMode=yes guildos-aws-vm "Get-Process | Sort-Object CPU -Descending | Select-Object -First 5"

# File transfer uses the same alias:
scp C:/local/file.txt guildos-aws-vm:C:/Users/Administrator/file.txt
scp guildos-aws-vm:C:/path/remote.log C:/local/remote.log
\`\`\`

Rules:
- **PowerShell 7 syntax on the remote side** (DefaultShell = pwsh.exe). No bash-isms, no cmd-isms like \`dir /b\`; \`&&\`/\`||\` chaining is fine.
- **Quoting (Windows-OpenSSH quirk — verified 2026-07-03):** sshd on Windows strips ONE layer of *double* quotes in transit but keeps *single* quotes, so the remote pwsh word-splits any double-quoted string that crossed the wire unescaped. Concretely:
    - Literal string → wrap the remote arg in SINGLE quotes: \`ssh guildos-aws-vm "echo 'Hello, World!'"\` → \`Hello, World!\` ✓
    - Needs \`$var\` expansion (single quotes don't expand in pwsh) → backtick-escape the inner double quotes so they arrive literal: \`ssh guildos-aws-vm "Write-Output \`\\"Hi from \`$env:COMPUTERNAME\`\\""\` ✓
    - Plain double quotes inside a single-quoted LOCAL string word-split on the remote — avoid: \`ssh guildos-aws-vm 'Write-Output "Hi $env:COMPUTERNAME"'\` ✗
  For anything past a one-liner, sidestep quoting entirely — BEST pattern (no scratch file, no quoting): pipe a here-string into remote pwsh over stdin. Lead the script with a \`#\` comment line so any UTF-8 BOM lands on a harmless comment:
\`\`\`powershell
@'
# leading comment absorbs the BOM
$f = "C:\\Users\\Administrator\\Desktop\\marker.txt"
Set-Content -Path $f -Value ("written " + (Get-Date)) -Encoding utf8
Get-Content $f
'@ | ssh guildos-aws-vm "pwsh -NoProfile -Command -"
\`\`\`
  Alternatives: scp a .ps1 to the VM and run it, or SSM RunCommand (troubleshoot has the pattern) — both skip the shell re-parse.
- **Windows session isolation (verified 2026-07-03):** commands over SSH run in a NON-interactive session, NOT the DCV/console desktop session. FILES you create (e.g. on the Desktop) DO appear on the DCV desktop (filesystem is shared — press F5 to refresh Explorer). But a GUI app you \`Start-Process\` over SSH launches in the hidden SSH session and is INVISIBLE on the DCV desktop. To make a window appear on the interactive desktop, launch it from a Scheduled Task with \`-Interactive\`, or just operate the GUI through DCV (see \`dcvDesktop\`).
- Long-running work: don't hold the SSH session open for hours — schedule on the VM (\`Register-ScheduledTask\`) or run via SSM RunCommand (troubleshoot has the pattern) and poll.
- Exit code of the remote PowerShell propagates to \`ssh\`'s exit code — check \`$LASTEXITCODE\` locally for automation.
`,
      input: { command: "string — PowerShell to run on the VM" },
      output: { exitCode: "number", stdout: "string" },
    },

    dcvDesktop: {
      description:
        "Graphically control the VM's Windows desktop through the Amazon DCV browser client, driven via Claude-in-Chrome (CIC). Use this ONLY when you need the GUI (see a window, click a native app); for headless commands use connect/runCommand over SSH. Two operator-gated steps (cert bypass + password) are unavoidable — the agent cannot cross either.",
      howTo: `
End state: the VM's live Windows desktop rendered inside a CIC-controllable Chrome tab, where CIC clicks/keystrokes on the DCV canvas drive the remote desktop.

## What DCV is
Amazon DCV Server is a browser-based remote-desktop gateway installed on the VM by \`weapon/aws.launchWindowsDesktop\` at creation. It serves at **https://<publicIp>:8443** (currently https://<REDACTED-IP>:8443). Service name \`dcvserver\`; it keeps a \`console\` session owned by Administrator. Firewall: 8443 is open to **<REDACTED-IP>/0** (operator decision 2026-07-03 — the team connects from anywhere; the Administrator password is the only barrier).

## The two walls the agent CANNOT cross (hand these to the operator)
1. **Self-signed cert interstitial.** On the FIRST CIC navigation to the DCV URL, Chrome shows "Your connection is not private" — a browser-privileged page CIC canNOT attach to (screenshot/read_page/click all fail with "showing error page" / "Cannot attach to this target"). The operator must click through ONCE (Advanced → Proceed, or click the page and type the Chrome bypass phrase). Chrome then remembers the bypass for the profile, and subsequent CIC navigations land straight on the DCV login form.
2. **Password entry.** Entering a password into a login field is a PROHIBITED agent action (hard safety rule) — this holds even though it's the operator's own VM. The agent MAY type the non-secret username (\`Administrator\`) into the DCV form; the operator types the password (operator-set — kept out of git) and clicks Sign in.

## The CIC flow (what the agent does vs. what the operator does)
1. Agent: \`navigate\` a CIC tab to https://<REDACTED-IP>:8443. If it lands on the cert interstitial → STOP and ask the operator to click through once (wall #1). If Chrome already remembers the bypass → it lands on the DCV "Sign in with your credentials" form.
2. Agent: \`left_click\` the Username field and \`type\` "Administrator" (allowed — not a secret). \`screenshot\` to confirm.
3. Operator: types the password into the Password field and clicks Sign in (wall #2).
4. Agent: once the desktop renders (tab title becomes "EC2AMAZ-469NL1I - Amazon DCV"), \`screenshot\` and drive the desktop — \`left_click\` on the DCV canvas maps to the remote desktop (Start menu, taskbar, app windows), \`type\`/\`key\` send keystrokes. A quick self-check demo: open the SSH-created \`Desktop\\marker.txt\`.

## Gotchas
- The CIC MCP tab group is separate from the operator's own Chrome tabs. If the operator logged into DCV in THEIR tab, that session is NOT CIC-controllable — bring DCV up in a CIC tab (step 1) instead.
- The DCV console is single-console: a second DCV login can take over the operator's current view. Coordinate if they're actively using it.
- Prefer SSH for anything doable headless — DCV control is slower (pixel round-trips) and needs the two operator-gated steps. Reach for DCV only for genuinely graphical tasks.
`,
      input: { tabId: "number — optional existing CIC tab to reuse; omit to open a fresh one" },
      output: {
        tabId: "number — the CIC tab now showing DCV",
        state: "'cert-wall' | 'login-form' | 'desktop' — where the flow stopped (cert-wall & login need the operator)",
      },
    },

    provisionWorker: {
      description:
        "Spin up a FULL GuildOS worker + team desktop VM ON DEMAND, and stop/terminate it. Each teammate gets a cloud workstation that leverages AWS's multi-Gbps bandwidth (their local internet is the bottleneck). Auto-stops when idle so it bills like a laptop-you-only-run-when-working, not a 24/7 server. Zero persistent fleet — launch per need.",
      howTo: `
Goal: one repeatable command turns a bare EC2 Windows instance into a complete worker (Node, Git, Chrome, uv/uvx, gh, ffmpeg, Claude Code CLI, Playwright browsers, GuildOS running) that the teammate reaches via the DCV browser desktop — then stops itself when idle.

## Why cloud workers (the operator's driver)
Team members' local internet is too slow; EC2 instances have multi-Gbps network. Running the network-heavy GuildOS work (scraping, biomarker hydration, cloud-agent dispatch, large media, package installs) ON the VM offloads all I/O to AWS's pipe. The VM is the *working environment*, accessed as a thin browser desktop.

## The provisioning source of truth
\`libs/weapon/aws/worker-provision.ps1\` — idempotent bootstrap that installs the full stack and pulls the env from SSM. It encodes exactly the steps proven manually 2026-07-03 plus the worker-only gap tools. It runs either as EC2 user-data on first boot OR via SSM RunCommand on an existing instance.

## One-time setup (per account, before the first launch)
1. **Secrets → SSM (secrets-at-launch, NOT baked into any image):** store the full \`.env.local\` as ONE SSM SecureString at \`/guildos/worker/env-local\` (KMS-encrypted). The bootstrap reads it at launch. Rotate once, every worker picks it up. NEVER bake secrets into an AMI.
2. **Instance role:** give the worker's instance profile \`AmazonSSMManagedInstanceCore\` (SSM + Parameter Store read) AND \`ec2:StopInstances\` on itself (for auto-stop). The existing \`guildos-ec2-ssm\` role has the first; add a small inline policy for the second.
3. **Launch Template** (the "template"): AMI = a base Windows Server 2022 AMI (or a pre-baked golden AMI for faster boots — optional), instance size = **chosen at launch** (operator is cost-conscious; default t3.large 8GB or t3.xlarge 16GB, NOT t3.2xlarge), SG = \`<REDACTED-HOST>-sg\` (DCV 8443 public, RDP/SSH per-IP), key pair, and user-data = worker-provision.ps1 (with per-worker \`-GuildName\` if it should also run cron).

## Launch a worker on demand
Use \`weapon/aws.launchWindowsDesktop\` (or launch from the Launch Template) with the SG opening 8443, then run the bootstrap via SSM:
\`\`\`
aws ssm send-command --instance-ids <id> --region us-east-2 --document-name AWS-RunPowerShellScript \\
  --parameters commands="$(Get-Content libs/weapon/aws/worker-provision.ps1 -Raw)"
\`\`\`
(or bake it as user-data so it self-provisions on first boot — no SSM step). ~10-15 min first boot; a golden AMI cuts that to ~1 min. Teammate then connects at https://<publicIp>:8443 (DCV), Administrator + the operator password.

## Bill like a laptop (auto-stop)
worker-provision.ps1 registers a \`GuildOS-auto-stop\` scheduled task: every 5 min it checks for zero active RDP/DCV sessions + low CPU; after \`-IdleStopMinutes\` (default 30) it calls \`ec2 stop-instances\` on itself. Resume = start the instance again (public IP rotates → the teammate re-opens the new DCV URL; give them an Elastic IP if a stable URL matters). Stopped instances bill only for EBS storage (~pennies/day), not compute.

## Per-worker identity (avoid quest-bus collisions)
Cron/questTick stay OFF by default (\`dev:app\` only). To make a worker a LIVE guild worker, pass a DISTINCT \`-GuildName\` — the bootstrap then runs full \`dev\` with \`LOCAL_GUILD_NAME\` set so it pulls only its own guild's quests. Two workers with the same (or empty-but-cron-on) name race for the same quests — the rogue-worker hazard.

## Golden AMI vs bootstrap-on-launch (see engineering_decisions.md)
Default is bootstrap-on-launch (one script is the source of truth; edit it and every future VM updates; no stale image, no sysprep). Bake a golden AMI only if the ~10-15 min first-boot install is too slow for the launch cadence — then sysprep before imaging and keep worker-provision.ps1 as the rebuild recipe.

## Teardown
\`weapon/aws.terminateInstance\` by id. Terminating is the full stop-the-billing move; \`stop\` (auto or manual) keeps the disk (and its state) for next time.
`,
      input: {
        name: "string — worker name/tag, e.g. 'guildos-worker-alice'",
        instanceType: "string — chosen at launch; default 't3.large'. Avoid t3.2xlarge unless the workload is heavy (operator is cost-conscious).",
        guildName: "string — optional LOCAL_GUILD_NAME to make it a live cron worker; omit to run app-only (cron OFF)",
        idleStopMinutes: "number — auto-stop after this many idle minutes; default 30, 0 = never",
      },
      output: {
        instanceId: "string",
        dcvUrl: "string — https://<publicIp>:8443",
        provisioned: "boolean — bootstrap completed",
      },
    },

    troubleshoot: {
      description:
        "Repair playbook when `ssh guildos-aws-vm` fails: operator IP changed (SG lockout), instance IP changed (stop/start), sshd dead, or key rejected. Includes the SSM out-of-band channel that fixes everything without a reboot.",
      howTo: `
Diagnose in this order — each failure has ONE distinct signature:

## 1. Timeout / no route → SG lockout (operator IP changed) or instance IP changed
Both present as \`Connection timed out\`.

\`\`\`powershell
# Current operator IP vs what the SG allows, and the instance's current public IP:
Invoke-RestMethod https://api.ipify.org
\`\`\`
Then via the aws MCP (\`mcp__aws__aws___call_aws\`):
\`\`\`
aws ec2 describe-instances --instance-ids <REDACTED-INSTANCE-ID> --region us-east-2 --query "Reservations[0].Instances[0].{ip:PublicIpAddress,state:State.Name}"
aws ec2 describe-security-groups --group-ids sg-00800404871e8dc0f --region us-east-2 --query "SecurityGroups[0].IpPermissions"
\`\`\`
- Instance IP ≠ \`~/.ssh/config\` HostName → edit the ONE HostName line (this is why the alias exists). The VM has NO Elastic IP, so every stop/start rotates the IP.
- Operator IP not in the SG's port-22 rule → add a fresh /32 rule (and prune the stale one):
\`\`\`
aws ec2 authorize-security-group-ingress --group-id sg-00800404871e8dc0f --region us-east-2 --ip-permissions "IpProtocol=tcp,FromPort=22,ToPort=22,IpRanges=[{CidrIp=<NEW_IP>/32,Description='SSH from operator workstation'}]"
aws ec2 revoke-security-group-ingress --group-id sg-00800404871e8dc0f --region us-east-2 --ip-permissions "IpProtocol=tcp,FromPort=22,ToPort=22,IpRanges=[{CidrIp=<OLD_IP>/32}]"
\`\`\`
NEVER widen to <REDACTED-IP>/0 to "make it work".

## 2. \`Connection refused\` or banner then drop → sshd not running on the VM
Repair over SSM (the out-of-band channel — instance profile \`guildos-ec2-ssm\` is attached exactly for this):
\`\`\`
aws ssm send-command --instance-ids <REDACTED-INSTANCE-ID> --region us-east-2 --document-name AWS-RunPowerShellScript --parameters commands="Start-Service sshd; Set-Service sshd -StartupType Automatic; Get-Service sshd"
aws ssm get-command-invocation --command-id <id> --instance-id <REDACTED-INSTANCE-ID> --region us-east-2
\`\`\`
SSM registration itself can lag ~10-30 min after instance start while the agent refreshes role creds — \`aws ssm describe-instance-information --region us-east-2\` must list the instance before send-command works.

## 3. \`Permission denied (publickey)\` → key mismatch
The SSH key is the workstation's \`~/.ssh/id_ed25519\`, planted in the VM's \`C:\\ProgramData\\ssh\\administrators_authorized_keys\` (admin accounts do NOT use ~/.ssh/authorized_keys on Windows OpenSSH). Re-plant over SSM:
\`\`\`
aws ssm send-command --instance-ids <REDACTED-INSTANCE-ID> --region us-east-2 --document-name AWS-RunPowerShellScript --parameters commands="Set-Content -Path C:\\ProgramData\\ssh\\administrators_authorized_keys -Value '<PUBKEY_LINE>' -Encoding ascii; icacls C:\\ProgramData\\ssh\\administrators_authorized_keys /inheritance:r /grant 'Administrators:F' /grant 'SYSTEM:F'"
\`\`\`
The strict icacls is mandatory — sshd silently ignores the file if non-admin ACLs are present.

## 4. Host-key prompt / MITM warning after a stop/start
A new public IP means a fresh known_hosts entry — accept once interactively, or \`ssh-keygen -R <old-ip>\`. Never disable StrictHostKeyChecking globally.

## 5. Instance stopped entirely
\`weapon/aws.describeInstance\` → if stopped, starting it is an operator-visible action on a desktop VM (it also rotates the public IP) — confirm before starting unless the task explicitly says so.
`,
      input: { symptom: "string — the ssh error line" },
      output: { fixed: "boolean", pathTaken: "string" },
    },

    facts: {
      description:
        "Canonical ids, endpoints, ports, and credential locations for the AWS VM. Single source of truth — read before hand-typing any id or address.",
      howTo: `
| Thing | Value | Notes |
|---|---|---|
| Instance id | \`<REDACTED-INSTANCE-ID>\` | Name tag \`<REDACTED-HOST>\`, t3.medium, Windows Server 2022 |
| Region | \`us-east-2\` | account 325698037467 (bosterbio) |
| Public IP | \`<REDACTED-IP>\` | NO Elastic IP — rotates on stop/start; alias HostName is the one place to update |
| Hostname | \`EC2AMAZ-469NL1I\` | smoke-test expected value |
| SSH alias | \`guildos-aws-vm\` | ~/.ssh/config on the operator workstation |
| SSH user | \`Administrator\` | key = workstation \`~/.ssh/id_ed25519\` → VM's administrators_authorized_keys |
| Remote shell | PowerShell 7 (\`pwsh.exe\`) | DefaultShell registry value; write PS syntax in remote commands |
| Security group | \`sg-00800404871e8dc0f\` (<REDACTED-HOST>-sg) | 22 → operator /32; 3389 → operator + \`<REDACTED-IP>\` (team) /32; 8443 → \`<REDACTED-IP>/0\` (public, per operator) |
| Admin login | user \`Administrator\`, password = operator-set (2026-07-03, NOT the old EC2-generated one) | LITERAL DELIBERATELY NOT COMMITTED — value lives in the operator's local notes / formulary, not in git |
| RDP | \`3389\`, \`Administrator\` / operator password | mstsc: use \`<REDACTED-IP>\\Administrator\` (the \`host\\\` prefix forces the VM's LOCAL account, not the workstation's cached Microsoft account) |
| DCV web desktop | \`https://<REDACTED-IP>:8443\` | browser desktop (Amazon DCV), same Administrator login; drive via CIC → \`dcvDesktop\` action |
| EC2 key pair | \`<REDACTED-HOST>-key\` → \`~/.ssh/<REDACTED-HOST>-key.pem\` | decrypts the ORIGINAL EC2 Administrator password only; STALE now that the password was reset — not the SSH key, not the current password |
| Team access | \`<REDACTED-IP>\` = a team member's IP (RDP); whole team via public DCV 8443 | operator's team is the PRIMARY user of this VM; do NOT revoke <REDACTED-IP> |
| Out-of-band | SSM, instance profile \`guildos-ec2-ssm\` | repair channel via aws MCP; not for daily commands |
| GuildOS app | \`C:\\GuildOS\` (staging branch), Node 24 + git via Chocolatey | \`next dev --turbo -p 3002\`; cron/questTick FORCED OFF |
| GuildOS service | Scheduled Task \`GuildOS-dev\` (runs as SYSTEM, AtStartup) → \`C:\\GuildOS\\run-dev.ps1\` | survives SSH-session close & reboot; logs to \`C:\\GuildOS\\dev.log\` |
| View GuildOS | \`ssh -L 3009:localhost:3002 -N guildos-aws-vm\` then browse \`http://localhost:3009\` | port 3002 is NOT open in the SG — tunnel only (least-exposure) |

**Running GuildOS on this VM (deployed 2026-07-03):** the app is a durable Scheduled Task, not an SSH child (Windows OpenSSH kills session children on disconnect — a plain \`Start-Process\` over ssh does NOT persist; that's why it's a task). The task launches \`run-dev.ps1\`, which sets \`CRON_ACTIVE=false\` / \`GUILDOS_DISABLE_QUESTTICK=true\` / \`GUILDOS_CARAVAN_DISPATCH=false\` in the process env BEFORE \`npm run dev:app\` so this host never becomes a rogue quest worker on the shared bus (see engineering_decisions.md). Manage it: \`ssh guildos-aws-vm 'Start-ScheduledTask GuildOS-dev'\` / \`Stop-ScheduledTask\` / \`Get-ScheduledTaskInfo GuildOS-dev\`. \`.env.local\` was scp'd from the operator workstation (never via SSM — SSM logs command text). To make this VM a LIVE guild worker, flip cron on in run-dev.ps1 and set \`LOCAL_GUILD_NAME\` — but that is an operator decision, not a default.

Credential locations: everything lives on the operator workstation under \`~/.ssh/\` (pem, password txt, ed25519 key). Nothing is in the repo and nothing may ever be committed. AWS API creds for the repair paths come from the formulary via \`weapon/aws\` / the aws MCP — see the \`aws\` skill book.

Boundary: launching/terminating/resizing instances = \`weapon/aws\` (launchWindowsDesktop, terminateInstance, describeInstance). Account-wide AWS discipline (tags, budget alarm, deployResource) = \`aws\` skill book. THIS book = reaching and operating this one VM.
`,
    },
  },
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/aws/cli.mjs -->
```js
#!/usr/bin/env node
/**
 * CLI for the aws weapon's EC2 actions. Resolves "@/..." imports the same way
 * run-action.mjs / aws-mcp-launch.mjs do (the weapon transitively imports
 * @/libs/council/database), then dispatches.
 *
 * Usage (run with --env-file=.env.local so SST_OWNER_USER_ID + Supabase keys load):
 *   node --env-file=.env.local libs/weapon/aws/cli.mjs launch [name] [--type t3.micro] [--cidr <REDACTED-IP>/0] [--region us-east-2]
 *   node --env-file=.env.local libs/weapon/aws/cli.mjs budget-alarm
 */
import { register } from "node:module";
import { pathToFileURL } from "node:url";
import { resolve as resolvePath } from "node:path";

register(
  pathToFileURL(resolvePath(import.meta.dirname, "..", "..", "..", "scripts", "_alias-hook.mjs")).href,
  pathToFileURL(resolvePath(import.meta.dirname, "..", "..", "..") + "/").href,
);

const mod = await import("@/libs/weapon/aws/index.js");
const args = process.argv.slice(2);
const cmd = args[0];
const flag = (n, d) => {
  const i = args.indexOf(`--${n}`);
  return i >= 0 && args[i + 1] ? args[i + 1] : d;
};
const userId = process.env.SST_OWNER_USER_ID || process.env.PIGEON_POST_OWNER_ID;

if (cmd === "launch") {
  const pos = args[1] && !args[1].startsWith("--") ? args[1] : null;
  const d = new Date();
  const stamp = `${d.getUTCFullYear()}${String(d.getUTCMonth() + 1).padStart(2, "0")}${String(d.getUTCDate()).padStart(2, "0")}-${String(d.getUTCHours()).padStart(2, "0")}${String(d.getUTCMinutes()).padStart(2, "0")}`;
  const name = pos || `guildos-vm-${stamp}`;
  const guard = await mod.ensureBudgetAlarm({ userId });
  console.log("BUDGET_GUARD " + JSON.stringify(guard));
  const res = await mod.launchVm({
    userId,
    name,
    instanceType: flag("type", "t3.micro"),
    sshCidr: flag("cidr", "<REDACTED-IP>/0"),
    region: flag("region", undefined),
  });
  console.log("LAUNCH " + JSON.stringify(res, null, 2));
  process.exit(res.ok ? 0 : 1);
} else if (cmd === "budget-alarm") {
  const res = await mod.ensureBudgetAlarm({ userId });
  console.log(JSON.stringify(res, null, 2));
  process.exit(res.ok ? 0 : 1);
} else {
  console.error("Usage: cli.mjs launch [name] [--type] [--cidr] [--region] | budget-alarm");
  process.exit(2);
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/aws/engineering_decisions.md -->
```markdown
# Engineering decisions — `libs/weapon/aws`

Rationale layer (per CLAUDE.md §6). Append; supersede in place when a decision
changes.

---

## 2026-05-28 — AWS credentials come from the Formulary (`profiles.env_vars`); `process.env` is a bootstrap-only fallback (LOCKED)

**Decision.** Credentials are read from `profiles.env_vars` (the Formulary)
exclusively. `process.env` is consulted ONLY as a fallback in the MCP launcher's
bootstrap case (before the database is reachable). `~/.aws/credentials` is never
read.

**Rejected alternative.** Read `~/.aws/credentials` or always permit `process.env`
(the AWS-SDK-native credential chain).

**Why.** A single source of truth is what makes credential rotation and per-user
audit possible. Dual sources (filesystem + env + Formulary) drift: a rotation
script writes the Formulary while the running process keeps reading a stale file or
shell env, silently using the wrong keys — and multi-user setups can't audit who
changed a key. The bootstrap fallback exists only because the launcher runs before
the DB is available.

**Consequence if violated.** Silent use of stale/wrong credentials after a
rotation; loss of the audit trail for who last changed a key.

---

## 2026-06-30 — EC2 `launchVm` signs requests in-weapon (SigV4), NOT through the `aws` MCP proxy

**Decision.** `launchVm` / `ensureBudgetAlarm` make EC2/CloudWatch/SNS calls via
the in-weapon `signedAwsRequest` SigV4 signer (Query protocol, reusing the same
helper STS/IAM use), independent of the `aws` MCP server.

**Rejected alternative.** The skill-book `aws.deployResource` howTo says all
deployments go through `mcp__aws__aws___call_aws`. The native move for a future
agent is therefore to delete this SigV4 EC2 code and route through the proxy.

**Why.** The in-session `mcp__aws__*` tools can fail to bind at Claude-Code
startup — the first-run `uvx` install of `mcp-proxy-for-aws` overruns the
connect window, and there is no in-session way to force a `/mcp` reconnect. When
that happens the proxy path is simply unavailable, yet the launcher + creds are
healthy. A self-contained signer keeps EC2 deploys working regardless of MCP
binding, with the full `deployResource` discipline (cost-guard, tags,
list-before-create, post-create verify) applied in code. This is a transport
choice, not "raw call_aws improvisation" — the discipline is preserved.

**Consequence if violated.** Deleting the SigV4 EC2 path re-couples VM launches
to a transport that intermittently fails to connect, reintroducing the
hard-stop this weapon action was written to remove.

---

## 2026-06-30 — New-account bootstrap: console-mint a key, then IMMEDIATELY rotate + delete it (LOCKED)

**Decision.** When onboarding a fresh AWS account there are no API creds yet, so
the first `mcp-server` access key must be created through the CIC console — its
plaintext secret unavoidably transits the agent transcript once to reach the
formulary. The bootstrap is NOT complete until that console key is **rotated**
(`rotateAccessKey` → fresh secret minted via the API, never in the transcript)
**and the exposed console key is deleted** (`deleteAccessKey`, added this date).
Steady state = exactly one active key whose secret never appeared in a transcript.

**Rejected alternative.** Stop at "console key works + written to formulary" (the
naive done-state). Also rejected: skipping `deleteAccessKey` and merely leaving the
exposed key `Inactive`.

**Why.** A transcript is a durable, greppable log; an AdministratorAccess secret
sitting in one is a standing compromise. Rotating replaces the exposed secret with
one that was only ever handled in-process; deleting removes the exposed key
material entirely rather than leaving an inactive-but-reactivatable liability.
`rotateAccessKey` alone only *deactivates* the prior key — the delete step is why
`deleteAccessKey` exists (it refuses to delete the formulary's live key, to prevent
lock-out).

**Consequence if violated.** A live or reactivatable AdministratorAccess key whose
secret is permanently visible in a chat transcript — full account takeover on leak.

---

## 2026-06-30 — Windows GetPasswordData must be decrypted with RSA_NO_PADDING + manual unpad, NOT RSA_PKCS1_PADDING (LOCKED)

**Decision.** `launchWindowsDesktop` / `getWindowsPassword` decrypt the EC2
`GetPasswordData` blob via `crypto.privateDecrypt({ padding: RSA_NO_PADDING })`
and then strip the PKCS#1 v1.5 padding by hand (`decryptWindowsPassword` helper:
skip `00 02`, skip non-zero PS, take bytes after the `00` separator).

**Rejected alternative.** The obvious/native approach every AWS doc + example
shows: `crypto.privateDecrypt({ padding: RSA_PKCS1_PADDING }, blob)`. AWS
encrypts the password with RSA PKCS#1 v1.5, so this *looks* correct.

**Why.** Modern Node (>=18.19 / 20.11, Marvin/Bleichenbacher mitigation) **throws
`"RSA_PKCS1_PADDING is no longer supported for private decryption"`** from
`privateDecrypt`. In a poll-and-swallow loop this manifests as the password
*never becoming available* — the true cause (a hard crypto error) is masked as
"Windows still initializing," sending you down a wrong diagnostic path (bigger
instance, longer waits) that never fixes it. Decrypting raw + unpadding by hand
sidesteps the blocked padding mode.

**Consequence if violated.** Reverting to `RSA_PKCS1_PADDING` makes every Windows
launch silently fail to surface a password, indistinguishable from a slow boot —
the exact multi-attempt dead-end this replaced.

**Also note (context, not itself an ED).** Deactivating a single key via the IAM
console is hazardous: the per-key Actions menu co-mingles "Delete key" with
"Delete user", and a cookie-consent `[role=dialog]` overlays the real modal — a
loose button-text match once landed on the *Delete-user* dialog's "Deactivate
access keys" control and disabled BOTH keys. Do credential-key mutations through
the weapon (`rotateAccessKey` / `deleteAccessKey`), not console clicks.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/aws/index.js -->
```js
/**
 * AWS weapon — formulary-backed credential retrieval for AWS APIs.
 *
 * GuildOS rule (locked 2026-05-28): AWS credentials live in the formulary
 * (`profiles.env_vars`) — never in `~/.aws/credentials`, never in `process.env`
 * as the source of truth. Callers (weapon code, the MCP launcher script,
 * future skill books) resolve creds through this weapon.
 *
 * Keys read from the formulary:
 *   AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION,
 *   AWS_ACCOUNT_ID (informational), AWS_IAM_USER (informational; the IAM user
 *   the keys belong to — used by rotateAccessKey to know which user to rotate).
 *
 * Process.env is consulted ONLY as a fallback for the MCP launcher's bootstrap
 * case (no DB available yet), matching the precedent in
 * `libs/council/profileEnvVars.js`.
 */

import { database } from "@/libs/council/database";
import { publicTables } from "@/libs/council/publicTables";

const AWS_REGION_DEFAULT = "us-east-2";

export const toc = {
  loadCreds:
    "Read AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY / AWS_REGION from the formulary (profiles.env_vars) for a given userId. Returns { accessKeyId, secretAccessKey, region, accountId, iamUser, source }. Falls back to process.env ONLY when no userId is passed (launcher bootstrap path).",
  checkCredentials:
    "Verify the formulary AWS creds work by calling STS GetCallerIdentity. Returns { ok, arn?, account?, msg? }. Doesn't mutate anything.",
  rotateAccessKey:
    "Rotate the IAM user's access key: create new key via IAM → write to formulary → deactivate the old key. Atomic only at the formulary-write boundary; on partial failure the IAM user may end up with two active keys (recoverable). Returns { ok, newKeyId, deactivatedKeyId, msg? }.",
  deleteAccessKey:
    "Permanently delete a specific IAM access key by id (deactivate-then-DeleteAccessKey), signed with the formulary creds. Use for credential hygiene — e.g. removing a bootstrap key that was exposed (rotateAccessKey only deactivates the prior key; it never deletes). NEVER pass the access key id currently in the formulary (that would lock you out). Returns { ok, deletedKeyId, msg? }.",
  launchVm:
    "Launch an EC2 VM via the EC2 Query API signed with formulary creds (NO aws-sdk, NO MCP/proxy — reuses the in-weapon SigV4 signer). Picks the latest Amazon Linux 2023 x86_64 AMI, creates a key pair (private key saved to ~/.ssh/<name>-key.pem, mode 600), a security group with the requested SSH CIDR, runs one instance with GuildOS tags, then polls until running and returns the public IP. Idempotent on the Name tag. Returns { ok, instanceId, publicIp, state, amiId, sgId, keyName, pemPath, region, existed, msg? }.",
  ensureBudgetAlarm:
    "Cost guard (skill-book deployResource STEP 0): check for the CloudWatch billing alarm 'guildos-budget-monthly' (us-east-1). If absent, create an SNS topic + email subscription to the owner and a $50/mo EstimatedCharges alarm. SNS email needs a one-click confirmation by the user. Returns { ok, existed, created, needsEmailConfirm, msg? }.",
  launchWindowsDesktop:
    "Launch a Windows Server 2022 desktop (RDP) via the EC2 Query API + formulary creds (same in-weapon SigV4 signer as launchVm — no aws-sdk/MCP). Picks the latest Windows_Server-2022-English-Full-Base AMI, creates a key pair (~/.ssh/<name>-key.pem), a security group opening RDP 3389 ONLY to the given rdpCidr (lock to your /32 — never <REDACTED-IP>/0), runs one t3-class instance, waits for running, then polls GetPasswordData and RSA-decrypts the Administrator password with the key pair. Idempotent on the Name tag. Returns { ok, instanceId, publicIp, adminUser:'Administrator', password, passwordReady, pemPath, sgId, region, existed, msg? }. Connect with an RDP client (Windows 'Remote Desktop'/mstsc) to publicIp as Administrator. Optional `userData` (PowerShell in <powershell></powershell>, base64'd for you) runs on first boot — e.g. to auto-install a browser desktop gateway (NICE/Amazon DCV) — and `extraPorts` opens more TCP ports to the same rdpCidr (e.g. 8443 for DCV's web client, so the desktop renders in a browser/CIC).",
  launchFromAmi:
    "Launch one EC2 instance from an EXPLICIT amiId (e.g. a subscribed AWS Marketplace AMI) via the EC2 Query API + formulary creds. Like launchVm but skips the Amazon-Linux AMI lookup — you pass amiId. Creates key pair (~/.ssh/<name>-key.pem) + security group (opens the given ingressPort, default 22, to ingressCidr), runs the instance with GuildOS tags, polls until running. Requires the account to already be subscribed to the Marketplace product (else RunInstances fails OptInRequired — the user must click Subscribe/accept terms in the console first). Returns { ok, instanceId, publicIp, state, keyName, pemPath, sgId, region, existed, msg? }.",
  terminateInstance:
    "Terminate an EC2 instance by id (teardown counterpart to launchVm/launchWindowsDesktop/launchFromAmi), signed with formulary creds via the EC2 Query API. Returns { ok, instanceId, previousState, currentState, msg? }.",
  describeInstance:
    "Diagnose one EC2 instance by instanceId OR Name tag: returns its state, public IP, instance type, and the 2 status checks (systemStatus/instanceStatus — 'ok' when the instance is fully booted and healthy). Read-only. Returns { ok, found, instanceId, state, publicIp, instanceType, systemStatus, instanceStatus, region, msg? }.",
  getWindowsPassword:
    "Fetch + RSA-decrypt the Administrator password for a Windows instance, standalone (for when launchWindowsDesktop's inline poll timed out — the password appears a few min after 'running'). Needs the key pair's .pem (pemPath). Returns { ok, ready, password?, adminUser, msg? }; ready:false means Windows is still initializing — retry.",
};

/**
 * @typedef {{ accessKeyId: string, secretAccessKey: string, region: string, accountId: string|null, iamUser: string|null, source: 'formulary'|'env'|'mixed' }} AwsCreds
 */

/**
 * @param {string|undefined} userId
 * @returns {Promise<AwsCreds|null>}
 */
export async function loadCreds(userId) {
  let formularyVars = null;
  if (userId) {
    try {
      const db = await database.init("service");
      const { data } = await db
        .from(publicTables.profiles)
        .select("env_vars")
        .eq("id", userId)
        .maybeSingle();
      formularyVars = data?.env_vars || null;
    } catch {
      // fall through to env
    }
  }
  const pick = (key) => {
    const fv = formularyVars && typeof formularyVars[key] === "string" ? formularyVars[key].trim() : "";
    if (fv) return { value: fv, src: "formulary" };
    const ev = (process.env[key] || "").trim();
    if (ev) return { value: ev, src: "env" };
    return { value: "", src: null };
  };
  const ak = pick("AWS_ACCESS_KEY_ID");
  const sk = pick("AWS_SECRET_ACCESS_KEY");
  if (!ak.value || !sk.value) return null;
  const region = pick("AWS_REGION").value || AWS_REGION_DEFAULT;
  const account = pick("AWS_ACCOUNT_ID").value || null;
  const iamUser = pick("AWS_IAM_USER").value || null;
  const sources = new Set([ak.src, sk.src].filter(Boolean));
  const source = sources.size === 1 ? [...sources][0] : "mixed";
  return {
    accessKeyId: ak.value,
    secretAccessKey: sk.value,
    region,
    accountId: account,
    iamUser,
    source,
  };
}

/**
 * Verify the formulary creds via STS GetCallerIdentity.
 * @param {{ userId?: string }} input
 */
export async function checkCredentials({ userId } = {}) {
  const creds = await loadCreds(userId);
  if (!creds) return { ok: false, msg: "AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY not in formulary or env" };
  try {
    const ident = await stsGetCallerIdentity(creds);
    return { ok: true, arn: ident.Arn, account: ident.Account, userId: ident.UserId, source: creds.source };
  } catch (e) {
    return { ok: false, msg: e instanceof Error ? e.message : String(e) };
  }
}

/**
 * Rotate the IAM user's access key. Creates a new key, writes both to the
 * formulary, then deactivates the old key.
 *
 * @param {{ userId: string, iamUser?: string }} input
 */
export async function rotateAccessKey({ userId, iamUser } = {}) {
  if (!userId) return { ok: false, msg: "rotateAccessKey requires userId" };
  const creds = await loadCreds(userId);
  if (!creds) return { ok: false, msg: "No current creds in formulary; cannot rotate" };
  const targetUser = (iamUser || creds.iamUser || "").trim();
  if (!targetUser) return { ok: false, msg: "AWS_IAM_USER not set in formulary; cannot determine which IAM user to rotate" };

  // 1. Create new key.
  let newKey;
  try {
    newKey = await iamCreateAccessKey(creds, targetUser);
  } catch (e) {
    return { ok: false, msg: `iam:CreateAccessKey failed: ${e instanceof Error ? e.message : String(e)}` };
  }

  // 2. Write new key to formulary BEFORE deactivating old (so a crash here doesn't lock us out).
  const db = await database.init("service");
  const { data: row } = await db
    .from(publicTables.profiles)
    .select("env_vars")
    .eq("id", userId)
    .maybeSingle();
  const next = {
    ...(row?.env_vars || {}),
    AWS_ACCESS_KEY_ID: newKey.AccessKeyId,
    AWS_SECRET_ACCESS_KEY: newKey.SecretAccessKey,
  };
  const { error: pErr } = await db.from(publicTables.profiles).update({ env_vars: next }).eq("id", userId);
  if (pErr) {
    // The new key is active in IAM but not persisted. Surface so caller can recover manually.
    return {
      ok: false,
      msg: `Formulary write failed after creating new key ${newKey.AccessKeyId}: ${pErr.message}. Manually update formulary or deactivate the new key in IAM.`,
      newKeyId: newKey.AccessKeyId,
    };
  }

  // 3. Deactivate old key using the NEW credentials (proves they work end-to-end).
  try {
    const newCreds = { ...creds, accessKeyId: newKey.AccessKeyId, secretAccessKey: newKey.SecretAccessKey };
    await iamUpdateAccessKey(newCreds, targetUser, creds.accessKeyId, "Inactive");
  } catch (e) {
    return {
      ok: true,
      newKeyId: newKey.AccessKeyId,
      deactivatedKeyId: null,
      msg: `New key persisted, but deactivating old key ${creds.accessKeyId} failed: ${e instanceof Error ? e.message : String(e)}. Deactivate manually in IAM.`,
    };
  }

  return { ok: true, newKeyId: newKey.AccessKeyId, deactivatedKeyId: creds.accessKeyId };
}

/**
 * Permanently delete a specific IAM access key (credential hygiene). Deactivates
 * first (idempotent if already inactive), then DeleteAccessKey. Refuses to delete
 * the key currently in the formulary — deleting your own live key locks you out.
 *
 * @param {{ userId: string, accessKeyId: string, iamUser?: string }} input
 */
export async function deleteAccessKey({ userId, accessKeyId, iamUser } = {}) {
  if (!userId || !accessKeyId) return { ok: false, msg: "deleteAccessKey requires userId and accessKeyId" };
  const creds = await loadCreds(userId);
  if (!creds) return { ok: false, msg: "No creds in formulary; cannot delete key" };
  if (accessKeyId === creds.accessKeyId) {
    return { ok: false, msg: `Refusing to delete ${accessKeyId}: it is the live key in the formulary (would lock you out). Rotate first.` };
  }
  const targetUser = (iamUser || creds.iamUser || "").trim();
  if (!targetUser) return { ok: false, msg: "AWS_IAM_USER not set in formulary; cannot determine which IAM user owns the key" };
  try {
    // Best-effort deactivate (no-op if already Inactive), then delete.
    try {
      await iamUpdateAccessKey(creds, targetUser, accessKeyId, "Inactive");
    } catch {
      // ignore — DeleteAccessKey works on active keys too; this is belt-and-suspenders.
    }
    await iamDeleteAccessKey(creds, targetUser, accessKeyId);
    return { ok: true, deletedKeyId: accessKeyId };
  } catch (e) {
    return { ok: false, msg: e instanceof Error ? e.message : String(e) };
  }
}

// ─── EC2 / CloudWatch / SNS query-protocol helpers ────────────────────────
// All three are AWS "Query" protocol services (form-urlencoded POST, XML
// response) — the SAME shape the STS/IAM signer below already handles. We reuse
// signedAwsRequest rather than pulling in aws-sdk. See engineering_decisions.md
// (2026-06-30): EC2 launch goes through this signer, NOT the MCP proxy, because
// the in-session aws MCP tools can fail to bind at startup and the launch must
// not depend on that transport.

const xmlUnescape = (s) =>
  s.replace(/&lt;/g, "<").replace(/&gt;/g, ">").replace(/&quot;/g, '"').replace(/&#10;/g, "\n").replace(/&amp;/g, "&");

// Decrypt an EC2 Windows GetPasswordData blob. AWS encrypts it with RSA PKCS#1
// v1.5, but modern Node (>=18.19 / 20.11) REJECTS RSA_PKCS1_PADDING for
// privateDecrypt (Marvin/Bleichenbacher mitigation) — it throws
// "RSA_PKCS1_PADDING is no longer supported for private decryption". So we
// decrypt raw (RSA_NO_PADDING) and strip the PKCS#1 v1.5 padding by hand:
// the block is 0x00 0x02 <non-zero random PS> 0x00 <message>.
async function decryptWindowsPassword(keyMaterial, passwordDataB64) {
  const crypto = await import("node:crypto");
  const raw = crypto.privateDecrypt(
    { key: keyMaterial, padding: crypto.constants.RSA_NO_PADDING },
    Buffer.from(passwordDataB64, "base64"),
  );
  let i = raw[0] === 0x00 ? 1 : 0;
  if (raw[i] !== 0x02) throw new Error("unexpected PKCS1 block type: " + raw[0] + "," + raw[1]);
  i++;
  while (i < raw.length && raw[i] !== 0x00) i++;
  if (i >= raw.length) throw new Error("PKCS1 separator not found");
  return raw.slice(i + 1).toString("utf8");
}

async function queryApi(creds, { service, host, region, params }) {
  const body = Object.entries(params)
    .map(([k, v]) => `${encodeURIComponent(k)}=${encodeURIComponent(String(v))}`)
    .join("&");
  const res = await signedAwsRequest({
    creds,
    service,
    host,
    region,
    method: "POST",
    path: "/",
    body,
    contentType: "application/x-www-form-urlencoded; charset=utf-8",
  });
  if (!res.ok) throw new Error(`${service} ${res.status}: ${res.text.slice(0, 600)}`);
  return res.text;
}

/**
 * Launch an EC2 VM signed with formulary creds via the EC2 Query API.
 * @param {{ userId?: string, name: string, instanceType?: string, sshCidr?: string, region?: string }} input
 */
export async function launchVm({ userId, name, instanceType = "t3.micro", sshCidr = "<REDACTED-IP>/0", region } = {}) {
  if (!name) return { ok: false, msg: "launchVm requires a name" };
  const creds = await loadCreds(userId);
  if (!creds) return { ok: false, msg: "No AWS creds in formulary or env" };
  const rgn = region || creds.region || AWS_REGION_DEFAULT;
  const host = `ec2.${rgn}.amazonaws.com`;
  const V = "2016-11-15";
  const ec2 = (params) => queryApi(creds, { service: "ec2", host, region: rgn, params: { Version: V, ...params } });

  try {
    // 1. Idempotency — existing pending/running instance with this Name tag?
    const existXml = await ec2({
      Action: "DescribeInstances",
      "Filter.1.Name": "tag:Name",
      "Filter.1.Value.1": name,
      "Filter.2.Name": "instance-state-name",
      "Filter.2.Value.1": "pending",
      "Filter.2.Value.2": "running",
    });
    const existId = existXml.match(/<instanceId>(.*?)<\/instanceId>/)?.[1];
    if (existId) {
      const ip = existXml.match(/<ipAddress>(.*?)<\/ipAddress>/)?.[1] || null;
      const st = existXml.match(/<instanceState>[\s\S]*?<name>(.*?)<\/name>/)?.[1] || "unknown";
      return { ok: true, existed: true, instanceId: existId, publicIp: ip, state: st, region: rgn, name };
    }

    // 2. Latest Amazon Linux 2023 x86_64 AMI (owned by amazon).
    const imgXml = await ec2({
      Action: "DescribeImages",
      "Owner.1": "amazon",
      "Filter.1.Name": "name",
      "Filter.1.Value.1": "al2023-ami-2023.*-x86_64",
      "Filter.2.Name": "state",
      "Filter.2.Value.1": "available",
      "Filter.3.Name": "architecture",
      "Filter.3.Value.1": "x86_64",
    });
    let amiId = null;
    let best = "";
    for (const item of imgXml.split("<item>")) {
      const id = item.match(/<imageId>(.*?)<\/imageId>/)?.[1];
      const cd = item.match(/<creationDate>(.*?)<\/creationDate>/)?.[1];
      if (id && cd && cd > best) {
        best = cd;
        amiId = id;
      }
    }
    if (!amiId) return { ok: false, msg: "No Amazon Linux 2023 AMI found in region " + rgn };

    // 3. Key pair — private key returned ONCE, save it to ~/.ssh.
    const kpName = `${name}-key`;
    const kpXml = await ec2({ Action: "CreateKeyPair", KeyName: kpName });
    const keyMaterial = xmlUnescape(kpXml.match(/<keyMaterial>([\s\S]*?)<\/keyMaterial>/)?.[1] || "");
    if (!keyMaterial) return { ok: false, msg: "CreateKeyPair returned no key material" };
    const { writeFileSync, mkdirSync } = await import("node:fs");
    const os = await import("node:os");
    const pathMod = await import("node:path");
    const sshDir = pathMod.join(os.homedir(), ".ssh");
    mkdirSync(sshDir, { recursive: true });
    const pemPath = pathMod.join(sshDir, `${kpName}.pem`);
    writeFileSync(pemPath, keyMaterial.endsWith("\n") ? keyMaterial : keyMaterial + "\n", { mode: 0o600 });

    // 4. Security group + SSH ingress (default VPC).
    const sgName = `${name}-sg`;
    const sgXml = await ec2({
      Action: "CreateSecurityGroup",
      GroupName: sgName,
      GroupDescription: `GuildOS ${name} SSH access`,
    });
    const sgId = sgXml.match(/<groupId>(.*?)<\/groupId>/)?.[1];
    if (!sgId) return { ok: false, msg: "CreateSecurityGroup returned no groupId: " + sgXml.slice(0, 300) };
    await ec2({
      Action: "AuthorizeSecurityGroupIngress",
      GroupId: sgId,
      "IpPermissions.1.IpProtocol": "tcp",
      "IpPermissions.1.FromPort": "22",
      "IpPermissions.1.ToPort": "22",
      "IpPermissions.1.IpRanges.1.CidrIp": sshCidr,
    });

    // 5. Run the instance with GuildOS tag convention.
    const runXml = await ec2({
      Action: "RunInstances",
      ImageId: amiId,
      InstanceType: instanceType,
      MinCount: "1",
      MaxCount: "1",
      KeyName: kpName,
      "SecurityGroupId.1": sgId,
      "TagSpecification.1.ResourceType": "instance",
      "TagSpecification.1.Tag.1.Key": "Name",
      "TagSpecification.1.Tag.1.Value": name,
      "TagSpecification.1.Tag.2.Key": "ManagedBy",
      "TagSpecification.1.Tag.2.Value": "guildos",
      "TagSpecification.1.Tag.3.Key": "CreatedBy",
      "TagSpecification.1.Tag.3.Value": "aws.launchVm",
    });
    const instanceId = runXml.match(/<instanceId>(.*?)<\/instanceId>/)?.[1];
    if (!instanceId) return { ok: false, msg: "RunInstances returned no instanceId: " + runXml.slice(0, 400) };

    // 6. Poll until running + public IP assigned.
    let state = "pending";
    let publicIp = null;
    let az = null;
    for (let i = 0; i < 12; i++) {
      const dXml = await ec2({ Action: "DescribeInstances", "InstanceId.1": instanceId });
      state = dXml.match(/<instanceState>[\s\S]*?<name>(.*?)<\/name>/)?.[1] || state;
      publicIp = dXml.match(/<ipAddress>(.*?)<\/ipAddress>/)?.[1] || publicIp;
      az = dXml.match(/<availabilityZone>(.*?)<\/availabilityZone>/)?.[1] || az;
      if (state === "running" && publicIp) break;
      await new Promise((r) => setTimeout(r, 5000));
    }

    return {
      ok: true,
      existed: false,
      instanceId,
      publicIp,
      state,
      az,
      amiId,
      sgId,
      keyName: kpName,
      pemPath,
      instanceType,
      sshCidr,
      region: rgn,
    };
  } catch (e) {
    return { ok: false, msg: e instanceof Error ? e.message : String(e) };
  }
}

/**
 * Launch a Windows Server 2022 desktop (RDP), signed with formulary creds via
 * the EC2 Query API — same signer path as launchVm, no aws-sdk/MCP. Opens RDP
 * 3389 ONLY to rdpCidr, then polls GetPasswordData and RSA-decrypts the
 * Administrator password with the key pair's private key.
 *
 * @param {{ userId?: string, name: string, instanceType?: string, rdpCidr: string, region?: string }} input
 */
export async function launchWindowsDesktop({ userId, name, instanceType = "t3.small", rdpCidr, region, userData, extraPorts = [] } = {}) {
  if (!name) return { ok: false, msg: "launchWindowsDesktop requires a name" };
  if (!rdpCidr) return { ok: false, msg: "launchWindowsDesktop requires rdpCidr (lock RDP to your IP, e.g. <REDACTED-IP>/32 — never <REDACTED-IP>/0)" };
  const creds = await loadCreds(userId);
  if (!creds) return { ok: false, msg: "No AWS creds in formulary or env" };
  const rgn = region || creds.region || AWS_REGION_DEFAULT;
  const host = `ec2.${rgn}.amazonaws.com`;
  const V = "2016-11-15";
  const ec2 = (params) => queryApi(creds, { service: "ec2", host, region: rgn, params: { Version: V, ...params } });

  try {
    // 1. Idempotency on the Name tag.
    const existXml = await ec2({
      Action: "DescribeInstances",
      "Filter.1.Name": "tag:Name",
      "Filter.1.Value.1": name,
      "Filter.2.Name": "instance-state-name",
      "Filter.2.Value.1": "pending",
      "Filter.2.Value.2": "running",
    });
    const existId = existXml.match(/<instanceId>(.*?)<\/instanceId>/)?.[1];
    if (existId) {
      const ip = existXml.match(/<ipAddress>(.*?)<\/ipAddress>/)?.[1] || null;
      const st = existXml.match(/<instanceState>[\s\S]*?<name>(.*?)<\/name>/)?.[1] || "unknown";
      return {
        ok: true, existed: true, instanceId: existId, publicIp: ip, state: st, region: rgn, name,
        adminUser: "Administrator", password: null, passwordReady: false,
        msg: `Instance already exists. The Administrator password is only decryptable from the key pair created at first launch; re-fetch via the EC2 console (Connect → RDP client → Get password) using ~/.ssh/${name}-key.pem.`,
      };
    }

    // 2. Latest Windows Server 2022 Full Base AMI (owned by amazon).
    const imgXml = await ec2({
      Action: "DescribeImages",
      "Owner.1": "amazon",
      "Filter.1.Name": "name",
      "Filter.1.Value.1": "Windows_Server-2022-English-Full-Base-*",
      "Filter.2.Name": "state",
      "Filter.2.Value.1": "available",
    });
    let amiId = null;
    let best = "";
    for (const item of imgXml.split("<item>")) {
      const id = item.match(/<imageId>(.*?)<\/imageId>/)?.[1];
      const cd = item.match(/<creationDate>(.*?)<\/creationDate>/)?.[1];
      if (id && cd && cd > best) {
        best = cd;
        amiId = id;
      }
    }
    if (!amiId) return { ok: false, msg: "No Windows Server 2022 AMI found in region " + rgn };

    // 3. Key pair — private key returned ONCE (needed to decrypt the Windows password).
    const kpName = `${name}-key`;
    const kpXml = await ec2({ Action: "CreateKeyPair", KeyName: kpName });
    const keyMaterial = xmlUnescape(kpXml.match(/<keyMaterial>([\s\S]*?)<\/keyMaterial>/)?.[1] || "");
    if (!keyMaterial) return { ok: false, msg: "CreateKeyPair returned no key material" };
    const { writeFileSync, mkdirSync } = await import("node:fs");
    const os = await import("node:os");
    const pathMod = await import("node:path");
    const sshDir = pathMod.join(os.homedir(), ".ssh");
    mkdirSync(sshDir, { recursive: true });
    const pemPath = pathMod.join(sshDir, `${kpName}.pem`);
    writeFileSync(pemPath, keyMaterial.endsWith("\n") ? keyMaterial : keyMaterial + "\n", { mode: 0o600 });

    // 4. Security group + RDP ingress locked to rdpCidr ONLY (default VPC).
    const sgName = `${name}-sg`;
    const sgXml = await ec2({
      Action: "CreateSecurityGroup",
      GroupName: sgName,
      GroupDescription: `GuildOS ${name} RDP access`,
    });
    const sgId = sgXml.match(/<groupId>(.*?)<\/groupId>/)?.[1];
    if (!sgId) return { ok: false, msg: "CreateSecurityGroup returned no groupId: " + sgXml.slice(0, 300) };
    await ec2({
      Action: "AuthorizeSecurityGroupIngress",
      GroupId: sgId,
      "IpPermissions.1.IpProtocol": "tcp",
      "IpPermissions.1.FromPort": "3389",
      "IpPermissions.1.ToPort": "3389",
      "IpPermissions.1.IpRanges.1.CidrIp": rdpCidr,
    });
    // Extra TCP ports (e.g. 8443 for the DCV web client), locked to the same CIDR.
    for (const port of extraPorts) {
      await ec2({
        Action: "AuthorizeSecurityGroupIngress",
        GroupId: sgId,
        "IpPermissions.1.IpProtocol": "tcp",
        "IpPermissions.1.FromPort": String(port),
        "IpPermissions.1.ToPort": String(port),
        "IpPermissions.1.IpRanges.1.CidrIp": rdpCidr,
      });
    }

    // 5. Run the instance with GuildOS tag convention.
    const runXml = await ec2({
      Action: "RunInstances",
      ImageId: amiId,
      InstanceType: instanceType,
      MinCount: "1",
      MaxCount: "1",
      KeyName: kpName,
      "SecurityGroupId.1": sgId,
      ...(userData ? { UserData: Buffer.from(userData, "utf8").toString("base64") } : {}),
      "TagSpecification.1.ResourceType": "instance",
      "TagSpecification.1.Tag.1.Key": "Name",
      "TagSpecification.1.Tag.1.Value": name,
      "TagSpecification.1.Tag.2.Key": "ManagedBy",
      "TagSpecification.1.Tag.2.Value": "guildos",
      "TagSpecification.1.Tag.3.Key": "CreatedBy",
      "TagSpecification.1.Tag.3.Value": "aws.launchWindowsDesktop",
    });
    const instanceId = runXml.match(/<instanceId>(.*?)<\/instanceId>/)?.[1];
    if (!instanceId) return { ok: false, msg: "RunInstances returned no instanceId: " + runXml.slice(0, 400) };

    // 6. Poll until running + public IP.
    let state = "pending";
    let publicIp = null;
    for (let i = 0; i < 24; i++) {
      const dXml = await ec2({ Action: "DescribeInstances", "InstanceId.1": instanceId });
      state = dXml.match(/<instanceState>[\s\S]*?<name>(.*?)<\/name>/)?.[1] || state;
      publicIp = dXml.match(/<ipAddress>(.*?)<\/ipAddress>/)?.[1] || publicIp;
      if (state === "running" && publicIp) break;
      await new Promise((r) => setTimeout(r, 5000));
    }

    // 7. Poll GetPasswordData (Windows generates it a few min after boot) + RSA-decrypt.
    let password = null;
    for (let i = 0; i < 40; i++) {
      const pXml = await ec2({ Action: "GetPasswordData", InstanceId: instanceId });
      const pd = (pXml.match(/<passwordData>([\s\S]*?)<\/passwordData>/)?.[1] || "").replace(/\s+/g, "");
      if (pd) {
        try {
          password = await decryptWindowsPassword(keyMaterial, pd);
          break;
        } catch {
          // key/ciphertext not aligned yet — keep polling
        }
      }
      await new Promise((r) => setTimeout(r, 15000));
    }

    return {
      ok: true,
      existed: false,
      instanceId,
      publicIp,
      state,
      amiId,
      sgId,
      keyName: kpName,
      pemPath,
      instanceType,
      rdpCidr,
      region: rgn,
      adminUser: "Administrator",
      password,
      passwordReady: !!password,
      msg: password
        ? undefined
        : `Instance is up but the Windows password wasn't ready within the poll window (can take several minutes on small instances). Re-fetch later via the EC2 console (Connect → RDP client → Get password) using ${pemPath}.`,
    };
  } catch (e) {
    return { ok: false, msg: e instanceof Error ? e.message : String(e) };
  }
}

/**
 * Terminate an EC2 instance by id, signed with formulary creds via the EC2
 * Query API. Teardown counterpart to launchVm / launchWindowsDesktop.
 *
 * @param {{ userId?: string, instanceId: string, region?: string }} input
 */
export async function terminateInstance({ userId, instanceId, region } = {}) {
  if (!instanceId) return { ok: false, msg: "terminateInstance requires instanceId" };
  const creds = await loadCreds(userId);
  if (!creds) return { ok: false, msg: "No AWS creds in formulary or env" };
  const rgn = region || creds.region || AWS_REGION_DEFAULT;
  const host = `ec2.${rgn}.amazonaws.com`;
  try {
    const xml = await queryApi(creds, {
      service: "ec2",
      host,
      region: rgn,
      params: { Version: "2016-11-15", Action: "TerminateInstances", "InstanceId.1": instanceId },
    });
    const prev = xml.match(/<previousState>[\s\S]*?<name>(.*?)<\/name>/)?.[1] || null;
    const cur = xml.match(/<currentState>[\s\S]*?<name>(.*?)<\/name>/)?.[1] || null;
    if (!cur) return { ok: false, msg: "TerminateInstances unexpected response: " + xml.slice(0, 300) };
    return { ok: true, instanceId, previousState: prev, currentState: cur };
  } catch (e) {
    return { ok: false, msg: e instanceof Error ? e.message : String(e) };
  }
}

/**
 * Launch one EC2 instance from an EXPLICIT amiId (e.g. a subscribed Marketplace
 * AMI). Like launchVm but skips the Amazon-Linux lookup. The account must
 * already be subscribed to the Marketplace product or RunInstances fails
 * OptInRequired.
 *
 * @param {{ userId?: string, name: string, amiId: string, instanceType?: string, ingressPort?: number, ingressCidr?: string, region?: string }} input
 */
export async function launchFromAmi({ userId, name, amiId, instanceType = "t3.small", ingressPort = 22, ingressCidr = "<REDACTED-IP>/0", region } = {}) {
  if (!name) return { ok: false, msg: "launchFromAmi requires a name" };
  if (!amiId) return { ok: false, msg: "launchFromAmi requires amiId" };
  const creds = await loadCreds(userId);
  if (!creds) return { ok: false, msg: "No AWS creds in formulary or env" };
  const rgn = region || creds.region || AWS_REGION_DEFAULT;
  const host = `ec2.${rgn}.amazonaws.com`;
  const V = "2016-11-15";
  const ec2 = (params) => queryApi(creds, { service: "ec2", host, region: rgn, params: { Version: V, ...params } });
  try {
    // Idempotency on the Name tag.
    const existXml = await ec2({
      Action: "DescribeInstances",
      "Filter.1.Name": "tag:Name",
      "Filter.1.Value.1": name,
      "Filter.2.Name": "instance-state-name",
      "Filter.2.Value.1": "pending",
      "Filter.2.Value.2": "running",
    });
    const existId = existXml.match(/<instanceId>(.*?)<\/instanceId>/)?.[1];
    if (existId) {
      const ip = existXml.match(/<ipAddress>(.*?)<\/ipAddress>/)?.[1] || null;
      const st = existXml.match(/<instanceState>[\s\S]*?<name>(.*?)<\/name>/)?.[1] || "unknown";
      return { ok: true, existed: true, instanceId: existId, publicIp: ip, state: st, region: rgn, name };
    }
    // Key pair.
    const kpName = `${name}-key`;
    const kpXml = await ec2({ Action: "CreateKeyPair", KeyName: kpName });
    const keyMaterial = xmlUnescape(kpXml.match(/<keyMaterial>([\s\S]*?)<\/keyMaterial>/)?.[1] || "");
    if (!keyMaterial) return { ok: false, msg: "CreateKeyPair returned no key material" };
    const { writeFileSync, mkdirSync } = await import("node:fs");
    const os = await import("node:os");
    const pathMod = await import("node:path");
    const sshDir = pathMod.join(os.homedir(), ".ssh");
    mkdirSync(sshDir, { recursive: true });
    const pemPath = pathMod.join(sshDir, `${kpName}.pem`);
    writeFileSync(pemPath, keyMaterial.endsWith("\n") ? keyMaterial : keyMaterial + "\n", { mode: 0o600 });
    // Security group + ingress.
    const sgName = `${name}-sg`;
    const sgXml = await ec2({ Action: "CreateSecurityGroup", GroupName: sgName, GroupDescription: `GuildOS ${name} access` });
    const sgId = sgXml.match(/<groupId>(.*?)<\/groupId>/)?.[1];
    if (!sgId) return { ok: false, msg: "CreateSecurityGroup returned no groupId: " + sgXml.slice(0, 300) };
    await ec2({
      Action: "AuthorizeSecurityGroupIngress",
      GroupId: sgId,
      "IpPermissions.1.IpProtocol": "tcp",
      "IpPermissions.1.FromPort": String(ingressPort),
      "IpPermissions.1.ToPort": String(ingressPort),
      "IpPermissions.1.IpRanges.1.CidrIp": ingressCidr,
    });
    // Run.
    const runXml = await ec2({
      Action: "RunInstances",
      ImageId: amiId,
      InstanceType: instanceType,
      MinCount: "1",
      MaxCount: "1",
      KeyName: kpName,
      "SecurityGroupId.1": sgId,
      "TagSpecification.1.ResourceType": "instance",
      "TagSpecification.1.Tag.1.Key": "Name",
      "TagSpecification.1.Tag.1.Value": name,
      "TagSpecification.1.Tag.2.Key": "ManagedBy",
      "TagSpecification.1.Tag.2.Value": "guildos",
      "TagSpecification.1.Tag.3.Key": "CreatedBy",
      "TagSpecification.1.Tag.3.Value": "aws.launchFromAmi",
    });
    const instanceId = runXml.match(/<instanceId>(.*?)<\/instanceId>/)?.[1];
    if (!instanceId) return { ok: false, msg: "RunInstances returned no instanceId: " + runXml.slice(0, 400) };
    // Poll until running + public IP.
    let state = "pending";
    let publicIp = null;
    for (let i = 0; i < 24; i++) {
      const dXml = await ec2({ Action: "DescribeInstances", "InstanceId.1": instanceId });
      state = dXml.match(/<instanceState>[\s\S]*?<name>(.*?)<\/name>/)?.[1] || state;
      publicIp = dXml.match(/<ipAddress>(.*?)<\/ipAddress>/)?.[1] || publicIp;
      if (state === "running" && publicIp) break;
      await new Promise((r) => setTimeout(r, 5000));
    }
    return { ok: true, existed: false, instanceId, publicIp, state, amiId, sgId, keyName: kpName, pemPath, instanceType, ingressPort, ingressCidr, region: rgn };
  } catch (e) {
    return { ok: false, msg: e instanceof Error ? e.message : String(e) };
  }
}

/**
 * Diagnose one EC2 instance by instanceId OR Name tag. Read-only. Includes the
 * two status checks (DescribeInstanceStatus) so callers can tell 'running' (the
 * hypervisor started it) from 'ok/ok' (fully booted + reachable).
 *
 * @param {{ userId?: string, instanceId?: string, name?: string, region?: string }} input
 */
export async function describeInstance({ userId, instanceId, name, region } = {}) {
  const creds = await loadCreds(userId);
  if (!creds) return { ok: false, msg: "No AWS creds in formulary or env" };
  const rgn = region || creds.region || AWS_REGION_DEFAULT;
  const host = `ec2.${rgn}.amazonaws.com`;
  const V = "2016-11-15";
  const ec2 = (params) => queryApi(creds, { service: "ec2", host, region: rgn, params: { Version: V, ...params } });
  try {
    let params;
    if (instanceId) {
      params = { Action: "DescribeInstances", "InstanceId.1": instanceId };
    } else if (name) {
      params = {
        Action: "DescribeInstances",
        "Filter.1.Name": "tag:Name",
        "Filter.1.Value.1": name,
        "Filter.2.Name": "instance-state-name",
        "Filter.2.Value.1": "pending",
        "Filter.2.Value.2": "running",
        "Filter.2.Value.3": "stopping",
        "Filter.2.Value.4": "stopped",
      };
    } else {
      return { ok: false, msg: "describeInstance requires instanceId or name" };
    }
    const xml = await ec2(params);
    const id = xml.match(/<instanceId>(i-[0-9a-f]+)<\/instanceId>/)?.[1] || null;
    if (!id) return { ok: true, found: false, region: rgn };
    const state = xml.match(/<instanceState>[\s\S]*?<name>(.*?)<\/name>/)?.[1] || null;
    const publicIp = xml.match(/<ipAddress>(.*?)<\/ipAddress>/)?.[1] || null;
    const instanceType = xml.match(/<instanceType>(.*?)<\/instanceType>/)?.[1] || null;
    let systemStatus = null;
    let instanceStatus = null;
    try {
      const sXml = await ec2({ Action: "DescribeInstanceStatus", "InstanceId.1": id });
      systemStatus = sXml.match(/<systemStatus>[\s\S]*?<status>(.*?)<\/status>/)?.[1] || null;
      instanceStatus = sXml.match(/<instanceStatus>[\s\S]*?<status>(.*?)<\/status>/)?.[1] || null;
    } catch {
      // status checks best-effort
    }
    return { ok: true, found: true, instanceId: id, state, publicIp, instanceType, systemStatus, instanceStatus, region: rgn };
  } catch (e) {
    return { ok: false, msg: e instanceof Error ? e.message : String(e) };
  }
}

/**
 * Fetch + RSA-decrypt a Windows instance's Administrator password, standalone
 * (for when launchWindowsDesktop's inline poll timed out). Needs the key pair
 * .pem. ready:false ⇒ Windows still initializing, retry.
 *
 * @param {{ userId?: string, instanceId: string, pemPath: string, region?: string }} input
 */
export async function getWindowsPassword({ userId, instanceId, pemPath, region } = {}) {
  if (!instanceId) return { ok: false, msg: "getWindowsPassword requires instanceId" };
  if (!pemPath) return { ok: false, msg: "getWindowsPassword requires pemPath" };
  const creds = await loadCreds(userId);
  if (!creds) return { ok: false, msg: "No AWS creds in formulary or env" };
  const rgn = region || creds.region || AWS_REGION_DEFAULT;
  const host = `ec2.${rgn}.amazonaws.com`;
  try {
    const { readFileSync } = await import("node:fs");
    const keyMaterial = readFileSync(pemPath, "utf8");
    const xml = await queryApi(creds, {
      service: "ec2",
      host,
      region: rgn,
      params: { Version: "2016-11-15", Action: "GetPasswordData", InstanceId: instanceId },
    });
    const pd = (xml.match(/<passwordData>([\s\S]*?)<\/passwordData>/)?.[1] || "").replace(/\s+/g, "");
    if (!pd) return { ok: true, ready: false, msg: "Password not generated yet (Windows still initializing)." };
    const password = await decryptWindowsPassword(keyMaterial, pd);
    return { ok: true, ready: true, password, adminUser: "Administrator" };
  } catch (e) {
    return { ok: false, msg: e instanceof Error ? e.message : String(e) };
  }
}

/**
 * Cost guard — ensure a CloudWatch billing alarm exists. Best-effort: creating
 * the SNS email subscription requires a user click to confirm.
 * @param {{ userId?: string, monthlyUsd?: number, email?: string }} input
 */
export async function ensureBudgetAlarm({ userId, monthlyUsd = 50, email = process.env.GUILDOS_OWNER_EMAIL } = {}) {
  const creds = await loadCreds(userId);
  if (!creds) return { ok: false, msg: "No AWS creds in formulary or env" };
  if (!email) return { ok: false, msg: "No alert email — pass { email } or set GUILDOS_OWNER_EMAIL in .env.local" };
  // Billing metrics + their alarms live ONLY in us-east-1.
  const cwHost = "monitoring.us-east-1.amazonaws.com";
  const cw = (params) => queryApi(creds, { service: "monitoring", host: cwHost, region: "us-east-1", params: { Version: "2010-08-01", ...params } });
  const alarmName = "guildos-budget-monthly";
  try {
    const existing = await cw({ Action: "DescribeAlarms", "AlarmNames.member.1": alarmName });
    if (existing.match(/<AlarmName>guildos-budget-monthly<\/AlarmName>/)) {
      return { ok: true, existed: true, created: false, needsEmailConfirm: false };
    }
    // Create SNS topic + email subscription.
    const snsHost = "sns.us-east-1.amazonaws.com";
    const sns = (params) => queryApi(creds, { service: "sns", host: snsHost, region: "us-east-1", params: { Version: "2010-03-31", ...params } });
    const topicXml = await sns({ Action: "CreateTopic", Name: "guildos-budget-alerts" });
    const topicArn = topicXml.match(/<TopicArn>(.*?)<\/TopicArn>/)?.[1];
    if (topicArn) {
      await sns({ Action: "Subscribe", TopicArn: topicArn, Protocol: "email", Endpoint: email });
      await cw({
        Action: "PutMetricAlarm",
        AlarmName: alarmName,
        AlarmDescription: "GuildOS monthly estimated charges guard",
        Namespace: "AWS/Billing",
        MetricName: "EstimatedCharges",
        "Dimensions.member.1.Name": "Currency",
        "Dimensions.member.1.Value": "USD",
        Statistic: "Maximum",
        Period: "21600",
        EvaluationPeriods: "1",
        Threshold: String(monthlyUsd),
        ComparisonOperator: "GreaterThanThreshold",
        "AlarmActions.member.1": topicArn,
        ActionsEnabled: "true",
      });
      return { ok: true, existed: false, created: true, needsEmailConfirm: true, topicArn };
    }
    return { ok: false, msg: "Could not create SNS topic for budget alarm" };
  } catch (e) {
    return { ok: false, msg: e instanceof Error ? e.message : String(e) };
  }
}

// ─── SigV4 + low-level AWS request helpers ────────────────────────────────
// Keep this self-contained — no aws-sdk dep. Services: STS, IAM (global,
// us-east-1) plus EC2 / CloudWatch / SNS (region-scoped query protocol, via
// queryApi above).

async function stsGetCallerIdentity(creds) {
  const body = "Action=GetCallerIdentity&Version=2011-06-15";
  const res = await signedAwsRequest({
    creds,
    service: "sts",
    host: "sts.amazonaws.com",
    region: "us-east-1",
    method: "POST",
    path: "/",
    body,
    contentType: "application/x-www-form-urlencoded; charset=utf-8",
  });
  if (!res.ok) throw new Error(`STS ${res.status}: ${res.text.slice(0, 500)}`);
  const m = res.text.match(/<Arn>(.*?)<\/Arn>.*?<UserId>(.*?)<\/UserId>.*?<Account>(.*?)<\/Account>/s);
  if (!m) throw new Error(`STS response did not match expected shape: ${res.text.slice(0, 200)}`);
  return { Arn: m[1], UserId: m[2], Account: m[3] };
}

async function iamCreateAccessKey(creds, userName) {
  const body = `Action=CreateAccessKey&UserName=${encodeURIComponent(userName)}&Version=2010-05-08`;
  const res = await signedAwsRequest({
    creds,
    service: "iam",
    host: "iam.amazonaws.com",
    region: "us-east-1",
    method: "POST",
    path: "/",
    body,
    contentType: "application/x-www-form-urlencoded; charset=utf-8",
  });
  if (!res.ok) throw new Error(`IAM ${res.status}: ${res.text.slice(0, 500)}`);
  const id = res.text.match(/<AccessKeyId>(.*?)<\/AccessKeyId>/)?.[1];
  const secret = res.text.match(/<SecretAccessKey>(.*?)<\/SecretAccessKey>/)?.[1];
  if (!id || !secret) throw new Error(`IAM CreateAccessKey unexpected shape: ${res.text.slice(0, 200)}`);
  return { AccessKeyId: id, SecretAccessKey: secret };
}

async function iamUpdateAccessKey(creds, userName, accessKeyId, status) {
  const body = `Action=UpdateAccessKey&UserName=${encodeURIComponent(userName)}&AccessKeyId=${encodeURIComponent(accessKeyId)}&Status=${encodeURIComponent(status)}&Version=2010-05-08`;
  const res = await signedAwsRequest({
    creds,
    service: "iam",
    host: "iam.amazonaws.com",
    region: "us-east-1",
    method: "POST",
    path: "/",
    body,
    contentType: "application/x-www-form-urlencoded; charset=utf-8",
  });
  if (!res.ok) throw new Error(`IAM UpdateAccessKey ${res.status}: ${res.text.slice(0, 500)}`);
  return true;
}

async function iamDeleteAccessKey(creds, userName, accessKeyId) {
  const body = `Action=DeleteAccessKey&UserName=${encodeURIComponent(userName)}&AccessKeyId=${encodeURIComponent(accessKeyId)}&Version=2010-05-08`;
  const res = await signedAwsRequest({
    creds,
    service: "iam",
    host: "iam.amazonaws.com",
    region: "us-east-1",
    method: "POST",
    path: "/",
    body,
    contentType: "application/x-www-form-urlencoded; charset=utf-8",
  });
  if (!res.ok) throw new Error(`IAM DeleteAccessKey ${res.status}: ${res.text.slice(0, 500)}`);
  return true;
}

/**
 * Minimal SigV4 signer for AWS query-protocol services (STS, IAM).
 * @param {{
 *   creds: AwsCreds,
 *   service: string,
 *   host: string,
 *   region: string,
 *   method: "GET"|"POST",
 *   path: string,
 *   body: string,
 *   contentType: string,
 * }} req
 */
async function signedAwsRequest({ creds, service, host, region, method, path, body, contentType }) {
  const crypto = await import("node:crypto");
  const now = new Date();
  const amzDate = now.toISOString().replace(/[:-]|\.\d{3}/g, "");
  const dateStamp = amzDate.slice(0, 8);
  const payloadHash = crypto.createHash("sha256").update(body, "utf8").digest("hex");

  const canonicalHeaders =
    `content-type:${contentType}\n` +
    `host:${host}\n` +
    `x-amz-date:${amzDate}\n`;
  const signedHeaders = "content-type;host;x-amz-date";
  const canonicalRequest = [method, path, "", canonicalHeaders, signedHeaders, payloadHash].join("\n");
  const algo = "AWS4-HMAC-SHA256";
  const scope = `${dateStamp}/${region}/${service}/aws4_request`;
  const stringToSign = [
    algo,
    amzDate,
    scope,
    crypto.createHash("sha256").update(canonicalRequest, "utf8").digest("hex"),
  ].join("\n");

  const kDate = crypto.createHmac("sha256", `AWS4${creds.secretAccessKey}`).update(dateStamp).digest();
  const kRegion = crypto.createHmac("sha256", kDate).update(region).digest();
  const kService = crypto.createHmac("sha256", kRegion).update(service).digest();
  const kSigning = crypto.createHmac("sha256", kService).update("aws4_request").digest();
  const signature = crypto.createHmac("sha256", kSigning).update(stringToSign, "utf8").digest("hex");

  const authorization = `${algo} Credential=${creds.accessKeyId}/${scope}, SignedHeaders=${signedHeaders}, Signature=${signature}`;

  const res = await fetch(`https://${host}${path}`, {
    method,
    headers: {
      "content-type": contentType,
      "x-amz-date": amzDate,
      Authorization: authorization,
    },
    body,
  });
  const text = await res.text();
  return { ok: res.ok, status: res.status, text };
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/aws/worker-provision.ps1 -->
```powershell
# GuildOS worker-VM provisioning bootstrap (Windows Server 2022).
#
# Turns a bare EC2 Windows instance into a full GuildOS worker + team desktop.
# Idempotent: safe to re-run. Runs via EC2 user-data on first boot OR via SSM
# RunCommand on an existing instance. Codifies the steps proven manually
# 2026-07-03 (see libs/skill_book/awsVm) plus the worker-only gap tools.
#
# Secrets are NOT in this file. It pulls the GuildOS env from SSM Parameter
# Store (SecureString) at run time — see $EnvParamName. That is the
# secrets-at-launch pattern: the AMI/script stay reusable and safe to share,
# each launch injects the env from a KMS-encrypted param the instance role can read.
#
# Prereqs on the instance role (instance profile): AmazonSSMManagedInstanceCore
# (SSM + Parameter Store read) and, for auto-stop, ec2:StopInstances on self.

param(
  [string]$Branch      = "staging",
  [string]$RepoUrl     = "https://github.com/boster00/GuildOS.git",
  [string]$RepoDir     = "C:\GuildOS",
  [string]$EnvParamName = "/guildos/worker/env-local",   # SSM SecureString holding the full .env.local
  [string]$GuildName   = "",                              # per-worker LOCAL_GUILD_NAME; empty = cron stays OFF
  [int]$IdleStopMinutes = 30                              # auto-stop after this many idle minutes; 0 = never
)

$ErrorActionPreference = "Stop"
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
function Log($m) { Write-Output ("[provision] " + (Get-Date -Format 'HH:mm:ss') + " " + $m) }
function Refresh-Path { $env:Path = [Environment]::GetEnvironmentVariable('Path','Machine') + ';' + [Environment]::GetEnvironmentVariable('Path','User') }

# --- 1. Chocolatey ---
if (-not (Get-Command choco.exe -ErrorAction SilentlyContinue)) {
  Log "installing Chocolatey"
  Set-ExecutionPolicy Bypass -Scope Process -Force
  iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
}
$choco = "C:\ProgramData\chocolatey\bin\choco.exe"

# --- 2. Core + worker tooling via choco ---
Log "installing nodejs-lts git googlechrome ffmpeg gh"
& $choco install nodejs-lts git googlechrome ffmpeg gh -y --no-progress --ignore-checksums
Refresh-Path

# --- 3. uv/uvx (for the aws MCP proxy) via the official installer ---
if (-not (Get-Command uv -ErrorAction SilentlyContinue)) {
  Log "installing uv"
  powershell -NoProfile -ExecutionPolicy Bypass -Command "irm https://astral.sh/uv/install.ps1 | iex"
  Refresh-Path
}

# --- 4. PowerShell 7 as the OpenSSH DefaultShell (matches the manual setup) ---
if (-not (Test-Path "C:\Program Files\PowerShell\7\pwsh.exe")) {
  Log "installing PowerShell 7"
  & $choco install powershell-core -y --no-progress
}
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "C:\Program Files\PowerShell\7\pwsh.exe" -PropertyType String -Force | Out-Null
Set-Content -Path "C:\Program Files\PowerShell\7\profile.ps1" -Value "function uptime { Get-Uptime }" -Encoding ascii

# --- 5. GuildOS repo + deps ---
$npm = "C:\Program Files\nodejs\npm.cmd"
$git = "C:\Program Files\Git\cmd\git.exe"
if (-not (Test-Path "$RepoDir\.git")) {
  Log "cloning GuildOS ($Branch)"
  & $git clone --branch $Branch --depth 1 $RepoUrl $RepoDir
} else {
  & $git -C $RepoDir pull --ff-only
}
Log "npm install"
Push-Location $RepoDir
& $npm install --no-audit --no-fund
Log "installing Playwright browsers (leafBrowser weapon)"
& $npm exec -- playwright install chromium
Pop-Location

# --- 6. Claude Code CLI (worker agent runtime) ---
Log "installing Claude Code CLI"
& $npm install -g @anthropic-ai/claude-code
Refresh-Path

# --- 7. Secrets at launch: pull .env.local from SSM Parameter Store ---
Log "fetching env from SSM $EnvParamName"
$region = (Invoke-RestMethod -Uri "http://<REDACTED-IP>/latest/meta-data/placement/region" -TimeoutSec 5)
try {
  $envText = & $choco --version *> $null; $envText = $null
  $awsCli = (Get-Command aws.exe -ErrorAction SilentlyContinue)
  if ($awsCli) {
    $envText = & aws ssm get-parameter --name $EnvParamName --with-decryption --region $region --query "Parameter.Value" --output text
  } else {
    # SSM Agent is present even without the CLI; use the instance-role creds via the metadata service through a tiny .NET SSM call fallback is complex — prefer installing awscli:
    & $choco install awscli -y --no-progress; Refresh-Path
    $envText = & aws ssm get-parameter --name $EnvParamName --with-decryption --region $region --query "Parameter.Value" --output text
  }
  if ($envText) { Set-Content -Path "$RepoDir\.env.local" -Value $envText -Encoding utf8; Log ".env.local written from SSM" }
  else { Log "WARN: SSM param empty or missing — .env.local NOT written" }
} catch { Log ("WARN: could not fetch env from SSM: " + $_.Exception.Message) }

# --- 8. Per-worker identity + cron policy ---
# Cron/questTick stay OFF unless a distinct GuildName is supplied (else workers collide on the quest bus).
$cronActive = if ($GuildName) { 'true' } else { 'false' }

# --- 9. Durable app service (Scheduled Task, runs dev:app; cron gated by GuildName) ---
$runDev = @"
`$env:Path = [Environment]::GetEnvironmentVariable('Path','Machine') + ';' + [Environment]::GetEnvironmentVariable('Path','User')
`$env:CRON_ACTIVE = '$cronActive'
`$env:GUILDOS_DISABLE_QUESTTICK = '$(if ($GuildName) { "false" } else { "true" })'
`$env:LOCAL_GUILD_NAME = '$GuildName'
Set-Location $RepoDir
& "C:\Program Files\nodejs\npm.cmd" run $(if ($GuildName) { 'dev' } else { 'dev:app' }) *>> $RepoDir\dev.log
"@
Set-Content -Path "$RepoDir\run-dev.ps1" -Value $runDev -Encoding ascii
$act = New-ScheduledTaskAction -Execute "C:\Program Files\PowerShell\7\pwsh.exe" -Argument "-NoProfile -ExecutionPolicy Bypass -File $RepoDir\run-dev.ps1"
$trg = New-ScheduledTaskTrigger -AtStartup
$prn = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount -RunLevel Highest
$set = New-ScheduledTaskSettingsSet -RestartCount 3 -RestartInterval (New-TimeSpan -Minutes 1) -ExecutionTimeLimit ([TimeSpan]::Zero)
Register-ScheduledTask -TaskName "GuildOS-dev" -Action $act -Trigger $trg -Principal $prn -Settings $set -Force | Out-Null
Start-ScheduledTask -TaskName "GuildOS-dev"

# --- 10. Auto-stop-when-idle (needs ec2:StopInstances on the instance role) ---
if ($IdleStopMinutes -gt 0) {
  $idleScript = @"
`$id = Invoke-RestMethod http://<REDACTED-IP>/latest/meta-data/instance-id -TimeoutSec 5
`$region = Invoke-RestMethod http://<REDACTED-IP>/latest/meta-data/placement/region -TimeoutSec 5
# idle = no active RDP/DCV session AND low CPU
`$sessions = (quser 2>`$null | Select-Object -Skip 1 | Measure-Object).Count
`$cpu = (Get-CimInstance Win32_Processor | Measure-Object -Property LoadPercentage -Average).Average
if (`$sessions -eq 0 -and `$cpu -lt 10) {
  `$flag = 'C:\GuildOS\.idle-since'
  if (-not (Test-Path `$flag)) { Set-Content `$flag (Get-Date).Ticks }
  `$since = [datetime]::new([long](Get-Content `$flag))
  if (((Get-Date) - `$since).TotalMinutes -ge $IdleStopMinutes) { & aws ec2 stop-instances --instance-ids `$id --region `$region }
} else { Remove-Item 'C:\GuildOS\.idle-since' -ErrorAction SilentlyContinue }
"@
  Set-Content -Path "$RepoDir\auto-stop-idle.ps1" -Value $idleScript -Encoding ascii
  $sa = New-ScheduledTaskAction -Execute "C:\Program Files\PowerShell\7\pwsh.exe" -Argument "-NoProfile -ExecutionPolicy Bypass -File $RepoDir\auto-stop-idle.ps1"
  $st = New-ScheduledTaskTrigger -Once -At (Get-Date).AddMinutes(5) -RepetitionInterval (New-TimeSpan -Minutes 5)
  Register-ScheduledTask -TaskName "GuildOS-auto-stop" -Action $sa -Trigger $st -Principal $prn -Settings $set -Force | Out-Null
  Log "auto-stop scheduled: idle > $IdleStopMinutes min (requires ec2:StopInstances on the instance role)"
}

Log "provision complete"
```
<!-- /GUILDOS-FILE -->
