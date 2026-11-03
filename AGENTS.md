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
