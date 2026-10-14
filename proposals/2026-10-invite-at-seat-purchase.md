---
title: Invite teammates right after buying users
status: proposed        # proposed | approved | in-progress | built | dropped
owner: Qamar Ul Islam
area: docs/areas/billing.md, docs/areas/workspace-settings.md
screens: [settings-billing, settings-team]
tickets: [SCRUM-1225]
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
Offer **"Invite teammates"** at the three places where users are bought or visible. Each one opens the **existing** Invite member dialog on Settings › Team. No new invite UI and no backend change.

**Screens** (design.nrtur.io):
- `settings-billing` › Subscription Details › Seats:
  1. **After "Add users" succeeds**, the dialog's done state becomes "Users added". It says "Your plan is now N users" and "You have K open seats. Invite people now, or any time from Settings › Team. Each invitation holds a seat until it's accepted or revoked." Buttons: **Later** and **Invite teammates**.
  2. **Under the Seats field**, whenever seats are open: "K open seats · **Invite teammates**".
- **Return from Stripe checkout** (the app's `/billing/success` page; in the design, a dialog on `settings-billing` after "Continue to checkout"):
  3. "Payment received — welcome to <plan>". When seats are open: "You bought N users, and K seats are open. Invite your team now, or any time from Settings › Team." Buttons: **View billing** and **Invite your team**.
- `settings-team`: a deep link (`/settings?tab=team&invite=1`) opens the Invite member dialog straight away.

## Rules
- **Open seats** = seats on the plan − (accepted members + live pending invitations). A pending invitation holds a seat (`checkInviteCap`). This is the same count the Invite dialog's seat line already uses (`GET /settings/usage`).
- **Showing the invite offer:**
  - The "Invite teammates" / "Invite your team" buttons and the "K open seats" line appear **only when K > 0**.
  - With no open seats, the done state says every seat is held, and the only button is Done / View billing.
- **Removing users** doesn't show the invite offer.
- **Solo** sells no seats and can't invite, so none of this appears on Solo.
- **The deep link** opens the dialog only for owners and admins, the same people who can invite today. Anyone else lands on Team with no dialog.
  - It opens once per click. Coming back to Team later doesn't reopen it.
- **The Invite member dialog doesn't change:** one email and a role (Admin or Member), its seat line, and its refusals (`PLAN_LIMIT_REACHED` and so on). To fill several seats, the owner sends several invites, as today.
- **After Stripe checkout** the new seat count may arrive a moment later, because the webhook may not have run yet.
  - The return page reads usage when it loads. If the count hasn't landed, it shows the plan as active without the open-seats line.
  - The Invite dialog always reads live usage when it opens, so its seat line is right either way.
- **Who sees it:** only the owner. Buying users and the Subscription Details card are owner-only.

## Permissions
No change. Owners buy users; owners and admins invite.

## Plan and pricing impact
None. No price, limit or plan changes.

## Out of scope
- Inviting several people at once (a multi-email invite form).
- Putting an invite step back into onboarding.
- Any backend change. The invite and seat endpoints already exist.
- Inviting during a trial from Billing. On the trial (3 seats) the Team page already invites, and buying users happens only through checkout.

## Tickets
- **FE:** SCRUM-1225 (already exists). The onboarding half is done (SCRUM-1090). What's left:
  - the "Users added" done state with Invite teammates in `SeatDoorDialog`;
  - the "K open seats · Invite teammates" line on `SubscriptionDetailsCard`;
  - "Invite your team" on `BillingSuccessNotice` when seats are open;
  - `?invite=1` on Settings › Team opening `InviteMemberDialog`.
- **BE:** none.
