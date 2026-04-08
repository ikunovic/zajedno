# CLAUDE.md - Zajedno Custody Calendar

## Project Overview

Zajedno is a custody calendar web application for Croatian families. It allows parents to visually plan and track child custody schedules using a drag-to-paint calendar interface. The app supports cloud sync via Firebase, AI-powered schedule suggestions via Google Gemini, and offline-first local storage.

**Language**: All UI text is in Croatian (lang="hr"). Code comments mix Croatian and English.

## Architecture

**This is a single-file application.** The entire app lives in `index.html` (~1000 lines). There is no build system, no package.json, no bundler, and no framework.

### File Structure

```
index.html                          # Complete application (HTML + CSS + JS)
zajedno-netlify-raspored_v1-*.json  # Sample custody schedule data
zajedno-netlify-raspored_v2-*.json  # Sample custody schedule data
```

### Tech Stack

- **Frontend**: Vanilla JavaScript, Tailwind CSS (CDN), Google Fonts (Roboto)
- **Cloud**: Firebase Auth (Google OAuth), Firestore (real-time sync)
- **AI**: Google Gemini 2.5 Flash API (schedule generation/analysis)
- **Storage**: localStorage (offline), Firestore (cloud)
- **Hosting**: Static file hosting (Netlify-compatible)

### Code Organization (inside index.html)

The file is organized in this order:

1. **`<head>` / `<style>`** (lines 1-277): All CSS including responsive breakpoints, calendar grid, compare mode, holidays, ghost days, sidebar, floating menu, AI chat, modals, toasts, print styles
2. **`<body>` HTML** (lines 279-367): Sidebar, top bar, calendar container, mobile floating menu, AI chat window, modals, toast
3. **`<script>` - Core Logic** (lines 369-903): Global variables, core functions, UI rendering, interaction handlers, data management
4. **`<script type="module">` - Firebase** (lines 907-1000): Firebase initialization, auth, Firestore sync, cloud listener

### Key Patterns

- All functions are attached to `window.*` for global access across script blocks
- The Firebase module script overrides `window.saveDataToCloud` when cloud mode is active
- State is managed via global variables on `window` (no state management library)
- Calendar re-renders are triggered by calling `window.initCalendar()` which rebuilds the entire DOM

## Data Model

### Calendar Data Format

Each schedule file stores date-to-custody-type mappings:

```json
{
  "2026-01-21": "father",
  "2026-01-22": "mother",
  "2026-02-03": "neutral"
}
```

- Keys: ISO date strings (`YYYY-MM-DD`)
- Values: `"father"` | `"mother"` | `"neutral"`
- Unassigned dates have no entry (not stored)

### Files Array Structure

```javascript
window.files = [
  {
    id: 1234567890.123,    // Date.now() + Math.random()
    name: "Moj Raspored",  // User-visible name
    data: { /* date-custody mappings */ },
    isDirty: false          // Has unsaved changes
  }
]
```

- Maximum 10 files per user
- `window.activeFileId` tracks the currently displayed file
- `window.compareFileIds` tracks files selected for split-view comparison

### Firestore Document Structure

```
users/{uid} -> { files: [...], lastUpdated: ISO string }
```

## Key Global Variables

| Variable | Purpose |
|----------|---------|
| `window.files` | Array of all schedule files |
| `window.activeFileId` | Currently displayed file ID |
| `window.compareFileIds` | Files being compared (split view) |
| `window.currentTool` | Active paint tool: `father`, `mother`, `neutral`, `eraser` |
| `window.isDragging` | Whether user is currently drag-painting |
| `window.isMobilePaintMode` | Mobile paint vs scroll toggle |
| `window.undoStack` | Array of previous states (max 20) |
| `window.currentUser` | Firebase auth user object |
| `window.db` | Firestore database instance |
| `window._isSaving` | Guard flag to prevent snapshot echo |
| `window._pendingCloudData` | Deferred cloud update during drag |
| `window._isFirstSnapshot` | First-login merge detection flag |

## Key Functions

### Core
- `initCalendar()` - Rebuilds entire calendar DOM
- `generateYearBlock(year)` - Creates year section with 12 months
- `generateMonth(year, month, varHolidays)` - Creates month grid with day cells
- `paintDay(element, isoDate)` - Applies current tool color to a day
- `updateAllStats()` - Recalculates all statistics displays
- `calculateStats(data, yearFilter, monthFilter)` - Counts father/mother/neutral days

