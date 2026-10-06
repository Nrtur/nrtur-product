---
title: nrtur pricing
status: source of truth
verified_against: frontend@1bb6aa5d, backend@577bae44
verified_on: 2026-10-06
---
# nrtur pricing

This is the single source of truth for plans, prices, limits, usage rates and the trial. Where the code and the older pricing spec (`nrtur-docs/nrtur-pricing.html`, "v4", 2 September 2026) disagree, the code wins. Each disagreement is listed under [Discrepancies resolved](#discrepancies-resolved).

Where each number comes from:
- **Limits, features and bundles:** `internal/platform/entitlements/catalog.go` in the backend. It holds no prices, by design.
- **Usage rates, top-up floor and welcome credit:** `internal/platform/wallet/money.go` and `internal/modules/billing/service/welcome_credit.go`.
- **Plan, extra-user and add-on list prices:** the frontend (`src/features/billing/components/BillingSection.tsx`, `lib/checkoutCatalog.ts`). The amount actually charged is whatever the Stripe price resolves to, and there is no catalog endpoint, so these are list prices. Stripe price IDs are deliberately not recorded here.

All prices are in US dollars and exclude tax. Existing customers keep their price for 12 months after any change. The app says so under the plan grid, and the backend accepts a list of price IDs per plan so older prices keep working.

## Plans and prices

| | Solo | Team | Pro | Business |
|---|---|---|---|---|
| Monthly, per month | $15 | $45 | $85 | $145 |
| Yearly, per month | $12 | $35 | $69 | $119 |
| Yearly saving | 20% | 22% | 19% | 18% |
| Each extra user, monthly | not available | $37 | $68 | $115 |
| Each extra user, yearly (per month) | not available | $29 | $55 | $95 |
| Users included in the price | 1 | 1 | 1 | 1 |
| Most users | 1 (cannot invite) | no ceiling | no ceiling | no ceiling ("5+ recommended" is copy, not a rule) |
| Welcome wallet credit (once, first paid invoice) | none | $15 | $40 | $75 |

How billing works:
- Plans are billed monthly or yearly.
- Extra users are bought in the same billing interval as the plan.
- Add-ons are always monthly. A yearly customer who buys add-ons gets a second, monthly subscription for them.
- `starter` is a legacy internal name for Team, with identical limits. It is not sold.

## Limits per plan

Contacts, leads, companies and deals are capped **separately**: each type has its own limit.

| Limit | Trial | Solo | Team | Pro | Business |
|---|---|---|---|---|---|
| Contacts | 2,500 | 5,000 | 50,000 | 500,000 | 2,000,000 |
| Leads | 2,500 | 5,000 | 50,000 | 500,000 | 2,000,000 |
| Companies | 2,500 | 5,000 | 50,000 | 500,000 | 2,000,000 |
| Deals | 2,500 | 5,000 | 50,000 | 500,000 | 2,000,000 |
| Seats | 3 | 1 | 1 + purchased | 1 + purchased | 1 + purchased |
| Pending invitations | unlimited | 0 | unlimited | unlimited | unlimited |
| Mailboxes included | 0 | 1 | 1 per user | 1 per user | 1 per user |
| Most mailboxes you can hold | 0 | 2 | 10 or 2 per user, whichever is more | 20 or 2 per user, whichever is more | 50 or 2 per user, whichever is more |
| Phone lines included | 0 | 0 | 1 per user | 1 per user | 1 per user |
| Most phone lines you can hold | 0 | 1 | 2 per user | 2 per user | 2 per user |
| File storage | 25 GB per user | 2 GB per user | 10 GB per user | 25 GB per user | 100 GB per user |
| Pipelines | 25 | 1 | 3 | 25 | 100 |
| Stages per pipeline | unlimited | unlimited | unlimited | unlimited | unlimited |
| Custom fields per record type | 300 | 50 | 150 | 300 | 500 |
| Active automations | 50 | 1 | 5 | 50 | 200 |
| Active sequences | 0 | 0 | 0 | unlimited | unlimited |
| Total flows (incl. drafts, paused) | unlimited | unlimited | unlimited | unlimited | unlimited |
| Tags, folders, inbox labels | unlimited | unlimited | unlimited | unlimited | unlimited |
| **Automated sends per month** | 0 | 0 | 1,000 | **7,500** | **35,000** |

How the limits are measured:
- "Per user" counts accepted members.
- Storage uses decimal gigabytes (1 GB = 1,000,000,000 bytes).
- The mailbox and line cap is the smaller of the hold and (included + purchased add-ons).
- An automated send is one email or text sent by an automation or a sequence. Manual email is not capped.
- The send allowance resets each billing cycle. For a trial the cycle is anchored on the signup day.

**`read_only`** is what a cancelled or expired workspace becomes. Every limit above is 0 and every feature below is off. All existing data is kept, but nothing new can be created.

## Features per plan

| Feature | Trial | Solo | Team | Pro | Business |
|---|---|---|---|---|---|
| Automations can send email and texts | no | no | yes | yes | yes |
| Sequences | no | no | no | yes | yes |
| Lead scoring (rules editor) | yes | no | no | yes | yes |
| Duplicate detection and merge | yes | no | yes | yes | yes |
| API access and webhooks | yes | no | no | yes | yes |
| Single sign-on | no | no | no | no | yes |
| Audit log | no | no | no | no | yes |

API access, SSO and audit log are set per tier but have no product surface yet.

Enforcement: every limit, feature and the automated-sends meter is **enforced** (`entitlements.Boot()`):
- **Records on paid plans** warn at 80%. The save that crosses the line is allowed, and every save beyond it is refused. Imports stop at the line.
- **Records on trial and read-only** are refused at the line.
- **Any other cap** is refused at the create that would exceed it, with the remedy upgrade, add a user, or buy an add-on.
- **Downgrading** deletes nothing. Over-limit items are kept, but new ones are refused until the workspace is back under the limit.

## Add-ons (monthly)

| Add-on | Price per month | Notes |
|---|---|---|
| Extra mailbox | $5 | Above the included allowance, up to the plan's hold |
| Extra local phone line | $5 | Solo includes none but may buy one |
| Toll-free line | $8 | None included on any plan |
| Texting | $9 | Per workspace. **Not on sale**: refused until carrier (A2P) registration exists |
| Texting, high volume | $29 | **Not on sale**, as above |

## Usage: the prepaid wallet

Calls and texts are paid from a prepaid balance (rate card `v1`).

| Item | Rate |
|---|---|
| Text message (SMS) | $0.020 per segment (160 characters, or 70 with an emoji) |
| Picture message (MMS) | $0.050 per message |
| Calling | $0.030 per minute of the call, rounded up |
| Inbound calls to a toll-free number | $0.050 per minute |

Wallet rules:
- **Top-up amounts:** minimum $20, maximum $999,999.99 per top-up, whole cents. Presets in the app are $20, $50 and $100.
- **Who can top up:** only workspaces on a paid plan. A trial cannot buy credit.
- **Running out:** when the balance cannot cover a send, the send is refused. A charge the balance could not cover is shown as an unpaid charge and is not billed later.
- **Refunds:** balance is never refunded to a card. A send that did not happen is reversed back into the wallet.
- **Rate changes:** every charge records the rate card version and the unit price, so a later rate change never rewrites history.
- **Welcome credit:** granted once per workspace on the first paid plan invoice. It is not granted again on a resubscribe or a plan change.

## Trial

- **Length:** 21 days from signup (`trial_ends_at` = signup + 21 days).
- **What it includes:**
  - up to 3 users
  - 2,500 of each record type
  - otherwise Pro's counts and features
  - except **no** mailbox, phone line, sequences or automated sends: these need a paid plan
- **Card:** none is needed to start.
- **When it ends:** a background job moves the workspace to `read_only` once `trial_ends_at` passes. Data is kept and the app shows "Your trial ended on {date}". Buying a plan restores access.

## Payment failure and cancellation

- **Card failing (`past_due`):** the workspace keeps its paid tier, so access is unchanged while Stripe retries the card.
- **First payment not yet succeeded (`incomplete`):** no tier is granted.
- **Plan subscription deleted:** the workspace becomes `read_only` and its data is kept.
- **Removing users:** the seat quantity drops immediately with no refund. The removed users stay paid through the current period end, so billing for them effectively stops at the next renewal.
- **Adding users or add-ons:** charged now for the rest of the period.

## Discrepancies resolved

The old spec is `nrtur-docs/nrtur-pricing.html` (v4, 2 Sep 2026). In every case the code is the truth.

| Topic | Old HTML said | Code says (truth) |
|---|---|---|
| Automated sends per month, Pro | 25,000 | **7,500**. Matches the ground rule. |
| Automated sends per month, Business | 100,000 | **35,000**. Matches the ground rule. |
| Business users | "5 or more" users; "Business starts at five users" | No minimum. Seats are 1 + purchased like Team and Pro. "5+ users recommended" is display copy only. |
| Trial length | 21 days | 21 days. Matches; no change. Note: the prototype's landing and signup pages say 14 days, which is wrong. |
| Trial mailbox and line | No mailbox, no phone line (§03) | Matches: 0 / 0. The 2026-09-18 internal handoff guide still says 1 / 1, which is out of date. |
| Records limit | "each of contacts, leads, companies and deals" in §02; "2,500 records of each type" for the trial | Matches: four separate caps. The 2026-09-18 internal guide describing one pooled `records` cap is out of date. |
| Data migration (Business, "1 source, 8 hrs") | Listed as a plan feature | Not in the catalog or the product. Removed from the pricing source of truth. |
| Support level per plan (email / priority email / phone + named contact) | Listed in §02 | Not modelled in code. The app's plan cards still mention priority email (Pro) and phone support (Business) as copy only. |
| Auto top-up | "Auto top-up is optional and off by default", with a customer-set monthly ceiling | **Not built.** There is no endpoint or UI; only a ledger source name exists. |
| Texting add-on | On sale at $9 / $29, with a three-month minimum | **Not on sale.** Checkout refuses it until A2P registration exists. No three-month minimum is implemented. |
| Enforcement status (§10) | "enforcement is on" | Matches: everything is in enforce mode. Some comments in `catalog.go` still say "watch" and are out of date. |
| What happens when the trial ends | "read-only for 12 months, with an export link, then it is deleted" | Read-only: **verified**. 12-month retention, export link and deletion: **unverified**. No such job or flow was found in the code. |
| Failed payment grace | "14 days of full access, then read-only, then deletion at 12 months" | Full access while `past_due`: **verified**. The **14-day length is unverified**: it is Stripe's retry and dunning configuration, not code. Read-only follows only when Stripe cancels the subscription. Deletion at 12 months: **unverified**, not found. |
| Removing users | "the seat stops billing at the next renewal, not immediately" | **Verified**, with a nuance. The quantity drops immediately with no refund, so the seat is paid through period end and is not billed at renewal. |
| Mailboxes and lines left over after removing a seat | Roll to $5 a month rather than disconnecting | **Verified.** Extras above the new included allowance are billed as add-ons on the next invoice. |
| Wallet balance after cancellation | Usable for 12 months | The app's top-up copy says so. No expiry logic was found in code, so the balance simply persists: **unverified** as a rule. |
| Inbound messages and minutes drawing from the wallet | "Incoming … draw from it too"; incoming is free once the balance is empty | **Verified.** Inbound is settled against the wallet. When the balance is empty it is still received, and what could not be covered is recorded as an unpaid charge that is never collected. |
| What a failed payment does to the plan | Read-only after the grace period | The code only goes read-only when Stripe **deletes** the subscription. An `unpaid` status keeps the paid tier. So the actual timing depends entirely on Stripe's dunning settings. |
| Refunds on plans (annual pro rata within 30 days, monthly current month) | Listed in §06 | Not implemented in code. It would be a manual or Stripe-side policy: **unverified**. |
| Prototype billing page prices | Starter free / Pro $59 / Business $99 (prototype `BILLING_PLANS`); landing Business $149 | Wrong. The real plans are Solo, Team, Pro and Business at the prices above. |
