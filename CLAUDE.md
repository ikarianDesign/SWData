# SWData Medical Dashboard

## Project Overview

Clinical monitoring application for Somos XR's virtual reality hand therapy. The dashboard displays patient session performance, accuracy metrics per gesture, pain levels, and session duration trends over configurable time windows.

**Current Phase:** Fix zoom system, UI features, and bugs in dashboard-v4.html  
**Future Phase:** Apply design system (post-feature completion)

---

## Tech Stack

- **Frontend:** HTML5, Canvas API, JavaScript
- **Data:** JSON patient records with session arrays (DAY_DATA global)
- **Styling:** CSS Grid + Flexbox, light/dark theme support
- **Target:** Desktop browser (1920px+ viewport), clinician-facing
- **Framework:** Currently vanilla JS. Propose frameworks if they'd improve the architecture; I'll discuss and decide.

---

## Key Files

| File | Purpose |
|------|---------|
| `dashboard-v4.html` | Main prototype; canvas-based chart system |
| `built-site/` | Current production-ready builds |
| `ZOOM_SPEC.md` | Detailed task: 4-level zoom rebuild (1mo → 1yr) |
| `CLAUDE_CODE_BRIEF.md` | UX context and data structure reference |
| `early iterations/` | Exploration, archived prototypes |

---

## Current Task: Zoom & Navigation System

**File:** `dashboard-v4.html`  
**Spec:** `ZOOM_SPEC.md` (read first)

### What's changing:
- Replace Month/3mo/All toggle → 4-level zoom system (1mo, 3mo, 6mo, 1yr)
- Fixed chart width: `CHART_WIDTH = 1000px` (remove ResizeObserver)
- Dynamic toolbar buttons (shown/hidden based on therapy duration)
- New navigation: `<` `[date range]` `>` arrows with step sizes per zoom
- Gesture checkboxes in accuracy panel (enabled/disabled by introduction week)
- Aggregation logic for 6mo/1yr: pain candlesticks, accuracy percentiles

### Critical constraints:
- **Do NOT change:** drawYAxes(), autoY(), sunWeekIndex(), addDays(), fmtWeekLabel()
- **Do NOT remove:** annotation lane (buildAnnot stays as-is)
- All drawing functions receive (ctx, slots, zoomLevel, CHART_WIDTH)
- Clamping logic: derive windowStart from windowEnd + zoom duration, never adjust windowEnd when clamping

### Data structure reference:

**DAY_DATA (global):**
```js
[
  { key: "2024-12-01", date: Date, sess: { pain: 5, minutes: 23, gestures: [...] } },
  { key: "2024-12-02", date: Date, sess: null },  // rest day
  ...
]
```

**Gesture object (inside session):**
```js
{ 
  name: "Extension",
  introducedWeek: 2,
  accuracy: { ... },  // frame-by-frame accuracy per gesture
  minutes: 5
}
```

---

## Drawing Functions (all need updates)

| Function | Current params | New params | Changes |
|----------|---|---|---|
| `drawHeader()` | ctx, slots, totalW | ctx, slots, zoomLevel, totalW | Multi-level labels per zoom; month separator lines (3mo) |
| `drawColumnBg()` | ctx, slots, totalW | ctx, slots, zoomLevel, totalW | Banding logic per zoom (day parity, month, year) |
| `drawAccuracy()` | ctx, slots, activeGestures, totalW | ctx, slots, zoomLevel, activeGestures, totalW | Lines connect week/month dots (6mo/1yr); skip drawYAxes() pass |
| `drawPain()` | ctx, slots, totalW | ctx, slots, zoomLevel, totalW | Circle+value (1mo/3mo); candlestick (6mo/1yr) |
| `drawMins()` | ctx, slots, totalW | ctx, slots, zoomLevel, totalW | Aggregate minutes per slot |
| `drawFooter()` | ctx, slots, totalW | ctx, slots, zoomLevel, totalW | Label format per zoom |

---

## UX Patterns (from CLAUDE_CODE_BRIEF)

- **Pain visualization:** dark brown (#7B4A2D) circles (detail) → candlesticks (trend)
- **Gesture toggling:** checkboxes in left label panel; disabled (grey) for not-yet-introduced gestures
- **Navigation:** step sizes match zoom level (7d, 28d, 84d, 168d); arrows hide when at dataset edges
- **Focus:** recent progress is clinical priority; 3mo is the default working view

---

## Before You Start

1. Read `ZOOM_SPEC.md` in full — it's the single source of truth for this task
2. Understand the slot building logic for each zoom: 1mo (per day), 3mo (per day), 6mo (per week), 1yr (per month)
3. Pain aggregation helper is provided in the spec (painStats function)
4. Test with patient datasets of varying durations (< 6 weeks, 8 weeks, 12 weeks, 16+ weeks) to verify button visibility logic

---

## Local Development Notes

- Open dashboard-v4.html directly in browser (no build step needed)
- Use browser DevTools Console to inspect DAY_DATA after loadPatient()
- Canvas rendering is synchronous; timing issues rare but check requestAnimationFrame if scrolling stutters
- Annotation lane operates independently; changes to zoom do not require annotation updates
