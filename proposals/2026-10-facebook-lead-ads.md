---
title: Facebook Lead Ads integration (Facebook and Instagram lead forms → nrtur leads)
status: proposed        # proposed | approved | in-progress | built | dropped
owner: Qamar Ul Islam
area: docs/areas/leads.md, docs/areas/workspace-settings.md (Integrations)
screens: [settings-integrations, leads, lead-detail]
tickets: []             # filled in after approval
created: 2026-10-26
---

# Facebook Lead Ads integration

## Problem
Most of our customers run Facebook and Instagram lead ads. Today those leads sit in Facebook. The persona research already says "Facebook ads (no integration with CRM)". Teams download a CSV or copy leads by hand, so the leads go cold before anyone calls. The Lead API (2026-10) helps only teams that can wire up Zapier or Make.

We want a native connection: a team connects Facebook once, picks the Pages and lead forms, and every new form submission becomes an nrtur lead within seconds.

This follows the way the integration works in our other product (aninja), but fixes its known problems: secrets in logs, an API key used as the OAuth state, plaintext tokens, no duplicate check, silent failures, and no data-deletion callback.

## Proposal
**v1 is Lead Ads only.** Facebook Messenger, Instagram DMs and page comments are out of scope. Instagram lead ads are included, because Meta delivers them through the same Page lead forms.

**Screens** (design.nrtur.io):
- `settings-integrations`: a new **Facebook Lead Ads** card under the Lead API card. The **Facebook** preview toggle switches between Not connected · Choose pages · Connected · Needs reconnect · Solo (gated) · Read-only · Member view.
- `leads` / `lead-detail`: Source **Facebook**, and a timeline entry "Created from Facebook Lead Ads · <Page> › <Form> · Ad: <ad name>".

