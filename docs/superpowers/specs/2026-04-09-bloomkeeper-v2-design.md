# Bloomkeeper v2 — Advanced Features Design Spec

**Date:** 2026-04-09
**Status:** Approved
**Scope:** 9 new features added to existing single-file vanilla HTML/CSS/JS habit tracker

---

## Constraints

- Single `index.html` file (existing pattern)
- Vanilla HTML, CSS, JavaScript only — no frameworks, no build step, no npm
- All data in localStorage — no backend
- Backward compatible with existing habit/completion data
- Dark/light theme support for all new UI
- Mobile responsive (existing breakpoint: 860px sidebar, 700px grids)

---

## Feature 1: Flexible Completion Types

### What
Habits can be one of four types: binary (default), quantity, duration, or rating.

### Data Model Changes
```
// Habit object gets new field:
habit.completionType = 'binary' | 'quantity' | 'duration' | 'rating'
habit.target = number | null  // e.g., 8 for "8 glasses", 30 for "30 min"

// Completions change from:
completions[date][habitId] = true
// To:
completions[date][habitId] = true  // backward compat for old binary
// OR
completions[date][habitId] = { value: 4.2 }  // new format
```

### UI
- Create/edit modal: new "Completion Type" dropdown after Frequency, with conditional "Target" input
- Garden view: binary habits keep tap-to-complete. Non-binary habits show a quick-entry popover on tap (number input + checkmark button)
- Plant growth for non-binary: scales with value/target ratio (0-25% = stage 1, 25-50% = stage 2, etc.)
- Detail modal: shows historical values, average, best day

### Helper Changes
- `isCompleted(id, ds)` must handle both `true` and `{ value }` formats
- `getCompletionValue(id, ds)` — new helper returning the numeric value or 1 for binary
- `getCompletionPercent(id, ds)` — value / target, capped at 100%

---

## Feature 2: Habit Stacking

### What
Optional chaining: "After I complete Habit A, do Habit B."

### Data Model
```
habit.afterHabit = habitId | null  // the habit this one follows
```

### UI
- Create/edit modal: new "After completing" dropdown (lists other habits + "None")
- Garden view: stacked habits have a subtle dashed connector line between their plant pots (CSS `::after` pseudo-element on the pot, pointing right to the next one)
- When habit A is completed and habit B follows it, habit B's pot gets a brief glow animation (CSS `@keyframes stackGlow`)
- Circular chains prevented: validation in saveHabit()

### Logic
- `getStack(habitId)` — returns ordered array of chained habits starting from the root
- Garden grid renders stacked habits adjacently

---

## Feature 3: Weekly Review

### What
A reflection modal that surfaces weekly stats and stores notes.

### Data Model
```
// New localStorage key: bk_reviews
reviews[weekKey] = {
  completedAt: ISO date,
  stats: { totalDue, totalDone, bestHabit, worstHabit },
  wentWell: "string",
  gotInWay: "string"
}
// weekKey format: "2026-W15"
```

### UI
- New sidebar nav item: "Reviews" (under Views section, icon: 📝)
- Clicking it opens the Reviews view showing past reviews as cards
- "Write Review" button opens modal with:
  - Auto-computed stats for the past 7 days (read-only)
  - "What went well?" textarea
  - "What got in the way?" textarea
  - Save button
- On Sunday, if no review exists for the current week, a subtle banner appears on Garden view: "Time for your weekly review →"

---

## Feature 4: Streak Grace Period

### What
One free skip per week per habit. Missing one due day wilts the plant but doesn't break the streak. Missing two consecutive due days breaks it.

### Data Model
```
// Track in completions metadata (or compute on the fly)
// Computed approach (no new storage):
// In getStreak(), when encountering a missed due day:
//   - Check if another missed due day follows immediately
//   - If not, count it as a grace skip and continue the streak
//   - Allow max 1 grace skip per 7-day window
```

### UI
- Plant with an active grace skip: leaves have a slight droop (CSS transform: rotate on leaf elements), subtle amber tint on the pot border
- Small badge on plant pot: "1 skip used" in amber text
- Streak display changes from "12 day streak" to "12 day streak (1 skip)"

### Logic Change
- `getStreak(id)` modified: track consecutive misses. If only 1 miss in a 7-day window, skip it and continue counting. If 2+ consecutive misses, break.

---

## Feature 5: Habit Pausing

### What
Pause a habit to exclude it from daily tracking without breaking the streak.

