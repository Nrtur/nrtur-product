---
area: Automations
status: built
verified_against: frontend@1bb6aa5d, backend@577bae44
verified_on: 2026-10-06
---
# Automations

## What it does
Automations are trigger-based workflows. When something happens to a record (a contact is created, a lead is assigned, a deal changes stage, a call is logged, and so on), the record is enrolled and walks a top-to-bottom flow of steps: actions, if/then conditions, split paths, waits, "wait until" and goals. The list lives in the **Engage** hub (`/engage?tab=automations`), and the builder is a full-page canvas at `/automations/new` and `/automations/{id}`. Engage also holds the sequence list (see `sequences.md`), Lead scoring rules and the workspace Deliverability guardrails (frequency cap, quiet hours, concurrent-flow limit, re-engagement rule, suppression list). Every outbound message that a flow sends passes a server-side gate chain: kill switch, suppression, the step's send time, quiet hours, frequency cap, provider circuit breaker and the daily workspace send budget, then (inside the send executor) the per-mailbox quota for email and the plan's monthly automated-send allowance. Owners and admins author flows. Members can view them and enroll records into sequences.

## Screens
| Route (app) | Prototype page id | Purpose |
|---|---|---|
| `/engage` (`?tab=` / `?sub=`) | `engage` | Engage hub. Its live tabs are Sequences (SMS, Email), Automations, Rules (Lead scoring only) and Deliverability (Frequency cap, Re-engagement, Suppression list). |
| `/engage?tab=automations` | `engage` → `settings-automations` (`SettingsAutomationsPage`) | Automation list: concept banner, folder rail, cards with an on/off switch (shown on drafts too, for owners and admins), Enrolled, **Logs**, Edit and a ··· menu (Duplicate, Delete), plus a header **Templates** button that opens the template browser. On mobile the secondary actions fold into the ··· menu, where Logs is labelled "Run history" (`AutomationCard.tsx`). |
| `/automations` | — | Redirects to `/engage?tab=automations`, so old links keep working. |
| `/automations/new`, `/automations/{id}` | `automation-builder` | The builder: trigger tile and drawer, step canvas, step picker, step settings drawer, flow summary sidebar, on-canvas test run, run history and the activate switch. `?recipe=` preloads a template. |
| `/engage?tab=delivery&sub=frequency` | `settings-frequency` (rendered inside `EngageDelivery`) | Frequency cap per channel (email and SMS), the transactional exemption, the max concurrent flows per contact, and quiet hours. |
| `/engage?tab=delivery&sub=reengagement` | `settings-reengagement` (rendered inside `EngageDelivery`) | The automatic sunsetting rule (days, then win-back or suppress). The cold-contact finder renders with no data. |
| `/engage?tab=rules&sub=scoring` | `engage` → `settings-scoring` | Lead scoring (owned by the lead-scoring area). |

## Behaviour and rules
- **Engage hub tabs** (`features/engage/components/engageTabs.ts`): Sequences, Automations, Rules and Deliverability are live. Funnels, Forms, Templates and Ad leads render as inert "coming soon" tabs. Inside Rules, Assignment and Validation are inert. Inside Sequences, Push and In-app are inert. A deep link to a disabled tab falls back to the first live tab, which is Sequences. No tab is plan-gated or role-gated today (`lib/engagePlanEntitlement.ts` maps nothing).
- **Triggers.** The backend catalog (`internal/modules/automations/triggers.go`) defines **64 triggers in 12 categories**. Only **9 are live**, meaning a producer writes an activity row that the dispatch tail matches (`dispatch/registry.go`):
  - `contact.created`: Contact created
  - `contact.updated`: Contact updated
  - `contact.status_changed`: Status changed
  - `lead.status_changed`: Lead status changed
  - `lead.assigned`: Lead assigned
  - `deal.stage_changed`: Deal stage changed
  - `deal.moved_to_stage`: Deal moved to <stage> (needs `trigger_config.stage_id`; `pipeline_id` is optional)
  - `sms.reply_received`: SMS reply received (inbound only)
  - `call.logged`: Call logged
