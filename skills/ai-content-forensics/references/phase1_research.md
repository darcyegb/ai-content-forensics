# Phase 1: Research & Corpus Building

This is the largest and most critical phase. Everything downstream depends on the quality of this research. Think of it like building a case file — you need airtight evidence before you can draw conclusions.

## Adaptation Principle

Adapt to the actual target creator. Do not force interview analysis onto a solo educator, or vice versa. Do not blend unlike formats into one fake law set. Classify first, segment if needed, then extract.

## Rules

1. API first, web search second, scrape third
2. Never guess ambiguous data — verify or ask one precise question
3. Keep raw evidence separate from interpretation
4. Preserve titles and transcripts verbatim
5. Save thumbnails as files
6. Log every fallback path used
7. Every constitution rule must trace to corpus evidence
8. Exclude Shorts, clips, side feeds unless needed for disambiguation
9. Mixed-format creators: segment corpus by format family before analysis
10. If a metric is unavailable, log it — do not fabricate

---

## Step 1: Creator Resolution

Resolve `target_youtuber` to a canonical channel.

The input may be a name, handle, URL, or ambiguous string. Resolve to: canonical channel name, channel ID, main vs side feeds, format patterns.

Verify using channel page, metadata, upload behavior — not just search titles. If ambiguous, ask one precise question and wait.

## Step 2: Format Classification

Classify the creator's content into format families before extraction:

- Solo educational / tutorial
- Founder / build-in-public
- Interview / podcast
- Essay / commentary
- Documentary / explainer
- Mixed (segment corpus by family)

Create better families if the evidence demands it. The point is to avoid analyzing interview-style content with solo-educator patterns. Each family may have different packaging rules.

### Enforcement: Tag Every Video

Do not estimate format family distribution (e.g., "~160 solo essay"). Programmatically tag EVERY video with its format family using:
- Duration (livestreams >30 min, deep dives 10-20 min, standard 3-10 min)
- Title patterns (guest names for interviews, "livestream"/"live" keywords)
- Description keywords
- Known channel patterns (e.g., alternating hosts, series titles)

Add a `format_family` field to the video index. All analysis should be runnable per-family, not just pooled.

### Multi-Host Detection

If the channel has multiple regular hosts (e.g., Vlogbrothers alternates between Hank and John Green), identify this during format classification. For multi-host channels:

1. **Tag every video with its primary host** using description text, greeting patterns, or metadata clues (e.g., "Good morning, John" = Hank is speaking; publish day patterns).
2. **Run all analyses both pooled AND per-host.** A finding that only appears for one host is a host effect, not a packaging law.
3. **Add a `host` field** to the video index and packaging features files.
4. **Check whether apparent day-of-week or scheduling effects are actually host effects** (e.g., if Host A always posts Tuesdays and Host B always posts Fridays, "Friday outperforms Tuesday" might just mean "Host B outperforms Host A").

## Step 3: Reference Profile (Optional)

**Only if `your_channel_handle` is provided:**

Inspect the user's YouTube channel and build a reference profile covering: positioning, content pillars, formats, audience, packaging patterns, weaknesses. This becomes the portability filter for all findings.

If `your_channel_handle` is empty, skip entirely and proceed to data collection.

## Step 4: Data Collection

### Environment Check

First, check the local environment for:
1. **API credentials**: Source `~/.claude/secrets.env` and verify `YOUTUBE_API_KEY` is set. If not available, STOP and ask the user. Do NOT proceed without the API key.
2. **Node.js + Playwright**: Verify `node` is available and `playwright` npm package is installed in the working directory. If not, run `npm install playwright`.
3. **Chrome**: Verify Google Chrome is installed at `/Applications/Google Chrome.app` (macOS).
4. **Gemini API**: Check for `GOOGLE_API_KEY` or `GEMINI_API_KEY` in the environment for visual analysis.
5. **Prior research**: Check for existing `logs/checkpoint.json` from previous runs.

- **YouTube Analytics API**: Check for OAuth2 credentials that could access the YouTube Analytics API. If the user owns or manages the target channel, Analytics data is available and dramatically more useful than public metrics:
  - **Click-through rate (CTR)**: The single best packaging metric — directly measures "did the title+thumbnail make people click?" Not available from the public Data API.
  - **Impressions**: How many times the video was shown to potential viewers.
  - **Audience retention curves**: Where do viewers drop off? Reveals hook and structure effectiveness.
  - **Traffic sources**: Did views come from search, browse, subscriptions, or external? Packaging matters most for browse and search traffic.

  If Analytics API access is available, use CTR as the primary packaging metric instead of total views. If not available (the common case for third-party analysis), note this limitation and proceed with total views.

