# SignWave Patient Performance Dashboard — Claude Code Handoff Brief

## What this is

This is a **prototype data visualization dashboard** for hand therapists, built as part of the SignWave product by Somos XR. SignWave is a therapeutic game that measures a patient's ability to match hand gesture targets (called "tendon glide gestures") that appear dynamically on screen. The game records accuracy, speed, and session duration per gesture per session.

The dashboard gives hand therapists a visual read of their patient's recovery trajectory over time. It is not a production app — it is a **design prototype** used to validate data display concepts before engineering builds the real system.

---

## Who it's for

**Primary user: hand therapists.** They review this between or after patient sessions to:
- Track accuracy trends per gesture over weeks of therapy
- Identify pain spikes and session gaps that may indicate overuse or non-adherence
- Make decisions about when to add new gestures to the game
- Get a quick visual read of recovery progress to inform clinical decisions

**Secondary context:** the prototype is also shown to stakeholders (product team, investors, clinical advisors) to demonstrate what the data product will look like.

---

## The patient / data scenario

The prototype uses a synthetic but clinically plausible dataset for a fictional patient:

- **Patient:** Tyler M., 27, soldier
- **Injury:** Closed distal radius fracture, treated with closed reduction and casting
- **Timeline:** Weeks 6-26 post cast removal (Jan 8 - Jun 2, 2024 = 147 days, 42 sessions)
- **Gestures introduced progressively:** Extension + Hook (week 6) -> Tabletop (week 8) -> Closed Fist (week 12) -> Flat Fist (week 22)
- **Notable narrative events:** Pain spike + 9-day gap at week 9 (overuse), pain spike + 10-day gap at week 17 (duty-related strain), full recovery by week 26

---

## The five data dimensions

1. **Gesture accuracy (%)** -- hit rate against calibrated target. Shown as multi-line chart, one line per active gesture.
2. **Pain (0-10 PROMS)** -- self-reported pain score per session. Shown as a straight line with filled circles containing the numeric value.
3. **Minutes played** -- session duration. Shown as bars with the minute value printed inside.
4. **Speed score (0-100 normalized)** -- how quickly the patient achieved each calibrated gesture match. Higher = faster. Shown with IQR band. *(Present in earlier explorations, removed in current v4 for simplicity.)*
5. **Gesture quality (0-100% of absolute form)** -- how close to clinically perfect form, independent of the game's calibration. Shown as box plots. *(Present in exploration 04, removed in v4.)*

---

## Why we moved from Chart.js to custom canvas

The prototype went through several Chart.js iterations before being fully rewritten in vanilla Canvas 2D. The switch was not about Chart.js's general capabilities -- it was about specific UI design requirements that Chart.js cannot fulfill without fighting its architecture at every step.

**The UI requirements that broke Chart.js:**

**1. Day-columns-as-spaces, not data points.**
The design treats each day as a column -- a rectangular region bounded by vertical lines -- with data centered inside it. Chart.js renders data on an axis of points. There is no native concept of "a day is a space between two lines." Every approximation (custom tick spacing, bar chart hacks) produced misaligned results that were hard to control precisely.

**2. Shared header/footer row across all charts.**
The design has one row of week labels ("Jan 8", "Mar 3") and one row of day-of-week labels (Su Mo Tu...) that appear once above all charts and mirror below the last chart. In Chart.js, each chart instance owns its own axis -- there is no mechanism to share a single x-axis header across multiple Chart instances while keeping them spatially synchronized.

**3. Value labels inside bars and circles.**
Minutes bars display the minute value printed in white inside the bar. Pain data points display the numeric value inside a filled circle. Chart.js supports datalabels via plugin, but the circle-with-value pattern requires the label to sit inside a drawn shape Chart.js doesn't natively render. Custom afterDraw plugins were brittle and broke on resize.

