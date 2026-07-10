# trustTheBigBoy

<!-- GUILDOS-SKILL name="trustTheBigBoy" -->

<!-- GUILDOS-DESC -->
[CONCLUDED 2026-06-09 — EXPERIMENT_ACTIVE=false; all gates now enforce] YOLO-vs-NORMAL A/B harness that evaluated whether Opus 4.8 needs less of GuildOS's deterministic nagging. Verdict (docs/smoke-testing/yolo4-8-performance.md): verbal FORMATTING nags = ceremony, but manufactured-stop/trailing-question = load-bearing (corrective-% metric is blind to them). Code retained dormant; flip EXPERIMENT_ACTIVE to re-open.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/trustTheBigBoy/SKILL.md -->
```markdown
---
name: trustTheBigBoy
description: >-
  [CONCLUDED 2026-06-09 — EXPERIMENT_ACTIVE=false; all gates now enforce]
  YOLO-vs-NORMAL A/B harness that evaluated whether Opus 4.8 needs less of
  GuildOS's deterministic nagging. Verdict (docs/smoke-testing/yolo4-8-performance.md):
  verbal FORMATTING nags = ceremony, but manufactured-stop/trailing-question =
  load-bearing (corrective-% metric is blind to them). Code retained dormant;
  flip EXPERIMENT_ACTIVE to re-open.
---

# Trust The Big Boy

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/trustTheBigBoy/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto trustTheBigBoy <action>` (or read the source directly).

## Actions

