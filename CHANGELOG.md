# Voice Email Reader — Changelog

Newest entries at the top. The Voice Email Reader was built before this changelog existed; earlier change history lives in the GitHub commit log for [alene-pixel/email-voice-reader](https://github.com/alene-pixel/email-voice-reader). Future notable changes to the app — voice command behavior, classification rules, mobile-specific fixes, OAuth or Gmail API changes, sanitization rules, etc. — get logged here.

---

## 2026-07-07 — Ignore an accidental repeat of a voice command (fixes the "did it twice" bug)

**Summary**: On iPhone there is a delay between when Alene speaks and when the app reacts (root cause in KNOWN-ISSUES.md, 2026-07-07 entry — inherent to iOS speech recognition, not fixable here). During that delay the app keeps reading, so Alene often thinks it did not hear her and repeats the command. The first command then ran on the intended email AND the repeat ran on the NEXT email — e.g. archiving an email she never meant to touch. Now, if the app hears the **same** command again within a short window after running it, it treats the repeat as accidental and ignores it. This is the voice analog of the existing button double-tap guard (`buttonsLockedUntil`).

**How it works**:
- The controller remembers the id and timestamp of the last voice command it actually ran (`lastExecutedCommandId` / `lastExecutedCommandTime`, set in `handleCommand`).
- A new pure pre-check, `shouldSuppressRepeat(spokenCommand)`, returns true when the spoken words map to the SAME command id within `repeatCommandWindowMs` (default **3000 ms**, matching the existing post-dictation `commandCooldown`). It matches the same first-`voiceMatch`-wins way `handleCommand` does, so the id it compares is exactly the command that would otherwise run.
- `SpeechService.onresult` (command mode) calls this pre-check **before** `this.cancel()`. On a suppressed repeat it returns early — so it neither routes the command nor cancels the email currently being read. The email keeps playing instead of the app falling silent (which was part of why the delay felt broken). Wired via the new `speech.onCommandShouldSuppress` callback.

**Scope / keying**: Suppression is keyed on the command **id**, so only an identical repeat is dropped; a *different* command said right after (e.g. "archive" then "stop") always goes through. Applies to every voice command uniformly. For navigation this is the desired behavior too: saying "next" twice during the delay now skips one email (hear the next) instead of skipping two (blowing past an unheard email). Confirmation-flow commands (yes/no/send/edit) are naturally unaffected because acting on them changes state so the same id is no longer available. On-screen button taps are untouched — they already have `buttonsLockedUntil`.

**Tuning**: `repeatCommandWindowMs` is a single constant in the controller constructor. Shrink it if it ever eats an *intended* quick repeat; grow it if doubles still slip through when Alene waits longer before repeating. Cannot be tested on iOS timing from Claude's side — empirical, like the Nilsson pronunciation loop. Verified with a `node --check` syntax pass on the main inline script.

**Reported by**: Alene.

---

## 2026-07-07 — "Previous" now works across a batch boundary (back into the last batch)

**Summary**: When the reader finished a batch (~10 emails) and fetched the next one, "previous" stopped working on the first email of the new batch and the Previous button disappeared — there was no way to step back to the last email of the prior batch. Fixed by having `checkForMoreEmails()` **append** the newly-fetched batch onto the emails already read this session (and point the position at the first new email) instead of **replacing** the whole list and resetting the position to 0. Now the prior batch stays in memory, so `canGoPrevious` is true on the first email of a new batch and "previous"/⏮️ steps back to the last email of the last batch; "next" then returns forward as expected. Reported by Alene.

**Root cause**: `checkForMoreEmails()` did `setEmails(unseenEmails)` + `setCurrentEmailIndex(0)`. That discarded the finished batch and reset the index, so `canGoPrevious` (`currentEmailIndex > 0`) was false and the Previous command's `isAvailable` returned false, hiding the button. The change computes `firstNewIndex = this.model.emails.length` before growing the array, then `setEmails(this.model.emails.concat(unseenEmails))` and `setCurrentEmailIndex(firstNewIndex)`.

**Bonus fix**: de-duplication is now reliable across batches. `seenIds` is built from `this.model.emails`, which previously held only the latest batch; an unread email left un-acted-upon in an earlier batch could reappear as "unseen" and be re-read. With the full session history retained, `seenIds` covers everything already seen, so no email is read twice. Verified with a `node --check` syntax pass on both inline scripts and a navigation simulation across a batch boundary.

---

## 2026-06-25 — "FedEx" now triggers the Edit command (speech-recognition variant)

**Summary**: Speech recognition was mis-hearing Alene's spoken "edit" as "FedEx" on the send-confirmation screen. Added `fedex` and `fed ex` to the Edit command's `voiceMatch` so both spellings (one word or two, depending on how the recognizer splits it) now route to Edit. Same approach the app already uses for other mis-hearings (e.g., the Spam command matching "marcus spam" / "marcus van," Archive matching "are by" / "barberry"). Display and behavior are otherwise unchanged.

---

## 2026-06-16 — Revert "Nilsson" to "nill sun" (period form regressed in the real app)

**Summary**: Reverted `'Nilsson'` from `'nill. sun'` back to `'nill sun'`. The period form had tested as "NILL-sun" in the standalone sandbox, but once live the app read it as "neel-SUN" — the worst of both worlds (wrong vowel AND wrong stress). Per the app's "revert failed fixes completely" rule, restored the prior `'nill sun'` (short "i," stress slightly late), the best in-app result so far. The entry just below — which announced the period fix — is left in place as the honest record of what was tried; this entry supersedes it.

**Root cause**: The pronunciation swap runs inside `speakChunk`, AFTER `splitIntoSentences`. In a real email the surname sits mid-sentence ("Isabella Nilsson is …"), so after the swap the spoken utterance is "Isabella nill. sun is …" — the inserted period acts as an internal phrase break, ending one phrase on "nill" and starting a fresh one on "sun …". On the default iOS voice that flipped both the vowel and the stress. The standalone sandbox never surfaced this because it spoke the name in isolation / phrase-final, with nothing after "sun."

**Fix forward**: `nilsson-sandbox.html` is being upgraded to speak each candidate IN CONTEXT — embedded in a full sentence and run through the same `formatTextForSpeech` → `splitIntoSentences` → `speakChunk` path the app uses — so candidates are tested the way the app actually says them. Next idea queued: "Nill's son" (the surname derives from "Nils's son"), whose natural English possessive stress falls on the first word.

---

## 2026-06-16 — "Nilsson" emphasis fixed: switch to "nill. sun" (period form)

**Summary**: Changed `'Nilsson'` in `LIC_PRONUNCIATION_REPLACEMENTS` from `'nill sun'` to `'nill. sun'` — adding a period after "nill." This resolves the last open piece of the Nilsson pronunciation (see the earlier 2026-06-16 entry below, which fixed the short "i" vowel but left the emphasis on the 2nd syllable, "nill-SUN"). The period makes "nill" its own mini-sentence, so the iOS voice gives it a falling, end-of-sentence emphasis on the FIRST syllable ("NILL-sun"), while keeping the two pieces split preserves the short "i." Confirmed by ear by Alene.

**How we found it**: Built a standalone pronunciation sandbox (`nilsson-sandbox.html`) — a phone-friendly page with a free-text box, one-tap candidate buttons, a voice picker, and an SSML check — so candidate spellings could be A/B-tested on-device without a deploy per attempt. The period form ("nill. sun") won across roughly a dozen candidates. Full attempt log and diagnosis live in the 2026-06-16 entry of KNOWN-ISSUES.md.

**Safe with sentence chunking**: Same mechanism as the "U. S. A." entry below — the substitution runs inside `speakChunk`, *after* `splitIntoSentences` has already split the email text, so the period introduced in "nill. sun" never acts as a sentence boundary for the chunker. It lives inside a single utterance, where the iOS voice treats it as the internal phrase break that produces the first-syllable emphasis. Displayed email text is unchanged; only the spoken pronunciation changes.

---

## 2026-06-16 — Speak the LIC domain as "Legal Impact for Chickens"

**Summary**: Added `'legalimpactforchickens': 'Legal Impact for Chickens'` to `LIC_PRONUNCIATION_REPLACEMENTS` so the TTS reader speaks LIC email addresses naturally — e.g., `alene@legalimpactforchickens.org` now reads as "alene at Legal Impact for Chickens dot org" instead of mashing the domain into one garbled word.

**Why this works**: The existing `formatTextForSpeech()` step already converts email addresses to a spoken form ("@" → "at", "." → "dot") before the text is split into sentences, so `alene@legalimpactforchickens.org` is "alene at legalimpactforchickens dot org" by the time each chunk reaches `speakChunk()`. There, `applyLICDictionary(text, LIC_PRONUNCIATION_REPLACEMENTS)` does a case-insensitive word-boundary swap, which catches the now-standalone `legalimpactforchickens` token and replaces it with the four words "Legal Impact for Chickens." Displayed email text is unchanged; only the spoken pronunciation changes. The same one-line trick extends to any other long run-together domain that sounds bad. Requested by Alene; previously tracked in the app folder's TODO.md (removed now that this shipped).

---

## 2026-06-16 — Pronounce "Nilsson" with a short "i" (vowel fixed; emphasis tuning ongoing)

**Summary**: Added `'Nilsson': 'nill sun'` to `LIC_PRONUNCIATION_REPLACEMENTS` so the TTS reader says Isabella Nilsson's surname with a short "i" ("nill," like "pill") instead of the long-"ee" "NEEL-son" the iOS voice defaulted to. The vowel is now correct; the emphasis still lands on the 2nd syllable ("nill-SUN") rather than the 1st ("NILL-sun"). Across roughly nine respellings we found the voice trades the two off: any one-word spelling gives the right stress but the wrong vowel, while a hyphen or space gives the right vowel but stresses the last syllable. The current spelling favors the correct vowel, which Alene prioritized. Full attempt log, diagnosis, and untried ideas for a future session live in the 2026-06-16 entry of KNOWN-ISSUES.md. Displayed email text is unchanged; only the spoken pronunciation changes.

---

## 2026-06-12 — Pronounce "USA" as "U. S. A."

**Summary**: Added `'USA': 'U. S. A.'` to `LIC_PRONUNCIATION_REPLACEMENTS` so the TTS reader spells out the letters instead of saying "oosa." Safe with sentence chunking: the substitution runs inside `speakChunk`, after `splitIntoSentences` has already split the text, so the periods in "U. S. A." never act as sentence boundaries. Displayed text still reads "USA"; only the spoken pronunciation changes.

---

## 2026-06-11 — Pronounce "Blome" as "Bloom"

**Summary**: Added the first entry to `LIC_PRONUNCIATION_REPLACEMENTS`: when the TTS reader encounters "Blome" in email text, it now speaks "Bloom" instead. Uses the existing case-insensitive word-boundary helper (`applyLICDictionary`), applied in `speakChunk` right before each utterance — so the displayed text still reads "Blome" while only the spoken pronunciation changes.

---

## 2026-06-08 — Add "Kathryn" and "Greenfire" dictation rules

**Summary**: Extended `LIC_DICTATION_REPLACEMENTS` with two new rules Alene requested. Whatever speech recognition hears as "Catherine," "Katherine," "Catheryn," or "Kathryn" now writes "Kathryn" (Kathryn Evans, LIC's staff attorney). Whatever it hears as "Greenfire" or "Green Fire" now writes "Greenfire" (as in Greenfire Law PC). Both rules rely on the existing case-insensitive word-boundary helper, so capitalization is normalized regardless of what Wispr / the Web Speech API emits.

---

## 2026-06-07 — Drop "or tap" from the OCR cancel prompt

**Summary**: The OCR-starting announcement now says "Say Stop to cancel" instead of "Say or tap Stop to cancel." Alene preferred the shorter phrasing. Single-line change in the scanned-PDF branch of the attachment-reading flow.

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
