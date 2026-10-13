---
title: Email and SMS templates (save a message once, reuse it anywhere)
status: approved        # proposed | approved | in-progress | built | dropped
owner: Qamar Ul Islam
area: docs/areas/inbox.md, docs/areas/sequences.md, docs/areas/automations.md
screens: [engage (Templates tab; aliases email-templates, sms-templates), inbox, email-sequence-builder, sms-sequence-builder, automation-builder]
tickets: []             # filled in after approval
created: 2026-10-13
---

# Email and SMS templates

## Problem
Reps write the same emails and texts again and again: the intro after a demo request, the proposal follow-up, the meeting reminder. Today nrtur has nowhere to save a message. Each rep keeps their own copy somewhere else and pastes it in, so wording drifts, mistakes repeat, and a new rep starts from nothing. Sequence and automation steps have the same problem: every step is written from scratch.

## Proposal
A shared **template library** for the workspace, with one list for email and one for SMS. Anyone can save a message as a template, and anyone can **insert** one wherever they write a message: the inbox, sequence steps and automation steps.

Inserting a template **copies** its text in. The person can still change it before sending or saving. Later edits to the template never change a message that was already sent or a step that was already saved.

**Screens** (design.nrtur.io):
- `engage` › **Templates** (new tab, between Automations and Rules), with sub-tabs **Email templates** and **SMS templates**. The page ids `email-templates` and `sms-templates` open it directly.
  - A card grid with search and a sort (Recently updated / Name).
  - Each card shows the name, the subject (email only), the start of the text, and "Updated … by …". SMS cards also show the SMS count.
  - A ⋯ menu on each card: Edit (or View), Duplicate, Delete.
  - An empty state, and a "New template" button.
- **Template editor** (dialog):
  - Fields: name; subject (email only); the message itself, in the same rich-text editor as the inbox for email, or plain text for SMS.
  - An "Insert merge field" row, and a live preview filled with a sample contact.
  - SMS shows a character and SMS count.
- **"Templates" button** (opens a searchable list) in:
  - `inbox`: New email, reply and forward, the SMS thread composer, and New SMS (also used on the contact page).
  - `email-sequence-builder` / `sms-sequence-builder`: the step editor ("Insert from a template").
  - `automation-builder`: the Send email and Send SMS steps ("Insert from a template").

## Rules

**A template**
- Belongs to one channel: **email** or **SMS**. An email template can't be inserted into an SMS, or the other way round.
- **Name:** required, up to 100 characters. It must be **unique within its channel** in the workspace, ignoring case. An email and an SMS template may share a name.
- **Email:** subject (required, up to 255 characters) and body (required). The body is formatted text (bold, italic, bullet list, link), stored as HTML, like the inbox composer.
- **SMS:** body only (required), plain text, up to **1,600 characters** (10 SMS). The editor shows the character and SMS count with sample values filled in, using the same GSM-7 / Unicode counting as sequences. A long message is allowed but flagged in amber once it goes over one SMS.
- **Up to 500 templates per channel** per workspace. At the limit, "New template" and Duplicate are disabled with a tooltip.
- **Visible to the whole workspace.** There are no private templates in v1.
- Each template records who created it, who last updated it, and when.

**Merge fields**
- These are the same 11 merge fields sequences and automations already use: `{{first_name}}` `{{last_name}}` `{{full_name}}` `{{email}}` `{{phone}}` `{{company_name}}` `{{owner_first_name}}` `{{owner_full_name}}` `{{deal_name}}` `{{deal_amount}}` `{{workspace_name}}`.
- The fallback form `{{first_name|there}}` works as it does today.
- An **unknown merge field** (e.g. a typo like `{{frist_name}}`) is refused when the template is saved: 422, naming the field. This is the same rule sequences and automations apply.
- The editor inserts a field at the cursor, in the subject or the body.

**Inserting a template**
- If the message or step already has text, nrtur asks "Replace with ‘<name>’?" first. Otherwise the template goes straight in.
- **Email:** it fills the subject and the body.
  - In a **reply or forward**, only the body is filled. The subject stays "Re: …" / "Fwd: …".