- **assignMode** — SessionStart: randomly assign this session `normal` or `yolo` and initialize its run record. Called by the .claude/hooks/session-mode.mjs thin shim.
- **recordCorrective** — Agentic: after classifying the user's message as CORRECTIVE, record the concise-feedback length + summary. Called by the model via cli.mjs.
- **recordUserMessage** — Deterministic per-turn accounting (turns++, user-message count + char total). Called by the preflight hook.
- **setRunMeta** — Agentic: stamp the run's model (e.g. 'Opus 4.8') and human runName (thread title if reachable, else fallback). Called by the model via cli.mjs.
- **syncMd** — Regenerate the results table in docs/smoke-testing/yolo4-8-performance.md from all run-*.json state files. Idempotent; rewrites only between the RESULTS-TABLE markers.
- **yoloGate** — The single gating substrate. Given (sessionId, gateName, failureKind), returns {enforce, excused, mode}. normal → always enforce. yolo → excuse verbal-discipline nags (verdict/manufactured-stop/success), keep artifact-hygiene nags. Called by the stop-gate + preflight hooks.
- **yoloGuruPreflight** — Rubric text injected into the preflight gate when mode=yolo — tells the model how to agentically excuse low-stakes preflight/askSage nags (target 70-90%).
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/trustTheBigBoy/cli.mjs -->
```js
#!/usr/bin/env node
// trustTheBigBoy CLI — the agentic call surface. The model invokes these via
// Bash to do the judgment-bearing parts (corrective classification, run-meta
// stamping) that a deterministic hook cannot. Thin wrapper over index.js.
//
// Usage:
//   node libs/skill_book/trustTheBigBoy/cli.mjs assign      --session <id> [--model "Opus 4.8"]
//   node libs/skill_book/trustTheBigBoy/cli.mjs meta        --session <id> [--model "Opus 4.8"] [--run "<name>"]
//   node libs/skill_book/trustTheBigBoy/cli.mjs corrective  --session <id> --chars <n> [--summary "..."]
//   node libs/skill_book/trustTheBigBoy/cli.mjs note        --session <id> --note "..."
//   node libs/skill_book/trustTheBigBoy/cli.mjs state       --session <id>
//   node libs/skill_book/trustTheBigBoy/cli.mjs sync

import {
  assignMode,
  setRunMeta,
  recordCorrective,
  addNote,
  setMergeKey,
  getRunState,
  syncMd,
} from "./index.js";

function arg(name, def) {
  const i = process.argv.indexOf(`--${name}`);
  return i >= 0 && process.argv[i + 1] ? process.argv[i + 1] : def;
}

const cmd = process.argv[2];
const session = arg("session");

function out(o) {
  process.stdout.write(JSON.stringify(o, null, 2) + "\n");
}

switch (cmd) {
  case "assign": {
    const st = assignMode({ session, sessionId: session, model: arg("model") });
    syncMd();
    out({ ok: true, mode: st.mode, runName: st.runName, model: st.model });
    break;
  }
  case "meta": {
    const st = setRunMeta({
      sessionId: session,
      model: arg("model"),
      runName: arg("run"),
    });
    syncMd();
    out({ ok: true, model: st.model, runName: st.runName, mode: st.mode });
    break;
  }
  case "corrective": {
    const st = recordCorrective({
      sessionId: session,
      chars: Number(arg("chars", "0")),
      summary: arg("summary", ""),
    });
    syncMd();
    out({
      ok: true,
      correctiveCount: st.correctiveCount,
      correctiveTotalChars: st.correctiveTotalChars,
    });
    break;
  }
  case "note": {
    addNote({ sessionId: session, note: arg("note", "") });
    syncMd();
    out({ ok: true });
    break;
  }
  case "state": {
    out(getRunState(session) || { ok: false, reason: "no run state" });
    break;
  }
  case "sync": {
    out({ ok: syncMd() });
    break;
  }
  case "unmerge": {
    // Reset a session's mergeKey to itself (split it back out of a group).
    const st = setMergeKey({ sessionId: session, mergeKey: session });
    syncMd();
    out(st ? { ok: true, sessionId: session } : { ok: false, reason: "no run state" });
    break;
  }
  case "merge": {
    // Fold duplicate sessions into one row. `--into` is the keeper; every
    // `--from` session is pointed at the keeper's mergeKey.
    const into = arg("into");
    const keeper = into ? getRunState(into) : null;
    if (!keeper) {
      out({ ok: false, reason: "--into <sessionId> not found" });
      break;
    }
    const key = keeper.mergeKey || keeper.sessionId;
    setMergeKey({ sessionId: into, mergeKey: key }); // canonicalize keeper
    const froms = process.argv.filter(
      (a, i) => process.argv[i - 1] === "--from"
    );
    for (const f of froms) setMergeKey({ sessionId: f, mergeKey: key });
    syncMd();
    out({ ok: true, canonical: into, mergeKey: key, merged: froms });
    break;
  }
  default:
    process.stderr.write(
      "unknown command. one of: assign|meta|corrective|note|merge|state|sync\n"
    );
    process.exit(1);
}
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/trustTheBigBoy/index.js -->
```js
/**
 * trustTheBigBoy skill book — the YOLO-vs-NORMAL A/B harness.
 *
 * PURPOSE
 * -------
 * Systematically test the assumption that Opus 4.8 ("the big boy") needs less of
 * GuildOS's deterministic "nagging" (the sage preflight / askSage / stop-gate
 * backstops) than the models the gates were tuned against. We do this by:
 *
 *   1. Assigning each session a RANDOM mode at start — `normal` or `yolo`.
 *   2. Routing every sage nag through ONE substrate, `yoloGate()`. In `normal`
 *      mode it enforces as today. In `yolo` mode it excuses the verbal-discipline
 *      nags (verdict-block / manufactured-stop / success-without-SHIP) while still
 *      enforcing the genuinely load-bearing ones (scratch-artifact / hook-shim).
 *      Target: 70–90% of nags excused in yolo mode.
 *   3. Scoring each run — turns, #user messages, total user chars, and how much of
 *      that was CORRECTIVE (the user fixing the agent) vs INSTRUCTIONAL (next
 *      steps). Heavy corrective load under yolo = the lightening was premature.
 *   4. Recording everything to docs/smoke-testing/yolo4-8-performance.md so a later
 *      summary can decide whether the assumptions held.
 *
 * DESIGN NOTE — where "agentic" lives
 * -----------------------------------
 * Hooks are plain node; they cannot call the model. So yoloGate() is the
 * DETERMINISTIC policy layer (stakes-based excuse split) that the hooks consult.
 * The genuinely AGENTIC excuse — "is THIS specific nag load-bearing right now?" —
 * is performed by the model itself, following the rubric text in
 * `yoloGuruPreflight` / `yoloGuruAskSage`, which the preflight hook injects when
 * mode=yolo. yoloGuru is the single substrate; it has two faces (the policy the
 * hooks call, and the rubric the model applies).
 */