### Data Collection Fallback Chain

For every data point, try these sources in order and stop at the first success:

1. **YouTube Data API** (if `YOUTUBE_API_KEY` available):
   - Channel resolution, uploads pagination
   - Per-video: ID, title, publish date, duration, thumbnail URL, description, metrics (views, likes, comments)
   - Chapter/timestamp text from descriptions

2. **Web search** (always available — this is the universal fallback):
   - Search for "{creator name} YouTube channel" to find channel metadata
   - Search for individual video titles + "YouTube" to find metadata
   - Search for "{video title} transcript" to find transcript text
   - Use web search results to extract view counts, publish dates, and other metrics from search snippets

3. **Web scraping tools** (if Apify or similar available):
   - Use Apify RAG web browser to scrape channel pages
   - Use public transcript extraction services

4. **Transcripts** — USE THE CHROME CDP METHOD (see below). Do NOT use `youtube-transcript-api`, `yt-dlp`, or direct HTTP to `/api/timedtext` — YouTube aggressively 429-blocks that endpoint. The Chrome CDP method is the only reliable approach for bulk extraction.

5. **Thumbnails**: Download the actual thumbnail image for every qualifying video.

   ```python
   import urllib.request
   url = video["thumbnailUrl"]  # from API response (maxres or high)
   urllib.request.urlretrieve(url, f"{TRANS_DIR}/../thumbnails/{video_id}.jpg")
   ```

   Save to `raw/thumbnails/{video_id}.jpg`. If download fails, save the URL and log the failure.

   **Thumbnail visual analysis**: Claude is multimodal. After downloading, analyze each thumbnail image to extract:
   - Face count and which face is dominant
   - Emotion/expression (neutral, surprise, concern, joy, etc.)
   - Text overlay (transcribe any text on the thumbnail)
   - Background complexity (simple solid/gradient vs. detailed scene)
   - Color palette (dominant colors, contrast level)
   - Composition style (centered face, rule of thirds, split frame, etc.)
   - Whether it contains screenshots, diagrams, props, or B-roll frames

   For the top 25 and bottom 25 videos by total views, read each thumbnail image and record these features in `06_packaging_features.json`. For the full corpus, batch-analyze in groups of 10-20 to manage context.

### Transcript Extraction — Chrome CDP Method (MANDATORY)

**WARNING**: YouTube's `/api/timedtext` endpoint returns HTTP 429 for bulk requests regardless of tool — `youtube-transcript-api` (all versions), `yt-dlp` (even with `--cookies-from-browser`), direct `curl`, and even in-browser `fetch()` from headless Chrome all get blocked. The innertube `get_transcript` endpoint returns 400 "Precondition check failed" without a valid PoToken. OAuth2 `captions.download` only works for videos you own. Do NOT waste time on these dead ends.

**The ONLY reliable method** is Chrome CDP + Playwright DOM scraping:

1. **Ask the user to quit Chrome** (`Cmd+Q` on macOS). Chrome locks its profile to one process.

2. **Copy the Chrome profile** (excluding large caches) to a temp directory:
   ```bash
   rsync -a --exclude='Cache' --exclude='Code Cache' --exclude='Service Worker' \
     --exclude='GrShaderCache' --exclude='GPUCache' \
     "$HOME/Library/Application Support/Google/Chrome/" /tmp/chrome_debug_profile/
   ```

3. **Launch Chrome with remote debugging**:
   ```bash
   "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
     --remote-debugging-port=9222 \
     --user-data-dir="/tmp/chrome_debug_profile" \
     --profile-directory=Default \
     --no-first-run --no-default-browser-check \
     "about:blank" &
   ```

4. **Connect Playwright via CDP** (requires `npm install playwright` in working dir):
   ```javascript
   const { chromium } = require('playwright');
   const browser = await chromium.connectOverCDP('http://localhost:9222');
   const page = browser.contexts()[0].pages()[0];
   ```

