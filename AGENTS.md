# SongBook — Codex development guide (2026-11-02)

## Purpose
SongBook is a Hebrew-first personal song/chord web application. Users import one or more song screenshots; OCR extracts right-to-left Hebrew lyrics and Latin guitar chords; the UI allows correcting recognition, transposing, selecting capo/easier key, saving songs in a library, and preparing print/export views. The product should work well on iPhone/iPad as well as desktop. Preserve lyric order and horizontal placement of chords relative to their lyrics.

## Repositories and safety boundaries
- Production: `nyx3six9/SongBook`, GitHub Pages `https://nyx3six9.github.io/SongBook/`. **Do not push or merge to production/main without explicit user approval.**
- Experimental website: `nyx3six9/Songbook_test_site`, Pages `https://nyx3six9.github.io/Songbook_test_site/`. Prototype development and testing happen here.
- Private fixtures/diagnostics: `nyx3six9/songbook_tests`, issue #1 documents OCR experiments. Screenshots and full lyrics are private/copyrighted. Never copy them into public repos, PR descriptions, logs, or public CI artifacts.
- Production development branch `fix/layout-aware-chord-ocr`, draft PR #4. Treat it as prior work, not authorization to merge.
- Confirm branch and repository before writing; use separate commits/PRs, explain changes in plain Hebrew. Never delete the user's library or change persisted data format without a migration and approval.

## User-facing requirements
- Hebrew lyrics RTL; chords remain Latin LTR, aligned with the appropriate words (not merely grouped in a list).
- Import multiple screenshots for one song, including overlapping screenshots; do not automatically assume overlapping chorus indicates an upload-order error.
- Editable OCR per line, ability to delete recognition from an individual image, view original image.
- Transposition in semitones, easier version/automatic key, capo choice and clear indication of capo/no capo in print.
- Edits apply to original key; chord display should transpose consistently (historical bugs included first chord not transposing and missing C chords).
- Mark chorus (`פזמון`) bold; distinguish headings (`פזמון:`, `מעבר:`) from chords.
- Song library and save/restore; export PDF/Word and printable presentation; filename should use song title.
- Maintain correct apostrophe vs Hebrew yod handling, preserve full Hebrew lines, and avoid chord text contaminating Hebrew OCR.
- Validate existing features in actual code before asserting they work: this is a product requirements list, not a verified feature inventory.

## OCR architecture in experimental index.html
Current experimental version: `visual-badges-prototype-4` (commit `da7d96e0b4cdc4e55b5ccbdccf77d02e71e39f04`).
- `makeOCRCanvas(file)`: normalize contrast and scale for OCR.
- `detectChordBadges(file)`: connected-component segmentation of saturated, filled colored chord-pill regions, without hardcoding hue. Gray section-title pills should not be treated as chord badges.
- `prepareBadgeCanvases(file,boxes)`: create chord-only mask and remove colored badges from lyric OCR image.
- Hebrew and English Tesseract OCR run separately; TSV word coordinates retained via `tsvLines`.
- `chordCandidates` and `assembleLayout` parse Latin chord grammar, match candidates to Hebrew lines using y-coordinate, sort right-to-left by x. **Current output still loses exact per-word horizontal alignment.**
- `retryFailedBadges` (v4): for boxes with no valid chord, try three binarization thresholds (145, 190, 225), 4x crop, PSM 7; accept only chord-shaped text with confidence >=55. This is experimental and not yet browser-validated.
- Diagnostics include `version`, `image`, `image detail`, `layout`, `badge details`, `chord candidates`, `assembled lines`, and in v4 `badge retries`. Keep image index and coordinates for debugging.
- Current implementation is largely a single HTML file. Refactor only with explicit tests and without disrupting Pages deployment.

## Evidence and baseline
Same dark-mode image tested on v3: normalized 2412x3584, 29 saturated regions, 19 Hebrew OCR rows, 28 English rows, 24 chord candidates, 1 unmatched.
- 23 badges had at least one valid chord; six badges needing review:
  - #3 `Fa` (suspected F#)
  - #6 `B8bm` (suspected Bbm)
  - #11 no OCR text
  - #12 `8bm`
  - #18 `Ci` (confidence 0)
  - #28 no OCR text
- #23 yielded `B8bm` plus valid `Ab`, with `Ab` unmatched to a lyric line. Some adjacent badges merge into a single detected component (#21, #23). Do not assume a badge marked `recognized` has every chord correct.
- Earlier baselines: dark 20 Hebrew lines / 23 chord candidates / 0 unmatched; light 35 / 43 / 1; two white print screenshots together 15 / 9 / 0. These are diagnostic counts, **not accuracy scores**.
- Visual color segmentation on an original dark-mode screenshot found ~29 complete badge regions and 3 clipped at bottom; do not claim universal recall from this one image.
- v4 has been committed and JavaScript syntax-checked; **actual OCR performance not yet tested**.

