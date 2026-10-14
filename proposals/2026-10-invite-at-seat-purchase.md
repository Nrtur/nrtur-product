---
title: Invite teammates right after buying users (plus billing copy fixes)
status: proposed        # proposed | approved | in-progress | built | dropped
owner: Qamar Ul Islam
area: docs/areas/billing.md, docs/areas/workspace-settings.md
screens: [settings-billing, settings-team, settings-phone-numbers]
tickets: [SCRUM-1225 (invite at seat purchase), SCRUM-1229 ("line" → "phone number"), SCRUM-1230 (shorter top-up text)]
created: 2026-10-14
---

# Invite teammates right after buying users

## Problem
From the Oct-1 demo: people skipped the invite step in onboarding, so it was removed. The onboarding wizard is now Workspace › Pipeline › Contacts › Done (SCRUM-1090, already built; `auth-and-onboarding.md` already says so).

What's still missing is the moment that matters: when an owner **buys users**, nothing invites anyone into the new seats. They have to know to go to Settings › Team afterwards.
- The seat dialog ends on "Done".
- The page after Stripe checkout only links to Billing.

Paid seats sit empty.

## Proposal
Offer **"Invite teammates"** at the three places where users are bought or visible. Each one opens the Invite dialog on Settings › Team, which now takes **several people at once**. No backend change.

**Screens** (design.nrtur.io):
- `settings-billing` › Subscription Details › Seats:
  1. **After "Add users" succeeds**, the dialog's done state becomes "Users added". It says "Your plan is now N users" and "You have K open seats. Invite people now, or any time from Settings › Team. Each invitation holds a seat until it's accepted or revoked." Buttons: **Later** and **Invite teammates**.
  2. **Under the Seats field**, whenever seats are open: "K open seats · **Invite teammates**".
- **Return from Stripe checkout** (the app's `/billing/success` page; in the design, a dialog on `settings-billing` after "Continue to checkout"):
  3. "Payment received — welcome to <plan>". When seats are open: "You bought N users, and K seats are open. Invite your team now, or any time from Settings › Team." Buttons: **View billing** and **Invite your team**.
- `settings-team`:
  - the **Invite dialog** takes several people (below);
  - a deep link (`/settings?tab=team&invite=1&rows=K`) opens it straight away with K empty rows.

**The Invite dialog, several people at once** (replaces today's one-email dialog everywhere):
- Title "Invite teammates". The seat line reads "6 of 9 seats in use on the Pro plan · 3 open".
- One row per person, each with an email and a role (Admin or Member; Member by default).
  - From the Team page it opens with **1 row** and an **"+ Add another"** link.
  - From Billing it opens with **one empty row per open seat**, so buying 3 users gives 3 rows.
  - Rows can't exceed the open seats (at most 25). At that point "One row per open seat." replaces "Add another".
  - Each row can be removed with ×. **Blank rows are skipped.**
- The button reads "Send invite" or "Send N invites", counting the filled rows.
- **Checks before sending, per row:**
  - a valid email;
  - not already a member;
  - not already invited (pending);
  - not the same email twice in the form.

  A row that fails shows its reason and isn't sent.
- **Sending:** the rows go **one after another** to the existing `POST /workspaces/{id}/invite`, with a short gap (about 400ms) between requests.
  - Each row shows a spinner while sending, then ✓.
  - A row the API refuses keeps its reason. For example, `PLAN_LIMIT_REACHED` shows "No seat left for this invite. Add a user in Billing, or revoke a pending invite."
- **Afterwards:**
  - If everything was sent, the dialog closes with "N invites sent".
  - Otherwise the sent rows clear and the failed rows stay, with the toast "2 of 3 invites sent".
- **Closing:** "Skip for now" (from Billing) or "Cancel" (from Team) closes it at any time, except while sending.

**Billing copy fixes** (same screen):
- **SCRUM-1229, "line" → "phone number" everywhere a customer reads it:**
  - "Extra local lines" → "Extra local numbers" and "Toll-free lines" → "Toll-free numbers" (checkout and add-ons);
  - "Lines: N" → "Phone numbers: N" (Subscription Details);
  - "mailboxes and lines" → "mailboxes and phone numbers";
  - the Phone numbers page's buy prompts ("Add a local number for $5 a month").

  Stripe and the code keep saying "line".
- **SCRUM-1230, shorter Add credit dialog:**
  - The subtitle becomes "Charged to your card on file. Minimum $20."
  - The three lines of fine print become one: "Used for calls and texts, sent and received. Non-refundable."
  - A **"How credit works"** link expands the full rules:
    - received calls and texts cost the same;
    - when the balance is empty, sending stops and receiving continues;
    - credit doesn't expire while the account is open, and stays usable for 12 months after cancelling.

## Rules
- **Open seats** = seats on the plan − (accepted members + live pending invitations). A pending invitation holds a seat (`checkInviteCap`). This is the same count the Invite dialog's seat line already uses (`GET /settings/usage`).
- **Showing the invite offer:**
  - The "Invite teammates" / "Invite your team" buttons and the "K open seats" line appear **only when K > 0**.
  - With no open seats, the done state says every seat is held, and the only button is Done / View billing.
- **Removing users** doesn't show the invite offer.
- **Solo** sells no seats and can't invite, so none of this appears on Solo.
- **The deep link** opens the dialog only for owners and admins, the same people who can invite today. Anyone else lands on Team with no dialog.
  - It opens once per click. Coming back to Team later doesn't reopen it.
- **The invite rules don't change:** roles, the seat cap, rate limits and the refusals stay as they are. The dialog sends the same request once per row.
- **After Stripe checkout** the new seat count may arrive a few seconds later, because the webhook may not have run yet.
  - The return page shows **"Invite your team" whenever the plan bought has more than 1 user**, without waiting for the count.
  - The return page re-reads usage a few times, so the "K seats are open" line appears once the count lands.
  - The Invite dialog reads live usage when it opens, so its seat line and row limit are always right.
- **Who sees it:** only the owner. Buying users and the Subscription Details card are owner-only.

## Permissions
No change. Owners buy users; owners and admins invite.

## Plan and pricing impact
None. No price, limit or plan changes.

## Out of scope
- Putting an invite step back into onboarding.
- Any backend change. The invite and seat endpoints already exist.
- Inviting during a trial from Billing. On the trial (3 seats) the Team page already invites, and buying users happens only through checkout.

## Tickets
- **FE:** SCRUM-1225 (already exists). The onboarding half is done (SCRUM-1090). What's left:
  - the "Users added" done state with Invite teammates in `SeatDoorDialog`;
  - the "K open seats · Invite teammates" line on `SubscriptionDetailsCard`;
  - "Invite your team" on `BillingSuccessNotice` when more than 1 user was bought, with a usage re-read;
  - `InviteMemberDialog` becoming multi-row (sequential sends, per-row status);
  - `?invite=1&rows=K` on Settings › Team.
- **FE:** SCRUM-1229 (already exists): the "line" → "phone number" copy.
- **FE:** SCRUM-1230 (already exists): the shorter Add credit dialog.
- **BE:** none.
