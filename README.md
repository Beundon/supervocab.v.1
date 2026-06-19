# Vocabmaster.V.1.1
# This is a superapp using to learn vocabulary in English about 10k words in various area. 

A self-contained English vocabulary builder app — 10,100 pre-loaded words, offline-first, installable on any device. No server, no database, no signup required to use (signup is local only, stored in the browser via `localStorage`).

> Single HTML file. Open it and it works. That's the whole pitch.

---

## What's inside

| Feature | Details |
|---|---|
| Words pre-loaded | 10,100 entries (cleaned from `comprehensive_vocab_10k.csv`) |
| Fields per word | Word · Part of Speech · Definition · Category · Example Sentence · Synonyms · Difficulty |
| Difficulty levels | Easy (A1/A2) · Medium (B1/B2) · Hard (C1/C2) — mapped from CEFR |
| Practice modes | Flashcards · Quiz · Spelling · Review |
| Progress tracking | Mastery %, review count, streak counter |
| User accounts | Local-only (username + password, hashed in-browser) |
| Import / Export | CSV · XLSX · JSON |
| Storage | `localStorage` — per-user word lists |
| Dependencies | Chart.js (CDN), SheetJS/XLSX (CDN) — both optional if import/export not used |

---

## Files in this project

```
vocabmaster.V.1.1.html          ← the app (single file, drop anywhere)
README.md                        ← this file
comprehensive_vocab_10k.csv      ← source dataset (10,100 rows)
build_vocabmaster.py             ← rebuild script (CSV → HTML)
verify_vocabmaster.js            ← sanity check (parses JS, validates count)
```

---

## Quick start

### 1. Just open it
Double-click `vocabmaster.V.1.1.html` → opens in your default browser → sign up → all 10,100 words are auto-loaded into your account.

> **First login is slow** (~2–3 seconds) because the 10k words are being seeded into `localStorage`. Subsequent logins are instant.

### 2. Run locally with a server (recommended)
Some browsers block `localStorage` on `file://` URLs. Use any static server:

```bash
# Python
python3 -m http.server 8000
# → open http://localhost:8000/vocabmaster.V.1.1.html

# Node (one-shot)
npx serve .

# Or VS Code: install "Live Server" extension → right-click → Open with Live Server
```

---

## How the 10k words get loaded

The HTML contains a single JS array:

```js
const DEFAULT_VOCABULARY = [
  ["Alleviate","Verb","Make suffering or a problem less severe.","Business & Work",
   "This medicine will help alleviate the pain.","relieve, ease","easy"],
  // ... 10,099 more rows
];
```

When a user logs in and their word list is empty, `ensureDefaultVocabulary()` runs and seeds all entries. Users who already have words are never overwritten — so existing accounts keep their progress.

To rebuild from an updated CSV:

```bash
python3 build_vocabmaster.py
# → regenerates download/vocabmaster.V.1.1.html
```

---

## Deployment options

### A. GitHub Pages (web app + installable PWA)

```bash
# 1. Create a repo, push the HTML as index.html
git init
mv vocabmaster.V.1.1.html index.html
git add . && git commit -m "VocabMaster V.1.1"
git remote add origin git@github.com:<you>/vocabmaster.git
git push -u origin main

# 2. Repo → Settings → Pages → Branch: main / root
# → live at https://<you>.github.io/vocabmaster/
```

