# Zoom System Rebuild — 5 Sequential Tasks

Execute these tasks in order. Each task is a complete PR that can be tested independently.

Read ZOOM_SPEC.md before starting any task.

---

## Task 1: State & Helper Functions

**Goal:** Set up all state variables and utility functions needed for the zoom system.

**Work in:** dashboard-v5.html, `<script>` section

**Create:**

1. **Replace state variables:**
   - Remove: `let viewMode = ...`
   - Add:
     ```js
     const CHART_WIDTH = 1000; // px, fixed for prototype
     let zoomLevel  = '1mo';   // current zoom
     let windowEnd  = null;    // Date — last visible day (inclusive)
     let windowStart = null;   // always derived, never set directly
     ```

2. **Add helper functions** (paste before any function that calls them):
   ```js
   function getAvailableZooms(therapyDays) {
     const zooms = ['1mo'];
     if (therapyDays >= 42)  zooms.push('3mo');
     if (therapyDays >= 70)  zooms.push('6mo');
     if (therapyDays >= 112) zooms.push('1yr');
     return zooms;
   }

   function defaultZoom(availableZooms) {
     if (availableZooms.includes('3mo')) return '3mo';
     return '1mo';
   }

   function deriveWindowStart(end, zoom) {
     switch(zoom) {
       case '1mo': return addDays(end, -27);   // 28 days inclusive
       case '3mo': return addDays(end, -83);   // 84 days inclusive
       case '6mo': return addDays(end, -167);  // 168 days inclusive
       case '1yr': return addDays(end, -364);  // 365 days inclusive
     }
   }

   function painStats(sessions) {
     const values = sessions.map(s => s.pain).sort((a, b) => a - b);
     const n = values.length;
     if (n === 0) return null;
     if (n === 1) return { median: values[0], min: values[0], max: values[0], q1: values[0], q3: values[0] };
     const median = n % 2 === 0
       ? (values[n/2 - 1] + values[n/2]) / 2
       : values[Math.floor(n/2)];
     const q1 = values[Math.floor(n * 0.25)];
     const q3 = values[Math.floor(n * 0.75)];
     return { median, min: values[0], max: values[n-1], q1, q3 };
   }
   ```

3. **Remove ResizeObserver** if present (CHART_WIDTH is now fixed)

**Testing:**
- Open browser console, verify no errors on page load
- Type `zoomLevel`, `CHART_WIDTH` in console, confirm they exist and have correct initial values
- Call `getAvailableZooms(100)` in console, confirm it returns `['1mo', '3mo', '6mo']`

**Commit message:**
```
Task 1: Add zoom state variables and helper functions
- Replace viewMode with zoomLevel, windowEnd, windowStart
- Add CHART_WIDTH = 1000px constant (fixed)
- Add getAvailableZooms(), defaultZoom(), deriveWindowStart(), painStats()
- Remove ResizeObserver
```

---

## Task 2: Toolbar UI & Navigation

**Goal:** Build the new toolbar with zoom buttons, navigation arrows, and date range label.

**Work in:** dashboard-v4.html, HTML + CSS + JS

**Replace in HTML:**
Replace the existing toolbar section with:
```html
<div class="toolbar">
  <div class="nav-controls">
    <button id="btnPrev" onclick="stepWindow(-1)">‹</button>
    <span id="dateRangeLabel">—</span>
    <button id="btnNext" onclick="stepWindow(1)">›</button>
  </div>
  <div class="spacer"></div>
  <div class="seg" id="zoomSeg">
    <!-- populated dynamically after data loads -->
  </div>
</div>
```

**Add to CSS:**
```css
.toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 20px;
  border-bottom: 1px solid var(--color-border-line, #ccc);
  background: var(--color-bg, #fff);
}

.nav-controls {
  display: flex;
  align-items: center;
  gap: 12px;
}

.nav-controls button {
  background: none;
  border: 1px solid var(--color-border-line, #ccc);
  padding: 6px 12px;
  cursor: pointer;
  border-radius: 4px;
  font-size: 16px;
}

.nav-controls button:hover {
  background: var(--color-hover, #f0f0f0);
}

.nav-controls button:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

#dateRangeLabel {
  font-size: 14px;
  min-width: 140px;
  text-align: center;
  color: var(--color-text, #333);
}

.spacer {
  flex: 1;
}

.seg {
  display: flex;
  gap: 8px;
}

.seg button {
  padding: 6px 14px;
  border: 1px solid var(--color-border-line, #ccc);
  border-radius: 4px;
  background: var(--color-bg, #fff);
  cursor: pointer;
  font-size: 13px;
  transition: all 200ms;
}

.seg button.active {
  background: var(--color-active, #000);
  color: #fff;
  border-color: var(--color-active, #000);
}

.seg button:hover {
  border-color: var(--color-active, #000);
}
```

