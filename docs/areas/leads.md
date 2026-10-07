---
area: Leads
status: built
verified_against: frontend@1bb6aa5d, backend@577bae44
verified_on: 2026-10-06
---
# Leads

## What it does
Leads are unqualified prospects that a team works before turning them into contacts. The Leads page is a server-paginated table with system views (including Converted and Archived buckets), user saved views, a condition-builder filter, a column chooser, a full bulk bar (owner, status, tags, convert, archive, delete), and CSV import and export. Every lead carries a system-computed 0–100 score with a Cold / Warm / Hot band; a workspace tunes the scoring rules under Engage → Rules → Lead scoring. Converting a lead creates a contact and, optionally, a company and a deal, in one step for one lead or in bulk. The lead page shows inline-editable properties, the score, custom fields, tasks, a note timeline, and (once converted) a link to the contact it became.

## Screens
| Route (app) | Prototype page id | Purpose |
|---|---|---|
| `/leads` | `leads` | List: views rail, search, filters, columns, bulk bar, import/export, duplicates mode |
| `/leads/[id]` | `lead-detail` | Lead record: properties, score, recommended action, tasks, timeline, convert |
| (side sheet on `/leads`) | `add-lead` | Create a lead (`LeadFormSheet`); edit uses `LeadEditSheet` |
| (dialog) | — (`ConvertLeadModal` in `leads`/`lead-detail`) | Convert one lead (`LeadConvertDialog`) |
| `/engage` → Rules → Lead scoring | `settings-scoring` (also Engage › Rules in the prototype) | Scorecard for every role on plans with `lead_scoring` (the GET is plan-gated too); rules editor for owner/admin |

## Behaviour and rules
**List**
- Page size 25; backend default 20, max 100.
- Search `q` matches name, email and company name.
- Sort: Lead name, Score and Created (`name_asc|name_desc|score_asc|score_desc|recent|oldest`). Est. value is not sortable.
- Columns: Lead, Company, Source, Status, Score, Owner, Est. value, Created (default); Phone, Tags (optional). Status and Owner are editable inline. Column choice is remembered in the browser per workspace and user.
- System views: All Leads, My Leads, New, Working, Qualified · Ready (the browser resolves the status name to an id, and the server filters by `status_id` / `owner_id`); Hot Score, This Week, No Owner (filtered in the browser from the loaded page only, with a notice saying so); plus Converted (`converted=only`) and Archived (`archived=only&converted=include`) at the bottom of the rail.
- The open list hides converted and archived leads by default (`archived` and `converted` are independent `active|include|only` gates).
- Saved views: `/saved-views` with `object_type=leads`: personal or shared, rename, save changes, delete. Same rules as contacts.
- Filters: condition builder, AND only. Core fields Source, Status, Owner, Tags take one value each ("is any of" / "has any of"); custom fields go to `cf_filters` (≤ 10 clauses, ≤ 4096 bytes).
- Bulk bar: Assign owner (or Unassigned), Change status, Add tag, Remove tag, Convert, Archive / Restore, Delete, Clear. Everything except Convert goes through `POST /leads/bulk` (at most 100 ids, per-row results); Convert uses `POST /leads/convert` (see Conversion). The selection is one page (at most 25) and is dropped on any page or view change. Change status and Convert are withheld in the Archived and Converted views, and Assign owner is also withheld in the Archived view. A converted or archived lead's status cannot change (`LEAD_STATUS_LOCKED`, reported per row).
- Row actions (hover-revealed buttons, no menu): Convert (not for converted/archived), Archive (confirm); in the Archived view, Restore (no confirm).
- Inline Status editing stays enabled in the Converted view; a change there is refused with 409 `LEAD_STATUS_LOCKED`.
- Tools gear: Find & merge duplicates (`GET /leads/duplicates`: name, email and phone signals; plan feature `duplicate_detection`; a converted or archived lead cannot be merged), Manage properties (owner/admin), Manage statuses.
- Export: `GET /leads/export` streams the current view as CSV (same filters, including `converted` and `cf_filters`). It is disabled for the browser-filtered views (Hot Score, This Week, No Owner) and in duplicates mode, and it refuses (with a toast) while the filter is over the server's cap, a registry it needs failed or is still loading, or the view's status no longer exists.
- Import: `BulkImportWizard object="lead"`. It is create-only (`duplicate_strategy=create`); `name` is the one required column. A row whose email or phone digits match an active lead (or an earlier row in the file), or whose email belongs to a contact, is skipped and reported. Same file limits as contacts.

