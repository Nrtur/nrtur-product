---
area: Contacts
status: built
verified_against: frontend@1bb6aa5d, backend@577bae44
verified_on: 2026-10-06
---
# Contacts

## What it does
Contacts are the people a workspace sells to. The Contacts page is a searchable, sortable, server-paginated table with a rail of system views and user saved views (personal or shared), a condition-builder filter, a column chooser, and a small bulk bar. A side sheet creates or edits one contact; the "Add Multiple Contacts" menu item opens a spreadsheet-style record sheet; CSV import and export run from the top bar. The contact page shows an editable About panel (with custom fields), communication preferences (do-not-contact), linked deals, sequence enrolments, tasks, and an aggregated activity timeline with a note composer. Duplicate finding and merging is a mode inside the list, reached from the tools gear.

## Screens
| Route (app) | Prototype page id | Purpose |
|---|---|---|
| `/contacts` | `contacts` | List: views rail, search, filters, columns, bulk bar, import/export, duplicates mode |
| `/contacts/[id]` | `contact-detail` | Contact record: About panel, comms preferences, deals, sequences, tasks, timeline |
| (side sheet on `/contacts`) | `add-contact` | Create or edit one contact (`ContactFormSheet`) — the app has no separate add page |
| (record sheet from the "New Contact" split menu) | `add-contact` (multiple mode) | Spreadsheet entry of several contacts (`features/quick-add/components/RecordSheet.tsx`) |

## Behaviour and rules
**List**
- Page size 25 (`useContactsListState.ts`); backend default 20, max 100.
- Search `q` is a case-insensitive substring over first/last name, email, phone and linked company names; debounced 300 ms in the UI.
- Sort: only Name (asc/desc) and Last activity are sortable in the UI. They map to `name_asc` / `name_desc` / `last_contacted`; everything else falls back to `recent` (`contactsService.ts` `contactSortToken`). The backend also accepts `oldest` and `cf:<key>:asc|desc`.
- Columns: Name, Company, Status, Last activity, Owner (default); Email, Phone, Job title, Source, Tags (optional). Status and Owner are editable inline. The column choice is remembered per workspace and user in this browser, not on the account.
- System views: All Contacts, My Contacts (`owner_id` = me), New This Week (`created_from` = local midnight 6 days ago), Customers (status named "Customer"), Archived (`archived=only`). Uncontacted, Needs Follow-up, Hot Prospects and No Owner are rendered as disabled "Coming soon" rows; they are cut from the design.
- Saved views (`/saved-views`, `object_type=contacts`): save the current list state with a name and scope (Personal or Shared); rename or move scope (gear), delete, and "Save changes" when the list has drifted from the active view. The stored filter is the same query string the URL carries. No favourites, no folders, no Smart Lists.
- Filters: one condition builder (`ContactsFiltersMenu`). Match mode is locked to AND. Core fields: Status (`status_id`), Owner (`owner_id`), Tags (`tag_id`; only one tag is actually sent), Created date (`created_from`/`created_to`). Custom fields go to `cf_filters` (at most 10 clauses and 4096 bytes; over the cap the list stops loading and explains why). Operators the server cannot express are shown disabled.
- Bulk bar: Add tag, Remove tag (real `POST /contacts/tags/bulk-attach|bulk-detach`, at most 500 contacts and 50 tags per call), Add to sequence, Clear. Assign owner, Set company, Change status, Archive/Restore and Delete are deliberately withheld (SCRUM-921: there is no contacts bulk endpoint, and fanning out trips the 120 req/min per-user limit).
- Row menu: View details, Archive (asks for confirmation), Restore (in the Archived view), Delete (asks for confirmation).
- Tools gear (`ObjectToolsMenu`): Find & merge duplicates (in-page mode), Manage properties (Settings → Custom fields, owner/admin), Manage statuses (Settings → Statuses). "Manage tags" is always disabled.
- Duplicates mode: pairs from `GET /contacts/duplicates` (name signal; a cached per-workspace scan; 504 if it takes more than 20 s), with a merge drawer (`POST /contacts/{id}/merge`). "Not a duplicate" lasts only for the session. Bulk merge is withheld. This is a plan feature (`duplicate_detection`: not on Solo).

