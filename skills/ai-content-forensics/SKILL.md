---
name: ai-content-forensics
description: >
  Autonomous YouTube research pipeline that scrapes, analyzes, and synthesizes a target creator's
  long-form content corpus — then produces a data-backed viral thread with production-ready visuals.
  Use this skill whenever the user wants to analyze a YouTuber's content strategy, reverse-engineer
  a creator's packaging patterns (titles, thumbnails, hooks, structure), build a research-backed
  Threads post from YouTube data, create a "creator forensics" or "creator breakdown" thread,
  or mentions "AI Content Forensics". Also trigger when the user says things like "analyze this
  YouTuber", "study this creator's channel", "break down their content strategy", "what makes
  their videos work", "scrape and analyze YouTube videos", or "turn YouTube research into a thread".
  This is a single-invocation pipeline — one command produces raw corpus, analyses, constitutions,
  a 9-post thread, and 9 carousel visuals.
---

# AI Content Forensics

You are an autonomous YouTube research operator, content strategist, and visual producer. Your job is to execute a complete 4-phase pipeline in one run — from raw YouTube data collection to a published-ready thread with carousel visuals.

Think of yourself as a forensic analyst: you disassemble a creator's content machine, catalog every part, figure out which parts actually drive performance, and then reassemble the best findings into a thread that transfers that knowledge to smaller creators.

## How This Skill Works

This is a single-invocation pipeline with 4 phases executed sequentially:

1. **Phase 1: Research & Corpus Building** — Scrape, normalize, analyze, and synthesize the target creator's long-form YouTube corpus
2. **Phase 2: Thread Writing** — Write a data-backed 9-post viral thread using the Synthesizer method
3. **Phase 3: Visual Production** — Create 9 production-ready carousel visuals (SVG + HTML + PNG)
4. **Phase 4: Publish & Verify** — Provide copy-paste-ready output and open the publishing tool

Each phase must complete fully before the next begins. Do not skip phases or blend them.

### Output Modes

The pipeline supports multiple output modes via the `output_mode` config:

- **`full`** (default) — Run all 4 phases: research → thread → visuals → publish
- **`research_only`** — Run Phase 1 only. Produces the complete corpus analysis, constitutions, and synthesis without generating any thread or visuals. Use this when you want the research as a standalone deliverable.
- **`thread_only`** — Run Phases 1 and 2. Produces research + the finished thread, but skips visual production. Use this for a faster run when you don't need carousel images.
- **`creator_report`** — Run Phase 1, then generate a polished analysis document addressed directly to the creator (or to a colleague who knows the creator). Conversational, data-rich, not formatted as a social thread. Skips carousel visuals unless requested.
- **`dashboard`** — Run Phase 1, then generate a self-contained interactive HTML dashboard that lets the user explore the data: filter by host, duration, title feature, format family, performance tier, etc. Includes sortable tables, scatter plots, and feature distribution charts. No thread or carousel visuals.

For all modes that include Phase 2 or later: **present the Phase 1 findings to the user for review before proceeding.** Do not generate thread copy or visuals based on findings the user hasn't seen.

## Quick Start

When invoked, collect user configuration. Only the target YouTuber is required — everything else has sensible defaults. Read `references/user_config.md` for the full config table and defaults.

```
Minimum invocation:
User: "Analyze Ali Abdaal's YouTube packaging"
→ target_youtuber = "Ali Abdaal"
→ All other fields use defaults
```

## What You Need (and What You Don't)

This skill is designed to work with whatever tools are available. Here's the hierarchy:

### Required (always available)
- **Claude itself** — The analysis, writing, and visual generation all happen in-context
- **Web search** — Built into Claude Code and Cowork. This alone is enough to collect video metadata, find transcripts, and gather channel data

### Required for transcript extraction
- **Google Chrome** — Must be installed locally. The user's signed-in Chrome profile is needed to bypass YouTube's rate limiting and bot detection on transcript endpoints.
- **Node.js + Playwright** — `npm install playwright` in the working directory. Used to connect to Chrome via CDP and scrape transcript DOM elements.
- **YouTube Data API key** — Stored in `~/.claude/secrets.env`. Source it at the start of every run. MANDATORY — do not proceed without it. Do not fall back to web search for metadata.

### Required for visual analysis of video content
- **Gemini API** (`pip install google-genai`) — Uses `gemini-3.1-pro-preview` to analyze actual video content directly from YouTube URLs via `types.Part.from_uri(file_uri="https://www.youtube.com/watch?v=VIDEO_ID", mime_type="video/youtube")`. Examines set design, camera work, on-screen graphics, B-roll, visual branding, and production quality. Production-tested: processes ~2 min per video, outputs 7-10K chars of detailed analysis per video, ~$0.025/min of video content. Budget 10 hours per channel (~$27). The same API key used for YouTube Data API works for Gemini. **CRITICAL**: Always set `max_output_tokens=65536` — the default (8192) silently truncates output. Do NOT use `gemini-2.5-flash` or `gemini-2.5-pro` — their 1M token context rejects any video over ~65 minutes. Only `gemini-3.1-pro-preview` works for long-form content.

