# Task: Rebuild zoom and navigation system in dashboard-v4.html

Read CLAUDE_CODE_BRIEF.md before starting. This task replaces the existing
Month / 3 Months / All toggle with a new 4-level zoom system driven by a
date-range window. The chart width is fixed at 1000px. Do not change
drawYAxes() or autoY(). All other drawing functions will need updates as
described below.

---

## Fixed chart width

Add at the top of the script:

```js
const CHART_WIDTH = 1000; // px, fixed for prototype
```

Remove the ResizeObserver. All canvas widths use CHART_WIDTH.

---

## The four zoom levels

| Zoom | Label  | Window   | Column unit  | Minutes     | Accuracy    | Pain display          | Bar label | Step size |
|------|--------|----------|--------------|-------------|-------------|-----------------------|-----------|-----------|
| 1mo  | 1 mo   | 28 days  | 1 day        | actual      | actual      | circle + value        | yes       | 7 days    |
| 3mo  | 3 mo   | 84 days  | 1 day        | actual      | actual      | circle + value        | no        | 28 days   |
| 6mo  | 6 mo   | 168 days | 1 week       | total       | average     | candlestick (see §Pain)| yes      | 84 days   |
| 1yr  | 1 yr   | 365 days | 1 month      | total       | average     | candlestick (see §Pain)| yes      | 168 days  |

---

## Which zoom buttons are shown

Zoom buttons are only rendered when the patient's therapy duration makes them
meaningful. Therapy duration = days from first session date to last session date
in DAY_DATA.

| Button | Show when therapy duration >= |
|--------|-------------------------------|
| 1 mo   | always                        |
| 3 mo   | 42 days  (6 weeks)            |
| 6 mo   | 70 days  (10 weeks)           |
| 1 yr   | 112 days (16 weeks)           |

Build the available zoom levels dynamically after data loads:

```js
function getAvailableZooms(therapyDays) {
  const zooms = ['1mo'];
  if (therapyDays >= 42)  zooms.push('3mo');
  if (therapyDays >= 70)  zooms.push('6mo');
  if (therapyDays >= 112) zooms.push('1yr');
  return zooms;
}
```

Render only the buttons for available zooms. Re-render the toolbar buttons
after loadPatient() resolves.

---

## Default zoom on load

Set the default to the **highest available zoom level**, with a maximum of 3mo.
The rationale: recent progress is the clinical focus; 3mo is the working view.
6mo and 1yr are narrative/historical views the therapist navigates to
intentionally.

```js
function defaultZoom(availableZooms) {
  if (availableZooms.includes('3mo')) return '3mo';
  return '1mo';
}
```

---

## Date window state

Replace viewMode with:

```js
let zoomLevel  = '1mo';   // current zoom
let windowEnd  = null;    // Date — last visible day (inclusive)
let windowStart = null;   // always derived, never set directly
```

windowEnd is initialized to the last session date from DAY_DATA after load.

```js
function deriveWindowStart(end, zoom) {
  switch(zoom) {
    case '1mo': return addDays(end, -27);   // 28 days inclusive
    case '3mo': return addDays(end, -83);   // 84 days inclusive
    case '6mo': return addDays(end, -167);  // 168 days inclusive
    case '1yr': return addDays(end, -364);  // 365 days inclusive
  }
}
```

After deriving windowStart, clamp: if windowStart < START, set windowStart = START.
Do not adjust windowEnd when clamping — just show fewer columns on the left.

---

## Navigation: < > buttons

Step sizes:
- 1mo → 7 days
- 3mo → 28 days
- 6mo → 84 days
- 1yr → 168 days

**Stepping forward (>):** add step to windowEnd.
Clamp: windowEnd must not exceed the last session date. If it would, set
windowEnd = last session date.

**Stepping backward (<):** subtract step from windowEnd.
Clamp: derived windowStart must not go before START. If it would, adjust
windowEnd so that deriveWindowStart(windowEnd, zoom) === START exactly:
windowEnd = addDays(START, windowDuration - 1) where windowDuration is the
number of days in the current zoom window.

**Arrow visibility:**
- Hide > when windowEnd === last session date
- Hide < when windowStart === START (after clamping)
- Both hidden means the full dataset fits in the current window — this is
  normal for small datasets in wide zoom levels.

---

## Date range label

Display between the arrows:

```
[windowStart] – [windowEnd]
```

Format:
- Same year: "Jan 8 – Feb 4"
- Different years: "Dec 1, 2024 – Feb 26, 2025"

Use toLocaleDateString with { month:'short', day:'numeric' } and add year
only when the two dates are in different years.

---

## Slot building

A slot is one drawable column. Build the slot array from windowStart to
windowEnd based on zoomLevel. Pass this array to all drawing functions.

