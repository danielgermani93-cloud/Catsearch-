# CLAUDE.md — CatSearch Codebase Guide

## Overview

CatSearch is a Progressive Web App (PWA) for product catalog management with voice search capabilities. It is a **zero-build, CDN-based** single-page application — no npm, no bundler, no compile step. All code lives in three files.

**Language**: Italian-localized UI (`it-IT`), created by Daniel Germani.

---

## Repository Structure

```
Catsearch-/
├── index.html      # Entire React application (673 lines)
├── manifest.json   # PWA manifest
└── sw.js           # Service Worker (offline support)
```

No `package.json`, no `node_modules`, no build output. All dependencies load from CDNs at runtime.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| UI Framework | React 18.2.0 (UMD, via cdnjs CDN) |
| JSX Transpilation | Babel Standalone 7.23.5 (in-browser) |
| Styling | Tailwind CSS v3 (CDN) |
| Database | Firebase Firestore 9.22.0 (compat SDK) |
| File Storage | Firebase Cloud Storage |
| PWA | Service Worker + Web App Manifest |
| Voice | Web Speech API (`SpeechRecognition`) |

---

## Running the App

**No build step required.** Two options:

```bash
# Option 1: Python HTTP server (required for Service Worker to work)
cd /home/user/Catsearch-
python3 -m http.server 8000
# Open http://localhost:8000

# Option 2: Open the file directly (Service Worker won't register over file://)
open index.html
```

There are no tests, linters, or CI pipelines.

---

## Application Architecture

### `index.html`

The entire React app is defined in a single `<script type="text/babel">` tag. Structure:

```
Head (lines 1–51)
  ├── Meta/PWA tags
  ├── CDN imports (React, Babel, Tailwind, Firebase)
  ├── Firebase initialization
  └── Service Worker registration

Body (lines 52–673)
  └── <div id="root">
       └── App component (lines 55–671)
            ├── State declarations
            ├── Firebase effect (loadItems)
            ├── CRUD functions
            ├── Voice search functions
            └── JSX UI (3 tabs: Search, Catalog, Add)
```

### Component: `App`

Single root component. No child components or separate files.

**State variables:**

| Variable | Purpose |
|----------|---------|
| `activeTab` | Current view: `'search'`, `'catalog'`, `'add'` |
| `items` | Product list from Firestore |
| `search` | Search input text |
| `desc`, `code`, `img` | Add-form inputs |
| `editId`, `editD`, `editC`, `editI` | Edit mode state |
| `listening` | Voice recognition active flag |
| `uploading` | Async operation in progress |
| `showInstall` | PWA install prompt visibility |

**Key functions:**

| Function | Description |
|----------|------------|
| `loadItems()` | Real-time Firestore `onSnapshot` listener; auto-deduplicates by code |
| `addItem()` | Add single product with optional image upload |
| `addBulk()` | Batch import; parses lines matching `"Description CODE/NUMBER"` format |
| `delItem(id)` | Delete product document |
| `startEdit(item)` / `saveEdit()` | Edit workflow |
| `startVoiceSearch()` / `stopVoiceSearch()` | Web Speech API control |
| `handleInstallClick()` | PWA `beforeinstallprompt` handler |

---

## Firebase Configuration

Firebase is initialized with hardcoded credentials in `index.html` (lines 22–29):

```javascript
firebase.initializeApp({
    apiKey: "AIzaSyDNfGTMYx6sB6DS0M9P7pYdAu_ZWvtLELU",
    authDomain: "catsearch-585e0.firebaseapp.com",
    projectId: "catsearch-585e0",
    storageBucket: "catsearch-585e0.firebasestorage.app",
    messagingSenderId: "126444973350",
    appId: "1:126444973350:web:862e494e41158f375ae5e7"
});
```

Accessed globally via `window.db` (Firestore) and `window.storage` (Cloud Storage).

---

## Firestore Data Model

**Collection**: `items`

```javascript
{
  id: string,           // Auto-generated Firestore document ID
  description: string,  // Product name/description
  code: string,         // Product code (format: "NUMBER/NUMBER", e.g. "123/456")
  imageUrl: string|null,// Cloud Storage URL or null
  createdAt: string     // ISO 8601 timestamp
}
```

**Cloud Storage path**: `items/{timestamp}_{filename}`

---

## Bulk Import Format

The "Add" tab supports bulk import via textarea. Each line must match:

```
Product Description CODE/NUMBER
```

Regex used: `/\b\d+\/\d+\b/` — extracts the first `digits/digits` pattern per line.

Example input:
```
Red Widget 100/200
Blue Gadget 101/300
```

---

## Service Worker (`sw.js`)

- **Cache name**: `catsearch-v1`
- **Strategy**: Network-first, falls back to cache for GET requests
- **Excluded from cache**: Firebase API domains (`firebasestorage.googleapis.com`, `firestore.googleapis.com`)
- **Cached on install**: `/`, `/index.html`, `/manifest.json`
- **Offline fallback**: Returns HTTP 503 if both network and cache miss

To update the cache, increment the version string in `sw.js`:
```javascript
const CACHE_NAME = 'catsearch-v2'; // bump this
```

---

## PWA Manifest (`manifest.json`)

| Property | Value |
|----------|-------|
| Name | CatSearch |
| Theme color | `#10b981` (Emerald green) |
| Background | `#000000` (Black) |
| Display | `standalone` |
| Orientation | `portrait-primary` |
| Icons | SVG, 192×192 and 512×512 |

---

## Code Conventions

- **No TypeScript** — plain JavaScript with JSX
- **No component splitting** — entire app in one `App` function
- **Inline event handlers** — arrow functions directly in JSX
- **Tailwind utility classes** — no custom CSS, dark theme with emerald accents
- **camelCase** for variables and functions
- **Emoji iconography** — used in place of icon libraries
- **Error handling** — try/catch with `alert()` for user-facing errors
- **No state management library** — `useState` only

---

## Development Workflow

### Making changes

1. Edit `index.html` directly — no recompile needed
2. Refresh browser to see changes (Babel transpiles JSX on page load)
3. For Service Worker changes, bump the cache version in `sw.js`

### Adding a new feature

All UI and logic changes go into `index.html`. Typical workflow:
1. Add state with `useState` near the top of `App`
2. Add handler functions before the `return` statement
3. Add JSX in the appropriate tab section

### Deployment

The app is static. Deploy by uploading all three files to any static host:
- GitHub Pages
- Firebase Hosting
- Vercel / Netlify
- Any web server

---

## Key Constraints for AI Assistants

1. **No build tools** — do not suggest npm packages, webpack, or bundler changes
2. **No TypeScript migration** — keep plain JavaScript unless explicitly requested
3. **No component splitting** — do not refactor into separate files unless asked; the single-file approach is intentional for this project's simplicity
4. **Firebase SDK version** — uses the compat SDK (v9 compat, not modular); keep `firebase.*` global namespace calls consistent with existing patterns
5. **No new dependencies** — if a library is needed, add it as a CDN `<script>` tag in `<head>`, not via npm
6. **Tailwind only** — do not introduce inline `style={}` props or CSS files for layout; use Tailwind classes
7. **Mobile-first** — the app is designed for portrait mobile use; preserve responsive behavior
8. **Italian locale** — voice search uses `it-IT`; keep UI text consistent with Italian if modifying visible strings

---

## Git Info

- **Main branch**: `main` (or `master`)
- **Development branch**: `claude/add-claude-documentation-QvzRB`
- **Remote**: `danielgermani93-cloud/Catsearch-`

Commit messages follow plain English imperative style (e.g., `Add bulk import validation`).
