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

In the browser console, `window.__nrturGoTo('<page id>')` jumps to a page. `window.__nrturBillingState(...)` switches the billing page between its states (trial, active, read-only and so on).
