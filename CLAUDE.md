# Resource Manager — project context

A single-file HTML/Tailwind/vanilla-JS team resource-planning timeline (Toggl-Plan-style):
people/project rows across a date grid, drag-to-create and drag-to-reschedule task
bookings, deadlines, vacations, holidays, and a People (team member) manager. No
backend — everything persists to `localStorage`.

This file exists so a new Claude Code session has the history without re-deriving it.
Read this before making structural changes; it captures decisions that look arbitrary
in the code but aren't.

## Stack / constraints
- Single HTML file (`index.html`), no build step.
- Tailwind via the CDN `<script>` (JIT, class-scans the live DOM on mutation) — **needs
  internet access to load**; there's no local Tailwind build. Same for the Lucide icon
  CDN script.
- All rendering is `innerHTML` string-templating + `lucide.createIcons()` after each
  render — no framework, no virtual DOM.
- State lives in one `let state = {...}` object; persisted piecemeal to `localStorage`
  under keys `resource_mgr_tasks_v4`, `resource_mgr_holidays_v4`,
  `resource_mgr_vacations_v4`, `resource_mgr_settings_v4`, `resource_mgr_members_v4`.

## Load-bearing fixes — don't reintroduce these

1. **`renderProjectRow` used to not exist.** `renderTimeline()` called it, Project View
   crashed. It's implemented now, alongside `renderMemberRow` — keep both in sync if you
   change one (row height calc, lane stacking, sidebar layout pattern).
2. **Double sidebar-offset bug (fixed).** The row's task-overlay `<div>` is positioned
   `left: sideWidth` already. `renderTaskBlock`'s own `left`/`deadlineLeft` must be
   relative to *that* div, not the row — do **not** add `sideWidth` again inside
   `renderTaskBlock`. This was the original bug: tasks rendered a full 240px off from
   their real date.
3. **Click-vs-drag on task blocks.** `mouseup` clears `dragType` *before* the browser's
   `click` event fires, so `onTaskClick` can't check `dragType` to know if a drag just
   happened. Fixed via a separate `taskDidMove` flag set inside the `mousemove` drag
   handler and checked (then reset) in `onTaskClick`. If you touch the drag logic, keep
   this flag wired or dragging a task will pop its edit modal open on release again.
4. **Overlapping tasks used to render on top of each other** (no stacking at all).
   `computeLanes()` does greedy interval-scheduling lane assignment; `LANE_HEIGHT` /
   `OVERLAY_TOP` / `rowHeightForLanes()` derive row height and each task's `top` from
   lane count. Both `renderMemberRow` and `renderProjectRow` use this — don't special
   case one without the other, since Project View is *more* likely to have overlaps
   (multiple assignees on the same project/dates).
5. **User-typed text (task/project names, member name/role, holiday titles) is passed
   through `escapeHtml()`** before going into template strings. Don't skip this when
   adding new fields — it's not just XSS hygiene, an unescaped `"` or `'` breaks the
   generated HTML attributes outright.
6. **Members didn't persist or have an edit UI at all** originally — hardcoded
   `DEFAULT_MEMBERS`, wiped every reload. Now full CRUD via the "People" modal,
   persisted like everything else. Deleting a member with tasks assigned **requires**
   picking someone to reassign those tasks to first (blocks deletion if they're the
   only member) — don't let a delete path silently orphan `task.assigneeId`.

## Established conventions (matches the sibling "OS" app's design language)
- Double-click to rename in place: member name/role/avatar initials in the People
  sidebar, project name in Project View (cascades to every task with that project
  string, since projects aren't a real entity — just a shared string on tasks).
  Task titles are **not** inline-double-click-editable — a single click on a task
  already opens its full edit modal, and there's no clean way to layer "double-click
  renames, single click opens everything" without racing the browser's own click
  sequence. Don't try to bolt this on without solving that first.
- Deletes (task/vacation/holiday) show a 5s undo toast (`showToast(msg, undoFn)`)
  rather than a confirm dialog. Member deletion is the exception — it cascades to
  other people's task assignments, so it gets an inline reassignment step instead of
  undo.
- No dark mode, no native `alert()/confirm()/prompt()` — same reasoning as the OS app.
- Column width always fits the visible range to screen width
  (`Math.max(MIN_COL_WIDTH, floor(availableWidth / daysCount))`, `MIN_COL_WIDTH = 56`)
  for every zoom level (14/21/30/45 days). Below the floor it scrolls horizontally
  instead of squashing days unreadable — that's intentional, not a bug to "fix" by
  removing the floor.
- Each team member gets a color from `PERSON_PALETTE` (rotates on add, overridable via
  a color swatch in the People form) — used for their avatar, not for task-block color
  (tasks keep their own independent `task.color`).

## Verifying changes
No test suite. The habit that's caught every real bug so far: after touching
render/drag/state logic, actually load the file (or run it headlessly — jsdom +
stubbed `window.lucide = { createIcons(){} }` works, since jsdom can't fetch the
Tailwind/Lucide CDN scripts) and exercise the specific function, rather than trusting
`node --check` (syntax-only) or eyeballing the diff. Syntax-checking has passed on
every version of this file, including the ones with the double-offset and missing-
function bugs.

## Ideas discussed but not yet built
- Task-block titles inline-editable (see "double-click" note above — needs a real
  design for the click-vs-dblclick race before attempting).
- Any kind of backend/sync — currently single-browser, single-`localStorage`-origin
  only. No accounts, no multi-device.
- Nothing else currently queued — ask before assuming a feature is wanted; this file
  reflects what's been decided, not a roadmap.