- **Inbox:** merge fields are filled in **when the template is inserted**, using the recipient. The rep sees exactly what will be sent.
  - The recipient is the contact matched by the email address in To, or the contact or lead of the SMS thread.
  - The owner fields come from that contact's owner, and `{{workspace_name}}` from the workspace.
  - There's no deal in the inbox, so `{{deal_name}}` and `{{deal_amount}}` have no value there.
  - A field with no value uses its fallback, if it has one. Otherwise **it stays in the text**.
  - **While any merge field is left in the text, Send is disabled.** A notice lists the fields, e.g. "Fill in or remove this merge field before sending: {{deal_name}}". This also applies to a merge field the rep typed by hand. Today the inbox would send `{{first_name}}` literally, because it never fills merge fields.
  - If To is empty when the template is inserted, the fields stay in the text until the rep replaces them.
- **Sequence and automation steps:** the merge fields are **kept** and filled in when each message sends, exactly as today.
  - A **sequence email step** edits plain text, so the template's formatting becomes plain text there. Line breaks stay, and list items keep a "•".
  - An **automation Send email step** keeps the HTML, because that step's body is HTML today.
- Nothing links a message or step back to its template. A template can be edited or deleted at any time, and nothing else changes.

**Managing templates**
- **Duplicate** makes a copy named "<name> (copy)" ("(copy 2)" and so on if that name is taken). The person who duplicates it becomes the creator.
- **Delete** asks first. It's a hard delete, and nothing else is affected because steps hold their own copy.
- **Read-only workspace** (cancelled or trial ended): templates can be viewed but not created, edited, duplicated or deleted. Inserting is still possible, but sending is refused as it is today.
- Plan limits on sending (SMS wallet, automated-send meter, sequences being a Pro and Business feature) are unchanged. A template is just text.

## Permissions
| | Owner | Admin | Member |
|---|:-:|:-:|:-:|
| See the template library | ✓ | ✓ | ✓ |
| Insert a template (anywhere they can already write) | ✓ | ✓ | ✓ |
| Create or duplicate a template | ✓ | ✓ | ✓ |
| Edit or delete **any** template | ✓ | ✓ | — |
| Edit or delete a template **they created** | ✓ | ✓ | ✓ |

This follows the shared saved-views rule (creator, owner or admin). A member who opens someone else's template sees it read-only: "Created by <name>. Only they or an admin can edit it.", with a **Duplicate** button.

Inserting into a sequence or automation step needs the rights that step already needs (owner or admin to edit automations and sequences). Nothing new there.

## Plan and pricing impact
Available on **every plan, including the trial**. Templates cost nothing to store, and sending still goes through today's plan gates. `pricing/pricing.md` doesn't change. The 500-per-channel limit is a safety limit, not a plan limit.

## Out of scope (not in this version)
- Categories, folders, favourites, or per-template usage and open/reply stats. The vision design has these; they were trimmed.
- Private (personal) templates and team-only sharing.
- Attachments or images in templates.
- A drag-and-drop email designer or HTML import.
- **Linking** steps to templates, so that editing a template updates every step that uses it.
- Starter templates in a new workspace (it starts empty).
- Templates in the deal **stage email** composer. That screen is still being built; it can add the same "Templates" button later.
- Filling merge fields on the server for inbox sends. The inbox fills them in the browser when a template is inserted, as described above.
- Rich-text editing for sequence email steps.

## Decisions (owner, 2026-10-13)
Approved in PR #5 with the open questions accepted as proposed:
1. **Permissions:** everyone can create and duplicate; editing or deleting needs the creator, an owner or an admin.
2. **Limit:** 500 templates per channel per workspace (a safety limit, not a plan limit).
3. **Inbox merge fields:** filled in the browser when a template is inserted; Send is blocked while any known merge field is left. No server-side filling for inbox sends in v1.
4. **Sequence email steps** stay plain text in v1; a template's formatting is flattened there.

## Tickets (to create after approval)
- **BE:** message templates.
  - A `message_templates` table: workspace, channel, name, subject, body, created_by, updated_by and timestamps, with unique name per workspace and channel ignoring case.
  - `GET /templates?channel=` (search `q`, sort), `POST /templates`, `PATCH /templates/{id}` and `DELETE /templates/{id}`.
  - Validation: lengths, the merge-field vocabulary (reuse `IsMergeTag`), 500 per channel, the creator-or-`config:manage` rule for edit and delete, and refusal while the workspace is read-only.
  - OpenAPI.
- **FE:** the Templates tab in Engage (list, search, sort, editor with preview and merge-field insert, duplicate, delete, empty and read-only states).
- **FE:** the "Templates" picker, with the "Replace?" confirm:
  - in the inbox (new email, reply and forward, SMS thread, New SMS), with merge-field filling for the recipient and the leftover-field Send block;
  - in the sequence step editor and the automation Send email / Send SMS steps.