**Create / edit (side sheet)**
- UI rules (`schemas/contactSchema.ts`): first name and last name required (≤ 50), email required and must be a valid email, job title ≤ 100, company ≤ 100, source ≤ 50, permissive phone check.
- Backend rules: at least one of first name, last name, email or phone; first/last ≤ 100, job title ≤ 150, email ≤ 255, phone ≤ 20, source ≤ 100 (`contacts/service/widths.go`). Phone must be E.164 (`+…`), or the API returns 422 `PATTERN`. Email is unique per workspace (case-insensitive) and so is the E.164 phone; a clash is 409 `DUPLICATE_CONTACT`.
- Owner defaults to the creator; status defaults to the workspace default status. Seeded statuses are Lead (default), Prospect, Customer and Lost, and are configurable in Settings → Statuses.
- Create is a chain: required custom fields are checked first, then `POST /contacts` → tag sync → `POST /contacts/{id}/companies` (primary) → custom-field values. A retry reuses the contact already created.
- No duplicate check on create in the UI; the backend unique email/phone index is the only guard.
- The sheet's "Single / Multiple" toggle has Multiple disabled. Multi-add is the separate record sheet: first name, last name, email (all required) and strict E.164 phone; rows are created 3 at a time; there are no status, owner, tag or company columns.
- Plan cap: New Contact, the split menu, Import and the sheet's Create are disabled when the per-type contacts cap blocks (trial or read_only at the cap, or any tier over it); at 80% a warning shows. Editing is never gated. Caps per type: trial 2,500, Solo 5,000, Team 50,000, Pro 500,000, Business 2,000,000.