### File Management
- `addFileToList(name, data)` - Creates new schedule file
- `setActiveFile(id, skipScroll)` - Switches to a file, re-renders calendar
- `trySwitchToFile(id)` - Switch with unsaved-changes guard
- `deleteFile(id, event)` - Removes file with confirmation
- `duplicateCurrentFile()` - Copies current file
- `handleFileImport(input)` - Imports JSON file
- `saveCurrentFileToDisk()` - Exports current file as JSON download

### Data Persistence
- `saveDataToCloud()` - Saves to localStorage (overridden by Firebase module for cloud)
- `loadDataFromLocal()` - Loads from localStorage on startup
- `scheduleSave(delay)` - Debounced save (500ms default)

### UI
- `selectTool(tool)` - Sets active paint tool and updates button styles
- `renderSidebar()` - Rebuilds sidebar file list
- `toggleMobilePaintMode()` - Toggles mobile paint/scroll mode
- `toggleFloatingMenu()` - Mobile floating action menu
- `toggleAIChat()` - AI chat window visibility
- `showToast(msg, type)` - Toast notification (`success` | `error`)
- `openModal(id, msg, action)` / `closeModal(id)` - Modal dialog management

### History
- `pushToHistory()` - Saves current state to undo stack
- `performUndo()` - Restores last state from undo stack

## Conventions

### CSS Classes
- `.bg-father` (green #4ade80), `.bg-mother` (red #f87171), `.bg-neutral` (blue #60a5fa)
- `.holiday-major` (bold yellow border), `.holiday-minor` (thin yellow border)
- `.ghost-day` - Days from adjacent months (faded)
- `.compare-mode` - Split-view day cells
- `.no-print` - Hidden when printing

### Responsive Breakpoints
- Mobile: `max-width: 767px` - Body scrolls, sticky headers offset from top bar, floating action menu visible
- Desktop: `min-width: 768px` - Body hidden overflow, `#main-scroll-container` scrolls, sidebar always visible

### Holidays
- Fixed holidays: Defined in `window.fixedHolidays` (Croatian national holidays, keyed by `MM-DD`)
- Variable holidays: Computed from Easter date (`getEaster()`) - Uskrs, Uskrsni ponedjeljak, Tijelovo
- June 20 marked as "Ljetni raspored (Pocatak)" - custody-specific major date

### Calendar Range
- `window.startYear = 2026`, `window.endYear = 2028`
- Week starts on Monday (P, U, S, C, P, S, N)

## Cloud Sync Behavior

1. On startup, app loads from localStorage (`loadDataFromLocal`)
2. Firebase module initializes and sets up auth listener
3. On login, `startCloudListener()` begins real-time Firestore sync
4. `_isFirstSnapshot` flag handles first-login merge (local dirty data uploaded to cloud)
5. `_isSaving` flag prevents processing own write echoes
6. `_pendingCloudData` defers cloud updates during active drag operations
7. Undo stack is preserved across cloud sync updates

## Known Issues / Missing Features

- `window.toggleSidebar()` is called from HTML buttons but not implemented in JavaScript
- API keys (Firebase, Gemini) are hardcoded in the HTML
- No automated tests
- No CI/CD pipeline
- No build process or minification

## Development

### Running Locally

Open `index.html` directly in a browser, or serve with any static file server:

```bash
python3 -m http.server 8000
# or
npx serve .
```

The app works in local-only mode without Firebase configuration.

### Making Changes

Since everything is in a single HTML file:
- CSS changes go in the `<style>` block (lines 8-277)
- HTML structure changes go in the `<body>` (lines 279-367)
- Core JS logic goes in the first `<script>` block (lines 369-903)
- Firebase/cloud logic goes in the `<script type="module">` block (lines 907-1000)

### Testing

No automated tests exist. Test manually by:
1. Opening in browser
2. Painting days with each tool (father/mother/neutral/eraser)
3. Testing drag painting on desktop and mobile
4. Testing file operations (create, rename, duplicate, delete, import, export)
5. Testing undo (Ctrl+Z)
6. Testing compare mode (checkbox in sidebar)
7. Verifying cloud sync if Firebase is configured