- **Triggers in the UI.** The picker (`TriggerPicker.tsx`) shows all 64 entries from `GET /flows/trigger-catalog`, but only those 9 can be selected. The other 55 render as greyed "Coming soon" tiles. The server accepts a non-live key on a draft but refuses to **activate** it. `deal.moved_to_stage` is the only trigger with a parameter control (a stage picker). The trigger drawer also offers an optional **enrollment condition** ("Only enroll records matching a condition"), stored as `cond`.
- **The other 55 triggers are not wired**: Tag added/removed, Health score dropped, Birthday, Lead created/qualified/score reached/source is/converted, New ad lead received, all four Company triggers, Deal created/won/lost/value changed/inactive N days, the 3 Smart list triggers, Email received/opened/link clicked, No reply in N days, the 4 Task triggers, Meeting booked, Form submitted, the 7 Appointment triggers, Scheduled (cron), Date field reached, Webhook/Zapier, the 4 Behavioral triggers, the 6 Payments triggers and the 3 Custom object triggers.
- **There is no event bus.** A poller tails the `activities` table (`dispatch/tail.go`), and each trigger matches on activity type, channel, link entity type and metadata.
- **Actions** (`GET /flows/action-catalog`, `actions.go`). These work: Assign to rep, Create task, Flag for review, Update a field, Add tag, Remove tag, Call webhook, Create / update lead, Convert lead, Create company, Create deal, Send email, Send SMS, Enroll in SMS sequence and Enroll in email sequence.
- **Actions that do not work.** **Notify Slack** (`provider_not_configured:slack`, and no executor exists) and **Schedule meeting** (`not_implemented`) are in the catalog as unavailable. The app still lists them in the step picker with a "won't run" warning chip. The engine-internal `sequenceMessage` action is never offered.
- **Email actions need a connected mailbox.** The "Send email" and "Enroll in email sequence" tiles show "Needs a connected mailbox" when there is none, and the server refuses save or activate with `MAILBOX_REQUIRED`.
- **Logic and timing steps** (`nodes.go`, `lib/nodes.ts`): If / then branch (`condition`), Split into paths (`branch`, rule-based lanes with a default, up to 10 lanes), Wait / delay (`wait`), Wait until (`waitUntil`, a rule plus a timeout) and Goal / exit (`goal`).
  - **Unsupported steps.** `mapFields` is in the vocabulary but the server refuses it at save. A step like that loaded from the server renders as "Unsupported step".
  - **No split test.** There is no percentage or A/B split node (the contract says a future `split` kind would carry weights).
- **Waits** (`config/WaitConfig.tsx`, which uses `features/scheduling/ScheduleField`). Three tabs: **"As soon as it ends"** (wait N minutes, hours, days or weeks), **"At a time"** (wait N, then a time of day, with optional weekdays) and **"On a date"** (a calendar date and time).
  - **Date mode cannot carry a wait.** The Wait row is hidden on "On a date"; a fixed date has nothing to offset from.
  - **If the time has already passed** ("At a time"): wait for the next matching day, or send now instead.
  - **If the date has already passed** ("On a date"): send now, skip this step, or end the enrollment. Ending records the exit reason `calendar_date_passed`.
  - **Timezone** is the workspace's or a fixed IANA zone. A contact or sender timezone is reserved on the wire but returns 422.
  - **Schedule preview.** `POST /flows/timing/preview` exists in the backend, but the frontend does not call it (`schedulingService.previewAvailable = false`).
- **Conditions.** These 11 fields can be authored: Status, Owner, Tag, Deal value, Last contacted, Created date, Lead status, Lead source, Lead score, Company type and Company industry (`lib/conditions.ts`), each with a per-field operator list.
  - **Nested groups.** Nested AND/OR groups (up to depth 5 on the wire) render read-only and cannot be authored.