5. **For each video**, extract the transcript from the DOM:
   ```javascript
   await page.goto(`https://www.youtube.com/watch?v=${videoId}`, { waitUntil: 'networkidle', timeout: 30000 });
   await page.waitForTimeout(2000);

   // Expand description
   const moreBtn = page.locator('#expand, tp-yt-paper-button#expand').first();
   if (await moreBtn.isVisible({ timeout: 2000 })) await moreBtn.click();
   await page.waitForTimeout(1000);

   // Click "Show transcript"
   await page.locator('button').filter({ hasText: /show transcript/i }).first().click({ timeout: 5000 });

   // Wait for transcript segments to render in DOM
   await page.waitForSelector('ytd-transcript-segment-renderer', { timeout: 15000 });

   // Scrape segments
   const segments = await page.evaluate(() => {
     return Array.from(document.querySelectorAll('ytd-transcript-segment-renderer')).map(el => ({
       start: el.querySelector('.segment-timestamp')?.textContent?.trim(),
       text: el.querySelector('.segment-text')?.textContent?.trim()
     }));
   });
   ```

6. **Rate limit**: Wait 3 seconds between videos to avoid triggering any blocks.

7. **After extraction**, close the browser: `await browser.close();`

**Why this works**: Using the real Chrome profile with CDP preserves the user's authenticated session, cookies, and browser fingerprint. YouTube treats the requests as coming from a real signed-in user, bypassing both the timedtext 429 and the innertube 400.

**Why other approaches fail**:
- `youtube-transcript-api` / `yt-dlp` → hit `/api/timedtext` → 429 rate limit (IP-level, lasts 30min–24hrs)
- Innertube `get_transcript` → 400 without valid PoToken (requires real browser attestation)
- `launchPersistentContext` → times out if Chrome needs user interaction at startup
- Copying only cookie files → Chrome encrypts cookies with macOS Keychain → session state lost
- Headless Chrome (even headed) without real profile → bot detection → 400

**If a video has no transcript**: The `waitForSelector` will timeout after 15 seconds. Log it and continue. Some videos genuinely have no transcript available.

### Visual Analysis via Gemini API (Step 4b)

After collecting metadata and transcripts, run visual analysis on a subset of videos using the Gemini API. This sends the actual YouTube video to Gemini for multimodal analysis of set design, camera work, on-screen graphics, B-roll, visual branding, and production quality — information that is impossible to extract from metadata or transcripts alone.

#### Video selection
Limit visual analysis to a maximum of **10 hours of total video** per channel. Select ~10 videos spanning performance tiers:
1. Top 5 performers by views
2. Bottom 3 performers by views
3. 2-3 from the mid-range

Exclude any video over ~160 minutes (2.7 hours) — these may exceed the model's context window.

#### Requirements
- `pip install google-genai` (Python package, `google-genai>=1.68`)
- API key stored in `~/.claude/secrets.env` — the same key used for YouTube Data API works for Gemini
- Source the key at runtime: `source ~/.claude/secrets.env`

#### Model selection — CRITICAL
- **USE `gemini-3.1-pro-preview`** — it has the largest context window and handles videos up to ~2.5 hours
- **Do NOT use `gemini-2.5-flash` or `gemini-2.5-pro`** — their 1M token context limit rejects any video over ~65 minutes, which is ALL long-form podcast content
- **CRITICAL: Always set `max_output_tokens=65536`** — the default (8192) silently truncates dense analysis output. This was confirmed in production testing where 8192 produced truncated results.

#### Cost and timing (production-tested)
| Metric | Observed Value |
|--------|---------------|
| Processing speed | ~2 min per video (range: 107-196 seconds) |
| Output per video | 7,000-10,000 chars (~8-10 detailed sections) with max_output_tokens=16384; expect more with 65536 |
| Cost per minute of video | ~$0.025 |
| 10 videos × avg 110 min | ~$27.50 total |
| Batch of 9 videos | ~22 minutes wall time |

#### Implementation

```python
import google.genai as genai
from google.genai import types
import json, time, os

client = genai.Client(api_key=os.environ.get("YOUTUBE_API_KEY") or os.environ.get("GEMINI_API_KEY"))

