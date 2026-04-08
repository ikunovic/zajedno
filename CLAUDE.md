# CLAUDE.md — Zajedno Custody Calendar

## What This Is

Single-file web app (`index.html`, ~1100 lines) — a custody/co-parenting calendar for Croatian-speaking users. Parents paint days on a multi-year calendar grid (father/mother/neutral/eraser), export JSON files, and optionally sync via Google (Firebase Firestore).

No build step, no package.json, no framework. Everything is in `index.html`.

---

## Architecture

```
index.html
├── <style>          — Tailwind CDN + custom CSS (lines ~8–320)
├── HTML body        — Sidebar, top bar, calendar container, AI chat, modals (lines ~320–370)
└── <script>         — Core logic (lines ~370–910)
    └── <script type="module"> — Firebase module (lines ~910–1010)
```

### Global State (`window.*`)

| Variable | Type | Purpose |
|---|---|---|
| `window.files` | `Array<FileObj>` | All loaded schedule files |
| `window.activeFileId` | `number` | Currently selected file ID |
| `window.compareFileIds` | `Array<number>` | Files shown in split-view compare |
| `window.currentTool` | `string` | Active paint tool: `'father'`, `'mother'`, `'neutral'`, `'eraser'` |
| `window.isDragging` | `boolean` | True during active mouse/touch drag paint |
| `window.isMobilePaintMode` | `boolean` | Mobile paint mode toggle state |
| `window.undoStack` | `Array` | History of file data snapshots |
| `window.db` | Firestore | Set by Firebase module when logged in |
| `window.currentUser` | Firebase User | Set by Firebase module when logged in |
| `window._isSaving` | `boolean` | Prevents echo from own Firestore writes |
| `window._pendingCloudData` | Object/null | Deferred cloud update while dragging |
| `window._isFirstSnapshot` | `boolean` | Handles local-vs-cloud conflict on first login |

### FileObj schema
```js
{
  id: Number,          // timestamp-based unique ID
  name: String,        // display name
  data: {              // key: "YYYY-MM-DD", value: "father"|"mother"|"neutral"
    "2026-07-15": "father",
    ...
  },
  isDirty: Boolean     // true = unsaved local changes
}
```

---

## Key Functions

| Function | Location | Purpose |
|---|---|---|
| `initCalendar()` | ~line 506 | Rebuilds calendar DOM from window.files |
| `generateYearBlock(year)` | ~line 491 | Renders one year's 12 months |
| `generateMonth(year, month, varHolidays)` | ~line 448 | Renders a single month grid |
| `addInteractionHandlers(el, isoDate)` | ~line 515 | Attaches mouse/touch paint events to a day cell |
| `paintDay(el, isoDate)` | ~line 634 | Applies current tool color to a cell + marks dirty |
| `pushToHistory()` | ~line 648 | Snapshot current file data to undoStack |
| `performUndo()` | ~line 655 | Restores last undoStack snapshot |
| `selectTool(tool)` | ~line 808 | Updates currentTool + button styles + paint indicator |
| `toggleMobilePaintMode()` | ~line 705 | Toggles mobile paint/scroll mode |
| `toggleSidebar()` | ~line 707 | Shows/hides sidebar on mobile |
| `sendAiMessage()` | ~line 725 | Calls Gemini API, handles response/errors |
| `applyAiChanges()` | ~line 783 | Merges pending AI JSON changes into active file |
| `loadDataFromLocal()` | ~line 910 | Loads files from localStorage (with JSON.parse guard) |
| `saveDataToCloud()` | ~line 900 | Default: saves to localStorage; overridden by Firebase module |
| `scheduleSave(delay)` | ~line 897 | Debounced save (500ms default) |
| `renderSidebar()` | ~line 524 | Rebuilds sidebar file list DOM |
| `showToast(msg, type)` | ~line 799 | Shows transient toast notification |
| `_finishDrag()` | ~line 855 | Clears isDragging, clears lastPaint attrs, applies deferred cloud data |
| `updatePaintIndicator()` | ~line 723 | Updates the fixed paint indicator bar text |
| `escapeHtml(str)` | ~line 726 | XSS-safe HTML escaping for user/AI text insertion |

---

## CSS Conventions

- **Tailwind CDN** for utility classes. No build step — use standard Tailwind v3 class names.
- Custom CSS lives in `<style>` block. Key classes:
  - `.day-cell` — calendar day square, `touch-action: manipulation`
  - `.bg-father` / `.bg-mother` / `.bg-neutral` — day color states
  - `.paint-mode-active` — applied to `<body>` when mobile paint mode is on; sets `overflow: hidden; touch-action: none`
  - `.msg.ai.error` — red error state for AI chat messages
  - `#paint-indicator` — fixed bottom bar showing active tool (visible only in paint mode)
  - `.ai-loading::after` — CSS animated `...` dots for loading state
  - `@keyframes pulse-paint` — pulsing shadow on the paint mode button
