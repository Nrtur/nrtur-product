---
title: Lead API v2 (named keys, in-app docs, request log)
status: proposed        # proposed | approved | in-progress | built | dropped
owner: Qamar Ul Islam
area: docs/areas/leads.md, docs/areas/workspace-settings.md
screens: [settings-integrations]
tickets: []             # filled in after approval
created: 2026-10-20
---

# Lead API v2

Builds on the approved and built [Lead API](2026-10-lead-api.md) (v1). Everything in v1 still applies unless a rule below changes it.

## Problem
Demo feedback on v1 raised three gaps:
- **One unnamed key.** A workspace that sends leads from a website form, Zapier and a partner has to share one key between all of them. It can't tell them apart, and revoking a leak breaks every connection at once.
- **No docs.** The card shows one curl example. Anyone wiring up a form has to guess the fields, the rules and the error responses.
- **No visibility.** When leads don't arrive, nobody can see whether requests reached nrtur or why they were refused.

## Proposal
In **Settings › Integrations** (`settings-integrations` on design.nrtur.io), the Lead API card changes in three ways:

1. **Named keys.** A workspace can have up to **10 active keys**, each with a name such as "Website form" or "Zapier". The card lists them with their usage. Owners and admins can create, rename, regenerate and revoke each key on its own.
2. **In-app API docs.** The endpoint row gets a **View API docs** button. It opens a drawer with four tabs: Overview, Fields, Responses and Examples. There is no separate docs site.
3. **Recent requests.** Owners and admins see a log of the last 30 days of requests: which key sent it, and whether it created a lead or why it was refused.

"Allow duplicate leads" stays one workspace-wide setting, unchanged.

## Rules

**Keys**
- At most **10 active keys** per workspace. Revoked keys don't count.
- At 10 active keys, **New key** is disabled with the tooltip "You can have up to 10 active keys. Revoke one to add another." The API refuses an 11th create the same way.
- **Name:** required, trimmed, at most **50 characters**, and unique among the workspace's **active** keys, ignoring case ("zapier" clashes with "Zapier"). A revoked key's name can be reused.
- **New key** asks for a name, then shows the full key **once** in the existing dialog ("You won't see this key again", Copy). After that only the masked prefix is shown, e.g. `nrtur_live_7Kd2…`.
- **Rename** uses the same name rules. It doesn't change the key value, so anything using the key keeps working.
- **Regenerate** replaces one key's value. The old value stops working **immediately**. A confirm dialog says so. The name and the key's request history stay. The new value is shown once. The key **keeps its original creator** and created date: its leads stay owned by that member, and it's auto-revoked only if they leave. Last used resets to "Never used".
- **Revoke** deletes one key after a confirm dialog. It stops working immediately and leaves the active list. The workspace's other keys keep working.
- Each key row shows: name, masked prefix, "Created by <member> · <date>", "Last used <time ago>" or "Never used", and "<N> leads (30d)", the number of leads that key created in the last 30 days.
- The list shows about 5 rows, then scrolls.
- **Empty state:** "No keys yet. Create one to start sending leads to nrtur."
- Keys are stored hashed. nrtur can never show a key again, so a lost key means regenerating it.

**Ownership and automatic revoke (now per key)**
- A lead created through a key is owned by **the member who created that key**.
- When a member is removed or suspended, **every key they created** is revoked automatically. Keys created by other members keep working.

**Plans and read-only (now per key)**
- Available on Trial, Pro and Business through `api_access`, as in v1.
- **Revoke works on every plan.** On Solo or Team the card shows the plan gate, and below it owners and admins see the list of keys that still exist, each with **Revoke**, so a leaked key can't come back on a re-upgrade. Every other action stays plan-gated.
- **Read-only workspace:** the card shows the read-only notice. The key list, endpoint, docs and request log stay visible. **Revoke** still works. New key, Rename, Regenerate and the duplicates setting are disabled.

**Migration**
- The existing v1 key becomes a key named **"Default key"**. Its value, creator, created date and last-used time stay the same, and it keeps working unchanged. Nobody has to update an integration.
- A workspace with no v1 key starts with no keys.