### Data Model
```
habit.paused = false  // default
habit.pausedAt = null  // ISO date when paused
```

### UI
- Detail modal: "Pause" button (or "Resume" if already paused)
- Garden view: paused habits render with 40% opacity, a pause icon (⏸) overlay, and are moved to the end of the grid
- Paused habits excluded from: daily progress ring, stats bar, weekly view due counts
- Sidebar category counts exclude paused habits

---

## Feature 6: Data Export/Import

### What
Export all app data as JSON, import from JSON file, reset all data.

### UI
- New sidebar nav section: "Data" (at bottom, before theme toggle)
- Three buttons stacked vertically:
  - "Export Data" — downloads `bloomkeeper-backup-YYYY-MM-DD.json`
  - "Import Data" — hidden file input, loads JSON, confirms before replacing
  - "Reset All" — confirm dialog, clears all localStorage keys starting with `bk_`

### Export Format
```json
{
  "version": 2,
  "exportedAt": "2026-04-09T...",
  "habits": [...],
  "completions": {...},
  "reviews": {...}
}
```

### Import Logic
- Validate version field exists
- Confirm with user: "This will replace all current data. Continue?"
- Write to localStorage, reload app

---

## Feature 7: Shareable Progress Cards

### What
Generate a PNG image of your weekly/overall progress, downloadable for sharing.

### UI
- Dashboard view: "Share Progress" button in page header
- Clicking it renders a Canvas-based card (600x400px) with:
  - "Bloomkeeper" header with plant emoji
  - Row of plant emojis from top habits with their streak counts
  - Overall stats: total habits, best streak, avg completion %
  - Week summary: "Week of Apr 7 — 85% completion"
  - Dark theme styling (dark bg, green accents)
- Downloads as `bloomkeeper-progress.png` via `canvas.toBlob()` + download link

### Implementation
- Create offscreen canvas, draw with Canvas 2D API
- No external libraries
- Text rendered with `ctx.fillText()`, rounded rects with `ctx.roundRect()`

---

## Feature 8: Correlation Insights

### What
Surface cross-habit patterns from completion data.

### Analysis (all client-side)
For each pair of habits (A, B):
1. Count days both were due
2. Count days A was done AND B was done
3. Count days A was NOT done AND B was done
4. Compute: P(B done | A done) vs P(B done | A not done)
5. If difference > 20 percentage points and sample size > 14 days, surface as insight

Also compute:
- Best day of week (highest avg completion across all habits)
- Most productive time of day (based on habits' timeOfDay tags and their completion rates)
- Current longest active streak across all habits

### UI
- Dashboard view: new "Insights" card (full-width, below existing cards)
- Shows top 3 correlation insights as sentences: "On days you meditate, you're 82% more likely to journal"
- Shows best day, best time, longest active streak
- Refresh button to recompute

---

## Feature 9: Long-Term Habit Score

### What
A cumulative score per habit that rewards consistency over time and survives broken streaks.

### Formula
```
score = (totalCompletions * 10) + (longestStreak * 5) + (consistencyRate * 2)
```
Where consistencyRate is the all-time completion % (0-100).

### UI
- Habit detail modal: large score display with label "Habit Score"
- Garden view: small score badge below the category tag on each plant pot (e.g., "847 pts")
- Dashboard: "Habit Scores" leaderboard card showing all habits ranked by score

---

## New Sidebar Navigation (final structure)

```
Views:
  🌿 Garden
  📅 Weekly
  📊 Monthly
  🗓 Year
  📈 Dashboard
  📝 Reviews        ← NEW

Categories:
  (dynamic)

Data:                ← NEW
  Export Data
  Import Data
  Reset All

[Theme Toggle]
```

---

## Migration / Backward Compatibility

- On load, check completion values. If `completions[ds][id] === true`, treat as binary done. If object with `value`, use new format.
- Old habits without `completionType` default to `'binary'`
- Old habits without `paused`, `afterHabit` fields default to `false`/`null`
- No migration script needed — defaults handle everything

---

## Build Sequence

Features are independent. Recommended order (each builds on stable ground):

1. Flexible completion types (touches data model first)
2. Habit pausing (simple, unblocks others)
3. Streak grace period (modifies getStreak)
4. Habit stacking (new UI + garden rendering)
5. Data export/import (safety net before bigger changes)
6. Weekly review (new view + modal)
7. Long-term habit score (pure computation + UI)
8. Correlation insights (pure computation + UI)
9. Shareable progress cards (Canvas rendering, last because it's the most self-contained)
