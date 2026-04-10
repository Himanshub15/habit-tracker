# Bloomkeeper v2 — Advanced Features Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add 9 advanced features to Bloomkeeper (flexible completions, habit stacking, weekly reviews, streak grace, habit pausing, data export/import, shareable cards, correlation insights, habit scores) — all within a single vanilla `index.html`.

**Architecture:** Single-file app. All state in localStorage. Each feature adds CSS at the end of the `<style>` block, HTML modals/views in the `<body>`, and JS functions in the `<script>`. Features are independent — each task produces a working commit. Backward compatible with existing data.

**Tech Stack:** Vanilla HTML, CSS, JavaScript. Canvas 2D API for shareable cards. No libraries, no build step.

**File:** All changes to `/tmp/habit-tracker/index.html`

---

### Task 1: Flexible Completion Types

**Files:**
- Modify: `/tmp/habit-tracker/index.html` (CSS, create modal HTML, JS helpers, garden rendering)

**Summary:** Add quantity/duration/rating completion types. Binary habits keep tap-to-complete. Non-binary show a quick-entry popover.

- [ ] **Step 1: Add CSS for the quick-entry popover and completion type badges**

Add before the closing `</style>` tag:

```css
/* ===== QUICK ENTRY POPOVER ===== */
.quick-entry {
  position: absolute; bottom: 100%; left: 50%; transform: translateX(-50%);
  background: var(--bg-modal); border: 1px solid var(--border);
  border-radius: 12px; padding: 12px; z-index: 50;
  box-shadow: 0 8px 32px rgba(0,0,0,0.3); min-width: 160px;
  animation: modalSlide 0.2s ease;
}
.quick-entry-row { display: flex; gap: 6px; align-items: center; }
.quick-entry input {
  width: 80px; padding: 6px 10px; border-radius: 7px;
  border: 1px solid var(--border); background: var(--bg-input);
  color: var(--text-primary); font-size: 14px; font-family: var(--font-body);
  text-align: center; outline: none;
}
.quick-entry input:focus { border-color: var(--accent-glow); }
.quick-entry-label { font-size: 11px; color: var(--text-muted); margin-bottom: 6px; }
.quick-entry-confirm {
  padding: 6px 12px; border-radius: 7px; border: none; cursor: pointer;
  background: var(--accent-glow); color: #0a0e0a; font-weight: 700; font-size: 13px;
}

.completion-type-badge {
  font-size: 9px; color: var(--text-muted); margin-top: 2px;
  font-weight: 600; text-transform: uppercase; letter-spacing: 0.5px;
}

/* Rating stars */
.rating-input { display: flex; gap: 4px; }
.rating-star {
  font-size: 20px; cursor: pointer; opacity: 0.3; transition: opacity 0.1s;
}
.rating-star.active { opacity: 1; }
.rating-star:hover { opacity: 0.7; }
```

- [ ] **Step 2: Add completion type fields to the create/edit modal HTML**

Find the form group for "Frequency" (the `<select>` with id `habitFrequency`). Insert this new form row BEFORE it (after the Category/Time of Day row):

```html
<div class="form-row">
  <div class="form-group">
    <label class="form-label">Completion Type</label>
    <select class="form-input" id="habitCompletionType" onchange="toggleTargetInput()">
      <option value="binary">Done / Not Done</option>
      <option value="quantity">Quantity</option>
      <option value="duration">Duration (min)</option>
      <option value="rating">Rating (1-5)</option>
    </select>
  </div>
  <div class="form-group" id="targetGroup" style="display:none">
    <label class="form-label" id="targetLabel">Target</label>
    <input type="number" class="form-input" id="habitTarget" placeholder="e.g., 8" min="1">
  </div>
</div>
```

- [ ] **Step 3: Update JS helpers for flexible completions**

Add these new helper functions after the existing `getCompletionRate` function:

```javascript
function getCompletionValue(id, ds) {
  const c = completions[ds] && completions[ds][id];
  if (!c) return 0;
  if (c === true) return 1;
  if (typeof c === 'object' && c.value !== undefined) return c.value;
  return 1;
}

function getCompletionPercent(id, ds) {
  const habit = habits.find(h => h.id === id);
  if (!habit || !habit.target) return isCompleted(id, ds) ? 100 : 0;
  return Math.min(100, Math.round((getCompletionValue(id, ds) / habit.target) * 100));
}

function toggleTargetInput() {
  const type = document.getElementById('habitCompletionType').value;
  const group = document.getElementById('targetGroup');
  const label = document.getElementById('targetLabel');
  if (type === 'binary') { group.style.display = 'none'; return; }
  group.style.display = 'block';
  if (type === 'quantity') label.textContent = 'Target Amount';
  else if (type === 'duration') label.textContent = 'Target (minutes)';
  else if (type === 'rating') { group.style.display = 'none'; } // rating is always 1-5
}
```

- [ ] **Step 4: Update `isCompleted` to handle both formats**

Replace the existing `isCompleted` function:

```javascript
function isCompleted(id, ds) {
  const c = completions[ds] && completions[ds][id];
  if (!c) return false;
  if (c === true) return true;
  if (typeof c === 'object' && c.value !== undefined) return c.value > 0;
  return false;
}
```

- [ ] **Step 5: Update `saveHabit` to include completionType and target**

In the `saveHabit()` function, after `const duration = ...` line, add:

```javascript
const completionType = document.getElementById('habitCompletionType').value;
const target = completionType !== 'binary' && completionType !== 'rating'
  ? (parseInt(document.getElementById('habitTarget').value) || null)
  : (completionType === 'rating' ? 5 : null);
```

In the `Object.assign` call for editing, add `completionType, target` to the object.
In the `habits.push` call for creating, add `completionType, target` to the object.

- [ ] **Step 6: Update `toggleHabitToday` for non-binary habits**

Replace `toggleHabitToday`:

