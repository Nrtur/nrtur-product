---
area: Billing
status: built
verified_against: frontend@1bb6aa5d, backend@577bae44
verified_on: 2026-10-06
---
# Billing

## What it does
Billing is how a workspace pays nrtur. It is nrtur's own SaaS billing. It is not the customer-facing "Payments" feature (invoices, quotes and payment links for your own customers), which is not built.

A new workspace starts on a 21-day trial. The owner then buys one of four plans (Solo, Team, Pro or Business), paid monthly or yearly through Stripe Checkout. After that the owner can switch plan, buy extra users and add-ons (mailboxes and phone lines), and manage the card and invoices in the Stripe customer portal.

Calls and texts are paid from a prepaid wallet that the owner tops up. The backend holds every plan limit in one catalog and enforces it on every create. The app warns before a limit is hit and shows the right remedy: upgrade, add a user, or buy an add-on.

A cancelled plan or an expired trial makes the workspace read-only. Nothing is deleted. Everything lives on one Settings tab, "Billing & usage", plus app-wide banners and two Stripe return pages.

The full price and limit table is in [`pricing/pricing.md`](../../pricing/pricing.md).

## Screens
| Route (app) | Prototype page id | Purpose |
|---|---|---|
| `/settings?tab=billing` ("Billing & usage") | `settings-billing` | Shown top to bottom: wallet balance, wallet history and unpaid charges; plan usage; subscription details; invoices; the plan grid (owner only); add-ons (owner only). |
| `/billing/success` | — | Stripe return page ("Payment received") for a plan checkout or a wallet top-up. With no `session_id` it redirects to the billing tab. |
| `/billing/cancel` | — | Stripe return page ("Checkout canceled"). Nothing was charged. |
| App-wide banner strip (`GlobalBannerHost` + `TrialBanner` + `ReadOnlyBanner`, in `components/layout/app-shell.tsx`) | banner presets in the prototype | Shows trial countdown, trial ended and read-only/cancelled states. |
| Dialogs on the billing tab | — | `PlanCheckoutDialog`, `PlanSwitchDialog`, `SeatDoorDialog`, `AddonOfferDialog`, top-up dialog. |

## Behaviour and rules
**Plans and checkout**
- There are four buyable plans: `solo`, `team`, `pro` and `business`, each monthly or yearly.
- The plan grid lists list prices hardcoded in `BillingSection.tsx`, because there is no plan-catalog endpoint. Stripe charges whatever the price IDs resolve to. Under the grid: "List prices. Existing customers keep their price for 12 months after any change."
- The cycle toggle defaults to yearly. Once a plan exists the toggle is locked to that plan's interval.
- The card for the current tier is marked "Current plan". Solo is hidden once the workspace has bought extra users.
- Checkout uses `POST /billing/checkout`. The client sends only the plan price; the server picks the seat and add-on prices.
  - `additional_users` and the add-on quantities are amounts *above* what the plan includes, each 0–1000. Unknown body fields are refused with 400.
  - If 1 + additional users is fewer than the current members, checkout returns `409 SEATS_BELOW_MEMBERS`, and the dialog pre-fills the minimum.
- Texting is withdrawn from sale.
  - Checkout refuses it with `409 TEXTING_NOT_AVAILABLE`.
  - In the app, `TEXTING_PURCHASABLE = false` (`lib/textingSale.ts`), so the dialog shows "Texting — coming soon" instead.
- **Welcome credit.** Wallet credit of $15 on Team, $40 on Pro and $75 on Business. It is granted by the webhook once per workspace, on the first paid plan invoice (`service/welcome_credit.go`). Solo grants nothing and does not use up the credit, so Solo → Team/Pro/Business earns it on the first full paid invoice (proration invoices are skipped). The plan cards only promise it while the tier is `trial`.

**The two-subscriptions model**
- Stripe will not mix billing intervals in one subscription, and every add-on is monthly.
  - A **monthly** customer has one subscription: plan base, seats and add-ons.
  - A **yearly** customer who buys add-ons gets a second, monthly add-ons-only subscription.