### 1mo and 3mo — one slot per calendar day

```js
const slots = [];
let d = new Date(windowStart);
while (d <= windowEnd) {
  const key = d.toISOString().slice(0, 10);
  const dayEntry = DAY_DATA.find(dd => dd.key === key);
  slots.push({
    date: new Date(d),
    sessions: dayEntry?.sess ? [dayEntry.sess] : [],
    type: 'day'
  });
  d = addDays(d, 1);
}
```

### 6mo — one slot per Sun–Sat calendar week

Group days from windowStart to windowEnd into Sun–Sat buckets using
sunWeekIndex(). Each slot:

```js
{
  date: Sunday of the week,   // used for x-axis label
  sessions: [...],            // all sessions in this week
  type: 'week',
  minutes: sum of all session minutes,
  accuracy: { extension: avg%, hook: avg%, ... },  // per gesture average
  pain: { median, min, max, q1, q3 }               // see §Pain aggregation
}
```

Only include slots where at least some days overlap with [windowStart, windowEnd].

### 1yr — one slot per calendar month

Group by year+month. Each slot:

```js
{
  date: first day of the month,
  sessions: [...],
  type: 'month',
  minutes: sum of all session minutes,
  accuracy: { extension: avg%, hook: avg%, ... },
  pain: { median, min, max, q1, q3 }
}
```

Only include months that contain at least one day in [windowStart, windowEnd].

---

## Pain chart — candlestick for 6mo and 1yr

For aggregated zoom levels (6mo, 1yr), pain is displayed as a candlestick
per slot instead of a single circle.

