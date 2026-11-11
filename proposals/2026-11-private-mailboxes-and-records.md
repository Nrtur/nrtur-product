---
title: Private mailboxes and private records
status: approved        # proposed | approved | in-progress | built | dropped
owner: Qamar Ul Islam
area: docs/areas/inbox.md, docs/areas/leads.md, docs/areas/contacts.md, docs/areas/deals-and-pipeline.md, docs/areas/companies.md, docs/areas/workspace-settings.md (Integrations, Team), docs/permissions.md, pricing/pricing.md
screens: [inbox, settings-integrations, leads, lead-detail, add-lead, contacts, contact-detail, add-contact, pipeline, deal-detail, add-deal, company-detail]
tickets: [7 tickets created 2026-11-11, Jira label `privacy`: Phase 1 BE + FE; Phase 2 BE 1-3 + FE 1-2]
created: 2026-11-11
---

# Private mailboxes and private records

## Problem
Everything in nrtur is shared today: every member sees every connected mailbox, every email in it, and every lead, contact and deal (`docs/permissions.md`: "There is no record-level visibility").

- **Mailboxes.** A rep who connects their own work email exposes all of it to the whole team: HR, payroll, private conversations. Reps won't connect their email on those terms, and then nrtur loses their customer emails too.
- **Records.** Some teams need a rep's leads, contacts and deals to stay with that rep: commission-based teams, agencies with a client per rep, or a sensitive deal. Today they can't.

Other CRMs (HubSpot, Pipedrive, Salesforce) separate the two: a **private inbox** that nobody else can read, and **private records** that only the owner (and the managers) can see.

## Proposal
Two parts, built in two phases. Each phase ships on its own.

**Phase 1: Private mailboxes.** Each connected mailbox is **Private** (only the person who connected it) or **Shared** (the whole team).

**Phase 2: Private records.** Leads, contacts and deals get a **Shared / Private** switch. A private record is visible only to its owner, plus the workspace owner and admins.

**Screens** (design.nrtur.io). A **Privacy** preview toggle on each screen switches between the states below.
- `settings-integrations` and `inbox`: the Private / Shared choice when connecting a mailbox, a badge on every mailbox, and mailbox settings.
- `inbox`: the **Private** switch in the thread reader, and a lock on private threads.
- `inbox`: the **Mailbox** filter above the email list, and **Connect Gmail** on the empty Inbox opening the same Private / Shared choice.
- `leads`, `contacts`, `pipeline` and their detail and add pages: the **Visibility** field, a lock icon on private records, and a "Private" list filter.
- `company-detail`: the "N private records hidden" line.
- `add-lead` / `add-contact`: the message for a private duplicate.

## Rules

### Phase 1: Private mailboxes

**Choosing**
- When connecting a mailbox, the person picks the options below. This choice appears wherever you can connect a mailbox: Settings › Integrations, the Inbox's **Connect account** link, and the **Connect Gmail** button on the empty Inbox.
  - **Private: only me.** For your own work email. **This is the default.**
  - **Shared: the whole team.** For a team address like `sales@` or `support@`. **Only an owner or admin** can choose Shared; a member's mailbox is always private.
- Every mailbox shows a badge: 🔒 **Private** or 👥 **Shared**, in the Inbox folder rail and in Settings › Integrations.
- Nrtur isn't launched yet, so there's no migration: existing test mailboxes become private (or the data is reset).

**Who can see a private mailbox**
- **Only the person who connected it.** It doesn't appear for anyone else in the Inbox: not in the mailbox list, the All and Email tabs, folder counts, search, threads, attachments or the From picker.
- **Not even the workspace owner or admins can read it.** They can see that it exists in Settings › Integrations (email address, who connected it, status) so they can manage seats, and they can **disconnect** it. They can't open its emails.
- The API refuses another person's request for a private mailbox's threads, messages or attachments with **404**, not 403, so nothing reveals that a thread exists.

**Who can see a shared mailbox**
- Every member, as today: read, reply, forward, label, archive.

**Mailbox filter in the Inbox**
- The Inbox stays unified: by default it shows every mailbox the person can read (their own private ones plus the shared ones).
- A **Mailbox** dropdown above the email list narrows it to one mailbox: "All mailboxes", then each mailbox with its Private / Shared label. It lists only the mailboxes the person can read, and it shows only when they can read two or more.
- The API gets an optional `account_id` filter on the thread list, the All feed and the folder counts. Asking for a mailbox the person can't read returns nothing, the same as the 404 rule above.

