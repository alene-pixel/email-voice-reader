# Voice Email Reader — Known Issues and Open Observations

Newest entries at the top. This file tracks user-reported issues with the Voice Email Reader that have not been resolved with a code change, but are worth keeping a record of so future-Claude (or future-Alene) can pick up the thread later. Once an issue is closed, mark it Resolved and keep the entry for reference.

---

## 2026-06-16 — "Nilsson" TTS pronunciation [RESOLVED] — the period form "nill. sun"

**Status**: RESOLVED — deployed 2026-06-16. Winning spelling: **`nill. sun`** (period after "nill," space before "sun"), now live in the app's `LIC_PRONUNCIATION_REPLACEMENTS`. The period makes "nill" its own mini-sentence, so the voice gives it a falling end-of-sentence emphasis on the FIRST syllable ("NILL-sun") while the split keeps the short "i." Found via the pronunciation sandbox (idea 1 below); Alene confirmed by ear. One gut-check still worth doing in normal use: that it sounds right inside a full email sentence ("Isabella Nilsson …"), since there the surname is not utterance-final; if it is ever off in context, the sandbox (`nilsson-sandbox.html`) remains for fine-tuning.

**Reporter**: Alene.

**Goal**: Make the app say Isabella Nilsson's surname as **"NILL-sun"**:
- **Vowel**: short "i" as in "pill" / "nil" — NOT a long "ee" ("NEEL").
- **Stress**: on the FIRST syllable ("NILL-sun") — NOT the second ("nill-SUN").

**Where the fix lives**: the `LIC_PRONUNCIATION_REPLACEMENTS` map in `index.html` (search for that symbol; ~line 671 as of 2026-06-16). The entry is `'Nilsson': '<respelling>'`. The value is the exact string handed to the text-to-speech voice; the displayed email text is unchanged. Matching is case-insensitive at word boundaries via `applyLICDictionary`, applied inside `speakChunk`.

**Environment**: iOS text-to-speech via the Web Speech API, on Alene's iPhone (she listens through the Voice Email Reader on her phone).

**Key constraint — Claude can't hear the result**: TTS output depends on the specific device/voice, and Claude cannot hear Alene's iPhone voice (the Mac/Chrome voice differs). So every attempt is empirical: Claude proposes a spelling → commits and pushes → Alene waits 1–2 minutes for GitHub Pages → hard-refreshes on her phone → listens → reports back. This makes the loop slow. **Efficiency idea for the next session**: build a tiny throwaway "pronunciation sandbox" page (a text box plus a Speak button, or several candidate spellings each with its own Speak button) so Alene can hear many candidates back-to-back on her phone without a deploy per attempt.

**What we tried (in order), and what Alene heard**:

| # | Spelling fed to TTS | Vowel | Stress | Net result |
|---|---|---|---|---|
| 1 | `NILL-sun` (hyphen, all caps) | short "i" ✓ | 2nd syllable ✗ | "nill-SUN" |
| 2 | `Nillson` (one word) | long "ee" ✗ | 1st syllable ✓ | "NEEL-sun" |
| 3 | `Nillsun` (one word) | long "ee" ✗ | (vowel was the reported problem) | "Neel…" |
| 4 | `Nihlsun` (one word) | long "ee" ✗ | (vowel was the reported problem) | "Neel…" |
| 5 | `Nilson` (one word — "Wilson with an N") | long "ee" ✗ | 1st syllable ✓ | "NEEL-son" |
| 6 | `Nill-sun` (hyphen, normal case) | short "i" ✓ | 2nd syllable ✗ | "nill-SUN" |
| 7 | `Nill-suhn` (hyphen, schwa 2nd syllable) | short "i" ✓ | 2nd syllable ✗ | "nill-SUN" |
| 8 | `Nill-son` (hyphen, surname suffix) | short "i" ✓ | 2nd syllable ✗ | "nill-SUN" |
| 9 | `nill sun` (two separate words) | short "i" ✓ | 2nd syllable ✗ | "nill-SUN" |
| 10 | `nill. sun` (period after "nill") | short "i" ✓ | 1st syllable ✓ | **"NILL-sun" — WINNER (deployed)** |

