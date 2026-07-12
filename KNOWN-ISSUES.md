# Voice Email Reader — Known Issues and Open Observations

Newest entries at the top. This file tracks user-reported issues with the Voice Email Reader that have not been resolved with a code change, but are worth keeping a record of so future-Claude (or future-Alene) can pick up the thread later. Once an issue is closed, mark it Resolved and keep the entry for reference.

---

## 2026-07-12 — App freezes on iPhone AND desktop [OPEN — cause not yet found]

**Reporter**: Alene (onset ~July 7–10, 2026; fine for months before). Reported 2026-07-10; investigated 2026-07-12.

**Symptom**: Intermittent freeze, on iPhone (both Safari and Chrome) AND on desktop. Most consistent form: stuck indefinitely on the startup screen "Fetching your emails... One moment please," right after the spoken "Let me check your inbox." Earlier form: reads one-to-a-few emails, then freezes right after a command like archive or mark-as-read. "It kinda freezes / just stops." **No error is shown** — so it is a SILENT hang on an unresolved `await`, not a thrown exception (the `try/catch` in `onMicSelected` would otherwise display "Could not fetch emails").

**Ruled out (with method)**:
- GitHub outage — status all-green; app serves fine (WebFetch of the live page).
- Gmail outage — Google Workspace Status Dashboard all-green.
- Connection — same freeze on Wi-Fi and cellular; ordinary Google search works throughout.
- Phone memory — a full power-off/on restart did not help.
- Stale cache — Alene cleared iPhone Safari history and reloaded; still froze (so she was running the newest code).
- Recent app changes — the July 7–9 commits (repeat-guard `4c4bab3`, batch-memory `1cb34de`, matchArchive `c1002bf`, altacpa `883a865`, markread-during-attachment `514d240`) do NOT touch the startup / sign-in / fetch path. That path (`onMicSelected` → `fetchUnreadEmails` → `fetchWithAuth`) is unchanged for months.
- Speech-hang theory — a `speakChunk()` watchdog was shipped (`1945575`) to force-resolve a hung utterance; it did NOT fix the freeze. Reverted per the bug-fix philosophy (don't leave an unsuccessful fix in the code). So this is NOT the iOS speechSynthesis `onend`-never-fires hang.
- Big / malformed email — checked `sizeEstimate` of all 15 unread-in-inbox messages via the Gmail API on 2026-07-12: largest 0.25 MB, total 0.64 MB. No monster email; not a large-attachment stall.
- iOS 26.5.2 (early-July 2026 security update) as the sole spark — weakened: desktop is not iOS, so an iOS-only update cannot be the whole story.

**Where it points now**: a SILENT stall in the post-sign-in path that every device shares — the inbox fetch (`fetchWithAuth` has NO per-request timeout, so any stalled request hangs forever) or the auth / token / Google Identity Services layer. Cross-device + tiny emails + internet-up + Gmail-up + unchanged app path together point away from app logic and email content, toward the account/auth/Google side or a network-level stall on one request.

**Leading hypothesis (2026-07-12)**: the freeze leaves the *pre-fetch* status showing, so execution hangs in `onMicSelected`'s chain BEFORE the inbox fetch returns — i.e. `speak()` (welcome), `setMicDevice()` (getUserMedia claim of the chosen mic), or the fetch itself. The **mic-claim step is a strong suspect**: it runs right after the welcome and calls getUserMedia, which a recent OS/browser security update could have started blocking or hanging; it fits cross-device + sudden onset, and the Claude Browser-pane mic-block popups drew attention to it. (Speech-hang is not fully excluded — the reverted watchdog may simply never have loaded, due to GitHub Pages caching during Alene's test.)

**Diagnostic build DEPLOYED 2026-07-12**: a TEMPORARY step-by-step progress readout in `onMicSelected` + `fetchUnreadEmails`. The second status line now names the current step — "Step 1 of 4: speaking welcome…", "Step 2 of 4: preparing microphone…", "Step 3 of 4: contacting Gmail… / loading email N of M…", "Step 4 of 4: starting to read…" — so whichever step is frozen on-screen pinpoints the stall. Also `console.log`'d with a `[startup]` prefix. AWAITING Alene's on-device report of the frozen step.

**Then — targeted fix by step** (and remove the diagnostic readout as part of it):
- Step 1 (speech): decouple the inbox fetch from the welcome speech so a hung utterance can't block it (more robust than the reverted watchdog).
- Step 2 (mic): timeout `setMicDevice`/getUserMedia and make a blocked/failed mic non-fatal — fall back to the default mic and proceed to the fetch.
- Step 3 (fetch): per-call timeouts on `fetchWithAuth` via `AbortController` (short for list/modify, long for `getAttachment`), so a stalled request becomes a visible, recoverable error instead of an infinite freeze.

**Also observed 2026-07-12 (Claude Browser pane, a quirky env — treat as a weak signal)**: visual stuck on "Fetching" while audio read ahead; after "mark as read" the confirmation fired (button turned purple, spoken) but the app did not advance to the next email. Hints at a view-render desync separate from the startup hang; revisit only if the step readout doesn't explain the real-device freeze.

**Status**: OPEN. Do not mark resolved until Alene confirms on-device.

---

## 2026-07-07 — iPhone speech delay [OPEN] + double-command side effect [MITIGATED]

**Reporter**: Alene.

**Two linked issues:**

