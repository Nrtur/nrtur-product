---
area: Deals and pipeline
status: built
verified_against: frontend@1bb6aa5d, backend@577bae44
verified_on: 2026-10-06
---
# Deals and pipeline

## What it does
A deal is a revenue opportunity. Every deal sits in exactly one pipeline and one stage of that pipeline. The Pipeline page (`/pipeline`) is the main deals screen. It shows the active pipeline as a kanban board, a list, or a weighted forecast, with a KPI strip on top. A workspace can have several pipelines; the board title is a switcher between them. Owners and admins can create, rename, reorder, make default and delete pipelines, and add, rename, reorder, re-weight and delete stages, all from an "Edit stages" mode on the board. The deal page (`/deals/[id]`) shows a hero card (amount, win probability, stage, pipeline, time in stage, stage stepper), an About panel, custom fields, tasks and the deal's own activity timeline. A flat, paginated deals table also exists at `/deals`, but it is not in the rail; it is reached from global search "See all", the pipeline-delete refusal, and one settings link.

## Screens
| Route (app) | Prototype page id | Purpose |
|---|---|---|
| `/pipeline` | `pipeline` | Board / List / Forecast views of one pipeline, KPI strip, stage and pipeline management |
| `/deals/[id]` | `deal-detail` | Deal record: hero, About panel, custom fields, tasks, timeline |
| (side sheet `DealFormSheet`, opened from New Deal, a column's "Add deal", contact/company pages, Quick add) | `add-deal` (redirects to quick add) | Create one deal. There is no full-page add form |
| (side sheet `DealEditSheet`, from the deal page overflow and the `/deals` row) | — (prototype uses `RecordEditDrawer`) | Edit one deal |
| `/deals` | none | Flat deals table across all pipelines: search, filters, sort, archive scope, pagination |

## Behaviour and rules
**Pipelines** (`internal/modules/pipelines/service/pipelines_service.go`)
- Every workspace is seeded with one default pipeline, "Sales Pipeline": Prospecting 15% → Qualified 40% → Proposal 55% → Negotiation 70% → Won 100% (won) → Lost 0% (lost).
- Pipeline name: required, at most 120 characters, unique in the workspace.
- A new pipeline must have at least one stage, and at least one won stage and one lost stage. The create drawer seeds New (open, 10%), In progress (open, 50%), Won (won, 100%), Lost (lost, 0%). Each draft stage has a name, kind (open / won / lost) and probability.
- Exactly one pipeline is the default. The default is pinned first in the switcher; the rest follow `position`. Reorder is up/down buttons in Edit stages mode (`PipelineOrderControls`), not drag. "Make default" is a star button in Edit stages mode.
- Delete is a soft delete. It is refused with 409 `PIPELINE_IS_DEFAULT` (promote another first) or 409 `PIPELINE_HAS_DEALS`. Archived, won and lost deals count toward "has deals". The refusal dialog links to `/deals?archived=include` filtered to that pipeline, so the user can find and move the deals. Delete is disabled when only one pipeline exists.
- Plan cap on the number of pipelines (see `pricing/pricing.md`; stages per pipeline are unlimited on every plan). New pipeline is disabled when blocked, with a notice; the server still decides (422 `PLAN_LIMIT_REACHED`).

**Stages** (`stages_service.go`, `PipelineColumn.tsx`)
- Edit stages mode (owner/admin only): inline rename, probability slider, drag to reorder, delete, and an "Add stage" tile at the end (new stages are named "New stage", "New stage 2", …). No colours, no rot days, no required fields, no move rights.
- Stage name required and unique in the pipeline. Kind is open, won or lost (a stage cannot be both won and lost). Kind is only chosen when the pipeline is created.
- Reorder must send the full set of active stage ids, or the server returns 400.
- Delete archives the stage. It is refused with 409 `STAGE_HAS_DEALS`, shown inline ("move its deals first"). The UI also blocks deleting the only stage and the Won/Lost stages.
- Stage colours in the UI are derived (won green, lost red, others a stable hue); they are not stored.

**Deals** (`internal/modules/pipelines/service/service.go`)
- Name required, at most 255 characters on the server (the form allows 120). Amount is a non-negative decimal; currency is a 3-letter ISO code, default USD (there is no currency picker). Close date is `YYYY-MM-DD`.
- On create, `pipeline_id` / `stage_id` default to the workspace default pipeline and its first stage; owner defaults to the creator. The board's New Deal and column "Add deal" create in the pipeline on screen (and that column's stage). Other entry points use the default pipeline.
- Probability: Won is always 100 and Lost always 0. Otherwise a manual `probability_override` wins. Otherwise the stage default applies. The create and edit sheets auto-fill probability from the chosen stage until the user types their own.
- A stage change re-derives probability, keeps any override, stamps `closed_at` when entering won/lost, clears it when leaving, and writes a `deal_stage_change` activity.
- A stage must belong to the deal's pipeline (`ErrStageWrongPipeline`); archived stages are rejected.
- Next action is one of 9 values: Call, Email / follow-up, Send proposal, Run demo, Negotiate, Get signature, Internal review, Waiting on them, Onboard.
- Tags: attach/detach shared workspace tags (`POST`/`DELETE /deals/{id}/tags`).
- Delete is a soft delete. Deal archive/unarchive exists on the API but has no button (see API only).
- Plan cap on deal records: New Deal and Create deal are gated through `useCapGate("deals")`; editing is never gated.