**Add to JS (inside or after loadPatient callback):**
```js
function renderZoomButtons(availableZooms) {
  const segEl = document.getElementById('zoomSeg');
  segEl.innerHTML = '';
  
  const zoomLabels = { '1mo': '1 mo', '3mo': '3 mo', '6mo': '6 mo', '1yr': '1 yr' };
  
  availableZooms.forEach(zoom => {
    const btn = document.createElement('button');
    btn.textContent = zoomLabels[zoom];
    btn.onclick = () => setZoom(zoom);
    btn.className = zoom === zoomLevel ? 'active' : '';
    segEl.appendChild(btn);
  });
}

function setActiveZoomButton(zoom) {
  document.querySelectorAll('.seg button').forEach(btn => {
    btn.classList.remove('active');
  });
  const activeBtn = Array.from(document.querySelectorAll('.seg button')).find(
    btn => btn.textContent === (['1mo', '3mo', '6mo', '1yr'].includes(zoom) ? 
      { '1mo': '1 mo', '3mo': '3 mo', '6mo': '6 mo', '1yr': '1 yr' }[zoom] : '')
  );
  if (activeBtn) activeBtn.classList.add('active');
}

function setZoom(newZoom) {
  zoomLevel = newZoom;
  setActiveZoomButton(zoomLevel);
  draw();
}

function stepWindow(direction) {
  // direction: -1 (back), +1 (forward)
  const stepSizes = { '1mo': 7, '3mo': 28, '6mo': 84, '1yr': 168 };
  const step = stepSizes[zoomLevel];
  const lastDate = DAY_DATA[DAY_DATA.length - 1].date;
  
  if (direction > 0) {
    // Step forward
    windowEnd = addDays(windowEnd, step);
    if (windowEnd > lastDate) windowEnd = new Date(lastDate);
  } else {
    // Step backward
    windowEnd = addDays(windowEnd, -step);
    windowStart = deriveWindowStart(windowEnd, zoomLevel);
    if (windowStart < START) {
      const windowDuration = { '1mo': 28, '3mo': 84, '6mo': 168, '1yr': 365 }[zoomLevel];
      windowEnd = addDays(START, windowDuration - 1);
    }
  }
  
  draw();
}

function updateDateRangeLabel(start, end) {
  const label = document.getElementById('dateRangeLabel');
  const fmt = { month: 'short', day: 'numeric' };
  let startStr = start.toLocaleDateString('en-US', fmt);
  let endStr = end.toLocaleDateString('en-US', fmt);
  
  if (start.getFullYear() !== end.getFullYear()) {
    startStr = start.toLocaleDateString('en-US', { ...fmt, year: 'numeric' });
    endStr = end.toLocaleDateString('en-US', { ...fmt, year: 'numeric' });
  }
  
  label.textContent = `${startStr} – ${endStr}`;
}

function updateArrowVisibility(start, end) {
  const lastDate = DAY_DATA[DAY_DATA.length - 1].date;
  document.getElementById('btnPrev').disabled = (start.getTime() === START.getTime());
  document.getElementById('btnNext').disabled = (end.getTime() === lastDate.getTime());
}
```

**Update loadPatient callback:**
After data loads, add:
```js
const firstDate = DAY_DATA.find(d => d.sess)?.date;
const lastDate  = [...DAY_DATA].reverse().find(d => d.sess)?.date;
const therapyDays = Math.round((lastDate - firstDate) / 86400000);

const availableZooms = getAvailableZooms(therapyDays);
renderZoomButtons(availableZooms);

windowEnd = new Date(lastDate);
zoomLevel = defaultZoom(availableZooms);
setActiveZoomButton(zoomLevel);

draw();
```

**Testing:**
- Toolbar appears with correct zoom buttons based on therapy duration
- Clicking a zoom button updates the active state (visual change)
- Arrow buttons show/hide appropriately at dataset edges
- Date range label updates when navigating
- No console errors

