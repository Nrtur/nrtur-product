---
area: Phone and SMS setup
status: built
verified_against: frontend@1bb6aa5d, backend@577bae44
verified_on: 2026-10-06
---
# Phone and SMS setup

## What it does
nrtur runs managed telephony: the workspace buys US phone numbers inside the app and never handles carrier credentials. Buying the first number quietly creates the workspace's telephony account in the background, so there is no separate "enable" step. Admins buy, release and route numbers under Settings → Phone numbers. Each number either rings one teammate or rings everyone. Any member can call from the browser softphone, which is mounted on every page so incoming calls ring wherever the user is. After a call, the user annotates it in a post-call form. Texts and calls are paid from the workspace's prepaid wallet: texts per segment, calls per minute. Carrier registration (A2P 10DLC), call recording and transcripts are not built.

## Screens
| Route (app) | Prototype page id | Purpose |
|---|---|---|
| `/settings?tab=phone-numbers` | `settings-phone-numbers` | "Your numbers" list (number, capabilities, status, inbound routing, monthly price, Release) and "Buy number". |
| Buy number sheet (from the page above) | `BuyNumberModal` | Search (country, Local or Toll-free, optional 3-digit area code) → priced results → confirmation summary → buy. |
| Softphone dock, on every app page (`<Softphone>` in `AppShell`) | none (closest is the `CallsWorkspace` dialer and `ContactDialerPopover`) | Dial pad with a caller-ID picker, live call (Mute, Keypad/DTMF, quick note, End) and a wallet countdown. |
| Incoming call dialog, on every app page | none | Accept or decline, with a best-effort caller name. |
| Post-call notes modal | `CallEnrichmentForm` | Outcome, reason, next step and notes, saved onto the call the backend logged automatically. |
| `/inbox` → Calls | `inbox` (tab `calls`) → `CallsWorkspace` | Call log, call detail, "Log a call", inline dialer, "Make a call". |
| Log a call modal (Calls tab, contact record, quick-add) | `LogCallModal` | Manually log a call that happened outside nrtur. |
| Contact and lead record "Call" button | `ContactDialerPopover` | Opens the global softphone with the record's number filled in. |

## Behaviour and rules
**Numbers**
- Search is US only (the country select has one option) and covers `local` or `toll_free` numbers. The area code is optional and must be exactly 3 digits. Each result shows `$X.XX/mo`. Before buying, a summary reads "You'll be charged $X/month for {number}, billed to your workspace". A purchase can take several seconds while the workspace's telephony account is created.
- Number status on the wire is `active` or `releasing`. Release asks for confirmation and cannot be undone.
- Inbound routing per number: assigned to one member, or "Everyone" (`assignedUserId: null`). Sending an empty string is a 400 error.
- The list header reads "N numbers · $X/mo · billed to your workspace". The default sender shows as a read-only "Default" badge.
- Plan cap `phone_numbers`: Trial 0; Solo 0 included (one can be bought as an add-on); Team, Pro and Business 1 per user, with add-ons up to 2 × users. The "Buy number" button is blocked when the cap is reached, and a server 422 `PLAN_LIMIT_REACHED` also blocks inside the open sheet. The pre-emptive notice on the page always names the local line add-on (`local_line`). When a buy is refused, the owner is offered the add-on that matches the number type: `tollfree_line` for a toll-free number, otherwise `local_line` (`billing/lib/addonDoor.ts`, `lineKindForNumberType`); the offer dialog retries the buy once.

**Readiness**
- `GET /telephony/status` returns `voiceReady`. When it is false, "Make a call" is disabled with a stated reason. Admins also get a Repair button (`POST /telephony/reprovision`, which repairs the account but does not create it: 409 `TELEPHONY_NOT_PROVISIONED` on a workspace that has never been set up). `provisionError` is shown only to admins; members are told to ask an admin.