**Pipeline board** (`PipelineBoard.tsx`)
- Loads one page of up to 100 deals for the active pipeline. When there are more, a banner says "Showing first 100 of N deals". There is no per-column "show more" and no Won/Lost collapse.
- KPI strip (from `GET /deals/forecast` for this pipeline and owner filter): Pipeline (open value, "N open"), Won ("N won"), Avg Deal, Win Rate ("NW · NL"), Forecast (weighted). The deltas are counts, not trends. When a stage, custom-field or client-side filter is active the strip says "whole pipeline, not this filter".
- Search matches deal **name** only. An empty result offers "Search company names instead".
- Saved views (`SavedViewMenu`, `GET/POST/PATCH/DELETE /saved-views`, object `deals`): system views All deals and My deals; user views are personal or shared and are kept per pipeline.
- Filters (`DealsFiltersMenu`): Stage and Owner (sent to the server, equality only), custom fields (sent as `cf_filters`), and Deal name, Company, Amount, Close date, Tags (applied only to the loaded deals; the page says so). Match is AND only; the OR toggle is shown disabled. Pipeline is a filter on `/deals` only.
- "…" View options: Sort within stages (Deal name, Amount, Owner, Close date, Created date; click again to flip direction; Clear sort), and Import deals (shared import wizard, object `deal`).
- Board: drag a card to another column to move it (optimistic, rolled back on error). Each card also has a keyboard "Move to…" select. Cards show name, company, amount, an "Overdue" flag when an open deal's close date has passed (otherwise an age dot and last-updated time), and the owner.
- Column header: stage name, count, total value, probability; a "⋯" menu with "Enroll stage in sequence…" (bounded audience, consent-checked), "Stage automation…" (an info sheet that links to Automations; no per-stage rule list), and "Edit stages" (owner/admin). "Select all in stage" is shown disabled with a "Soon" badge; "Email everyone in stage" is not rendered (SCRUM-921).
- Small screens get a stage chip strip instead of side-by-side columns.
- List view: fixed columns Opportunity, Company / Contact, Stage, Amount, Close Date, Owner, Age. Row click opens the deal. No selection, no bulk actions, no column chooser, no inline edits (there is no deal bulk endpoint).
- Forecast view: "By stage" (open deals, value and weighted value per stage, deal probability overrides stage probability, plus a "Closed won" line) and "By close date" (Overdue, this month and the next three months, Later, No close date). The headline figure is the server forecast; the breakdown covers only the loaded deals and says so when narrowed.