VISUAL_ANALYSIS_PROMPT = """Analyze the visual packaging and production design of this YouTube video comprehensively. Cover ALL of the following:

## 1. COLD OPEN (first 90 seconds)
Second-by-second visual sequence: what appears on screen, camera angles, text overlays, graphics, pacing of cuts.

## 2. SET DESIGN & ENVIRONMENT
Studio layout, color palette, lighting style (dramatic/flat/warm/cool), background elements, overall aesthetic.

## 3. CAMERA WORK
Number of cameras, primary shot types (close-up/medium/wide/over-shoulder), switching frequency (cuts per minute estimate), special movements (dolly/pan/zoom), host vs guest framing.

## 4. ON-SCREEN GRAPHICS & TEXT
Lower thirds style and frequency, pull quotes/highlighted statements, statistics/data shown, chapter markers, animated graphics, font styles.

## 5. B-ROLL & CUTAWAYS
Types used (stock footage, diagrams, product shots, etc.), frequency of insertion, how B-roll relates to the conversation.

## 6. VISUAL BRANDING
Logo placement, color scheme consistency, branded intro/outro sequences, watermarks, persistent on-screen elements.

## 7. PRODUCTION QUALITY
Estimated budget tier (low/medium/high/premium), visible microphone types, comparison to typical YouTube podcast production.

## 8. VISUAL ATTENTION PATTERNS
What visual techniques maintain viewer attention? How do visuals support/enhance the audio? What makes this visually distinctive from other YouTube podcasts?

## 9. MID-VIDEO & CLOSING PATTERNS
Any visual changes in the middle vs the opening. Sponsor segment visual treatment. Outro/CTA visual design.

Be extremely specific. Reference exact timestamps where possible."""

def analyze_video_visuals(video_id, output_dir="raw"):
    start = time.time()
    response = client.models.generate_content(
        model="gemini-3.1-pro-preview",
        contents=[
            types.Part.from_uri(
                file_uri=f"https://www.youtube.com/watch?v={video_id}",
                mime_type="video/youtube"
            ),
            VISUAL_ANALYSIS_PROMPT
        ],
        config=types.GenerateContentConfig(max_output_tokens=65536)  # CRITICAL — default 8192 truncates
    )
    elapsed = time.time() - start

    result = {
        "videoId": video_id,
        "model": "gemini-3.1-pro-preview",
        "analysis": response.text,
        "elapsed_seconds": elapsed,
        "chars": len(response.text)
    }

    outpath = os.path.join(output_dir, f"visual_analysis_{video_id}.json")
    with open(outpath, "w") as f:
        json.dump(result, f, indent=2)

    return result