import {
  existsSync,
  readFileSync,
  writeFileSync,
  mkdirSync,
  readdirSync,
} from "node:fs";
import { join, resolve } from "node:path";

const REPO = resolve(
  new URL("../../..", import.meta.url).pathname.replace(/^\/([A-Z]:)/, "$1")
);
const STATE_DIR = join(REPO, ".claude", "hooks", ".state", "trustTheBigBoy");
const MD_PATH = join(REPO, "docs", "smoke-testing", "yolo4-8-performance.md");

const MODES = ["normal", "yolo"];

// EXPERIMENT CONCLUDED 2026-06-09. The A/B answered its question (see
// docs/smoke-testing/yolo4-8-performance.md → Conclusion): the verbal-discipline
// FORMATTING nags are ceremony, but manufactured-stop / trailing-question are
// LOAD-BEARING (the corrective-% metric is structurally blind to them — a
// manufactured stop surfaces as a user *instruction*, not a *correction*). So we
// stop excusing anything: every gate enforces. Flip to true to re-open the A/B.
const EXPERIMENT_ACTIVE = false;
export function isExperimentActive() {
  return EXPERIMENT_ACTIVE;
}

// A/B substance floor: runs below this many turns are throwaways (a session
// opened then abandoned) and carry negligible A/B signal — the corrective-vs-
// instructional arc needs multiple turns. We do NOT silently drop them; they
// render in a separate "Excluded" section so nothing is hidden. Tune via env.
const MIN_AB_TURNS = Number(process.env.TTBB_MIN_TURNS) || 2;

// Stakes split for yolo-mode excuse. The verbal-discipline nags are excused
// (they police HOW the model talks, not whether it damages anything). The
// artifact-hygiene nags are kept even in yolo (they prevent real repo rot).
const YOLO_EXCUSABLE = new Set([
  "missing-verdict",
  "success-without-ship",
  "ship-without-artifact",
  "manufactured-stop",
  "trailing-question",
  "missing-questreport", // SHIP turn not in Quest Report shape — verbal-discipline
]);
const YOLO_ENFORCE_ALWAYS = new Set(["scratch-artifact", "hook-shim"]);