## Known failure modes and next priorities
1. Run identical dark image through v4 and compare with v3. Record retry attempts, actual correct chords vs ground truth, missing/false-positive chords and unmatched lines. A chord matching grammar may still be visually wrong.
2. Build a local automated OCR regression harness with private fixtures and per-image ground-truth annotations (chord text, bounding box, lyric row). Measure precision/recall, correct chord text, row association, and Hebrew line accuracy. Never expose fixtures publicly.
3. Improve extraction of white glyphs on varied saturated badges (purple, blue, etc.), dark/light backgrounds, and clipped pills. Avoid brittle hardcoded hues.
4. Split merged neighboring badges and recognize each independently while preserving x/y.
5. Fix unmatched/orphan chord association, including section headings and chord-only lines, without moving chords to wrong lyric lines.
6. Preserve horizontal chord positions relative to words in editable UI/print.
7. Validate print screenshots, dark and light mode, multi-image overlaps, and Hebrew punctuation.
8. Review regressions for transposition, capo, library persistence, and export.

## Working rules for Codex
- Communicate in Hebrew with the user; explain objective, planned changes, and measured outcomes.
- Before edits, inspect the actual repo, tests, Pages deployment and branch; don't trust this document over current code.
- Never claim a feature works solely because syntax parsing passes.
- Test against actual fixtures; report limitations honestly. If private fixtures are inaccessible, request repository authorization or sanitized ground-truth data.
- Avoid changing production until the user explicitly approves a reviewed PR.
- Keep test screenshots and copyrighted song text in private repositories only.
- For every iteration: document baseline, changes, test commands, metrics, remaining defects, commit and preview URL.

## Verified browser run — prototype 4 (2026-11-02)
The user ran the **same dark screenshot** in the live experimental site and supplied full diagnostic JSON. This is a real browser run, unlike the earlier syntax-only check.
- Version: `visual-badges-prototype-4`; normalized image 2412×3584; dark background; **29 badges**, **19 Hebrew rows**, **28 initial English rows**.
- `badge retries`: **6 problematic badges retried**, **4 accepted recoveries**. Chord candidates increased **24 → 28** versus v3. Unmatched candidates remained **1**. Badges with ≥1 grammar-valid chord increased **23 → 27**; two badges still have none. **These counts do not establish recognition accuracy.**
- Badge #3: original `Fa`; trials `F#` (confidence 89), `Fi` (85), `0` (16); accepted `F#`.
- Badge #6: original `B8bm`; trials `Bbm` (51), `Ebm` (80), `BE` (0); accepted `Ebm`. **Ambiguous: the confidence-maximizing answer may be wrong. Do not silently treat Ebm as ground truth.**
- Badge #11: no OCR text at any threshold; unresolved.
- Badge #12: original `8bm`; trials `Bbm` (89), `Bm` (52), `BEE` (20); accepted `Bbm`.
- Badge #18: original `Ci` (confidence 0); trial `C#` had confidence **0** and was rejected; unresolved. The screenshot may show C#, but do not assert without visual verification.
- Badge #28: original empty; trial `E` (59) accepted. **Low confidence; verify against screenshot.**
- Badge #23: merged region x1053 y1377 w98 h31 yielded `B8bm` (invalid) and `Ab` (valid). `Ab` remains unmatched. This region is near the `מעבר:` heading and may be a chord-only transition; preserve independent chord lines rather than forcing an incorrect lyric association.
- Badge #21: merged region x775 y1099 w110 h31 yielded both `Ebm` and `F#`; two words were detected but should ideally have independent badge geometry.
- Hebrew remains imperfect: examples include `בשביילי` and malformed `פזמון|:`; `מעבר:` is a section heading, not a lyric/chord.
- The assembled result appends the unmatched `Ab |` at the end, which is a **layout-order bug**: if the chord is a standalone transition, it belongs at its original vertical position, not after the final verse.

### Immediate instructions for Codex
1. **First task: create reproducible regression tests, not a speculative OCR rewrite.** Inspect actual code, verify whether OCR runs in browser only, and design a headless/browser fixture harness (e.g., Playwright if suitable). Use private screenshots only in a private execution environment.
2. Capture per-badge ground truth from the user's original screenshots; distinguish grammar-valid from visually correct chords. Do not claim that all four v4 recoveries are correct.
3. Preserve the v4 baseline and report both numeric deltas and actual correctness, including false positives, missed badges, merged regions, row association, and Hebrew accuracy.
4. Do not expose screenshots or copyrighted full song lyrics in the public repository or logs. Keep private fixture access explicitly authorized.
5. Propose a small reviewed PR against the experimental repository. Do **not** change production main without explicit user approval.
6. If you are already running a task, read this updated `AGENTS.md` before editing and state which repository/branch you are operating on.