- The webhook classifies each subscription by its items: `plan` (exactly one plan base) or `addons` (add-ons only). It refuses anything else, such as two bases, a seat of another tier, or an unknown price (`service/pricecatalog.go`).
- The add-on door reports `creates_addon_subscription` / `cancels_addon_subscription` so the UI can explain this.
- The checkout summary says add-ons are "billed monthly as a separate subscription".

**Plan switch** (`service/plan_switch.go`)
- The flow is preview, then confirm (`/billing/subscription/plan[/preview]`). The confirm takes a UUID `request_id`, and a preview is valid for 15 minutes.
- Moves are between Solo, Team, Pro and Business within the same interval, and take effect immediately in both directions.
  - An upgrade is charged now. If the card is declined (402), nothing changes.
  - A downgrade credits the unused time to the Stripe customer balance, which the next invoice uses up.
- Refusals:
  - no live plan: `409 NO_ACTIVE_PLAN`
  - same plan: `409 PLAN_UNCHANGED`
  - monthly↔yearly: `400 INTERVAL_CHANGE_NOT_SUPPORTED`, "contact support"
  - downgrade to Solo while holding paid seats: `SEATS_NOT_AVAILABLE`
  - `409 PLAN_NOT_SWITCHABLE` when the subscription is `PAST_DUE`, `INCOMPLETE` or `CANCEL_SCHEDULED`, or when this cycle's automated-sends usage is already above the target plan's allowance (`USAGE_ABOVE_TARGET_BUNDLE`)
- A downgrade does not check member, record or add-on counts. Anything over the new limits is kept, and the next create is refused.
- The app also disables the plan cards while `past_due` or while a cancel is scheduled.

**Seats** (`service/seat_change.go`, `entitlements/catalog.go`)
- Every paid plan includes 1 user. The seat cap is 1 + purchased seats. Solo is 1 and cannot invite; the trial is 3.
- `POST /billing/subscription/users` takes a **total** `additional_users`, not a change.
- Adding is charged now for the rest of the period. A declined card returns 402 and nothing changes.
- Removing drops the quantity immediately with no refund. The removed users are paid through the period end.
- Floor on removal: members + live pending invitations − 1 (`409 SEATS_BELOW_MEMBERS`).
- The seat door is hidden on Solo (`409 SEATS_NOT_AVAILABLE`).
- Seat changes are refused with `409 PLAN_NOT_SWITCHABLE` while the plan subscription is `PAST_DUE`, `INCOMPLETE` or `CANCEL_SCHEDULED` (the same `switchableStatus` check as a plan switch, `service/quantity_change.go`).
- A pending invitation reserves a seat when an invite is sent. Invite accept and member reactivation are judged against accepted members only.

**Add-ons** (`service/addon_change.go`)
- Five Stripe items exist: mailbox, local line, toll-free line, texting and high-volume texting. All are monthly.
- In the app the owner can change mailboxes, local lines and toll-free lines, via `AddonsCard` steppers or the point-of-need `AddonOfferDialog`. Texting is not editable.
- All three quantities are absolute totals and are required on every call.
- The stepper is clamped at the plan's hold minus what is included; above that is `422 INVALID_QUANTITY`. Lowering below what is in use is `409 ADDONS_BELOW_HELD`.
- Add-on changes are also refused with `409 PLAN_NOT_SWITCHABLE` while the plan subscription is `PAST_DUE`, `INCOMPLETE` or `CANCEL_SCHEDULED` (`service/addon_change.go`). The app does not pre-empt this; the refusal arrives on submit.
- When a member is removed and mailboxes or lines exceed the new included allowance, the extras are billed as add-ons on the next invoice.

**Wallet** (`internal/platform/wallet`, `service/wallet_topup.go`, `service/wallet_charge.go`)
- The prepaid balance pays for texts, picture messages and calls. It is shown to every member: balance, "About N minutes of calling", and a low-balance line at $1 or less.
- Top-up, owner only:
  - presets $20 / $50 / $100 or a custom amount
  - minimum $20, maximum $999,999.99, whole cents
  - only for workspaces on a plan; otherwise `409 NO_ACTIVE_PLAN`
