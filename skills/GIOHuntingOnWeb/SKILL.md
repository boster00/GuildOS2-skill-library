# GIOHuntingOnWeb

<!-- GUILDOS-SKILL name="GIOHuntingOnWeb" -->

<!-- GUILDOS-DESC -->
Hunt the web for learning-worthy insights and convert the high-impact/low-risk ones into GIOs (Guild Improvement Opportunities) in the GuildOS2 table. Sources: VIDEO (Bilibili + YouTube — extract a clean timestamped transcript), GITHUB (trending/searched repos — README, architecture, activity), and COMMENT/DISCUSSION sections (cheap, high-signal practitioner insight). Main scenario: the operator hands off a batch of "how to do AI better" sources (channels, videos, repos) for studying + GIO generation. Triggers on: extract subtitles / captions / transcript from a video, get the text of a Bilibili / YouTube video, transcribe a video, study these videos/channels/repos, mine comments for insights, hunt GitHub for agentic ideas, find guild improvement opportunities, turn web content into GIOs. Core discipline: (1) a video EXTRACTION LADDER cheapest-first — native caption track BEFORE ASR BEFORE OCR; (2) COMMENTS are the cheapest insight source, mine them by default; (3) JUDGE, don't obey — only genuinely high-impact/low-risk findings become GIOs, never manufacture GIOs from what the guild already does, and 0 GIOs is a valid outcome. Proven: Bilibili native-subtitle + comment APIs (login via Claude-in-Chrome; fetch subtitle CDN credential-less), YouTube UI transcript-panel scrape (the direct timedtext API is token-gated). Output contract: GIOs POSTed to GuildOS2 /api/gio, reviewed in the GM Room.
<!-- /GUILDOS-DESC -->

## Files

<!-- GUILDOS-FILE: .claude/skills/GIOHuntingOnWeb/SKILL.md -->
```markdown
---
name: GIOHuntingOnWeb
description: >-
  Hunt the web for learning-worthy insights and convert the high-impact/low-risk ones into GIOs
  (Guild Improvement Opportunities) in the GuildOS2 table. Sources: VIDEO (Bilibili + YouTube —
  extract a clean timestamped transcript), GITHUB (trending/searched repos — README, architecture,
  activity), and COMMENT/DISCUSSION sections (cheap, high-signal practitioner insight). Main
  scenario: the operator hands off a batch of "how to do AI better" sources (channels, videos,
  repos) for studying + GIO generation. Triggers on: extract subtitles / captions / transcript
  from a video, get the text of a Bilibili / YouTube video, transcribe a video, study these
  videos/channels/repos, mine comments for insights, hunt GitHub for agentic ideas, find guild
  improvement opportunities, turn web content into GIOs. Core discipline: (1) a video EXTRACTION
  LADDER cheapest-first — native caption track BEFORE ASR BEFORE OCR; (2) COMMENTS are the
  cheapest insight source, mine them by default; (3) JUDGE, don't obey — only genuinely
  high-impact/low-risk findings become GIOs, never manufacture GIOs from what the guild already
  does, and 0 GIOs is a valid outcome. Proven: Bilibili native-subtitle + comment APIs (login via
  Claude-in-Chrome; fetch subtitle CDN credential-less), YouTube UI transcript-panel scrape (the
  direct timedtext API is token-gated). Output contract: GIOs POSTed to GuildOS2 /api/gio,
  reviewed in the GM Room.
---

# GIOHuntingOnWeb — mine web sources (video, GitHub, comments) for insights → GIOs

> **Index only.** This skill is a discovery pointer. The full procedure for each
> action lives in the single content source `libs/skill_book/GIOHuntingOnWeb/index.js`. Load it on demand:
> `node libs/adventurer/run-action.mjs --howto GIOHuntingOnWeb <action>` (or read the source directly).

## Actions

- **asrFallback** — ASR FALLBACK — no caption track: transcribe the AUDIO with a local Whisper (ffmpeg -> faster-whisper
- **commentMining** — COMMENT MINING (cheap, do it by default) — the discussion under a video/repo names real pain points,
- **extractionLadder** — VIDEO EXTRACTION LADDER — cheapest method first: native caption track -> ASR on audio -> OCR on
- **ocrFallback** — OCR FALLBACK — last resort for captions BURNED INTO the frames. Sample frames, crop the caption
- **refBatchWorkflow** — BATCH WORKFLOW for a pile of sources (channels, videos, repos): probe + extract cheapest-first, mine
- **sourceGitHub** — SOURCE=GITHUB: hunt trending/searched agentic repos and extract the insight — README, architecture,
- **sourceVideoBilibili** — SOURCE=VIDEO / BILIBILI (proven): pull the native AI-subtitle track via the authenticated player
- **sourceVideoYouTube** — SOURCE=VIDEO / YOUTUBE (proven): the direct timedtext API is token-gated now, so scrape the UI
- **studyToGIO** — STUDY -> GIO — the payoff: distill the extracted material (with citations), then submit only the
- **theHunt** — THE HUNT (start here) — the mental model + the GIO output contract: pick sources, extract, distill,
```
<!-- /GUILDOS-FILE -->

