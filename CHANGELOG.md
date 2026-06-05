# Voice Email Reader — Changelog

Newest entries at the top. The Voice Email Reader was built before this changelog existed; earlier change history lives in the GitHub commit log for [alene-pixel/email-voice-reader](https://github.com/alene-pixel/email-voice-reader). Future notable changes to the app — voice command behavior, classification rules, mobile-specific fixes, OAuth or Gmail API changes, sanitization rules, etc. — get logged here.

---

## 2026-06-05 — Capitalize the first letter of each dictated sentence

**Summary**: Dictated replies now have proper sentence-start capitalization. Before this fix, every sentence Alene dictated landed in the reply box all lowercase because the Web Speech API returns raw lowercase transcripts (especially on mobile) and the app did no post-processing to restore capitalization. Added a small helper, `capitalizeForContext(text, existing)`, that uppercases the first letter of a chunk when it lands at the start of the box or right after sentence-ending punctuation, and also uppercases any letter inside the chunk that follows ". ", "! ", or "? ". Wired into both `handleDictationFinal` (the committed text) and `handleDictationInterim` (the live preview) so capitalization is correct both as she speaks and after each pause.

---

## 2026-06-02 — LIC-specific dictation and pronunciation dictionary

**Summary**: Added a segregated LIC-specific dictionary block near the top of `index.html` (right after the version banner) that lets Alene override (a) what speech recognition writes when it hears a given word, and (b) what TTS speaks when it encounters a given word in email text. Both maps share a single helper, `applyLICDictionary(text, dictionary)`, which does case-insensitive word-boundary replacement. First entry: hearing "Ashley" → write "Ashely" (so dictated replies use the correct spelling of Ashely Monti's name).

**Why this design**: Alene asked whether to (1) hardcode LIC-specific overrides directly, or (2) build a user-customizable UI. We picked a middle path — hardcoded for now since Alene is the only user, but in a clearly-labeled block with a single delete-this-block instruction so the app can be made generic later if shared.

**How it works**:
- `LIC_DICTATION_REPLACEMENTS` (currently `{ 'ashley': 'Ashely' }`) — applied to both interim and final transcripts in the `onresult` handler before they flow to the dictation callbacks, so the substitution appears live in the dictation box as Alene speaks.
- `LIC_PRONUNCIATION_REPLACEMENTS` (currently empty) — applied to text inside `speakChunk` before the SpeechSynthesisUtterance is created. Wired up but empty for now; Alene wants to check whether "LIC" actually needs phonetic spelling before adding it.
- The helper escapes regex metacharacters in keys and matches with `\b…\b` and the `gi` flag, so multi-word keys (e.g., `'Robert Patrick'`) and varying case both work.

**To make the app generic for other users**: delete the LIC-SPECIFIC DICTIONARY block (and the two `applyLICDictionary(...)` calls — one in `onresult`, one in `speakChunk`). No other code changes needed.

---

## 2026-05-29 — In-app OCR fallback for scanned PDFs

**Summary**: The Voice Email Reader can now read scanned (image-based) PDFs aloud. Previously, when Alene tapped "Read Attachments" on a scanned PDF, the app gave up with "PDF appears to be image-based or empty." This was the failure mode hit on the May 29, 2026 [Cherokee Roth filing](https://mail.google.com/mail/u/0/#all/19e7510352001bfa) in *LIC v. Alexandre Family Farm, LLC* (the 17-page "Reply to Opp to Defs' Mtn to Uphold.pdf" had a text layer of exactly zero characters).

**Why this fix and not the desktop OCR pipeline**: Alene's existing OCR workflow (`ocrmypdf` on her Mac, see [operations/ocr/CLAUDE.md](../../../ocr/CLAUDE.md)) requires her to be at her computer. The Voice Email Reader's use case is the opposite: she's walking around, hands occupied, listening on her phone. So the app itself needs the ability to OCR.

**How it works**:
- Added [Tesseract.js v5.1.1](https://github.com/naptha/tesseract.js) (CDN-loaded alongside pdf.js and mammoth.js) — a pure-WebAssembly OCR engine that runs entirely client-side, with no API key or backend.
- `extractPdfText` still tries fast text-layer extraction first; if it comes back empty, it now returns `needsOcr: true` along with the base64 PDF bytes and page count.
- New method `extractPdfTextViaOcr` on `GmailService`: renders each PDF page to a canvas via pdf.js (scale 2.0 for sharper OCR), then runs Tesseract.js on the canvas. Status messages flow back via an `onProgress` callback ("OCRing page 3 of 17…").
- `readAttachments` watches for `needsOcr`, announces via speech ("X.pdf is a scanned image, not text. Running OCR on N pages. This may take a few minutes. Tap Stop to cancel."), then runs the OCR path. The progress callback also doubles as the Stop check — returning truthy from it aborts OCR mid-page.

**Tradeoffs to know**:
- OCR is slow on mobile. Rough budget: ~30–60 seconds per page. The 17-page Alexandre filing might take 5–15 minutes; a 2-page invoice should be under 90 seconds. The Stop button works the whole time.
- First OCR per session pays a one-time download of ~10 MB of English language data (Tesseract `eng.traineddata`). Subsequent PDFs in the same session reuse it.
- Quality is roughly Tesseract baseline — generally good on clean scans, weaker than the desktop `ocrmypdf` pipeline because no preprocessing (deskew, denoise) is applied.
- English-only for now. Tesseract.js supports adding language packs, but adding them would inflate download size.

**Architecture**: All changes confined to `index.html` — the script tag at line 629, modifications to `extractPdfText` and the new `extractPdfTextViaOcr` method in `GmailService`, and the fallback wiring in `readAttachments`.

---

## 2026-05-14 — Changelog initialized

**Summary**: Created this CHANGELOG.md per the project-wide per-system changelog policy in the top-level [CLAUDE.md](../../../../CLAUDE.md). The Voice Email Reader has been live for some time, so prior history is not backfilled — that history is in the GitHub commits. From this date forward, each notable code or behavior change to the app should add an entry here in addition to the GitHub commit. The changelog captures the **why** (the rationale, the bug report, the prior failure mode), which git commits often don't.
