---
area: Sequences
status: built
verified_against: frontend@1bb6aa5d, backend@577bae44
verified_on: 2026-10-06
---
# Sequences

## What it does
A sequence is a drip campaign on one channel, **SMS** or **Email**: an ordered list of timed messages, plus exit rules. A sequence has no trigger of its own. Records enter only by explicit enrollment: from a contact or lead page, in bulk from the contacts list, from a deal stage, or from an automation's "Enroll in SMS/email sequence" action. Under the hood a sequence is the same runtime object as an automation (`flow_kind = sequence`). The server compiles the flat step list into flow nodes, so every send passes the same gate chain, guardrails and plan meter described in `automations.md`. The list lives in the Engage hub (`/engage?tab=sequences&sub=sms|email`), and the builder is a full page at `/sequences/{sms|email}/new` and `/sequences/{sms|email}/{id}`.

## Screens
| Route (app) | Prototype page id | Purpose |
|---|---|---|
| `/engage?tab=sequences&sub=sms` / `&sub=email` | `engage` → `settings-sequences` (`SettingsSequencesPage`) | Sequence list per channel: three stat cards, a banner, folder rail, rows with an on/off switch, Enrolled, Engagement and Edit, and "New sequence". |
| (drawer on the list) | `NewSequenceModal` | Name plus an optional starter (3 per channel), then opens the builder. Nothing is created until Save. |
| `/sequences/sms/new`, `/sequences/sms/{id}` | `sms-sequence-builder` | SMS builder: from-number card, A2P send-gate banner, a spine (Trigger tile, then Message 1, Wait, Message 2 …), exit conditions, sending rules and a phone preview. |
| `/sequences/email/new`, `/sequences/email/{id}` | `email-sequence-builder` | Email builder: the same spine with subject and body steps and a schedule on each step. |
| (drawer on contact / lead / contacts list) | `SequenceEnrollModal` | "Enroll in sequence": pick a sequence, confirm recipients, see the server's enrolled and skipped report. |
| (modal on deal pipeline stage) | `StageAutomationModal` | Enroll every deal in a stage, in batches of up to 100 (`features/deals/components/StageSequenceModal.tsx`; owned by the pipeline area). |

`/sequences` and `/sequences/{channel}` have a layout but **no page** and return 404. The push and in-app builders 404 by design (`isListableChannel`).

## Behaviour and rules
- **Channels.** Only `sms` and `email` exist. Migration 0065 / D54 removed `push` and `inapp` from the flow channel vocabulary, so the API returns 422 for them on every route. The app still renders Push and In-app as inert tabs (`SequenceChannelTabs.tsx`, `engageTabs.ts`).
- **Statuses.** A sequence is `draft`, `active`, `paused` or `error`. In practice the runner never moves a sequence to `error`: the flow-level error transition (`SetFlowErrored`) is pinned to `flow_kind = 'automation'`, so a failing sequence step fails only that enrollment and the sequence stays `active`.
  - **Create** always writes `draft`. An update never changes status, so "Save" and "Save as draft" do the same thing.
  - **Activate or pause.** Activation (`PATCH /sequences/{id}/status`) recompiles and validates the stored steps. A flow in `error` returns 409 `FLOW_ERRORED` until it is paused (this is reachable for automations, not for sequences), and 503 means no compiler is wired.
  - **No delete or archive.** There is no DELETE or archive route for a sequence (`service/sequences.go`), and `DELETE /automations/{id}` matches automations only. A sequence can only be paused.
  - **No trigger check.** The trigger-liveness gate is deliberately skipped for sequences (D65). Their stamped trigger is `manual.enrolled`.
- **Steps** (`SequenceStep`: `subject?`, `body`, `delay`, `sendTime` or `at`/`tz`).
  - **Required text.** Email needs a subject and a body. SMS needs a body.
  - **Merge tags.** An unknown merge tag is rejected client-side (`schemas/sequenceStepSchema.ts`).
  - **SMS segments.** SMS shows a GSM-7/UCS-2 segment counter.
  - **Delays.** Step 1 cannot have a delay. Each later step takes a delay of N minutes, hours, days or weeks.
- **Scheduling.**
  - **Email** steps use the shared `ScheduleField` with the calendar tab turned off (`allowCalendar={false}`, `WaitDrawer.tsx`): wait N, then send **"As soon as it ends"** or **"At a time"**. Weekdays are an optional refinement inside "At a time", not a mode of their own, so "as soon as eligible on chosen weekdays" cannot be stored. "At a time" carries an **"If the time has already passed"** choice: wait for the next matching day, or send now instead. There is no calendar date for sequences. The timezone is the workspace's or a fixed zone; the contact's timezone is not supported.
  - **Legacy send times.** A legacy 12-hour `sendTime` is shown as an equivalent schedule and is resent unchanged unless the user edits it. A step never carries both `sendTime` and `at`.
  - **SMS** has a delay only, with no send-time control.
  - **Quiet hours** (workspace guardrail) still hold any send outside the sending window.
