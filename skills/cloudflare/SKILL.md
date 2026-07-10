# cloudflare

<!-- GUILDOS-SKILL name="cloudflare" -->

<!-- GUILDOS-DESC -->
Audit Cloudflare zones, run controlled rollouts of rule changes, and interpret Cloudflare semantics. Read-only zone audits (DNS/firewall/cache/rulesets), post-change smoke-test rituals returning GREEN/YELLOW/RED, batched multi-change rollouts, and the semantic decoders that stop you misreading a legacy 'bypass' as a block or an origin 403 as a Cloudflare block. Primarily bosterbio.com.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/cloudflare/SKILL.md -->
```markdown
---
name: cloudflare
description: >-
  Audit Cloudflare zones, run controlled rollouts of rule changes, and interpret
  Cloudflare semantics. Read-only zone audits (DNS/firewall/cache/rulesets),
  post-change smoke-test rituals returning GREEN/YELLOW/RED, batched multi-change
  rollouts, and the semantic decoders that stop you misreading a legacy 'bypass'
  as a block or an origin 403 as a Cloudflare block. Primarily bosterbio.com.
---

# Cloudflare

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/cloudflare/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto cloudflare <action>` (or read the source directly).

## Actions

- **audit** — Run a read-only audit of a Cloudflare zone and return the compiled finding set.
- **classifyCloudflareResponse** — Classify an HTTP response as allow/challenge/block/origin-error/other using the right tells. Never guess by HTTP status alone — HTTP 403 with a CSP referencing challenges.cloudflare.com is a Managed Challenge (bot couldn't solve), NOT a block. HTTP 403 with body `<center>nginx</center>` is ORIGIN blocking (e.g. nginx UA blocklist), NOT Cloudflare — fetch the body, don't just check status.
- **diagnoseAIBotBlock** — When an AI/search bot UA gets 403 from a site behind Cloudflare, determine whether Cloudflare or the origin is blocking it. Do NOT assume Cloudflare.
- **interpretLegacyAction** — Translate a legacy Firewall Rule action ('bypass', 'skip', 'allow', 'challenge', 'managed_challenge', 'block', 'log') to what it actually does. Use this every time you read firewallRules output.
- **runControlledRollout** — Execute smokeTestRule for a whole batch of changes one-by-one, waiting observeMinutes between each. Designed for the days-long rollout pattern.
- **smokeTestRule** — Run the standard smoke-test ritual after toggling a single Cloudflare rule or setting. Returns GREEN/YELLOW/RED.
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/cloudflare/index.js -->
```js
/**
 * Cloudflare skill book — audit, smoke-test, and controlled-rollout recipes
 * for Cloudflare zones (primarily bosterbio.com).
 *
 * PURE TEXT REGISTRY: prompt strings only. No imports, no functions. Every
 * action's `howTo` tells the agent which weapon function to call, when, and
 * with what judgment. The executable orchestration lives in the paired weapon
 * (libs/weapon/cloudflare/index.js): read · search · normalize · audit.
 */

export const skillBook = {
  id: "cloudflare",
  title: "Cloudflare",
  description: "Audit Cloudflare zones, run controlled rollouts of rule changes, and interpret Cloudflare semantics.",
  steps: [],
  toc: {
    audit: {
      description: "Run a read-only audit of a Cloudflare zone and return the compiled finding set.",
      howTo: [
        "PRIMARY PATH: call weapon cloudflare.audit({zoneName}) — it chains all the reads below and returns { ok, items:{ zoneName, findings, scopeMissing, recommendations } }. The steps below document what that orchestration does and the judgment baked into it; run them by hand only when you need to inspect a single resource or extend the audit.",
        "Call weapon cloudflare.read({resource:'tokenCapabilities', zoneName}) first — this is non-obvious but critical: Cloudflare tokens are fine-grained and a valid-looking token may still be denied on zoneSettings / rulesets / botManagement. If any of those come back 403, mark them as MISSING_SCOPE in findings (scopeMissing) rather than skipping silently.",
        "Call read({resource:'zone'}) — plan tier gates which features/recommendations apply.",
        "Call read({resource:'dns'}) — look for cPanel/webmail/whm/webdisk subdomains that are PROXIED (orange cloud). Those are misconfigured: proxying cPanel breaks AutoSSL and can rate-limit admin logins. Flag. Also look for staging hostnames (dev., stage., staging., jr-staging., *.mobile., *.staging.) — these often leak publicly.",
        "Call read({resource:'firewallRules'}) — these are LEGACY. The action 'bypass' does NOT mean block; it means SKIP all CF security for matching traffic. A rule labeled '[Blacklist] … action=bypass' is inverted from intent. Use normalize({kind:'legacyFirewallAction', data:rule.action}) to annotate each rule with its real meaning in the report.",
        "Call read({resource:'pageRules'}) and read({resource:'rateLimits'}). Zero of either + zero Cache Rules = static/HTML content is at the mercy of origin headers; usually means cache hit is terrible.",
        "Call read({resource:'rulesets'}) if the token has Zone WAF scope. The rule IDs in firewallEventsAdaptiveGroups point at entries in these rulesets, NOT at legacy firewall rule IDs.",
        "Call search({resource:'cacheByContentType', windowHours:24}) and search({resource:'cacheByPath', windowHours:24, limit:50}). cacheStatus values mean: hit=served from edge (origin saved); expired=was cached, revalidating (also saves origin); miss=eligible, not in cache yet; dynamic=CF decided not to cache (usually cookies/Cache-Control); none=origin said no-store; bypass=a rule explicitly skipped cache. Only hit+expired save origin. Bypass+none+dynamic on static assets (CSS/JS/fonts/PDF) or on anonymous HTML is the top cache-related finding.",
        "AGGREGATE CACHE HIT RATIO IS MISLEADING ON BOT-HEAVY SITES (bosterbio.com verified 2026-04-25). A low HTML hit ratio (~4%) can look alarming but frequently reflects bot pressure, not a cache misconfiguration. Before concluding the cache rule is broken, run a controlled test: (a) via CIC, have the signed-in user's real browser fetch a cacheable path twice and inspect cf-cache-status — HIT on pass 2 means the cache is working for humans with cookies; (b) in GraphQL, filter HTML traffic by userAgent_like '%Chrome/13x%' and re-compute hit ratio — the aggregate is dominated by (i) Meta-ExternalAgent/facebookexternalhit challenges returning cache=none 403 pages (not cached is correct behavior for challenge responses), (ii) distributed scrapers enumerating faceted-nav query strings (?product_cat[0]=… varies per request so each is a unique cache key by default), (iii) other crawlers hitting paths excluded from the cache rule (admin/customer/rest etc). DISPROVEN on this site: the 'Magento session cookies segment cache keys' hypothesis — tested with 38 real Magento cookies including form_key, mage-cache-sessid, mage-cache-storage, private_content_version, section_data_ids, and got HIT identical to anonymous curl. Do not SSH to strip cookies unless you have empirically proven cookies are the segmenter.",
        "Call search({resource:'firewallEvents', windowHours:24}). NOTE: this dataset is hard-capped to 1-day windows on Pro plan — the weapon clamps for you. Cross-reference top rule IDs against firewall rules + rulesets.",
        "Fetch https://<zone>/robots.txt and a sample product page with a plain User-Agent and with 'ClaudeBot/1.0' — then call normalize({kind:'httpResponse'}) on each. If ClaudeBot returns classification='challenge', the AI-bot whitelist is not working (a WAF custom rule fires before the legacy skip).",
        "Compile findings: critical (origin-load, inverted rules, blocked AI bots, proxied admin) → material (missing cache/rate-limit policy, dominant skip traffic) → minor.",
        "Return: { findings: [...], scopeMissing: [...], recommendations: [...] }.",
      ].join("\n"),
      input: { zoneName: "string — e.g. bosterbio.com" },
      output: { findings: "array of finding objects", scopeMissing: "array of string", recommendations: "array" },
    },

    smokeTestRule: {
      description: "Run the standard smoke-test ritual after toggling a single Cloudflare rule or setting. Returns GREEN/YELLOW/RED.",
      howTo: [
        "A smoke test has a fixed shape — do NOT improvise, do NOT skip steps:",
        "1. BASELINE (T-0): before the change, capture (a) analytics metrics for affected paths via weapon cloudflare.search({resource:'cacheByPath' or 'firewallEvents', windowHours:1, limit:50}) AND (b) run the SAME curl probes you plan to run post-change. Save both. This catches pre-existing flakiness (e.g. a URL that already times out at origin) so you don't misattribute it to your change.",
        "2. APPLY (T+0): the caller has already applied the change; your job is observation. Record the change description + timestamp.",
        "3. ACTIVE SMOKE (T+0..5): re-run the baseline probes. For each, pass the response to weapon cloudflare.normalize({kind:'httpResponse', data:{status, headers, body}}). Expected classifications are in input.probes[i].expect. Compare each result against the pre-change baseline, not against the abstract expectation — a URL that was already slow is not a regression.",
        "4. OBSERVE (T+5..35): DO NOT make other changes in this window.",
        "5. METRICS (T+35): re-run the baseline queries. Compute deltas. Pull a sample of 10 IPs hitting affected paths via search({resource:'firewallEvents'}) — did any go from allow→block unexpectedly?",
        "6. DECIDE: GREEN (all probes classified as expected AND no 5xx spike >2× baseline AND no legit-UA 403 spike >1%). YELLOW (probes OK but one soft signal off — tune, don't revert). RED (any probe misclassified vs baseline OR 5xx spike OR legit-user blocked).",
        "RED triggers immediate revert. Always ask the caller to confirm the revert action before reporting back — agent does not auto-revert.",
        "CASE STUDY — load-bearing 'bypass' rule: A legacy Firewall Rule labeled '[Monitor only] Everyone Else' with wildcard host and action=bypass is NOT monitor-only — it was silently shielding 100% of fall-through traffic from managed WAF. Disabling it in one go causes a surge of previously-skipped traffic to hit managed WAF and Magento origin at once, producing RED smoke (timeouts on high-volume HTML URLs like /new-products). Correct path: phased migration — first narrow the filter scope to specific hostnames/paths, then migrate matches into explicit named Custom Rules, only then retire the catch-all. Never flip a wildcard-host bypass rule off in one step.",
      ].join("\n"),
      input: {
        zoneName: "string",
        changeDescription: "string",
        probes: "array of { url, userAgent?, headers?, expect: 'allow'|'challenge'|'block'|'origin-error' }",
        affectedPaths: "array of string — paths to sample in baseline/post-change analytics",
        baselineWindowMinutes: "int, default 60",
        observeMinutes: "int, default 30",
      },
      output: { decision: "GREEN|YELLOW|RED", probeResults: "array", metricDeltas: "object", notes: "string" },
    },

    runControlledRollout: {
      description: "Execute smokeTestRule for a whole batch of changes one-by-one, waiting observeMinutes between each. Designed for the days-long rollout pattern.",
      howTo: [
        "Process the input.changes array strictly in order. For each change: (a) pause until caller confirms the change is applied, (b) run the smokeTestRule recipe with the change's probe spec, (c) honor the decision — if RED, halt the batch and surface the revert requirement to the caller. Do not auto-advance on RED.",
        "Between changes, wait input.coolDownMinutes (default 15) even if decision was GREEN, so one change's effects don't contaminate the next change's baseline.",
        "Irreversible or extra-care changes (flagged 🔒 in the rollout plan) must NOT be in the same batch as ordinary changes — require a separate `phase: 'careful'` run.",
      ].join("\n"),
      input: {
        zoneName: "string",
        changes: "array of smokeTestRule inputs",
        coolDownMinutes: "int, default 15",
        phase: "'standard' | 'careful' — gate check",
      },
      output: { batchDecision: "GREEN|PARTIAL|HALTED", perChange: "array of smokeTestRule outputs" },
    },

    interpretLegacyAction: {
      description: "Translate a legacy Firewall Rule action ('bypass', 'skip', 'allow', 'challenge', 'managed_challenge', 'block', 'log') to what it actually does. Use this every time you read firewallRules output.",
      howTo: "Call weapon cloudflare.normalize({kind:'legacyFirewallAction', data:actionString}). Returns { action, meaning }. The most important non-obvious one: 'bypass' is a WHITELIST (skip WAF/Bot/Security-Level checks), not a block. A rule named '[Blacklist] … action=bypass' is inverted from intent. Related: 'skip' skips only the downstream products named in the rule; 'allow' lets traffic through all security products (whitelist); 'challenge'/'managed_challenge'/'js_challenge' are interactive challenges; 'block' drops the request; 'log' is monitor-only with no enforcement.",
      input: { action: "string — action name from legacy firewall rule" },
      output: { action: "string", meaning: "string" },
    },

    classifyCloudflareResponse: {
      description: "Classify an HTTP response as allow/challenge/block/origin-error/other using the right tells. Never guess by HTTP status alone — HTTP 403 with a CSP referencing challenges.cloudflare.com is a Managed Challenge (bot couldn't solve), NOT a block. HTTP 403 with body `<center>nginx</center>` is ORIGIN blocking (e.g. nginx UA blocklist), NOT Cloudflare — fetch the body, don't just check status.",
      howTo: "Capture the response as { status, headers, body } and pass to weapon cloudflare.normalize({kind:'httpResponse', data:{status, headers, body}}). Headers must include `cf-cache-status` and `content-security-policy` (lowercased keys are fine). Body must be at least the first 500 bytes — the origin-vs-CF distinction REQUIRES body inspection; status alone lies. Returns { classification, reason, cfCache, cfMitigated }. Precedence the weapon applies: 403+CSP→challenge; cf-mitigated header→authoritative (challenge|block); 5xx Cloudflare error page (Error code 1xxx)→origin-error; 403 with server=cloudflare→block; 2xx/3xx→allow.",
      input: { status: "int", headers: "object", body: "string — first 500+ bytes required for origin-vs-CF distinction" },
      output: { classification: "string", reason: "string", cfCache: "string|null", cfMitigated: "string|null" },
    },

    diagnoseAIBotBlock: {
      description: "When an AI/search bot UA gets 403 from a site behind Cloudflare, determine whether Cloudflare or the origin is blocking it. Do NOT assume Cloudflare.",
      howTo: [
        "Run `curl -sS -A '<bot UA>' -D headers.txt -o body.html https://site/`, then pass {status, headers, body} to weapon cloudflare.normalize({kind:'httpResponse'}) and interpret with the rules below.",
        "Inspect headers.txt: if `server: cloudflare` AND body contains `<center>nginx</center>` or `<center>apache</center>` — it's ORIGIN blocking. Cloudflare is just proxying the origin's 403. Fix requires SSH/cPanel access to edit the web server's UA blocklist, NOT a CF rule change.",
        "If body contains `challenges.cloudflare.com` CSP directive — it's a Managed Challenge (not a block).",
        "If `cf-mitigated` header is present — Cloudflare mitigated; the header value says why.",
        "Common Magento/nginx pattern: origin explicitly blocks {ClaudeBot, anthropic-ai, CCBot, Bytespider, Amazonbot, Meta-ExternalAgent} in nginx/httpd.conf UA filter. GPTBot, OAI-SearchBot, PerplexityBot, Googlebot typically pass. If the user's goal is to ALLOW certain training bots, editing CF alone will not fix it.",
      ].join("\n"),
      input: { url: "string", botUserAgent: "string" },
      output: { source: "'cloudflare-block'|'cloudflare-challenge'|'origin-block'|'allow'|'other'", fixRequires: "'cf-waf-rule'|'cf-toggle'|'origin-nginx-edit'|'none'", evidence: "string" },
    },
  },
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/weapon/cloudflare/index.js -->
```js
/**
 * Cloudflare weapon — unified read/search connector for Cloudflare zones.
 *
 * Auth: CLOUDFLARE_API_TOKEN from profiles.env_vars or process.env.
 *
 * Hides always-on plumbing: pagination for DNS, GraphQL scalar nuances
 * (Date=YYYY-MM-DD vs Time=RFC3339), the 1-day window cap on adaptive
 * analytics datasets (Pro), and the sunset legacy REST `/analytics/dashboard`
 * endpoint (GraphQL-only on new zones).
 */

import { database } from "@/libs/council/database";
import { skillActionOk, skillActionErr } from "@/libs/skill_book/actionResult.js";

export const toc = {
  read: "Read a Cloudflare resource by name (zone, zones, dns, firewallRules, accessRules, pageRules, rateLimits, zoneSettings, rulesets, ruleset, botManagement, tokenCapabilities). Input: {resource, zoneName?|zoneId?, rulesetId?}. Resolves zone by name when only zoneName is given.",
  search: "Run a GraphQL analytics query. Datasets: requestsDaily (7d), cacheByHost (24h), cacheByPath (24h), cacheByContentType (24h), firewallEvents (24h — hard cap), bypassedPaths (24h). Input: {resource, zoneName?|zoneId?, limit?, windowHours?}.",
  normalize: "Interpret ambiguous Cloudflare semantics. Kinds: legacyFirewallAction (translates bypass/skip to actual behavior), httpResponse (classifies a fetched response as allow/challenge/block/origin-error using CSP+body tells).",
  audit: "Run a read-only audit of a zone (chains read+normalize): token capabilities → DNS (proxied cPanel-family + staging exposure) → legacy firewall rules (inverted-bypass detection) → page/rate policy → static-asset cache leakage. Returns a skillActionOk envelope { zoneName, findings, scopeMissing, recommendations }. Input: {zoneName} (+ optional userId as first arg).",
};

const API = "https://api.cloudflare.com/client/v4";
const GRAPHQL = "https://api.cloudflare.com/client/v4/graphql";

// ---------------------------------------------------------------------------
// Auth
// ---------------------------------------------------------------------------

async function getToken(userId) {
  if (process.env.CLOUDFLARE_API_TOKEN) return process.env.CLOUDFLARE_API_TOKEN;
  const db = await database.init("service");
  if (userId) {
    const { data } = await db.from("profiles").select("env_vars").eq("id", userId).maybeSingle();
    const t = data?.env_vars?.CLOUDFLARE_API_TOKEN;
    if (t) return t;
  }
  // Fallback: any profile carrying the token (convenience for inline CLI use).
  const { data } = await db.from("profiles").select("id,env_vars");
  for (const p of data || []) {
    const t = p.env_vars?.CLOUDFLARE_API_TOKEN;
    if (t) return t;
  }
  return null;
}

async function cfFetch(path, opts = {}, userId) {
  const token = await getToken(userId);
  if (!token) throw new Error("Missing CLOUDFLARE_API_TOKEN");
  const url = path.startsWith("http") ? path : API + path;
  const res = await fetch(url, {
    ...opts,
    headers: { Authorization: `Bearer ${token}`, "Content-Type": "application/json", ...opts.headers },
  });
  const body = await res.text();
  let json;
  try { json = JSON.parse(body); } catch { json = { raw: body }; }
  return { status: res.status, ok: res.ok, json };
}

async function cfGraphQL(query, variables, userId) {
  const token = await getToken(userId);
  if (!token) throw new Error("Missing CLOUDFLARE_API_TOKEN");
  const res = await fetch(GRAPHQL, {
    method: "POST",
    headers: { Authorization: `Bearer ${token}`, "Content-Type": "application/json" },
    body: JSON.stringify({ query, variables }),
  });
  const json = await res.json();
  if (json.errors?.length) {
    throw new Error(`Cloudflare GraphQL error: ${JSON.stringify(json.errors).slice(0, 400)}`);
  }
  return json.data;
}

// ---------------------------------------------------------------------------
// Zone resolution
// ---------------------------------------------------------------------------

async function resolveZoneId({ zoneId, zoneName }, userId) {
  if (zoneId) return zoneId;
  if (!zoneName) throw new Error("resolveZone: provide zoneId or zoneName");
  const { json } = await cfFetch(`/zones?name=${encodeURIComponent(zoneName)}`, {}, userId);
  if (!json.success || !json.result?.[0]) {
    throw new Error(`Zone "${zoneName}" not found or token lacks access`);
  }
  return json.result[0].id;
}

// ---------------------------------------------------------------------------
// read
// ---------------------------------------------------------------------------

/**
 * @param {{resource: string, zoneId?: string, zoneName?: string, rulesetId?: string}} input
 * @param {string} [userId]
 */
export async function read(input = {}, userId) {
  const { resource, rulesetId } = input;
  if (!resource) throw new Error("read: resource required");

  if (resource === "zones") {
    const { json } = await cfFetch("/zones?per_page=50", {}, userId);
    return { zones: json.result || [] };
  }

  if (resource === "tokenCapabilities") {
    return readTokenCapabilities(input, userId);
  }

  // All other resources need a zone.
  const zoneId = await resolveZoneId(input, userId);

  if (resource === "zone") {
    const { json } = await cfFetch(`/zones/${zoneId}`, {}, userId);
    return { zone: json.result || null };
  }

  if (resource === "dns") {
    const all = [];
    let page = 1;
    while (true) {
      const { json } = await cfFetch(`/zones/${zoneId}/dns_records?per_page=200&page=${page}`, {}, userId);
      if (!json.result?.length) break;
      all.push(...json.result);
      const total = json.result_info?.total_pages || 1;
      if (page >= total) break;
      page++;
    }
    return { records: all };
  }

  if (resource === "firewallRules") {
    const { json } = await cfFetch(`/zones/${zoneId}/firewall/rules?per_page=100`, {}, userId);
    return { rules: json.result || [] };
  }

  if (resource === "accessRules") {
    const { json } = await cfFetch(`/zones/${zoneId}/firewall/access_rules/rules?per_page=200`, {}, userId);
    return { rules: json.result || [] };
  }

  if (resource === "pageRules") {
    const { json } = await cfFetch(`/zones/${zoneId}/pagerules`, {}, userId);
    return { rules: json.result || [] };
  }

  if (resource === "rateLimits") {
    const { json } = await cfFetch(`/zones/${zoneId}/rate_limits`, {}, userId);
    return { rules: json.result || [] };
  }

  if (resource === "zoneSettings") {
    const { status, json } = await cfFetch(`/zones/${zoneId}/settings`, {}, userId);
    if (status === 403) throw new Error("Token lacks Zone Settings:Read");
    const flat = {};
    for (const s of json.result || []) flat[s.id] = { value: s.value, editable: s.editable, modified_on: s.modified_on };
    return { settings: flat };
  }

  if (resource === "rulesets") {
    const { status, json } = await cfFetch(`/zones/${zoneId}/rulesets`, {}, userId);
    if (status === 403) throw new Error("Token lacks Zone WAF:Read");
    return { rulesets: json.result || [] };
  }

  if (resource === "ruleset") {
    if (!rulesetId) throw new Error("read ruleset requires rulesetId");
    const { status, json } = await cfFetch(`/zones/${zoneId}/rulesets/${rulesetId}`, {}, userId);
    if (status === 403) throw new Error("Token lacks Zone WAF:Read");
    return { ruleset: json.result || null };
  }

  if (resource === "botManagement") {
    const { status, json } = await cfFetch(`/zones/${zoneId}/bot_management`, {}, userId);
    if (status === 403) throw new Error("Token lacks Account Bot Management:Read");
    return { botManagement: json.result || null };
  }

  throw new Error(`read: unknown resource "${resource}"`);
}

async function readTokenCapabilities({ zoneName, zoneId }, userId) {
  const token = await getToken(userId);
  if (!token) throw new Error("Missing CLOUDFLARE_API_TOKEN");
  // Verify token first
  const verifyRes = await fetch(API + "/user/tokens/verify", { headers: { Authorization: `Bearer ${token}` } });
  const verify = await verifyRes.json();
  if (!verify.success) return { valid: false, verify, capabilities: {} };

  let zid = zoneId;
  if (!zid && zoneName) {
    try { zid = await resolveZoneId({ zoneName }, userId); } catch { /* leave null */ }
  }
  const endpoints = zid ? [
    ["zone", `/zones/${zid}`],
    ["dns", `/zones/${zid}/dns_records?per_page=1`],
    ["firewallRules", `/zones/${zid}/firewall/rules?per_page=1`],
    ["accessRules", `/zones/${zid}/firewall/access_rules/rules?per_page=1`],
    ["pageRules", `/zones/${zid}/pagerules`],
    ["rateLimits", `/zones/${zid}/rate_limits`],
    ["zoneSettings", `/zones/${zid}/settings`],
    ["rulesets", `/zones/${zid}/rulesets`],
    ["botManagement", `/zones/${zid}/bot_management`],
  ] : [];
  const capabilities = {};
  for (const [name, p] of endpoints) {
    const r = await fetch(API + p, { headers: { Authorization: `Bearer ${token}` } });
    capabilities[name] = { status: r.status, ok: r.ok };
  }
  return { valid: true, tokenId: verify.result?.id, zoneId: zid || null, capabilities };
}

// ---------------------------------------------------------------------------
// search (GraphQL analytics)
// ---------------------------------------------------------------------------

const fmtDate = (d) => d.toISOString().slice(0, 10);

/**
 * @param {{resource: string, zoneId?: string, zoneName?: string, limit?: number, windowHours?: number}} input
 * @param {string} [userId]
 */
export async function search(input = {}, userId) {
  const { resource, limit = 50, windowHours = 24 } = input;
  if (!resource) throw new Error("search: resource required");
  const zoneId = await resolveZoneId(input, userId);
  const now = new Date();
  const startWindow = new Date(now.getTime() - windowHours * 3600 * 1000);

  if (resource === "requestsDaily") {
    // 7-day daily rollup. Uses Date scalar (YYYY-MM-DD).
    const s = new Date(now.getTime() - 7 * 86400 * 1000);
    const q = `query($z:String!,$s:Date!,$e:Date!){viewer{zones(filter:{zoneTag:$z}){
      httpRequests1dGroups(limit:7,filter:{date_geq:$s,date_leq:$e},orderBy:[date_ASC]){
        dimensions{date}
        sum{requests cachedRequests bytes cachedBytes threats}
        uniq{uniques}
      }
    }}}`;
    const d = await cfGraphQL(q, { z: zoneId, s: fmtDate(s), e: fmtDate(now) }, userId);
    return { daily: d.viewer.zones[0]?.httpRequests1dGroups || [] };
  }

  if (resource === "cacheByHost") {
    const q = `query($z:String!,$d1:Time!,$d2:Time!,$n:Int!){viewer{zones(filter:{zoneTag:$z}){
      httpRequestsAdaptiveGroups(limit:$n,filter:{datetime_geq:$d1,datetime_leq:$d2},orderBy:[count_DESC]){
        count dimensions{cacheStatus clientRequestHTTPHost edgeResponseContentTypeName}
        sum{edgeResponseBytes}
      }
    }}}`;
    const d = await cfGraphQL(q, { z: zoneId, d1: startWindow.toISOString(), d2: now.toISOString(), n: limit }, userId);
    return { rows: d.viewer.zones[0]?.httpRequestsAdaptiveGroups || [] };
  }

  if (resource === "cacheByContentType") {
    const q = `query($z:String!,$d1:Time!,$d2:Time!,$n:Int!){viewer{zones(filter:{zoneTag:$z}){
      httpRequestsAdaptiveGroups(limit:$n,filter:{datetime_geq:$d1,datetime_leq:$d2},orderBy:[count_DESC]){
        count dimensions{cacheStatus edgeResponseContentTypeName}
        sum{edgeResponseBytes}
      }
    }}}`;
    const d = await cfGraphQL(q, { z: zoneId, d1: startWindow.toISOString(), d2: now.toISOString(), n: limit }, userId);
    return { rows: d.viewer.zones[0]?.httpRequestsAdaptiveGroups || [] };
  }

  if (resource === "cacheByPath" || resource === "bypassedPaths") {
    const cacheFilter = resource === "bypassedPaths" ? `,cacheStatus_in:["bypass","none","dynamic","miss"]` : "";
    const q = `query($z:String!,$d1:Time!,$d2:Time!,$n:Int!){viewer{zones(filter:{zoneTag:$z}){
      httpRequestsAdaptiveGroups(limit:$n,filter:{datetime_geq:$d1,datetime_leq:$d2${cacheFilter}},orderBy:[count_DESC]){
        count dimensions{cacheStatus edgeResponseContentTypeName clientRequestPath clientRequestHTTPHost}
        sum{edgeResponseBytes}
      }
    }}}`;
    const d = await cfGraphQL(q, { z: zoneId, d1: startWindow.toISOString(), d2: now.toISOString(), n: limit }, userId);
    return { rows: d.viewer.zones[0]?.httpRequestsAdaptiveGroups || [] };
  }

  if (resource === "firewallEvents") {
    // HARD CAP: adaptive firewall dataset rejects windows > 1 day on Pro.
    const clamp = Math.min(windowHours, 24);
    const d1 = new Date(now.getTime() - clamp * 3600 * 1000);
    const q = `query($z:String!,$d1:Time!,$d2:Time!,$n:Int!){viewer{zones(filter:{zoneTag:$z}){
      firewallEventsAdaptiveGroups(limit:$n,filter:{datetime_geq:$d1,datetime_leq:$d2},orderBy:[count_DESC]){
        count dimensions{action source ruleId clientRequestHTTPHost clientCountryName}
      }
    }}}`;
    const d = await cfGraphQL(q, { z: zoneId, d1: d1.toISOString(), d2: now.toISOString(), n: limit }, userId);
    return { events: d.viewer.zones[0]?.firewallEventsAdaptiveGroups || [], windowClampedToHours: clamp };
  }

  throw new Error(`search: unknown resource "${resource}"`);
}

// ---------------------------------------------------------------------------
// normalize (pure interpretation — no network)
// ---------------------------------------------------------------------------

/**
 * @param {{kind: string, data: unknown}} input
 */
export function normalize({ kind, data } = {}) {
  if (kind === "legacyFirewallAction") {
    // Legacy Firewall Rules action semantics differ from the plain-English names:
    // - "bypass": SKIP Cloudflare security features for matching requests (WAF, Bot Fight, Security Level, Rate Limiting by ID).
    //   NOT a block. A rule labeled "[Blacklist] … action=bypass" is inverted from intent.
    // - "skip": Skip specific downstream products (specified in `products`). Same family as bypass but scoped.
    // - "allow": Allow matching requests through all security products (same end state as bypass for matching traffic, but semantically different in audit trail).
    // - "challenge" / "managed_challenge" / "js_challenge": Interactive challenge.
    // - "block": Drop the request.
    // - "log": Log only, no enforcement.
    const a = String(data || "").toLowerCase();
    const meaning = {
      bypass: "SKIP all WAF/Bot/Security-Level checks for matching traffic (whitelist). NOT a block.",
      skip: "Skip the downstream products specified in the rule (see `products`). Whitelist-ish.",
      allow: "Allow through; bypasses Managed Challenges. Whitelist.",
      challenge: "Interactive CAPTCHA challenge.",
      managed_challenge: "Cloudflare Managed Challenge (JS + non-interactive for humans, fails bots).",
      js_challenge: "JS challenge (deprecated for Managed).",
      block: "Drop the request. Enforcement.",
      log: "Log only. No enforcement.",
    };
    return { action: a, meaning: meaning[a] || `Unknown action "${a}"` };
  }

  if (kind === "httpResponse") {
    // Classify a response object: { status, headers, body }.
    const r = /** @type {{status: number, headers: Record<string,string>, body?: string}} */ (data || {});
    const status = r.status;
    const headers = r.headers || {};
    const body = r.body || "";
    const csp = (headers["content-security-policy"] || headers["Content-Security-Policy"] || "").toLowerCase();
    const server = (headers.server || headers.Server || "").toLowerCase();
    const cfCache = headers["cf-cache-status"] || headers["CF-Cache-Status"] || null;
    const cfMitigated = headers["cf-mitigated"] || headers["CF-Mitigated"] || null;

    // Managed Challenge tell: 403 + CSP referencing challenges.cloudflare.com.
    if (status === 403 && csp.includes("challenges.cloudflare.com")) {
      return { classification: "challenge", reason: "Managed Challenge page (CSP references challenges.cloudflare.com)", cfCache, cfMitigated };
    }
    // cf-mitigated header is the authoritative signal when present.
    if (cfMitigated) {
      return { classification: cfMitigated === "challenge" ? "challenge" : "block", reason: `cf-mitigated=${cfMitigated}`, cfCache, cfMitigated };
    }
    // Cloudflare 1xxx error pages are edge-origin errors, not blocks.
    if (status >= 500 && /cloudflare/.test(body) && /Error code \d{3,4}/i.test(body)) {
      return { classification: "origin-error", reason: "Cloudflare error page (origin unreachable)", cfCache, cfMitigated };
    }
    if (status === 403 && server === "cloudflare") {
      return { classification: "block", reason: "HTTP 403 from Cloudflare edge (likely WAF or firewall block)", cfCache, cfMitigated };
    }
    if (status >= 200 && status < 400) {
      return { classification: "allow", reason: `HTTP ${status}`, cfCache, cfMitigated };
    }
    return { classification: "other", reason: `HTTP ${status}`, cfCache, cfMitigated };
  }

  throw new Error(`normalize: unknown kind "${kind}"`);
}

// ---------------------------------------------------------------------------
// audit — read-only zone audit (orchestrates read + normalize)
// ---------------------------------------------------------------------------

function pickInput(a, b) {
  if (b !== undefined && typeof b === "object" && b !== null && !Array.isArray(b)) return b;
  if (a !== undefined && typeof a === "object" && a !== null && !Array.isArray(a)) return a;
  return {};
}

/**
 * Read-only audit of a Cloudflare zone. Agent-facing entry point: chains
 * read() + normalize() into a compiled finding set. Pure reads — no mutation.
 *
 * Calling conventions (both supported, mirroring the adventurer-runtime shape):
 *   audit({ zoneName })            // input as first arg
 *   audit(userId, { zoneName })    // userId as first arg, input as second
 *
 * @param {unknown} [a] userId or input
 * @param {unknown} [b] input when a is userId
 * @returns {Promise<import("@/libs/skill_book/actionResult.js").SkillActionResult>}
 */
export async function audit(a, b) {
  const input = pickInput(a, b);
  const userId = typeof a === "string" ? a : undefined;
  const zoneName = String(input.zoneName ?? "bosterbio.com").trim();

  try {
    const caps = await read({ resource: "tokenCapabilities", zoneName }, userId);
    const scopeMissing = Object.entries(caps.capabilities || {})
      .filter(([, v]) => !v.ok)
      .map(([k]) => k);

    const findings = [];
    const recommendations = [];

    // DNS: cPanel-family proxied, staging exposure
    const { records } = await read({ resource: "dns", zoneName }, userId);
    const adminProxied = records.filter(
      (r) => r.proxied && /^(cpanel|webmail|whm|webdisk|cpcalendars|cpcontacts)\./i.test(r.name || ""),
    );
    if (adminProxied.length) {
      findings.push({
        severity: "critical",
        id: "admin-proxied",
        detail: `${adminProxied.length} cPanel-family records are orange-clouded (proxied). Proxying breaks AutoSSL and risks WAF-rate-limiting admin logins.`,
        records: adminProxied.map((r) => r.name),
      });
      recommendations.push({ id: "unproxy-admin", impact: "hygiene-must-do", effort: "10 min" });
    }

    const stagingHosts = records.filter((r) =>
      /^(dev|stage|staging|jr-staging)\./i.test(r.name || "") || /\.(staging|mobile)\./i.test(r.name || ""),
    );
    if (stagingHosts.length) {
      findings.push({
        severity: "critical",
        id: "staging-exposed",
        detail: `${stagingHosts.length} staging/dev hostnames exist in DNS and may be publicly accessible.`,
        records: stagingHosts.map((r) => r.name),
      });
    }

    // Legacy firewall rules semantic analysis (inverted-bypass detection)
    const { rules: fwRules } = await read({ resource: "firewallRules", zoneName }, userId);
    for (const rule of fwRules) {
      const n = normalize({ kind: "legacyFirewallAction", data: rule.action });
      const desc = rule.description || "";
      const claimsBlock = /\b(blacklist|ban|block)\b/i.test(desc);
      if (claimsBlock && (n.action === "bypass" || n.action === "skip" || n.action === "allow")) {
        findings.push({
          severity: "critical",
          id: "inverted-firewall-rule",
          detail: `Rule "${desc}" is labeled as blacklist but action="${rule.action}" which means ${n.meaning}`,
          ruleId: rule.id,
        });
      }
    }

    // Page rules / rate limits / cache policy
    const [{ rules: pageRules }, { rules: rateLimits }] = await Promise.all([
      read({ resource: "pageRules", zoneName }, userId),
      read({ resource: "rateLimits", zoneName }, userId),
    ]);
    if (pageRules.length === 0 && rateLimits.length === 0) {
      findings.push({
        severity: "material",
        id: "no-cache-or-rate-policy",
        detail: "Zero Page Rules, zero legacy Rate Limits. All cache/rate policy relies on origin headers + Custom Rules.",
      });
    }

    // Cache analytics — find worst cache=bypass content types
    try {
      const { rows: ctRows } = await search({ resource: "cacheByContentType", zoneName, limit: 50, windowHours: 24 }, userId);
      const byType = {};
      for (const row of ctRows) {
        const ct = row.dimensions.edgeResponseContentTypeName || "unknown";
        byType[ct] = byType[ct] || { hit: 0, bypass: 0, miss: 0, none: 0, dynamic: 0, totalBytes: 0 };
        const bucket = row.dimensions.cacheStatus;
        if (byType[ct][bucket] !== undefined) byType[ct][bucket] += row.count;
        byType[ct].totalBytes += row.sum?.edgeResponseBytes || 0;
      }
      const leaky = Object.entries(byType)
        .filter(([ct, v]) => ["css", "js", "pdf", "svg", "bin", "woff", "woff2", "ttf"].includes(ct) && v.bypass + v.none > v.hit)
        .map(([ct, v]) => ({ contentType: ct, ...v }));
      if (leaky.length) {
        findings.push({
          severity: "critical",
          id: "static-cache-bypass",
          detail: `Static asset content types where bypass+none exceeds hit — origin is serving cacheable assets`,
          contentTypes: leaky,
        });
        recommendations.push({ id: "static-cache-rule", impact: "cuts ~30+ GB/day off origin", effort: "20 min" });
      }
    } catch (e) {
      findings.push({ severity: "minor", id: "cache-analytics-failed", detail: String(e.message || e) });
    }

    return skillActionOk({ zoneName, findings, scopeMissing, recommendations });
  } catch (e) {
    return skillActionErr(e instanceof Error ? e.message : String(e));
  }
}

// ---------------------------------------------------------------------------
// Credential health — used by the WEAPONS registry.
// ---------------------------------------------------------------------------

export async function checkCredentials(userId) {
  try {
    const token = await getToken(userId);
    if (!token) return { ok: false, msg: "No CLOUDFLARE_API_TOKEN in env or profiles" };
    const r = await fetch(API + "/user/tokens/verify", { headers: { Authorization: `Bearer ${token}` } });
    const j = await r.json();
    if (!j.success) return { ok: false, msg: `Token invalid: ${JSON.stringify(j.errors).slice(0, 120)}` };
    return { ok: true, msg: `Token ${j.result?.id?.slice(0, 8)}… active` };
  } catch (e) {
    return { ok: false, msg: e instanceof Error ? e.message : String(e) };
  }
}
```
<!-- /GUILDOS-FILE -->
