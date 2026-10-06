---
area: Tasks
status: built
verified_against: frontend@1bb6aa5d, backend@577bae44
verified_on: 2026-10-06
---
# Tasks

## What it does
A task is a to-do with a title, priority, optional due date (all-day or timed), optional reminder, assignee, notes and a checklist. It can be linked to one primary record (contact, lead or company) and, independently, to one deal. The Tasks page (`/tasks`) is a personal triage list: preset tabs (Today, Upcoming, Overdue, Completed, All), assignee and linked-type filters, search, saved views, an advanced filter, an inline quick-add bar, expandable rows with a live checklist, a bulk bar, and a List/Board toggle where the board groups tasks by priority. A side drawer creates and edits a task. Every contact, lead, company and deal page has a Tasks card scoped to that record. Workspace task defaults pre-fill new tasks, and the backend sends reminder, due and overdue notifications to the assignee. Tasks are a standalone object; there is no calendar.

## Screens
| Route (app) | Prototype page id | Purpose |
|---|---|---|
| `/tasks` | `tasks` | Task list and priority board |
| (side drawer `TaskFormSheet`, from New Task, a row's actions, a board card, timeline Task launchers, record Tasks cards, Quick add) | — (prototype `TaskFormModal`) | Create or edit one task |
| (card `RecordTasks` on contact, lead, company and deal pages) | — (prototype `RecordTasks`) | A record's own tasks with quick-complete and pre-linked Add task |
| `/settings?tab=tasks` | `settings-tasks` | Workspace task defaults (page UI owned by the settings doc; rules below) |

## Behaviour and rules
**Task fields** (`internal/modules/tasks/service/service.go`, `schemas/taskSchema.ts`)
- Title required, at most 500 characters. Priority low / medium / high (default medium). Notes at most 20,000 characters, written with the note composer (@mentions of members, #tags).
- Checklist (subtasks): at most 100 items, each at most 500 characters. The server assigns item ids.
- Due: a civil date, optionally a time and an IANA time zone. An all-day task is pinned to 09:00 in the workspace zone. Quick chips: Today, Tomorrow, Weekend, Next week. On edit, an untouched due field is not re-sent; clearing it sends `null`.
- Reminder: None, At due time, 5, 10, 15, 30 min, 1 hour or 1 day before (any value ≥ 0 is accepted by the API). "Also email me" appears once a reminder is set.
- Assignee must be an active member of the workspace (422 `ASSIGNEE_NOT_MEMBER`).
- Related record: Type (Contact / Lead / Company) plus a record picker, and a separate Deal picker. At most one primary record. A related record from another workspace is 400 `INVALID_REFERENCE`.
- Marking done stamps `completed_at`; reopening clears it.
- Creating a task, and completing it, writes a `task` activity on the primary related record's timeline (not on the deal when both are set). Un-completing writes nothing.
- Assigning a task to someone, and @mentioning someone in its notes, sends them an in-app notification.
- Delete is a soft delete. Deleting one task (row menu or drawer) happens at once with no confirmation; bulk delete asks first.

**Tasks page** (`TasksWorkspace.tsx`)
- Defaults: preset Today, assignee Me. All list state (preset, assignee, type, search, advanced filter, view) is in the URL.
- Presets: Today = open and due today, plus overdue. Upcoming = open and due after today. Overdue = open and due before today. Completed = done. All. Days are computed in the workspace time zone. Each tab shows a count.
- Grouping: Today shows Overdue / Today sections; Upcoming shows This week / Later; All shows Overdue / Today / Upcoming / Done.
- Top bar: "Tasks", "N open · N overdue", New Task. A "My day" banner shows overdue and due-today counts as shortcuts.
- Toolbar: search (title), preset tabs, Assignee (Me, Anyone, each member), Type (All types, Contacts, Companies, Leads, Deals, Unlinked), saved views, advanced filter (Title, Type, Priority, Completed, Owner, Due date), Clear filters, List/Board toggle.
- Saved views (object `tasks`): store preset, assignee, type, search, view and advanced filter. No favourites.
- List: 20 rows per page with Previous/Next. Rows: select checkbox, done toggle, title (click to expand), linked-record chip (opens the record), due label, priority, assignee, actions menu (Edit task, Delete task). The expanded row shows due, priority, related records, assignee, created date, notes, and a checklist where ticking an item saves at once and "Add subtask…" appends one.
- Advanced-filter conditions the API can express are sent to the server; the rest run on the loaded rows, and the page shows a notice when results may be incomplete.
- Quick-add bar (list view): title, priority, due date, assignee, optional "Link to…" record. Creates an all-day open task. Priority, due and assignee start from the workspace defaults and stay set after each add; only the title clears.
- Board: three priority columns, Low → Medium → High. Cards show the assignee avatar. Drag a card (by its grip) to another column to change priority; arrow keys move between columns. Quick-complete on the card. "Load more" pages in more tasks. No multi-select on the board.
- Bulk bar (list view only): header select-all plus per-row select. Complete, Reassign, Reschedule (Today, Tomorrow, In a week, or a date; each task keeps its own time) and Delete (confirm). `POST /tasks/bulk`, at most 100 ids; it can partly succeed, and failed rows stay selected.
- Sidebar Tasks badge: overdue + due today, for me (`GET /tasks/counts`).
- Global search: a task hit opens `/tasks` filtered to that title (there is no task detail page).

**Record Tasks card** (`RecordTasks.tsx`)
- Lists the record's tasks via `GET /tasks?rel_type=&rel_record_id=` (for a deal it also matches tasks whose deal slot is that deal). Open and done groups, a priority filter, Due/Recent sort, quick-complete, and Add task (drawer pre-linked to the record; the link is locked for contact/lead/company).

**Reminders and notifications** (`internal/modules/tasks/notification/service.go`, worker every 60 s)
- Only open tasks with a due date and an assignee who is an active member. All-day tasks count as due at 09:00.
- Reminder: fires `reminder_minutes` before due. Always an in-app notification; an email too only if "Also email me" is on.
- Due now and Overdue: an in-app notification and an email to the assignee. These do not depend on the email toggle.
- Each notice is sent once per task. Notices older than 24 hours are skipped (no backlog burst after an outage). The time in the email is shown in the task's own time zone, else the workspace zone; all-day tasks show a date only.

**Task defaults** (`internal/modules/tasks/service/defaults.go`)
- Five workspace settings: default priority (low / medium / high), default reminder (none, 5, 15, 30, 60 or 1440 min), email reminder on/off, default assignee (Me = the creator, or Unassigned), default due date (none, today, tomorrow, in 3 days, in 1 week).
- System defaults: medium, no reminder, email off, Me, no due date.
- They pre-fill the task drawer and the quick-add bar in create mode. Values passed by the caller (for example a record link) win.

## Permissions
- Owner, admin and member can all create, view, edit, complete, delete and bulk-edit any task in the workspace, including tasks assigned to others. There is no role check on tasks in the backend.
- Task defaults: everyone can read them; only owner and admin can change them (403 otherwise; the UI shows them read-only to members).
- Saved views: as for every list (creator deletes a personal view; creator or owner/admin deletes a shared one).

## API
- `GET /tasks` · `POST /tasks` · `GET /tasks/{id}` · `PATCH /tasks/{id}` · `DELETE /tasks/{id}`
- `PATCH /tasks/{id}/subtasks/{subtaskId}`
- `POST /tasks/bulk`
- `GET /tasks/counts`
- `GET /settings/tasks` · `PATCH /settings/tasks`
- `GET /saved-views` · `POST /saved-views` · `PATCH /saved-views/{id}` · `DELETE /saved-views/{id}` (object `tasks`)

## API only (no UI yet)
- `GET /tasks` sorts other than due date (`recent`, `oldest`, `due_desc`, `priority_*`, `title_*`, `updated`): the page always sorts by due date ascending.
- `GET /tasks/counts` `today` / `upcoming` / `open_total`: only the badge sum is used.

## Not built (lives in the vision design)
- Calendar: the "Calendar" button on the Tasks page, the "Tasks sync with your Calendar" footer, tasks shown as calendar events, scheduling meetings from records.
- Manual drag-to-reorder of rows in the list (the ⠿ handle and stored order).
- Seeded shared views (High priority, Due this week) and favourite views.
- Role-based task permissions (separate create / edit / delete rights per role).
- Repeat / recurring tasks.
- A per-member default assignee in task defaults.

## Known gaps
- The task drawer still says "Reminders aren't delivered yet — saved for when scheduling ships" under the reminder field (`components/ReminderField.tsx`), but the backend scheduler is live and sends reminders. The copy is stale.
- The global search "pages" list shows Tasks as "Coming soon" with no link, although `/tasks` exists (`components/layout/sidebar-search.tsx`).
- The bulk delete confirmation says tasks can be restored from the Recycle Bin within 30 days; the Recycle bin settings entry is not built, so there is no restore UI.
- The default-reminder list (5, 15, 30, 60, 1440) differs from the drawer list (also offers At due time and 10 min); the API rejects 0 and 10 as defaults.
- A task linked to both a primary record and a deal writes its timeline activity only to the primary record.
- The advanced filter can only push a subset of conditions to the server; with many tasks, filtered results may be incomplete (a notice says so).
- `features/tasks/README.md` lags the code (it still describes a single 100-row fetch and saved views as deferred).