- How it pays:
  - With a card on file, `POST /billing/wallet/charge` charges it directly, with an idempotent `request_id`.
  - With no card on file the charge returns `409 NO_PAYMENT_METHOD` and the dialog does **not** fall back on its own. It offers two buttons: "Add a card" (opens the Stripe portal; the owner then tops up again) or "Pay with Stripe" (Stripe Checkout through `POST /billing/wallet/topup`, which collects a card).
  - The dialog goes straight to Checkout with the same amount only when the bank requires authentication (3DS: `402 PAYMENT_DECLINED` with `AUTHENTICATION_REQUIRED`); the server has already cancelled the off-session attempt (`TopUpButton.tsx`).
  - On a trial the owner sees a "Select your plan" button (to the plan grid) in place of the top-up button. The top-up dialog's terms text says calling and sending stop when the balance is empty while receiving continues, and that prepaid credit is non-refundable, does not expire while the account is open, and stays usable for 12 months after cancellation.
- Charges are written per send at the versioned rate card (`CardV1`). The rate version and unit price are stored on each ledger row.
- A send that did not happen is reversed back into the wallet. Wallet money is never refunded to a card.
- If the balance cannot cover a send, the send is refused with `402 WALLET_INSUFFICIENT_BALANCE`. The inbox SMS composer shows `LowBalanceNotice` / `EmptyWalletNotice`.
- A charge the balance could not cover appears under "Unpaid charges" (`/settings/wallet/shortfalls`). It is not billed later.
- Only telephony draws on the wallet.
  - SMS and MMS are charged before the send. Extra segments are settled afterwards.
  - Calls are settled when the call ends.
  - An outbound call first places a hold for the minutes the balance can cover. A call with no coverable minutes is refused. Stale holds are released by a sweep job, on holds older than 4 hours.
  - Inbound traffic is still received when the balance is empty, and becomes a shortfall.
- Owners are notified when a charge takes the balance across a line downward (`telephony/wallet_charge.go`, SCRUM-1112): "low on credit" when it crosses below **$5**, and "out of credit" when it falls below the price of one SMS (**$0.02**), the point at which sending stops. It fires on the crossing only, not on every later charge, and a single charge that crosses both lines sends only the "out of credit" notice.
- Ledger sources: top-up, auto top-up, welcome credit, promo, admin, migration, charge and reversal.

**Plan limits and enforcement** (`internal/platform/entitlements`)
- One catalog (`catalog.go`) holds every limit, feature switch and monthly bundle per tier. Tiers: `read_only`, `trial`, `solo`, `team`, `starter` (legacy alias of team), `pro` and `business`.
- `entitlements.Boot()` puts every resource, feature and meter in **enforce** mode in both the API and the worker.
- Refusal codes:
  - a count cap: `422 PLAN_LIMIT_REACHED`, with `details[]` including `remedy` = `UPGRADE` | `ADD_USER` | `BUY_ADDON`
  - a missing feature: `422 PLAN_FEATURE_UNAVAILABLE`, naming `available_on`
- Records (contacts, leads, companies, deals) are capped per type.
  - On paid tiers, the save that crosses the limit is allowed and saves beyond it are refused.
  - On trial and read-only, the save at the limit is refused.
  - CSV imports stop at the line and report "N of M imported".
- Mailboxes and phone lines have two numbers: *included* (free) and *hold* (the most the plan can hold, including purchases). The effective cap is the smaller of the hold and included + purchased.
- Automated sends are a monthly meter (Team 1,000, Pro 7,500, Business 35,000). One unit is used per email or text that an automation or sequence sends.
  - At 0 the step does not send: the automation goes to `error`, and the sequence pauses.
  - Trial and Solo have no automated sending, and saving a flow with a send step is refused.
- The app pre-empts refusals by reading `GET /settings/usage`:
  - `useCapGate` / `CapGateNotice` on every capped create, warning at 80%
  - `useFeatureGate` in sequences and the automation builder
  - `PlanRefusalNotice` and `AddonPreemptNotice` for refusals and add-on remedies