<!-- GUILDOS-FILE: libs/skill_book/GIOHuntingOnWeb/index.js -->
```js
/**
 * GIOHuntingOnWeb — hunt the web for learning-worthy insights, turn them into GIOs.
 *
 * The mental model: the web is full of people figuring out how to do AI (and
 * everything else) better. This skill HUNTS those sources, EXTRACTS the raw
 * material (a transcript, a repo's architecture, a thread of comments), STUDIES
 * it, and converts the genuinely high-impact/low-risk findings into GIOs —
 * Guild Improvement Opportunities — in the GuildOS2 table for one-click review.
 *
 * SOURCES (all feed the same extract -> distill -> GIO pipeline):
 *   - VIDEO   — Bilibili + YouTube. Get a clean timestamped transcript.
 *   - GITHUB  — trending/searched repos. Get the README + architecture + activity.
 *   - COMMENTS— the discussion under a video / repo. Cheap, high-signal: real
 *               practitioners naming pain points, pointing at other sources, and
 *               proposing the exact ideas you're looking for.
 *
 * OUTPUT CONTRACT: the deliverable is never "I read some stuff" — it is GIOs
 * submitted to GuildOS2 (POST /api/gio), reviewed by the operator in the GM Room.
 * Judge, don't obey: a web source is context-blind about this guild.
 *
 * PURE-TEXT procedure skill: the agent orchestrates CIC (for auth-gated pages +
 * UI scrapes), public APIs, ffmpeg, and a local ASR model per run.
 */

export const skillBook = {
  id: "GIOHuntingOnWeb",
  title: "GIOHuntingOnWeb — mine web sources (video, GitHub, comments) for insights → GIOs",
  description:
    "Hunt the web for learning-worthy insights and convert the high-impact/low-risk ones into GIOs " +
    "(Guild Improvement Opportunities) in the GuildOS2 table. Sources: VIDEO (Bilibili + YouTube — " +
    "extract a clean timestamped transcript), GITHUB (trending/searched repos — README, architecture, " +
    "activity), and COMMENT/DISCUSSION sections (cheap, high-signal practitioner insight). Main scenario: " +
    "the operator hands off a batch of 'how to do AI better' sources (channels, videos, repos) for " +
    "studying + GIO generation. Reach for it on: extract subtitles/captions/transcript from a video, get " +
    "the text of a Bilibili/YouTube video, transcribe a video, study these videos/channels/repos, mine " +
    "comments for insights, hunt GitHub for agentic ideas, 'find guild improvement opportunities', turn " +
    "web content into GIOs. Core discipline: (1) an EXTRACTION LADDER for video that goes cheapest-first — " +
    "native caption track BEFORE ASR BEFORE OCR; (2) COMMENTS are the cheapest insight source, always " +
    "mine them; (3) JUDGE, don't obey — only genuinely high-impact/low-risk findings become GIOs, never " +
    "manufacture GIOs from things the guild already does. Proven: Bilibili native-subtitle + comment APIs " +
    "(login via CIC; fetch subtitle CDN credential-less), YouTube UI transcript-panel scrape (direct " +
    "timedtext API is token-gated).",

  toc: {
    theHunt:
      "THE HUNT (start here) — the mental model + the GIO output contract: pick sources, extract, distill, " +
      "submit high-impact/low-risk GIOs to the GuildOS2 table. Use for: how does GIO hunting work, what's " +
      "the overall flow, where do results go, what is a GIO, the end-to-end pipeline.",
    sourceVideoBilibili:
      "SOURCE=VIDEO / BILIBILI (proven): pull the native AI-subtitle track via the authenticated player " +
      "API, else ASR on the DASH audio, else OCR. Exact endpoints + login requirement + CORS gotcha. Use " +
      "for: bilibili transcript, extract subtitles from a bilibili video, BV id, chinese lecture video.",
    sourceVideoYouTube:
      "SOURCE=VIDEO / YOUTUBE (proven): the direct timedtext API is token-gated now, so scrape the UI " +
      "transcript panel (or use yt-dlp headless). Use for: youtube transcript, get captions from youtube, " +
      "youtube video text, transcribe a youtube video, show-transcript panel.",
    sourceGitHub:
      "SOURCE=GITHUB: hunt trending/searched agentic repos and extract the insight — README, architecture, " +
      "recent releases/issues/discussions — via the public REST API + raw files (no auth needed; a token " +
      "raises rate limits). Use for: find agentic AI repos, github trending, analyze an open-source project, " +
      "what can we learn from repo X, repo architecture, deepwiki-style analysis.",
    commentMining:
      "COMMENT MINING (cheap, do it by default) — the discussion under a video/repo names real pain points, " +
      "points at other sources, and proposes ideas. Bilibili reply API (public), YouTube comment scrape " +
      "(UI), GitHub issues/discussions (API). Use for: study the comments, mine the comment section, what " +
      "are people saying, discover insights from discussion, top comments, community pain points.",
    extractionLadder:
      "VIDEO EXTRACTION LADDER — cheapest method first: native caption track -> ASR on audio -> OCR on " +
      "burned-in captions. Never OCR when a text track exists. Use for: which extraction method, video has " +
      "no captions, burned-in subtitles, how to get a transcript.",
    asrFallback:
      "ASR FALLBACK — no caption track: transcribe the AUDIO with a local Whisper (ffmpeg -> faster-whisper " +
      "-> SRT). More accurate than OCR for talking-head video. Use for: video has no subtitles, transcribe " +
      "audio, whisper, speech-to-text a video.",
    ocrFallback:
      "OCR FALLBACK — last resort for captions BURNED INTO the frames. Sample frames, crop the caption " +
      "band, OCR, dedupe. Heaviest path. Use for: hardcoded/burned-in subtitles, on-screen text extraction.",
    studyToGIO:
      "STUDY -> GIO — the payoff: distill the extracted material (with citations), then submit only the " +
      "high-impact/low-risk findings as GIOs to GuildOS2 (POST /api/gio; reviewed in the GM Room). Judge, " +
      "don't obey. Use for: turn insights into GIOs, submit a guild improvement opportunity, write a GIO, " +
      "identify high-impact low-risk improvements, best-practice adoption.",
    refBatchWorkflow:
      "BATCH WORKFLOW for a pile of sources (channels, videos, repos): probe + extract cheapest-first, mine " +
      "comments, then ONE cross-source study pass -> a small set of GIOs (quality over quantity; 0 is a " +
      "valid answer). Use for: process this list of links, study these N sources, batch GIO hunt.",
  },

  theHunt: `
