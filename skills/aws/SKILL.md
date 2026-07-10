# aws

<!-- GUILDOS-SKILL name="aws" -->

<!-- GUILDOS-DESC -->
Access the AWS console via Claude-in-Chrome and configure MCP server access to AWS so future Claude Code sessions can hit AWS APIs through MCP instead of CIC clicks. Covers opening/navigating the console, resolving + rotating formulary-stored AWS credentials, installing and verifying the AWS MCP wiring, and deploying/tagging resources with GuildOS discipline (list-before-create, tag convention, region default, budget cost-guard).
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/aws/SKILL.md -->
```markdown
---
name: aws
description: >-
  Access the AWS console via Claude-in-Chrome and configure MCP server access to
  AWS so future Claude Code sessions can hit AWS APIs through MCP instead of CIC
  clicks. Covers opening/navigating the console, resolving + rotating
  formulary-stored AWS credentials, installing and verifying the AWS MCP wiring,
  and deploying/tagging resources with GuildOS discipline (list-before-create,
  tag convention, region default, budget cost-guard).
---

# AWS

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/aws/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto aws <action>` (or read the source directly).

## Actions

- **deployResource** — Create or update an AWS resource via the AWS MCP server with GuildOS discipline baked in: list-before-create idempotency, tag convention, region default, and a cost-guard pre-check. ALL deployments — S3 bucket, EC2 instance, Lambda, ECS task, RDS, anything — go through this action's pattern, not raw `call_aws` improvisation.
- **installMcpServer** — Re-bootstrap the AWS MCP wiring on a fresh host. The `aws` MCP server is already live on this checkout (AWS-hosted `mcp-proxy-for-aws` launched by the formulary launcher) — this action is only needed when cloning to a new machine or after the wiring is torn down. Day-to-day, the server is just connected; you don't re-install it.
- **navigateConsole** — Drive the AWS console via CIC to read state (resource lists, settings, billing widgets) that the MCP server doesn't expose or hasn't been authorized for yet.
- **openConsole** — Open the AWS Management Console in CIC and confirm the signed-in IAM identity + active region before any account-touching work.
- **retrieveCreds** — Resolve the current AWS credentials for a given userId from the formulary via the aws weapon. Use this from ANY skill-book action or weapon that needs to make an authenticated AWS call — do NOT read ~/.aws/credentials, do NOT read process.env directly, do NOT hardcode in .mcp.json.
- **rotateCreds** — Rotate the IAM user's access key via the weapon — agent-driven, no console clicks. Creates a new key, writes it to the formulary, then deactivates the old key using the NEW key (which proves end-to-end the new credentials work).
- **verifyMcpAccess** — Re-verify that the AWS MCP server actually talks to AWS — run after a creds rotation, a fresh-host re-bootstrap, or any 'is it still wired?' doubt. Round-trips one read-only call and reconciles the answer against the CIC console identity. Never claim 'configured' on a bare /mcp listing.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/aws/index.js -->
```js
/**
 * AWS skill book — access the AWS console via Claude-in-Chrome and operate the
 * (now live) `aws` MCP server so sessions hit AWS APIs through MCP instead of
 * CIC clicks.
 *
 * Account context (switched 2026-06-30 off the personal account 759802535743 to
 * the company account — see engineering_decisions.md):
 *   - Account: 325698037467 (bosterbio)
 *   - Console root identity: <REDACTED-EMAIL> (root user — DO NOT create access keys here)
 *   - IAM user for programmatic access: `mcp-server` (AdministratorAccess)
 *     Arn = arn:aws:iam::325698037467:user/mcp-server
 *   - Default region: us-east-2 (Ohio)
 *
 * How it's wired (already live — this is steady state, not a to-do):
 *   `.mcp.json` runs `scripts/aws-mcp-launch.mjs` (NOT uvx directly). The
 *   launcher reads SST_OWNER_USER_ID from .env.local, calls
 *   `weapon/aws.loadCreds(userId)` to fetch AWS_* from the formulary, strips
 *   any AWS_* leaking from the parent env, and spawns the AWS-hosted
 *   `mcp-proxy-for-aws` (stdio shim → https://aws-mcp.us-east-1.api.aws/mcp)
 *   with the formulary creds injected as child env. Tool names: `mcp__aws__*`.
 *   See `installMcpServer` for the re-bootstrap stub (fresh host / creds
 *   rotation) and `verifyMcpAccess` for the re-verify command.
 *
 * LOCKED RULE (2026-05-28): AWS credentials live in the FORMULARY
 * (`profiles.env_vars`) — never in `~/.aws/credentials`, never as static
 * literals in `.mcp.json`, never as long-lived env on disk. Retrieval
 * goes through `weapon/aws` (loadCreds / checkCredentials / rotateAccessKey).
 * The launcher injects them into the proxy subprocess as
 * AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY / AWS_REGION env vars (boto3 picks
 * them up via the env credential provider). Static ~/.aws/credentials was
 * deleted as part of wiring; any future regression that recreates it should be
 * treated as a violation of this rule.
 *
 * Formulary keys (set via the council-hall/formulary UI or by the weapon's
 * rotateAccessKey action):
 *   AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION,
 *   AWS_ACCOUNT_ID (informational), AWS_IAM_USER (informational; required
 *   for rotateAccessKey).
 */

export const skillBook = {
  id: "aws",
  title: "AWS",
  description:
    "Access the AWS console via Claude-in-Chrome and configure MCP server access to AWS so future Claude Code sessions can hit AWS APIs through MCP instead of CIC clicks.",
  steps: [],
  toc: {
    openConsole: {
      description:
        "Open the AWS Management Console in CIC and confirm the signed-in IAM identity + active region before any account-touching work.",
      howTo: [
        "Preflight: run `browsercontrol.validateCIC` first. The console is a tier-'read' app in computer-use (browser), so CIC is the ONLY way you can drive it; a failed CIC = a hard STOP, not a workaround.",
        "Open a NEW tab in the MCP tab group (`tabs_create_mcp`) — never reuse a previous tab. Navigate to https://us-east-2.console.aws.amazon.com/console/home?region=us-east-2 (change region in the URL path AND the `region=` query param together — they must match or the page rewrites the URL and you lose your bearings).",
        "Verify sign-in: `read_page` filter=interactive and look for a button labelled with the IAM user/role short name in the top-right (account menu). It should read 'bosterbio (325698037467)'. If you instead see the AWS sign-in page → STOP and ask the user to sign in; never type credentials.",
        "Verify region: the region combobox in the top-right shows the current region (e.g. 'Regions (United States (Ohio))' = us-east-2). Mismatched region is the #1 source of 'where did my resource go?' errors — pin it explicitly in the account context before doing anything.",
        "Read the account/identity context once and pass it back as `{ iamIdentityName, region, accountAlias? }`. Subsequent steps in the same objective should reuse the same tab — only create a new tab when the objective changes.",
        "Caveats: console pages are JS-heavy and lazy-load widgets. Prefer `get_page_text` on article regions and `read_page` filter=interactive for buttons/links; raw `read_page filter=all` blows past the 50k char cap on the console home.",
      ].join("\n"),
      input: { region: "string — AWS region code, e.g. 'us-east-2'. Defaults to 'us-east-2'." },
      output: {
        tabId: "number — CIC tab to reuse for subsequent steps",
        iamIdentityName: "string — display name on the account menu button",
        region: "string — region code currently active in the console",
      },
    },

    retrieveCreds: {
      description:
        "Resolve the current AWS credentials for a given userId from the formulary via the aws weapon. Use this from ANY skill-book action or weapon that needs to make an authenticated AWS call — do NOT read ~/.aws/credentials, do NOT read process.env directly, do NOT hardcode in .mcp.json.",
      howTo: [
        "Import the weapon: `import { loadCreds, checkCredentials } from '@/libs/weapon/aws';` (server context only — the weapon uses the Supabase service role).",
        "Call `loadCreds(userId)` → returns `{ accessKeyId, secretAccessKey, region, accountId, iamUser, source }` or `null` if no creds are configured. `source` is `'formulary'` when the formulary holds the keys (expected steady state), `'env'` only as a launcher-bootstrap fallback (e.g. running outside a userId context).",
        "Before any mutating AWS call, optionally `await checkCredentials({ userId })` — round-trips STS GetCallerIdentity using the formulary creds and confirms the Arn matches `AWS_IAM_USER`. Cheaper than discovering a stale key mid-operation.",
        "For long-running processes / subprocesses (the MCP launcher is the canonical example), pass the creds to the child as env vars (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`) — do NOT write them to ~/.aws/credentials or any disk-persisted file. Strip any inherited `AWS_*` from the parent environment first so a stale local profile cannot leak through.",
        "Do NOT set `AWS_PROFILE` when running mcp-proxy-for-aws or boto3-based servers — empty string is interpreted as the literal profile `()` and errors out. Just supply the key + secret + region.",
      ].join("\n"),
      input: { userId: "string — the SST owner user id (or any GuildOS user id whose formulary holds AWS_*)" },
      output: { accessKeyId: "string", secretAccessKey: "string", region: "string", accountId: "string|null", iamUser: "string|null", source: "'formulary'|'env'|'mixed'" },
    },

    rotateCreds: {
      description:
        "Rotate the IAM user's access key via the weapon — agent-driven, no console clicks. Creates a new key, writes it to the formulary, then deactivates the old key using the NEW key (which proves end-to-end the new credentials work).",
      howTo: [
        "Preconditions: the formulary already has working AWS_* creds AND AWS_IAM_USER set (so the weapon knows which IAM user owns the keys).",
        "Call `await rotateAccessKey({ userId })` — the weapon performs (1) IAM CreateAccessKey, (2) formulary write, (3) IAM UpdateAccessKey old→Inactive. Step 2 happens BEFORE step 3 so a crash mid-rotation leaves a recoverable state (new key live + persisted, old key still active — re-run the rotation idempotently).",
        "Run this whenever a credential has been exposed (e.g. pasted into a chat transcript, leaked in a log, or a routine rotation cadence). Per the formulary discipline: never just `IAM Console → Create access key` by hand — the rotation must go through the weapon so the formulary stays the source of truth in lock-step.",
        "After rotation: any long-running MCP launcher subprocess holds the OLD key in its env until restart. Restart Claude Code (/mcp or quit-relaunch) so the launcher re-reads the formulary and the AWS MCP server picks up the new key.",
        "Per `sage.askSage` success-claim gate: don't report `rotated` until `checkCredentials({ userId })` returns `ok:true` AND `arn` matches the pre-rotation Arn.",
      ].join("\n"),
      input: { userId: "string", iamUser: "string — optional override; defaults to AWS_IAM_USER from formulary" },
      output: { ok: "boolean", newKeyId: "string", deactivatedKeyId: "string|null", msg: "string|undefined" },
    },

    installMcpServer: {
      description:
        "Re-bootstrap the AWS MCP wiring on a fresh host. The `aws` MCP server is already live on this checkout (AWS-hosted `mcp-proxy-for-aws` launched by the formulary launcher) — this action is only needed when cloning to a new machine or after the wiring is torn down. Day-to-day, the server is just connected; you don't re-install it.",
      howTo: [
        "How it's wired (steady state): `.mcp.json` runs the launcher, NOT uvx directly, so creds stay in the formulary and out of `.mcp.json`:",
        "    {\"mcpServers\":{\"aws\":{\"command\":\"node\",\"args\":[\"scripts/aws-mcp-launch.mjs\"],\"env\":{}}}}",
        "  The launcher (`scripts/aws-mcp-launch.mjs`) reads SST_OWNER_USER_ID from .env.local, calls `weapon/aws.loadCreds(userId)` to fetch AWS_* from the formulary, strips any AWS_* leaking from the parent env, and spawns the `mcp-proxy-for-aws` stdio shim (→ https://aws-mcp.us-east-1.api.aws/mcp) with the formulary creds injected as child env. The proxy only ever sees env-var creds — no `~/.aws/` involvement, and never `AWS_PROFILE` (boto3 errors on empty `AWS_PROFILE`). Per-host overrides live as env in the launcher (`AWS_MCP_UVX`, `AWS_MCP_PYTHON`, `AWS_MCP_ENDPOINT`); read the launcher source if a host needs them.",
        "RE-BOOTSTRAP ON A FRESH HOST (e.g. carbon pulling this repo from GitHub):",
        "    1. Install uv/uvx (`uvx --version` to check; install per https://docs.astral.sh/uv/ for the platform).",
        "    2. `git pull` the repo (this skill book + scripts/aws-mcp-launch.mjs + libs/weapon/aws/ must all be present at the same SHA).",
        "    3. Populate `.env.local` with NEXT_PUBLIC_SUPABASE_URL, SUPABASE_SECRETE_KEY, SST_OWNER_USER_ID (the GuildOS owner whose formulary holds the AWS_* keys — currently 4b2ae469-a474-43f0-907a-eec881413020). These secrets are NOT in the repo; they arrive through whatever secret-sync channel the host already uses. Do NOT auto-push Supabase secrets to new hosts — that's an audience-expansion action requiring explicit user approval.",
        "    4. Create `.mcp.json` at repo root with the block above (this file is .gitignored).",
        "    5. Smoke-test the launcher BEFORE relying on it (see `verifyMcpAccess`), then restart Claude Code so `.mcp.json` is picked up; `/mcp` should list `aws` as connected.",
        "Reversal: delete the `aws` block from `.mcp.json` and restart to fully remove the server.",
      ].join("\n"),
      input: {
        region: "string — AWS region for the AWS_REGION env, default 'us-east-2'",
      },
      output: {
        configFile: "string — absolute path to the mcp.json that was edited",
        prereqGaps: "string[] — list of prereqs that were missing and were either fixed or escalated",
      },
    },

    verifyMcpAccess: {
      description:
        "Re-verify that the AWS MCP server actually talks to AWS — run after a creds rotation, a fresh-host re-bootstrap, or any 'is it still wired?' doubt. Round-trips one read-only call and reconciles the answer against the CIC console identity. Never claim 'configured' on a bare /mcp listing.",
      howTo: [
        "Launcher smoke-test (no Claude Code restart needed): pipe a JSON-RPC `initialize` into the launcher and confirm the capabilities response. Windows, one line:",
        "  echo {\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"initialize\",\"params\":{\"protocolVersion\":\"2024-11-05\",\"capabilities\":{},\"clientInfo\":{\"name\":\"verify\",\"version\":\"0.0\"}}} | node --env-file=.env.local scripts/aws-mcp-launch.mjs",
        "  Expected (verified 2026-05-28): {\"jsonrpc\":\"2.0\",\"id\":1,\"result\":{\"protocolVersion\":\"2024-11-05\",\"capabilities\":{\"tools\":{\"listChanged\":false}},\"serverInfo\":{\"name\":\"MCP Proxy for AWS\",\"version\":\"1.5.0\"}, ...}}",
        "  An auth error here means the formulary creds aren't resolving — recheck the formulary AWS_* keys and `weapon/aws.checkCredentials({ userId })`. After a creds rotation, restart Claude Code so the launcher subprocess re-reads the new key.",
        "Through the live server: `/mcp` should list `aws` connected. Issue ONE read-only call — `mcp__aws__aws___call_aws` with the CLI string `aws sts get-caller-identity` (cheapest authenticated round-trip).",
        "Cross-check the returned `Arn` against the IAM identity name read off the console in `openConsole` — they MUST resolve to the same principal. If they differ, the credential profile and the browser session are two different identities; surface that before any mutation, since the agent operates as the CLI principal, not the console-logged-in one.",
        "Also call one harmless region-scoped read (e.g. `aws ec2 describe-regions`) to confirm region/network plumbing.",
        "Per `sage.askSage` success-claim gate: do NOT report 'MCP access verified' until BOTH calls round-trip cleanly AND the principal matches the console identity. A bare 'server appears in /mcp' is not verification.",
      ].join("\n"),
      input: { serverName: "string — 'aws' (hosted)" },
      output: {
        principalArn: "string — the Arn returned by sts get-caller-identity",
        matchesConsoleIdentity: "boolean — true iff principal resolves to the same identity as openConsole.iamIdentityName",
        regionProbe: "string — region returned/used by ec2 describe-regions",
      },
    },

    deployResource: {
      description:
        "Create or update an AWS resource via the AWS MCP server with GuildOS discipline baked in: list-before-create idempotency, tag convention, region default, and a cost-guard pre-check. ALL deployments — S3 bucket, EC2 instance, Lambda, ECS task, RDS, anything — go through this action's pattern, not raw `call_aws` improvisation.",
      howTo: [
        "STEP 0 — Cost guard (run once per account, before the FIRST deployment ever): confirm a billing alarm exists via `mcp__aws__aws___call_aws` with `aws cloudwatch describe-alarms --alarm-names guildos-budget-monthly`. If absent, create it: $50/mo default trigger to <REDACTED-EMAIL> via SNS. Doing this once means a runaway resource never silently burns the wallet — the worst-case AWS-MCP regression. Don't skip even if 'just one quick S3 bucket'.",
        "STEP 1 — Resource intent. State explicitly: service (s3/ec2/lambda/...), purpose (which GuildOS project — lifesci_intel, boster, etc.), expected lifetime (ephemeral / persistent), expected cost class (free-tier / <$1/day / <$10/day / higher → ESCALATE to user first).",
        "STEP 2 — Name + tags (CONVENTION; do not deviate):",
        "  - Name prefix: `<project>-<purpose>-<env>` e.g. `lifesci-intel-pmc-cache-prod`. Lowercase, hyphen-separated, project segment matches the skill-book id of the consuming project (lifesci_intel → 'lifesci-intel'). Names must be GLOBALLY-recognizable from the AWS console without context.",
        "  - Tags applied to every resource that supports tagging: { Project: '<project>', ManagedBy: 'guildos-mcp', CreatedBy: 'aws.deployResource', SkillBookSha: '<git rev-parse HEAD>' }. The SkillBookSha tag lets future agents trace WHICH version of this discipline created the resource.",
        "STEP 3 — Region. Default us-east-2 (the formulary AWS_REGION). Deviate only if the consuming project's skill book pins a different region (e.g. data-residency rule). Document the deviation in the resource's Project tag value.",
        "STEP 4 — List-before-create idempotency. ALWAYS call the list/describe API first with the intended name. If the resource exists and the config matches → no-op, return existing ARN. If it exists with different config → ESCALATE (the agent does NOT silently overwrite). Only if it's absent → create. This single discipline prevents the most common AWS regression: duplicate resources with `-2`, `-3`, `-final` suffixes accreting because the agent didn't check.",
        "STEP 5 — Execute via `mcp__aws__aws___call_aws` with the AWS CLI command string. The proxy SigV4-signs with the formulary creds (AdministratorAccess on user `mcp-server`). Capture the response — every create returns an ARN, save it to the calling skill book's output so subsequent steps can reference it.",
        "STEP 6 — Verify post-create. For S3: head-bucket. For EC2: describe-instances with the instance-id, wait for `state: running`. For Lambda: get-function. The MCP `call_aws` is fire-and-forget; the deployment isn't 'done' until a verify read confirms the resource is in the expected state.",
        "STEP 7 — Catalog. Append `{ kind, name, arn, region, createdAt, skillBookSha, costClass }` to the consuming project's deployment ledger (e.g. lifesci_intel keeps it in docs/lifesci-aws-resources.md or a Supabase table — defer to that project's convention; if there isn't one, prompt to add it before scaling deployments). The 'where is it?' question must have a one-look answer six months from now.",
        "TEARDOWN: the inverse procedure lives under the consuming project's skill book (e.g. lifesci_intel.teardownAwsResources), not here. Generic teardown is dangerous (delete the wrong project's bucket). Per-project teardown actions read their own ledger and delete only their own resources.",
        "REGRESSIONS to watch for (each = a past or anticipated failure):",
        "  - Creating without tags → resource is undiscoverable in 6 months → mandatory tag step.",
        "  - Hardcoding us-east-1 because 'AWS default' → splits the project across regions, doubles latency → use formulary AWS_REGION.",
        "  - Skipping the list-before-create → duplicate buckets/instances → mandatory list step.",
        "  - Creating in account 325698037467 (bosterbio) without a budget alarm → billing surprise → STEP 0 is non-skippable.",
      ].join("\n"),
      input: {
        project: "string — consuming skill book id, e.g. 'lifesci_intel'",
        service: "string — 's3' | 'ec2' | 'lambda' | 'ecs' | 'rds' | 'iam' | ...",
        purpose: "string — short-hyphen-id describing what this resource does, e.g. 'pmc-cache'",
        env: "'prod' | 'staging' | 'ephemeral'",
        costClass: "'free' | 'lt-1-per-day' | 'lt-10-per-day' | 'higher-escalate'",
        config: "object — service-specific parameters",
        region: "string — optional override of formulary AWS_REGION",
      },
      output: {
        arn: "string — ARN of the created or pre-existing resource",
        name: "string — the conventional name applied",
        existed: "boolean — true if list-before-create found a match",
        verifyState: "string — post-create state confirmed via describe/get",
      },
    },

    navigateConsole: {
      description:
        "Drive the AWS console via CIC to read state (resource lists, settings, billing widgets) that the MCP server doesn't expose or hasn't been authorized for yet.",
      howTo: [
        "Use CIC ONLY for reads — for state mutation prefer the MCP server (auditable, scriptable, region-pinned). CIC mutations on the console are tier-'read' and will be blocked anyway.",
        "Common entry points: EC2 https://<region>.console.aws.amazon.com/ec2/home?region=<region>, S3 https://s3.console.aws.amazon.com/s3/home?region=<region>, IAM https://us-east-1.console.aws.amazon.com/iam/home (IAM is global — region in URL is cosmetic), CloudWatch https://<region>.console.aws.amazon.com/cloudwatch/home?region=<region>, Billing/Cost https://console.aws.amazon.com/costmanagement/home.",
        "Use `read_page` filter=interactive to find tabs and side-nav links by accessible name; `get_page_text` on article regions for the human-readable summary. Console pages re-render on filter changes — re-read after every interaction, do not assume DOM stability.",
        "If a console page asks for MFA mid-session, escalate. Do not attempt to bypass or trigger MFA flows.",
      ].join("\n"),
      input: { service: "string — e.g. 'ec2', 's3', 'iam', 'cloudwatch', 'billing'", region: "string" },
      output: { tabId: "number", url: "string" },
    },
  },
};
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
