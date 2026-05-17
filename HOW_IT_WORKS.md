# TipOdds — How It All Works

A plain-English guide to every part of the system: what each service does, how they connect, and how changes get made.

---

## The Big Picture

```
You (Nathan)
    │
    ├── talks to Claude Code (AI developer)
    │       │
    │       └── edits index.html on your PC
    │               │
    │               └── pushed to GitHub
    │                       │
    │                       ├── Vercel auto-deploys the live site
    │                       │
    │                       └── GitHub Actions runs every 6 hours
    │                               │
    │                               ├── fetches odds from The Odds API
    │                               └── fetches fixtures/results from football-data.org
    │                                       │
    │                                       └── saves everything to Firebase
    │                                               │
    │                                               └── app reads live data from Firebase
```

---

## Every Service Explained

### 1. Your PC — Where Development Happens
- The working folder is `C:\Users\admin\Downloads\clause code tipping\tipodds-for-claude-code`
- This is a local copy of the code. Changes made here are **not live** until pushed to GitHub.
- Claude Code (see below) reads and edits files here.

### 2. Claude Code — The AI Developer
- Claude Code is an AI tool made by Anthropic that reads your code and makes changes on your behalf.
- You describe what you want in plain English ("make the names smaller", "remove the Groups page") and Claude edits the files.
- It reads `CLAUDE.md` automatically — this file tells it how the app works, what rules to follow, and what not to break.
- Claude Code **cannot** push to GitHub on its own — you approve each push.
- Think of it as a developer who works on your local files, then you hit "send" to publish.

### 3. GitHub — Version Control & Storage
- **Repo:** github.com/NT-create987/Tipping_odds
- Every version of the code is saved here. If something breaks you can roll back.
- GitHub also runs the automated update job (GitHub Actions) every 6 hours.
- **Branch:** `main` — the only branch. Whatever is on `main` is what Vercel deploys.
- **Secrets stored here:** `FIREBASE_DB_SECRET` (used by the automated workflow to write to the database securely)

### 4. Vercel — Hosting (The Live Website)
- **Live URL:** tippingoddsbased.vercel.app
- Vercel watches the GitHub repo. The moment anything is pushed to `main`, Vercel automatically rebuilds and republishes the site within about 30 seconds.
- No manual deploy step needed — push to GitHub = site is updated.
- The app is a single HTML file so there's no build step. Vercel just serves it as-is.

### 5. Firebase — The Database
- **Project:** tip-odds (asia-southeast1 region)
- Firebase is where all live data lives: users, pools, tips, match results, odds, leaderboards.
- The app talks to Firebase directly from the browser using Firebase's JavaScript SDK.
- The automated workflow also writes to Firebase (via its REST API) to push updated odds and results.
- **Security rules:** set to require login — only authenticated users can read or write. The automated workflow uses a database secret for access.

### 6. The Odds API — Live Bookmaker Odds
- **Website:** the-odds-api.com
- Provides live bookmaker odds (e.g. Home 2.10 / Draw 3.40 / Away 3.80) for football matches.
- Called from two places:
  - **GitHub Actions** every 6 hours to refresh stored odds in Firebase
  - **The browser** (live) when a user opens the Tips page, to show the freshest odds available
- Free tier has a monthly request limit — the 6-hour schedule is deliberately conservative to stay within it.

### 7. football-data.org — Fixtures & Results
- Provides the match schedule (who plays who, when, where) and final scores.
- **Only called from GitHub Actions** — this API blocks browser requests (CORS), so it can't be called directly from the app.
- The workflow imports new matches and writes results as they come in.
- Results are also entered manually by you (admin) in the Admin → Results page for any matches the API misses.

---

## How a Change Gets Made

1. You tell Claude Code what you want (e.g. "make the team names smaller")
2. Claude edits `index.html` on your PC
3. Claude runs integrity checks to make sure nothing is broken
4. You say "push through"
5. Claude pushes to GitHub
6. Vercel detects the push and deploys the new version (~30 seconds)
7. Done — the change is live