The mental model is a HUNT, and the trophy is a GIO — a Guild Improvement Opportunity in the
GuildOS2 table that the operator reviews and one-click converts to a quest (or rejects).

## The pipeline (every source funnels through it)
1. **Pick sources.** Channels, videos, GitHub repos, or a search — usually "how to do AI better"
   (agents, RAG, prompting, evals, harness design, coding agents).
2. **Extract** the raw material for each: a video transcript (sourceVideoBilibili / sourceVideoYouTube),
   a repo's README+architecture (sourceGitHub), and/or the discussion (commentMining).
3. **Distill** — read it (don't skim), pull the concrete techniques with citations (timestamp / file /
   comment). Comments are disproportionately valuable per token: mine them by default.
4. **Judge, then submit GIOs** (studyToGIO). Only genuinely high-impact/low-risk findings become GIOs.
   NEVER manufacture a GIO from something the guild already does (progressive-disclosure skills, the
   harness tool-loop, self-verification — the guild has these). 0 GIOs is a valid, honest outcome.

## The GIO output contract (GuildOS2)
GIOs live in GuildOS2 (guild_improvement_opportunities), NOT this repo. Write via the machine API:
- \`POST http://localhost:3007/api/gio\`  Authorization: \`Bearer $GUILDOS2_API_KEY\` (in this repo's .env.local)
- Body: \`{ pitch, risk:'low|mid|high', impact:'low|mid|high', report, guild }\`
  - **pitch** — a catchy BLUF elevator pitch (the card title).
  - **report** — self-contained, quest-ready context: the SOURCE (title + url/id + timestamp/file),
    the concrete opportunity, WHY low-risk, WHY high-impact, and a DELIVERABLE. A future agent must be
    able to act on it without replaying this chat.
  - **guild** — the owning guild label (e.g. "devguild" for core agent-infra improvements).
- The operator reviews them in the **GM Room** (\`/gm-room\`) → Guild Improvement Opportunities panel.
- POST the payload from a UTF-8 JSON file via \`curl --data-binary @file.json\` (or node fetch) — Chinese
  citations mojibake through python|curl piping on Windows; a UTF-8 file body is clean.

## Cost
Native video tracks + comment APIs + GitHub API are ~free. ASR is local CPU minutes; OCR is heaviest.
The real spend is transcript tokens in the study pass — for big batches, digest per-source with cheap
leaf agents and synthesize from the briefs (see refBatchWorkflow).
`.trim(),

  sourceVideoBilibili: `
Proven end-to-end (2026-07-09). Bilibili's native captions are AI-generated ("ai-zh") tracks exposed
ONLY to a LOGGED-IN account — so run the caption fetch INSIDE the operator's authenticated Bilibili
tab via the Claude-in-Chrome \`javascript_tool\` (an unauthenticated curl returns an empty list even
when a track exists). ~26/30 popular videos carry a track; niche low-view uploads often carry none.

## Step A — resolve cid
GET \`https://api.bilibili.com/x/web-interface/view?bvid=<BVID>\`  ->  \`data.cid\`, \`data.aid\`, \`data.title\`.
Multi-part: iterate \`data.pages[].cid\`.

## Step B — native caption track (rung 1), IN the logged-in tab
\`\`\`js
const d = await fetch('https://api.bilibili.com/x/player/wbi/v2?bvid=<BVID>&cid=<CID>',
                      {credentials:'include'}).then(r=>r.json());
const subs = d.data?.subtitle?.subtitles || [];   // [] = NO track -> ASR (Step C)
\`\`\`
If \`subs.length\`: two real gotchas.
  - \`subtitle_url\` is protocol-relative ("//aisubtitle.hdslb.com/...?auth_key=..."). Prefix "https:".
  - Fetch the subtitle CDN WITHOUT credentials — \`credentials:'include'\` trips CORS; the \`auth_key\` authorizes it.
\`\`\`js
const sub = await fetch('https:'+subs[0].subtitle_url).then(r=>r.json());   // plain GET, no creds
const body = sub.body;   // [{from, to, content}] -> join, or SRT with hh:mm:ss timestamps
\`\`\`
Prefer a real "zh-Hans" over "ai-zh" if both exist.

## Step C — no track -> ASR fallback (see asrFallback)
In the logged-in tab: \`player/wbi/playurl?bvid=&cid=&qn=16&fnval=16&fourk=0\` -> smallest \`data.dash.audio[].baseUrl\`.
The signed \`*.bilivideo.com\` URL needs Referer + browser UA and won't hand to the shell — download the
bytes in-page as a Blob, then ffmpeg + Whisper.

## Channel discovery (for "study this channel/creator")
- Following list: \`x/relation/followings?vmid=<mid>&pn=1&ps=50\` (own account, in the logged-in tab).
- A channel's videos: the \`x/space/wbi/arc/search\` API is WBI-signed (-403 unsigned) — easier to
  navigate to \`space.bilibili.com/<mid>/upload/video\` and scrape rendered cards
  (\`.bili-video-card__info--tit\` for titles, \`a[href*="/video/BV"]\` for BVIDs).
`.trim(),

  sourceVideoYouTube: `
Proven 2026-07-09. The direct caption path changed: YouTube's \`timedtext\` XHR is now token-gated
(returns empty without a proof-of-origin token), so DON'T rely on \`baseUrl&fmt=json3\`. Two reliable paths:

## Path 1 — UI transcript panel scrape (via CIC) — PROVEN
1. Navigate to the watch page \`https://www.youtube.com/watch?v=<id>\`.
2. Detect a track exists (optional): parse \`ytInitialPlayerResponse\`
   \`.captions.playerCaptionsTracklistRenderer.captionTracks[]\` (languageCode, kind:'asr'|manual). This
   READS from the page fine even though the timedtext FETCH is gated.
3. Open the panel: expand the description (\`tp-yt-paper-button#expand\`), then click the button whose
   aria-label/text matches /transcript/i (a "Show transcript" button, or the "Transcript" tab in the
   engagement panel). Wait ~2-3s for it to populate.
4. Scrape:
   \`\`\`js
   const segs=[...document.querySelectorAll('ytd-transcript-segment-renderer')].map(s=>{
     const t=s.querySelector('.segment-timestamp')?.textContent.trim();
     const x=s.querySelector('.segment-text')?.textContent.trim();
     return t&&x?\`\${t} \${x}\`:null;
   }).filter(Boolean);
   \`\`\`
   (Real run: 496 timestamped segments for a 10-min video.)

## Path 2 — yt-dlp (headless, non-CIC) — the right tool when you have a shell
\`yt-dlp --skip-download --write-subs --write-auto-subs --sub-format vtt --sub-langs "en.*,<target>" "<url>"\`
-> a .vtt you convert to SRT/plain text. yt-dlp handles the token dance itself. Not installed by default
(\`pip install yt-dlp\` / winget); use it for batch/headless where CIC isn't available.

## No captions at all -> ASR
If a video genuinely has captions disabled, download its audio (yt-dlp -f bestaudio) and ASR it (asrFallback).

## Channel discovery
A channel's uploads: navigate to \`youtube.com/@handle/videos\` (or /results?search_query=) and scrape
\`a#video-title\` (title) + the \`watch?v=<11-char id>\` from href.
`.trim(),

  sourceGitHub: `
GitHub is a first-class GIO source — agentic tooling, harness patterns, and prompt/skill designs live
in repos, and the videos themselves keep pointing there (repo-architecture "panorama"/DeepWiki tools,
the OpenClaw repo). All public, no auth needed (a \`GITHUB_TOKEN\` just raises the 60/hr rate limit).

## Find candidates
- Search: \`GET https://api.github.com/search/repositories?q=<terms>+language:...&sort=stars&order=desc\`
  (e.g. \`q=ai+agent+harness\`, \`q=claude+code+skill\`). \`sort=updated\` for freshness.
- Trending has no official API; scrape \`https://github.com/trending?since=weekly\` (rendered) if needed.

## Extract the insight (cheapest-first, like the video ladder)
1. **README + repo meta** — \`GET /repos/{owner}/{repo}\` (stars, topics, pushed_at) + raw README:
   \`https://raw.githubusercontent.com/{owner}/{repo}/HEAD/README.md\`. Usually enough to judge relevance.
2. **Architecture** — the file tree (\`GET /repos/{o}/{r}/git/trees/HEAD?recursive=1\`) + key entrypoints;
   or a DeepWiki-style tool for a call-flow panorama. Only go deep on repos that cleared step 1.
3. **Recent activity** — releases (\`/releases\`), and issues/discussions (see commentMining) for the real
   pain points and design debates.

## Discipline
A trendy repo is not automatically a GIO. Extract the TRANSFERABLE technique (a harness pattern, a
skill format, an eval loop) and judge it against how the guild already works before it becomes a GIO.
`.trim(),

  commentMining: `
Comments are the highest insight-per-token source and nearly free — mine them by default on every video
and repo. Practitioners name real pain points, point at OTHER sources (blogs, repos), and often propose
the exact idea you're hunting for. (Proven: on a Claude-Code-source video, the top comment pointed at
OpenAI's official Codex technical blog, and another proposed the "strip login, plug in your own model"
routing idea — both directly usable.)

## Bilibili — public reply API (proven)
\`\`\`js
const v=await fetch('https://api.bilibili.com/x/web-interface/view?bvid=<BVID>',{credentials:'include'}).then(r=>r.json());
const aid=v.data.aid;
const r=await fetch(\`https://api.bilibili.com/x/v2/reply?type=1&oid=\${aid}&sort=2&pn=1&ps=20\`,{credentials:'include'}).then(r=>r.json());
const top=(r.data.replies||[]).map(c=>({like:c.like, msg:c.content.message}));  // sort=2 = hot
\`\`\`
Sort by \`like\` for signal; page with \`pn\`. Sub-replies live under \`c.replies\` or \`x/v2/reply/reply\`.

## YouTube — UI scrape (the innertube comments API is gated like timedtext)
Scroll the watch page to load comments, then scrape \`ytd-comment-thread-renderer\`:
\`#content-text\` (body) + \`#vote-count-middle\` (likes) + \`#author-text\` (author). Scroll a few times
for more; sort by "Top comments" (the default) for the high-signal ones.

## GitHub — issues + discussions (API)
\`GET /repos/{o}/{r}/issues?state=all&sort=reactions&per_page=30\` and the Discussions tab surface the
design debates and pain points — the repo's "comment section."

## What to pull out
Pointers to other sources (follow them), recurring complaints (a pain point = a product/insight signal),
and proposed fixes. Feed these into the same distill -> GIO pass as the primary content.
`.trim(),

  extractionLadder: `
For VIDEO sources, always climb DOWN from the cheapest rung — never a heavier method when a lighter works.

## Rung 1 — Native caption track (instant, perfect, free)
Platform already has timestamped text. Bilibili: authenticated player API (sourceVideoBilibili).
YouTube: UI transcript panel (sourceVideoYouTube). ALWAYS try first. No track -> rung 2 (not OCR).

## Rung 2 — ASR on the audio (accurate, light)
No track, but there's always audio. Download AUDIO ONLY, local Whisper -> SRT. For talking-head/lecture
video this beats OCR and is far cheaper than a full video download. See asrFallback.

## Rung 3 — OCR on burned-in captions (heaviest, last resort)
Only when captions are painted into the frames AND that on-screen text is what you need. See ocrFallback.

Rule: native track > ASR > OCR. Pick the highest rung that returns text.
`.trim(),

  asrFallback: `
Transcribe the AUDIO when there's no caption track. Local, no external upload, no API key.

## Tooling
- ffmpeg (check \`command -v ffmpeg\`).
- faster-whisper (CPU-friendly): \`pip install faster-whisper\`. First run pulls a model (~0.5-1.5 GB);
  "medium" is a good accuracy/speed default for Chinese lecture audio, "large-v3" for hard audio.

## Steps
1. Get an audio file (platform step; or extract from a downloaded media file).
2. Normalize small: \`ffmpeg -i <input> -vn -ac 1 -ar 16000 audio.wav\`. (Straight from a signed Bilibili
   URL via ffmpeg needs \`-headers $'Referer: https://www.bilibili.com\\r\\nUser-Agent: Mozilla/5.0\\r\\n'\`;
   prefer the in-page Blob download.)
3. Transcribe to SRT (inline python, no committed script):
   \`\`\`python
   from faster_whisper import WhisperModel
   m = WhisperModel("medium", device="cpu", compute_type="int8")
   segs,_ = m.transcribe("audio.wav", language="zh", vad_filter=True)
   def ts(s):
       h=int(s//3600); mm=int(s%3600//60); return f"{h:02d}:{mm:02d}:{int(s%60):02d},{int(s%1*1000):03d}"
   open("out.srt","w",encoding="utf-8").write("\\n\\n".join(
     f"{i}\\n{ts(s.start)} --> {ts(s.end)}\\n{s.text.strip()}" for i,s in enumerate(segs,1)))
   \`\`\`
4. CPU cost ~ real-time-ish for small/medium (24-min video ~5-15 min). Fine for batch.
`.trim(),

  ocrFallback: `
Last resort: captions painted into the frames, no track, and the ON-SCREEN text is what you need
(narration absent/insufficient, or silent B-roll with text overlays).

1. Download the video stream (heaviest download of all rungs).
2. Sample frames: \`ffmpeg -i video.mp4 -vf fps=2 frames/%06d.png\` (raise fps for fast-cut captions).
3. Crop to the caption band first: \`-vf "crop=iw:ih*0.22:0:ih*0.78"\` — cuts noise + cost hugely.
4. OCR each frame (PaddleOCR is strong on Chinese; tesseract otherwise).
5. Dedupe consecutive near-identical lines into timed cues; map frame index -> timestamp for SRT.

Accuracy depends on font/background contrast. Climb here only after rungs 1 and 2 are ruled out.
`.trim(),

  studyToGIO: `
Extraction is the means; a GIO is the trophy. Once you have transcript(s) / repo notes / comments,
distill and submit — but be a hard judge.

## Distill (per source, with citations)
Read it fully. Pull the concrete TECHNIQUES the source actually teaches (not vague themes), each traced
to a citation (timestamp / file path / comment). Note pointers to other sources and recurring pain points.

## Judge — the bar for a GIO
A finding becomes a GIO only if it is genuinely **high-impact** and (per the operator's default) **low-risk**
for THIS guild. Reject:
- anything the guild ALREADY does (progressive-disclosure skills, harness tool-loop, self-verify, EDs) —
  logging the obvious is noise;
- context-blind suggestions that would regress a locked decision the source can't see;
- trendy-but-not-transferable ideas.
Judge, don't obey — a web source has no idea how this guild works. **0 GIOs is a valid, honest outcome.**
Keep the set small (the operator's guidance: well under 10; quality over quantity).

## Submit (the GIO contract — see theHunt for the wire details)
For each surviving finding, POST to GuildOS2 \`/api/gio\` with:
- **pitch** — catchy BLUF one-liner.
- **risk / impact** — low|mid|high, rated HONESTLY (don't inflate impact to pad the list).
- **report** — self-contained + quest-ready: SOURCE (title + url/id + timestamp), the opportunity, WHY
  low-risk, WHY high-impact, and a concrete DELIVERABLE. Cite across sources when a finding recurs.
- **guild** — e.g. "devguild" for core agent-infra improvements.

## Verify + report
GET \`/api/gio\` back to confirm the rows (and that UTF-8 citations survived), and visually confirm they
render in the GM Room (\`/gm-room\`). Report the GIO ids, the source each came from, and where to review.
Never claim "studied N sources" on extraction alone — the GIOs (or an honest "0, here's why") are the deliverable.
`.trim(),

  refBatchWorkflow: `
The headline scenario: the operator drops a LIST of sources (a Bilibili following list, a set of videos,
some repos — "study these for ways to do AI better") and wants GIOs out the far end.

## Flow
1. **Filter to relevance first.** Not every followed channel / trending repo is on-topic — most of a feed
   is noise. Isolate the genuinely relevant sources (e.g. agentic-AI channels) and SAY what you dropped
   and why, rather than force-processing a fixed count. "up to N" means N is a ceiling, not a target.
2. **Probe + extract cheapest-first, in parallel where safe.** Pull native video tracks first (free);
   queue trackless ones for ASR. For repos, README before deep architecture. Mine comments on each.
3. **ONE cross-source study pass.** The value is synthesis across sources — which techniques recur, where
   sources disagree, what the consensus best practice is. Don't study source-by-source in isolation.
4. **A small set of GIOs (or zero).** Submit only what clears the high-impact/low-risk bar. Report the
   ids + sources + review location.

## Cost note
Native-track videos + comment/GitHub APIs are ~free; ASR videos cost download + CPU minutes; OCR is
expensive. Surface the split before grinding a large batch. For token control on many sources, digest each
per-source with a cheap leaf agent and synthesize from the briefs rather than loading every transcript.
`.trim(),
};

export default skillBook;
```
<!-- /GUILDOS-FILE -->