**4. Y axes on both left and right simultaneously.**
The design shows Y axis tick labels on both edges of each chart -- a common clinical chart convention that helps when scrolled far from the left edge. Chart.js supports dual axes but treats them as separate scales with separate data bindings, not as mirrors. Keeping them identical required dummy datasets.

**5. Sticky Y axis that never scrolls.**
The left panel (title, subtitle, legend, Y axis labels) must remain fixed while the data canvas scrolls horizontally. Chart.js renders axis and data onto a single canvas -- there is no way to split them. The workaround of two separate Chart instances (one for Y axis, one for data) required pixel-perfect height matching that broke on zoom changes because of Chart.js's aspect ratio system.

**6. Canvas height vs. aspect ratio conflict.**
This was the most persistent bug. Chart.js with `responsive:false` still applies its default 2:1 aspect ratio unless `maintainAspectRatio:false` is also set. But even with both flags, calling `resize()` after setting canvas.width to a large value (e.g. 2600px for 147 days x 18px/day) caused Chart.js to recalculate height proportionally, producing ~1300px-tall charts. Pre-sizing the canvas before `new Chart()` partially resolved it but was fragile across view mode changes.

**The conclusion:** Chart.js is excellent for standard chart types where you control data density and don't need precise spatial relationships between chart elements. For a calendar-grid visualization with custom rendering, shared structural elements, and a sticky-panel layout, it added more friction than it removed. The custom Canvas 2D rewrite took fewer lines of code, was easier to debug, and gave complete control over every pixel.

---

## File structure

```
/case-signwave/
  dashboard-v4.html       <- active working file (no hardcoded data)
  tyler-m-001.json        <- Tyler's patient data scenario
  jane-d-002.json         <- (future) second patient scenario
```