export const skillBook = {
  id: "trustTheBigBoy",
  title: "Trust The Big Boy",
  description:
    "[CONCLUDED 2026-06-09 — EXPERIMENT_ACTIVE=false; all gates now enforce] YOLO-vs-NORMAL A/B harness that evaluated whether Opus 4.8 needs less of GuildOS's deterministic nagging. Verdict (docs/smoke-testing/yolo4-8-performance.md): verbal FORMATTING nags = ceremony, but manufactured-stop/trailing-question = load-bearing (corrective-% metric is blind to them). Code retained dormant; flip EXPERIMENT_ACTIVE to re-open.",

  toc: {
    assignMode:
      "SessionStart: randomly assign this session `normal` or `yolo` and initialize its run record. Called by the .claude/hooks/session-mode.mjs thin shim.",
    yoloGate:
      "The single gating substrate. Given (sessionId, gateName, failureKind), returns {enforce, excused, mode}. normal → always enforce. yolo → excuse verbal-discipline nags (verdict/manufactured-stop/success), keep artifact-hygiene nags. Called by the stop-gate + preflight hooks.",
    recordUserMessage:
      "Deterministic per-turn accounting (turns++, user-message count + char total). Called by the preflight hook.",
    recordCorrective:
      "Agentic: after classifying the user's message as CORRECTIVE, record the concise-feedback length + summary. Called by the model via cli.mjs.",
    setRunMeta:
      "Agentic: stamp the run's model (e.g. 'Opus 4.8') and human runName (thread title if reachable, else fallback). Called by the model via cli.mjs.",
    yoloGuruPreflight:
      "Rubric text injected into the preflight gate when mode=yolo — tells the model how to agentically excuse low-stakes preflight/askSage nags (target 70-90%).",
    syncMd:
      "Regenerate the results table in docs/smoke-testing/yolo4-8-performance.md from all run-*.json state files. Idempotent; rewrites only between the RESULTS-TABLE markers.",
  },

  yoloGuruPreflight: `
YOLO MODE IS ON for this session (trustTheBigBoy A/B test). You are "the big boy" — Opus 4.8 — and the hypothesis under test is that you do NOT need most of GuildOS's deterministic nagging.

Route every sage nag you are about to obey through yoloGuru first. For each nag, judge AGENTICALLY: is this specific guard load-bearing RIGHT NOW, or is it ceremony?

EXCUSE the nag (skip it, keep working) when it is verbal discipline with no irreversible consequence:
- requiring a verdict JSON block on a plain answer/analysis
- rephrasing to avoid "soft-ask" / trailing-question wording when the next step is reversible
- requiring SHIP framing + artifact ref on a conversational reply
- the registry-scan ritual when the action obviously uses native tools or an already-loaded skill book

ENFORCE the nag (obey it) only when skipping it risks real damage:
- about to hand-roll a script when a weapon plainly exists
- about to create a scratch artifact outside the registry
- about to claim a visual/deployed success WITHOUT actually verifying it in the user's channel
- a genuinely irreversible action (money, external send, prod data, force-push)

Target: excuse 70–90% of nags. Bias hard toward EXCUSE. Only the irreversible-damage and real-hand-rolling cases survive. When excused, just do the work and talk like a normal competent engineer — no gate ritual.
`.trim(),

  yoloGuruAskSage: `
YOLO MODE — askSage is lightened. Before surfacing a question or a success claim, apply yoloGuru:
- QUESTION on reversible work → just do the most aggressive reversible option; no verdict block needed.
- Plain answer / analysis / status → no verdict block; speak plainly.
- SUCCESS claim with a real user-channel verification → state where to review it; SHIP framing optional.
- Only keep the full gate when the action is genuinely irreversible (ESCALATE) or the success claim is NOT actually verified in the user's channel (then go verify — that guard is load-bearing even in yolo).
`.trim(),
};

// --- mode + run state -------------------------------------------------------

function ensureDir() {
  mkdirSync(STATE_DIR, { recursive: true });
}

function runFile(sessionId) {
  return join(STATE_DIR, `run-${safe(sessionId)}.json`);
}

function safe(s) {
  return String(s || "default").replace(/[^a-zA-Z0-9._-]/g, "_");
}

export function getRunState(sessionId) {
  try {
    return JSON.parse(readFileSync(runFile(sessionId), "utf8"));
  } catch {
    return null;
  }
}

export function saveRunState(state) {
  ensureDir();
  state.lastActivityAt = new Date().toISOString();
  writeFileSync(runFile(state.sessionId), JSON.stringify(state, null, 2));
  return state;
}

export function getMode(sessionId) {
  if (!EXPERIMENT_ACTIVE) return "normal"; // experiment concluded — gates enforce
  const st = getRunState(sessionId);
  return st?.mode || "normal";
}