**What each part represents:**
- **Top wick** = max pain value in the period
- **Bottom wick** = min pain value in the period
- **Box top** = 75th percentile (Q3) of pain values
- **Box bottom** = 25th percentile (Q1) of pain values
- **Number in box** = median pain value
- **Box color** = same dark brown as current pain circles (#7B4A2D)

**Pain aggregation function:**

```js
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

**Candlestick rendering in drawPain():**

```js
// cx = center x of slot column
// plotH, yMin, yMax = existing Y scale values
const stats = slot.pain;
if (!stats) return;

const yMedian = valToY(stats.median, yMin, yMax, plotH);
const yQ1     = valToY(stats.q1,     yMin, yMax, plotH);
const yQ3     = valToY(stats.q3,     yMin, yMax, plotH);
const yMin_px = valToY(stats.min,    yMin, yMax, plotH);
const yMax_px = valToY(stats.max,    yMin, yMax, plotH);

const boxW = Math.max(6, Math.min(16, dayW * 0.4));

// Wick (full range line)
ctx.beginPath();
ctx.moveTo(cx, yMin_px); ctx.lineTo(cx, yMax_px);
ctx.strokeStyle = '#7B4A2D'; ctx.lineWidth = 1.5; ctx.stroke();

// Box (IQR)
ctx.fillStyle = '#7B4A2D';
ctx.fillRect(cx - boxW/2, yQ3, boxW, yQ1 - yQ3);

// Median line
ctx.beginPath();
ctx.moveTo(cx - boxW/2, yMedian); ctx.lineTo(cx + boxW/2, yMedian);
ctx.strokeStyle = '#fff'; ctx.lineWidth = 1.5; ctx.stroke();

// Median value label
if (dayW >= 20) {
  ctx.font = `600 ${Math.max(8, boxW * 0.55)}px DM Sans, system-ui`;
  ctx.fillStyle = '#fff';
  ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
  ctx.fillText(String(Math.round(stats.median)), cx, yMedian);
}
```

For 1mo and 3mo, keep the existing circle+value rendering unchanged.

---

## Accuracy lines — connecting logic per zoom

### 1mo and 3mo
Draw lines connecting all session dots. Gaps (rest days) leave no dot but the
line spans across them to the next session dot — this is the existing spanGaps
behavior. Keep as-is.

### 6mo and 1yr
Draw lines connecting all week/month dots. Every slot that has sessions gets
a dot. Connect dots with straight lines (no bezier tension). Slots with no
sessions get no dot and the line spans the gap.

This is intentional — lines always appear at all zoom levels. Do not suppress
lines in 6mo or 1yr.

---

## Gesture checkboxes in the left label panel

Replace the current legend in the Accuracy panel's left label area with
interactive checkboxes. Each gesture gets a row:

```
[✓] Extension        (colored checkbox, active)
[✓] Hook             (colored checkbox, active)
[✓] Tabletop         (colored checkbox, active)
[ ] Closed Fist      (greyed out, disabled — not yet in patient's plan)
[ ] Flat Fist        (greyed out, disabled — not yet in patient's plan)
```

A gesture is **enabled** (interactive checkbox) if its introducedWeek is <=
the week number of the last session.
A gesture is **disabled** (greyed, non-interactive) if it has not yet been
introduced.

State: maintain a Set of active gesture keys. All enabled gestures start checked.
Toggling a checkbox adds or removes the gesture from the active set and calls
draw().

Only active gestures are drawn in drawAccuracy(). Disabled gestures show
greyed label and an empty/outlined checkbox.

Remove the gesture filter select from the toolbar — the checkboxes in the
label panel replace it entirely.

---

## X-axis labels — drawHeader() changes

Update drawHeader() to accept (ctx, slots, zoomLevel, totalW):

### 1mo
Top row: week label ("Aug 1") at each slot where date.getDay() === 0.
Bottom row: two-letter day abbreviation (Su Mo Tu We Th Fr Sa) for every slot.

### 3mo
Top row: month name ("June", "July", "August") at the first slot of each
calendar month. Draw a thicker vertical separator (rgba 0,0,0,0.22, 1.5px)
at month boundaries.
Bottom row: show only the date number (e.g. "1", "8", "15", "22") at each
Sunday slot. Leave non-Sunday day slots blank.

### 6mo
Top row: month name at the first slot of each calendar month.
Bottom row: "Aug 1" style label for the Sunday date of each week slot.

### 1yr
Top row: year label ("2024", "2025") at the first slot of each new year.
Bottom row: abbreviated month name ("Jan", "Feb"…) for each month slot.

---

## Column backgrounds — drawColumnBg() changes

Update drawColumnBg() to accept zoomLevel in addition to existing params.

### 1mo
Alternate by individual day. Parity anchored to day-of-week:
`band = d.date.getDay() % 2`

### 3mo
Alternate by calendar month. All day slots within the same calendar month
get the same band. Draw a thicker line (rgba 0,0,0,0.22, 1.5px) at the
first slot of each new calendar month.

```js
band = (d.date.getFullYear() * 12 + d.date.getMonth()) % 2
```

### 6mo
Alternate by calendar month (each week slot belongs to its Sunday's month).

```js
band = (slot.date.getFullYear() * 12 + slot.date.getMonth()) % 2
```

### 1yr
Alternate by calendar year.

```js
band = slot.date.getFullYear() % 2
```

---

## Annotation lane

Keep the existing buildAnnot() function and annotation lane HTML unchanged.
It operates on DAY_DATA regardless of zoom level.

In 6mo and 1yr views, annotation chip x positions are based on day-level
data mapped to the slot canvas width. This means chips may appear at positions
that do not align perfectly with week/month slot centers — this is acceptable
for now and deferred to a future refinement.

---

## Toolbar HTML — final structure

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

Style nav-controls so the arrow–label–arrow group is centered. Style the
zoom seg on the right. Remove the W+0/Date axis toggle entirely.

---

## Initialization sequence

After loadPatient() resolves:

```js
const firstDate = DAY_DATA.find(d => d.sess)?.date;
const lastDate  = [...DAY_DATA].reverse().find(d => d.sess)?.date;
const therapyDays = Math.round((lastDate - firstDate) / 86400000);

const availableZooms = getAvailableZooms(therapyDays);
renderZoomButtons(availableZooms);   // builds the seg buttons dynamically

windowEnd = lastDate;
zoomLevel = defaultZoom(availableZooms);
setActiveZoomButton(zoomLevel);

draw();
```

---

## draw() sequence

```js
function draw() {
  windowStart = deriveWindowStart(windowEnd, zoomLevel);
  // clamp windowStart to START
  if (windowStart < START) windowStart = new Date(START);

  const slots = buildSlots(windowStart, windowEnd, zoomLevel);

  updateDateRangeLabel(windowStart, windowEnd);
  updateArrowVisibility(windowStart, windowEnd);

  // pass slots and zoomLevel to each drawing function
  drawHeader(ctxHeader, slots, zoomLevel, CHART_WIDTH);
  drawFooter(ctxFooter, slots, zoomLevel, CHART_WIDTH);
  drawAccuracy(ctxAcc, slots, zoomLevel, activeGestures, CHART_WIDTH);
  drawPain(ctxPain, slots, zoomLevel, CHART_WIDTH);
  drawMins(ctxMins, slots, zoomLevel, CHART_WIDTH);

  buildAnnot(DAY_DATA, CHART_WIDTH / TOTAL_DAYS, CHART_WIDTH);
}
```

---

## What NOT to change

- drawYAxes() internals
- autoY() internals
- sunWeekIndex()
- addDays()
- fmtWeekLabel()
- loadPatient() and updatePatientHeader()
- The panel/lane HTML structure (left label + scroll area layout)
- Canvas sizing approach (prepCanvas pattern)
- syncScroll()