- **Tree limits** (`vocab.go`): max 200 nodes, depth 10, 10 lanes, condition depth 5 and 50 condition leaves, and 256 KB of steps JSON.
- **Statuses.** A flow is `draft`, `active`, `paused` or `error`.
  - **Activation** (`PATCH /automations/{id}/status`) re-validates the stored tree. A 422 response names the offending nodes, and the canvas marks those tiles.
  - **Saving** (`PATCH /automations/{id}`) replaces the step tree wholesale and creates a new version. Running enrollments stay on their pinned version.
  - **Error state.** An errored flow shows an Error badge with a readable `paused_reason`, for example after the circuit breaker trips or the plan's send allowance runs out.
- **Re-enrollment.** By default a record can enter a given flow **only once**: `flows.reenroll` defaults to false and no API field changes it (`service/enroll.go`). The resulting skip reason is `already_enrolled`.
- **Max concurrent flows.** This is a workspace guardrail (default 3, minimum 1). A record already in that many live flows is skipped with `max_concurrent_flows`.
- **Test run** (on the canvas, SCRUM-551). `POST /flows/{id}/test` runs the saved tree against one real record and writes nothing. The test bar plays back node by node. It is available only after the first save. Task-subject triggers cannot be tested because there is no task picker.
- **Run history.** The run log (`GET /flows/{id}/runs` and `/stats`) shows header stats, per-run rows and an expandable step trace with a snapshot of the record. Owners and admins can **retry from the failed step** (`POST /flows/{id}/enrollments/{enrollmentId}/retry`).
- **Enrolled modal and Engagement modal.** These are shared with sequences (`components/enrollment/`). The Enrolled modal shows four states (active, waiting, completed, exited) with filters. The Engagement modal shows the funnel, rates, clickers and recent events. Its "Top clicked links" block is disabled because nothing aggregates clicks per URL.
- **Templates.** The list header's "Templates" button (and "Browse templates" in the empty state) opens `GET /flows/template-catalog`, which has 4 templates: New Lead Welcome (`contact.created`), Speed-to-lead (`lead.assigned`), Status-change follow-up (`contact.status_changed`) and Deal stage triage (`deal.stage_changed`).
- **Folders.** Folders (`GET/POST/PATCH/DELETE /folders`, `PATCH /{object}/{id}/folder`, surface key `automations`) group the list. Deleting a folder unfiles its items.
- **List limits.** The list renders one page of up to 100 automations.
- **Duplicate and Delete.** Duplicate creates a paused "Copy of …". Delete is a soft delete, and the run log survives.
- **Gate chain on every send** (`runner/gates.go` `DefaultGates`, then the send executor), checked in this order:
  1. Kill switch (global `FLOW_ENGINE_ENABLED`, plus a per-workspace pause), which defers.
  2. Suppression, which skips the send and can end the enrollment.
  3. Send time (`SendTimeGate`), which holds a message step until the local send time set on the step.
  4. Quiet hours, which defers. When both apply, the later of send time and the quiet-hours window wins.
  5. Frequency cap per contact per channel, which defers.
  6. Provider circuit breaker, which defers until the next probe.
  7. The workspace send budget (`WorkspaceSendBudgetPerDay = 5000`, UTC day).
  8. Email only: the mailbox quota (250 a day and 10 a minute per connected inbox), reserved inside the email executor at send time (`service/executor_email.go`), which defers. The chain still has a `mailbox_quota` slot between the frequency cap and the breaker, but it is a no-op.
  9. Last, right before the provider call, the monthly **automated-send meter** draws one unit per email or text (`service/send_meter.go`). A refused draw fails the step with no auto-retry and flips an automation to `error` with `engine: plan_limit:automated_sends` (a sequence is not flipped; see `sequences.md`).
