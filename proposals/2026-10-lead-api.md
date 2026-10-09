---
title: Lead API (create leads from outside nrtur)
status: proposed        # proposed | approved | in-progress | built | dropped
owner: Qamar Ul Islam
area: docs/areas/leads.md, docs/areas/workspace-settings.md
screens: [settings-integrations, lead-detail, leads]
tickets: [SCRUM-1239 (design), SCRUM-1240 (BE), SCRUM-1241 (FE)]
created: 2026-10-09
---

# Lead API

## Problem
Leads come from places that aren't nrtur: website forms, landing-page builders, Zapier/Make, ad tools and partners. Today a team can get those leads into nrtur only by typing them in or importing a CSV. Leads go stale while they wait, and there is no way to connect a website form directly.

## Proposal
Each workspace can create **one Lead API key** in **Settings › Integrations**. An outside system sends that key with a simple HTTPS request, and a new lead appears in nrtur. The API creates leads **only**: it can't read, edit or delete anything.

The same card has a workspace setting, **Allow duplicate leads**, which decides what happens when an incoming lead matches an existing one.

This is **not** the automation "call webhook" step. That one sends data *out* of nrtur; this brings leads *in*.

**Screens** (design.nrtur.io):
- `settings-integrations`: a new **Lead API** card under Email accounts.
- `leads` / `lead-detail`: a lead created by the API shows Source **API**, and its timeline says "Created via Lead API".

## Rules

**The key**
- One active key per workspace.
- **Generate** creates it. The full key is shown **once**, in a dialog with a Copy button and the warning "You won't see this key again". After that, only a masked prefix is shown, e.g. `nrtur_live_7Kd2…`.
- The card shows: masked key, created by, created on, and last used ("Never used" if unused).
- **Regenerate** replaces the key. The old key stops working **immediately**. A confirm dialog says so.
- **Revoke** deletes the key. The API stops accepting requests until a new key is generated. A confirm dialog is shown.
- The key is stored hashed. nrtur can never show it again, so a lost key means regenerating.

**The request**
- `POST https://api.nrtur.io/api/public/v1/leads` (production), with the header `Authorization: Bearer <key>` and a JSON body.
- Accepted fields: `name`, `email`, `phone`, `company_name`, `job_title`, `source`, `estimated_value`, `notes`. These are the same fields as the in-app New Lead form, plus `notes`.
- `estimated_value` is a decimal string, e.g. `"5000.00"`, the same format as the in-app API.
- **Name or email is required**, the same rule as creating a lead in the app.
- `source` defaults to `"API"` when it's missing or blank. A sender may pass its own (e.g. `"Website form"`).
- **"API" is added to the lead Source list** in the app, so API leads can be filtered by source like any other.
- `notes`, if present, becomes the first note on the lead's timeline.
- **Status:** the workspace default lead status.
- **Owner:** the person who generated the key.
- **Activity:** the lead's timeline shows "Created via Lead API".
- The response is `201` with the new lead's `id` and its link in nrtur.

**Duplicates** (the "Allow duplicate leads" setting, default **off**)
- **Off:** if an active lead already has the same email or phone, the request is refused with `409 DUPLICATE_LEAD` and the existing lead's id. If the email belongs to an existing contact, it's refused with `409 LEAD_EMAIL_IS_CONTACT` and the contact's id. These are the same rules the app uses today.
- **On:** the lead is created even if it matches an existing lead. A clash with a contact's email is still refused, because otherwise the lead would later convert into a duplicate contact.
- The setting applies **only to leads created through the API**. Creating leads in the app and by CSV import keeps today's rules.

**Limits and refusals**
- **Plan:** available on Trial, Pro and Business, through the existing `api_access` plan feature. On Solo and Team the card shows the plan gate and the API refuses with `403 PLAN_FEATURE_UNAVAILABLE`.
- **Records cap:** the leads cap applies exactly as in the app. At the cap the API refuses with `422 PLAN_LIMIT_REACHED`.
- **Read-only workspace** (cancelled or trial ended): the API refuses every request with `403`, and the card shows the read-only notice.
- **Rate limit:** 60 requests per minute per workspace. Above that the API returns `429` with a `Retry-After` header.
- **Missing or wrong key:** `401` with no detail, so the response gives nothing away to someone guessing keys.
- **Validation errors** (bad email or phone, both name and email missing): `422` naming the field. The phone check is the same E.164 rule as the app.

**What the card shows to help people use it**
- The endpoint URL with a Copy button.
- A short example request (curl) with a Copy button.
- A one-line explanation of the duplicate setting.

## Permissions
| | Owner | Admin | Member |
|---|:-:|:-:|:-:|
| See the Lead API card, endpoint and example | ✓ | ✓ | ✓ (read-only) |
| Generate, regenerate or revoke the key | ✓ | ✓ | — |
| Change "Allow duplicate leads" | ✓ | ✓ | — |
| See the full key | only once, right after generating | only once, right after generating | never |

Members see "Ask an owner or admin to manage the Lead API key". This follows the same `config:manage` rule as the rest of the workspace setup.

## Plan and pricing impact
No price change. The plan catalog already has **API access** (`api_access`) on Trial, Pro and Business. This is the first feature to use it. `pricing/pricing.md` is updated to say what "API access" means.

## Out of scope (not in this version)
- Reading, updating or deleting leads through the API, and contacts, companies or deals.
- More than one key per workspace, named keys, or per-key permissions.
- Setting tags, custom fields or a specific owner or status per request.
- Outbound webhooks ("tell my system when something changes").
- A "Lead created" automation trigger. API leads won't start automations yet, because the live triggers include no "lead created". That should be its own proposal.
- A public developer docs site. The card's example and the OpenAPI spec are enough for v1.

## Decisions (owner, 2026-10-09)
1. **Owner of API leads:** the person who generated the key.
2. **"Allow duplicate leads" applies to the API only.** Creating leads in the app and by CSV import keeps today's rules (SCRUM-992).
3. **Rate limit:** 60 requests per minute per workspace.
4. **Notifications for new API leads:** not in v1.
5. **API address:** production is `https://api.nrtur.io/api/public/v1/leads`. Staging uses the staging API host with the same path. The settings card shows the address of the environment it runs in.
