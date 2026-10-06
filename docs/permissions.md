---
status: built
verified_against: frontend@1bb6aa5d, backend@577bae44
verified_on: 2026-10-06
---
# Permissions

nrtur has exactly **three roles**:
- **Owner:** one per workspace, the person who signed up.
- **Admin:** set up the workspace.
- **Member:** do the work.

Roles are enforced in the backend by `internal/platform/authz/policy.go` plus checks in each service. A legacy `manager` role still exists in old data and cannot be assigned. It holds admin's *capabilities* (configuration, labels, phone numbers, automations and sequences), but service checks that test for owner/admin literally refuse it: team management (invite, resend, revoke, role change, remove), General settings edits, mailbox disconnect, and pipelines/stages.

There is **no record-level visibility**: every member sees every contact, lead, company, deal, task, conversation and call in the workspace. Record scope (own / team / all), field redaction and "view as role" are vision-only.

## The model in one line

**Members do the work, admins set up the workspace, and the owner holds the money and the workspace itself.**

## Matrix

| Action | Owner | Admin | Member |
|---|:-:|:-:|:-:|
| **Records** (contacts, leads, companies, deals, tasks): view, create, edit, delete, import, export, merge, bulk edit, convert | ✓ | ✓ | ✓ |
| Notes: edit or delete | author only | author only | author only |
| Saved views: personal | creator | creator | creator |
| Saved views: shared, edit or delete | ✓ | ✓ | creator only |
| **Inbox**: read everything, send email and SMS, connect a mailbox, label threads, log calls, place calls | ✓ | ✓ | ✓ |
| Inbox: disconnect a mailbox; create, rename or delete labels | ✓ | ✓ | — |
| **Phone numbers**: search, buy, assign, release | ✓ | ✓ | — |
| **Automations and sequences**: view (read-only builder) | ✓ | ✓ | ✓ |
| Automations and sequences: enroll records | ✓ | ✓ | ✓ |
| Automations and sequences: create, edit, activate, pause; folders; test run; retry | ✓ | ✓ | — |
| Automations: delete | ✓ | ✓ | — |
| Sequences: delete or archive | not possible for anyone (no endpoint) | | |
| Deliverability guardrails, workspace automation pause | ✓ | ✓ | — |
| Lead-scoring rules: edit | ✓ | ✓ | read-only |
| **Configuration**: pipelines, stages, statuses, custom fields, compliance, task defaults, general settings | ✓ | ✓ | read-only |
| Contact communication preferences (do-not-contact / suppressions): add or remove (`POST /suppressions`, `DELETE /suppressions/{id}`, `config:manage`; a legacy manager passes) | ✓ | ✓ | read-only |
| Tags: create | ✓ | ✓ | ✓ |
| Tags: delete | ✓ | ✓ | hidden in the UI, but allowed by the API (gap) |
| **Team**: invite, resend, revoke | ✓ | ✓ | — |
| Team: change role, remove | anyone | members only | — |
| Invite as | admin, member | admin, member | — |
| **Billing**: view wallet, usage, unpaid charges | ✓ | ✓ | ✓ |
| Billing: buy or change plan, seats, add-ons, top up the wallet, Stripe portal | ✓ | — | — |
| Billing alerts | ✓ | — | — |
| Delete workspace | ✓ | — | — |
| Duplicates: scan, dismiss, merge (plan feature) | ✓ | ✓ | ✓ |
| Dashboard: Recent Activity and team feed | ✓ | ✓ | hidden in the UI, but the API returns it (gap) |
| Own profile, password, sessions, notification preferences | ✓ | ✓ | ✓ |

Plan limits are checked **before** role checks where both apply. For example, sequences are a Pro and Business feature, so on other plans every role is refused. See [../pricing/pricing.md](../pricing/pricing.md).

## Known gaps (code, not design)

- **The activity feed is not role-checked in the API.** `GET /activities` has no role gate, so a member can fetch the whole workspace's activity feed through the API. Only the UI hides it.
- **Tag delete is not role-checked in the API.** `DELETE /tags/{id}` lets any member delete a tag. Only the UI hides the button.
- **Legacy `manager` is half-admin.** It passes capability checks (`authz.Can`) but fails the literal owner/admin checks in team management, General settings, mailbox disconnect and pipelines/stages. Old manager rows should be migrated to admin or member.
- **Calls have no ownership check.** Any member can edit or delete any call log entry.
- **Admins and members on the billing page.** They are refused the subscription read, but the subscription and invoice cards still render for them, empty.