- **Guardrail defaults** (`service/guardrails.go`):
  - Email cap is 3 per 168 h and SMS cap is 2 per 168 h.
  - The transactional exemption is on but **not enforceable**, because no send has a class.
  - Quiet hours are on, 08:00–19:00 in the workspace timezone. The fields name the **sending** window. A window can wrap past midnight, and `start == end` returns 422.
  - Max concurrent flows is 3.
  - Re-engagement is off, with 90 days and the winback action.
  - The frequency cap counts flow sends only.
- **Plan limits** (`internal/platform/entitlements/catalog.go`):

  | Tier | Active automations | Automated sends / month | Automations can send |
  |---|---|---|---|
  | Trial | 50 | 0 | no |
  | Solo | 1 | 0 | no |
  | Team/Starter | 5 | 1,000 | yes |
  | Pro | 50 | **7,500** | yes |
  | Business | 200 | **35,000** | yes |

  - **Matches the ground rules.** Pro 7,500 and Business 35,000 match what the owner decided.
  - **How the frontend applies them** (`lib/planGates.ts`): it pre-empts `total_flows` on create, `active_automations` on activate (pausing is never gated), and `automation_sends` on save of a flow that has a send step.
  - **Enforced.** `entitlements.Boot()` turns on strict enforcement of every cap, meter and feature gate in both the API and the worker. Code comments that still mention `watch` mode are out of date.

## Permissions
- **Owner and admin** (`config:manage`; a legacy `manager` row counts as admin): create, edit, activate or pause, duplicate and delete automations; test run; retry a failed enrollment; create and assign folders; edit the Deliverability guardrails.
- **Member** (`crm:read`, `crm:write`): view the list, the builder (read-only, with a banner), run history, Enrolled and Engagement; read the guardrails (read-only banner). With `crm:write` a member can also enroll records into a flow (`POST /flows/{id}/enroll`) and, through the API, unenroll them.
- **Workspace pause** (`PUT /settings/flow-engine`): `config:manage`.