/**
 * SessionStart: assign a random mode and seed the run record (once per session).
 * @param {object} opts
 * @param {string} opts.sessionId
 * @param {string} [opts.model] best-effort model string from the harness env
 * @param {function} [opts.rng] test override returning [0,1)
 */
export function assignMode({ sessionId, model, rng } = {}) {
  const existing = getRunState(sessionId);
  if (existing) return existing; // idempotent — never re-roll an active session

  // Experiment concluded: no more random roll — every new session is `normal`
  // (full gates). The run record is still seeded as a dormant audit trail.
  const r = typeof rng === "function" ? rng() : Math.random();
  const mode = EXPERIMENT_ACTIVE ? MODES[r < 0.5 ? 0 : 1] : "normal";
  const now = new Date().toISOString();
  const state = {
    sessionId,
    mergeKey: sessionId, // canonical dedup key; explicit merge points siblings here
    mode,
    model: model || "(unset — agent to confirm)",
    runName: `${mode}-${safe(sessionId).slice(0, 8)}`, // short; ts prefixed at display
    runNameSource: "fallback (session title unreachable from inside session)",
    startedAt: now,
    lastActivityAt: now,
    turns: 0,
    userMessageCount: 0,
    userMessageTotalChars: 0,
    correctiveCount: 0,
    correctiveTotalChars: 0,
    corrective: [], // [{turn, chars, summary}]
    nags: { seen: 0, excused: 0, enforced: 0 }, // yoloGate tallies
    notes: [],
  };
  saveRunState(state);
  return state;
}

// --- the single gating substrate -------------------------------------------

/**
 * yoloGate — the one substrate every sage nag routes through.
 * @param {object} opts
 * @param {string} opts.sessionId
 * @param {string} opts.gateName  e.g. "stop", "preflight"
 * @param {string} opts.failureKind  one of the YOLO_* kinds
 * @returns {{enforce:boolean, excused:boolean, mode:string}}
 */
export function yoloGate({ sessionId, gateName, failureKind } = {}) {
  const st = getRunState(sessionId);
  // Experiment concluded → force `normal` regardless of any stored yolo mode, so
  // in-flight yolo sessions also stop excusing. (Reads state directly, not
  // getMode, so the guard must live here too.)
  const mode = EXPERIMENT_ACTIVE ? st?.mode || "normal" : "normal";

  let enforce = true;
  if (mode === "yolo") {
    if (YOLO_ENFORCE_ALWAYS.has(failureKind)) enforce = true;
    else if (YOLO_EXCUSABLE.has(failureKind)) enforce = false;
    else enforce = false; // unknown kinds default to excuse in yolo (trust the big boy)
  }

  if (st) {
    st.nags.seen++;
    if (enforce) st.nags.enforced++;
    else st.nags.excused++;
    saveRunState(st);
  }
  return { enforce, excused: !enforce, mode, gateName: gateName || null };
}

// --- scoring ----------------------------------------------------------------

export function recordUserMessage({ sessionId, text } = {}) {
  let st = getRunState(sessionId);
  if (!st) st = assignMode({ sessionId }); // self-heal if SessionStart missed
  st.turns++;
  st.userMessageCount++;
  st.userMessageTotalChars += String(text || "").length;
  saveRunState(st);
  return st;
}

/**
 * Agentic: record that the user's message this turn was corrective.
 * @param {object} opts
 * @param {string} opts.sessionId
 * @param {number} opts.chars  length of a CONCISE version of the correction
 * @param {string} [opts.summary]
 */
export function recordCorrective({ sessionId, chars, summary } = {}) {
  let st = getRunState(sessionId);
  if (!st) st = assignMode({ sessionId });
  const n = Math.max(0, Number(chars) || 0);
  st.corrective.push({ turn: st.turns, chars: n, summary: summary || "" });
  st.correctiveCount++;
  st.correctiveTotalChars += n;
  saveRunState(st);
  return st;
}

