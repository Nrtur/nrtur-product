# Billing and limits decisions (September 2026)

These are owner decisions taken while billing was built. The current numbers live in [../pricing/pricing.md](../pricing/pricing.md). This file records why they are what they are.

## 2026-09-09

- **Advertised send volumes are lowered to what can actually be delivered.** The old 1,000 / 25,000 / 100,000 automated sends per month could not be reached.
  - Why: sending is deliberately capped at 250 per mailbox per day (half of Gmail's practical floor) and 5,000 per workspace per day. Raising those caps would risk Google throttling or suspending customers' mailboxes.
  - Result: the pricing page changes, not the code. The numbers were set on 2026-09-11, see below.
- **A cancelled account becomes read-only.** It keeps its data but has zero limits, which is modelled as its own plan tier.
  - Accepted cost: the record of which plan they last paid for is lost, so win-back offers can't see it.

## 2026-09-10

- **A "user" (seat) is an accepted member.** You buy a seat, then invite someone. The seat is used only when the person accepts. Pending invitations don't count. Deactivated (suspended) members don't count either, and reactivating one is subject to the seat cap.
- **Mailbox and phone-line caps are additive:** a base allowance for the workspace, plus 2 for every additional member.
  - Example (mailboxes on Team): 1 user = 10, 2 users = 12, 3 users = 14.
  - The pricing page's old wording "10 or 2 per user" was wrong and was reworded.
  - **Superseded by SCRUM-1054.** Mailboxes and lines now have two numbers each: how many are *included* (1 per user), and the most you can *hold* by buying add-ons. For mailboxes, the most you can hold is the larger of the base or 2 per user, so Team with 2 users can hold 10, not 12. This matches the pricing page's "10 or 2 per user" wording. `pricing/pricing.md` has the current numbers.
- **The Business 5-user minimum is a recommendation, not a rule.** A 2-seat Business subscription is allowed and is billed for 2. The UI advises 5.
- **Annual plan, monthly add-ons.** An annual customer has two subscriptions: a yearly one for the plan and a monthly one for add-ons (mailboxes, lines, texting).

## 2026-09-11

- **Automated sends per month: Team 1,000 · Pro 7,500 · Business 35,000.** The trial gets Pro's 7,500.
- **A mid-cycle downgrade keeps the higher send allowance until the cycle ends.**
- **Cancelling the plan also cancels the add-on subscription**, immediately and without proration. A read-only workspace must not keep paying for mailboxes it cannot use.
- **One live plan per workspace.** Checkout refuses while a plan is live, including one set to cancel at period end.

## 2026-09-15 — plan switching

- **A downgrade takes effect immediately, with a proration credit** on the next invoice. "At period end" scheduling is out of v1 and can be added later.
- **An upgrade is charged immediately.** If the card is declined, the switch fails and the plan stays as it was.
- **A preview is shown before switching:** the amount charged now, or the credit for a downgrade, including tax.
- **What the first version refuses:**
  - changing billing interval (monthly ↔ annual)
  - switching while payment is past due
  - switching to a retired (grandfathered) price
  - switching to Solo while the workspace has extra users
- Only the workspace owner can switch plans.

## 2026-09-21 — trial and records

- **The trial stays at 21 days.**
- **Record limits are per record type:** contacts, leads, companies and deals each get the same number.
  - Trial 2,500 · Solo 5,000 · Team 50,000 · Pro 500,000 · Business 2,000,000 · read-only 0.
  - The owner may still switch to a ratio (companies and deals at 20%); that option is tracked in SCRUM-1127.
- **The trial holds nothing that costs money.**
  - It has no mailbox, no phone line and no sequences. Automations cannot send or enroll.
  - Simple automations still work: assign, task, flag, update field, tags, webhook, and create/convert records.
- **Sending needs at least one connected mailbox in the workspace.** The author doesn't need their own, because the sender is the record owner.
- **Trial export stays open.** The pricing page's "export unlocks with a card" line was dropped. Gating it would be a new small ticket if wanted.
- **All seven transactional emails use the same simple light theme.**

## Open (not yet decided)

- Trial export: leave it open, or gate it on adding a card?
- Records: keep the same number per type, or use the 20% ratio for companies and deals?
