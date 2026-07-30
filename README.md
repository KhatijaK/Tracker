# Job Search Tracker

A single-file dashboard for running a structured 12-week job search. No build step, no dependencies, no server.

**[Live demo](#)** · **[Source](tracker.html)** — one file, ~900 lines

---

## What it does

Tracks daily and weekly application counts against a fixed plan, surfaces which employers to contact in the current week, keeps a per-week task checklist, and charts the application funnel so the response rate can be compared against a realistic baseline.

The main view is a grid of 225 squares — one per application the plan calls for, grouped into twelve week-rows. Squares fill as applications are logged. This was a deliberate choice over a percentage bar: a completion percentage communicates almost nothing at week five, whereas eighty filled squares is visibly a lot of accumulated work. The interface exists to be looked at on the days the effort feels futile, so making the effort legible was the primary design goal.

## Stack

Plain HTML, CSS, and ES5-compatible JavaScript. One file. The only external request is a webfont, which degrades to system fonts if it fails.

Framework-free was a deliberate constraint. The app has one state object and a few hundred rows of data; a framework would have added a build step and a dependency tree to a problem that did not need either.

## Architecture

One state object is the single source of truth. Every mutation goes through a single function:

```
commit(fn)  ->  fn() mutates state
            ->  render() repaints from state
            ->  saveState() persists in the background
```

Rendering is a full repaint rather than an incremental diff. At this data size that is both fast enough and easier to reason about, and it eliminates the class of bugs where the view and the state disagree. Every displayed number is derived at render time rather than stored, so there is no cached total that can drift.

Persistence writes to two independent stores on every save and reads from whichever holds more data. If both refuse the write, the app keeps running from memory and displays a warning — silently pretending to save is worse than not saving, because the data is lost without the user knowing.

---

## Two bugs worth writing up

Both were reported as "it doesn't work," and neither produced a useful error at the point of failure.

### 1. A crash that only happened in Safari

**Symptom:** `TypeError: undefined is not an object (evaluating 'W.label')`. Worked in Chrome, crashed in Safari.

**Investigation:** `W` comes from `WEEKS[currentWeek() - 1]`, so an undefined `W` meant the week number was out of range. Tracing back, the week number is derived from the number of days between the plan start date and today. Today's date was being produced by `toLocaleDateString('en-CA')` — a common shortcut for getting `YYYY-MM-DD`.

**Cause:** That shortcut is not guaranteed. Chrome returned the expected format; WebKit did not. The unexpected string could not be parsed back into a date, which produced `NaN` days, which produced a `NaN` week number, and `WEEKS[NaN - 1]` is `undefined`. The thrown error appeared several steps downstream of the actual fault, which is why the message pointed at the wrong place.

**Fix:** All dates are now built and parsed from explicit numeric components, with no locale formatting anywhere in the path. Dates anchor at midday so daylight-saving transitions cannot shift a date across a day boundary during subtraction. The parser returns `null` rather than an invalid date, so callers receive a falsy value they must handle instead of a `NaN` that propagates silently.

**Also added:** a validation boundary between storage and rendering, and an error boundary around the render itself. The original failure blanked the entire page; the same fault now produces a readable message and a recovery option.

### 2. Buttons that did nothing, with no error

**Symptom:** Every control in the settings row was inert. No console error, no failed request, nothing in the network tab.

**Investigation:** The working controls were all plain DOM manipulation. The broken ones had one thing in common — every single one called `prompt()`, `confirm()`, or triggered a file download.

**Cause:** A restricted browsing context blocks all of those. It blocks them *silently*: no exception is raised, the call simply returns. There is nothing to catch and nothing to log, which is why the code looked correct and behaved as though it were not running at all.

**Fix:** Each blocked affordance was replaced with an ordinary page element.

| Was | Now |
|---|---|
| `prompt()` for dates | `<input type="date">` in a settings panel |
| `confirm()` before delete | two-press arm-then-confirm button, auto-disarming |
| `alert()` for feedback | inline status text beside the control |
| File download for export | data written to a pre-selected `<textarea>` with a copy button |

The generalized rule: don't depend on a browser affordance the host context can revoke. Everything is now ordinary DOM, which nothing can turn off.

---

## What I'd do next

- Server-side persistence, so the data follows the user across devices rather than living per-browser
- Automated tests around the date arithmetic — it is the component that has already broken once and the one where a fault is least visible
- Import from a CSV export, closing the loop on the existing export

---

## Note on how this was built

Written with AI assistance. I specified the behaviour, made the design and architecture decisions, and did the debugging and testing. The two bug write-ups above are mine — the diagnosis in both cases was the work, and neither symptom pointed at its cause.
