# TaskGlitch – Bug Fixes & Improvements

This repository contains fixes and improvements made as part of the Round-1 assignment.  
The goal was not only to make the app “work”, but to make it **correct, deterministic, and production-safe**.

Below is a detailed explanation of each identified bug, its root cause, and the implemented solution, followed by additional improvements.

---

## Bug 1: Duplicate Task Fetch on App Load

### Problem
On application load, task data was being fetched more than once.  
This was caused by **React 18 StrictMode**, which intentionally mounts and unmounts components twice in development to surface side-effects.

This resulted in:
- Multiple API calls
- Potential duplicated state updates
- Noisy console/network logs

### Fix
The data-fetching effect was made **idempotent** using an `AbortController`.

- Any in-flight fetch is aborted during cleanup
- Only the final, valid fetch updates state
- Works correctly in both development (StrictMode) and production

### Result
- Tasks load exactly once
- No duplicate state updates
- No reliance on fragile flags or disabling StrictMode

---

## Bug 2: Undo Snackbar Restoring Stale Tasks

### Problem
When a task was deleted, its data was stored in `lastDeleted`.  
If the snackbar auto-closed (without clicking Undo), this state was **never cleared**.

This caused:
- Old tasks reappearing unexpectedly
- Undo working outside its intended time window
- Phantom restores

### Fix
Undo state was explicitly expired when the snackbar closes.

- Introduced `clearLastDeleted()` in the task state layer
- Called this function on snackbar close (manual or auto)
- `undoDelete()` still clears state when Undo is clicked

### Result
- Undo is strictly time-bound
- No stale or phantom task restoration
- State lifecycle matches UI lifecycle

---

## Bug 3: Unstable Sorting (UI Flickering)

### Problem
When multiple tasks had the same:
- ROI
- Priority

their order changed randomly on each re-render.  
This was due to a missing **final deterministic tie-breaker** in the sort logic.

### Fix
A stable, deterministic tie-breaker was added to the sorting function.

Sorting order is now:
1. ROI (descending)
2. Priority (High → Low)
3. Final tie-breaker (created ID / creation time)

### Result
- Sorting is consistent across renders and reloads
- No UI jitter or flickering
- Fully deterministic task ordering

---

## Bug 4: Multiple Dialogs Opening on a Single Click

### Problem
Clicking Edit or Delete buttons inside a task row also triggered the row’s click handler.  
This caused multiple dialogs (View + Edit/Delete) to open simultaneously.

Root cause: **event bubbling**.

### Fix
Event propagation was explicitly stopped on child action buttons.

- `event.stopPropagation()` added to Edit/Delete handlers
- Parent row click handler is no longer triggered unintentionally

### Result
- Only the intended dialog opens per action
- Clean and predictable user interaction

---

## Bug 5: Invalid ROI Calculations (NaN / Infinity)

### Problem
ROI was calculated even when:
- `timeTaken` was `0`, negative, or missing
- `revenue` was invalid

Additionally, invalid `timeTaken` values were being silently auto-corrected to `1`, which produced misleading ROI values.

### Fix
Two changes were made:

1. **Normalization no longer invents data**
   - Missing or invalid `timeTaken` values are preserved as-is

2. **ROI calculation is guarded**
   - ROI returns `null` when inputs are invalid or non-finite
   - No `NaN` or `Infinity` propagates into UI, sorting, or metrics

### Result
- ROI is honest and mathematically safe
- Invalid data does not break analytics
- UI displays placeholders (`—`) instead of misleading numbers

---

## Improvement: Persisting User-Created Tasks with Local Storage

### Requirement
User-created tasks should persist across page refreshes, without replacing the initial API/seed data.

### Implementation
- Initial tasks are always loaded from `/tasks.json` (or seed data)
- Only **user-created tasks** are stored in `localStorage`
- On app load, persisted user tasks are merged with API data

### Result
- Refresh-safe user experience
- Clear separation between system data and user data
- No stale cache or data override issues

---

## Improvement: Persisting Activity Log

### Problem
The activity log was reset on every refresh.

### Fix
- Activity state is hydrated from `localStorage` on load
- Activity updates are persisted automatically
- Log remains capped to a fixed size to prevent unbounded growth

### Result
- Activity history survives refresh
- UI component remains stateless and clean

---

## Commit History

Changes were made using **small, focused commits**, following best practices:

- One bug or concern per commit
- Clear intent in commit messages
- No large “catch-all” commits

This reflects real-world development workflows and makes the review process transparent.

---

## Summary

All fixes focus on:
- Deterministic behavior
- Correct state lifecycle management
- Data integrity
- Predictable UI behavior

The application is now:
- Stable under React StrictMode
- Free of phantom state bugs
- Safe against invalid mathematical inputs
- More resilient and user-friendly