**Issue 1 — the speech delay (OPEN, likely not fixable here).** On iPhone there is a lag between when Alene finishes speaking a command and when the app reacts. During that lag the app keeps reading aloud, which makes it feel like the app did not hear her.

- **Root cause**: In command mode the app runs speech recognition with `continuous = false` and `interimResults = false` (see `SpeechService.initRecognition`). So the app is handed **nothing** until the recognizer decides the utterance is finished (its end-of-speech detection waits for a pause after she stops talking) and returns the single final result. Only when that final result arrives does `onresult` call `this.cancel()` to stop the reading. Everything between "she stops speaking" and "final result arrives" is the perceived delay, and the app is still reading during it.
- **Why it is hard to remove**: The obvious speed-up is to turn on interim results in command mode and cancel the reading the instant ANY speech is detected. But the microphone also picks up the app's own text-to-speech, so interim results in command mode produce phantom commands from the app hearing itself. The app already added mic echo cancellation (commit `e7ad3e6`) and startup-echo guards (`f4bf530`) to fight this class of problem; a full interim-results command mode would reopen it. Treated as inherent to iOS Web Speech for now. Alene has accepted this ("it seemed to be unfixable, which is okay").

**Issue 2 — the command ran twice (MITIGATED 2026-07-07).** Because of Issue 1, Alene sometimes said a command, thought it was not heard, and repeated it. The first command ran on the email she meant; the repeat then ran on the **next** email (which had already loaded and started reading), doing something she never wanted there — e.g. archiving an unheard email.

- **Fix shipped**: A rapid-repeat guard, keyed on the command **id**. The controller remembers the last voice command it ran (`lastExecutedCommandId` / `lastExecutedCommandTime` in `handleCommand`). A pure pre-check, `shouldSuppressRepeat`, runs inside `SpeechService.onresult` **before** `this.cancel()`, via the `onCommandShouldSuppress` callback. If the same command id is heard again within `repeatCommandWindowMs` (default 3000 ms), it is ignored — and because the pre-check returns before `cancel()`, the email currently being read is **not** interrupted (no falling-silent). This is the voice analog of the on-screen button double-tap guard (`buttonsLockedUntil`, commit `543b507`). Full write-up: CHANGELOG.md, 2026-07-07.
- **Only identical repeats are dropped.** A *different* command said right after always executes (keyed on id, not a blanket lock). For "next"/"previous" this is also the desired behavior — a repeated "next" during the delay now skips one email instead of two.
- **Open tuning question (needs Alene's on-phone testing)**: Is 3000 ms the right window? Too long and it could swallow an *intended* quick second command; too short and a double still slips through when she waits longer before repeating. It is a single constant (`repeatCommandWindowMs`) in the controller constructor, easy to adjust. Cannot be verified from Claude's side — iOS speech timing is empirical, same loop as the Nilsson pronunciation work.

---

## 2026-06-16 — "Nilsson" TTS pronunciation [REOPENED] — period form failed in-app; must tune in context

**Status**: OPEN (reopened 2026-06-16). The period form `nill. sun` won in the standalone sandbox but REGRESSED to "neel-SUN" once live — the worst of both worlds (wrong vowel AND wrong stress). **Root cause**: the app speaks the surname mid-sentence ("Isabella nill. sun is …"), where the inserted period becomes an internal phrase break that flips both vowel and stress on the default voice; the sandbox had spoken the name in isolation / phrase-final and never surfaced this. App reverted to `nill sun` (short "i," stress slightly late) — the best in-app result so far. **Key lesson: tune candidates IN CONTEXT — embedded in a full sentence, processed the same way the app processes email text — not in isolation.** The sandbox (`nilsson-sandbox.html`) is being upgraded to do exactly that. **Next idea: `Nill's son`** (Isabella's — the surname derives from "Nils's son"; English possessive stress naturally falls on the first word, "NILL's son," which could fix the vowel and the stress at once).

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
| 10 | `nill. sun` (period after "nill") | sandbox ✓ / app ✗ | sandbox ✓ / app ✗ | sandbox said "NILL-sun" but the **app said "neel-SUN"** — the period became a mid-sentence break in real emails; reverted |

**Diagnosis — what the pattern shows**:
- **The voice has two modes, and they trade off against each other:**
  - **One word** (any spelling — Nilson, Nillson, Nihlsun) → always the WRONG vowel ("NEEL"), but the RIGHT (first-syllable) stress. The voice treats a single unfamiliar "Ni…" token as a long-"ee" Scandinavian name — which it literally is (Nilsson is a Swedish surname).
  - **Hyphen or space** (Nill-sun, nill sun) → always the RIGHT short "i" (the isolated "nill" reads like "pill" / "nil"), but the WRONG (second-syllable) stress.
- **In the hyphen/space modes the stress is positional, not vowel-driven.** Respelling the 2nd syllable three different ways — `sun` (full vowel), `suhn` (schwa), `son` (a real surname suffix that voices usually de-accent) — did NOT move the stress off the 2nd syllable. The voice seems to simply stress the last syllable of a hyphenated or two-word item.
- **Capitalization does not affect stress.** `NILL-sun` (caps on the first syllable) still stressed the 2nd syllable.

**Current live state**: `'Nilsson': 'nill sun'` → "nill-SUN" in-app (short "i," stress slightly late). Reverted here on 2026-06-16 after the period form `nill. sun` regressed to "neel-SUN" in real emails (see Status). Best in-app result so far; the first-syllable emphasis is still being worked, now via in-context sandbox testing.

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