```javascript
let activeQuickEntry = null;

function toggleHabitToday(id) {
  const habit = habits.find(h => h.id === id);
  if (!habit) return;
  const type = habit.completionType || 'binary';

  if (type === 'binary') {
    toggleCompletion(id, today());
    renderGarden();
    return;
  }

  // Close existing popover
  if (activeQuickEntry) { activeQuickEntry.remove(); activeQuickEntry = null; }

  // Show quick-entry popover
  const pot = event.currentTarget;
  const popover = document.createElement('div');
  popover.className = 'quick-entry';
  popover.onclick = (e) => e.stopPropagation();

  if (type === 'rating') {
    const current = getCompletionValue(id, today());
    popover.innerHTML = `
      <div class="quick-entry-label">Rate today</div>
      <div class="rating-input">
        ${[1,2,3,4,5].map(n => `<span class="rating-star ${n <= current ? 'active' : ''}"
          onclick="submitQuickEntry('${id}',${n})">${n <= current ? '★' : '☆'}</span>`).join('')}
      </div>`;
  } else {
    const unit = type === 'duration' ? 'min' : (habit.target ? `/ ${habit.target}` : '');
    const current = getCompletionValue(id, today());
    popover.innerHTML = `
      <div class="quick-entry-label">${type === 'duration' ? 'Minutes' : 'Amount'} ${unit}</div>
      <div class="quick-entry-row">
        <input type="number" id="quickEntryInput" value="${current || ''}" min="0" placeholder="0"
          onkeydown="if(event.key==='Enter')submitQuickEntry('${id}')">
        <button class="quick-entry-confirm" onclick="submitQuickEntry('${id}')">✓</button>
      </div>`;
  }

  pot.style.position = 'relative';
  pot.appendChild(popover);
  activeQuickEntry = popover;
  const inp = popover.querySelector('input');
  if (inp) setTimeout(() => inp.focus(), 50);
}

function submitQuickEntry(id, ratingValue) {
  const ds = today();
  if (ratingValue !== undefined) {
    if (!completions[ds]) completions[ds] = {};
    completions[ds][id] = { value: ratingValue };
  } else {
    const inp = document.getElementById('quickEntryInput');
    const val = parseFloat(inp?.value) || 0;
    if (!completions[ds]) completions[ds] = {};
    if (val > 0) completions[ds][id] = { value: val };
    else delete completions[ds][id];
  }
  save();
  if (activeQuickEntry) { activeQuickEntry.remove(); activeQuickEntry = null; }
  renderGarden();
}

// Close popover on outside click
document.addEventListener('click', () => {
  if (activeQuickEntry) { activeQuickEntry.remove(); activeQuickEntry = null; }
});
```

- [ ] **Step 7: Update `openCreateModal` and `editHabit` to handle new fields**

In `openCreateModal()`, add after the `habitDuration` reset:
```javascript
document.getElementById('habitCompletionType').value = 'binary';
document.getElementById('habitTarget').value = '';
toggleTargetInput();
```

In `editHabit(id)`, add after setting `habitDuration`:
```javascript
document.getElementById('habitCompletionType').value = h.completionType || 'binary';
document.getElementById('habitTarget').value = h.target || '';
toggleTargetInput();
```

- [ ] **Step 8: Update plant rendering to show completion type info**

In the `renderGarden()` function, in the loop where plant pots are generated, update the streak display line. Replace:
```javascript
const time = habit.timeOfDay && habit.timeOfDay !== 'anytime' ? habit.timeOfDay : '';
```
With:
```javascript
const time = habit.timeOfDay && habit.timeOfDay !== 'anytime' ? habit.timeOfDay : '';
const cType = habit.completionType || 'binary';
const cValue = getCompletionValue(habit.id, ds);
const cTarget = habit.target;
let completionDisplay = '';
if (cType === 'quantity' && cValue > 0) completionDisplay = `<div class="completion-type-badge">${cValue}${cTarget ? ' / ' + cTarget : ''}</div>`;
else if (cType === 'duration' && cValue > 0) completionDisplay = `<div class="completion-type-badge">${cValue} min${cTarget ? ' / ' + cTarget : ''}</div>`;
else if (cType === 'rating' && cValue > 0) completionDisplay = `<div class="completion-type-badge">${'★'.repeat(cValue)}${'☆'.repeat(5 - cValue)}</div>`;
```

Then add `${completionDisplay}` after the plant-streak div in the HTML template.

- [ ] **Step 9: Update `getGrowthStage` for non-binary habits**

Replace `getGrowthStage`:
```javascript
function getGrowthStage(streak, habit, ds) {
  const type = habit?.completionType || 'binary';
  if (type !== 'binary' && ds) {
    const pct = getCompletionPercent(habit.id, ds);
    if (pct === 0) return streak === 0 ? 0 : getGrowthStageByStreak(streak);
    if (pct <= 25) return 1;
    if (pct <= 50) return 2;
    if (pct <= 75) return 3;
    if (pct < 100) return 4;
    return 5;
  }
  return getGrowthStageByStreak(streak);
}

function getGrowthStageByStreak(streak) {
  if (streak === 0) return 0;
  if (streak <= 2) return 1;
  if (streak <= 6) return 2;
  if (streak <= 13) return 3;
  if (streak <= 29) return 4;
  return 5;
}
```

Update the `renderPlant` call in `renderGarden` to pass the extra args:
```javascript
const { html: ph, isWilted } = renderPlant(habit, streak, ds);
```
And update `renderPlant` signature and its internal `getGrowthStage` call:
```javascript
function renderPlant(habit, streak, ds) {
  const stage = getGrowthStage(streak, habit, ds);
  // ... rest unchanged
}
```

- [ ] **Step 10: Verify and commit**

Open `index.html` in browser. Create a new habit with type "Quantity" (target: 8). Tap it — should show number popover. Enter 5, confirm. Plant should show "5 / 8" badge. Create a "Rating" habit. Tap — should show stars. Old binary habits should still tap-to-toggle.

```bash
cd /tmp/habit-tracker
git add index.html
git commit -m "feat: add flexible completion types (quantity, duration, rating)"
```

---

### Task 2: Habit Pausing

**Files:**
- Modify: `/tmp/habit-tracker/index.html`

**Summary:** Add pause/resume to habit detail modal. Paused habits render greyed out and are excluded from progress.

- [ ] **Step 1: Add CSS for paused state**

```css
/* ===== PAUSED STATE ===== */
.plant-pot.paused { opacity: 0.4; filter: grayscale(0.5); }
.plant-pot.paused::after { content: '⏸'; position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); font-size: 28px; z-index: 10; }
.plant-pot.paused .plant-visual { opacity: 0.5; }
.pause-badge { font-size: 9px; color: var(--accent-warm); font-weight: 700; text-transform: uppercase; letter-spacing: 0.5px; margin-top: 3px; }
```

- [ ] **Step 2: Update garden rendering to handle paused habits**

In `renderGarden()`, update the `dueHabits` filter:
```javascript
const dueHabits = habits.filter(h => !h.paused && isDueOn(h, new Date()));
```