- **Exit rules** (`exit_rules`, `lib/exitRules.ts`).
  - **What the server accepts.** Seven authorable reasons: `goal_met`, `reply_keyword`, `link_clicked`, `became_customer`, `entered_sequence`, `deal_stage_reached` and `field_changed`.
  - **What the builder can build.** Only **"A field changes"** (`field_changed`, operator `is`) on Status, Owner, Tag, Lead status, Lead source, Company type or Company industry. Its exit actions can only be **Mark sequence complete** and **Add tag**, with up to 10 actions per rule.
  - **Exits that are not built.** The other 11 exit conditions and 6 exit actions can still be added in the UI. Each carries a "doesn't sync to the server yet" note, and the server never receives it.
  - **Rules from elsewhere are preserved.** A rule the editor cannot fully model is kept raw and re-sent exactly as it was. A PUT replaces the whole rule set in the server's own order.
- **Automatic exits, which are not rules** (`service/reply_exit.go`):
  - **Email reply.** An inbound email reply ends live enrollments with `replied`. Matching uses the provider thread id.
  - **Suppression.** Suppressed recipients (including unsubscribes) are skipped at the suppression gate and the enrollment can end with `suppressed`.
  - **Not detected yet.** `meeting_booked` and `bounced` exist in the vocabulary, but nothing in the backend writes them.
  - **No SMS reply exit.** Only an inbound **email** reply ends an enrollment. An SMS reply does not end an SMS sequence (the worker wires only the inbox observer, `service/reply.go`).
- **Enrollment** (`POST /flows/{id}/enroll`).
  - **Batch size.** Up to **100** distinct ids per request. More than that returns 422 and nothing is enrolled; the API never does a partial truncate.
  - **Server report.** The server returns enrolled and skipped counts with a reason per skipped record: `already_enrolled`, `condition_not_met`, `max_concurrent_flows`, `suppressed` or `no_contact_linked`.
  - **Once per sequence.** A record can enter a given sequence only once, ever, because re-enrollment is off by default and not configurable.
  - **Concurrency limit.** The workspace "max concurrent flows" guardrail (default 3) applies.
  - **Entry points.** Contact detail, lead detail, the contacts bulk toolbar (`EnrollInSequenceButton`), deal stage (`StageSequenceModal`, chunks of 100), and the `enrollSms` / `enrollEmail` automation actions.
- **Sending requirements.**
  - **Email** needs a connected mailbox. Save or activate is refused with `MAILBOX_REQUIRED` and the builder shows a dialog.
  - **SMS** needs an owned number and an approved A2P campaign. The builder shows a non-blocking "build now, send later" banner (`SmsSendGateBanner`) read from `/telephony/numbers` and `/telephony/status`.
- **List.** One page per channel. The stat cards are derived from row stats. Folders work as in automations (surface keys per channel).
  - **Row actions:** an on/off switch (optimistic, with rollback), Enrolled, Engagement and Edit.
  - **Enrolled and Engagement modals** are shared with automations. Opening the builder with an enrollment deep link focuses that enrollment.
- **Plan gates** (`lib/planGates.ts`, `internal/platform/entitlements/catalog.go`).
  - **Which plans include sequences.** The `sequences` feature is **off on Trial, Solo and Team/Starter** and on for Pro and Business, where active sequences are unlimited.
  - **What is checked.** Create checks `total_flows`. Activate checks `active_sequences` and the `sequences` feature.
  - **Plan-limit refusal.** Each send also draws the monthly automated-send meter (Pro 7,500, Business 35,000). When the plan refuses a draw, the step fails with `plan_limit:automated_sends` and is not auto-retried, so that enrollment ends `failed`. The sequence is **not** paused and does **not** go to `error`: unlike an automation (which flips to `error` with `engine: plan_limit:automated_sends`, see `automations.md`), a sequence stays `active` and every later enrollment that reaches a send step fails the same way. After upgrading or the cycle resetting, an owner or admin can retry each failed enrollment from the failed step.
  - **Enforced.** `entitlements.Boot()` turns on strict enforcement of every cap, meter and feature gate in both the API and the worker. Code comments that still mention `watch` mode are out of date.
- **Starters.** The New-sequence drawer seeds from a local list (`lib/starters.ts`). SMS has New Lead Welcome, Proposal Follow-up and Re-engagement. Email has Lead Nurture, Demo Follow-up and Onboarding. The backend's `GET /flows/sequence-template-catalog` is not used.

