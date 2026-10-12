---
title: Billing page polish (wallet history infinite scroll, Plan usage layout)
status: proposed        # proposed | approved | in-progress | built | dropped
owner: Qamar Ul Islam
area: docs/areas/billing.md
screens: [settings-billing]
tickets: []
created: 2026-10-12
---

# Billing page polish

## Problem
- **Wallet history:** the list sits in a fixed 360px box with its own scroll (SCRUM-1232), but older rows still need a **Load more** click inside the box. It feels clunky for a long history.
- **Plan usage:** the app shows the right numbers, but as one long single-column list (Records / Limits / Features, then "Included this cycle" last). It is hard to scan and doesn't match the design.

## Proposal

**1. Wallet history: fixed height with infinite scroll** (`settings-billing`, "Wallet history" card)
- The list keeps its fixed height of **360px** with its own scroll area. The page doesn't grow.
- When the reader scrolls near the bottom of the box, the **next page loads automatically**. There is no Load more button.
- While a page loads, a row at the bottom says "Loading older activity…" with a small spinner.
- When everything is loaded, the last row says "That's everything." This only appears when there was more than one page.
- If a page fails to load, the row at the bottom says so, with a **Retry** link. Rows already shown stay, and so does the scroll position.
- The box stays keyboard-scrollable and labelled "Wallet history", as it is today.

**2. Plan usage: match the design layout** (`settings-billing`, "Plan usage" card)
- **Subtitle:** the plan and member count, e.g. "Pro plan · 4 members". The app already does this; the design now does too.
- **Top:** "Included this cycle: Automated sends" as the main bar, showing used of included and the reset date.
- **Middle:** every cap in a **2-column grid**. Each cap has its name, used / limit, and a thin bar. Unlimited caps show "Unlimited" with no fill. The Seats cap shows "N left to invite" under it.
- **Bottom:** the feature list. Each feature shows "Included", or "Not on this plan · Available on <plan>".
- **No change** to the read-only state or to the data shown. This is layout only.

## Rules
- **Wallet history:** pages come from the existing ledger cursor (`GET /settings/wallet/ledger`), using the page size the app already uses. Newest first, as today.
- **Plan usage:** the numbers and labels come from `GET /settings/usage`, as today. Nothing new is fetched.

## Permissions
No change. Every member can see the wallet history and plan usage.

## Plan and pricing impact
None.

## Out of scope
- Filtering or searching the wallet history.
- Exporting the wallet history.
- Any change to what Plan usage measures.

## Tickets (to create after approval)
- **FE:** Wallet history — replace Load more with infinite scroll in the fixed-height box.
- **FE:** Plan usage — restyle to the design layout (top sends bar, 2-column cap grid, feature list), keeping the current data and states.
- **BE:** none. The ledger already pages by cursor, and usage already returns everything.