- The Usage section lists every cap with its used / limit and features as "Included" or "Not on this plan · Available on X". It shows "Included this cycle" for automated sends, with a reset date.

**Trial**
- `trial_ends_at` = signup + 21 days (`db/sqlc/users.sql.go`).
- Trial limits are Pro's counts except: 3 seats, 2,500 of each record type, no mailbox, no phone line, no sequences and no automated sends.
- The trial's usage window is anchored on signup.
- The cycle-roll worker (`entitlements.cycle-roll.v1`, every 15 minutes) moves an expired trial to `read_only`. `trial_ends_at` is kept so the app can say when it ended.
- `TrialBanner` text: "Your trial ends on {date} — N days remaining". It turns to warning severity at 3 days or fewer. It is dismissible, and the owner's call to action is "Choose a plan".

**Payment states and cancelled accounts**
- `GET /billing/subscription` returns `payment.state`, checked in this order: `past_due`, `ok`, `unpaid`, `ended`, `trial_expired`, `none`. `SubscriptionDetailsCard` / `lib/paymentState.ts` show:
  - `past_due` → "Your payment is past due"
  - `unpaid` → "Your plan ended after payment failed", or "Your add-ons ended after payment failed"
  - `ended` → "Your plan ended on {date}"
  - `trial_expired` → "Your trial ended on {date}"
  - cancel at period end → "Your plan ends on {date}"
- `past_due` keeps the paid tier, so access is unchanged during Stripe's retry period. Owners are notified on the change to `past_due` and again when payment recovers.
- `incomplete` (first payment not yet succeeded) does not grant a tier.
- `unpaid` leaves the tier as it is. Only the Stripe *deleted* event changes the workspace.
- When a plan subscription is deleted, the webhook:
  - sets the tier to `read_only`: every limit 0 and every feature off, while all data is kept
  - cancels any add-on subscription immediately
- What `read_only` refuses: every capped create (records, imports, pipelines, fields, tags, invites, accepting invites, mailbox connect, number purchase) and every gated feature.
- What `read_only` does **not** refuse: editing or deleting existing records, reads, manual email, and texts or calls paid from a remaining wallet balance. There is no general write lock. `ReadOnlyBanner` then shows "… — this workspace is read-only · Your data is kept" on every page.
- Cancelling happens in the Stripe portal. The app says that cancelling the plan also cancels add-ons immediately, while cancelling only the add-ons keeps the plan.
- To resubscribe, start a new checkout from the plan grid. The welcome credit is not granted again (unless the workspace has only ever paid for Solo).

**Return pages**
- `/billing/success` shows plan or top-up copy depending on `?kind`. It refreshes the usage and wallet queries and links back to Settings.
- `/billing/cancel` says no charge was made.

## Permissions
- **Owner** can do everything in this area: checkout, portal, subscription read, plan switch, seats, add-ons and wallet top-up/charge.
  - The backend checks this with `requireOwner` → `authz.CapBillingManage`, which only the owner role holds.
  - Anyone else gets `403 NOT_AUTHORIZED` ("only the workspace owner can manage billing").
- **Admin and member** can read the wallet balance, wallet history, unpaid charges and plan usage (`/settings/wallet*`, `/settings/usage`: any active member).
  - The plan grid is replaced by "Billing is restricted".
  - Banners and gates tell them to "Ask your workspace owner…".
- **Billing alerts** (webhook-driven notifications) go to the roles that hold `CapBillingManage`, which is the owner only.
- **Legacy `manager` rows** cannot manage billing (owner only), the same as admin.