**API docs drawer** (opens from **View API docs**, for everyone who can see the card)
- **Overview:** endpoint `POST https://api.nrtur.io/api/public/v1/leads` (production) with Copy, and the note "Staging uses the staging API host with the same path". Auth header `Authorization: Bearer <key>`. JSON body. Creates leads only. Rate limit 60 requests per minute per workspace, shared by all keys, with `429` and `Retry-After` above it. A summary of the duplicates setting showing its current value.
- **Fields:** a table of name, type, required, notes and example for `name`, `email`, `phone`, `company_name`, `job_title`, `source`, `estimated_value` and `notes`, with the v1 rules: name or email required; phone 7–15 digits after removing spaces, dashes, dots and brackets, with an optional leading `+`; `source` one of Web form, Referral, Event, Cold outreach, Ad, LinkedIn, Import, API (case-insensitive), default "API"; `estimated_value` a decimal string such as `"5000.00"`; `notes` becomes the first timeline note.
- **Responses:** status, code, when and an example body for each response in `nrtur-backend/api/openapi.yaml` (`publicCreateLead`): `201` (`id` and `url` of the lead), `400 BAD_REQUEST`, `401 UNAUTHORIZED` (no detail), `409 DUPLICATE_LEAD` (`details`: `existing_lead_id` and `matched_on` email or phone, never the name), `409 LEAD_EMAIL_IS_CONTACT` (`details`: `contact_id`), `413 PAYLOAD_TOO_LARGE` (over 64 KB), `415 UNSUPPORTED_MEDIA_TYPE`, `422 VALIDATION_ERROR` (one `{field, code, message}` per failing field), `422 PLAN_FEATURE_UNAVAILABLE` (tier in `details`), `422 PLAN_LIMIT_REACHED` and `429 RATE_LIMITED` (`Retry-After`). Every error uses the standard `{error: {code, message, details}}` envelope. The OpenAPI file stays the contract; if it changes, this tab changes with it.
- **Examples:** curl, JavaScript `fetch` (with the note to send requests from a server, never from a visitor's browser), and a "Using Zapier or Make" tip: their Webhooks or HTTP action with POST, the URL, the header and a JSON body. Every code block has Copy.
- The card itself keeps a compact endpoint row (POST, URL, Copy, View API docs). The v1 inline example block moves into the drawer.

**Request log ("Recent requests")**
- Every request with a valid key is recorded: time, key, result and, for refusals, the status code, error code and field.
- **Result** is one of: **Created**, **Duplicate** (`409 DUPLICATE_LEAD` or `409 LEAD_EMAIL_IS_CONTACT`), **Invalid** (`400`, `413`, `415` or `422 VALIDATION_ERROR`), **Rate limited** (`429`) and **Plan refused** (`422 PLAN_FEATURE_UNAVAILABLE` or `PLAN_LIMIT_REACHED`).
- Each row shows the time, key name, a result chip, and either **View lead** (Created, opens the lead) or the detail, e.g. `422 · phone` or `409 · DUPLICATE_LEAD · matched on email`.
- **Filters:** key (All keys, or one key) and result (All, Created, Duplicate, Invalid, Rate limited, Plan refused). Rows from a revoked key show its last name with "(revoked)" and stay filterable.
- Newest first, **25 rows at a time** with **Load more**.
- The screen shows **the last 30 days** only, with the note "Showing the last 30 days."
- **Retention:** the backend keeps the log for **12 months**, then a cleanup job deletes older rows.
- **Privacy:** request bodies are **never stored**, only the result, codes and field name.
- **Old keys are recognised.** When a key is revoked or regenerated, its hash is kept (marked revoked), so a request that still uses it can be tied to the workspace and the key. Those requests are refused with `401` as today, and a single line above the list counts them: "N requests used a revoked or replaced key in the last 30 days. Update whatever is still sending it."
- **Keys nrtur never issued** (typos, random strings) can't be tied to any workspace, so they aren't counted or listed anywhere. They only count toward the per-IP failed-lookup limit, as today.
- On Solo and Team the log is behind the plan gate with the rest of the card.

## Permissions
| | Owner | Admin | Member |
|---|:-:|:-:|:-:|
| See the key list (names, prefixes, usage), endpoint and API docs | ✓ | ✓ | ✓ (read-only) |
| Create, rename, regenerate or revoke a key | ✓ | ✓ | — |
| Revoke a key on Solo, Team or read-only | ✓ | ✓ | — |
| Change "Allow duplicate leads" | ✓ | ✓ | — |
| See Recent requests | ✓ | ✓ | — (hidden) |
| See a full key | only once, right after creating or regenerating it | only once, right after creating or regenerating it | never |

Members see "Ask an owner or admin to manage Lead API keys." This follows the same `config:manage` rule as v1.

## Plan and pricing impact
None. Still the `api_access` plan feature on Trial, Pro and Business. No limit or price in `pricing/` changes. The 10-key limit is a product limit, the same on every plan that has the API.

## Out of scope (not in this version)
- Per-key permissions or scopes.
- Choosing a different owner for a key's leads (an owner picker per key).
- Outbound webhooks.
- A public developer docs site.
- Exporting the request log.
- Showing request bodies in the log.

## Decisions (owner, 2026-10-20)
1. **Up to 10 active keys** per workspace.
2. **Leads are owned by the key's creator**, and a removed or suspended member's keys are revoked automatically.
3. **The full request history is kept (12 months); the screen shows the last 30 days.**

## Follow-up (owner, 2026-10-21)
1. **Regenerate keeps the original creator.** It doesn't change who owns the key's leads or whose removal revokes it.
2. **"Plan refused" is its own option** in the request log's result filter.

## Tickets (to create after approval)
- **BE**
  - `lead_api_keys`: one row per key with its own id and `name`; partial unique index on lower(name) among active keys; at most 10 active keys per workspace.
  - Migration: the existing key becomes a row named "Default key", same hash, creator and dates; it keeps working.
  - Endpoints to create, list, rename, regenerate and revoke a key, owner/admin only (`config:manage`). Revoke allowed on every plan and in read-only; the rest plan-gated. The list includes "leads (30d)" per key.
  - Automatic revoke of every key a member created when they are removed or suspended. API leads owned by the key's creator.
  - Request-log table (time, key id, result, status, error code, field; no bodies) with 12-month retention and a cleanup job. Revoked and replaced keys keep their hash (status revoked) so later requests with them are recognised; a per-workspace count of those requests.
  - `GET` log endpoint: filters by key and result, 30-day window, paged by 25, plus the count of requests that used a revoked or replaced key in the window. Owner/admin only.
- **FE**
  - Key list with New key / Rename / Regenerate / Revoke dialogs and the 10-key limit, the plan-gate list with Revoke, and the member read-only view.
  - View API docs drawer (Overview, Fields, Responses, Examples).
  - Recent requests section with filters, Load more and the revoked/replaced-key count.