**Commit message:**
```
Task 2: Build toolbar UI with zoom buttons and navigation
- Add toolbar HTML with nav arrows, date range label, zoom button container
- Add CSS styling for toolbar, buttons, active states
- Add renderZoomButtons(), setZoom(), stepWindow(), updateDateRangeLabel()
- Update loadPatient() to initialize toolbar after data loads
- Arrow visibility tied to dataset edges
```

---

## Task 3: Slot Building Logic

**Goal:** Create the buildSlots() function that structures data differently per zoom level.

**Work in:** dashboard-v4.html, JS section

**Add function:**
```js
function buildSlots(start, end, zoom) {
  const slots = [];
  
  if (zoom === '1mo' || zoom === '3mo') {
    // One slot per calendar day
    let d = new Date(start);
    while (d <= end) {
      const key = d.toISOString().slice(0, 10);
      const dayEntry = DAY_DATA.find(dd => dd.key === key);
      slots.push({
        date: new Date(d),
        sessions: dayEntry?.sess ? [dayEntry.sess] : [],
        type: 'day'
      });
      d = addDays(d, 1);
    }
  } else if (zoom === '6mo') {
    // One slot per Sun–Sat calendar week
    let d = new Date(start);
    const grouped = {};
    
    while (d <= end) {
      const key = d.toISOString().slice(0, 10);
      const dayEntry = DAY_DATA.find(dd => dd.key === key);
      const weekIdx = sunWeekIndex(d);
      
      if (!grouped[weekIdx]) {
        const sunday = new Date(d);
        sunday.setDate(d.getDate() - d.getDay());
        grouped[weekIdx] = {
          date: sunday,
          sessions: [],
          type: 'week',
          minutes: 0,
          accuracy: {},
          pain: null
        };
      }
      
      if (dayEntry?.sess) {
        grouped[weekIdx].sessions.push(dayEntry.sess);
        grouped[weekIdx].minutes += dayEntry.sess.minutes || 0;
      }
      
      d = addDays(d, 1);
    }
    
    Object.values(grouped).forEach(week => {
      if (week.sessions.length > 0) {
        week.pain = painStats(week.sessions);
        // TODO: aggregate accuracy per gesture in Task 4
      }
      slots.push(week);
    });
  } else if (zoom === '1yr') {
    // One slot per calendar month
    let d = new Date(start);
    d.setDate(1); // start of month
    const grouped = {};
    
    while (d <= end) {
      const monthKey = `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}`;
      
      if (!grouped[monthKey]) {
        grouped[monthKey] = {
          date: new Date(d.getFullYear(), d.getMonth(), 1),
          sessions: [],
          type: 'month',
          minutes: 0,
          accuracy: {},
          pain: null
        };
      }
      
      const key = d.toISOString().slice(0, 10);
      const dayEntry = DAY_DATA.find(dd => dd.key === key);
      
      if (dayEntry?.sess) {
        grouped[monthKey].sessions.push(dayEntry.sess);
        grouped[monthKey].minutes += dayEntry.sess.minutes || 0;
      }
      
      d = addDays(d, 1);
    }
    
    Object.values(grouped).forEach(month => {
      if (month.sessions.length > 0) {
        month.pain = painStats(month.sessions);
        // TODO: aggregate accuracy per gesture in Task 4
      }
      slots.push(month);
    });
  }
  
  return slots;
}
```

**Update draw() function:**
```js
function draw() {
  windowStart = deriveWindowStart(windowEnd, zoomLevel);
  // clamp windowStart to START
  if (windowStart < START) windowStart = new Date(START);

  const slots = buildSlots(windowStart, windowEnd, zoomLevel);

  updateDateRangeLabel(windowStart, windowEnd);
  updateArrowVisibility(windowStart, windowEnd);

  // TODO: pass slots and zoomLevel to drawing functions in Task 4
  // For now, verify slots are built correctly
  console.log(`Slots built: ${slots.length} for zoom ${zoomLevel}`, slots);
}
```

**Testing:**
- Open browser console, navigate through zoom levels
- Verify slot count changes appropriately (28 slots for 1mo, ~12 for 1yr, etc.)
- For 6mo/1yr, verify pain stats are calculated (not null if sessions exist)
- No console errors