**Contact page**
- Header: breadcrumb, record switcher (position counter and Previous/Next through the list the contact was opened from; absent on a deep link), Enroll in sequence, ⋯ menu with Edit contact and Delete contact.
- Hero: avatar, name, title · company, editable status, Do-not-email/SMS/call pills, and four quick actions: Send Email (a plain `mailto:` link), Make call (in-app softphone; needs calling set up and a dialable number), Add Note (focuses the composer), Send SMS (SMS composer). Each is disabled by the matching suppression row. "Log manually" opens the log-call modal and is not blocked by do-not-contact.
- A do-not-contact banner shows when a `channel: all` suppression exists.
- About panel (`RecordProperties`): inline edit (commit on blur/Enter, Escape cancels) of name, email, phone, job title, company (re-links via the association endpoints), owner, status, tags, source. Additional emails/phones, mailing address and LinkedIn are read-only (no backend columns). Custom fields are editable below it (values come embedded on the detail response).
- "Converted from lead" card when the contact came from a lead conversion.
- Communication preferences: do-not-contact master switch plus per-channel suppression toggles (`/suppressions`); editable by owner/admin only.
- Deals card (`GET /deals?contact_id=`), Sequences card (hidden unless enrolled), Tasks (`RecordTasks`).
- Timeline: `GET /contacts/{id}/timeline` merges the contact's own events with events from its linked deals, companies and originating lead, each tagged with its source. It is capped at the 200 most recent, with no pagination. Filter tabs (All, Emails, Calls, Notes, Tasks, Deals, Changes) filter client-side.
- Composer: Note (with @mentions, #tags and file attachments up to 25 MB each, within the plan's attachment storage) and a Task launcher. Notes can be edited or deleted by their author only.
- Auto-recorded contact events: created, field changes, status_change, archived, unarchived, deleted; plus calls, SMS and email rows written by telephony and inbox.
- Delete is a soft delete (it frees a cap slot); archive keeps the record and its count. There is no restore after delete.

**Import / export (from the Contacts page)**
- Import wizard (`BulkImportWizard object="contact"`): Upload → Map → Review → Done. CSV only, ≤ 10 MiB, ≤ 10,000 rows, ≤ 100 columns. The write runs async and is polled. One active import per workspace. Duplicate strategy: Skip (default), Update existing, or Create anyway; dedup key is email, then E.164 phone. Extra mappable columns: owner email, status, tags, custom fields. Rows refused by the plan cap are reported with the server's message, and a page notice persists until dismissed.
- Export: "Export" in the top bar downloads the current view as CSV (`GET /contacts/export`), with the same filters minus pagination, streamed, with a BOM and formula-injection guards. It is never plan-gated.

## Permissions
- Owner, admin and member can all create, edit, delete, archive, tag, bulk-tag, import, export, merge duplicates and add notes. The backend has no role check on contact records (`authz` `crm:read`/`crm:write` are held by every role).
- Every member sees every contact in the workspace (no owner-scoped visibility).
- Owner/admin only (`config:manage`): contact-status CRUD and custom-field definitions.
- Saved views: anyone can create personal or shared views. Personal views are editable/deletable by their creator only; shared views by their creator or an owner/admin.
- Notes: edit/delete by author only (no admin override).
- UI-only gate: communication-preference toggles are editable by owner/admin only in the UI.
- Legacy role `manager` is treated as admin by the backend.

## API
- `GET /contacts` · `POST /contacts` · `GET /contacts/{id}` · `PUT /contacts/{id}` · `DELETE /contacts/{id}`
- `POST /contacts/{id}/archive` · `POST /contacts/{id}/unarchive`
- `GET /contacts/export`
- `POST /contacts/{id}/companies` · `DELETE /contacts/{id}/companies/{companyId}`
- `POST /contacts/{id}/tags` · `DELETE /contacts/{id}/tags/{tagId}` · `POST /contacts/tags/bulk-attach` · `POST /contacts/tags/bulk-detach` · `GET /tags` · `POST /tags`
- `GET /contacts/{id}/timeline` · `GET /contacts/{id}/activities` · `POST /contacts/{id}/activities` · `PATCH|DELETE /contacts/{id}/activities/{activityId}` · `POST /contacts/{id}/activities/{activityId}/attachments`
- `POST /contacts/{id}/custom-fields` · `GET /custom-fields/definitions`
- `GET /contact-statuses`
- `GET /contacts/duplicates` · `POST /contacts/{id}/merge`
- `POST /imports/contacts/analyze|validate|import` · `GET /imports/{id}` · `GET /imports/{id}/rows`
- `GET|POST /saved-views` · `PATCH|DELETE /saved-views/{id}`
- `GET /workspaces/{id}/members` (owner pickers) · `/suppressions` · `POST /flows/{id}/enroll` · `GET /deals?contact_id=` · `/tasks`

## API only (no UI yet)
- List filters `updated_from`/`updated_to`, sort `oldest` and `cf:<key>` sorts.
- `GET /contacts/{id}/activities/facets` (a wrapper exists; nothing calls it).
- `GET /contacts/{id}/companies` (list all linked companies; the UI shows only the primary).
- Activity types `email` and `call` on `POST /contacts/{id}/activities` (the composer offers only Note and Task).
- Rich activity `payload` (email thread, call transcript, talk-time, AI summary, meeting guests): the schema exists and the UI can render it, but nothing in the backend writes it.

## Not built (lives in the vision design)
- Smart Lists (condition-built auto-updating segments), view favourites and view folders.
- System views Uncontacted, Needs Follow-up, Hot Prospects, No Owner, and the sidebar view shortcuts for them.
- OR / ALL-ANY matching in filters, and filters on name, email, phone, job title, source or deal value.
- Bulk Assign owner, Set company, Change status, Archive, Delete, and the type-DELETE confirm.
- Duplicate check on create (`DuplicateDetectionModal`), and the "Not a duplicate" persistence.
- Star contact, Schedule (calendar event), Create invoice, in-app email compose from the record, call dialer popover with status update.
- Local-time chip / "Outside hours", contact frequency indicator, ad attribution card, related records (custom objects).
- AI "Recommended next action" content (the app shows an empty card).
- Timeline Email/Call composer tabs, and rich call/meeting payloads (recording, transcript, talk ratio, AI summary).
- "Create a custom field" from the column chooser; "Manage tags" from the gear.
- Contact score.

## Known gaps
- UI and API validation differ: the UI requires first name, last name and email and caps names at 50; the API needs only one identity field and allows 100. The UI phone check is permissive while the API requires E.164, so a non-E.164 phone fails on save. The API does not validate email format for contacts (leads do).
- The create sheet still sends the legacy `status: "lead"` alongside `status_id`.
- Export ignores the New This Week date filter (`GET /contacts/export` has no `created_from`/`created_to`); the success toast says so.
- The Tags filter sends only the first selected tag.
- Timeline is capped at 200 rows, with no total or "load more"; counts may undercount on very busy contacts.
- "Created by" in the footer is always "—" (no `created_by` column for contacts).
- The About panel shows a Score field that is never populated.
- The sheet subtitle promises "⌘↵ to save & add another", but no key handler exists.
- The "Recommended next action" card is a static "No recommendations yet." placeholder.
- The Source picker in the About panel offers Website/Referral/LinkedIn/Cold outreach/Event/Other; the API accepts any free text.