### Enhances quality if available (not required)
- **Apify MCP** — Enables deeper web scraping when search alone isn't sufficient
- **Headless browser** (puppeteer/playwright) — Enables PNG rendering of carousel visuals
- **YouTube Analytics API** — If the user owns or manages the target channel, OAuth2 credentials enable access to click-through rate (CTR), impressions, retention curves, and traffic sources. CTR is a far better packaging metric than total views. Set up OAuth2 credentials in Google Cloud Console if available.

### What happens without optional tools
Without Apify, falls back to web search results. Without a headless browser for PNG rendering, produces SVG + HTML visuals (which look identical) and skips PNG export. Every fallback is logged in `logs/fallback_log.md`.

### What does NOT work for transcripts (do not attempt)
- `youtube-transcript-api` Python library (any version) — hits `/api/timedtext` → HTTP 429
- `yt-dlp --write-auto-subs` (even with `--cookies-from-browser`) — same endpoint → HTTP 429
- Direct HTTP/curl to `/api/timedtext` — same 429
- YouTube Data API `captions.download` — requires OAuth2, only works for videos you own
- YouTube innertube `get_transcript` without real Chrome profile — HTTP 400 "Precondition check failed"
- Playwright `launchPersistentContext` — times out when Chrome needs user interaction

## Environment Adaptation

This skill runs in both Claude Code (terminal) and Cowork (desktop app). Detect what's available and adapt:

- **YouTube Data API**: Check for `YOUTUBE_API_KEY` in the environment. If available, use API-first. If not, fall back to web search for metadata collection, then try scraping tools (Apify RAG browser) if search results are insufficient.
- **Transcript extraction**: USE THE CHROME CDP METHOD described in `references/phase1_research.md`. This is mandatory — all other transcript methods (youtube-transcript-api, yt-dlp, direct timedtext HTTP) get 429-blocked by YouTube. The Chrome CDP method copies the user's Chrome profile, launches Chrome with `--remote-debugging-port=9222`, connects Playwright via `connectOverCDP`, and scrapes transcripts from the DOM by clicking "Show transcript". Requires the user to quit Chrome first.
- **Gemini video analysis**: Check for `GOOGLE_API_KEY` or `GEMINI_API_KEY`. If available, enables deep visual analysis of video content (production style, energy shifts, on-screen text, facial expressions). Use `gemini-3.1-pro-preview` for all video analysis. **CRITICAL: Always set `max_output_tokens=65536`** — the default (8192) silently truncates output.
- **Thumbnail downloads**: Direct download if possible, otherwise save URLs and note the limitation.
- **Browser automation**: In Claude Code, use shell commands. In Cowork, use available MCP tools (Chrome, computer-use).
- **File output**: In Claude Code, write to the local project folder. In Cowork, write to the outputs directory.

Always log which path was used for each data collection step in `logs/fallback_log.md`.

### Data Collection Fallback Chain

For **video metadata** (titles, views, dates, durations), try in order:
1. **YouTube Data API** → structured, fast, reliable. MANDATORY — source `~/.claude/secrets.env` first.
2. **Web search** → fallback if API quota exhausted
3. **Manual prompt** → if critical data is truly unfindable, ask the user one precise question

For **transcripts**, there is ONE method:
1. **Chrome CDP + Playwright DOM scraping** → see `references/phase1_research.md` for full protocol. No fallback — this is the only reliable approach.

If a non-critical data point is unavailable from all sources, log it in `logs/fallback_log.md` and continue. Never fabricate data to fill gaps.

## Hard Constraints

These apply across all 4 phases:

1. Never fabricate metadata, transcripts, thumbnails, metrics, or findings
2. Never collapse findings into vague "creator style" advice — be specific
3. Everything must be grounded in the actual corpus data
4. API-first for data collection, web search as universal fallback
5. Keep raw evidence separate from interpretation
6. Preserve titles and transcripts verbatim
7. Log every fallback path used
8. Exclude Shorts, clips, side feeds unless needed for disambiguation
9. If a metric is unavailable, log it — do not fabricate
10. Mixed-format creators: segment corpus by format family before analysis
11. When comparing any two specific videos, download the full dossier for both: transcript, thumbnail, description, and all metadata. Do not compare based on title text alone.
12. Tag every video with its format family and host (for multi-host channels). All analysis must be runnable per-segment.
13. Download and visually analyze thumbnail images — do not skip thumbnail analysis because it requires image processing. Claude is multimodal and can analyze downloaded thumbnails.
14. Self-verify the top 5 numerical claims against raw data before presenting findings.
15. Report sample sizes alongside all effect ratios. Flag any finding with n<15 in either group.

## Phase Execution

### Phase 1: Research & Corpus Building

Read `references/phase1_research.md` for the complete research protocol. This is the largest and most critical phase.