To make it installable on Android home screen, add a `manifest.json` + `sw.js` and register a service worker. See the [PWA setup guide](#pwa-setup-optional) below.

### B. Android APK (WebView wrapper — offline, sideload)

1. Android Studio → New Project → Empty Views Activity.
2. Copy `index.html` into `app/src/main/assets/`.
3. `MainActivity.onCreate`:
   ```java
   WebView wv = findViewById(R.id.web);
   wv.getSettings().setJavaScriptEnabled(true);
   wv.getSettings().setDomStorageEnabled(true);   // required for localStorage
   wv.loadUrl("file:///android_asset/index.html");
   ```
4. Build → Build APK → install on phone.

### C. Google Play Store (Trusted Web Activity)

After Path A (PWA live on GitHub Pages):

```bash
npm i -g @bubblewrap/cli
bubblewrap init --manifest https://<you>.github.io/vocabmaster/manifest.json
bubblewrap build
# → produces signed AAB ready for Play Console upload
```

---

## PWA setup (optional)

Add three files next to `index.html`:

**`manifest.json`**
```json
{
  "name": "VocabMaster",
  "short_name": "Vocab",
  "start_url": "./index.html",
  "display": "standalone",
  "background_color": "#f8fafc",
  "theme_color": "#6366f1",
  "icons": [
    { "src": "icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

**`sw.js`** (cache-first, offline-capable)
```js
const CACHE = 'vocab-v1';
self.addEventListener('install', e => {
  e.waitUntil(caches.open(CACHE).then(c => c.addAll(['./index.html'])));
});
self.addEventListener('fetch', e => {
  e.respondWith(caches.match(e.request).then(r => r || fetch(e.request)));
});
```

**In `index.html` `<head>`:**
```html
<link rel="manifest" href="manifest.json">
<script>
  if ('serviceWorker' in navigator) navigator.serviceWorker.register('sw.js');
</script>
```

After this, Android Chrome → menu → **Install app** → icon appears on home screen, launches fullscreen, works offline.

---

## Data model

Each word stored in `localStorage` under key `vocabWordsData`:

```json
{
  "<userId>": [
    {
      "id": 17187654320001,
      "word": "Alleviate",
      "pos": "Verb",
      "definition": "Make suffering or a problem less severe.",
      "category": "Business & Work",
      "sentence": "This medicine will help alleviate the pain.",
      "synonym": "relieve, ease",
      "difficulty": "easy",
      "notes": "",
      "date": "2026-06-19T03:26:00.000Z",
      "mastery": 0,
      "reviews": 0,
      "lastReview": null
    }
  ]
}
```

Other keys:
- `vocabUsers` — list of registered accounts
- `vocabSession` — current session token
- `streak_<userId>` — daily streak count
- `lastActive_<userId>` — last active date string

---

## Rebuilding from CSV

The build script cleans the source CSV before injecting:

| Source artifact | Cleanup |
|---|---|
| `Alleviate (ID-1)` | `Alleviate` |
| `...severe. [Context variant 1]` | `...severe.` |
| `...pain. (Observation sequence #1)` | `...pain.` |
| `Adj` / `Adv` | `Adjective` / `Adverb` |
| `A1` / `A2` | `easy` |
| `B1` / `B2` | `medium` |
| `C1` / `C2` | `hard` |

Run:

```bash
python3 build_vocabmaster.py
node verify_vocabmaster.js   # optional sanity check
```

Output:
- `download/vocabmaster.V.1.1.html` (~2.0 MB, 12,556 lines)

---

## Privacy

- All data lives in the user's browser. Nothing is sent anywhere.
- "Accounts" are local-only — password is hashed with a non-cryptographic `simpleHash()` (good enough for casual use, **not** for real security).
- Clearing browser data wipes everything. Use the in-app Export button to back up.

---

## Limitations & known issues

- `localStorage` has a ~5 MB cap per origin. 10,100 words take ~3 MB, leaving room for ~5,000 more user-added words before quota errors. For larger datasets, switch to IndexedDB.
- CDN dependencies (Chart.js, SheetJS) require internet on first load. For full offline, vendor them locally.
- No multi-device sync. Each browser/device has its own dataset.
- The auto-seed only runs when the user's word list is empty — to re-seed, log out, clear site data, log back in.

---

## Roadmap ideas

- [ ] IndexedDB backend (lift the 5 MB cap)
- [ ] CEFR filter (A1/A2/B1/B2/C1/C2) instead of just Easy/Medium/Hard
- [ ] Spaced repetition (SRS) scheduler based on `mastery` + `lastReview`
- [ ] Cloud sync via GitHub Gist or Firebase
- [ ] Dark mode toggle
- [ ] Audio pronunciation (Web Speech API — already works in modern browsers, just needs a button)

---

## License

MIT — do whatever you want. Attribution appreciated but not required.

## Credits

- App template: original `vocabmaster.V.1.0.html`
- Dataset: `comprehensive_vocab_10k.csv` (10,100 entries, CEFR-tagged)
- Libraries: [Chart.js](https://chartjs.org), [SheetJS](https://sheetjs.com)