**Create / edit**
- Fields: full name, company name, company domain, job title, email, phone, source, status, estimated value, expected close date, owner, tags, industry, employees, plus custom fields. The edit sheet (`LeadEditSheet`) has custom fields but no tags; tags are edited from the lead page.
- Rules (UI and API agree): name or email required on create; the edit sheet also requires a name. Email must be a valid bare address. Phone: 7–15 digits with optional `+`; separators allowed; stored as typed (not E.164). Free-text fields are trimmed; only the API refuses control characters (422 `INVALID`).
  - Width limits: UI caps name, company name, company domain and job title at 120, industry at 80 and phone at 50. The API allows name, company name, company domain, industry and email 255, job title 150, phone 50.
  - Estimated value ≥ 0; employees a whole number ≥ 0; expected close date `YYYY-MM-DD`.
- Source must be one of: Web form, Referral, Cold outreach, Event, Ad, LinkedIn, Import.
- Status is workspace-configurable (`/lead-statuses`). Seeded: New (default), Working, Nurturing, Qualified, Unqualified. Converted and Archived are lifecycle states, not statuses.
- Owner defaults to the creator and must be an active member (`ASSIGNEE_NOT_MEMBER`).
- Duplicates are refused at write time:
  - 409 `DUPLICATE_LEAD` for the same email or phone digits as another active lead.
  - 409 `LEAD_EMAIL_IS_CONTACT` when the email belongs to a contact.
  - Both show inline with a link to the existing record.
- A client-sent score is not used: an in-range value is replaced by the computed score, and one outside 0–100 is rejected with 400.
- Plan cap: New Lead and Create are gated by the per-type leads cap (trial 2,500, Solo 5,000, Team 50,000, Pro 500,000, Business 2,000,000).

**Lead page**
- Header: breadcrumb, record switcher, an "Archived" chip when archived, Convert Lead (or a "Converted" chip plus "View contact"), Enroll in sequence (hidden once converted, disabled when archived), a disabled Schedule button ("coming soon"), ⋯ menu with Edit lead (disabled when converted or archived) and Delete lead (confirm).
- An archived banner shows when the lead is archived.
- Left column:
  - Hero card with the lead score bar (or "Not yet scored").
  - About panel, inline edit: name, company, job title, domain, email, phone, source, status, expected close date, estimated value, owner, tags. Everything locks when the lead is converted or archived.
  - Custom fields.
- Right column:
  - "Recommended next action" card, only for a Hot lead that is not converted or archived: "This lead is ready to convert!" with "Convert now".
  - Tasks.
  - Timeline: `GET /leads/{id}/activities`, paged 20 at a time, up to 100. Composer: Note (attachments, mentions) and Task, both disabled once converted. Recorded lead events: status_change, assignment, lead_converted, deal_created, notes.
  - Conversion history once converted.

**Conversion** (`POST /leads/{id}/convert`, `service/convert.go`; one transaction)
- Always creates a contact. Contact fields are read-only in the dialog: name split into first/last, email, job title, plus owner, source and live tags carried over. The phone is composed to E.164 or dropped. The contact lands on the default contact status.
- Optional company ("Create company", on by default when the lead names one): find-or-create by company name, linked as the contact's primary company. Company details are read-only.
- Optional deal ("Create deal"): editable deal name and value; pipeline choice; the stage is always the pipeline's first stage. Tags are carried to the deal, never to the company.
- An already-converted lead returns 409 with the existing contact (idempotent). An archived lead returns 409. A same-email contact returns 409, and the dialog offers "Link to existing contact" (`on_duplicate=link`), which unions the tags.
- The contact, company and deal are each checked against their own plan cap, but as one save: if any one is refused, the whole conversion rolls back.
- On success the app goes to the new deal, or to the contact if no deal was created. The lead keeps `converted_at` / `converted_to_contact_id`, and its status is locked.
- Bulk convert (`POST /leads/convert`, up to 500 server-side and guarded at 500 in the client (`BULK_CONVERT_MAX`); in practice the selection is one page, at most 25): company/deal toggles, a pipeline select, the estimated value as the deal amount, `on_duplicate=link`. Each lead converts in its own transaction.

**Lead scoring**
- Computed by the backend (`leads/service/scoring.go`) on create, update, status change and merge, clamped to 0–100. Assign does not rescore. A rules change enqueues a workspace re-score.
- Signals, with default points:
  - Profile fit: has company 5, has phone 5, preferred industry 15, senior title keyword 15, company-size tiers 5 / 15 / 20 (first matching tier wins).
  - Engagement: email opened, link clicked.
  - Behavior: page visit, demo request.
  - Engagement and Behavior always score 0 today (nothing produces those events); the UI labels them "Not scoring yet".