**How it works for the user**
1. An owner or admin clicks **Connect Facebook** and signs in with Facebook (Meta's permission screen).
2. Back in nrtur, they **choose which Pages** to connect. Nothing is pre-selected.
3. Each connected Page lists its **lead forms**. They turn on **Receiving leads** per form. It's off by default, so nothing arrives by surprise.
4. For each form they check the **mapping**: which question fills which lead field, and who owns the leads. Sensible defaults are filled in automatically.
5. From then on, every submission on those forms becomes a lead in nrtur within seconds.
6. **Recent Facebook leads** shows what arrived: created, duplicate, skipped or failed, and why.
7. **Import past leads** pulls in the last 90 days for a form (Meta keeps form leads for 90 days).

## Rules

**Connection**
- **One Facebook connection per workspace.** It belongs to the workspace, not to a person.
- The person who connects is recorded as "Connected by". Their Facebook access is what nrtur uses to read leads.
- **Who:** owner or admin (`config:manage`) can connect, choose pages, edit forms and disconnect. Members see status, pages and forms read-only.
- **Sign-in:** standard Facebook Login (OAuth) with the permissions listed under **Meta app setup**.
  - The OAuth `state` is a **signed, single-use nonce** tied to the workspace and user, with a 15-minute expiry (the same pattern as the Gmail connect, `INBOX_STATE_SECRET`). It is never an API key or a user id.
  - After sign-in, the short-lived token is exchanged for a **long-lived user token** (about 60 days). Each chosen Page's **Page access token** is then fetched; these normally don't expire while the user's access stands.
- **Tokens are encrypted at rest** (AES-256-GCM, its own key, like `INBOX_ENCRYPTION_KEY`). They are never returned to the browser, never logged, and never put in URLs.
- **Missing permissions** at sign-in (the person unticked something on Meta's screen) are refused clearly: "nrtur needs permission to read your Pages' leads. Reconnect and keep all permissions on." The connection is not half-saved.
- **Choosing Pages:** only Pages the person administers are listed (paged, so large accounts work). Only chosen Pages are **subscribed** to nrtur (`POST /{page}/subscribed_apps` with `subscribed_fields=leadgen`). Unchosen Pages are never touched.
- **Manage pages** reopens the picker. Unticking a Page unsubscribes it; its forms stop receiving leads, and leads already created stay.
- **Disconnect** (confirm dialog):
  - unsubscribes every Page;
  - revokes nrtur's access (`DELETE /me/permissions`);
  - deletes the stored tokens.
  - Pages, forms and mappings are kept but inactive, so reconnecting restores them. Leads already created stay.
  - Disconnect works on every plan and in a read-only workspace, like Lead API revoke.

**Forms**
- **Listing:** each connected Page lists its lead forms (name and Facebook status Active / Archived), refreshed when the card opens and once a day.
- **Receiving leads** is a per-form switch, **off by default** for every form found, including forms created later. A form must be on for its submissions to become leads. Submissions to forms that are off are ignored, not stored.
- **Per form we show:** owner, "Last lead <time>" / "No leads yet", and leads in the last 30 days.
- **Archived forms** keep their settings but get no new submissions from Facebook.

**Mapping** (per form; the "Map ‘<form>’ to nrtur" drawer)
- **Two groups of rows.**
  - **Form questions:** one row per question on the Facebook form.
  - **Facebook details:** Page name, Form name, Ad name, Campaign name. These are off ("Don't import") by default, because they are already on the lead's timeline entry. Ad and campaign names need `ads_read`.
- **Each row maps to:**
  - a standard lead field: Name, Email, Phone, Company name, Job title or Estimated value;
  - **any lead custom field** in the workspace (the same list as Settings › Custom fields for leads);
  - **Add to the lead's note**;
  - **Don't import**.
- **Create a custom field from the mapping:** "+ Create custom field…" in the dropdown asks for a name (unique among lead custom fields) and a type (Text, Long text, Number, Date). It creates the field in Settings › Custom fields and maps the row to it in one step. Only owners and admins can do this, the same people who edit mappings.
- **Combining answers:**
  - Several rows can map to the **same text field** (Name, Company name, Job title, a text custom field, the note). Their answers are joined in the order shown, with a separator chosen per field: Space, Comma, Dot ( · ) or New line. Example: Company name = "Company name" + "City", joined with Comma.
  - **Email, Phone and Estimated value take one row each.** Two rows on one of them is an error, and Save stays disabled.
- **Defaults by question type:**
  - full name → Name;
  - first name + last name → Name, joined;
  - email → Email;
  - phone number → Phone;
  - company name → Company name;
  - job title → Job title;
  - everything else, including custom questions → **Add to the lead's note**, written as "Question: answer" lines.
- **Values are validated exactly as the app validates them:**
  - email format;
  - phone: 7–15 digits after removing spaces, dashes, dots and brackets, with an optional leading +;
  - estimated value as a decimal.
  - An invalid value isn't dropped silently: it goes into the note ("Phone (not valid): …") and the lead is still created.
- **Lead owner:** a member picked per form. The default is the person who connected. If that member is later removed or suspended, the form's owner falls back to the workspace owner, and a "Facebook connection needs attention" notification is sent.
- **Source** is always **Facebook**. "Facebook" is added to the lead Source list in the app, next to Web form, Ad and the others, so these leads can be filtered.
- **"Also save ad, campaign and form names on the lead"** (on by default) adds them to the timeline entry. It needs the `ads_read` permission; without it the names are left out and nothing fails.
- **Status:** the workspace default lead status.

**A new submission → a lead**
1. Meta calls nrtur's webhook. nrtur **checks the signature** (`X-Hub-Signature-256`, HMAC-SHA256 of the raw body with the app secret) **before doing anything**. A bad signature is refused with 401 and no work is done.
2. nrtur **answers Meta straight away** (200) and puts the work on the worker queue. It never calls Facebook inside the webhook request.
3. The worker fetches the lead (`GET /{leadgen_id}` with the Page token), applies the form's mapping, and creates the lead **through the same code path as the Lead API**, so the same records cap, validation and duplicate rules apply.
4. **Exactly once:** each Facebook lead id is stored with a unique constraint per workspace, so Meta's retries and repeated webhooks never create two leads. A repeat is recorded as already handled.
5. **Retries:** Facebook errors and timeouts are retried with backoff (up to 10 tries over about a day). After the last try the row shows **Failed**, with a **Retry** button.

**Duplicates** (the same rules as the app and the Lead API, using the workspace "Allow duplicate leads" setting from the Lead API card)
- **Setting off (default):** a submission whose email or phone matches an active lead does **not** create a new lead. Instead:
  - the existing lead gets a timeline entry "Submitted ‘<form>’ again on Facebook", with the answers;
  - the row in Recent Facebook leads shows **Duplicate**, with a link to that lead.
- **Setting on:** a new lead is created anyway.
- **Email belongs to a contact:** never creates a lead. The contact gets the same timeline entry, and the row shows Duplicate with a link to the contact.

**Skipped and refused**
- **No name and no email:** not created. The row shows **Skipped — no name or email**.
- **Records cap reached:** not created. The row shows **Failed — leads limit reached**, the owner gets the existing cap notification, and Retry works after space is freed.
- **Plan without the feature, or a read-only workspace:** submissions are not processed. Rows show **Failed — not on this plan**. Meta's subscription stays, so re-upgrading resumes without reconnecting.

**Timeline and activity**
- **Created entry:** "Created from Facebook Lead Ads · <Page> › <Form> · Ad: <ad name>" (ad and campaign only when that option is on).
- **Note:** a note with the unmapped answers, when there are any.
- **No automations fire, and no "new lead" notification,** the same as Lead API leads today. The backend already has the automation trigger `lead.ad_received` ("New ad lead received"), not live yet; turning it on is its own proposal. The lead owner still gets the existing `lead.assigned` notification.

**Recent Facebook leads** (owner/admin)
- **Rows:** the last 30 days, newest first. Each row has the time, Page › form, a result chip (**Created / Duplicate / Skipped / Failed**), and a link to the lead or the reason.
- **Filters:** by form and by result. **Retry** on Failed rows.
- **What's stored:** every processed submission keeps its result, Facebook lead id, form id and lead id for 12 months. **Form answers are not stored there**, only on the lead itself.

**Import past leads**
- **Scope:** per form, the last 90 days (Meta's limit).
- **The dialog** shows "N leads from the last 90 days · M already in nrtur" and imports the rest through the same path, including duplicates, the cap and mapping.
- **Paging:** uses Facebook's cursors server-side. The browser never sees a Facebook URL or token.
- **Limits:** one import per form at a time, on the worker queue, with progress shown.

**Health and reconnecting**
- **Detecting a broken connection:** Facebook errors that mean the access stopped working (expired or invalidated token, permission removed, Page access lost) set the connection to **Needs reconnect**.
- **What the user sees:** the card's amber banner explains it, the forms show "Paused — reconnect needed", and the owner and admins get one **"Facebook connection needs attention"** notification. This is a new notification type, like `inbox.mailbox_disconnected`.
- **Fixing it:** Reconnect runs sign-in again and keeps every Page, form and mapping.
- **Daily check:** the token and permissions are checked once a day as well, so a quiet form doesn't hide a broken connection.
- **Lost submissions:** submissions that arrived while the connection was broken are retried after reconnecting, within their retry window. Older ones can be pulled with Import past leads.

**Meta requirements**
- **Deauthorize callback:** when a person removes nrtur from their Facebook settings, Meta tells nrtur. nrtur marks the connection **Needs reconnect** and deletes the tokens.
- **Data deletion callback:** when a Facebook user asks Meta to delete their data, nrtur deletes the stored Facebook tokens and ids for that person and returns Meta's required confirmation code and status URL. Leads already created are the workspace's own business data and stay.
- **App review:** the permissions below need Meta App Review (Advanced Access) before customers outside our own business can connect. Until then only people with a role on the Meta app can use it, which is enough for staging and testing.

**Plan**
- Gated by the existing **`api_access`** plan feature ("API access and webhooks"), the same as the Lead API: **Trial, Pro and Business**.
- On Solo and Team the card shows the plan gate. Disconnect still works.

## Meta app setup (done once, per environment)
In the Meta developer dashboard for our app:
- **Products:** Facebook Login (or Facebook Login for Business) and **Webhooks**.
- **Valid OAuth redirect URI:** `<API base>/api/v1/integrations/facebook/callback`.
- **Webhooks → Page:**
  - callback `<API base>/api/v1/integrations/facebook/webhook`;
  - verify token = our `FACEBOOK_WEBHOOK_VERIFY_TOKEN`;
  - subscribe to the **`leadgen`** field.
- **Deauthorize callback URL:** `<API base>/api/v1/integrations/facebook/deauthorize`.
- **Data deletion request URL:** `<API base>/api/v1/integrations/facebook/data-deletion`.
- **Privacy policy URL** and **terms URL:** our public pages.
- **Permissions** (requested at sign-in; App Review needed for each):

| Permission | Why |
|---|---|
| `pages_show_list` | List the Pages the person manages |
| `pages_manage_metadata` | Subscribe the chosen Pages to the `leadgen` webhook |
| `pages_read_engagement` | Read Page details needed with the above |
| `leads_retrieval` | Read the lead form submissions |
| `business_management` | Pages owned through a Business Manager |
| `ads_read` | Optional: ad and campaign names on the lead |

- **Environments:** one Meta app per environment (staging and production), following the environment split (SCRUM-1235–1238). Each has its own App ID, secret and webhook URL. The person connecting also needs **Leads Access** on the Page in Meta Business Suite.

## Configuration (backend environment; names only, values never in this repo)

| Name | What it is | Who provides it |
|---|---|---|
| `FACEBOOK_APP_ID` | The Meta app's App ID | From the Meta dashboard (owner) |
| `FACEBOOK_APP_SECRET` | The Meta app's App Secret: token exchange and webhook signatures | From the Meta dashboard (owner) |
| `FACEBOOK_GRAPH_VERSION` | Graph API version, e.g. `v23.0` | We set it |
| `FACEBOOK_WEBHOOK_VERIFY_TOKEN` | A random string Meta echoes when the webhook is set up | We generate it, and paste it into the Meta dashboard |
| `FACEBOOK_STATE_SECRET` | Signs the OAuth state | We generate it |
| `FACEBOOK_ENCRYPTION_KEY` | 32-byte base64 key for encrypting tokens | We generate it, one per environment |

The callback and webhook URLs are built from the existing `PUBLIC_BASE_URL`; the return to the app uses `FRONTEND_URL`. **No personal access token is needed**: each workspace's tokens come from its own sign-in.

## Permissions

| | Owner | Admin | Member |
|---|:-:|:-:|:-:|
| See the card, status, Pages and forms | ✓ | ✓ | ✓ (read-only) |
| Connect, Manage pages, Reconnect, Disconnect | ✓ | ✓ | — |
| Turn forms on or off, edit mapping and owner, Import past leads | ✓ | ✓ | — |
| See Recent Facebook leads, Retry | ✓ | ✓ | — |

## Plan and pricing impact
No price change. Facebook Lead Ads joins the Lead API under the existing `api_access` feature (Trial, Pro, Business). `pricing/pricing.md` will say that "API access" includes the Facebook Lead Ads integration.

## Out of scope (not in this version)
- Facebook Messenger, Instagram Direct and Page comments.
- Sending data back to Meta (Conversions API, offline events).
- Creating or managing ads, audiences or forms from nrtur.
- More than one Facebook connection per workspace.
- Turning on the "New ad lead received" automation trigger (`lead.ad_received`). That gets its own proposal.
- Google, LinkedIn or TikTok lead forms. They'll reuse the same pattern later.

## Decisions (owner, 2026-10-27)
- **Mapping matches aninja and goes further.** It covers:
  - mapping to any lead custom field;
  - creating a custom field from the mapping;
  - combining several answers into one field with a chosen separator;
  - mapping Facebook details (Page, form, ad and campaign names) into fields.

## Open questions
1. **Plan:** gate with `api_access` (Trial, Pro, Business), or open it to Team as well?
2. **Ad and campaign names:** ask for `ads_read` (extra App Review item) to show them, or skip it in v1?

## Tickets (to create after approval)
- **BE:** Facebook connection and webhook.
  - **Tables:** connection, pages, forms (with mapping and owner), and processed submissions (unique per workspace + Facebook lead id, 12-month retention). Tokens are AES-GCM encrypted.
  - **Sign-in:** OAuth connect URL with a signed state, callback (long-lived token exchange, permission check, Page list paged), choose-pages (subscribe and unsubscribe), disconnect (unsubscribe, revoke, delete tokens).
  - **Webhook:** verify GET and signed POST; ack fast and enqueue.
  - **Worker job:** fetch the lead → mapping → create through the Lead API create path with `via=facebook_lead_ads`, plus duplicate handling (timeline entry on the existing lead or contact), retries and a final Failed state. Retry endpoint.
  - **Forms:** list and refresh, on/off, mapping (rows for questions and Facebook details, any lead custom field, combined fields with a separator) and owner.
  - **Import past leads** job (server-side cursors).
  - **Health:** daily health check, Needs reconnect state, and the new notification type.
  - **Meta callbacks:** deauthorize and data-deletion.
  - **Plan and source:** `api_access` gate; "Facebook" added to the allowed sources.
  - Config, OpenAPI and tests.
- **FE:** the Facebook Lead Ads card.
  - All the states (not connected, choose pages, connected, needs reconnect, gated, read-only, member).
  - Pages with forms, the mapping drawer (custom fields, "+ Create custom field" through the existing custom-fields API, combining with a separator, Facebook details rows), the import dialog and Recent Facebook leads.
  - Source "Facebook" in lead filters and badges.