**Outbound calls (softphone)**
- The browser places the call through the Voice SDK, and the backend answers with instructions to dial out. There is no endpoint to start a call. The caller ID must be a number the workspace owns, and the picker lists the workspace's numbers.
- Microphone permission is requested when the dialer opens. A blocked microphone shows a persistent banner with the fix.
- Wallet gate: `callable_minutes === 0` disables dialing. Two messages explain why: the balance is empty (with a Top up link for the owner), or the balance is held by calls already in progress. Below 5 minutes, a hint shows the remaining minutes. The call is cut off at the budget captured when it was dialed (Twilio `timeLimit`), with a warning 60 s before the limit. A call that ends at the limit reads "Call ended: your balance ran out". This is inferred, because the backend sends no end reason.
- The in-call timer is for display only. The backend logs each SDK call with the carrier-measured duration, and the post-call form only edits that record; it never creates a call.
- Hold is not supported (the app shows a disabled "Hold isn't available on browser calls yet" tile). Mute and the keypad (DTMF tones for phone menus) work.
- One device per browser tab is registered, and it is torn down on logout. The access token is refreshed before it expires.

**Inbound calls**
- An assigned number rings only its member. An unassigned number rings all active members at once (at most 10), and the first to answer takes the call. A call rings for 25 s, then counts as missed. There is no voicemail.
- Incoming calls are never blocked by the wallet. They are charged after they end.

**Call log**
- Outcome, reason and next step are free text. The forms offer: Outcome = Connected, No answer, Left voicemail, Busy, Wrong number. Reason = Discovery, Follow-up, Demo, Negotiation, Check-in, Support, Other. Next step = None, Schedule follow-up, Send proposal, Send email, Create task, Mark deal won, Mark deal lost. Choosing a next step only records it; nothing is executed.
- The list's filter chips (All, Missed, Voicemails, Today) and its search work on the loaded pages only. "Load more" pages through the full history. The detail view shows Total calls, Talk time and Last contacted (from `GET /telephony/calls/stats`). "View full history" filters the list to that number.
- Log a call: phone number (optional when opened from a record), direction, "Reached them?", optional duration, outcome, reason, next step, optional notes.

**SMS sending** (screens are in `inbox.md`)
- The sender is chosen in this order: a pinned `fromNumberId`, then the number already used in a thread with that recipient, then the workspace default. The picker lists only active numbers that can send SMS. Sending from a number without SMS returns 409 `TELEPHONY_NOT_SMS_CAPABLE`. Media attached while the selected number cannot send MMS blocks Send in the composer.
- Each send carries an `Idempotency-Key`. `IDEMPOTENCY_IN_FLIGHT` keeps the message queued and is not treated as a failure.
- A recipient who opted out returns 409 `TELEPHONY_RECIPIENT_OPTED_OUT`. Inbound STOP, STOPALL, UNSUBSCRIBE, CANCEL, END or QUIT suppresses the number. START, YES or UNSTOP lifts the suppression. Both are written to the contact timeline.
- This STOP list (`sms_suppressions`) is the only thing that blocks a manual text. The workspace suppression list (`/suppressions`, including an SMS or `all` do-not-contact row set on a contact) does not block `POST /telephony/sms/send`; it blocks only automation and sequence SMS steps, through the flow runner's suppression gate. Calls are never checked against either list. See `inbox.md` → "What blocks a send…".

**SMS segments and billing (prepaid wallet, rate card v1)**
- The composer counts segments live: GSM-7 is 160 characters, or 153 per segment when split; UCS-2 (emoji, most non-Latin text) is 70, or 67 per segment. A warning appears at 4 or more segments.
- SMS costs $0.020 per segment. MMS costs $0.050 per message, whatever the caption length. Calls cost $0.030 per started minute. Inbound minutes on a toll-free number cost $0.050. Inbound texts and inbound minutes are charged as well, and that includes STOP and HELP messages.
- The text is charged before it is sent. If the wallet cannot cover it, the send returns 402 `WALLET_INSUFFICIENT_BALANCE` and nothing is sent. The charge is refunded only if the provider definitely rejected the message.
- SMS does not count against the "automated sends" allowance (Pro 7,500 / Business 35,000, confirmed in `entitlements/catalog.go`). That allowance is used only by automation sends.