## API
- `GET /automations`
- `POST /automations`
- `PATCH /automations/{id}`
- `DELETE /automations/{id}`
- `PATCH /automations/{id}/status`
- `POST /automations/{id}/duplicate`
- `GET /flows/{id}`
- `GET /flows/trigger-catalog`
- `GET /flows/action-catalog`
- `GET /flows/template-catalog`
- `GET /flows/{id}/runs`
- `GET /flows/{id}/stats`
- `GET /flows/{id}/enrollments`
- `GET /flows/{id}/enrollments/{enrollmentId}`
- `GET /flows/enrollments` (a record's enrollments)
- `GET /flows/{id}/engagement`
- `GET /flows/{id}/engagement/events`
- `POST /flows/{id}/test`
- `POST /flows/{id}/enroll`
- `POST /flows/{id}/enrollments/{enrollmentId}/retry`
- `GET /folders`, `POST /folders`, `PATCH /folders/{id}`, `DELETE /folders/{id}`
- `GET /{object}/{id}/folder`, `PATCH /{object}/{id}/folder`
- `GET /settings/automation-guardrails`, `PUT /settings/automation-guardrails`
- `GET /settings/usage` (plan caps, features and meters for the pre-emptive gates)

## API only (no UI yet)
- `GET /flows/runs` and `GET /flows/runs.csv`: the cross-automation run log and its CSV export.
- `GET /flows/node-catalog`: node kinds with config schemas. The frontend keeps its own logic palette.
- `DELETE /flows/{id}/enrollments/{enrollmentId}`: unenroll one record.
- `GET /automations/{id}/kill-switch`: the per-flow kill-switch and breaker state. The list reads `paused_reason` instead.
- `GET /settings/flow-engine` and `PUT /settings/flow-engine`: the workspace-wide pause of the flow engine.
- `POST /flows/timing/preview`: server-resolved send-time preview.
- The 55 non-live triggers are in the catalog but cannot be activated.

## Not built (lives in the vision design)
- The Funnels, Forms, Templates (Email and SMS) and Ad leads tabs of the Engage hub; the Rules → Assignment and Validation sub-tabs.
- 55 of the 64 triggers, including all the time-based ones (cron, date reached, deal inactive, no reply, task due or overdue), the webhook trigger and its URL or sample capture, tag triggers and the per-trigger parameter questions.
- These palette steps: Notify a teammate, Add a note, Generate report, Remove from sequence, Send push, Send in-app, Notify Slack, Schedule meeting, Split test (percentage), Map webhook fields; the email/SMS template pickers on send steps.
- The "When this runs" per-flow execution window (Workspace hours, Custom or Always 24/7).
- The Logs sub-tab (cross-automation run log with filters and CSV export).
- In the Enrolled modal: per-enrollment Pause, Resume, Move to step, Remove and bulk selection.
- Engagement "Top clicked links".
- The "+1 day" simulated-clock controls and the client-side automation runtime.
- The 14-template prototype gallery (the app serves 4); the condition fields Health score, Has open deal and the engagement fields (Replied, Opened, Clicked, Meeting booked).
- Push and in-app frequency caps; the seeded cold-contact finder with win-back and sunset actions.

## Known gaps
- **The app shows triggers and actions that do not work, and the trimmed design does not.** The trigger picker still renders the 55 non-live triggers as "Coming soon". The step picker still offers Slack and Schedule meeting with a "won't run" chip. Either the app hides them or it keeps a disabled state the design no longer shows.
- **Re-engagement does nothing yet.** The rule is stored by `PUT /settings/automation-guardrails`, but the sweep that would act on it is not built (`service/guardrails.go`). The cold-contact finder has no data source and renders an explanatory empty state. The win-back picker lists automations only, and its copy says sequences arrive "once the sequence API ships", which is stale because the sequence API is live.
- **The transactional exemption toggle saves but has no effect.**
- **The Engage destination description is stale.** `components/layout/destinations.ts` still says "Sequences, automations, funnels, forms & templates".
- **The task action's time field is ignored.** The "At time" field shows a "not applied yet" hint; the executor always creates the task all-day.
- **The automated-send meter can over-count.** When an SMS is refused for an empty prepaid wallet, the unit is still drawn (`send_meter.go`).
- **Folder membership is read one item at a time**, which is slow for large lists.
- **Flow partitions have no retention policy.** `flow_events` and `flow_stats_daily` gain a partition every month and nothing removes them (backend README).
- **The "Deal moved to" hint says the stage is optional, but it is required.** The trigger drawer's hint reads "Leave unset to fire on every stage change" (`TriggerDrawer.tsx`), but the backend's trigger config marks `stage_id` required and refuses the flow with 422 without it (`triggers.go`). "Deal stage changed" is the trigger for every stage change.
- **The step picker shows raw catalog descriptions.** Catalog descriptions override the app's own copy (`lib/nodes.ts`, `entry.description ?? preset?.description`), so Create task reads "… Config: `mentions` — user ids of …" and the two enroll actions read "… Config: `sequenceId` — …" (`actions.go`).
- **Deliverability still shows unbuilt controls the trimmed design drops.** Suppression's Domains tab reads "Domain blocking is coming soon" (`SuppressionDomainsPanel.tsx`); Export CSV is disabled with "CSV export has no backend endpoint yet" (`SuppressionPanel.tsx`); "Global opt-out suppression" and "Honor GDPR deletion requests" carry a "Coming soon" pill (`ComplianceRulesCard.tsx`); and the Frequency cap's "contacts at their cap right now" count shows "—" because there are no per-contact counters (`FrequencyCapPanel.tsx`). The app should hide them or keep them clearly disabled.
- **Concurrency label mismatch.** The Frequency cap page labels the guardrail "Max simultaneous sequences per contact", but the backend's `max_concurrent_flows` counts automations and sequences together.
- **The Speed-to-lead template's task due time isn't a choice in the task step.** The recipe sets `due: "15 minutes"` (`templates.go`), but the task step's "Due in" options start at "1 hour" (`lib/actionFields.ts` `DUE_IN_OPTIONS`), so the select can't show the template's value. The design keeps the template's value as an extra option.