## API
- `POST /billing/checkout`: plan checkout session (with quantities)
- `POST /billing/portal`: Stripe customer portal session
- `GET /billing/subscription`: plan, quantities, payment state, invoices
- `POST /billing/subscription/plan/preview`, `POST /billing/subscription/plan`: switch plan
- `POST /billing/subscription/users/preview`, `POST /billing/subscription/users`: change seats
- `POST /billing/subscription/addons/preview`, `POST /billing/subscription/addons`: change add-ons
- `POST /billing/wallet/charge`: top up from the card on file
- `POST /billing/wallet/topup`: top up through Stripe Checkout
- `GET /settings/wallet`, `GET /settings/wallet/ledger`, `GET /settings/wallet/shortfalls`
- `GET /settings/usage`: tier, `trial_ends_at`, caps, features, units
- `POST /billing/stripe/webhook`: Stripe events. Unauthenticated and signature-checked.

## API only (no UI yet)
- **Texting add-on (standard / high volume).** It exists as Stripe items and in the subscription read. Checkout refuses it until carrier (A2P) registration exists, and the app hides it.
- **Features with no product surface:** `api_access`, `sso` and `audit_log`. They are set per tier in the catalog and reported in `/settings/usage`, but no routes or screens use them yet.
- **Auto top-up.** Only a ledger source (`auto_topup`) and a monthly-ceiling error exist in `internal/platform/wallet`. There is no endpoint and no UI, so it is effectively not built.

## Not built (lives in the vision design)
- **Customer Payments** (`payments`, `PaymentsPage`) and **hosted pay checkout** (`pay-checkout`, `PayCheckoutPage`): invoices, quotes, products, payment links, customer subscriptions, the revenue widget, "Create invoice/quote" on record pages, payment automation triggers, invoice items in the activity feed, and pay-at-booking.
- **On the billing page:**
  - the 3-plan prototype grid (Starter free / Pro $59 / Business $99) and per-seat hero
  - the "Manage seats" drawer with named seats
  - an in-app payment-method card
  - "Usage & spending controls" (monthly spending cap, pause or allow overage)
  - "Billing alerts" toggles
  - itemised "Phone & SMS charges billed at cost"
  - "White-label reports" and "Extra email sync" add-ons
  - "Pause auto-renew", and an in-app "Cancel plan" danger zone
  - trial "Add a payment method" with no charge until trial end, and the "Data preservation" countdown
  - the per-state page layouts and the floating billing-state switcher

## Known gaps
- **Subscription read is owner-only.** `GET /billing/subscription` is restricted to the owner, but `SubscriptionDetailsCard` and `InvoicesCard` are mounted for every member. For admins and members the read is refused, so those cards have nothing to show.
- **Monthly↔yearly switching** is refused in the app ("contact support").
- **Add-ons past the hold at checkout.** Checkout could sell add-ons past the plan's hold, while the add-on door refuses that. This was a listed backend follow-up in the 2026-09-18 handoff and was not re-verified here.
- **Stale comments** in `entitlements/catalog.go` still say caps run in watch mode; they all enforce. The frontend billing README is also out of date, for example `ADDONS_PURCHASABLE` (now true).
- **Legacy tier name.** `starter` is still a legal tier (the old name for Team) until its contract migration lands. The plan grid maps it to the Team card.
- **Grace and dunning timing** come from Stripe's retry settings, not code. The code only defines what each status means. See [`pricing/pricing.md`](../../pricing/pricing.md).
- **Plan prices are hardcoded** in the frontend, with no catalog endpoint, so they can drift from Stripe.
- **Failed webhook events are lost silently.** The webhook answers 200 as soon as an event is queued, so Stripe never redelivers it. An event that keeps failing is archived after its retries with no alert.
- **Automated-sends over-count.** An automation SMS step refused for an empty wallet still uses its allowance unit, so the allowance can over-count.
- **Records-cap batch gap.** On paid tiers, the one save allowed at the records line is not capped for batch admissions: a latent gap in `records_policy.go`.
- **Unpaid charges are never collected.** Shortfalls are recorded, but nothing collects them later.
- **The Stripe portal only allows** changing the payment method and cancelling. Plan changes go through the app's own plan-switch screen.
- **Seat and add-on doors do not pre-empt a blocked subscription.** While the plan is `past_due`, `incomplete` or set to cancel, the add-on steppers and seat door stay enabled and the backend refuses on submit with `409 PLAN_NOT_SWITCHABLE`. Only the plan cards are disabled up front.
