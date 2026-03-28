# Phase 4: Publish & Verify

This is the final phase. After all visuals are complete and verified, provide the copy-paste-ready output, optionally open the publishing tool, and run the complete verification checklist.

---

## Publishing

### Platform-Specific Publishing

**Threads (default):**
Open `https://threadify.app/plans` in the user's browser.
- Claude Code: Use `open` (macOS), `xdg-open` (Linux), or `start` (Windows) shell command
- Cowork: Use available browser MCP tools (Chrome MCP navigate, or computer-use to open a browser)

**X, LinkedIn, Bluesky:**
Do not auto-open a publishing tool. Instead, present the thread in copy-paste-ready format and let the user publish manually or with their preferred tool.

### Copy-Paste Output

Regardless of platform, always ensure `thread/final_thread.md` contains the complete thread with posts separated by em dash lines (—), ready to copy and paste directly into the target platform.

---

## Complete Verification Checklist

Before declaring the pipeline complete, verify everything across all 4 phases:

### Phase 1 Verification
- [ ] Target creator correctly resolved (main channel, not clips/side feed)
- [ ] Time window correctly applied
- [ ] All included videos have: title, thumbnail, transcript, metrics (or logged as unavailable)
- [ ] All fallback paths logged in `logs/fallback_log.md`
- [ ] Constitutions cite actual corpus evidence
- [ ] Portability analysis applied correctly (or skipped if no reference channel)
- [ ] Synthesis contains ranked list of 15+ insight candidates
- [ ] All metrics computed upfront (total views, VPD, age-filtered VPD)
- [ ] Metric diagnostic run and results logged
- [ ] Sensitivity analysis completed
- [ ] Event-driven outliers flagged
- [ ] Format family tagged on every video
- [ ] Multi-host detection completed (if applicable)
- [ ] Natural experiments identified and documented with full dossiers
- [ ] Findings reviewed with user before proceeding to Phase 2
- [ ] Thumbnail images downloaded for at least top 25 + bottom 25 videos
- [ ] Thumbnail visual analysis completed (face count, emotion, text, composition)
- [ ] Self-verification completed: top 5 claims spot-checked against raw data (logged in verification_log.md)
- [ ] Statistical significance assessed for top findings (sample sizes reported, permutation tests where applicable)
- [ ] Feature correlation matrix computed (correlated features identified)
- [ ] Temporal trends documented
- [ ] Comments collected for top 10 + bottom 10 (if API quota permits)

### Phase 2 Verification
- [ ] Thread has the correct number of posts (default 9, adjusted if insight count differs) separated by em dashes (—)
- [ ] All insights backed by exact corpus numbers
- [ ] No fabricated statistics
- [ ] Insight selection criteria satisfied (counterintuitive, diverse categories)
- [ ] Voice is consistent (natural casing, sharp, evidence-led)
- [ ] Posts 2-8 all include "{takeaway_label}:"
- [ ] Insight mix rules satisfied (2+ counterintuitive, 1+ title, 1+ hook, 1+ structure)
- [ ] Hook selected from Synthesizer Hook Bank
- [ ] No two insights overlap
- [ ] All posts fit within platform character limits

### Phase 3 Verification
- [ ] Correct number of visual assets created (one per thread post)
- [ ] Post order matches thread order
- [ ] Hook text on cover is exact
- [ ] Recap scope counts are exact
- [ ] Every on-image number traceable to source data
- [ ] Design system consistent across all 9 assets
- [ ] Main number visually dominant on every insight slide
- [ ] Visual variety (no repeated chart types)

### Phase 4 Verification
- [ ] Publishing tool opened (Threads) or copy-paste output presented (other platforms)
- [ ] All files written to correct paths in output structure
- [ ] Run report (`00_run_report.md`) complete
- [ ] All logs complete

---

## Final Report

When the full pipeline is complete, return a summary to the user:

1. **Summary of what was completed** — which phases ran successfully
2. **Exact output paths** — where all files are saved
3. **Corpus counts**: videos scanned, videos included, transcripts captured, thumbnails saved, constitutions created
4. **Any missing data or blockers encountered** — what couldn't be collected and why
5. **The single most important packaging insight discovered** — the one finding that's most surprising and actionable
6. **The single most important experiment the user should test** (only if reference channel was provided)

Keep this report concise and factual. The user can dig into the full output files for details.
