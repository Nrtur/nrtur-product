# Design

`index.html` is the whole prototype in one self-contained file (React, Babel and Tailwind from CDNs). It is published to **design.nrtur.io** on every push to `main` that touches this folder.

## Preview locally

```bash
python3 -m http.server 4317 --directory design
```

Then open http://localhost:4317/.

## Page ids

Screens are selected by a page id in `function App()` (`page==='…'`). Refer to screens by these ids in proposals and tickets, never by line number.

| Group | Page ids |
|---|---|
| Auth | `signup` `signin` `forgot-password` `verify-email` `reset-expired` `accept-invite` `unsubscribe` `onboarding-1` … `onboarding-4` |
| Core | `dashboard` `reports` (the activity feed) `tasks` `inbox` |
| Records | `contacts` `contact-detail` `add-contact` `leads` `lead-detail` `add-lead` `companies` `company-detail` `add-company` |
| Deals | `pipeline` `deal-detail` `add-deal` |
| Engage | `engage` `automation-builder` `sms-sequence-builder` `email-sequence-builder` |
| Settings | `profile` `settings-notifications` `settings-general` `settings-team` `settings-compliance` `settings-tasks` `settings-properties` `settings-tags` `settings-statuses` `settings-duplicates` `settings-integrations` `settings-phone-numbers` `settings-billing` |

In the browser console, `window.__nrturGoTo('<page id>')` jumps to a page. `window.__nrturBillingState(...)` switches the billing page between its states (trial, active, read-only and so on). `window.__nrturTemplates('empty')` empties the template library (`'seed'` restores it).

## Screens from proposals (not built yet)

The design also shows features that are **proposed** (open PR) or **approved but not built**. Look here before building from a screen:

| Feature | Status | Screens it adds or changes | Spec |
|---|---|---|---|
| Lead API | approved (not built) | `settings-integrations` (Lead API card), `leads` and `lead-detail` (Source "API") | [proposals/2026-10-lead-api.md](../proposals/2026-10-lead-api.md) |
| Billing page polish | approved (not built) | `settings-billing` (wallet history infinite scroll; Plan usage layout and subtitle) | [proposals/2026-10-billing-page-polish.md](../proposals/2026-10-billing-page-polish.md) |
| Email and SMS templates | approved (not built) | `engage` (new Templates tab; `email-templates` and `sms-templates` open it), `inbox` (Templates button in new email, reply/forward, SMS thread and New SMS), `email-sequence-builder` and `sms-sequence-builder` (step editor), `automation-builder` (Send email / Send SMS steps) | [proposals/2026-10-message-templates.md](../proposals/2026-10-message-templates.md) |
| Invite at seat purchase, plus billing copy | approved (not built) | `settings-billing` ("Users added" done state, "K open seats · Invite teammates", the post-checkout "Payment received" dialog, "phone numbers" wording, shorter Add credit dialog), `settings-team` (multi-person Invite dialog and deep link), `settings-phone-numbers` ("local number" wording) | [proposals/2026-10-invite-at-seat-purchase.md](../proposals/2026-10-invite-at-seat-purchase.md) |
| Duplicates: fast scan | approved (not built) | `settings-duplicates` (scan status line, scan states, paging, saved Not a duplicate and Restore, Merge all near-certain; the Duplicates preview toggle switches states); the same view in duplicates mode on `contacts`, `companies` and `leads` | [proposals/2026-10-duplicates-fast-scan.md](../proposals/2026-10-duplicates-fast-scan.md) |
| Live merge fields | proposed | `inbox` (template merge fields fill as the recipient is typed; fill panel with Use / Remove; "Merge fields filled for …") | [proposals/2026-10-live-merge-fields.md](../proposals/2026-10-live-merge-fields.md) |

When a feature ships, its row is removed here as part of the close-out.