**Commit message:**
```
Task 3: Implement buildSlots() for all zoom levels
- buildSlots() returns array of slots (day/week/month) based on zoom
- 1mo/3mo: one slot per calendar day
- 6mo: one slot per Sun–Sat week with aggregated minutes and pain stats
- 1yr: one slot per calendar month with aggregated minutes and pain stats
- Update draw() to build slots; log to console for verification
```

---

## Task 4: Update Drawing Functions

**Goal:** Modify all drawing functions to accept slots and zoomLevel, implement zoom-specific logic.

**Work in:** dashboard-v4.html, drawing functions section

Update function signatures and logic for each:

1. **drawHeader(ctx, slots, zoomLevel, totalW)**
   - 1mo: Week label at Sundays (top row), two-letter day abbr every slot (bottom)
   - 3mo: Month name at month start (top), date number at Sundays only (bottom), thick month separator lines
   - 6mo: Month name at first week of month (top), "Aug 1" format at each Sunday slot (bottom)
   - 1yr: Year label at first slot of new year (top), month abbr "Jan" "Feb" (bottom)

2. **drawColumnBg(ctx, slots, zoomLevel, totalW)**
   - 1mo: Alternate by day parity (day.getDay() % 2)
   - 3mo: Alternate by calendar month, thick separator at month boundaries
   - 6mo: Alternate by calendar month (each week's Sunday's month)
   - 1yr: Alternate by calendar year

3. **drawAccuracy(ctxAcc, slots, zoomLevel, activeGestures, totalW)**
   - 1mo/3mo: Lines connect all session dots (keep existing spanGaps logic)
   - 6mo/1yr: Lines connect all week/month dots; no dots for empty slots but lines span gaps

4. **drawPain(ctxPain, slots, zoomLevel, totalW)**
   - 1mo/3mo: Circle + value (existing logic)
   - 6mo/1yr: Candlestick per the spec (wick, box, median line, median label)

5. **drawMins(ctxMins, slots, zoomLevel, totalW)**
   - Aggregate minutes per slot; bar height proportional to minutes

6. **drawFooter(ctxFooter, slots, zoomLevel, totalW)**
   - Format labels per zoom level

**Reference the ZOOM_SPEC.md for exact rendering details per zoom.**

**Note:** Do NOT modify drawYAxes() or autoY() internals.

**Update draw() to call all functions:**
```js
function draw() {
  windowStart = deriveWindowStart(windowEnd, zoomLevel);
  if (windowStart < START) windowStart = new Date(START);

  const slots = buildSlots(windowStart, windowEnd, zoomLevel);

  updateDateRangeLabel(windowStart, windowEnd);
  updateArrowVisibility(windowStart, windowEnd);

  drawHeader(ctxHeader, slots, zoomLevel, CHART_WIDTH);
  drawFooter(ctxFooter, slots, zoomLevel, CHART_WIDTH);
  drawColumnBg(ctxColumnBg, slots, zoomLevel, CHART_WIDTH);
  drawAccuracy(ctxAcc, slots, zoomLevel, activeGestures, CHART_WIDTH);
  drawPain(ctxPain, slots, zoomLevel, CHART_WIDTH);
  drawMins(ctxMins, slots, zoomLevel, CHART_WIDTH);

  buildAnnot(DAY_DATA, CHART_WIDTH / DAY_DATA.length, CHART_WIDTH);
}
```

**Testing:**
- Navigate between all 4 zoom levels
- Verify headers show correct labels per zoom (week labels for 1mo, month names for 3mo, etc.)
- Verify column backgrounds alternate correctly
- Verify pain circles (1mo/3mo) and candlesticks (6mo/1yr) render
- Verify accuracy lines connect appropriately
- No console errors, no misaligned text

**Commit message:**
```
Task 4: Update all drawing functions for zoom-specific rendering
- drawHeader(): zoom-aware labels (weeks, months, years)
- drawColumnBg(): zoom-aware banding (day parity, month, year)
- drawAccuracy(): lines connect day dots (1mo/3mo) or week/month dots (6mo/1yr)
- drawPain(): circles (1mo/3mo) or candlesticks (6mo/1yr)
- drawMins(), drawFooter(): aggregate per slot
- Update draw() to pass slots and zoomLevel to all functions
```

---

## Task 5: Gesture Checkboxes & Final QA

**Goal:** Replace gesture filter select with interactive checkboxes in the left label panel. Run full QA.

**Work in:** dashboard-v4.html