In the garden grid loop, add paused class and skip click handler for paused habits:
```javascript
const isPaused = habit.paused;
html += `<div class="plant-pot ${done ? 'completed' : ''} ${isWilted ? 'wilted' : ''} ${isPaused ? 'paused' : ''}"
  onclick="${isPaused ? `showDetail('${habit.id}')` : `toggleHabitToday('${habit.id}')`}"
  ondblclick="showDetail('${habit.id}')">
  <div class="plant-visual">${ph}</div>
  <div class="plant-name">${habit.name}</div>
  ${isPaused ? '<div class="pause-badge">Paused</div>' :
    `<div class="plant-streak">${streak > 0 ? `<strong>${streak}</strong> day streak` : due ? 'Tap to water' : 'Rest day'}</div>`}
  ${completionDisplay}
  ${time ? `<div class="plant-time">${time}</div>` : ''}
  <div class="plant-category" style="color:${CAT_COLORS[habit.category] || CAT_COLORS.custom}">${habit.category}</div>
</div>`;
```

- [ ] **Step 3: Add pause/resume button to detail modal**

In `showDetail()`, update the detail-actions div to include a pause button:
```javascript
const isPaused = h.paused;
// Replace the detail-actions section:
<div class="detail-actions">
  <button class="btn" onclick="editHabit('${id}')">Edit</button>
  <button class="btn" onclick="togglePause('${id}')" style="color:${isPaused ? 'var(--accent-glow)' : 'var(--accent-warm)'}">
    ${isPaused ? '▶ Resume' : '⏸ Pause'}
  </button>
  <button class="btn btn-danger" onclick="deleteHabit('${id}')">Delete</button>
  <button class="btn" onclick="closeModal('detailModal')" style="margin-left:auto">Close</button>
</div>
```

- [ ] **Step 4: Add `togglePause` function**

```javascript
function togglePause(id) {
  const h = habits.find(x => x.id === id);
  if (!h) return;
  h.paused = !h.paused;
  h.pausedAt = h.paused ? today() : null;
  save();
  closeModal('detailModal');
  renderAll();
}
```

- [ ] **Step 5: Exclude paused habits from weekly view and stats**

In `renderWeekly()`, filter: `const activeHabits = habits.filter(h => !h.paused);` and use `activeHabits` instead of `habits` in the loop.

In `renderSidebarCategories()`, filter: `const activeHabits = habits.filter(h => !h.paused);` and use it.

- [ ] **Step 6: Verify and commit**

Open browser. Double-click a habit → detail modal. Click "Pause". Habit should grey out with ⏸ icon. Progress ring should exclude it. Click again → "Resume". Should return to normal.

```bash
git add index.html
git commit -m "feat: add habit pausing with visual state and progress exclusion"
```

---

### Task 3: Streak Grace Period

**Files:**
- Modify: `/tmp/habit-tracker/index.html`

**Summary:** Modify `getStreak` to allow 1 missed day per 7-day window without breaking the streak. Visual feedback on plants using grace.

- [ ] **Step 1: Add CSS for grace state**

```css
/* ===== GRACE PERIOD ===== */
.plant-pot.grace .leaf-left, .plant-pot.grace .leaf-right { transform: rotate(15deg); }
.plant-pot.grace { border-color: var(--accent-warm); }
.grace-badge {
  font-size: 9px; color: var(--accent-warm); font-weight: 600;
  display: inline-block; margin-top: 2px;
}
```

- [ ] **Step 2: Rewrite `getStreak` with grace period logic**

Replace the existing `getStreak` function:

```javascript
function getStreak(id) {
  let streak = 0;
  let d = new Date();
  let graceUsed = false;
  let daysSinceGrace = 0;
  const habit = habits.find(h => h.id === id);
  if (!habit) return { count: 0, graceActive: false };

  // If not completed today, start checking from yesterday
  if (!isCompleted(id, dateStr(d))) {
    // Check if today is a due day — if so, it could be the grace day
    if (isDueOn(habit, d)) {
      if (!graceUsed) {
        graceUsed = true;
        daysSinceGrace = 0;
      } else {
        return { count: 0, graceActive: false };
      }
    }
    d.setDate(d.getDate() - 1);
  }

  for (let i = 0; i < 1000; i++) {
    const ds = dateStr(d);
    if (isCompleted(id, ds)) {
      streak++;
      if (graceUsed) daysSinceGrace++;
      d.setDate(d.getDate() - 1);
    } else if (!isDueOn(habit, d)) {
      d.setDate(d.getDate() - 1);
    } else {
      // Missed a due day
      if (!graceUsed && daysSinceGrace < 7) {
        graceUsed = true;
        daysSinceGrace = 0;
        d.setDate(d.getDate() - 1);
        continue;
      }
      break;
    }
    if (daysSinceGrace >= 7) {
      graceUsed = false; // reset grace window
      daysSinceGrace = 0;
    }
  }
  return { count: streak, graceActive: graceUsed };
}
```

- [ ] **Step 3: Update all callers of `getStreak` to use `.count`**

Search for every call to `getStreak(` in the file. Each one now returns `{ count, graceActive }`. Update:

- `renderGarden()`: `const streakData = getStreak(habit.id); const streak = streakData.count; const graceActive = streakData.graceActive;`
- `renderDashboard()`: `const s = getStreak(h.id).count;` and `getStreak(b.id).count`
- `showDetail()`: `const s = getStreak(id).count;`
- `getLongestStreak()` — this can stay as-is since it uses its own loop
- Stats bar: `const allStreaks = habits.map(h => getStreak(h.id).count);`

- [ ] **Step 4: Update garden rendering to show grace state**

In the garden grid loop, add the grace class and badge:
```javascript
const graceClass = graceActive ? 'grace' : '';
// Add to plant-pot classes
// Add after streak display:
const graceBadge = graceActive ? '<div class="grace-badge">1 skip used</div>' : '';
```

Include `${graceClass}` in the plant-pot class list and `${graceBadge}` after the streak div.

- [ ] **Step 5: Verify and commit**

Open browser. Find a habit with a streak. In the console, manually delete yesterday's completion: `delete completions['2026-04-08']['h_med']; localStorage.setItem('bk_completions', JSON.stringify(completions));` then refresh. The streak should survive with "1 skip used" badge and amber border. Delete another day's completion — streak should break.

```bash
git add index.html
git commit -m "feat: add streak grace period (never miss twice)"
```

---

### Task 4: Habit Stacking

**Files:**
- Modify: `/tmp/habit-tracker/index.html`

**Summary:** Optional chaining — "After I complete Habit A, do Habit B." Visual connectors in garden, glow animation on trigger.

- [ ] **Step 1: Add CSS for stacking visuals**