**Diagnosis — what the pattern shows**:
- **The voice has two modes, and they trade off against each other:**
  - **One word** (any spelling — Nilson, Nillson, Nihlsun) → always the WRONG vowel ("NEEL"), but the RIGHT (first-syllable) stress. The voice treats a single unfamiliar "Ni…" token as a long-"ee" Scandinavian name — which it literally is (Nilsson is a Swedish surname).
  - **Hyphen or space** (Nill-sun, nill sun) → always the RIGHT short "i" (the isolated "nill" reads like "pill" / "nil"), but the WRONG (second-syllable) stress.
- **In the hyphen/space modes the stress is positional, not vowel-driven.** Respelling the 2nd syllable three different ways — `sun` (full vowel), `suhn` (schwa), `son` (a real surname suffix that voices usually de-accent) — did NOT move the stress off the 2nd syllable. The voice seems to simply stress the last syllable of a hyphenated or two-word item.
- **Capitalization does not affect stress.** `NILL-sun` (caps on the first syllable) still stressed the 2nd syllable.

**Current live state**: `'Nilsson': 'nill. sun'` → "NILL-sun" (short "i," first-syllable emphasis). Deployed 2026-06-16. This solves both the vowel and the emphasis; the period (a sibling of the comma idea below, originally set aside over a feared pause) turned out to be the lever that finally moved the stress forward.

