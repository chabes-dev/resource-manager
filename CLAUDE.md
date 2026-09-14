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
7. **A blocked Lucide CDN used to silently break buttons.** `openVacationModal` /
   `openHolidayModal` / `openPeopleModal` all call a `renderXList()` helper — which
   calls `lucide.createIcons()` — *before* unhiding their modal. If `lucide` isn't
   defined (ad blocker, flaky network, offline demo — `unpkg.com` blocked, same as
   this repo's own sandboxed test env), the unguarded call threw and aborted the rest
   of the function, so the modal just never appeared. No console-visible error to a
   normal user — it looked like the button did nothing. Fixed by routing every
   `lucide.createIcons()` call through `safeCreateIcons()` (swallows the error,
   icons degrade to blank, everything else still runs). **Always call
   `safeCreateIcons()`, never `lucide.createIcons()` directly** — grep for
   `lucide.createIcons()` before adding a new call site.
8. **The deadline marker used to be a full-row-height vertical line** (`top:0;
   bottom:0` in the row-overlay), so it visually cut through whatever task card
   happened to sit at that x-position — including tasks in *other* lanes, not just
   its own. Now it's a fixed 12px "pin" anchored just above its own task's card
   (`top: topOffset - 12`, flush with the card's top edge) — shows the exact
   deadline day without ever crossing into the card body. Don't reintroduce a
   full-height line.
9. **The REAL Vacations/Holidays-button bug (the Lucide fix above was real but
   wasn't the cause of this): the People modal's form markup had 2 unclosed
   `<div>` tags** (a leftover `<div class="grid grid-cols-2 gap-3"><div>` that
   should have been deleted when the layout became a 3-column grid, right above
   it). Because of that, `#vacation-modal`, `#holiday-modal`, and everything else
   after `#people-modal` in the HTML were nested *inside* `#people-modal`'s DOM
   subtree instead of being its siblings — present since this file's very first
   commit. While `#people-modal` carried `class="... hidden"` (its default,
   unopened state), every element nested inside it — including the other two
   modals — collapsed to a zero-size box, REGARDLESS of that element's own
   classes. `openVacationModal()`/`openHolidayModal()` still correctly removed
   `hidden` from their own element and `getComputedStyle().display` still read
   `flex` — so anything checking `classList` or the element's own computed
   `display` (exactly what earlier debugging here did) reported "it's open" while
   the screen showed nothing. Only `getBoundingClientRect()` (or an actual
   screenshot) exposed it: `{width: 0, height: 0}`. **Lesson for next time a
   button "does nothing": check the element's actual rendered bounding box, not
   just its own class list/display value — an ancestor can be hiding it.** Also:
   this repo's own no-op-Tailwind-stub testing convention (see "Testing note"
   below) never would have caught this either, since it's pure HTML structure,
   not CSS — when a modal "isn't opening" is reported here again, compile the
   real Tailwind output (`npx tailwindcss -i input.css -o output.css` against
   this file, content glob pointed at it) and check a real bounding rect before
   concluding the JS is fine.
10. **Shift+click on a member's day cell toggles their personal vacation for
    that single day** (`onCellMouseDown` checks `e.shiftKey` before starting the
    task-create drag; `toggleVacationDay()` mirrors `toggleDayOff()`'s guard — it
    only ever owns a single-day entry it created, a click inside an existing
    multi-day vacation opens the Vacations modal instead of mutating it). Plain
    click there is already task-create, so this couldn't be a bare click.

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
- Task cards fill with a light tint of `task.color` (inline `background-color:
  ${task.color}22` — 6-digit hex + 2-digit alpha, ~13%), not `bg-white`. Assumes
  `task.color` is always a 6-digit `#rrggbb` hex (true everywhere it's set today);
  if that ever stops holding, the alpha-suffix trick breaks silently.
- Company-wide days off (`state.holidays`, "Holidays" button) and one person's days
  off (`state.vacations`, "Vacations" button) are two separate, deliberately
  different features — not a bug that there are two. Holidays got a fast single-click
  header toggle (see "Header click is a quick day-off toggle" above); Vacations is
  still modal-only (pick member + date range). If a faster per-person toggle gets
  built, it can't be a plain click on a day cell — that's already task-create.

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

**The no-op Tailwind stub is fine for logic tests but blind to real layout/CSS bugs**
(see load-bearing fix #9 — a pure-HTML nesting bug that made two modals silently
collapse to zero size). `npm install tailwindcss@3` works in this sandbox (the npm
registry isn't blocked, only the CDN hosts are) — when a bug report is specifically
about something not being *visible* (a modal, a dropdown, anything about layout
rather than state), compile the real CSS and check actual `getBoundingClientRect()`
/ take a screenshot, don't trust `classList`/computed-`display` checks alone:
```
mkdir -p /tmp/twtest && cd /tmp/twtest && npm init -y && npm install tailwindcss@3
# tailwind.config.js: content: ["/path/to/index.html"]
# input.css: @tailwind base; @tailwind components; @tailwind utilities;
npx tailwindcss -i input.css -o output.css
# then page.route the cdn.tailwindcss.com request to inject output.css as a
# real <style> tag (plus `window.tailwind = {config:{}}` so the page's own
# `tailwind.config = {...}` assignment doesn't throw) instead of a no-op stub.
```