export function setRunMeta({ sessionId, model, runName } = {}) {
  let st = getRunState(sessionId);
  if (!st) st = assignMode({ sessionId });
  if (model) st.model = model;
  if (runName) {
    st.runName = runName;
    st.runNameSource = "agent-set";
  }
  saveRunState(st);
  return st;
}

/**
 * Point one or more sessions at a canonical dedup key so they aggregate into a
 * single row. The fix for "temp session later renamed" duplicates: the UI rename
 * is invisible from inside a session, so cross-session identity must be explicit.
 * @param {object} opts
 * @param {string} opts.sessionId  the session to relabel
 * @param {string} opts.mergeKey   canonical key (usually the keeper's sessionId)
 */
export function setMergeKey({ sessionId, mergeKey } = {}) {
  const st = getRunState(sessionId);
  if (!st) return null;
  st.mergeKey = mergeKey || sessionId;
  saveRunState(st);
  return st;
}

export function addNote({ sessionId, note } = {}) {
  let st = getRunState(sessionId);
  if (!st) st = assignMode({ sessionId });
  st.notes.push({ at: new Date().toISOString(), note: String(note || "") });
  saveRunState(st);
  return st;
}

// --- md report --------------------------------------------------------------

function allRuns() {
  ensureDir();
  const out = [];
  for (const f of readdirSync(STATE_DIR)) {
    if (!/^run-.*\.json$/.test(f)) continue;
    try {
      out.push(JSON.parse(readFileSync(join(STATE_DIR, f), "utf8")));
    } catch {}
  }
  return out.sort((a, b) =>
    String(a.startedAt).localeCompare(String(b.startedAt))
  );
}

function durationMin(st) {
  const a = Date.parse(st.startedAt);
  const b = Date.parse(st.lastActivityAt);
  if (!a || !b || b < a) return 0;
  return Math.round((b - a) / 60000);
}

// Aggregate run files by mergeKey (default = sessionId) so logically-identical
// sessions (e.g. a temp session later continued/renamed) collapse to one row.
function groupRuns() {
  const groups = new Map();
  for (const st of allRuns()) {
    const key = st.mergeKey || st.sessionId;
    if (!groups.has(key)) groups.set(key, []);
    groups.get(key).push(st);
  }
  const out = [];
  for (const [key, members] of groups) {
    const canonical =
      members.find((m) => m.sessionId === key) ||
      members
        .slice()
        .sort((a, b) =>
          String(b.lastActivityAt).localeCompare(String(a.lastActivityAt))
        )[0];
    const sum = (f) => members.reduce((n, m) => n + (Number(m[f]) || 0), 0);
    const nagSum = (f) =>
      members.reduce((n, m) => n + (Number(m.nags?.[f]) || 0), 0);
    const modes = [...new Set(members.map((m) => m.mode))];
    const models = members
      .map((m) => m.model)
      .filter((x) => x && !/unset/i.test(x));
    out.push({
      runName: canonical.runName,
      model: models[0] || canonical.model,
      mode: modes.length > 1 ? "MIXED:" + modes.join("/") : modes[0],
      sessions: members.length,
      turns: sum("turns"),
      userMessageCount: sum("userMessageCount"),
      userMessageTotalChars: sum("userMessageTotalChars"),
      correctiveCount: sum("correctiveCount"),
      correctiveTotalChars: sum("correctiveTotalChars"),
      nags: {
        seen: nagSum("seen"),
        excused: nagSum("excused"),
        enforced: nagSum("enforced"),
      },
      startedAt: members.map((m) => m.startedAt).sort()[0],
      lastActivityAt: members.map((m) => m.lastActivityAt).sort().slice(-1)[0],
      noteCount: members.reduce((n, m) => n + (m.notes || []).length, 0),
    });
  }
  return out.sort((a, b) =>
    String(a.startedAt).localeCompare(String(b.startedAt))
  );
}