## Permissions
- **Any active member**: view numbers and telephony status, get a voice token, place and receive calls, send and read SMS, log calls, and edit or delete any call (there is no ownership check).
- **Owner and admin** (`config:manage`): search, buy, assign and release numbers, and run Repair. The frontend hides these for members (`isWorkspaceManager`), failing closed while the role is still loading.
- **Owner only**: buy the add-on when the line cap is reached, and use the wallet Top up link.

## API
- `GET /telephony/status`, `POST /telephony/reprovision`
- `GET /telephony/numbers/available`, `GET /telephony/numbers`, `POST /telephony/numbers`, `PATCH /telephony/numbers/{id}`, `DELETE /telephony/numbers/{id}`
- `POST /telephony/voice/token`
- `GET /telephony/calls`, `GET /telephony/calls/stats`, `GET /telephony/calls/{id}`, `PATCH /telephony/calls/{id}`, `POST /telephony/calls/log`
- `POST /telephony/sms/send`, `GET /telephony/sms/conversations`, `GET /telephony/sms/conversations/{id}`, `POST /telephony/sms/conversations/{id}/read`, `GET /telephony/sms/media/{id}`
- `GET /settings/wallet` (`callable_minutes`), `GET /settings/usage` (cap gate)
- Twilio webhooks (public, signed): `POST /telephony/webhooks/sms/inbound`, `/sms/status`, `/voice/inbound`, `/voice/status`

## API only (no UI yet)
- `PATCH /telephony/numbers/{id}` with `isDefault: true` makes a number the workspace's default sender. The UI only shows the badge.
- `DELETE /telephony/calls/{id}` soft-deletes a call.
- `GET /telephony/status` also returns `a2pStatus` (reserved, not acted on) and mobile push readiness fields used by the mobile app.

## Not built (lives in the vision design)
- Trust center: business profile, A2P brand, A2P campaigns, the registration wizard, toll-free verification, CNAM caller-ID name, international regulatory bundles, and the "Simulate +1 day" review clock.
- The SMS blocking that depends on A2P registration. The texting add-on is refused with 409 `TEXTING_NOT_AVAILABLE` until A2P exists.
- Number porting, and international numbers (CA, GB, AU).
- "Set as default" on a number.
- The "Texting spend limit" card. Billing is a prepaid wallet, not a monthly usage invoice with a cap.
- Voice provider choice (Built-in / Twilio / Google Voice) and connecting your own Twilio.
- Call recording and playback, transcripts, talk ratio, AI call summary, voicemail, Hold, and call transfer.
- Picking a contact by search inside the dialer (the app dials a typed number or a number passed from a record).
- "Update contact status" in the post-call form, and running the chosen next step automatically.
- The call-history overlay with answer rate, outcome bars and sparkline.

## Known gaps
- **A2P mismatch:** SMS sends work today without carrier registration, while billing says texting is not sold until A2P exists. US carriers may filter unregistered 10DLC traffic. This is a product and compliance decision, not a UI detail.
- Telephony types in the frontend are hand-written against the spec, not generated from it (`telephonyService` uses `bffFetch`).
- Matching a post-call note to its call relies on a heuristic until the backend fills `twilioCallSid`. It is designed to fail to find a row rather than pick the wrong one.
- `contactName` on calls and SMS conversations is not yet filled by the backend, so the lists show numbers.
- A trial workspace's phone-number cap is 0, so trials cannot buy a number or test calling or SMS.
- **Compliance gap:** manual SMS and outbound calls do not check the workspace suppression list. A contact marked do-not-contact, or with an SMS or calls suppression row, can still be texted (unless they texted STOP) and called. Only automation and sequence SMS honour the list.
- The prototype has no incoming-call, wallet-gate or microphone-blocked states; the app has all of them. (The prototype's Calls workspace does show a "Calling isn't ready." banner with Repair for admins.)
- A comment in `voice_webhook.go` still says receiving a call costs nothing. Inbound minutes have been charged since SCRUM-1167.
- `LogCallModal`, `Dialer` and a few other sheets still lack the `min-h-0` scroll fix, so a long sheet can show two scrollbars.
