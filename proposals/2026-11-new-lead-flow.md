---
title: New lead flow (automations, owner alerts, Lead API extras)
status: approved        # proposed | approved | in-progress | built | dropped
owner: Qamar Ul Islam
area: docs/areas/leads.md, docs/areas/automations.md, docs/areas/notifications.md, proposals/2026-10-lead-api.md, proposals/2026-10-facebook-lead-ads.md
screens: [automations, lead-detail, settings-integrations]
tickets: [Jira label `pre-launch`; keys to be added]
created: 2026-11-16
---

# New lead flow

## Problem
Leads arrive in four ways: the Lead API, Facebook Lead Ads, CSV import and manual create. After that, nothing happens:
- **No automation starts.** "Lead created" and "New ad lead received" are not live triggers, and creating a lead does not count as "Lead assigned", so the Speed-to-lead template (`lead.assigned`) never runs on a new lead.
- **Nobody is told.** The owner of a new API or manual lead gets no notification.

The Lead API and Facebook proposals both left this for "its own proposal". This is that proposal.

Customers who connect a web form also hit three Lead API limits:
- an unknown `source` is refused with 422;
- a retry can create the same lead twice;
- there is no way to send custom fields or UTM tracking.

## Proposal
1. **New leads start automations.** "Lead created" and "New ad lead received" become live triggers. A new lead with an owner also counts as **"Lead assigned"**, so the Speed-to-lead template works. **CSV imports never start automations.**
2. **The owner is told.** The owner of a new lead gets the existing "Lead assigned to you" notification.
3. **Lead API extras:** more `source` values with no 422 for unknown ones, an `Idempotency-Key` header, `custom_fields`, and UTM fields.

**Screens** (design.nrtur.io):
- `automations`: "Lead created" and "New ad lead received" are selectable triggers, not "Coming soon".
- `lead-detail`: a **Tracking** card with the UTM values, shown only when the lead has any.
- `settings-integrations` (Lead API docs drawer): the new fields, the `Idempotency-Key` header, and the new response.

## Rules

### Which leads start automations

| How the lead was created | Lead created | New ad lead received | Lead assigned | Owner notified |
|---|:-:|:-:|:-:|:-:|
| Manual (New lead, Quick add) | ✓ | — | ✓ if it has an owner | ✓ if the owner isn't the person who created it |
| Lead API | ✓ | — | ✓ | ✓ |
| Facebook Lead Ads | ✓ | ✓ | ✓ | ✓ |
| CSV import | — | — | — | — (the existing "Import finished" notification) |
| An automation's "Create / update lead" step | — | — | — | — |

- **CSV imports never start automations.** Importing 5,000 old leads must not send 5,000 Speed-to-lead emails. If someone wants to work imported leads, they enroll them in a sequence by hand, as today.
- **Leads created by an automation step don't start automations**, so one flow can't set off a chain of others.
- **A duplicate the API or Facebook refuses or merges** creates no new lead, so nothing fires. A Facebook submission merged into an existing lead (the "Allow duplicates" setting off) adds to that lead's timeline but does not count as a new lead.
- **"Lead created" and "Lead assigned" can both fire** for the same new lead. They are different triggers, so a workspace with one automation on each runs both. Each flow still enrolls a lead only once (`flows.reenroll` stays false).
- **Unassigned leads:** "Lead assigned" fires later, when someone is given the lead, as it does today.
- **Speed-to-lead template** (`lead.assigned`) now runs for manual, API and Facebook leads with no change to the template.

### Owner notification
- The owner gets the existing **"Lead assigned to you"** notification (`lead.assigned`). It links to `/leads/{id}`, uses the same channels and follows the same per-person notification settings.
- **Never notify yourself:** a manual lead you own gets no notification.
- **CSV import:** no per-lead notifications. The person importing gets the existing "Import finished" notification.

### Lead API: `source`
- **New values:** Google Ads, TikTok Ads, Website and **Other** join the list. They also appear in the in-app Source picker and filters, and in automation conditions.
- **Unknown values are no longer refused.** A `source` outside the list is saved as **Other**. The value that was sent is kept, and the timeline shows it: "Created via Lead API · source sent: Bing Ads". The response is still `201`, with a `warnings` entry: `{ "field": "source", "code": "UNKNOWN_SOURCE_SAVED_AS_OTHER" }`.
- Matching stays case-insensitive. A missing or blank source is still "API".

### Lead API: `Idempotency-Key`
- An optional header, up to 255 characters. The key is remembered for **24 hours**, per workspace.
- **Same key, same body:** returns the first response again (the same `201` and lead id). No second lead is created and nothing fires twice.
- **Same key, different body:** `409 IDEMPOTENCY_KEY_REUSED`.
- **Same key while the first request is still running:** `409 IDEMPOTENCY_IN_FLIGHT`, and the sender retries.
- Without the header, behaviour is unchanged.
- Error responses (4xx) are not remembered, so a fixed request can reuse the key.

### Lead API: custom fields
- An optional `custom_fields` object, keyed by the field's **key** as shown in Settings › Custom fields (Leads). For example: `"custom_fields": { "budget": "5000", "preferred_time": "Morning" }`.
- Values are checked against the field's type, the same rules as the app. A wrong type is `422 VALIDATION_ERROR`, with the field named in `details`.
- **Unknown keys are ignored, not refused**, so deleting a field doesn't break a live web form. They are listed in the `201` response's `warnings` (`UNKNOWN_CUSTOM_FIELD`).

### Lead API: UTM
- Optional fields: `utm_source`, `utm_medium`, `utm_campaign`, `utm_term`, `utm_content` (strings, up to 255 characters each).
- They are saved on the lead and shown on the lead page in a **Tracking** card, only when at least one is set.
- They are read-only in the app.
- Only the Lead API sets them. Facebook and manual leads don't have them.

### Docs drawer
- The in-app Lead API docs drawer lists the new fields, the header and the new responses, so a developer can wire a form without asking us.

## Permissions
No change.
- Who can create leads, set up automations and see notifications is the same as today.
- Lead API keys are still managed by owners and admins.

## Plan and pricing impact
- **Automations on new leads:** every plan that has automations, as today.
- **Lead API extras:** come with the Lead API (`api_access`: Trial, Pro, Business).
- No price or limit changes.

## Out of scope
- **Round-robin or automatic assignment** of new leads. "Assign to rep" is still an automation action. Rotation needs its own proposal (who is in it, skipping people who are away).
- An option to start automations from a CSV import.
- UTM capture for manual or Facebook leads, UTM filters, and carrying UTM over to the contact on conversion.
- Turning on any of the other "Coming soon" triggers (Lead qualified, Score reached, Source is, Converted, and so on).

## Decisions (owner, 2026-11-16)
1. CSV imports never start automations.
2. The new source values are Google Ads, TikTok Ads, Website and Other.
