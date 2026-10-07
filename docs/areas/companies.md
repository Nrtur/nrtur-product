---
area: Companies
status: built
verified_against: frontend@1bb6aa5d, backend@577bae44
verified_on: 2026-10-06
---
# Companies

## What it does
Companies are the organisations that contacts, deals and (after conversion) leads belong to. The Companies page is a searchable, server-paginated table with a small rail of system views, a column chooser, CSV import and export, and an in-page duplicates mode. A side sheet creates or edits a company. The company page shows an editable properties panel with custom fields, stat tiles, the company's deals and people, tasks, and the company's own activity timeline with a note composer. Compared with contacts and leads, companies are the thinnest list: filters, saved views and bulk actions are not live yet.

## Screens
| Route (app) | Prototype page id | Purpose |
|---|---|---|
| `/companies` | `companies` | List: views rail, search, columns, import/export, duplicates mode |
| `/companies/[id]` | `company-detail` | Company record: properties, stat tiles, deals, people, tasks, timeline |
| (side sheet on `/companies`) | `add-company` | Create or edit one company (`CompanyFormSheet`) — no separate add page |
| (record sheet from Quick add → Multiple → Companies) | `add-company` (multiple) | Spreadsheet entry: name and domain per row |

## Behaviour and rules
**List**
- Page size 25; backend default 20, max 100 (a larger `page_size` falls back to 20 rather than being clamped).
- Search `q` matches name and domain (debounced 300 ms).
- Sort: Name only in the UI (`name_asc` / `name_desc`, default `recent`). The backend also accepts `oldest`.
- Columns: Company, Type, Industry, Contacts, Open deals, Owner (default); Annual revenue, Employees, Location, Website, Tags (optional). Type and Owner are editable inline. Contacts and Open deals always render empty (the API returns no counts).
- System views: All Companies, My Companies (`owner_id` = me), Archived. Customers, Prospects, With deals, Recently Active, Going Cold and No Owner render as disabled "Coming soon" rows (cut from the design). "Save view" is disabled: there are no user saved views for companies yet.
- Filters: the Filters popover builds conditions but they are never sent. It shows a "Preview — advanced filtering is coming soon" banner (SCRUM-1206). Treat filtering as not built.
- Bulk: selecting rows shows only "N selected" and Clear. Archive, Delete, Change type, tags and Assign owner are withheld (SCRUM-921; there is no companies bulk endpoint).
- Row actions (inline buttons, no menu): an Archive icon on each row (asks for confirmation); in the Archived view, a Restore button.
- Tools gear: Find & merge duplicates (in-page mode; `GET /companies/duplicates` uses name and phone signals; merge via `POST /companies/{id}/merge`; plan feature `duplicate_detection`, not on Solo), Manage properties (owner/admin), Manage statuses. "Manage tags" is disabled.

**Create / edit**
- UI rules (`schemas/companySchema.ts`): name required (≤ 120); website/domain ≤ 120 and must look like `company.com`; type is one of customer, prospect, partner, competitor, other (create defaults to prospect); employees whole number; location ≤ 120; owner; tags; custom fields. Industry is picked from a fixed list of eight (`COMPANY_INDUSTRIES`: Marketing & Advertising, Software / SaaS, Design / Creative, Consulting, Media, E-commerce, Finance, Healthcare), in both the sheet and the properties panel; the API accepts any text ≤ 100.
- Backend rules: name required (≤ 255). The domain is normalised to a bare host and must be unique per live company in the workspace (409 `COMPANY_DOMAIN_EXISTS`, shown inline with a link to the existing company). `company_type` is restricted to the same five values and stored lowercase. `description` is an alias for `notes`.
- The sheet's "Multiple records" toggle is disabled; multi-add exists only through Quick add.
- Plan cap: New Company, Import and the sheet's Create are disabled when the per-type companies cap blocks (trial or read_only at the cap, or any tier over it); paid tiers at exactly the cap only warn, and at 80% a warning shows. Editing and export are never gated. Caps per type: trial 2,500, Solo 5,000, Team 50,000, Pro 500,000, Business 2,000,000.