**Remove:**
- Gesture filter select from toolbar (if present)

**Add to left label panel (Accuracy section):**
```html
<div id="gesturePanel">
  <!-- populated dynamically after data loads -->
</div>
```

**Add to JS:**
```js
let activeGestures = new Set();

function renderGestureCheckboxes(lastSessionWeek) {
  const panel = document.getElementById('gesturePanel');
  if (!panel) return;
  
  panel.innerHTML = '';
  
  // Collect all unique gestures from all sessions
  const allGestures = new Map();
  DAY_DATA.forEach(day => {
    if (day.sess?.gestures) {
      day.sess.gestures.forEach(g => {
        if (!allGestures.has(g.name)) {
          allGestures.set(g.name, g);
        }
      });
    }
  });
  
  allGestures.forEach((gesture, gestureName) => {
    const isEnabled = gesture.introducedWeek <= lastSessionWeek;
    const div = document.createElement('div');
    div.className = 'gesture-checkbox-row';
    
    const checkbox = document.createElement('input');
    checkbox.type = 'checkbox';
    checkbox.id = `gesture-${gestureName}`;
    checkbox.disabled = !isEnabled;
    checkbox.checked = isEnabled;
    
    if (isEnabled) {
      activeGestures.add(gestureName);
      checkbox.onchange = (e) => {
        if (e.target.checked) {
          activeGestures.add(gestureName);
        } else {
          activeGestures.delete(gestureName);
        }
        draw();
      };
    }
    
    const label = document.createElement('label');
    label.htmlFor = `gesture-${gestureName}`;
    label.textContent = gestureName;
    if (!isEnabled) label.style.opacity = '0.5';
    
    div.appendChild(checkbox);
    div.appendChild(label);
    panel.appendChild(div);
  });
}
```

**Add CSS:**
```css
.gesture-checkbox-row {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 0;
  font-size: 13px;
}

.gesture-checkbox-row input[type="checkbox"] {
  cursor: pointer;
}

.gesture-checkbox-row input[type="checkbox"]:disabled {
  cursor: not-allowed;
  opacity: 0.5;
}

.gesture-checkbox-row label {
  cursor: pointer;
  user-select: none;
}

.gesture-checkbox-row input[type="checkbox"]:disabled ~ label {
  opacity: 0.5;
  cursor: not-allowed;
}
```

**Update loadPatient callback:**
```js
// After renderZoomButtons and other initialization:
const lastSessionWeek = Math.ceil(therapyDays / 7); // rough estimate; refine if needed
renderGestureCheckboxes(lastSessionWeek);
```

**QA Checklist:**
- [ ] All 4 zoom levels work without errors
- [ ] Navigation arrows show/hide correctly at edges
- [ ] Date range label updates correctly
- [ ] Gesture checkboxes appear; enabled gestures are checked; disabled are greyed
- [ ] Toggling a gesture checkbox redraws the chart
- [ ] Column backgrounds alternate correctly per zoom
- [ ] X-axis labels match zoom (weeks for 1mo, months for 3mo, etc.)
- [ ] Pain visualization matches zoom (circles for 1mo/3mo, candlesticks for 6mo/1yr)
- [ ] Accuracy lines connect appropriately
- [ ] Annotation lane still works (chips visible at correct positions)
- [ ] Test with 3 different patient datasets (short duration, medium, long)
- [ ] No console errors or warnings
- [ ] Responsive to viewport changes (if applicable)

**Commit message:**
```
Task 5: Add gesture checkboxes and complete zoom system QA
- Replace gesture filter select with interactive checkboxes in left panel
- Enabled gestures (introducedWeek <= last session) are checked and interactive
- Disabled gestures (not yet introduced) are greyed and non-interactive
- Full QA: all 4 zoom levels, navigation, pain rendering, labels, annotation lane
- Test with multiple patient datasets

Closes zoom rebuild epic.
```

---

## Summary

| Task | Focus | Lines of code | Time est. |
|------|-------|---------------|-----------|
| 1 | State & helpers | ~50 | 15 min |
| 2 | Toolbar UI | ~150 | 30 min |
| 3 | Slot building | ~100 | 20 min |
| 4 | Drawing functions | ~300 | 90 min |
| 5 | Gestures & QA | ~100 | 45 min |

**Total:** ~700 LOC, ~3 hours

Do each task in a separate Claude Code session. After each task, test in the browser before moving to the next.
