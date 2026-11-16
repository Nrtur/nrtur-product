---
title: Pre-launch permission fixes
status: proposed        # proposed | approved | in-progress | built | dropped
owner: Qamar Ul Islam
area: docs/permissions.md, docs/areas/tasks.md, docs/areas/leads.md, docs/areas/workspace-settings.md, docs/areas/billing.md
screens: []             # no new screens; members just stop seeing buttons they can't use
tickets: []             # filled in after approval
created: 2026-11-16
---

# Pre-launch permission fixes

## Problem
QA found that members can do things meant for owners and admins. The UI often hides the button, but the API allows it.
- **Tags:** members can create and delete workspace tags.
- **Calls:** members can edit and delete any call log.
- **Tasks:** members can edit and delete other people's tasks, including the owner's.
- **Converted leads:** members can edit them.
- **Activity feed:** members can read the whole team's activity feed.
- **Task settings:** a member's `PATCH /settings/tasks` returns 500 instead of 403.
- **Read-only workspaces** (cancelled, or the trial ended) can still create saved views, custom fields and lead statuses.

Some of these are written down in `docs/permissions.md` as today's behaviour, so fixing them changes the documented rules. This proposal sets the rules.

## Proposal
Enforce the rules below **in the API**, and hide the controls in the app to match. There are no new screens.

## Rules
- **Tags**
  - **Create:** everyone. Reps need to tag records as they work, and a new tag is harmless.
  - **Delete:** owner and admin only. The API refuses members with 403.
- **Call logs:**
  - members edit and delete only the calls they logged;
  - owners and admins can edit and delete any call log;
  - everyone can still see all calls (subject to private records).
- **Tasks:**
  - everyone can create tasks and see all tasks;
  - members can edit, complete and delete only tasks they **created or are assigned to**;
  - owners and admins can change any task;
  - bulk edit skips tasks a member can't change and says how many were skipped.
- **Converted leads:**
  - read-only for everyone: fields, owner and status are locked (409 `LEAD_CONVERTED`);
  - notes and tasks can still be added, so the history stays complete;
  - owners and admins can still archive one.
- **Activity feed** (`GET /activities`): owner and admin only, matching the app.
- **Task settings** (`PATCH /settings/tasks`): a member gets **403**, not 500.
- **Read-only workspace:** **every** write is refused, the same as other writes today (`422 PLAN_FEATURE_UNAVAILABLE`, tier `read_only`). This includes saved views, custom fields and lead statuses.

## Permissions

| | Owner | Admin | Member |
|---|:-:|:-:|:-:|
| Tags: create | ✓ | ✓ | ✓ |
| Tags: delete | ✓ | ✓ | — |
| Call logs: edit, delete | any | any | own only |
| Tasks: edit, complete, delete | any | any | created by or assigned to them |
| Converted lead: edit fields | — | — | — |
| Activity feed | ✓ | ✓ | — |

## Plan and pricing impact
None.

## Out of scope
- Record-level visibility. That is covered by `proposals/2026-11-private-mailboxes-and-records.md`.
- The legacy `manager` role.

## Open questions
- **Tag create:** should members be able to create new tags? *Recommendation: yes (see Rules). Choose "no" if you want a fixed tag list per workspace.*