**Company page**
- Header: breadcrumb, record switcher (Previous/Next through the source list), Delete (confirm), New deal (deal sheet prefilled with this company), Schedule (a live button that only toasts "Scheduling is coming soon."), Edit.
- Hero: logo (favicon derived from the domain), domain link, type badge, and stat tiles Contacts / Open deals / Pipeline that jump to the panels (computed from the loaded lists).
- Properties (inline edit): name, website, industry, owner, employees, annual revenue, location, tags; secondary fields LinkedIn, description, billing address. Parent company and shipping address are read-only (no backend fields). Custom fields are editable.
- Deals panel: `GET /deals?company_id=` (first 6, "Show all", fetched up to 100).
- People panel: `GET /companies/{id}/contacts` (primary-first, fetched up to 100). "Add contact" opens the contact sheet and links the new contact to this company.
- Tasks (`RecordTasks`).
- Timeline: the company's own activities (`GET /companies/{id}/activities` + `/facets`). The composer offers Note (attachments, mentions, tags) and Task; notes can be edited or deleted by their author.
- Delete is a soft delete. Archive is offered only from the list row menu.

**Import / export**
- Import (`BulkImportWizard object="company"`): same wizard and limits as contacts (CSV, 10 MiB, 10,000 rows, 100 columns, async, one active import per workspace). Strategies Skip / Update / Create; dedup by domain, then name, and name-only stub companies are filled in.
- Export: `GET /companies/export` streams the current view (search, owner, archived) as CSV.

## Permissions
- Owner, admin and member can all create, edit, delete, archive, import, export and merge companies. There is no role check on company records in the backend.
- Every member sees every company.
- Owner/admin only: custom-field definitions (Manage properties).
- Notes: edit/delete by author only.

## API
- `GET /companies` · `POST /companies` · `GET /companies/{id}` · `PUT /companies/{id}` · `DELETE /companies/{id}`
- `POST /companies/{id}/archive` · `POST /companies/{id}/unarchive`
- `GET /companies/export`
- `GET /companies/{id}/contacts`
- `POST /companies/{id}/tags` · `DELETE /companies/{id}/tags/{tagId}`
- `GET|POST /companies/{id}/custom-fields`
- `GET /companies/{id}/activities` · `GET /companies/{id}/activities/facets` · `POST /companies/{id}/activities` · `PATCH|DELETE /companies/{id}/activities/{activityId}` · `POST /companies/{id}/activities/{activityId}/attachments` · `GET /companies/{id}/activities/{activityId}/attachments/{attachmentId}` (download)
- `GET /companies/duplicates` · `POST /companies/{id}/merge`
- `POST /imports/companies/analyze|validate|import` · `GET /imports/{id}` · `GET /imports/{id}/rows`
- `GET /deals?company_id=` · `GET /workspaces/{id}/members` · `/tasks`

## API only (no UI yet)
- List filters `industry`, `city` and `cf_filters`, and saved views with `object_type=companies` (the backend accepts them; UI work is SCRUM-1206).
- `GET /companies/{id}/timeline`: an aggregated feed (own events plus linked contacts' and deals' events). The UI uses only the company's own activities.
- Address fields (`address_line1/2`, `city`, `state`, `country`, `postal_code`) beyond the billing-address display.

## Not built (lives in the vision design)
- Working Filters (condition builder), Smart Lists, user saved views, favourites and folders.
- System views Customers, Prospects, With deals, Recently Active, Going Cold, No Owner.
- Bulk actions (assign owner, add/remove tag, change type, archive, delete).
- Company type "Vendor" (the app has Competitor and Other instead).
- Schedule (calendar event), Create invoice, Related records (custom objects).
- List-row counts for contacts and open deals.
- "Create a custom field" from the column chooser; "Manage tags" from the gear.
- Multiple-records mode inside the company sheet.

## Known gaps
- Company field changes are not written to the timeline (the backend company service writes only an audit-log row, unlike contacts).
- The company page shows "Created by —" (no creator column surfaced).
- The People and Deals panels fetch at most 100 rows. A company with more shows exact counts only up to that cap.
- UI name limit 120 vs backend 255; domain 120 vs 255.
- The Contacts and Open deals list columns are always empty.
- Schedule shows a "Scheduling is coming soon." toast rather than being hidden.
- Industry is a closed list of eight in the UI, while the API accepts any text ≤ 100. Values set by import or the API can fall outside the picker's options.