**Ideas NOT yet tried (for the next session)**:
1. **Pronunciation sandbox page** — DONE (2026-06-16). Built and deployed as a standalone page that does NOT touch the app: https://alene-pixel.github.io/email-voice-reader/nilsson-sandbox.html (source: `nilsson-sandbox.html` in the repo root; added in commit `91e80bd`, separate from `index.html`). It mimics the app's exact TTS path (`SpeechSynthesisUtterance`, rate/pitch 1.0, no voice override = system default, plus the paused-resume workaround) and adds: a free-text box (type any spelling, hear it — zero further deploys needed for new ideas), a voice picker (try iOS voices other than the default — idea 2 above), speed/pitch sliders (default 1.0/1.0 to match the app; slowing speed helps hear which syllable is stressed), an "Isabella …" full-name toggle (test in realistic context), an SSML-`<emphasis>` check (idea 3 above), and one-tap buttons for the candidate spellings (accent marks like `níll sun` / `níllson`, comma/period breaks like `nill, sun`, alt-vowel forms like `nyllson`, the IPA stress mark `ˈnill sun`, and the drag-out-the-syllable idea like `nilllll sun`). Awaiting Alene's on-phone test results.
2. **Check which iOS voice Alene is using, and try others.** Stress handling is voice-specific. A different iOS voice — or Safari versus Chrome (the app runs in both; see the 2026-06-03 entry below on Safari being the more reliable speech engine on iPhone) — may stress the name differently. Worth checking whether the app exposes a voice picker.
3. **SSML `<emphasis>` or `<prosody>`** on the utterance. Likely a dead end — the Web Speech API generally ignores SSML on iOS / WebKit — but worth a quick confirm; if any subset works, it is the real "stress this syllable" dial we have been missing.
4. **A comma**: `nill, sun`. Forces "nill" to end an intonation phrase, which can pull a nuclear accent onto it. Downside: probably adds an audible pause, and the trailing "sun" may take its own accent. Considered but not tried, because of the pause.
5. **A stress diacritic** on the first vowel (e.g., `níll`). Some engines honor it; the risk is that it flips the vowel back to long. Not tried.
6. **Drag out the first syllable with repeated letters** (Alene's idea, 2026-06-16). Stress is carried largely by duration, so lengthening the first syllable may make the voice perceive it as stressed — a different lever than spelling/hyphen/accent. Repeat the **L** (e.g., `nilll sun`, `nilllll sun`) to add length without changing the vowel; repeating the **i** (e.g., `niiill`) risks flipping the vowel back to long "ee". Queued in the sandbox; awaiting on-phone results.
7. **Accept "nill-SUN"** as the practical best — the correct sound with a minor emphasis quirk. Possibly the right place to stop if the above do not pan out. (Offered to Alene on 2026-06-16; she chose to continue in a future session instead.)

**Conversation**: Investigated in a chat session between Alene and Claude on 2026-06-16. Commit range `5c03d4e`…`10414b1` on [alene-pixel/email-voice-reader](https://github.com/alene-pixel/email-voice-reader).

---

## 2026-06-03 — Intermittent failures in Chrome on iPhone

**Status**: Open. Not blocking at the moment of report — see below.

**Reporter**: Alene.

**Symptom**: The Voice Email Reader has been working only intermittently in Chrome on iPhone for the past several weeks. It works sometimes and fails other times. Safari on iPhone has continued to work reliably the whole time. Alene has not been using the app on desktop much recently, so the desktop side has not been observed during this same window.

**Environment as of report**:
- Chrome on iPhone: version `148.0.7778.166`.
- Current latest stable Chrome for iOS at time of report: `149.0.7827.45`, released June 2, 2026. Alene was one major version behind when the symptom was being reported.
- iOS version on Alene's iPhone: not captured.

**Latency status as of June 3, 2026**: Chrome on iPhone is working for Alene at the moment of this report, so the issue is currently latent rather than blocking. This entry is being preserved so we can return to it if the symptom comes back.

**What we investigated on June 3, 2026**:

- Chrome on iPhone still uses Apple's WebKit engine, not Google's Blink. Google's Blink-on-iPhone port is in development under the European Union's iOS 17.4 rules but has not shipped to general users as of mid-2026. See [Chrome iOS Browser on Blink — Igalia, Aug 2024](https://blogs.igalia.com/gyuyoung/2024/08/08/chrome-ios-browser-on-blink/).
- Chrome 148 (released May 6, 2026) patched 151 vulnerabilities. Three of those touched Chrome's Speech component specifically:
  - CVE-2026-3916 — out-of-bounds read in Web Speech (High severity).
  - CVE-2026-7935 — UI spoofing in Chrome's Speech component (Medium). See [CVE-2026-7935 writeup — WindowsNews](https://windowsnews.ai/article/cve-2026-7935-chrome-ui-spoofing-via-speech-component-fixed-in-version-148.417078).
  - CVE-2026-7960 — race condition in Chrome's Speech component (Medium). See [CVE-2026-7960 — WindowsForum](https://windowsforum.com/threads/cve-2026-7960-chrome-speech-race-patch-now-to-close-renderer-memory-leak-risk.417021/).
- The timing of those three Speech-component patches — about one month before Alene's report — aligns circumstantially with the symptom showing up. The [Chrome 148 release notes](https://developer.chrome.com/release-notes/148) do NOT call out any observable Web Speech behavior change for legitimate apps. Google does not publish exact technical details of security fixes, to limit exploit risk for unpatched users.
- There is precedent for security-patch regressions to the Speech component. The Chrome 130 incident in late 2024 silently broke `speechSynthesis` for many users due to a Google Voice loading change, fixed in Chrome 131. See [Chromium issue 374263394](https://issues.chromium.org/issues/374263394).
- We did not find a public report specifically describing a Chrome 148 Web Speech regression on iPhone. The link to Chrome 148 is circumstantial.
- The Web Speech API has long-standing iOS limitations that affect every iPhone browser, including iOS's autoplay policy extending to `speechSynthesis.speak()` and quirks in `SpeechRecognition`. See [Lessons Learned Using the speechSynthesis API — talkrapp.com](https://talkrapp.com/speechSynthesis.html) and [Web Speech API bugs in iOS — Apple Developer Forums](https://developer.apple.com/forums/thread/694847). Safari handles these edge cases more gracefully than Chrome's WebKit wrapper does, which would explain the "Safari works, Chrome is flaky" pattern even if no specific 2026 regression exists.

**If the symptom returns, in this order**:

1. **Update Chrome on iPhone to 149 or later** if not already done, and re-test.
2. **If still failing, default to Safari on iPhone** for the Voice Email Reader. Safari has tighter integration with iOS's audio session and microphone permissions than Chrome's WebKit wrapper does, so this is a clean workaround.
3. **Consider adding a small in-app banner suggesting Safari for iPhone users**, so other iPhone users (and future-Alene on a fresh device) get a nudge.
4. **Consider filing a public Chromium bug** capturing the symptom — intermittent Web Speech failures in Chrome on iPhone since approximately May 2026 — so a real regression, if one exists, can be diagnosed and fixed for everyone.

**Conversation**: Investigated in a chat session between Alene and Claude on June 3, 2026.