- Bands: Cold < 34, Warm 34–66, Hot ≥ 67, tunable via `warm_min` / `hot_min`. Hot wins ties.
- Editor rules: points and caps 0–100 (a cap of 0 disables a signal); `max_employees` 0 means no upper bound; `warm_min` ≤ `hot_min`. PUT is a full replacement, and there is no "reset to defaults".
- The rules screen is a plan feature (`lead_scoring`: Trial, Pro, Business; not Solo or Team). The score itself is computed on every plan.

## Permissions
- Owner, admin and member can all create, edit, delete, archive, assign, bulk-edit, convert, import, export and merge leads. There is no role check on lead records.
- Every member sees every lead.
- Owner/admin only (`config:manage`): lead-status CRUD and the default status; custom-field definitions; editing lead-scoring rules (`PUT /lead-scoring-rules` returns 403 for a member). Members see the scorecard read-only (both `GET` and `PUT /lead-scoring-rules` require the `lead_scoring` plan feature).
- Saved views: as for contacts (creator, or owner/admin for shared views).
- Legacy `manager` role: passes capability checks like admin, but is refused wherever the backend checks for owner/admin literally (see [permissions.md](../permissions.md)).

## API
- `GET /leads` · `POST /leads` · `GET /leads/{id}` · `PUT /leads/{id}` · `DELETE /leads/{id}`
- `POST /leads/{id}/archive` · `POST /leads/{id}/unarchive`
- `POST /leads/{id}/assign` · `POST /leads/{id}/unassign` · `PATCH /leads/{id}/status`
- `POST /leads/{id}/tags` · `DELETE /leads/{id}/tags/{tagId}`
- `POST /leads/bulk` · `POST /leads/convert` · `POST /leads/{id}/convert`
- `GET /leads/export`
- `GET /leads/{id}/activities` · `POST /leads/{id}/activities` · `POST /leads/{id}/activities/{activityId}/attachments` · `GET /leads/{id}/activities/{activityId}/attachments/{attachmentId}` (download)
- `POST /leads/{id}/custom-fields`
- `GET /lead-statuses`
- `GET /lead-scoring-rules` · `PUT /lead-scoring-rules`
- `GET /leads/duplicates` · `POST /leads/{id}/merge`
- `POST /imports/leads/analyze|validate|import` · `GET /imports/{id}` · `GET /imports/{id}/rows`
- `GET|POST /saved-views` · `PATCH|DELETE /saved-views/{id}` · `GET /pipelines` · `GET /workspaces/{id}/members` · `POST /flows/{id}/enroll`

## API only (no UI yet)
- `GET /leads/{id}/timeline` (the UI reads `/activities`).
- `GET /leads/{id}/custom-fields` (values are read from the lead response instead).
- Sorts `updated`, `last_contacted` and `cf:<key>`; bulk-convert options `template_amount` and per-lead `deal_overrides`.
- `PATCH /leads/{id}/status` with a status name and a `note`. The UI sends only the id.

## Not built (lives in the vision design)
- Board (kanban) view of leads by status.
- Disqualify flow with reasons (`DisqualifyLeadModal`) and the "Disqualified" status; the app's terminal-ish status is the configurable "Unqualified".
- Prototype status set New / Contacted / Nurturing / Sales-Ready / Disqualified (the app seeds New / Working / Nurturing / Qualified / Unqualified).
- Editable contact and company fields in the convert dialog; choosing the deal stage or win probability.
- Text (SMS) and Schedule header actions; "Email lead" and "Dismiss" in the recommended-action card.
- SLA badge, qualification card and badge, ad attribution card, local-time chip, related records (custom objects).
- Smart Lists, view favourites and folders; OR filters.
- Lead-scoring "Reset to defaults", base score, score decay, custom rules, renamable signals, live preview.
- Bulk Enroll in sequence (enrol is per lead from the lead page).

## Known gaps
- The Hot Score, This Week and No Owner views narrow only the current page of 25, not the whole list. The headline count is hidden for them (unless the page is the whole list), and a notice gives the page numbers instead. Export is disabled for them.
- Inline Status stays editable in the Converted view, and every change there fails with 409 `LEAD_STATUS_LOCKED`; it should be locked like the bulk Change status.
- Engagement and Behavior scoring weights are editable but have no effect until a producer writes those activity types.
- The lead timeline shows the lead's own events only. Note edit/delete is not offered for leads (the backend has no PATCH/DELETE for lead notes).
- Quick add → Bulk import shows Leads as "Coming soon" even though lead import works from the Leads page.
- Duplicate detection runs on Team but lead scoring does not. The backend comment calls both "Pro and Team features"; the catalog gives `lead_scoring` to Trial, Pro and Business only.
- Schedule, "Email lead" and "Dismiss" render as disabled controls in the app today; the trimmed design should drop them.
- The convert dialog's company-name lookup is case-insensitive find-or-create, so two leads naming the same company link to one company.
