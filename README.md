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
| `tests/e2e.screens.js` | Headless-browser smoke test and screenshots (needs Playwright and Chrome). |

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
If you host with the service worker, change `VERSION` in `sw.js` after editing so devices pick up the update.

## How the checking works
* Text and transcript are normalised: case, punctuation, curly quotes and accents are ignored, and
  hyphenated words are split ("sea-chest" = "sea chest" = "seachest"). Digits become words
  ("1863" = "eighteen sixty three" or "one thousand eight hundred and sixty three"), and common
  homophones ("their/there", "to/two") and UK/US spellings ("colour/color") count as the same,
  because they sound the same.
* A word-level Needleman-Wunsch/Levenshtein alignment (substitutions weighted by spelling similarity)
  labels each word as correct, substituted (shows what was heard), skipped, or extra.
  Words after the point where he stopped reading are "not reached" and don't count against him.
* Accuracy = correct ÷ words attempted. WPM = words attempted ÷ minutes from Start to Stop.
* Tapping a flagged word marks it as a "recogniser error", and the scores are recalculated and saved.

## Extension points (coming soon)
* `questions` and `keyPoints` arrays per text are already supported in the data (sample on the
  "Volunteers Bring Forgotten Canal Path…" text). The reading screen shows "Coming soon" cards for
  Summary, Key points and Questions. Build these as new routes (`#/summary/<id>` etc.) in the
  `route()` function and save answers alongside attempts.

## Privacy note
The app stores nothing off the device. **However, the speech recognition is the browser's own
service**: on iPhone/iPad (Safari, Siri dictation) and in Chrome/Edge, audio may be sent to
Apple, Google or Microsoft servers to be transcribed. Turning on on-device dictation (iOS settings)
may keep it local on newer devices, but the app cannot control or check this.