**Deal page** (`DealDetailView.tsx`)
- Header: "← Pipeline" breadcrumb, "Move to {next stage}" button, a disabled "Schedule · Coming soon" button, and a "⋯" menu with Edit deal and Delete deal (confirm). No record switcher.
- Hero: name and company, win probability (green ≥ 70, amber ≥ 40), amount, stage pill (Won/Lost), pipeline chip, "In {stage} for N days" (from the newest stage-change activity, falling back to created date; hidden for closed deals), clickable stage stepper.
- Pipeline chip (`DealPipelineMover`): "Move to pipeline" sends one `PUT` with the new `pipeline_id` and the target's first non-lost stage. Moving a won/lost deal warns that it reopens it. A static chip when there is only one pipeline. There is no multi-pipeline enrollment.
- About panel: Name, Amount, Expected close date, Owner (read-only here, edited via Edit deal); Primary contact and Company (click to edit with a searchable picker, clearable, with an open-record link); Next action (select); Tags (picker); Status (derived Open / Won / Lost). Pipeline, stage and probability are in the hero only.
- Custom fields card (`GET`/`POST /deals/{id}/custom-fields`).
- Tasks card (`RecordTasks`, shared with tasks).
- Timeline: the deal's own events only (`GET /deals/{id}/activities`), tabs All · Notes · Tasks · Deals with counts. Composer: Note (with attachments) and Task (opens the task drawer pre-linked to the deal). Rows expand to show details.
- Edit deal sheet: name, amount, stage, probability, close date, owner, contact, company.

**`/deals` list** (`DealsWorkspace.tsx`)
- Columns: Deal, Stage, Amount, Probability, Owner, Close Date (overdue in red), row actions (Edit). Sortable by Deal, Amount, Close Date only. 25 per page.
- Toolbar: search, the same Filters menu (plus Pipeline), archive scope (Active deals / Active + archived / Archived only), New Deal. This is the only place an archived deal can be seen. No import, no My Deals pill.
- All list state (search, filters, sort, page, archive scope) lives in the URL, so views are shareable.

## Permissions
- Owner, admin and member can all create, edit, move, delete and tag deals, add notes, and use the board, list, forecast and `/deals` (all roles hold `crm:write`).
- Owner and admin only: create, rename, reorder, make default and delete pipelines; create, rename, re-weight, reorder and delete stages. The service returns 403 to members; the UI hides "Edit stages" and "New pipeline" from them.
- Everyone sees every deal and every pipeline (there is no per-pipeline access).
- Saved views: anyone can create personal or shared views. A personal view can be deleted only by its creator; a shared view by its creator or an owner/admin.

## API
- `GET /deals` · `POST /deals` · `GET /deals/{id}` · `PUT /deals/{id}` · `DELETE /deals/{id}`
- `POST /deals/{id}/stage` (board drag)
- `GET /deals/forecast`
- `POST /deals/{id}/tags` · `DELETE /deals/{id}/tags/{tagId}`
- `GET /deals/{id}/custom-fields` · `POST /deals/{id}/custom-fields`
- `GET /deals/{id}/activities` · `POST /deals/{id}/activities` · `GET /deals/{id}/activities/facets`
- `POST /deals/{id}/activities/{activityId}/attachments` · `GET …/attachments/{attachmentId}`
- `GET /pipelines` · `POST /pipelines` · `PUT /pipelines/{pipeline_id}` · `PUT /pipelines/{pipeline_id}/default` · `DELETE /pipelines/{pipeline_id}`
- `POST /pipelines/{pipeline_id}/stages` · `PUT /pipelines/{pipeline_id}/stages/{stage_id}` · `PUT /pipelines/{pipeline_id}/stages/reorder` · `DELETE /pipelines/{pipeline_id}/stages/{stage_id}`
- `GET /saved-views` · `POST /saved-views` · `PATCH /saved-views/{id}` · `DELETE /saved-views/{id}` (object `deals`)
- `POST /imports/deals/analyze` · `/validate` · `/import` (via the shared import wizard)
- Stage "Enroll stage in sequence…": `POST /flows/{id}/enroll` once per contact (sequence list from the sequences feature); see the sequences/automations area docs.