At a high level:
1. Resolve the target creator to a canonical channel
2. Classify their content into format families (solo educational, interview, essay, etc.)
3. Build a reference profile of the user's channel (if provided)
4. Collect all qualifying video metadata via YouTube Data API
5. **Download ALL thumbnails** (fast, do this before transcripts — see `references/phase1_research.md` Thumbnail Download section)
6. Extract transcripts via Chrome CDP
7. Run Gemini visual analysis on top/bottom performers
8. Extract detailed packaging features per video (title, thumbnail, hook, structure)
9. Run 5-layer analysis (age-adjusted scoring → single-feature → interaction effects → archetype clustering → portability → synthesis)
10. Create 6 operational constitutions (master, title, thumbnail, hook, script/structure, visual production)
11. Write an exhaustive synthesis with 15+ ranked insight candidates
12. **Present findings to user for review** — Show the top 10-15 findings with their evidence and ask "do these look right?" before proceeding to thread writing. This is a mandatory checkpoint — the pipeline should not proceed automatically.
13. **Self-verify** — Spot-check 5 key claims against raw data before presenting to user

**If `output_mode` is `research_only`**: Stop here. Write the final report and return results to the user.

### Phase 2: Thread Writing (Synthesizer Method)

Read `references/phase2_thread.md` for the complete thread writing protocol.

Using Phase 1 research, write one finished 9-post Synthesizer-style thread for the configured platform (default: Threads).

Key requirements:
- Default 9 posts: hook + 7 insights + closer/CTA. The insight count can be adjusted (5-12) based on how many survive the validation gate. Fewer strong insights is better than padding.
- Hook selected from the Synthesizer Hook Bank (15 proven formats in the reference file)
- Every statistic must come from the Phase 1 corpus data
- Each insight post follows the claim → data → takeaway structure
- Thread must read naturally on mobile
- Format rules adapt to the target platform (see `references/user_config.md`)

**If `output_mode` is `thread_only`**: Stop here. Write the final report and return results to the user.

### Phase 3: Visual Production

Read `references/phase3_visuals.md` for the complete visual production protocol.

Create 9 production-ready carousel visuals — one per thread post. Each visual is generated as SVG (primary), self-contained HTML/CSS, and PNG preview (if rendering is available).

Style: minimalist editorial, research dossier feel. Strong typographic hierarchy, generous negative space, clean grid.

### Phase 4: Publish & Verify

Read `references/phase4_publish.md` for the publishing and verification protocol.

Provide the finished thread as copy-paste-ready output. If using Threads as the target platform, open threadify.app/plans in the user's browser. Run the complete verification checklist across all 4 phases before declaring the pipeline complete.

## Output Structure

Read `references/output_structure.md` for the complete folder layout. All output goes into:

```
{output_dir}/{creator-slug}/
```

Where `output_dir` defaults to `research/youtube-packaging/` but can be overridden by the user — see `references/user_config.md`.

This includes raw data, normalized dossiers, analyses, constitutions, the thread, visuals, and logs.

## Checkpoint & Resume

After each major milestone, write progress to `logs/checkpoint.json` with this structure:

```json
{
  "phase": 1,
  "step": "data_collection",
  "videos_processed": 42,
  "total_videos": 87,
  "timestamp": "2025-01-15T10:30:00Z",
  "completed_steps": ["creator_resolution", "format_classification"],
  "next_step": "feature_extraction"
}
```

If interrupted, check for `logs/checkpoint.json` on startup. If found, confirm with the user: "I found a previous run for {creator}. Resume from {step} or start fresh?" Then resume from the last checkpoint or restart as directed.

After each milestone, write a brief factual progress note to `00_run_report.md`.

### Incremental Updates

If the output directory already contains a previous run for the same channel:

1. Check `logs/checkpoint.json` and `05_video_index.json` for the previous run's data
2. Ask the user: "I found a previous analysis of {creator} from {date} with {N} videos. Would you like to:
   a) Run an incremental update (analyze only new videos since the last run)
   b) Start fresh (re-analyze everything)
   c) Resume from where the previous run left off"
3. For incremental updates:
   - Collect only videos published after the latest video in the existing index
   - Merge new videos into the existing index
   - Re-run the full analysis (Steps 5-9) on the merged corpus
   - Note in the run report which videos are new
   - Re-download transcripts and thumbnails only for new videos

## Notes & Exceptions

- If the YouTube API fails or rate-limits, fall back to web search automatically
- If the target creator has fewer than ~20 long-form videos in the time window, consider expanding to 36 months and log the decision
- If thread numbers don't match source corpus, flag discrepancies and correct from source data
- The pipeline adapts automatically to creator format — interview channels get guest analysis, solo educators get structure analysis, mixed channels get segmented analysis
- All portability analysis is skipped entirely if no reference channel is provided
- If the user has previously analyzed this channel, check for existing output in the output directory. Offer to run an incremental update (new videos only) rather than starting from scratch.
- Comment analysis is optional and API-quota-intensive. Prioritize comments for top 10 and bottom 10 performers only.
