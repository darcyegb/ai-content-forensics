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

First, check the local environment for: API credentials (`YOUTUBE_API_KEY`), extraction scripts, transcript tools, prior research from previous runs (check for existing `logs/checkpoint.json`).

- **YouTube Analytics API**: Check for OAuth2 credentials that could access the YouTube Analytics API. If the user owns or manages the target channel, Analytics data is available and dramatically more useful than public metrics:
  - **Click-through rate (CTR)**: The single best packaging metric — directly measures "did the title+thumbnail make people click?" Not available from the public Data API.
  - **Impressions**: How many times the video was shown to potential viewers.
  - **Audience retention curves**: Where do viewers drop off? Reveals hook and structure effectiveness.
  - **Traffic sources**: Did views come from search, browse, subscriptions, or external? Packaging matters most for browse and search traffic.

  If Analytics API access is available, use CTR as the primary packaging metric instead of total views. CTR directly measures packaging effectiveness while total views conflates packaging with topic interest and algorithmic promotion.

  If not available (the common case for third-party analysis), note this limitation and proceed with total views.

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

4. **Transcripts** — use this specific fallback chain:

   a. **`youtube-transcript-api` Python library** (pip install youtube-transcript-api). This is the most reliable automated method. In v1.x+, use instance methods:
      ```python
      from youtube_transcript_api import YouTubeTranscriptApi
      api = YouTubeTranscriptApi()
      transcript = api.fetch(video_id)
      text = " ".join([entry.text for entry in transcript])
      ```
      CRITICAL: Always read video IDs from saved data files — never type them from memory.

   b. **Gemini API video understanding** (if `GOOGLE_API_KEY` or `GEMINI_API_KEY` available). Gemini can extract transcript text directly from a YouTube URL — no download or caption API needed. This bypasses all timedtext/innertube rate limits. See the **Gemini Video Analysis** section below for full details and code. Use Gemini for transcript extraction when `youtube-transcript-api` fails (429/403) or when you also want visual analysis of the same video.

   c. **YouTube Captions API** (`captions.list` endpoint) — useful for **checking whether captions exist** (`kind=asr` for auto-generated, `kind=standard` for manual). The docs say OAuth2 is required, but in practice `captions.list` works with just an API key for public videos (confirmed empirically March 2026). However, the `captions.download` endpoint requires the user to **have permission to edit the video** (per [official docs](https://developers.google.com/youtube/v3/docs/captions/download)). This means only the video owner or an authorized editor can download caption tracks. Neither OAuth2 credentials nor a service account from a third party will work — you'll get a 403 Forbidden. This is a YouTube API design limitation, not an auth configuration problem. Do not waste time trying to solve it with credentials.

   d. **Apify transcript actors** — if the user has an Apify account. Actors like `akash9078/youtube-transcript-extractor` work via REST API. Note: each actor has a different input schema (some use `videoUrl`, some use `urls`, some use `url`). Test with one video first to verify the schema.

   e. **Web search** for "{video title} transcript" → fan wikis, community sites.

   **Expected failures**: Livestreams, community events, and some older videos often lack accessible transcripts. This is normal — log it and continue. Do not treat it as a pipeline failure.

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

### Gemini Video Analysis (Optional Enhancement)

**What it does:** Google's Gemini API can natively process YouTube videos by URL — analyzing visual content, audio, on-screen text, facial expressions, energy levels, and scene transitions. This goes far beyond transcript-only analysis and is the only tool that can "watch" a video the way a human viewer does.

**When to use it:**
- When `youtube-transcript-api` fails (429/403 rate limits)
- For visual analysis that Claude can't do from thumbnails alone (in-video graphics, B-roll, energy shifts, pacing)
- For the top and bottom performers where deep visual analysis would strengthen findings
- For natural experiment pairs where you want to compare the full viewing experience, not just metadata

**Environment check:** Look for `GOOGLE_API_KEY` or `GEMINI_API_KEY` in the environment. If neither is available, skip Gemini analysis and log the fallback.

#### Model Selection

| Model | Model ID | Best For | Cost/Min |
|:------|:---------|:---------|:---------|
| **Gemini 3.1 Pro** | `gemini-3.1-pro-preview` | Deep reasoning, max accuracy, complex scenes | ~$0.025 |
| **Gemini 3 Flash** | `gemini-3-flash-preview` | Best price-to-performance, recommended default | ~$0.005 |
| **Gemini 3.1 Flash-Lite** | `gemini-3.1-flash-lite-preview` | Bulk analysis, lowest cost | ~$0.0015 |

All models support 1M token context and 65,536 max output tokens. Videos tokenize at ~300 tokens/sec (default resolution) or ~100 tokens/sec (low resolution).

**Use Flash for bulk corpus analysis.** Use Pro only for the top 5-10 videos where you need maximum detail.

#### YouTube URL Limitations

- **Public videos only** — unlisted and private videos are not supported
- **Up to 10 videos per request** (Gemini 2.5+) — enables batch and comparative analysis
- **Free tier:** max 8 hours of YouTube video per day
- **Max video length:** ~1 hour at default resolution, ~3 hours at low resolution (1M context models)

#### Implementation

```python
from google import genai
from google.genai import types
import json, os

# Use GOOGLE_API_KEY or GEMINI_API_KEY from environment
api_key = os.environ.get("GOOGLE_API_KEY") or os.environ.get("GEMINI_API_KEY")
client = genai.Client(api_key=api_key)

def gemini_analyze_video(video_id, prompt, model="gemini-3-flash-preview"):
    """Analyze a YouTube video using Gemini's native video understanding.

    CRITICAL: Always set max_output_tokens=65536. The default (8192) will
    silently truncate dense captioning output.
    """
    response = client.models.generate_content(
        model=model,
        config=types.GenerateContentConfig(
            max_output_tokens=65536,  # MANDATORY — default is only 8192
        ),
        contents=types.Content(
            parts=[
                types.Part(
                    file_data=types.FileData(
                        file_uri=f"https://www.youtube.com/watch?v={video_id}"
                    )
                ),
                types.Part(text=prompt)
            ]
        )
    )
    return response.text
```

#### Analysis Prompts for the Pipeline

**1. Transcript extraction** (when other methods fail):
```python
transcript = gemini_analyze_video(video_id,
    "Transcribe all spoken words in this video verbatim, with timestamps "
    "every 30 seconds. Format: [MM:SS] text. Include speaker labels if "
    "there are multiple speakers.")
```

**2. Visual packaging analysis** (for top/bottom performers):
```python
visual_analysis = gemini_analyze_video(video_id,
    "Analyze this video's visual packaging and production choices:\n"
    "1. OPENING (first 30 seconds): What does the viewer see? Cold open, "
    "   title card, host face, B-roll? How quickly does the hook land?\n"
    "2. ON-SCREEN TEXT: List every piece of text that appears (lower thirds, "
    "   title cards, callouts, slides). Note timestamps.\n"
    "3. VISUAL ENERGY: Rate energy 1-10 at the 1-min, 5-min, midpoint, and "
    "   final-minute marks. Note any energy spikes or drops.\n"
    "4. PRODUCTION STYLE: Lighting, camera angles, set design, B-roll usage, "
    "   graphics/animations, split-screen, picture-in-picture.\n"
    "5. FACIAL EXPRESSIONS: Dominant expressions of the host/guest at key moments.\n"
    "6. THUMBNAIL MATCH: Does the actual video content match what the thumbnail promises?\n"
    "Output as structured JSON.")
```

**3. Hook comparison** (for natural experiment pairs):
```python
# Gemini 2.5+ supports up to 10 videos per request
hook_comparison = client.models.generate_content(
    model="gemini-3-flash-preview",
    config=types.GenerateContentConfig(max_output_tokens=65536),
    contents=types.Content(
        parts=[
            types.Part(file_data=types.FileData(
                file_uri=f"https://www.youtube.com/watch?v={video_id_a}")),
            types.Part(file_data=types.FileData(
                file_uri=f"https://www.youtube.com/watch?v={video_id_b}")),
            types.Part(text=(
                "These two videos cover the same topic but have different packaging. "
                "Compare the first 60 seconds of each video. What is different about "
                "the visual approach, hook structure, energy level, and production style? "
                "Which opening is more likely to retain a casual viewer, and why?"))
        ]
    )
)
```

**4. Clip identification** (for creator_report output mode):
```python
clips = gemini_analyze_video(video_id,
    "Watch this video and identify the 10 highest-energy moments that would "
    "work as standalone short-form clips (30-90 seconds). For each:\n"
    "1. Timestamp range (start - end)\n"
    "2. What's happening (topic, emotional shift, revelation)\n"
    "3. Why it would work as a clip (surprise, controversy, humor, insight)\n"
    "4. Suggested clip title (attention-grabbing, under 60 chars)\n"
    "5. Energy score (1-10)\n"
    "Rank by clip potential. Output as JSON.",
    model="gemini-3.1-pro-preview")  # Use Pro for clip ID — needs deep reasoning
```

#### Integration with the Pipeline

- **Step 4 (Data Collection):** Use Gemini as fallback (b) in the transcript chain when `youtube-transcript-api` fails. Log as `"Fallback Used": "Gemini API"` in `logs/fallback_log.md`.
- **Step 5 (Feature Extraction):** For the top 10 and bottom 10 videos by total views, run the visual packaging analysis prompt. Save results to `raw/gemini_visual/{video_id}.json`. Feed these features into the packaging features file alongside title/thumbnail/hook features.
- **Step 6 (Analysis):** For natural experiment pairs, use the multi-video comparison prompt to get side-by-side visual analysis. This is stronger evidence than metadata-only comparison.
- **Step 8 (Synthesis):** If Gemini visual analysis was performed, add a "Visual Production Patterns" section to the synthesis covering production style trends, energy patterns, and title-content alignment across the corpus.

#### Cost Management

For a typical 50-video corpus using Flash:
- Transcript extraction (50 videos, avg 15 min each): 50 × 15 × $0.005 = **~$3.75**
- Visual analysis (top+bottom 20 videos): 20 × 15 × $0.005 = **~$1.50**
- Hook comparisons (5 pairs, Pro): 10 × 5 × $0.025 = **~$1.25**
- **Total: ~$6.50** — well within typical API budgets

For cost-sensitive runs, use Flash-Lite for transcripts (~$1.13 for 50 videos) and Flash for visual analysis.

#### Fallback Behavior

If Gemini API is unavailable or the video is not public:
1. Log the failure in `logs/fallback_log.md`
2. Fall back to the next method in the transcript chain (Captions API, Apify, web search)
3. For visual analysis, fall back to Claude's thumbnail analysis (already in the pipeline) and note the limitation
4. Never skip a video entirely because Gemini failed — the pipeline must gracefully degrade

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
- Top 20-50 comments (text, like count) — for packaging signal extraction

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

This is optional because it's API-quota-intensive (each video's comments costs quota). Prioritize collecting comments for the top 10 and bottom 10 performers, where the contrast is most informative.

Log results in `analyses/comment_signals.md`.

### Performance Metric Selection — Recency Bias Warning

**Views per day (VPD) is unreliable as a primary metric** when the corpus spans more than a few months. Most YouTube views occur in the first week after publication, so VPD massively favors recent videos (a 1-day-old video with 80K views = 80,000 VPD; a 2-year-old video with 140K views = 192 VPD). This creates false patterns where recent content appears to dominate.

**Required approach:**
1. Use **total views** as the primary performance metric for ranking and quartile comparisons
2. Calculate VPD as a secondary/supplementary metric only
3. When reporting findings, **validate every pattern against both metrics** before declaring it real. If a finding only appears with VPD but not total views (or vice versa), flag this discrepancy and investigate whether recency or age is driving the difference.
4. For the most robust analysis, filter to videos older than 30-60 days before running comparative analyses, to exclude videos still in their initial spike period
5. Log which metric was used for each finding in the analysis documents

A finding that holds up under both total views AND VPD is high-confidence. A finding that only appears under one metric should be flagged as potentially biased and investigated before inclusion in constitutions or the thread.

**Neither metric is perfect.** Total views favors older videos that have had more time to accumulate views. VPD favors recent videos in their initial spike. The ideal metric would be views at a fixed time point (e.g., views after 30 days), but this is not available from the public YouTube API. Acknowledge this limitation in the methodology document.

**Before generating the thread, present the key findings to the user for review.** The recency bias issue was caught by a human reviewer, not by the pipeline. Allow the user to challenge the numbers before they're baked into the thread and visuals.

### Metric Computation — Do This Immediately After Data Collection

Before proceeding to feature extraction or analysis, compute ALL of the following for every video:

1. **Total views** (primary metric)
2. **Views per day** (secondary, directional only)
3. **Age-filtered VPD** — VPD computed only for videos older than 30 days (excludes initial spike period)
4. **Age bucket** — categorize each video: <7 days, 7-30 days, 30-90 days, 90-365 days, >365 days

Then run a **metric diagnostic** before proceeding:
- Rank all videos by total views. Rank again by VPD. Compare the top quartiles.
- If fewer than 50% of videos appear in BOTH top quartiles, flag to the user: "The two metrics disagree significantly — findings will need dual-metric validation."
- Compute avg age for top and bottom quartiles (by total views). If the ratio exceeds 1.5x, note this as a potential age bias in the methodology document.

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

### Hook Features
- Exact opening lines (first 3-5 sentences from transcript)
- First 15-second and 30-second summary
- Time to first high-stakes statement
- Time to first concrete promise or payoff
- Whether opening validates title/thumbnail promise quickly
- Hook archetype
- Title-thumbnail-hook alignment assessment

### Transcript Depth Features (for videos with full transcripts)

Beyond hook analysis, extract these features from the full transcript:

- **Title-content alignment**: Does the video deliver what the title promises? Score as strong/moderate/weak alignment. A clickbait title with weak alignment should be flagged.
- **Readability score**: Compute Flesch-Kincaid grade level or similar. Compare top vs bottom performers.
- **Word count**: Total words as a proxy for information density relative to duration.
- **Emotional arc**: Sample sentiment at 5 points through the transcript (0%, 25%, 50%, 75%, 100%). Does the video build tension, maintain steady tone, or follow a different arc?
- **Key phrase density**: Count how often the title's core concept appears in the transcript (reinforcement frequency).
- **Question density**: How many questions does the speaker ask? This may correlate with audience engagement style.

These features are secondary to title/hook analysis but can reveal structural patterns in top performers that aren't visible from metadata alone.

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

### Event-Driven Outlier Detection

For each video, assess whether its performance is primarily driven by external events rather than packaging:

- **Flag as event-driven** if the video is about a breaking news event, viral moment, or time-sensitive topic AND its view count is >3x the channel's median
- **Indicators**: title references a specific date, person in the news, or ongoing event; description contains news links; video was published within 48 hours of a major event
- **Tag in the video index**: `event_driven: true/false`
- **In analysis**: Include event-driven videos in corpus totals but **do not let them drive packaging rules.** A title format that only outperforms because one event-driven video used it is not a packaging insight — it's a topic insight. Report them separately in `analyses/outliers.md`.

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

## Step 6: Analysis (4 Layers)

### Layer 1 — Descriptive
What patterns repeat in titles, thumbnails, hooks, structures? If multiple format families exist, what differs by family?

#### Temporal Trends

Split the corpus into 3-4 chronological buckets (e.g., quarters or 6-month periods). For each bucket:
- What is the average duration? Is it increasing?
- Which title features are becoming more/less common?
- Is performance (total views) trending up or down?
- Are there packaging shifts — did the creator start using ellipsis titles more recently?

This reveals whether the creator is already learning what the analysis discovers, and whether the "best" patterns are recent innovations or long-standing habits. Document in `analyses/temporal_trends.md`.

### Layer 2 — Comparative
Compare top-performing vs bottom-performing videos using **total views** as the primary ranking metric. Run the same comparison using VPD as a secondary check. Any finding that only appears under one metric must be flagged and investigated for recency or age bias before inclusion.

Identify patterns disproportionately present in top performers. State correlation, not causality. Check whether the top quartile is disproportionately recent or old — if the avg age differs by more than 1.5x between quartiles, the ranking metric may be biasing results.

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
- These pairs should be highlighted prominently in the synthesis — they are more compelling than any correlational average

When comparing any two specific videos (natural experiments or otherwise), **always download the full dossier for both**: transcript, thumbnail image, full description, and all metadata. Do not compare videos based on title text alone.

### Sensitivity Analysis (Required)

Before declaring any finding, test its robustness:

1. **Outlier sensitivity**: Remove the top 5% and bottom 5% of videos by total views. Re-run the comparison. If the finding disappears, it was driven by outliers — flag as "low robustness" and note which videos drove it.
2. **Age cohort check**: Split the corpus into older half and newer half by publish date. Does the finding hold in both halves? If it only appears in one, it may be confounded by time.
3. **Sample size floor**: Any finding based on fewer than 10 videos in the relevant group must be flagged as "insufficient sample" and excluded from constitutions (though it can appear in the synthesis as a hypothesis).

### Statistical Rigor

Do not report effect sizes without assessing their reliability:

1. **Sample size and confidence**: For any comparison (e.g., "videos with ellipsis get 1.62x views"), report the sample sizes (n=16 with, n=200 without). If either group has n<15, flag the finding as "suggestive but underpowered."

2. **Permutation test for key findings**: For the top 5-7 findings that will drive the thread, run a simple permutation test:
   - Randomly shuffle the feature labels 1000 times
   - Compute the ratio each time
   - Report what percentage of random shuffles produce a ratio as extreme as the observed one
   - If >5% of shuffles match or exceed the observed ratio, the finding is not statistically significant — flag it

3. **Multiple comparisons warning**: When testing 15+ features, some will appear significant by chance alone. Acknowledge this in the methodology document. Findings that survive both dual-metric validation AND permutation testing are high-confidence. Findings that survive only one test should be labeled "moderate confidence."

4. **Effect size with context**: Always report the absolute numbers alongside ratios. "1.62x" from 527K vs 325K is more credible than "1.62x" from 52K vs 32K. The absolute gap matters as much as the ratio.

### Feature Correlation Matrix

Before reporting individual feature effects, check which features are correlated with each other:

1. Compute pairwise correlation between all boolean title features (has_ellipsis, has_negative, has_colon, etc.)
2. Compute correlation between title features and duration, publish date, and host (for multi-host channels)
3. If two features are highly correlated (>0.3), note this — their individual effect sizes may be measuring the same underlying signal

For the strongest findings, test whether they survive when controlling for correlated features. For example, if ellipsis titles also tend to be longer, check whether the ellipsis effect holds within the same duration bucket.

Save the correlation matrix to `analyses/feature_correlations.md`.

### Layer 3 — Portability (ONLY if `your_channel_handle` provided)

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

If no reference channel provided, skip Layer 3 entirely.

### Layer 4 — Synthesis
Convert strongest findings into operational constitutions. Prefer actionable rules over vague commentary.

---

## Step 7: Constitutions

Create 5 constitutions, each containing:

1. **Master packaging constitution** — overview + cross-cutting rules
2. **Title constitution**
3. **Thumbnail constitution**
4. **Hook / opening constitution**
5. **Script and structure constitution**

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
9. Top vs bottom performer comparisons with effect sizes
10. Portable laws (if reference channel provided)
11. Contradictions and outliers
12. Recommended experiments
13. **RANKED LIST**: minimum 15 screenshot-worthy insight candidates, scored by:
    - Surprise
    - Specificity
    - Actionability
    - Shareability

Clearly separate: raw evidence → interpreted patterns → portable laws → creator-specific artifacts.

### Insight Validation Gate

Before finalizing the ranked insight list, run this validation:

1. **Metric robustness check**: Does each insight hold under both total views AND views/day? Mark each as "confirmed (both metrics)", "confirmed (total views only)", "confirmed (VPD only)", or "conflicting". Only "confirmed (both metrics)" insights should be rated high-confidence.
2. **Recency check**: For each insight, check whether the supporting videos are disproportionately recent (<60 days old). If so, the insight may be an artifact of the initial view spike.
3. **Sample size check**: Flag any insight based on fewer than 10 videos in the relevant group.
4. **Create a recency bias correction document** (`analyses/recency_bias_correction.md`) listing any insights that were retracted or corrected during validation.

This ranked insight list is the direct input for Phase 2 thread writing.

### Video-Level Recommendations

After completing the synthesis, generate 5-10 specific counterfactual recommendations for individual videos:

1. **Identify underperforming videos with fixable packaging.** Look for videos in the bottom half by total views whose content quality (based on like ratio, comment sentiment, or transcript analysis) suggests the topic was strong but the packaging was weak.
2. **For each, suggest a specific alternative title** that follows the discovered rules. Example: "Fighting TB in Kids: A Discussion with Doctors without Borders" → "How Panda Express Is Tuberculosis" (actual case — same topic, 150x the views).
3. **When generating counterfactuals, download the full dossier for each video** — transcript, thumbnail, description — so the recommendation is grounded in what the video actually contains, not just its title.
4. **Save to `analyses/video_recommendations.md`** with the format: current title, recommended title, reasoning (which rules it violates/follows), estimated impact based on similar videos.

These recommendations are especially useful for the `creator_report` output mode, where actionable specifics matter more than general patterns.

## Step 9: Self-Verification

Before presenting findings to the user, spot-check the analysis:

1. **Pick 5 claims from the synthesis** — preferably the ones with the most specific numbers.
2. **For each claim, go back to the raw data file** (video index JSON or packaging features JSON) and recompute the number from scratch using code.
3. **Compare the recomputed number to what appears in the synthesis.**
4. **If any number doesn't match**, investigate and correct before proceeding.

This step exists because:
- Numbers can drift during the analysis process (rounding, subsetting, copy errors)
- The recency bias correction proved that initial numbers can be systematically wrong
- The user should never see a number that hasn't been verified against source data

Log the verification results in `logs/verification_log.md` with the format:
```
| Claim | Synthesis Value | Recomputed Value | Match? | Source File |
```

Do not skip this step. It typically takes 5 minutes and has caught errors in every tested run.
