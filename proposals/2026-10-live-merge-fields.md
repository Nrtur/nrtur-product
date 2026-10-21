---
title: Live merge fields in the inbox composers
status: proposed        # proposed | approved | in-progress | built | dropped
owner: Qamar Ul Islam
area: docs/areas/inbox.md (templates in the composers); changes proposals/2026-10-message-templates.md › Inserting a template
screens: [inbox, contact-detail, lead-detail, deal-detail]
tickets: []             # filled in after approval
created: 2026-10-20
---

# Live merge fields in the inbox composers

## Problem
Found in the demo (2026-10-20). In the inbox, a template's merge fields are filled **only at the moment the template is inserted**. When **To** is still empty, which is how most people start an email, the message keeps `{{first_name}}`, `{{company_name}}` and the other fields. Typing the recipient afterwards fills nothing, and Send stays blocked until the rep replaces every field by hand.

The backend **doesn't fill merge fields for inbox sends**, and it shouldn't be trusted to: the rep should see the exact text that will be sent. So the composer has to fill them, at runtime.

## Proposal
**Screens** (design.nrtur.io): `inbox`, `contact-detail`, `lead-detail`, `deal-detail`. Covers the email composer (Compose / Quick Add and the record pages), reply and forward, the New SMS dialog (also used on the contact page) and the SMS thread composer.

**The rule (owner, 2026-10-21):** when we know who the email is for, nrtur fills every merge field itself. Only when the person isn't in nrtur does the salesperson fill the rest, using the fill panel.

- **Email from the record:** "Send email" on a **contact** page, the **Email** action on a **lead** page, and the **Email** action on a **deal** page open the in-app composer with **To already set** and the record attached. Picking a template fills every field at once. From a deal page, `deal_name` and `deal_amount` fill too, and the email goes to the deal's primary contact. (Before, the contact page's "Send Email" opened the computer's mail app.)
- **To suggestions:** typing two or more letters in **To** lists matching **contacts and leads** (name, email, company, and a Contact / Lead label). Picking one sets the address and the record, so every field fills.

- The composer **keeps the inserted template's merge fields live**. As soon as **To** matches a contact, and again whenever **To** changes, every field fills with that person's details.
- A green line confirms it: "✓ Merge fields filled for Sarah Chen".
- Fields that can't be filled show in an amber **fill panel**, one row per field. Each row has a box to type a value (with a **Use** button) and a **Remove** link. The panel's hint says "Add the recipient in To and these fill in automatically, or type a value now", or, once a recipient is set, "We couldn't fill these for this recipient. Type a value or remove them before sending."
- Send stays disabled while any merge field is left, as today.

## Rules
- **Which recipient:**
  - email opened from a record or picked from the suggestions: that contact, lead or deal (while **To** still holds that address);
  - email typed by hand: the contact, or else the lead, matched by the email address in **To** (the first address if several);
  - New SMS: the contact matched by name or phone in **To**;
  - SMS thread: the thread's contact or lead.
- **Values:**
  - first/last/full name, email, phone and company from the contact or lead;
  - owner first/full name from the record's owner;
  - `workspace_name` from the workspace;
  - `deal_name` and `deal_amount` only when the composer was opened from a deal page; anywhere else they have no value.
  - A field with no value uses its `{{key|fallback}}` fallback when it has one.
- **When To changes:**
  - **Message unchanged since the last fill:** everything is re-filled from the template for the new recipient. Switching from Sarah to James turns "Hi Sarah" into "Hi James". An unknown recipient puts the fields back.
  - **Message edited by the rep,** including a value typed into the fill panel: only the fields still left in the text are filled. **Text the rep typed is never overwritten.**
- **Fill panel:**
  - **Use** replaces every copy of that field in the subject and body with the typed value. The value is escaped in an email body.
  - **Remove** deletes every copy of the field.
- **Without a template:** the live link only exists after a template is inserted. A merge field typed by hand still appears in the fill panel and still blocks Send, as today.
- **Sending:** what's sent is exactly what's on screen. The backend changes nothing.

## Permissions
No change.

## Plan and pricing impact
None.

## Out of scope
- Server-side merge filling for inbox sends.
- Deal fields when the email isn't started from a deal page.
- Merge-field chips in the editor. They're part of the template editor upgrade proposal; this proposal works with or without them.

## Tickets (to create after approval)
- **FE:** live merge fields in the inbox composers.
  - Keep the template source after insert, and re-fill when the recipient changes, following the never-overwrite rule.
  - Replace the leftover-field note with the fill panel (Use / Remove).
  - Add the "filled for" line.
  - Cover the email composer, reply/forward, New SMS and the SMS thread.
  - "Send email" on contact, lead and deal pages opens the in-app composer with To and the record set (no more `mailto:`); deal fields fill from a deal page.
  - Match leads as well as contacts; a To suggestion list of contacts and leads.
  - Follows SCRUM-1283; frontend only.
- **BE:** none.
