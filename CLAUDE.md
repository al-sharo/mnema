# Greek Flashcards PWA — Project Context

## What this is
A personal Progressive Web App for learning Modern Greek, built for one user (Sasha) who is a Russian speaker studying Greek in Cyprus. The app is installable on iPhone from Safari and works offline.

**Live URL:** https://al-sharo.github.io/greek-cards  
**Repo:** https://github.com/al-sharo/greek-cards

---

## Tech stack
- **Single file app** — everything is in `index.html` (HTML + CSS + JS, no build step, no frameworks)
- **Supabase** — auth (magic link / passwordless) + cloud sync for progress and custom words
- **GitHub Pages** — hosting, auto-deploys on push to main
- **Service worker** (`sw.js`) — offline support, caches the app shell
- **PWA manifest** (`manifest.json`) — makes it installable on iPhone home screen

No npm, no bundler, no framework. Just vanilla JS + Supabase JS SDK loaded from CDN.

---

## Supabase setup
- **Project URL:** `https://oqubpyoceimwwputhtei.supabase.co`
- **Anon key:** `sb_publishable__UpDT25FqPFn_dSscchCww_-Hmwpovv`
- **Auth method:** Magic link (passwordless email)
- **Tables:**
  - `progress` — SRS state per card per user (`user_id`, `card_key`, `interval`, `repetitions`, `efactor`, `due`)
  - `custom_words` — user-added vocabulary (`user_id`, `greek`, `russian`, `hint`, `deck`)
- **Row Level Security** enabled on both tables — users only see their own data

---

## App architecture

### Screens
| ID | Description |
|----|-------------|
| `#login` | Email input → magic link auth |
| `#home` | Stats, deck selector, direction toggle, study/browse/add buttons |
| `#study` | Flashcard study session with flip animation |
| `#done` | Session complete summary |
| `#browse` | Full word list with search, edit/delete for custom words |
| `#add-word` | Form to add or edit a custom word |

### Key JS objects
- **`CARDS`** — hardcoded vocabulary array (~200 words from Ελληνικά Τώρα 1+1 textbook)
- **`CustomWords`** — loads/saves user-added words to localStorage + Supabase
- **`Store`** — localStorage wrapper for SRS state
- **`App`** — session state (current deck, direction, queue, progress)
- **`SM2`** — spaced repetition algorithm (SM-2), calculates next review interval
- **`supa`** — Supabase client instance

### Card keys
Built-in cards use their index in the `CARDS` array as a string key (e.g. `"42"`).  
Custom words use `"custom_" + uuid` (the Supabase row ID).

### Sync strategy
- localStorage is the primary store (fast, offline)
- Every `Store.setCardState()` call also writes to Supabase `progress` table
- Every custom word add/update/delete syncs to Supabase `custom_words`
- On login, cloud data is fetched and merged into localStorage (cloud wins on conflicts)

---

## SRS algorithm (SM-2)
Ratings: 1=Again, 2=Hard, 3=Good, 4=Easy  
On "Again": reset repetitions, interval = 1 day  
On others: standard SM-2 interval calculation with efactor adjustment  
Intervals shown on rating buttons before user taps

---

## Vocabulary
Source: **Ελληνικά Τώρα 1+1** (Greek Now) textbook, lessons 1–3  
~200 built-in cards organised into decks:
- `greetings` — greetings and common expressions
- `phrases` — useful phrases, prepositions, particles
- `verbs` — verb conjugations
- `nouns` — nouns and adjectives
- `numbers` — 0–1,000,000
- `food` — food and drinks
- `places` — locations and places
- `custom` — user-added words (default for new words)

---

## Study modes
Three direction options on home screen:
- **🇬🇷 Greek first** — see Greek, recall Russian translation
- **🇷🇺 Russian first** — see Russian, recall Greek word
- **🔀 Both** — randomly alternates per card

---

## Design
- **Font:** Inter (single font throughout, no serifs, no italics)
- **Theme:** Dark, `#0f0f1a` background
- **Accent:** `#e8c547` (gold/yellow) — primary accent
- **Accent 2:** `#7fb3d3` (blue) — secondary, used for translations
- **CSS variables** defined in `:root`
- Mobile-first, designed for iPhone Safari
- Safe area insets handled with `env(safe-area-inset-*)`

---

## Deployment
```bash
# After making changes:
git add .
git commit -m "description of change"
git push
# GitHub Pages auto-deploys in ~60 seconds
```

---

## Known limitations / potential improvements
- Offline custom word creation is not yet queued for sync (words added offline won't sync until next login)
- No bulk import of words (e.g. from CSV)
- No statistics screen (streaks, history charts)
- No audio pronunciation
- Supabase project may "pause" after 7 days of inactivity (free tier) — unpause from supabase.com dashboard