## Permissions
- **Owner and admin** (`config:manage`): create, edit, activate or pause sequences; manage folders. Retry from a failed step uses the same rule as automations.
- **Member** (`crm:write`): view the list (the on/off switch, folder button and Edit are hidden, `SequenceRow.tsx` `canManage &&`), so the read-only builder is reachable only by URL or an enrollment deep link; **enroll** records they can edit (`POST /flows/{id}/enroll`); unenroll through the API.
- **Plan gate first.** Create, update and status routes are wrapped in `RequireFeature(sequences)` before the permission check (`cmd/api/main.go`).

## API
- `GET /sequences?channel=sms|email`
- `GET /sequences/{id}`
- `POST /sequences`
- `PUT /sequences/{id}`
- `PATCH /sequences/{id}/status`
- `POST /flows/{id}/enroll`
- `GET /flows/{id}/enrollments`
- `GET /flows/{id}/enrollments/{enrollmentId}`
- `GET /flows/enrollments`
- `GET /flows/{id}/engagement`
- `GET /flows/{id}/engagement/events`
- `GET /flows/action-catalog` (channel send availability)
- `GET /contact-statuses`, `GET /pipelines`, `GET /tags` (exit-rule pickers)
- `GET /telephony/numbers`, `GET /telephony/status` (SMS send gate)
- `GET /folders` and the folder write routes (see `automations.md`)
- `POST /unsubscribe/{token}` (public one-click unsubscribe from email footers)

## API only (no UI yet)
- `GET /flows/sequence-template-catalog`: server-side starter templates. The app uses local starters.
- `POST /flows/{id}/test`: a dry run works for any flow kind, but the sequence builder's Test button is disabled.
- `DELETE /flows/{id}/enrollments/{enrollmentId}`: unenroll.
- `POST /flows/timing/preview`: send-time preview.
- Six of the seven authorable exit reasons (all except `field_changed`), and exit actions other than complete and `addTag`.

## Not built (lives in the vision design)
- Push and In-app sequences, their builders, phone and in-app mockups, simulated delivery logs and seeded analytics.
- The sequence "trigger" / "Start / Timing" node (contact added, deal stage, tag, smart list, form, date-based), entry timing, business-days-only, per-contact timezone, and the re-enrollment and enrollment-limit safeguards.
- A/B testing on a step (`SeqABPanel`) and the A/B results modal.
- The sequence Test run modal.
- Email template picker and "design in email builder" on a step (email templates are not built).
- Per-sequence sending rules (quiet hours, frequency cap per week, auto-suppress on hard bounce, link tracking) and a per-sequence from-number choice.
- 11 of the 12 exit conditions (replies, reply keyword, clicks, books a meeting, unsubscribes, becomes customer, enters another sequence, deal stage changes, goal met, manually removed, bounced) and 6 of the 8 exit actions (move to another sequence, change status, notify owner, add to suppression, create task, remove from all).
- In the Enrolled modal: Pause, Resume, Move to step, Remove (with a reason), and bulk actions; a sequence-level stop.
- The list-header "Enroll" button and the per-row "Enroll" button; the "+1 day" simulated clock; the client-side sequence engine (`seqTick` and related).

## Known gaps
- **The builder still shows controls that do nothing.** `SendingRulesCard` (quiet hours, frequency cap, bounce suppress, link tracking) and the SMS from-number picker save nothing. The from-number card's code says so in a comment, but the sending-rules card's labels read as if they apply. Unsaved exit conditions show a per-rule note. The trimmed design drops all three; the app should either hide them or label them clearly.
- **The disabled A/B panel and Test button are still in the app** (`SeqABPanel` in a disabled fieldset, `data-orphan="sequence-test-run"`).
- **The email step still shows a template-picker placeholder** (`data-be-pending="SCRUM-611"`).
- **A banner on the sequence list is stale.** It says the "Enroll in sequence" automation action "isn't wired up yet", but the backend now marks `enrollSms` and `enrollEmail` available and registers their executor (`SequencesWorkspace.tsx`).
- **`schedule_summary` is not on the wire**, so wait tiles show a plain summary built from the delay and schedule.
- **The automated-send meter can over-count SMS.** When the prepaid wallet is empty, the unit is still drawn (see `automations.md`).
- **`/sequences` and `/sequences/{channel}` return 404** instead of redirecting to the Engage tab.
- **A plan-limited sequence has no single stop.** When the automated-send allowance runs out, an automation flips to `error` once and notifies owners and admins. A sequence cannot enter `error` (`SetFlowErrored` is pinned to automations), so it stays `active`, each enrollment fails at its next send with `plan_limit:automated_sends`, no flow-errored notification fires, and each failed enrollment needs its own retry.
- **The list can render an Error status that sequences never reach.** `lib/status.ts` and `SequenceRow.tsx` map `error` and a "Last run failed" line, but the backend never sets `error` on a sequence.