---

## How Data Updates Automatically

Every 6 hours, GitHub Actions runs the script in `.github/workflows/update-data.yml`:

1. Looks at every comp stored in Firebase
2. For each comp with an odds key → fetches latest odds from The Odds API → writes to Firebase
3. For each comp with a football-data code → fetches latest fixtures and results → writes to Firebase
4. Auto-locks rounds that are within 7 days (so tipping closes before kick-off)

You can also trigger this manually: GitHub repo → **Actions** tab → **Auto-update odds and results** → **Run workflow**.

---

## The App Itself — How It's Built

The entire app is a **single HTML file** (`index.html`, ~2,600 lines). It contains:

| Section | What it does |
|---|---|
| `<style>` | All visual styling (colours, fonts, layout) |
| `<div>` pages | The HTML structure of every page (Tips, Leaderboard, History, etc.) |
| `<script type="module">` | All the app logic in JavaScript |

There is **no build step, no npm, no node_modules**. It's intentionally kept as one file so it's easy to edit, deploy, and understand.

The app uses **ES Modules** — a modern JavaScript feature that keeps the code organised but means variables aren't accessible from the browser console (they live in "module scope").

### Pages in the app
| Page | What it shows |
|---|---|
| Tips | Pick your match outcomes for the current round |
| Leaderboard | Points standings for your pool |
| History | Your past tips and results |
| Bonuses | Joker picks, tournament winner, top scorer |
| Standings | League table (auto-calculated from results) |
| Bracket | Knockout bracket for tournaments |
| Admin | Comp management, results entry, odds, user management |

---

## Repo Structure

```
Tipping_odds/
│
├── index.html                        ← The entire app (HTML + CSS + JavaScript)
│
├── .github/
│   └── workflows/
│       └── update-data.yml           ← Automated odds + results updater (runs every 6 hours)
│
├── CLAUDE.md                         ← Instructions for Claude Code — tells the AI how the app works
├── CLAUDE_CODE_SETUP.md              ← How to set up Claude Code on a new machine
├── HOW_IT_WORKS.md                   ← This document
├── README.md                         ← Short overview for anyone visiting the GitHub repo
└── vercel.json                       ← Vercel config (sets security headers)
```

---

## Suggested Improvements to the Repo

The current structure is fine for a one-person project. Here are optional improvements worth considering:

### Add a `dev` branch (low effort, high value)
Right now all changes go straight to `main` = straight to the live site. A `dev` branch lets you test changes on a separate Vercel preview URL before pushing them live.
- How: Create a `dev` branch in GitHub, connect it to Vercel as a preview deployment
- Work is done on `dev`, then merged to `main` when you're happy

### Add a `docs/` folder
If you want to keep admin guides, player guides, or notes, a `docs/` folder keeps them organised without cluttering the root.

### Add a `CHANGELOG.md`
A running log of what changed and when. Useful when something breaks and you want to know what was last changed. Example:
```
## 2026-05-17
- Removed Groups page
- Team name font size reduced
- Fixed round tabs for tournaments
```

### What NOT to change
- Don't split `index.html` into multiple files — the simplicity of one file is intentional and makes it easy for Claude Code to edit reliably.
- Don't add a build system (webpack, vite, etc.) — it adds complexity with no real benefit for a project this size.

---

## Accounts & Access Summary

| Service | Account | Used for |
|---|---|---|
| GitHub | NT-create987 | Code storage, version history, automated workflow |
| Vercel | (linked to GitHub) | Hosting the live site |
| Firebase | tip-odds project | Database (users, pools, tips, odds, results) |
| The Odds API | — | Live bookmaker odds |
| football-data.org | — | Fixtures and results |
| Anthropic (Claude) | admin account | Claude Code AI developer |
