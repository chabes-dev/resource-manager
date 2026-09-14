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
  `resource_mgr_settings_v4` also carries `pinnedMemberId`, `pinnedProjectName`,
  `projectOrder`, and `emptyProjects` (see "Row order & pin" below) — it's a grab-bag
  of small persisted flags, not just `workWeekends`.

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

## Row order, pin, and the "+" ghost rows

- **Members reorder by mutating `state.members` itself** — the array order IS the
  display order (no separate `order` field). Projects aren't a real entity, so their
  order lives in `state.projectOrder` (an array of project-name strings), rebuilt
  from the current on-screen order the moment a project drag starts (see
  `onRowHandleMouseDown`) so untouched projects keep a stable position even before
  they've ever been dragged.
- **Pin is single-pin, per view** (`state.pinnedMemberId`, `state.pinnedProjectName`)
  — deliberately chosen over multi-pin when this was built. If multi-pin is wanted
  later, it needs its own sort key; don't just make `pinnedMemberId` an array without
  rethinking `getOrderedMembers()`.
- **Drag-to-reorder reuses the existing task-drag plumbing**: a grip icon's
  `mousedown` sets `dragType = 'row_reorder'` (+ `reorderView`/`reorderId`), the
  shared `window mousemove` listener moves the row live via `elementFromPoint` +
  `moveInArray()`, `mouseup` persists. Same pattern as `task_move` — don't build a
  parallel drag system for this.
- **Project rows can now exist with zero tasks.** `state.emptyProjects` holds names
  created via the "+ Add project" ghost row; `renderTimeline()` merges them into the
  task-derived project list (deduped by name) so a project survives its first
  bookingless moment instead of vanishing. An empty project's row shows a trash icon
  to delete it — real (task-having) projects don't get one, since deleting *them*
  would need to decide what happens to their tasks and that isn't built.
- **The "+ Add person" / "+ Add project" ghost row is a single shared piece of UI**
  (`renderAddRow`, `state.addingRow`) swapped for a text input on click; Enter
  commits via `quickAddMember`/`quickAddProject`, Escape or blur-with-no-cancel-flag
  handles the rest. `quickAddMember` intentionally skips the People modal (no
  role/capacity/color prompt) — full editing still happens via the modal or
  double-click-to-rename afterward.
- **Header click is a quick day-off toggle, not always the Holidays modal.** A plain
  click (no drag) on a date header creates/removes a single *exact* one-day
  `{start,end}` holiday entry named "Day Off". Dragging across days (`headerDidDrag`)
  still opens the full modal for a named range. A click that lands inside an
  *existing* holiday that isn't itself a 1-day entry it could own also opens the
  modal instead of mutating it — this is deliberate, so a stray single click can't
  silently shrink or delete a named multi-day range like "Christmas Break". If you
  touch `onHeaderMouseDown`/`toggleDayOff`, keep that guard.

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
- Multi-pin (see "Row order, pin" above) — single-pin was the deliberate choice so far.
- Deleting a project that actually has tasks (only empty, task-less projects are
  deletable today, via the trash icon on their row).
- Nothing else currently queued — ask before assuming a feature is wanted; this file
  reflects what's been decided, not a roadmap.

## Testing note (this env specifically)
Real Tailwind/Lucide CDN fetches are blocked by this sandbox's egress proxy policy
(`cdn.tailwindcss.com`, `unpkg.com` both get `connect_rejected`) — Playwright/headless
testing here needs `page.route()` stubs for both (empty `window.tailwind={config:{}}`
and `window.lucide={createIcons(){}}`) rather than hitting the real CDNs. One
consequence: with Tailwind stubbed out, none of its utility classes actually apply
(no real CSS loaded), so pixel-coordinate-based interactions (e.g. dragging a small
icon) become unreliable — prefer calling the handler function directly
(`page.evaluate` with a fake event object) over clicking tiny elements by coordinate
when testing in this environment. On a real deploy (Vercel, or any real browser with
internet) both CDNs load normally and this doesn't apply.
