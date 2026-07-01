# gre-tracker
just to track my progress. It has no backend and is a cross-device study tracker built as a single HTML file using GitHub itself as a database 

_cause I need to check how fucked I am_

<img width="1056" height="555" alt="Screenshot 2026-07-01 at 11 33 16 AM" src="https://github.com/user-attachments/assets/d67cddec-1efe-45ca-9ed0-5b58466a3de1" />

> 

[![Live Demo](https://img.shields.io/badge/Live%20Demo-GitHub%20Pages-22C55E?style=flat-square&logo=github)](https://pallavikailas.github.io/gre-tracker)
[![Stack](https://img.shields.io/badge/Stack-Vanilla%20JS%20%2B%20GitHub%20API-3B82F6?style=flat-square)](https://developer.github.com/v3/repos/contents/)
[![Size](https://img.shields.io/badge/Bundle%20size-1%20file-B8860B?style=flat-square)]()
[![No backend](https://img.shields.io/badge/Backend-None-B5495B?style=flat-square)]()

---

## What it does

Tracks three metrics toward fixed targets across any number of devices — with real-time charts and persistent sync — using no server, no database, and no build step.

| Metric | Target | Control |
|--------|--------|---------|
| Study hours | 120 hrs | +1 / −1 tap |
| GRE Quant | 170 | Log individual score |
| GRE Verbal | 170 | Log individual score |

Every change writes to `data.json` in this repo via the GitHub Contents API and is immediately visible on any other device that opens the page.

---

## Architecture

The core constraint: **fully static hosting, real cross-device persistence, zero external services.**

```
┌─────────────────────────────────────────────────────────────┐
│                      GitHub Pages                           │
│                    (hosts index.html)                       │
│                                                             │
│   ┌──────────┐    reads / writes    ┌──────────────────┐    │
│   │ Browser  │ ◄──────────────────► │  GitHub Contents │    │
│   │ (any     │    api.github.com    │  API             │    │
│   │ device)  │                      │                  │    │
│   └──────────┘                      └────────┬─────────┘    │
│        │                                     │              │
│        │ localStorage                        │ commits      │
│        │ (PAT only —                         ▼              │
│        │  never in repo)          ┌──────────────────┐      │
│        └─────────────────────────►│   data.json      │      │
│                                   │   (in this repo) │      │
│                                   └──────────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

GitHub's Contents API supports full CORS from any origin so every read and write is a direct browser-to-API call with no proxy, no lambda, and no auth server.

---

## Concurrency model

GitHub's Contents API uses **SHA-based optimistic locking** — every write must include the current file's SHA, or it is rejected with a 409 Conflict. This prevents lost updates when two devices write simultaneously.

```
Device A                    GitHub                    Device B
   │                          │                          │
   │── GET data.json ────────►│                          │
   │◄─ {content, sha:"abc"} ──│                          │
   │                          │── GET data.json ────────►│
   │                          │◄─ {content, sha:"abc"} ──│
   │                          │                          │
   │── PUT {sha:"abc"} ──────►│                          │
   │◄─ 200 {sha:"def"} ───────│                          │
   │                          │                          │
   │                          │◄── PUT {sha:"abc"} ──────│
   │                          │─── 409 Conflict ────────►│
   │                          │                          │
   │                          │                        reload()
   │                          │                          │
   │                          │◄── GET data.json ────────│
   │                          │─── {sha:"def"} ─────────►│
   │                          │                          │
   │                          │                     PUT {sha:"def"} ──►
```

On a 409, the app automatically re-fetches the latest state and retries the write with the corrected SHA so, no data is lost.

---

## Write batching

Rapid inputs (e.g. tapping +1 several times) are **debounced into a single commit**. A 900 ms idle window collapses N state mutations into one GitHub API call, keeping the repo history readable.

```js
function scheduleSave() {
  clearTimeout(_saveTimer);
  _saveTimer = setTimeout(doSave, 900);  // resets on every tap
}
```

Five taps in 800 ms → one commit. One commit per logical "session of changes."

---

## Fault-tolerant asset loading

Chart.js is requested from jsDelivr on page load. If that CDN is unreachable, `init()` detects `typeof Chart === 'undefined'` and injects a fallback `<script>` from unpkg before any rendering happens. Charts appear regardless of which CDN responds first.

```js
if (typeof Chart === 'undefined') {
  await new Promise(resolve => {
    const s = document.createElement('script');
    s.src = 'https://unpkg.com/chart.js@4.4.4/dist/chart.umd.min.js';
    s.onload = s.onerror = resolve;   // continue either way
    document.head.appendChild(s);
  });
}
```

---

## Security

| What | Where stored | Ever in git? |
|------|-------------|--------------|
| GitHub PAT | `localStorage` only | ✗ Never |
| Data (hours, scores) | `data.json` in repo | ✓ Yes — intentionally |
| App code | `index.html` in repo | ✓ Yes |

The PAT is sanitised of non-ASCII characters at save time to prevent invisible Unicode (zero-width spaces, smart quotes) from corrupting HTTP `Authorisation` headers — a real failure mode when copy-pasting tokens from browser UIs.

---

## Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Hosting | GitHub Pages | Zero config, CDN-backed |
| Storage | GitHub Contents API | CORS-native, versioned, free |
| Charts | Chart.js 4 (UMD) | No bundler needed |
| Auth | Fine-grained PAT | Scoped to single repo, `Contents: read+write` only |
| State | Vanilla JS + `localStorage` | No framework overhead |
| Style | CSS custom properties | Per-card accent theming without preprocessors |

Total JS dependencies: **1** (Chart.js). No npm. No build step. No CI pipeline required — deploy by pushing one file.

---

## Setup

### 1. Enable GitHub Pages

Push `index.html` and `data.json` to `main`, then:

**Settings → Pages → Deploy from branch → `main` → `/ (root)`**

### 2. Create a fine-grained PAT

**GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens**

- Repository access: **Only selected repositories** → this repo
- Permissions → Contents: **Read and write**

### 3. Connect on each device

Open the Pages URL → click **⚙** → enter your username, repo, branch (`main`), and PAT → **Save & Connect**.

Credentials live in `localStorage` on that device only. Repeat once per browser.

---

## Data format

`data.json` is human-readable and directly editable via GitHub's web UI:

```json
{
  "hours": 22,
  "hoursTarget": 120,
  "qTarget": 170,
  "vTarget": 170,
  "history": [
    ["2026-01-01T00:00:00.000Z", 22,  "",  ""],
    ["2026-06-01T00:00:00.000Z", "", 162, 155]
  ]
}
```

History rows: `[ISO timestamp, hours|"", quant|"", verbal|""]`. Hours and scores are tracked in independent rows — logging a Quant score does not require a Verbal score in the same entry.

---

## Why this approach

I spent too much time on this as a distraction from my actual GRE prep, so there's no way I'm gonna figure out Firebase, Supabase, or a custom REST API on top of that. This one treats **git as a key-value store**, and the Contents API is a thin HTTP layer over object storage that also provides versioning, a full change log, conflict detection, and free hosting for the client. The tradeoff is write latency (~400 ms per commit), which is fine for a study tracker but would not suit anything requiring real-time collaboration BECAUSE I'M IN THIS ALONE.

The architecture is fully auditable: every data change is a commit, every commit is timestamped and reversible, and the entire application state is a single JSON file you can read and edit by hand.

---

*Built as a personal productivity tool during GRE prep. Targets: Quant 170 · Verbal 170 · 120 study hours. Pray for me yall*