**Both files must be in the same folder.** The HTML fetches JSON via `fetch('./tyler-m-001.json')`. This requires a local server (Five Server or Live Server in VS Code). Opening the HTML directly from the filesystem (file://) will fail with a CORS error on the fetch call.

---

## Data architecture

### JSON schema (tyler-m-001.json)

Patient data is separated from rendering logic. Each patient scenario is a self-contained JSON file with three top-level sections:

```json
{
  "patient": {
    "id": "TM-001",
    "name": "Tyler M.",
    "age": 27,
    "occupation": "Soldier",
    "injury": "Closed distal radius fracture",
    "treatment": "Closed reduction and casting",
    "castRemovalDate": "2024-01-08",
    "therapyStartWeek": 6,
    "therapyEndWeek": 26,
    "outcome": "Cleared for return to duty"
  },
  "gestures": {
    "extension": { "label": "Extension", "color": "#378ADD", "introducedWeek": 6 },
    "hook":      { "label": "Hook",      "color": "#D4537E", "introducedWeek": 6 }
  },
  "sessions": [
    {
      "date": "2024-01-08",
      "week": 6, "dayOfWeek": 1,
      "pain": 7, "minutes": 8,
      "note": "First session -- calibration day.",
      "gestures": {
        "extension": { "hits": 6, "attempts": 20 },
        "hook":      { "hits": 5, "attempts": 20 }
      }
    }
  ]
}
```

Key schema decisions:
- **Sessions use real ISO dates** ("2024-01-08"), not week/day offsets. START and TOTAL_DAYS are derived from actual session dates at load time, so the dashboard adapts to any therapy window without code changes.
- **Only active gestures appear in each session's gestures object.** If a gesture hasn't been introduced yet, it is simply absent -- no nulls needed.
- **Gesture config (colors, labels, introduction week) lives in the JSON**, not hardcoded in the HTML. Adding or recoloring a gesture is a JSON edit, not a code edit.

### Loading flow in dashboard-v4.html

```
loadPatient('tyler-m-001')
  -> fetch + parse JSON
  -> derive START and TOTAL_DAYS from session dates
  -> build GCONF, GSTART from gesture definitions
  -> convert { hits, attempts } -> { h, a } internal format
  -> build DAY_DATA array (one entry per calendar day)
  -> update patient header UI (name, injury, cast date, session count, outcome)
  -> set _dataReady = true
  -> call draw()
```

draw() is only called after data is loaded. The ResizeObserver is gated on _dataReady so window resizes before load don't trigger a draw with null data.

### Adding a new patient scenario

1. Create `jane-d-002.json` following the same schema as `tyler-m-001.json`
2. Add `{ id: 'jane-d-002', label: 'Jane D.' }` to the PATIENTS array near the top of the script in `dashboard-v4.html`
3. Reload -- a patient selector dropdown automatically appears in the toolbar when more than one patient is registered

---

## Dashboard architecture (dashboard-v4.html)

### Layout

```
[fixed left label panel 120px] | [horizontally scrollable canvas area]
```

Each chart panel is a flex row. The left panel (title, subtitle, legend) is fixed width and never scrolls. The right panel contains a canvas whose width is set by JavaScript based on dayW x TOTAL_DAYS + YAXIS_W x 2. The canvas is 40px wider on each side to accommodate Y axis labels drawn directly on the canvas.

A shared header canvas (week labels + day-of-week labels) sits above all charts. An identical footer canvas mirrors it below the minutes chart. The annotation lane (chip row for pain spikes and gaps) also uses this layout with a 120px left spacer.

All scroll containers are synced: scrolling any one panel moves all others via onscroll listeners.

### Key functions

- `loadPatient(id)` -- async, fetches JSON, populates all global state, calls updatePatientHeader()
- `draw()` -- main render entry point; called after load, on view change, on resize, on gesture filter change
- `drawColumnBg(ctx, dayW, totalW, h, days)` -- alternating column bands + vertical day/Sunday lines
- `drawHeader(ctx, dayW, totalW, days)` -- week span labels + day-of-week row; used for both header and footer canvas
- `drawAccuracy(ctx, ...)` -- multi-line accuracy chart with session dots
- `drawPain(ctx, ...)` -- straight-line graph with filled value circles on session days
- `drawMins(ctx, ...)` -- bar chart with inline white minute labels
- `drawYAxes(ctx, yMin, yMax, totalW, plotH, tickCb)` -- Y labels on both sides + dashed horizontal guide lines
- `buildAnnot(days, dayW, totalW)` -- renders pain spike and gap annotation chips into the annotation lane
- `sunWeekIndex(date)` -- returns calendar week number anchored to reference Sunday Dec 31 2023; used for band alternation and week grouping
- `autoY(vals, hardMin, hardMax, pad)` -- computes Y axis range from actual data with padding, snapped to nearest 5
- `getDayW()` -- computes pixel width per day column based on view mode and container width
- `snapToRecent()` -- scrolls all panels to the right edge (most recent data) on load and view change

### View modes

- **Month** -- session-level columns, 28 days fills the viewport width. Scrolls left for history. Snaps to most recent on load. Day-level alternating column bands (odd/even by day-of-week parity).
- **3 Months** -- session-level columns, 84 days fills the viewport. Same scroll and snap behavior as Month.
- **All** -- every day fits in the viewport (no scroll needed). Columns sized to fill available width exactly. Week-level alternating bands (grey/white blocks change at Sunday boundary).

---

## Known issues to fix

Fix in this order of priority:

### 1. Date label / Sunday line misalignment

**What:** In Month and 3-Month views, the week label ("Apr 28") and the dark vertical Sunday boundary line do not always sit at the same x position. They can be offset by 1-2 columns.

**Root cause:** drawHeader() groups days by sunWeekIndex() and attempts to find the Sunday column within each group to place the separator line and label. When the group's first data column is not a Sunday (e.g. the dataset's first week starts on Monday Jan 8), lineX may fall back to startI rather than the actual Sunday column index within the group. Meanwhile, drawColumnBg() correctly draws the dark line at every column where getDay()===0, so the two systems are drawing lines at different x positions.

**What correct looks like:** The dark vertical line in the chart body, the tick in the header row, and the "Mon DD" label are all at the same x -- the left edge of the Sunday column.