```css
/* ===== HABIT STACKING ===== */
.stack-connector {
  position: absolute; right: -14px; top: 50%;
  width: 14px; height: 2px; border-top: 2px dashed var(--accent-glow);
  z-index: 10; opacity: 0.5;
}
@keyframes stackGlow {
  0% { box-shadow: 0 0 0 0 rgba(74, 223, 106, 0.4); }
  70% { box-shadow: 0 0 0 12px rgba(74, 223, 106, 0); }
  100% { box-shadow: 0 0 0 0 rgba(74, 223, 106, 0); }
}
.plant-pot.stack-triggered { animation: stackGlow 1s ease; }
.stack-label { font-size: 9px; color: var(--accent-sky); margin-top: 2px; }
```

- [ ] **Step 2: Add "After completing" dropdown to create/edit modal**

Insert after the Frequency form group:

```html
<div class="form-group">
  <label class="form-label">After Completing (Stack)</label>
  <select class="form-input" id="habitAfterHabit">
    <option value="">None (standalone)</option>
  </select>
</div>
```

- [ ] **Step 3: Populate the afterHabit dropdown dynamically**

Add function and call it in `openCreateModal` and `editHabit`:

```javascript
function populateAfterHabitDropdown(excludeId) {
  const sel = document.getElementById('habitAfterHabit');
  sel.innerHTML = '<option value="">None (standalone)</option>';
  for (const h of habits) {
    if (h.id === excludeId) continue;
    sel.innerHTML += `<option value="${h.id}">${h.emoji} ${h.name}</option>`;
  }
}
```

In `openCreateModal()`, add: `populateAfterHabitDropdown(null); document.getElementById('habitAfterHabit').value = '';`

In `editHabit(id)`, add: `populateAfterHabitDropdown(id); document.getElementById('habitAfterHabit').value = h.afterHabit || '';`

- [ ] **Step 4: Save afterHabit field**

In `saveHabit()`, add after `customDays`:
```javascript
const afterHabit = document.getElementById('habitAfterHabit').value || null;
```
Add `afterHabit` to both the `Object.assign` and `habits.push` calls.

Add circular chain validation before saving:
```javascript
// Validate no circular chains
if (afterHabit) {
  let check = afterHabit;
  const visited = new Set([editingHabitId || 'new']);
  while (check) {
    if (visited.has(check)) { alert('Circular habit chain detected!'); return; }
    visited.add(check);
    const parent = habits.find(h => h.id === check);
    check = parent?.afterHabit;
  }
}
```

- [ ] **Step 5: Sort garden grid to show stacked habits adjacently**

In `renderGarden()`, before the grid loop, sort habits so stacked ones are adjacent:

```javascript
function getOrderedHabits() {
  const ordered = [];
  const placed = new Set();
  // Find chain roots (habits that no other habit points to as afterHabit)
  const targets = new Set(habits.filter(h => h.afterHabit).map(h => h.afterHabit));

  function addChain(id) {
    if (placed.has(id)) return;
    placed.add(id);
    const h = habits.find(x => x.id === id);
    if (h) ordered.push(h);
    // Find habits that come after this one
    const followers = habits.filter(x => x.afterHabit === id);
    for (const f of followers) addChain(f.id);
  }

  // Start with roots of chains, then standalone
  for (const h of habits) {
    if (!h.afterHabit) addChain(h.id);
  }
  // Any remaining (shouldn't happen but safety)
  for (const h of habits) {
    if (!placed.has(h.id)) { placed.add(h.id); ordered.push(h); }
  }
  return ordered;
}
```

Use `getOrderedHabits()` instead of `habits` in the garden grid rendering loop.

- [ ] **Step 6: Add connector lines and stack labels**

In the garden grid loop, after generating each plant pot, check if the next habit in the ordered list has `afterHabit` pointing to the current one. If so, add a connector div inside the pot:

```javascript
const nextInStack = orderedHabits[idx + 1];
const hasFollower = nextInStack && nextInStack.afterHabit === habit.id;
const isFollower = habit.afterHabit;
const connectorHtml = hasFollower ? '<div class="stack-connector"></div>' : '';
const stackLabel = isFollower ? `<div class="stack-label">after ${habits.find(h => h.id === habit.afterHabit)?.emoji || ''}</div>` : '';
```

Include `${connectorHtml}` inside the plant-pot div (at the end, before closing), and `${stackLabel}` after plant-name.

- [ ] **Step 7: Add glow trigger on completion**

In `toggleHabitToday` (for binary) and `submitQuickEntry`, after completing a habit, check if any habit has `afterHabit` pointing to this one:

```javascript
function triggerStackGlow(completedId) {
  const follower = habits.find(h => h.afterHabit === completedId);
  if (!follower) return;
  setTimeout(() => {
    const pots = document.querySelectorAll('.plant-pot');
    // Find the pot for the follower by checking rendered order
    pots.forEach(pot => {
      if (pot.querySelector('.plant-name')?.textContent === follower.name) {
        pot.classList.add('stack-triggered');
        setTimeout(() => pot.classList.remove('stack-triggered'), 1000);
      }
    });
  }, 100);
}
```

Call `triggerStackGlow(id)` after toggling completion in both `toggleHabitToday` and `submitQuickEntry`.

- [ ] **Step 8: Verify and commit**

Create two habits: "Meditate" and "Journal". Edit "Journal" and set "After completing" to "Meditate". Garden should show them adjacent with a dashed connector. Complete "Meditate" — "Journal" pot should glow briefly.

```bash
git add index.html
git commit -m "feat: add habit stacking with visual connectors and glow triggers"
```

---

### Task 5: Data Export/Import

**Files:**
- Modify: `/tmp/habit-tracker/index.html`

**Summary:** Add export/import/reset buttons in sidebar. JSON file download and upload.

- [ ] **Step 1: Add CSS**

```css
/* ===== DATA MANAGEMENT ===== */
.data-section { margin-top: 10px; }
.data-btn {
  display: flex; align-items: center; gap: 9px;
  padding: 7px 10px; border-radius: 7px; cursor: pointer;
  font-size: 12px; font-weight: 500; color: var(--text-secondary);
  border: none; background: none; width: 100%; text-align: left;
  font-family: var(--font-body); transition: all 0.15s ease;
}
.data-btn:hover { background: var(--bg-secondary); color: var(--text-primary); }
.data-btn.danger:hover { color: var(--accent-coral); }
```

- [ ] **Step 2: Add HTML to sidebar**

Insert before the `sidebar-footer` div:

```html
<div class="nav-section data-section">
  <div class="nav-label">Data</div>
  <button class="data-btn" onclick="exportData()"><span style="width:22px;text-align:center">📥</span> Export Data</button>
  <button class="data-btn" onclick="document.getElementById('importFile').click()"><span style="width:22px;text-align:center">📤</span> Import Data</button>
  <input type="file" id="importFile" accept=".json" style="display:none" onchange="importData(event)">
  <button class="data-btn danger" onclick="resetAllData()"><span style="width:22px;text-align:center">🗑</span> Reset All</button>
</div>
```

- [ ] **Step 3: Add JS functions**

```javascript
function exportData() {
  const data = {
    version: 2,
    exportedAt: new Date().toISOString(),
    habits: habits,
    completions: completions,
    reviews: JSON.parse(localStorage.getItem('bk_reviews') || '{}')
  };
  const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `bloomkeeper-backup-${today()}.json`;
  a.click();
  URL.revokeObjectURL(url);
}

function importData(event) {
  const file = event.target.files[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = (e) => {
    try {
      const data = JSON.parse(e.target.result);
      if (!data.version || !data.habits) { alert('Invalid backup file.'); return; }
      if (!confirm('This will replace all current data. Continue?')) return;
      habits = data.habits;
      completions = data.completions || {};
      save();
      if (data.reviews) localStorage.setItem('bk_reviews', JSON.stringify(data.reviews));
      renderAll();
      alert('Data imported successfully!');
    } catch (err) { alert('Error reading file: ' + err.message); }
  };
  reader.readAsText(file);
  event.target.value = '';
}

function resetAllData() {
  if (!confirm('This will permanently delete all habits, completions, and reviews. Are you sure?')) return;
  if (!confirm('Really? This cannot be undone.')) return;
  localStorage.removeItem('bk_habits');
  localStorage.removeItem('bk_completions');
  localStorage.removeItem('bk_reviews');
  localStorage.removeItem('bk_seeded');
  habits = []; completions = {};
  renderAll();
}
```

- [ ] **Step 4: Verify and commit**

Click Export — should download a JSON file. Open it, verify it has habits and completions. Click Import with that file — should restore. Click Reset — should clear everything after two confirms.

```bash
git add index.html
git commit -m "feat: add data export/import and reset functionality"
```

---

### Task 6: Weekly Review

**Files:**
- Modify: `/tmp/habit-tracker/index.html`

**Summary:** Add Reviews view in sidebar. Weekly reflection modal with stats and notes.

- [ ] **Step 1: Add CSS**

```css
/* ===== REVIEWS ===== */
.review-card {
  background: var(--bg-card); border: 1px solid var(--border);
  border-radius: 14px; padding: 22px; margin-bottom: 14px;
}
.review-card h3 {
  font-family: var(--font-display); font-size: 16px; margin-bottom: 4px;
}
.review-card .review-date { font-size: 12px; color: var(--text-muted); margin-bottom: 14px; }
.review-stats-row {
  display: flex; gap: 14px; margin-bottom: 14px; flex-wrap: wrap;
}
.review-stat-mini {
  background: var(--bg-secondary); border-radius: 8px; padding: 8px 14px;
  text-align: center; flex: 1; min-width: 80px;
}
.review-stat-mini .val { font-family: var(--font-display); font-size: 18px; font-weight: 700; }
.review-stat-mini .lbl { font-size: 9px; color: var(--text-muted); text-transform: uppercase; letter-spacing: 0.5px; }
.review-note { margin-top: 10px; }
.review-note h4 { font-size: 12px; color: var(--text-muted); text-transform: uppercase; letter-spacing: 0.5px; margin-bottom: 4px; }
.review-note p { font-size: 13px; color: var(--text-secondary); line-height: 1.6; font-style: italic; }
.review-banner {
  background: var(--bg-card); border: 1px solid var(--accent-warm);
  border-radius: 10px; padding: 12px 18px; margin-bottom: 18px;
  display: flex; align-items: center; justify-content: space-between;
  cursor: pointer;
}
.review-banner:hover { background: var(--bg-card-hover); }
.review-banner span { font-size: 13px; color: var(--accent-warm); font-weight: 500; }
```

- [ ] **Step 2: Add Reviews view HTML**

Insert after the Dashboard view div (before closing `</main>`):

```html
<!-- Reviews View -->
<div class="view" id="reviewsView">
  <div class="page-header">
    <div>
      <h1 class="page-title">Weekly Reviews</h1>
      <p class="page-subtitle">Reflect on your growth</p>
    </div>
    <button class="btn btn-primary" onclick="openReviewModal()">+ Write Review</button>
  </div>
  <div id="reviewsList"></div>
</div>
```

- [ ] **Step 3: Add Review modal HTML**

Insert after the detail modal:

```html
<!-- Review Modal -->
<div class="modal-overlay" id="reviewModal">
  <div class="modal">
    <div class="modal-header">
      <h2>Weekly Review</h2>
      <button class="modal-close" onclick="closeModal('reviewModal')">&times;</button>
    </div>
    <div class="modal-body">
      <div id="reviewStats"></div>
      <div class="form-group">
        <label class="form-label">What went well?</label>
        <textarea class="form-input" id="reviewWentWell" rows="3" placeholder="Wins, progress, good days..."></textarea>
      </div>
      <div class="form-group">
        <label class="form-label">What got in the way?</label>
        <textarea class="form-input" id="reviewGotInWay" rows="3" placeholder="Obstacles, missed days, challenges..."></textarea>
      </div>
      <div class="form-actions">
        <button class="btn" onclick="closeModal('reviewModal')">Cancel</button>
        <button class="btn btn-primary" onclick="saveReview()">Save Review</button>
      </div>
    </div>
  </div>
</div>
```

- [ ] **Step 4: Add sidebar nav item for Reviews**

In the sidebar, after the Dashboard nav-item, add:

```html
<div class="nav-item" data-view="reviews" onclick="switchView('reviews')">
  <span class="nav-icon">📝</span> Reviews
</div>
```

- [ ] **Step 5: Add JS for reviews**