**Customer emails on record timelines**
- An email in a **private** mailbox that **matches a contact or lead** still shows on that record's timeline, to anyone who can see the record. The team keeps the customer history; the rest of the mailbox stays private.
- **Emails that match no contact or lead** (HR, payroll, newsletters, internal mail) are never shown to anyone else.
- **Per-thread "Private" switch:** in the thread reader, the mailbox owner can mark any thread **Private**. It then disappears from timelines for everyone else, and the timeline shows nothing in its place. Switching it back shows it again.
- **Per-mailbox default:** in the mailbox settings, "Show matched customer emails on timelines" is **on** by default. Turning it off makes every thread private unless the owner shares it one by one.
- Emails in a **shared** mailbox always show on matched timelines, as today. Shared mailboxes have no per-thread Private switch.

**Sending**
- Only the mailbox owner can send from a private mailbox in the Inbox; others don't see it in From.
- **Automations and sequences** can still send from a rep's private mailbox (for example "From: deal owner"). Those emails are customer emails, so they show on the record's timeline. Replies arrive in the rep's private mailbox and show on the timeline the same way.

**Changing**
- **Shared → Private:** an owner or admin. The team loses access to the mailbox at once.
- **Private → Shared:** only the person who connected it, after a confirm: "Everyone in the workspace will be able to read every email in this mailbox." **No one else can make someone's private mailbox shared**, not even the owner. An owner or admin can only disconnect it.

**When a member is removed**
- Their **private** mailboxes are **disconnected** automatically, and their synced emails deleted, as Disconnect does today. Emails already shown on customer timelines stay as timeline entries.
- Their **shared** mailboxes stay connected; they now show "Connected by: removed member". An owner or admin can reconnect it from their own Google account or disconnect it.

### Phase 2: Private records

**Which records**
- **Leads, contacts and deals** can be **Shared** (default) or **Private**.
- **Companies are always shared.** Everyone can see a company's name, domain and owner, so two reps never create the same company twice. Inside a company page, private contacts, deals and activity that the viewer can't see are hidden, with a line "2 private records hidden".

**Who can see a private record**
- **Its owner**, plus the **workspace owner and admins**. Owners and admins need the whole pipeline and forecast, and need to reassign records when someone leaves.
- **Nobody else.** For them the record doesn't exist anywhere: lists, the pipeline board, the detail page (404), global search, export, saved views, counts, the dashboard, the forecast and reports, the activity feed, notifications, live updates, the duplicates page, and the To suggestions in the composer.
- **Everything attached follows the record:** timeline, notes, tasks, calls, SMS, files and emails. A task on a private deal is visible only to people who can see the deal, even if the task is assigned to someone else. Assigning a task on a private record to someone who can't see the record is refused: "Make the deal shared first, or assign the task to someone who can see it".

**Setting it**
- A **Visibility** field (Shared / Private) on the add form and the detail page of leads, contacts and deals. Default: Shared.
- **Who can change it:** the record owner, an owner or an admin.
- **A private record must have an owner.** Making a private record **Unassigned** asks first and turns it back to Shared.
- **Changing the owner of a private record** keeps it private, so the new owner sees it and the old one doesn't. The confirm says so.
- **Bulk edit:** owners and admins can set visibility for many records at once. Members can bulk edit only their own records.
- **Imports, the Lead API and Facebook Lead Ads** create **shared** records, as today.

**Lead conversion**
- Converting a private lead makes the new **contact and deal private too**, with the same owner. The company is shared as always.
- If conversion links to an **existing shared contact**, that contact stays shared.

**Duplicates and "already exists" messages**
- The unique rules stay workspace-wide: there's still only one lead or contact per email and phone, private or not.
- If someone adds an email or phone that belongs to **a private record they can't see**, the save is refused with **no link and no details**: "A private lead with this email already exists, owned by Sarah Chen. Ask them or an admin to share or reassign it." The record owner's name is shown, nothing else.
- The **Lead API and Facebook** duplicate handling is unchanged: they work as the workspace, not as a person.
- The **duplicates page** shows a member only groups where they can see every record. Owners and admins see everything.

**Automations, sequences and lead scoring**
- These run on private records like on any other. They work for the workspace, not for a person.
- An automation that notifies or assigns a task to someone who can't see the record notifies the record owner instead.

**Dashboard, forecast and reports**
- A member's numbers leave out private records they can't see. Owners and admins see the full totals.

**When a member is removed**
- Their private records go to the person their work is reassigned to (the existing `reassign_to`). With no reassignment, they become **Unassigned and Shared**.

**SMS and calls**
- Phone numbers stay shared. An SMS conversation or call linked to a private contact or lead is visible only to people who can see that record, in the Inbox and on the timeline. A text or call from a number that matches no record is shared, as today.

## Permissions

