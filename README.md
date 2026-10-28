# Read Aloud: reading-practice prototype

A small offline-friendly web app. Pick a text, read it aloud, and the browser's speech
recogniser makes a transcript. The app compares the transcript with the text word by word and
shows accuracy, words per minute, words missed or misread, and words to practise. History is kept
on the device (localStorage) and can be exported as JSON or CSV.

No server, no accounts, no tracking. Everything is in `index.html` (HTML, CSS and JS inline).

## Files
| File | Purpose |
|---|---|
| `index.html` | The whole app. It works on its own. |
| `manifest.json`, `sw.js`, `icon*.png`, `icon.svg` | Optional: let it be added to the home screen and work offline (PWA). |
| `tests/align.test.js` | Unit tests for the word-alignment logic: `node tests/align.test.js` |
| `tests/whisper.test.js` | Unit tests for the OpenAI Whisper engine (request building, response parsing, word-timing WPM, error handling), using mocked API responses: `node tests/whisper.test.js` |
| `tests/e2e.screens.js` | Headless-browser smoke test and screenshots (needs Playwright and Chrome). |
| `tests/e2e.whisper.js` | Headless test of the Whisper flow with a mocked microphone/MediaRecorder and mocked OpenAI API. |

The `tests/` folder is not needed on the website. Upload it or not, as you prefer.

## Hosting (HTTPS is required for the microphone)
iPhone/iPad Safari only allows speech recognition on an `https://` page. Opening the file directly
(`file://`) works for browsing texts and the typed-transcript mode, but not the microphone.
Any static host works. Upload this folder as it is:
* **Netlify Drop**: https://app.netlify.com/drop, drag the folder in (free account to keep it).
* **GitHub Pages**: new repo → upload files → Settings › Pages → deploy from `main`.
* **Cloudflare Pages**: Workers & Pages › Create › Pages › Upload assets.
Then on iPad/iPhone open the link in Safari → Share → **Add to Home Screen**.

## Adding or editing texts
Open `index.html` in a text editor and find `const TEXTS = [` (near the bottom, in the
`<script id="texts-data">` block). Copy an existing entry and change it:

```js
{
  id: "story-my-new-text",          // unique, no spaces; history is linked to this
  title: "My New Text",
  type: "story",                     // news | essay | letter | review | excerpt | story | speech | poem
  level: 3,                          // 1 easiest … 5 hardest
  source: "Written for this app",    // credit line; say "Public domain" for old texts
  body: `First paragraph…

Second paragraph (blank line between paragraphs; single line breaks are kept, good for poems).`,
  // Optional, for later phases:
  keyPoints: ["…", "…"],
  questions: [ { q: "Why…?", type: "short", answer: "…" },
               { q: "Which…?", type: "choice", options: ["A", "B", "C"], answer: "B" } ]
},
```
Use back-ticks around `body` so quotes and apostrophes need no escaping. Avoid using a back-tick inside
the text. Write numbers as words where you can (digits work too: "40" matches "forty").
If you host with the service worker, change `VERSION` in `sw.js` (currently `read-aloud-v2`) after editing so devices pick up the update.

## How the checking works
* Text and transcript are normalised: case, punctuation, curly quotes and accents are ignored, and
  hyphenated words are split ("sea-chest" = "sea chest" = "seachest"). Digits become words
  ("1863" = "eighteen sixty three" or "one thousand eight hundred and sixty three"), and common
  homophones ("their/there", "to/two") and UK/US spellings ("colour/color") count as the same,
  because they sound the same.
* A word-level Needleman-Wunsch/Levenshtein alignment (substitutions weighted by spelling similarity)
  labels each word as correct, substituted (shows what was heard), skipped, or extra.
  Words after the point where he stopped reading are "not reached" and don't count against him.
* Accuracy = correct ÷ words attempted. WPM = words attempted ÷ minutes (Start to Stop for the built-in engine; first to last spoken word for Whisper).
* In the CSV export, `seconds` is the time used for WPM, `recordedSeconds` is Start to Stop, and `engine`/`timing` say which was used.
* Tapping a flagged word marks it as a "recogniser error", and the scores are recalculated and saved.

## Transcription engines
Choose in **Settings › Transcription engine**:

* **Built-in (free, instant)**: the browser's own speech recogniser (Web Speech API). Words appear live
  while he reads. Words per minute is timed from Start to Stop.
* **OpenAI Whisper (more accurate)**: usually much better with accents, names and older vocabulary.
  The whole reading is recorded on the device (MediaRecorder: `audio/mp4` on iPhone/iPad Safari,
  `webm/opus` in Chrome/Edge/Firefox). After Stop, the app shows "Checking your reading…" and sends the audio straight from the
  browser to `https://api.openai.com/v1/audio/transcriptions` (`whisper-1`, `language=en`,
  `response_format=verbose_json`, word timestamps, `temperature=0`). No words appear while he is reading.
  * **The original text is deliberately NOT sent as a prompt.** Whisper leans towards any prompt it is
    given, so that would hide his real mistakes.
  * Words per minute uses the time from his **first to last spoken word** (from Whisper's word
    timings), so the pauses before and after he reads don't count. The results screen says so.
  * If the upload fails (wrong key, no internet, no credit, rate limit, network/CORS problem,
    file over 25 MB), the recording is **kept in memory** with buttons to try sending again, type or paste instead,
    save the recording, or discard it. It survives moving between screens, but not closing the app.
  * The engine stays Built-in until a key is saved. Saving a key offers to switch.

### Setting up an OpenAI key
1. On platform.openai.com create a **separate project** for this app, give it a **low monthly
   budget/spend limit**, and create an API key in that project.
2. In the app: Settings › paste the key › **Save key** › **Test key** (lists models; costs nothing) ›
   switch to Whisper.
3. **Remove key** deletes it from the device. Also revoke it on the OpenAI website if a device is lost.

**Cost:** about **$0.006 per minute of audio** (roughly 0.6 cents) at OpenAI's published
whisper-1 price when this was written, so a 2-minute reading costs about 1 cent. **Check OpenAI's current pricing page**.

**Key safety:** the key is stored in this browser's localStorage on this device only. It is sent only to
OpenAI, in the request header. Anyone with access to the unlocked device and browser could in principle
read it, which is why a dedicated key with a low spend limit is recommended. It is not included in the
JSON/CSV exports.

## Extension points (coming soon)
* `questions` and `keyPoints` arrays per text are already supported in the data (sample on the
  "Volunteers Bring Forgotten Canal Path…" text). The reading screen shows "Coming soon" cards for
  Summary, Key points and Questions. Build these as new routes (`#/summary/<id>` etc.) in the
  `route()` function and save answers alongside attempts.

## Privacy note
The app has no server and stores nothing off the device.
**If Whisper is switched on**, each reading's audio is sent to OpenAI for transcription (under
OpenAI's API data terms). Nothing goes to OpenAI with the built-in engine.
**With the built-in engine, the speech recognition is the browser's own service**: on iPhone/iPad (Safari, Siri dictation) and in Chrome/Edge, audio may be sent to
Apple, Google or Microsoft servers to be transcribed. Turning on on-device dictation (iOS settings)
may keep it local on newer devices, but the app cannot control or check this.