- Mobile breakpoint: `max-width: 767px`. Desktop: `min-width: 768px`.
- `no-print` class hides elements in print media.

---

## Firebase / Google Sync

Firebase project: `zajedno-4ff5a` (config hardcoded in module script).

**Auth flow:**
1. `window.login()` → `signInWithPopup(auth, GoogleAuthProvider)` → popup OAuth
2. `onAuthStateChanged` → sets `window.currentUser`, overrides `saveDataToCloud()`, starts `startCloudListener()`
3. `startCloudListener()` → `onSnapshot` on `users/{uid}` Firestore document
4. `window.logout()` → signOut + page reload

**Common Google auth errors and fixes:**

| Error code | Cause | Fix |
|---|---|---|
| `auth/unauthorized-domain` | Current hostname not in Firebase authorized domains | Firebase Console → Authentication → Settings → Authorized domains → add the domain |
| `auth/popup-blocked` | Browser blocked the OAuth popup | Allow popups for the site in browser settings |
| `auth/popup-closed-by-user` | User closed the popup | Not an error, just inform user |

The app now displays the specific domain name in the error message so the user knows exactly what to add.

**Firestore document structure:**
```
users/{uid} → { files: FileObj[], lastUpdated: ISO string }
```

**Conflict resolution:**
- If user has unsaved local changes on first login, local state is pushed to cloud (not overwritten)
- `_isSaving` flag prevents the app's own writes from triggering the snapshot listener
- `_pendingCloudData` defers remote updates until drag is complete

---

## Gemini AI Integration

- Model: `gemini-2.5-flash`
- API key: hardcoded as `GEMINI_API_KEY` (line ~386)
- Endpoint: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent`
- System prompt: tells the model to either return pure JSON `{"YYYY-MM-DD": "father|mother|neutral"}` for changes, or plain Croatian text for analysis
- Context sent: up to 15,000 chars of current file's data JSON
- Error handling: checks `response.ok`, extracts `error.message` from Google's error response, displays it in chat (not just "Greška")
- XSS: user input and AI text responses are sanitized via `window.escapeHtml()` before `innerHTML` insertion; the "Apply changes" button HTML is our own trusted markup and is not escaped

---

## Mobile Paint Mode

1. User taps "Scroll" button → `toggleMobilePaintMode()` → sets `isMobilePaintMode = true`
2. `body.paint-mode-active` class applied → `overflow: hidden; touch-action: none` on body + scroll container
3. Mobile mode button gets pulse animation
4. `#paint-indicator` bar appears at bottom showing current tool name
5. `touchstart` on any `.day-cell` → clears all `data-last-paint` attributes (prevents stale tool dedup), sets `isDragging = true`, calls `paintDay()`
6. Global `touchmove` → if `isMobilePaintMode`, uses `elementFromPoint()` to find cell under finger, paints if `data-last-paint !== currentTool`
7. `touchend` / `touchcancel` → `_finishDrag()` → clears `isDragging`, clears all `data-last-paint` attrs

**The `data-last-paint` dedup mechanism:** prevents re-painting the same cell with the same tool during a single drag stroke (avoids redundant DOM updates). It is cleared on every new stroke start AND on drag end, so tool switches work correctly.

---

## Development Workflow

This is a single-file app with no build system.

**To test:**
- Open `index.html` directly in a browser (file:// works for local-only mode)
- For Google auth, serve over HTTPS on an authorized domain (Firebase Console)
- For Gemini AI, the API key is embedded — it works from any origin

**To deploy:**
- Upload `index.html` to any static host (Netlify, Vercel, GitHub Pages, Firebase Hosting)
- After deploying to a new domain, add the domain to Firebase Console → Authentication → Settings → Authorized domains

**Data files (.json):**
- The repo contains sample data exports: `zajedno-netlify-raspored_v1-*.json`, `zajedno-netlify-raspored_v2-*.json`
- These can be imported via the Import button in the app

---

## What Not to Do

- Do NOT add a build step, package.json, or bundler unless explicitly requested
- Do NOT split into multiple files — the single-file architecture is intentional for easy deployment
- Do NOT use `innerHTML` with unsanitized user input or AI response text — always use `window.escapeHtml()`
- Do NOT skip `response.ok` check after fetch calls to external APIs
- Do NOT hardcode new API keys without the user's awareness
- Do NOT add global event listeners without checking if `isMobilePaintMode` / `isDragging` guards are needed

---

## Branch Convention

Development branch: `claude/add-claude-documentation-O2UXN`

Commit messages should be descriptive and reference the layer (Layer 1 debug / Layer 2 robustness / Layer 3 UI).