| | Owner | Admin | Member |
|---|:-:|:-:|:-:|
| Connect a **private** mailbox | ✓ | ✓ | ✓ |
| Connect a **shared** mailbox, or make a mailbox shared → private | ✓ | ✓ | — |
| Read and send from a **shared** mailbox | ✓ | ✓ | ✓ |
| Read and send from a **private** mailbox | own only | own only | own only |
| Make a private mailbox shared | own only | own only | own only |
| Mark a thread Private, change the mailbox's timeline default | own mailbox | own mailbox | own mailbox |
| See that a private mailbox exists (address, who, status) | ✓ | ✓ | own only |
| Disconnect a mailbox | any | any | — (as today) |
| See and work on **shared** leads, contacts and deals | ✓ | ✓ | ✓ |
| See and work on **private** leads, contacts and deals | all | all | own only |
| Make a lead, contact or deal private or shared | any | any | own only |

"Own" for a mailbox means the person who connected it. For a record it means the record's owner.

## Plan and pricing impact
- **Private mailboxes: every plan that has mailboxes** (Solo, Team, Pro, Business). Reps won't connect their email without it, so it isn't a selling point to hold back. Trial has no mailboxes, as today. Private and shared mailboxes count the same toward the mailbox limit.
- **Private records: Pro and Business**, plus **Trial** (which has Pro's features). A new plan feature, `private_records`.
  - On Solo and Team, the Visibility field is hidden. Members see a "Private records are on Pro and Business" note on the record page, linking to Billing for the owner.
  - **On a downgrade** to a plan without it, private records **stay private** (nothing is exposed by a plan change), but no new record can be made private. A private record can still be made shared.
- `pricing/pricing.md` gets the new feature row. No price or limit changes.

## Out of scope
- **Teams, groups or territories** ("the EMEA team sees EMEA records"). Visibility is only Shared or Owner-only.
- **Hiding single fields** (for example a deal's amount).
- **Private companies.**
- **Private phone numbers** and per-rep SMS lines.
- **Creating a lead from an email** in the Inbox, and auto-creating leads from unknown senders. That gets its own proposal.
- **Sharing a private record with one named person.**
- An owner or admin reading a private mailbox, for any reason.
- **A "by user" filter and assigning threads** in the Inbox. nrtur has neither today: threads have no assignee ("Assigned to me" exists only in Notifications). With private mailboxes, a person's own mail is already their private mailbox, so the Mailbox filter covers "show me one person's mail". Assigning shared-mailbox threads to a teammate (as in a shared `sales@` inbox) is a separate feature and would get its own proposal.
- The mobile app (it uses the same API, so it gets the filtering automatically, but its screens are not designed here).

## Phases and tickets (to create after approval)

**Phase 1: Private mailboxes**
- **BE:**
  - `visibility` on mailboxes, set at connect;
  - filtering in every inbox read: accounts, feed, threads, messages, attachments, counts, search, From;
  - the optional `account_id` filter on the thread list, feed and counts;
  - timeline filtering, the per-thread Private switch and the mailbox's timeline default;
  - the change rules;
  - disconnecting private mailboxes on member removal.
- **FE:**
  - the Private / Shared choice on connect, including from the empty Inbox;
  - the Mailbox filter above the email list;
  - badges;
  - mailbox settings (visibility, timeline default);
  - the thread Private switch;
  - the private → shared confirm.

**Phase 2: Private records**
- **BE 1, the access rule:**
  - one shared "can this person see this record" check;
  - `visibility` on leads, contacts and deals, and the `private_records` plan feature;
  - lists, detail, the pipeline board, search, export, saved views, counts, bulk edit, conversion, owner change and Unassigned.
- **BE 2, everything attached:**
  - timelines, notes, tasks, calls, SMS, files, the activity feed;
  - notifications, live updates, the composer's To suggestions;
  - company pages;
  - member removal.
- **BE 3, the totals and duplicate checks:**
  - the private duplicate message;
  - the duplicates page;
  - dashboard, forecast and reports;
  - automations' notify and assign fallback.
- **FE 1:** the Visibility field (add and detail pages, bulk edit), lock icons, the Private filter and the plan-gated note.
- **FE 2:** the private duplicate message, the hidden-records line on company pages, conversion and owner-change confirms.
- **Every BE ticket ends with a two-person test:** member B must not find member A's private mailbox or record through any endpoint.

## Decisions (owner, 2026-11-11)
1. Owners and admins **can** see private leads, contacts and deals. They **can't** read private mailboxes.
2. Matched customer emails from a private mailbox **show on timelines by default**, with a per-thread Private switch.
3. Not launched yet, so no migration for existing mailboxes. New mailboxes default to Private.
4. Private mailboxes on every plan with mailboxes; private records on Pro, Business and Trial.
5. Creating leads from emails is a separate proposal.