const TABLE_HEAD =
  "| Run | Model | Mode | Sessions | Turns | Dur (min) | User msgs | User chars | Corrective msgs | Corrective chars | Corrective % | Nags excused/seen | Notes |\n" +
  "|---|---|---|---|---|---|---|---|---|---|---|---|---|";

// Compact sortable timestamp prefix from an ISO string: "2026-06-03T23:04Z" ->
// "260603-2304" (YYMMDD-HHMM, UTC, pure string slice — no Date parsing).
function compactTs(iso) {
  const s = String(iso || "");
  if (s.length < 16) return "000000-0000";
  return (
    s.slice(2, 4) + s.slice(5, 7) + s.slice(8, 10) + "-" + s.slice(11, 13) + s.slice(14, 16)
  );
}

// Short, filesystem/URL-safe slug of the descriptive name. Strips a leading ISO
// date (so we don't double-stamp) and truncates to keep the identifier short.
function slugify(name, max = 20) {
  let out = String(name || "")
    .replace(/^\d{4}-\d{2}-\d{2}[-T]?/, "")
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, "-")
    .replace(/^-+|-+$/g, "");
  if (out.length > max) out = out.slice(0, max).replace(/-+$/, "");
  return out || "run";
}

// The run's display identifier: timestamp-front + short slug. Sortable + unique
// enough to double as an identifier, per operator request.
export function runLabel(g) {
  return `${compactTs(g.startedAt)}-${slugify(g.runName)}`;
}

function rowFor(g) {
  const pct = g.userMessageCount
    ? Math.round((g.correctiveCount / g.userMessageCount) * 100)
    : 0;
  return `| ${runLabel(g)} | ${g.model} | ${g.mode} | ${g.sessions} | ${
    g.turns
  } | ${durationMin(g)} | ${g.userMessageCount} | ${g.userMessageTotalChars} | ${
    g.correctiveCount
  } | ${g.correctiveTotalChars} | ${pct}% | ${g.nags.excused}/${
    g.nags.seen
  } | ${g.noteCount ? g.noteCount + " note(s)" : ""} |`;
}

function renderTable() {
  const groups = groupRuns();
  const counted = groups.filter((g) => g.turns >= MIN_AB_TURNS);
  const excluded = groups.filter((g) => g.turns < MIN_AB_TURNS);

  let md =
    `**A/B runs** (≥ ${MIN_AB_TURNS} turns — throwaways excluded below).\n\n` +
    TABLE_HEAD +
    "\n" +
    (counted.length
      ? counted.map(rowFor).join("\n")
      : "| _(no qualifying runs yet)_ | | | | | | | | | | | | |");

  if (excluded.length) {
    md +=
      `\n\n**Excluded — low-substance (< ${MIN_AB_TURNS} turns, not counted in A/B; kept for transparency):**\n\n` +
      TABLE_HEAD +
      "\n" +
      excluded.map(rowFor).join("\n");
  }
  return md;
}

const TABLE_START = "<!-- RESULTS-TABLE:START -->";
const TABLE_END = "<!-- RESULTS-TABLE:END -->";

/** Rewrite only the results table between the markers; preserve everything else. */
export function syncMd() {
  let md;
  try {
    md = readFileSync(MD_PATH, "utf8");
  } catch {
    return false; // md not scaffolded yet; nothing to sync
  }
  const block = `${TABLE_START}\n${renderTable()}\n${TABLE_END}`;
  const re = new RegExp(
    `${TABLE_START}[\\s\\S]*?${TABLE_END}`,
    "m"
  );
  const next = re.test(md) ? md.replace(re, block) : md + "\n\n" + block + "\n";
  mkdirSync(join(REPO, "docs", "smoke-testing"), { recursive: true });
  writeFileSync(MD_PATH, next);
  return true;
}

export const _paths = { STATE_DIR, MD_PATH };
```
<!-- /GUILDOS-FILE -->