```javascript
function getWeekKey(d) {
  const dt = new Date(d || new Date());
  const oneJan = new Date(dt.getFullYear(), 0, 1);
  const weekNum = Math.ceil(((dt - oneJan) / 86400000 + oneJan.getDay() + 1) / 7);
  return `${dt.getFullYear()}-W${String(weekNum).padStart(2, '0')}`;
}

function getReviews() {
  try { return JSON.parse(localStorage.getItem('bk_reviews') || '{}'); } catch { return {}; }
}

function saveReviews(reviews) {
  localStorage.setItem('bk_reviews', JSON.stringify(reviews));
}

function getWeekStats() {
  const now = new Date();
  let totalDue = 0, totalDone = 0;
  const habitStats = [];
  for (const h of habits) {
    if (h.paused) continue;
    let due = 0, done = 0;
    for (let i = 0; i < 7; i++) {
      const d = new Date(now); d.setDate(d.getDate() - i);
      if (isDueOn(h, d)) { due++; if (isCompleted(h.id, dateStr(d))) done++; }
    }
    totalDue += due; totalDone += done;
    habitStats.push({ name: h.name, emoji: h.emoji, due, done, rate: due > 0 ? Math.round((done / due) * 100) : 0 });
  }
  habitStats.sort((a, b) => b.rate - a.rate);
  return {
    totalDue, totalDone,
    pct: totalDue > 0 ? Math.round((totalDone / totalDue) * 100) : 0,
    best: habitStats[0] || null,
    worst: habitStats[habitStats.length - 1] || null
  };
}

function openReviewModal() {
  const stats = getWeekStats();
  document.getElementById('reviewStats').innerHTML = `
    <div class="review-stats-row">
      <div class="review-stat-mini"><div class="val">${stats.pct}%</div><div class="lbl">Completion</div></div>
      <div class="review-stat-mini"><div class="val">${stats.totalDone}/${stats.totalDue}</div><div class="lbl">Check-ins</div></div>
      <div class="review-stat-mini"><div class="val">${stats.best ? stats.best.emoji : '-'}</div><div class="lbl">Best</div></div>
      <div class="review-stat-mini"><div class="val">${stats.worst ? stats.worst.emoji : '-'}</div><div class="lbl">Needs Work</div></div>
    </div>`;
  const reviews = getReviews();
  const weekKey = getWeekKey();
  const existing = reviews[weekKey];
  document.getElementById('reviewWentWell').value = existing?.wentWell || '';
  document.getElementById('reviewGotInWay').value = existing?.gotInWay || '';
  document.getElementById('reviewModal').classList.add('show');
}

function saveReview() {
  const reviews = getReviews();
  const weekKey = getWeekKey();
  const stats = getWeekStats();
  reviews[weekKey] = {
    completedAt: new Date().toISOString(),
    stats: { totalDue: stats.totalDue, totalDone: stats.totalDone, pct: stats.pct,
             bestHabit: stats.best?.name, worstHabit: stats.worst?.name },
    wentWell: document.getElementById('reviewWentWell').value.trim(),
    gotInWay: document.getElementById('reviewGotInWay').value.trim()
  };
  saveReviews(reviews);
  closeModal('reviewModal');
  renderReviews();
}

function renderReviews() {
  const reviews = getReviews();
  const keys = Object.keys(reviews).sort().reverse();
  if (keys.length === 0) {
    document.getElementById('reviewsList').innerHTML = `
      <div class="empty-state"><div class="empty-icon">📝</div>
      <h3>No reviews yet</h3><p>Write your first weekly review to start tracking your reflections.</p>
      <button class="btn btn-primary" onclick="openReviewModal()">+ Write Review</button></div>`;
    return;
  }
  let html = '';
  for (const key of keys) {
    const r = reviews[key];
    html += `<div class="review-card">
      <h3>${key}</h3>
      <div class="review-date">${new Date(r.completedAt).toLocaleDateString('en-US', { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' })}</div>
      <div class="review-stats-row">
        <div class="review-stat-mini"><div class="val">${r.stats.pct}%</div><div class="lbl">Completion</div></div>
        <div class="review-stat-mini"><div class="val">${r.stats.totalDone}/${r.stats.totalDue}</div><div class="lbl">Check-ins</div></div>
      </div>
      ${r.wentWell ? `<div class="review-note"><h4>What went well</h4><p>${r.wentWell}</p></div>` : ''}
      ${r.gotInWay ? `<div class="review-note"><h4>What got in the way</h4><p>${r.gotInWay}</p></div>` : ''}
    </div>`;
  }
  document.getElementById('reviewsList').innerHTML = html;
}
```

- [ ] **Step 6: Add Reviews to switchView and renderAll**

In `switchView()`, add: `else if (view === 'reviews') renderReviews();`

In `renderAll()`, add `renderReviews();` to the call list.

- [ ] **Step 7: Add Sunday review banner to garden view**

In `renderGarden()`, after the daily progress section, add:

```javascript
const reviews = getReviews();
const currentWeek = getWeekKey();
const isSunday = new Date().getDay() === 0;
if ((isSunday || new Date().getDay() === 6) && !reviews[currentWeek]) {
  document.getElementById('dailyProgress').insertAdjacentHTML('afterend',
    `<div class="review-banner" onclick="switchView('reviews');setTimeout(openReviewModal,100)">
      <span>📝 Time for your weekly review</span><span>→</span></div>`);
}
```

- [ ] **Step 8: Verify and commit**

Click "Reviews" in sidebar. Click "Write Review". Fill in stats, text, save. Should appear as a card. Navigate to Garden on a Saturday/Sunday — banner should appear.

```bash
git add index.html
git commit -m "feat: add weekly review system with stats, notes, and garden banner"
```

---

### Task 7: Long-Term Habit Score

**Files:**
- Modify: `/tmp/habit-tracker/index.html`

**Summary:** Cumulative score per habit. Shown in detail modal, garden pots, and dashboard leaderboard.

- [ ] **Step 1: Add CSS**

```css
/* ===== HABIT SCORE ===== */
.habit-score {
  font-family: var(--font-display); font-size: 10px;
  color: var(--accent-warm); font-weight: 600; margin-top: 2px;
}
.score-big {
  font-family: var(--font-display); font-size: 32px; font-weight: 700;
  color: var(--accent-warm); text-align: center; margin: 10px 0 4px;
}
.score-label { text-align: center; font-size: 10px; color: var(--text-muted); text-transform: uppercase; letter-spacing: 1px; }
```

- [ ] **Step 2: Add `getHabitScore` function**

```javascript
function getHabitScore(id) {
  const total = getTotalCompletions(id);
  const longest = getLongestStreak(id);
  const rate = getCompletionRate(id);
  return (total * 10) + (longest * 5) + (rate * 2);
}
```

- [ ] **Step 3: Show score in garden plant pots**

In the garden grid loop, add after the category div:
```javascript
const score = getHabitScore(habit.id);
// Add to the plant pot HTML:
`<div class="habit-score">${score} pts</div>`
```

- [ ] **Step 4: Show score in detail modal**

In `showDetail()`, add a score display. Insert after the second detail-stats grid:
```javascript
const score = getHabitScore(id);
// Add HTML:
`<div class="score-big">${score}</div><div class="score-label">Habit Score</div>`
```

- [ ] **Step 5: Add score leaderboard to Dashboard**

In `renderDashboard()`, add a new card after the Categories card:

```javascript
html += '<div class="dash-card"><h3>🏆 Habit Scores</h3><div class="streak-list">';
const byScore = [...habits].sort((a, b) => getHabitScore(b.id) - getHabitScore(a.id));
for (const h of byScore) {
  const sc = getHabitScore(h.id);
  const maxScore = Math.max(...habits.map(x => getHabitScore(x.id)));
  const pct = maxScore > 0 ? Math.round((sc / maxScore) * 100) : 0;
  html += `<div class="streak-item"><span class="streak-emoji">${h.emoji}</span>
    <div class="streak-info"><div class="name">${h.name}</div><div class="days">${sc} points</div></div>
    <div class="streak-bar"><div class="streak-bar-fill" style="width:${pct}%;background:var(--accent-warm)"></div></div></div>`;
}
html += '</div></div>';
```

- [ ] **Step 6: Verify and commit**

Check garden — each pot should show a score. Open detail modal — big score number. Dashboard — leaderboard card. Scores should differ based on completions/streaks.

```bash
git add index.html
git commit -m "feat: add long-term habit scores with leaderboard"
```

---

### Task 8: Correlation Insights

**Files:**
- Modify: `/tmp/habit-tracker/index.html`

**Summary:** Cross-habit pattern analysis in Dashboard. Shows top correlations, best day, best time.

- [ ] **Step 1: Add CSS**

```css
/* ===== INSIGHTS ===== */
.insight-item {
  padding: 12px 14px; background: var(--bg-secondary);
  border-radius: 9px; margin-bottom: 8px;
  font-size: 13px; line-height: 1.5;
}
.insight-item strong { color: var(--accent-glow); }
.insight-item .insight-label {
  font-size: 9px; color: var(--text-muted); text-transform: uppercase;
  letter-spacing: 1px; margin-bottom: 4px; display: block;
}
```

- [ ] **Step 2: Add correlation computation function**

```javascript
function computeInsights() {
  const insights = [];
  const activeHabits = habits.filter(h => !h.paused);

  // Cross-habit correlations
  for (let i = 0; i < activeHabits.length; i++) {
    for (let j = i + 1; j < activeHabits.length; j++) {
      const a = activeHabits[i], b = activeHabits[j];
      let bothDue = 0, aDoneAndBDone = 0, aNotDoneAndBDone = 0;
      const dates = Object.keys(completions);
      for (const ds of dates) {
        const d = new Date(ds);
        if (!isDueOn(a, d) || !isDueOn(b, d)) continue;
        bothDue++;
        const aDone = isCompleted(a.id, ds);
        const bDone = isCompleted(b.id, ds);
        if (aDone && bDone) aDoneAndBDone++;
        if (!aDone && bDone) aNotDoneAndBDone++;
      }
      if (bothDue < 14) continue;
      const aDoneCount = dates.filter(ds => { const d = new Date(ds); return isDueOn(a, d) && isDueOn(b, d) && isCompleted(a.id, ds); }).length;
      const aNotDoneCount = bothDue - aDoneCount;
      const pBGivenA = aDoneCount > 0 ? Math.round((aDoneAndBDone / aDoneCount) * 100) : 0;
      const pBGivenNotA = aNotDoneCount > 0 ? Math.round((aNotDoneAndBDone / aNotDoneCount) * 100) : 0;
      const diff = pBGivenA - pBGivenNotA;
      if (diff > 20) {
        insights.push({ type: 'correlation', text: `On days you do <strong>${a.emoji} ${a.name}</strong>, you're <strong>${pBGivenA}%</strong> more likely to complete <strong>${b.emoji} ${b.name}</strong>`, score: diff });
      }
    }
  }

  // Best day of week
  const dayTotals = [0,0,0,0,0,0,0], dayCounts = [0,0,0,0,0,0,0];
  const dayNames = ['Sunday','Monday','Tuesday','Wednesday','Thursday','Friday','Saturday'];
  for (const ds in completions) {
    const d = new Date(ds);
    const dow = d.getDay();
    for (const h of activeHabits) {
      if (isDueOn(h, d)) { dayCounts[dow]++; if (isCompleted(h.id, ds)) dayTotals[dow]++; }
    }
  }
  let bestDay = 0, bestDayRate = 0;
  for (let i = 0; i < 7; i++) {
    const rate = dayCounts[i] > 0 ? Math.round((dayTotals[i] / dayCounts[i]) * 100) : 0;
    if (rate > bestDayRate) { bestDay = i; bestDayRate = rate; }
  }
  if (bestDayRate > 0) {
    insights.push({ type: 'bestday', text: `Your most productive day is <strong>${dayNames[bestDay]}</strong> with <strong>${bestDayRate}%</strong> completion rate`, score: 0 });
  }

  // Best time of day
  const timeTotals = {}, timeCounts = {};
  for (const h of activeHabits) {
    const t = h.timeOfDay || 'anytime';
    if (!timeTotals[t]) { timeTotals[t] = 0; timeCounts[t] = 0; }
    timeTotals[t] += getCompletionRate(h.id);
    timeCounts[t]++;
  }
  let bestTime = '', bestTimeRate = 0;
  for (const t in timeTotals) {
    if (t === 'anytime') continue;
    const avg = Math.round(timeTotals[t] / timeCounts[t]);
    if (avg > bestTimeRate) { bestTime = t; bestTimeRate = avg; }
  }
  if (bestTime && bestTimeRate > 0) {
    insights.push({ type: 'besttime', text: `Your <strong>${bestTime}</strong> habits have the highest consistency at <strong>${bestTimeRate}%</strong>`, score: 0 });
  }

  // Sort correlations by score, then append day/time insights
  insights.sort((a, b) => b.score - a.score);
  return insights.slice(0, 5);
}
```

- [ ] **Step 3: Add insights card to Dashboard rendering**

In `renderDashboard()`, after the habit scores card, add:

```javascript
const insights = computeInsights();
if (insights.length > 0) {
  html += '<div class="dash-card full-width"><h3>💡 Insights</h3>';
  for (const ins of insights) {
    html += `<div class="insight-item">${ins.text}</div>`;
  }
  html += '</div>';
}
```

- [ ] **Step 4: Verify and commit**

Open Dashboard. With the demo data, the Insights card should show at least the best day and potentially some correlations (depends on random seed). Insights should have bold highlighting.

```bash
git add index.html
git commit -m "feat: add correlation insights and analytics to dashboard"
```

---

### Task 9: Shareable Progress Cards

**Files:**
- Modify: `/tmp/habit-tracker/index.html`

**Summary:** Canvas-generated PNG of weekly progress, downloadable from Dashboard.

- [ ] **Step 1: Add "Share Progress" button to Dashboard page header**

In the Dashboard view HTML, update the page-header to include:

```html
<button class="btn" onclick="generateShareCard()">📸 Share Progress</button>
```

Add it inside the page-header div, after the subtitle wrapper.

- [ ] **Step 2: Add the canvas generation function**

```javascript
function generateShareCard() {
  const canvas = document.createElement('canvas');
  canvas.width = 600;
  canvas.height = 420;
  const ctx = canvas.getContext('2d');
  const isDark = document.documentElement.getAttribute('data-theme') === 'dark';

  // Background
  ctx.fillStyle = isDark ? '#111a11' : '#fafcf9';
  ctx.beginPath();
  ctx.roundRect(0, 0, 600, 420, 16);
  ctx.fill();

  // Border
  ctx.strokeStyle = isDark ? '#2a3a2a' : '#d4ddd3';
  ctx.lineWidth = 2;
  ctx.beginPath();
  ctx.roundRect(1, 1, 598, 418, 16);
  ctx.stroke();

  // Header bar
  ctx.fillStyle = isDark ? '#1a2a1a' : '#eef3ed';
  ctx.beginPath();
  ctx.roundRect(0, 0, 600, 70, [16, 16, 0, 0]);
  ctx.fill();

  // Title
  ctx.fillStyle = isDark ? '#4adf6a' : '#2a9a4a';
  ctx.font = 'bold 22px "DM Sans", sans-serif';
  ctx.fillText('🌱 Bloomkeeper', 24, 44);

  // Week label
  ctx.fillStyle = isDark ? '#8aaa8a' : '#5a7a5a';
  ctx.font = '13px "DM Sans", sans-serif';
  const weekLabel = `Week of ${new Date().toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' })}`;
  ctx.fillText(weekLabel, 600 - ctx.measureText(weekLabel).width - 24, 44);

  // Stats
  const activeHabits = habits.filter(h => !h.paused);
  const bestStreak = activeHabits.length ? Math.max(...activeHabits.map(h => getStreak(h.id).count)) : 0;
  const avgRate = activeHabits.length ? Math.round(activeHabits.reduce((s, h) => s + getCompletionRate(h.id), 0) / activeHabits.length) : 0;
  const weekStats = getWeekStats();

  const stats = [
    { label: 'Habits', value: `${activeHabits.length}` },
    { label: 'This Week', value: `${weekStats.pct}%` },
    { label: 'Best Streak', value: `${bestStreak}🔥` },
    { label: 'Avg Rate', value: `${avgRate}%` }
  ];

  const statY = 100;
  stats.forEach((s, i) => {
    const x = 24 + i * 142;
    ctx.fillStyle = isDark ? '#1e2e1e' : '#eef3ed';
    ctx.beginPath();
    ctx.roundRect(x, statY, 126, 64, 10);
    ctx.fill();
    ctx.fillStyle = isDark ? '#e8f0e8' : '#1a2e1a';
    ctx.font = 'bold 22px "DM Sans", sans-serif';
    ctx.fillText(s.value, x + 14, statY + 30);
    ctx.fillStyle = isDark ? '#5a7a5a' : '#8aaa8a';
    ctx.font = '600 9px "DM Sans", sans-serif';
    ctx.fillText(s.label.toUpperCase(), x + 14, statY + 50);
  });

  // Top habits
  ctx.fillStyle = isDark ? '#5a7a5a' : '#8aaa8a';
  ctx.font = '600 10px "DM Sans", sans-serif';
  ctx.fillText('TOP HABITS', 24, 200);

  const sorted = [...activeHabits].sort((a, b) => getStreak(b.id).count - getStreak(a.id).count).slice(0, 5);
  sorted.forEach((h, i) => {
    const y = 218 + i * 36;
    const streak = getStreak(h.id).count;
    const rate = getCompletionRate(h.id);

    ctx.font = '18px "DM Sans", sans-serif';
    ctx.fillText(h.emoji, 24, y + 14);

    ctx.fillStyle = isDark ? '#e8f0e8' : '#1a2e1a';
    ctx.font = '600 13px "DM Sans", sans-serif';
    ctx.fillText(h.name, 52, y + 10);

    ctx.fillStyle = isDark ? '#5a7a5a' : '#8aaa8a';
    ctx.font = '12px "DM Sans", sans-serif';
    ctx.fillText(`${streak} day streak · ${rate}%`, 52, y + 26);

    // Mini bar
    ctx.fillStyle = isDark ? '#1a2a1a' : '#eef3ed';
    ctx.beginPath();
    ctx.roundRect(430, y + 4, 146, 6, 3);
    ctx.fill();
    ctx.fillStyle = h.color;
    ctx.beginPath();
    ctx.roundRect(430, y + 4, Math.max(4, (rate / 100) * 146), 6, 3);
    ctx.fill();

    ctx.fillStyle = isDark ? '#8aaa8a' : '#5a7a5a';
  });

  // Footer
  ctx.fillStyle = isDark ? '#5a7a5a' : '#8aaa8a';
  ctx.font = '11px "DM Sans", sans-serif';
  ctx.fillText('Generated by Bloomkeeper — himanshub15.github.io', 24, 400);

  // Download
  canvas.toBlob(blob => {
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `bloomkeeper-progress-${today()}.png`;
    a.click();
    URL.revokeObjectURL(url);
  });
}
```

- [ ] **Step 3: Verify and commit**

Click "Share Progress" on Dashboard. A PNG should download. Open it — should show dark card with stats, top habits with bars, and Bloomkeeper branding.

```bash
git add index.html
git commit -m "feat: add shareable progress card generation (Canvas PNG)"
```

---

## Final Verification

- [ ] **Step 1: Full app test**

Open index.html fresh. Verify:
1. Demo data loads with all old features working
2. Create a new habit with "Quantity" type — popover works
3. Pause a habit — greys out, excluded from progress
4. Check streak grace — skip one day, streak survives
5. Set up a habit stack — connector lines show
6. Export data — valid JSON downloads
7. Write a weekly review — saves and displays
8. Check Dashboard — scores, insights, and leaderboard all render
9. Generate share card — PNG downloads correctly
10. Light/dark theme works for all new UI
11. Mobile responsive — sidebar, popovers, modals all work

- [ ] **Step 2: Final commit and push**

```bash
cd /tmp/habit-tracker
git log --oneline -10
git push
```