## API only (no UI yet)
- `POST /deals/{id}/archive` and `POST /deals/{id}/unarchive`: no archive or restore button anywhere. Archived deals can only be listed on `/deals`.
- `GET /deals/{id}/timeline`: aggregated timeline (own events plus linked contact/company events). The deal page still reads own events only.
- `GET /pipelines/{pipeline_id}/stages` (with `?archived=true`): stages ship embedded in `GET /pipelines`; archived stages are never shown.
- Stage `kind` change via `PUT /pipelines/{id}/stages/{stage_id}`: the UI sets kind only when creating a pipeline.
- `probability_override` / `is_overridden`: the edit sheet writes it, but nothing shows that a deal's probability is pinned.
- `GET /deals` `company_id` / `contact_id` filters are used by contact and company pages only.

## Not built (lives in the vision design)
- Multi-pipeline enrollment: Primary/Enrolled placements, ‹ › pipeline paging, Set as primary, Enroll in another pipeline, Remove from pipeline.
- Connected pipelines (auto-handoff rules) and the Customer Onboarding demo pipeline.
- Per-pipeline access by role, per-stage move rights by role.
- Win and loss reasons (outcome modal on close, reason line in the hero, reason editor, junk-loss exclusion from win rate); Reopen deal.
- Stage colours, rot/stale days, required fields to enter a stage (stage gate), blueprints, approval and validation gates on moves.
- Duplicate pipeline; insert a stage between two stages; custom-object pipelines (`ObjectBoard`, object picker in New pipeline).
- Board: card-field chooser, Won/Lost column collapse, 25-per-column "Show more", Select all in stage, deal bulk bar, merge deals, Email everyone in stage, per-stage automation rule list with retroactive run.
- List view: column chooser, frozen first column, row selection, inline stage/owner edits.
- Seeded shared deal views (Closing this month, Won, Lost, Closed) and view favourites.
- Deal page: record switcher, Schedule meeting, Create invoice, Create quote, ad attribution card, related-records panel, created-by/last-activity footer, Source / Amount type / Forecast category / Additional contacts fields, timeline roll-up from linked records.

## Known gaps
- The Statuses settings card "Deal stages → Open Pipeline" links to `/deals`, not `/pipeline` (`features/status/components/StatusesSettingsPage.tsx`).
- `/deals` has no prototype page; the design has nothing to show for it.
- The deal form caps the name at 120 characters; the server allows 255. Imported or API-created deals can have longer names.
- The deal "Delete" copy says the timeline goes with it; the backend soft-deletes, and there is no recycle bin UI to restore from.
- Board and forecast breakdowns only cover the first 100 deals loaded; filters on Deal name, Company, Amount, Close date and Tags run in the browser over those 100.
- Filters: OR, "is none of" / "is empty" on core fields, and Heat / Source fields are not supported by the API (SCRUM-850).
- The KPI strip ignores stage, custom-field and client-side filters (only pipeline and owner are sent to the forecast).
- "Select all in stage" is rendered disabled with "Soon" in the stage menu; it should be removed rather than shown.
- The "Stage automation…" sheet is a placeholder that points to Automations.
- The stage-menu deal count is scoped by owner only; under any other filter it reports the whole stage and the audience actions stand down.
- The backend pipeline/stage role check accepts only `owner` and `admin`; a legacy `manager` row (admin-equivalent elsewhere) gets 403.
- `features/deals/README.md` lags the code in places (for example it says Next action and Tags are not rendered on the deal page; they are).