### 2. Column band alternation off by one in All view

**What:** In All view, the grey/white week bands switch at Monday instead of Sunday. The dark Sunday line is correct but the band color changes one column too late, making the first column of each grey band appear to be Monday.

**Root cause:** sunWeekIndex() is anchored to Dec 31 2023 (a Sunday). The +1 offset in `(sunWeekIndex(d.date) + 1) % 2` shifts which week is grey vs. white, but may not fully resolve for all weeks. The band change and the Sunday dark line must coincide at the exact same column index.

**What correct looks like:** Every grey band starts on a Sunday column. The left edge of each grey band and the Sunday dark line are at the same x coordinate for every week in the dataset.

### 3. Y axis range -- pain whitespace and floating minutes bars

**What:** Pain chart has dead whitespace below the lowest pain value. Minutes bars appear to float above the bottom baseline rather than being anchored to it.

**Root cause:** autoY() uses `Math.floor((min - pad) / 5) * 5` for the lower bound, which over-pads and pushes yMin too low. For pain, values range 1-8 in Tyler's data, so yMin ends up at 0, wasting the lower portion of the chart. For minutes, PAD_TOP (6px) adds air above the tallest bar that makes bars look unanchored from the bottom of the plot area.

**What correct looks like:** Minutes bars grow from the exact bottom edge of the plot area with no gap. Pain yMin starts close to the actual minimum pain value observed -- not at 0.

### 4. Annotation chip overlap

**What:** In Month and 3-Month views, annotation chips ("3-day gap", "Pain +3") can overlap or be cropped when events are close together in time.

**Fix approach:** Simple horizontal collision detection on chip placement -- if two chips would overlap, offset the second one vertically by the chip height + 2px. The annotation lane height may need to increase to 40-44px to accommodate two rows.

---

## Design principles -- do not change without flagging

- **Day columns are spaces, not lines.** Vertical lines are column boundaries. Data points, bars, and circles are centered inside column spaces.
- **Alternating bands** switch at Sunday in All view (week-level blocks). In Month/3-Month view they alternate by individual day, anchored to day-of-week parity so Sunday always starts the same band.
- **Sunday columns** always have a darker (rgba 0,0,0,0.16), slightly thicker (1px) boundary line compared to regular day lines (rgba 0,0,0,0.05, 0.5px).
- **Week labels** always show the Sunday date of that week ("Jan 8"), placed at the left edge of the Sunday column. The separator tick and the label are at the same x.
- **Pain is a straight line** (no bezier tension). Only session days have a circle+value. Rest days have no mark at all.
- **Minutes bars** contain the value in white text. Color encodes effort: green >= 20min, blue 14-19min, pink < 14min.
- **Y axes** appear on both left and right of every chart canvas, drawn directly on the canvas at x=0..YAXIS_W and x=(totalW-YAXIS_W)..totalW.
- **Left label panel** (120px) is a separate fixed-width div that never scrolls. Contains title, subtitle, legend.
- **Header and footer** are identical canvases drawn with the same drawHeader() function.
- **All scroll areas sync** -- scrolling any panel moves all others simultaneously via onscroll.

---

## How to run locally

1. Open the case-signwave/ folder in VS Code
2. Make sure dashboard-v4.html and tyler-m-001.json are in the same folder
3. Click Go Live (Five Server or Live Server)
4. Open http://localhost:5500/case-signwave/dashboard-v4.html

Do not open the HTML via file:// -- the fetch() call will fail with a CORS error.

---

## What success looks like

A therapist opens the dashboard, selects Month view, scrolls left through the timeline, and can immediately see:
- Which days Tyler played and which he skipped
- Where pain spiked and how accuracy responded in subsequent sessions
- Which gestures are performing best vs. lagging
- The overall trend toward recovery across the 20-week arc

The visual language should feel clinical, clean, and scannable -- not decorative. Every pixel should earn its place by conveying information.