```

#### Additional Gemini prompts for the pipeline

**Transcript extraction** (when Chrome CDP is unavailable):
```python
transcript = client.models.generate_content(
    model="gemini-3.1-pro-preview",
    contents=[
        types.Part.from_uri(file_uri=f"https://www.youtube.com/watch?v={video_id}", mime_type="video/youtube"),
        "Transcribe all spoken words verbatim with timestamps every 30 seconds. Format: [MM:SS] text. Include speaker labels."
    ],
    config=types.GenerateContentConfig(max_output_tokens=65536)
)
```

**Hook comparison** (for natural experiment pairs — Gemini 2.5+ supports up to 10 videos per request):
```python
response = client.models.generate_content(
    model="gemini-3.1-pro-preview",
    contents=[
        types.Part.from_uri(file_uri=f"https://www.youtube.com/watch?v={video_id_a}", mime_type="video/youtube"),
        types.Part.from_uri(file_uri=f"https://www.youtube.com/watch?v={video_id_b}", mime_type="video/youtube"),
        "Compare the first 60 seconds of each video. What differs in visual approach, hook structure, energy level, and production style? Which opening is more likely to retain a casual viewer?"
    ],
    config=types.GenerateContentConfig(max_output_tokens=65536)
)
```

**Clip identification** (for creator_report output mode):
```python
clips = client.models.generate_content(
    model="gemini-3.1-pro-preview",
    contents=[
        types.Part.from_uri(file_uri=f"https://www.youtube.com/watch?v={video_id}", mime_type="video/youtube"),
        "Identify the 10 highest-energy moments for standalone clips (30-90s). For each: timestamp range, what's happening, why it works as a clip, suggested title (<60 chars), energy score (1-10). Rank by clip potential. Output as JSON."
    ],
    config=types.GenerateContentConfig(max_output_tokens=65536)
)
```

#### What this captures that metadata/transcripts cannot
- Physical set design, lighting rigs, and color grading choices
- Camera count, angles, and switching frequency patterns
- On-screen kinetic typography, lower thirds, and data card styles
- B-roll types, frequency, and editorial purpose
- Cold open visual pacing and hook construction
- Sponsor segment visual treatment
- Production quality tier relative to the YouTube podcast category
- How visual techniques correlate with viewer retention strategies

#### Integration with analysis
Visual analysis results feed into:
- **Thumbnail Features** (Step 5): compare thumbnail style to in-video visual branding
- **Hook Features** (Step 5): cold open visual pacing correlates with retention
- **Structure Features** (Step 5): B-roll frequency, graphics density, visual pacing changes
- **Comparative Analysis** (Step 6, Layer 2): cross-reference visual patterns with view performance
- **Constitutions** (Step 7): visual production rules should be included in the master and hook constitutions

Log every fallback path used in `logs/fallback_log.md` with the format:
```
| Data Point | Primary Source | Fallback Used | Reason |
```

### Long-Form Definition

- Include full-length main-channel uploads
- Exclude Shorts, clips, highlights, micro-edits, repost fragments
- **Adapt the duration threshold to the channel's format.** The default >5 minutes works for typical YouTube creators, but some channels (e.g., Vlogbrothers, daily vloggers) have a core format well under 5 minutes. In those cases, use >60 seconds (excluding Shorts) and treat ALL non-Short uploads as qualifying. Determine the threshold AFTER format classification (Step 2), not before.
- Use duration as primary signal, metadata as supporting signal
- Borderline videos go in `exclusions_log.md` with reasoning

### Time Window

Apply the `time_window_months` filter (default: 24 months). Only include videos published within this window.

If the creator has fewer than ~20 qualifying videos, consider expanding to 36 months. Log this decision in `00_run_report.md`.

### Per-Video Data Points

For each qualifying video, collect:

**Raw data:**
- URL, video ID, publish date, title, duration
- Thumbnail file path + URL
- Transcript file path + full text
- Views, likes, comments count (where available)
- Description, chapter timestamps
- Host/guest/collaborator info if relevant
- Top 20-50 comments (text, like count) — for packaging signal extraction (optional, API-quota-intensive)

**Derived metrics:**
- Age in days, views per day
- Like-to-view ratio
- Title: character count, word count, structure features
- Thumbnail: text presence, composition notes
- Opening/hook pattern, time to title payoff
- Structure notes, chapter structure

### Comment Analysis (Optional Enhancement)

If API quota permits, collect the top 20-50 comments (sorted by relevance) for each video. Analyze for:

1. **Packaging feedback**: Comments that explicitly reference the title, thumbnail, or opening ("I clicked because...", "the title made me think...", "that opening was...")
2. **Content-packaging mismatch signals**: Comments expressing surprise or disappointment about what the video actually covered vs. what the title/thumbnail implied
3. **Audience sentiment**: Overall positive/negative/neutral ratio per video — correlate with performance

This is optional because it's API-quota-intensive. Prioritize collecting comments for the top 10 and bottom 10 performers, where the contrast is most informative.

Log results in `analyses/comment_signals.md`.

### Performance Metric Selection — Recency Bias Warning

**Views per day (VPD) is unreliable as a primary metric** when the corpus spans more than a few months. Most YouTube views occur in the first week after publication, so VPD massively favors recent videos. This creates false patterns where recent content appears to dominate.

**Required approach:**
1. Use the **age-residual performance score** (see Step 6, Layer 0) as the primary performance metric
2. Calculate total views and VPD as secondary/supplementary metrics only
3. When reporting findings, **validate every pattern against both metrics** before declaring it real
4. Log which metric was used for each finding in the analysis documents

### Metric Computation — Do This Immediately After Data Collection

Before proceeding to feature extraction or analysis, compute ALL of the following for every video:

1. **Total views** (raw metric)
2. **Views per day** (secondary, directional only)
3. **Age in days**
4. **Age bucket** — categorize each video: <7 days, 7-30 days, 30-90 days, 90-365 days, >365 days

Save all metrics in the video index files (CSV and JSON) at this point. All downstream analysis reads from these files — never recompute metrics from raw data later in the pipeline.

### Data Integrity Rule

**NEVER type video IDs, API identifiers, or external data values from memory.** Always read them programmatically from saved data files (e.g., `json.load()` from the video index). Hallucinated identifiers cause cascading failures that are difficult to diagnose because they produce plausible-looking error messages (e.g., "video not found") that mimic real API issues.

### Checkpoint Enforcement

The checkpoint system described in SKILL.md is mandatory, not optional. After EACH of these steps completes, write `logs/checkpoint.json` with the current state:
- After creator resolution (Step 1)
- After format classification (Step 2)
- After data collection (Step 4)
- After feature extraction (Step 5)
- After analysis (Step 6)
- After constitutions (Step 7)
- After synthesis (Step 8)

If the pipeline is interrupted and resumed, read the checkpoint file and skip completed steps. Do not re-download data that already exists in the raw/ directory.

---

## Step 5: Feature Extraction

For every qualifying video, extract detailed packaging features across these categories:

### Title Features
- Exact title text
- Character count, word count
- Leading token pattern (number, question, how-to, name, statement, etc.)
- Uses: numbers, contrast, quoted claim, implied promise
- Trigger categories: pain, curiosity, speed, status, money, health, identity, certainty, fear, transformation
- Title archetype (template pattern)
- Claim specificity: vague / moderate / exact

### Thumbnail Features
- Local file path
- Face present (yes/no), face count, emotion level (high/med/low)
- Text on image (transcribe if present)
- Contrast level, color intensity, background simplicity
- Focal point clarity
- Screenshot/UI/diagram usage
- Visual metaphor usage
- Thumbnail archetype
- **From visual analysis (if available):** Does the thumbnail aesthetic match the in-video visual branding? Is the thumbnail representative of the actual production style?

### Hook Features
- Exact opening lines (first 3-5 sentences from transcript)
- First 15-second and 30-second summary
- Time to first high-stakes statement
- Time to first concrete promise or payoff
- Whether opening validates title/thumbnail promise quickly
- Hook archetype
- Title-thumbnail-hook alignment assessment
- **From visual analysis (if available):** Cold open visual pacing (cuts per second in first 90s), use of kinetic typography, B-roll montages, guest introduction graphics, physical props, any split-screen or special effects in the opening

### Transcript Depth Features (for videos with full transcripts)

Beyond hook analysis, extract these features from the full transcript:

- **Title-content alignment**: Does the video deliver what the title promises? Score as strong/moderate/weak alignment.
- **Readability score**: Compute Flesch-Kincaid grade level or similar.
- **Word count**: Total words as a proxy for information density relative to duration.
- **Emotional arc**: Sample sentiment at 5 points through the transcript (0%, 25%, 50%, 75%, 100%).
- **Key phrase density**: Count how often the title's core concept appears in the transcript.
- **Question density**: How many questions does the speaker ask?

### Structure Features
- High-level section map
- Intro structure (story-led, proof-led, direct, question-led)
- Who speaks first (for multi-speaker content)
- Story precedes teaching? Proof precedes teaching?
- Pacing notes, topic transition patterns
- Emotional turn points
- Practical takeaway density (high/med/low)
- CTA placement (early/mid/late/end), CTA style
- Ending structure

### Visual Production Features (from Gemini visual analysis, if available)
For videos with visual analysis data, extract these additional features:
- **Set design archetype**: dark/moody studio, bright casual, home office, outdoor, etc.
- **Camera setup**: number of cameras, primary shot types, switching frequency
- **On-screen graphics density**: lower thirds per hour, pull quotes per hour, data cards per hour
- **Kinetic typography usage**: present/absent, frequency, style (bold/subtle)
- **B-roll density and type**: stock footage, diagrams, behind-the-scenes, product shots — frequency per hour
- **Production quality tier**: low / medium / high / premium (relative to YouTube podcast category)
- **Visual branding consistency**: logo placement, color scheme adherence, branded intro/outro
- **Sponsor segment visual treatment**: how visually distinct are sponsored segments?
- **Visual pacing changes**: does the visual style change between opening, middle, and closing?
- **Attention maintenance techniques**: what visual tricks does the editor use to prevent viewer fatigue?

### Event-Driven Outlier Detection

For each video, assess whether its performance is primarily driven by external events rather than packaging:

- **Flag as event-driven** if the video is about a breaking news event, viral moment, or time-sensitive topic AND its view count is >3x the channel's median
- **Indicators**: title references a specific date, person in the news, or ongoing event; description contains news links; video was published within 48 hours of a major event
- **Tag in the video index**: `event_driven: true/false`
- **In analysis**: Include event-driven videos in corpus totals but **do not let them drive packaging rules.** Report them separately in `analyses/outliers.md`.

### Conditional Modules

Enable per format family:

**Interview/podcast module:**
- Guest name, category, fame/authority signals
- How much packaging depends on guest reputation
- Cold open type (guest clip, host monologue, stats, etc.)
- When guest authority is established in the video
- Question sequencing logic, tension escalation
- Thumbnail: host face, guest face, or both dominant

**Solo education module:**
- Title framing: problem, promise, blueprint, myth-bust, teardown, case study
- Opening: proof, result, tension, mistake, objection, outcome
- Examples arrive before theory?
- Structure: step-by-step, framework, teardown, story-led, comparative
- Thumbnail: tool UI, before/after, result framing, creator face

**Founder/build-in-public module:**
- Title: stakes, numbers, runway, revenue, failure, experiment
- Opening: tension, scoreboard, decision point, setback, reveal
- Narrative vs tactical lesson alternation
- Thumbnail: numbers, stakes, dashboards, personal reaction

---

## Step 6: Analysis (5 Layers)

### Layer 0 — Age-Adjusted Performance Scoring (MANDATORY)

Do NOT use raw total views or simple views-per-day (VPD) to rank videos. Both introduce bias:
- Raw views favors older videos that have had more time to accumulate
- VPD favors recent videos because most views come in the first days/weeks

Instead, build an **age-residual performance score**:

1. Fit a log-log regression across all videos: `log(views) = slope * log(age_days) + intercept`
2. For each video, calculate `expected_views = exp(slope * log(age_days) + intercept)`
3. Calculate `performance_ratio = actual_views / expected_views`
   - Ratio > 1.0 = overperformed for its age
   - Ratio < 1.0 = underperformed for its age

This removes age bias without VPD recency distortion. Use `performance_ratio` as the primary performance metric for ALL subsequent analysis layers.

Save the enriched dataset with performance ratios to `06_enriched_features.json`.

### Layer 1 — Single-Feature Correlation

For each extracted boolean feature (authority prefix, fear trigger, geopolitics, psychology, etc.), calculate:
- Mean and median `performance_ratio` for videos WITH the feature vs WITHOUT
- **Mean lift** = avg_ratio_with / avg_ratio_without
- Flag features with lift > 1.15x or < 0.85x as significant

Also analyze:
- **Authority role breakdown**: Compare performance by specific role label (Doctor, Expert, Whistleblower, etc.)
- **Duration buckets**: Compare performance by duration range
- **Title length**: Split into quartiles by character count and compare performance
- **Topic categories**: Classify by topic and compare age-adjusted performance

For videos with visual analysis data, identify whether visual production features correlate with performance.

#### Temporal Trends

Split the corpus into 3-4 chronological buckets. For each bucket:
- What is the average duration? Is it increasing?
- Which title features are becoming more/less common?
- Is performance trending up or down?
- Are there packaging shifts?

Document in `analyses/temporal_trends.md`.

### Layer 2 — Interaction Effects (Compound Features)

Test ALL pairs of boolean features for **synergy** — cases where the combination outperforms either feature alone:

1. For each pair (f1, f2), find videos with BOTH features and videos with NEITHER
2. Calculate lift = avg_ratio_both / avg_ratio_neither
3. Flag **synergy** when avg_ratio_both > max(avg_ratio_f1_only, avg_ratio_f2_only)

This catches compound effects that single-variable analysis misses entirely.

### Natural Experiment Detection

Systematically search for **same-topic, different-packaging pairs** — videos that cover substantially the same subject but use different title formats, thumbnail styles, or hook structures. These are the strongest evidence in the corpus because they partially control for topic.

How to find them:
1. Look for videos with overlapping keywords in titles or descriptions
2. Check for series or follow-up videos on the same topic
3. Look for event-driven pairs (multiple videos about the same news event)

For each pair found:
- Document both videos with full metadata in `evidence/natural_experiments.md`
- Download BOTH transcripts and thumbnails for comparison
- Note the specific packaging differences and the performance gap
- Use Gemini multi-video comparison if available

### Layer 3 — Archetype Clustering

Group videos into **performance archetypes** based on feature combinations:
- Define archetype patterns (e.g., "Authority + Fear/Threat", "Celebrity/Opens Up", "Geopolitics standalone")
- Calculate mean and median performance ratio per archetype
- Rank archetypes from highest to lowest performance
- Identify which archetypes are over-used vs under-used relative to their performance

### Sensitivity Analysis (Required)

Before declaring any finding, test its robustness:

1. **Outlier sensitivity**: Remove the top 5% and bottom 5% of videos by total views. Re-run the comparison. If the finding disappears, flag as "low robustness."
2. **Age cohort check**: Split the corpus into older half and newer half. Does the finding hold in both halves?
3. **Sample size floor**: Any finding based on fewer than 10 videos in the relevant group must be flagged as "insufficient sample" and excluded from constitutions.

### Statistical Rigor

1. **Sample size and confidence**: For any comparison, report the sample sizes. If either group has n<15, flag as "suggestive but underpowered."
2. **Permutation test for key findings**: For the top 5-7 findings, randomly shuffle labels 1000 times. If >5% of shuffles match or exceed the observed ratio, the finding is not statistically significant.
3. **Multiple comparisons warning**: When testing 15+ features, acknowledge that some will appear significant by chance.
4. **Effect size with context**: Always report absolute numbers alongside ratios.

### Feature Correlation Matrix

Before reporting individual feature effects, check which features are correlated:
1. Compute pairwise correlation between all boolean title features
2. Compute correlation between title features and duration, publish date, and host
3. If two features are highly correlated (>0.3), note this — their individual effect sizes may be measuring the same signal

Save to `analyses/feature_correlations.md`.

### Layer 4 — Portability (ONLY if `your_channel_handle` provided)

For each discovered pattern, score:
- `evidence_strength` (strong/moderate/weak)
- `prevalence_in_corpus` (percentage of videos)
- `effect_size_if_measurable`
- `portability_to_user_channel` (high/medium/low)
- `dependence_on`: guest fame, creator brand equity, production scale, timing/news cycle
- `risk_of_false_transfer`
- `recommendation`: adopt / test / ignore

Create three buckets:
- **A. Portable now** for user's channel
- **B. Conditional** — worth testing
- **C. Creator-specific artifacts** — do not blindly copy

If no reference channel provided, skip Layer 4 entirely.

### Layer 5 — Synthesis
Convert strongest findings from all layers into operational constitutions. Prefer actionable rules over vague commentary. Prioritize compound effects and archetype findings over single-feature correlations.

---

## Step 7: Constitutions

Create 6 constitutions, each containing:

1. **Master packaging constitution** — overview + cross-cutting rules
2. **Title constitution**
3. **Thumbnail constitution**
4. **Hook / opening constitution**
5. **Script and structure constitution**
6. **Visual production constitution** — set design, camera work, on-screen graphics, B-roll, and editing patterns (only if visual analysis data is available)

Each constitution must include:
- **Purpose**: what it governs
- **Non-negotiable rules**: hard rules with strong evidence
- **Strong patterns**: common but not universal
- **Conditional rules**: when to use which pattern
- **Anti-patterns**: what to avoid
- **Evidence base**: reference actual videos and data behind each rule
- **Portability notes** (only if reference channel provided)
- **Confidence labels**: mark each rule high / medium / low confidence

Write these as operational law, not fluffy commentary. Another creator or agent should be able to follow them mechanically.

---

## Step 8: Exhaustive Synthesis

Create a comprehensive synthesis document covering:

1. Corpus scope and method
2. Target creator profile and classification
3. Reference channel profile (if provided)
4. Format-family breakdown (if applicable)
5. Strongest title laws
6. Strongest thumbnail laws
7. Strongest hook / opening laws
8. Strongest script / structure laws
9. **Strongest visual production laws** (if visual analysis data available)
10. Top vs bottom performer comparisons with effect sizes
11. Portable laws (if reference channel provided)
12. Contradictions and outliers
13. Recommended experiments
14. **RANKED LIST**: minimum 15 screenshot-worthy insight candidates, scored by:
    - Surprise
    - Specificity
    - Actionability
    - Shareability

Clearly separate: raw evidence → interpreted patterns → portable laws → creator-specific artifacts.

### Video-Level Recommendations

After completing the synthesis, generate 5-10 specific counterfactual recommendations for individual videos:

1. **Identify underperforming videos with fixable packaging.** Look for videos in the bottom half by performance_ratio whose content quality suggests the topic was strong but the packaging was weak.
2. **For each, suggest a specific alternative title** that follows the discovered rules.
3. **Download the full dossier for each video** — transcript, thumbnail, description — so the recommendation is grounded in what the video actually contains.
4. **Save to `analyses/video_recommendations.md`**.

This ranked insight list is the direct input for Phase 2 thread writing.

## Step 9: Self-Verification

Before presenting findings to the user, spot-check the analysis:

1. **Pick 5 claims from the synthesis** — preferably the ones with the most specific numbers.
2. **For each claim, go back to the raw data file** (video index JSON or packaging features JSON) and recompute the number from scratch using code.
3. **Compare the recomputed number to what appears in the synthesis.**
4. **If any number doesn't match**, investigate and correct before proceeding.

Log the verification results in `logs/verification_log.md` with the format:
```
| Claim | Synthesis Value | Recomputed Value | Match? | Source File |
```

Do not skip this step. It typically takes 5 minutes and has caught errors in every tested run.
