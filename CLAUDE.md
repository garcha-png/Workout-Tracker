# LiftLog — Workout Tracker

## What this is
Single-file vanilla HTML/CSS/JS app (`index.html`). No build tools, no frameworks, no dependencies except Lucide icons via CDN. Runs directly in a browser or as a PWA.

## File structure
Everything lives in one file: `/Users/barjindersingh/Workout Tracker/index.html`
- Top: `<style>` block with all CSS
- Middle: HTML structure (onboarding overlay + app shell)
- Bottom: `<script>` block with all JS

---

## Architecture

### Layout
```
#onboarding   — full-screen overlay, shown on first launch (z-index 200)
#app
  #content    — flex:1, overflow hidden, position relative
    .tab#tab-home      — Home + active workout session
    .tab#tab-edit      — Exercise editor
    .tab#tab-stats     — Progress charts
    .tab#tab-settings  — Name, units, reset
  #navbar     — 4 tabs: Home | Edit | Stats | Settings
```

Tabs are `position:absolute;inset:0`. Only one has `.active` (display:flex) at a time. `switchTab(name)` handles routing.

### Key rule: Home tab doubles as the workout screen
When `activeWorkout !== null`, `renderHome()` immediately calls `renderWorkout()` which renders into `#tab-home`. There is no separate workout tab.

---

## State (all persisted to localStorage)

| Key | Type | Description |
|-----|------|-------------|
| `settings` | `{ name, unit: 'lbs'\|'kg' }` | User preferences |
| `muscle_groups` | `{ [groupName]: { exercises, hits, weekStart } }` | Exercise definitions + weekly hit counts |
| `workouts` | Array of completed workout objects | Full history |
| `active_workout` | Workout snapshot or null | In-progress session, auto-saved every 5s |
| `last_finished` | `{ workout, savedWorkoutId, hitsBefore }` | Enables "Did you finish?" undo flow |
| `onboarded` | `'true'` | Whether onboarding has been completed |

### Exercise definition shape
```js
{ name: string,  // always lowercase
  sets: number,
  alsoCountsFor: string[] }  // display-only, not used in hit counting
```

### Active workout shape
```js
{
  id, startTime,
  groups: [{
    name,
    exercises: [{
      name, alsoCountsFor,
      sets: [{ reps, weight, completed }]
    }]
  }]
}
```

---

## Muscle groups & categories

```js
const GROUP_CATEGORIES = [
  { label: 'Push',       groups: ['Chest', 'Shoulders', 'Triceps'] },
  { label: 'Pull',       groups: ['Lats', 'Upper Back', 'Lower Back', 'Biceps'] },
  { label: 'Legs',       groups: ['Quads', 'Hamstrings', 'Calves'] },
  { label: 'Supporters', groups: ['Core', 'Forearms'] }
];
const CAT_COLORS = { Push: '#d94545', Pull: '#4a80c4', Legs: '#9460c8', Supporters: '#4a9e4a' };
```

Weekly hit cap: 2 hits per group. Resets every Sunday via `checkWeekReset()`.

---

## Theme (CSS custom properties)

```css
--bg: #141414          /* page background */
--surface: #1e1e1e     /* cards */
--elevated: #272727    /* inputs, inner cards */
--border: #333
--text: #e0e0e0
--text2: #6a6a6a       /* secondary / muted */
--accent: #d45018      /* orange — buttons, active states, timers */
--accent-faint: rgba(212,80,24,.1)
--danger: #d94545      /* red — delete, discard, finish */
```

Category colors used on card left borders and section headers: `--c-push`, `--c-pull`, `--c-legs`, `--c-sup`.

---

## Key functions

### Routing
- `switchTab(name)` — shows tab, highlights nav button, calls render function

### Home / session flow
- `renderHome()` — if session active → `renderWorkout()`; else renders group selector
- `toggleGroup(el, g, color, selBg)` — select/deselect a muscle group card
- `startGym()` — builds `activeWorkout` from selected groups, calls `switchTab('home')`
- `continueWorkout()` — resumes crash-recovered session
- `discardWorkout()` — clears `active_workout`, re-renders home
- `finishGym()` — saves workout, increments hits, triggers undo snapshot, returns to home

### Workout rendering
- `renderWorkout()` — renders into `#tab-home`; set rows with reps/weight inputs + check button
- `toggleSetDone(gi, ei, si)` — marks set complete, shows rest picker
- `addGymSet / removeGymSet` — add/remove set from an exercise
- `updSet(gi, ei, si, field, val)` — live-updates reps/weight in activeWorkout

### Rest timer
- `showRestPicker(key)` — renders 6 duration buttons after a set is completed
- `startRestFromPicker(key, sec)` — starts countdown
- `startRestTimer(key, sec)` — core countdown logic, vibrates on finish
- `renderRestInline(key)` — updates the running timer UI (+30s / Skip buttons)
- `clearRestTimer(key)` / `skipRest(key)` / `addRestTime(key, sec)`
- `REST_OPTIONS`: 30s, 45s, 1min, 1m30s, 2min, 2m30s

### Session timer
- `startGymTimer()` — drives the `MM:SS` header, also saves `active_workout` every 5s

### Undo (last finished)
- `resumeLastWorkout()` — restores previous workout + reverts hits
- `dismissUnfinished()` — clears the undo snapshot

### Edit
- `renderEdit()` — full exercise editor with "also hits" compound tags
- `saveEdit()` — normalizes names to lowercase, cleans orphaned history entries

### Stats
- `renderStats() / renderStatsBody() / renderExStats()` — cascading render driven by group + exercise dropdowns
- `buildProgressChart(sessions)` — custom SVG: straight lines, filled circles, weight above + reps below each point, scrollable
- `renderSummary(g, exList)` — all-time averages table

### Settings
- `renderSettings()` — name input, LBS/KG toggle, Reset All Data
- `saveSettingsAuto(showFeedback)` — saves name + unit
- `settingsUnit(u)` — toggles unit in memory (saved on Save button)

### Utilities
- `load(k, d) / save(k, v)` — localStorage JSON helpers
- `uid()` — random ID for workouts
- `esc(s)` — HTML-escape
- `today()` — Toronto timezone date string `YYYY-MM-DD`
- `getSunday()` — ISO date of most recent Sunday
- `getCat(g) / getColor(g)` — group → category → color
- `hexFaint(hex, a)` — hex color → rgba with alpha
- `getPrevSets(group, exName)` — pulls last completed sets for pre-filling

---

## Conventions
- Exercise names always stored and compared **lowercase**
- `alsoCountsFor` is display-only in the workout screen; only directly selected groups get hits when finishing
- Icons: Lucide via CDN (`https://unpkg.com/lucide@latest`), `<i data-lucide="name">` in HTML, `lucide.createIcons()` called after every render
- No emojis anywhere in the UI
- Copy style: plain, short — "Rest up." not "You've already trained today!"
