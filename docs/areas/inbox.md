---
area: Inbox
status: built
verified_against: frontend@1bb6aa5d, backend@577bae44
verified_on: 2026-10-06
---
# Inbox

## What it does
The Inbox is one screen (`/inbox`) where a team reads and answers its email, text messages and calls. Email comes from mailboxes the team connects through a hosted provider sign-in (Gmail, Microsoft 365, and IMAP-based providers). Threads sync in the background, open in a reader, and can be replied to, forwarded, starred, archived, deleted and labelled. SMS conversations run on the workspace's own phone numbers, with picture messages (MMS) attached inside the SMS thread. The Calls tab is the workspace call log plus the in-browser dialer. The "All" tab is a merged, newest-first feed across email, SMS/MMS and calls. Everything is workspace-shared: every member sees every connected mailbox, every SMS conversation and every call. Recipients of automated email can opt out on a public unsubscribe page.

## Screens
| Route (app) | Prototype page id | Purpose |
|---|---|---|
| `/inbox` (default tab: Email; deep links `?channel=`, `?thread=`, `?call=`, `?conversation=`) | `inbox` | Channel tabs: All, Email, SMS, Calls inline; Push, In-app (and an MMS pointer) under "More". |
| `/inbox` → All | `inbox` (tab `all`) | Merged feed from `GET /inbox/feed`, with cross-channel search. A row click switches to the item's tab. Hidden behind the connect prompt until a mailbox exists. |
| `/inbox` → Email | `inbox` (tab `email`) | Folder rail (Inbox, Starred, Drafts, Sent, Archived, Trash), workspace labels, connected mailboxes, thread list with filter chips, reader, inline reply composer. |
| `/inbox` → SMS | `inbox` (tab `sms`) | Conversation list, thread with inline MMS, composer with sender-number picker and segment counter, "New message" modal. |
| `/inbox` → Calls | `inbox` (tab `calls`) | Call log with filter chips, call detail with editable notes, "Log a call", inline dialer. Covered in detail in `phone-and-sms-setup.md`. |
| `/inbox` → Push / In-app | `inbox` (tabs `push`, `inapp`) | Read-only notification history. Owned by the Notifications area. |
| Compose modal (Email tab, All-tab "Compose" menu, quick-add) | `EmailComposeModal` | New email from a chosen mailbox. |
| `/settings?tab=integrations` → email accounts card | `settings-integrations` | List, connect, reconnect and disconnect mailboxes. Shared with the Integrations area. |
| `/unsubscribe/[token]` (public, outside the app shell) | none | Recipient confirms opt-out from an automation email. |

## Behaviour and rules
**Mailboxes**
- Connect posts the provider's `serviceType` and is sent to the hosted consent page. No IMAP form is shown in the app; for IMAP providers the hosted page collects credentials. The provider cards are Gmail, Microsoft 365, Business email (IMAP), Yahoo, iCloud and Other. Yahoo, iCloud and Other connect over generic IMAP.
- The callback returns to `/inbox?connected=1`, and the modal then polls until the account appears. On failure it returns `?error=<reason>` (`invalid_state`, `not_completed`, `connect_failed`, `plan_limit_reached`).
- A mailbox can be in one of these states: synced, checking, not synced, or needs reconnect. An unknown status is shown as an error and never as healthy. A healthy mailbox shows no "live sync" pill, by product decision.
- The first sync backfills 30 days. After that, a webhook drives incremental sync on the worker. Message bodies and attachments are not stored: they are fetched from the provider when a thread is opened (`internal/modules/inbox`).
- Plan cap `mailboxes`: a trial gets 0 (no mailbox until a plan is bought). Solo gets 1. Team, Pro and Business get 1 per user, with add-on mailboxes up to a ceiling. The cap is checked at connect (422 `PLAN_LIMIT_REACHED`) and again at the callback. The frontend checks it in advance with `useCapGate("mailboxes")` and offers the owner a "$5/mo" add-on. Reconnect is never blocked by the cap.
- Disconnect permanently deletes the account and all its synced threads and messages. It cannot be undone.

**Email threads**
- The list uses cursor pages with infinite scroll. Filters `unread`, `starred` and `needs-reply` (which maps to `unread`) run on the server. "With deals" runs in the client, with a bounded auto-loader of 5 pages. Search is debounced by 300 ms.
- Opening an unread thread marks it read right away in the UI and also marks it read at the provider. Star, archive and delete call the provider.
- The reader renders HTML in a sandboxed iframe with a strict CSP (no scripts, no remote images). Inline `cid:` images go through the authenticated proxy. A message whose body failed to load shows a quiet "couldn't load" note.
- The CRM context strip shows the matched contact, plus a deal chip when the contact has a linked deal, and a "View contact" link.
- Sequence-sent mail shows a "Sequence · <name>" chip in the list and a "Sent automatically by…" line with links on the message. Automation-sent mail shows nothing.
- Reply, reply-all and forward use the inline composer. Reply-all is disabled on one-to-one threads. Compose has From (mailbox picker), To, Cc, Bcc and Subject, a rich-text body with links, attachments and Save draft. Attachments are capped at 20 MiB in total and 100 files, sent inline and never stored. A single download is capped at 25 MiB.
- Sends accept an optional `Idempotency-Key`. A failed send shows a toast. A failed mark-read stays silent.

**Labels**
- Workspace labels are listed from `GET /inbox/labels`. Owners and admins can create, rename, recolour and delete them: name 1–60 characters, colour `#RRGGBB`. Any member can apply or remove a label on a thread.
- No label shows a count. Clicking a label in the rail does not filter the list.

**All feed**
- The feed covers email, SMS, MMS and calls only. Push and in-app notifications never appear in it. Rows are two lines with a channel badge (icon and label) and carry no subject or star. A call with no summary reads "No summary".
- The tab badge comes from the feed's own unread rollup. Clicking an email row selects that thread in the Email tab. MMS rows open the SMS tab. Call rows open the call.
- The "Compose" menu on All offers Email, SMS and Call. Each option is disabled with a reason: "No mailbox", "No number" or "Not ready".

**SMS conversations** (send rules: see `phone-and-sms-setup.md`)
- The conversation list polls every 5 s. Previews show only the direction ("New message" / "You sent a message"); the backend does not provide message text for previews.
- Replies are sent from the thread's own number, which is never replaced by the workspace default. "New message" reuses an existing thread's number for that recipient. Otherwise it lets the backend choose the sender.
- A thread whose contact texted STOP shows an "opted out" banner in place of the composer. A START reply re-enables the thread.
- Inbound MMS media shows in the thread and in a media gallery. Outbound media is uploaded to workspace storage first (`POST /files`), with at most 10 parts. "Attach media" is blocked by the `attachment_bytes` storage cap. Sending text alone is never blocked by plan caps.
- An empty prepaid wallet blocks a send (402 `WALLET_INSUFFICIENT_BALANCE`). The UI shows an empty-wallet notice and no toast.

**Unsubscribe and compliance footer**
- Automation (flow) emails include a footer with the workspace mailing address and a per-recipient unsubscribe link (`<frontend>/unsubscribe/<token>`). Manual reply and forward include the address only, with no link (`inbox/compliance.go`). Automation email is not sent until an admin has set the mailing address and unsubscribe base under Settings → Compliance.
- The unsubscribe page never submits when it loads. It shows no address or workspace. It sends `POST /unsubscribe/{token}` only after an explicit click. The result writes a permanent suppression and ends live enrollments. Error states: invalid link (400, no retry), too many attempts (429, retry), generic failure (retry), incomplete link.
- No `List-Unsubscribe` header is sent, because the provider API cannot set custom headers.

**Routing (built)**
- Inbound email and SMS are matched to contacts by email address or phone number (`contact_identifiers`). Unknown senders stay unmatched, and no lead is created automatically.
- An inbound reply on a thread that a sequence sent ends that enrollment (inbox inbound observer). Email and SMS write timeline activities that link back to the thread.
- Calls logged before a contact or lead existed are linked to it afterwards (call-link backfill).

## Permissions
- **Any active member** (owner, admin or member): read every mailbox, thread, SMS conversation and call; connect a mailbox; reconnect; send, reply and forward email; save drafts; star, archive, delete and mark read; apply or remove labels; send SMS; read and edit the call log.
- **Owner and admin only**: disconnect a mailbox (403 otherwise), and create, rename or delete labels (`config:manage`). The frontend hides these controls for members with `canManageWorkspaceConfig`.
- **Owner only**: buy the mailbox add-on from the cap prompt. Everyone else sees who to ask.
- The unsubscribe endpoint is public. It is authenticated by its signed token and rate limited per IP.

## API
- `GET /inbox/accounts`, `POST /inbox/accounts/connect`, `DELETE /inbox/accounts/{id}`
- `GET /integrations/aurinko/callback` (public), `POST /inbox/webhooks/aurinko` (public)
- `GET /inbox/emails`, `GET /inbox/emails/{threadId}`, `GET /inbox/counts`, `GET /inbox/feed`
- `POST /inbox/emails/{threadId}/read`, `/star`, `/archive`, `/delete`
- `POST /inbox/emails/{threadId}/labels`, `DELETE /inbox/emails/{threadId}/labels/{labelKey}`
- `GET /inbox/labels`, `POST /inbox/labels`, `PATCH /inbox/labels/{key}`, `DELETE /inbox/labels/{key}`
- `POST /inbox/emails/{threadId}/reply`, `POST /inbox/emails/{threadId}/forward`, `POST /inbox/messages`, `POST /inbox/drafts`
- `GET /inbox/emails/{messageId}/attachments/{attachmentId}`
- `GET /sequences/{id}` (sequence chip lookup only)
- `GET /telephony/sms/conversations`, `GET /telephony/sms/conversations/{id}`, `POST /telephony/sms/conversations/{id}/read`, `POST /telephony/sms/send`, `GET /telephony/sms/media/{id}`
- `GET /telephony/calls`, `GET /telephony/calls/{id}`, `GET /telephony/calls/stats`, `POST /telephony/calls/log`, `PATCH /telephony/calls/{id}`
- `GET /notifications`, `GET /notifications/counts` (Push and In-app tabs)
- `POST /unsubscribe/{token}` (public)

## API only (no UI yet)
- **Scheduled send**: `scheduledAt` on compose, reply and forward, at most 365 days ahead; `GET /inbox/scheduled` and `DELETE /inbox/scheduled/{id}` to list and cancel. A cancel and the scheduled send cannot both win: the request that arrives first takes effect.
- `DELETE /telephony/calls/{id}`: soft-deletes a call that was logged by mistake.

## Not built (lives in the vision design)
- Email templates and SMS templates, both the pages and the template picker in compose. There is no email builder.
- AI draft in compose and reply, AI "suggested next steps" chips, and the "AI sort" ordering.
- Schedule send in the UI, Snooze, Mark as unread, "Mark all read", the list Sort control, and filtering by label.
- The in-app IMAP/SMTP connect form and the account-picker and consent steps (consent happens on the provider's hosted page).
- On an unknown sender: the "Create lead" card and "Lead captured from this email" card.
- SMS: merge-variable "Insert" chips, "Link to deal", the shared-media strip in the thread header, separate image, video and file attach buttons (the app has one "Attach media"), the "Configure SMS" button, and the carrier-registration (A2P) banner.
- MMS as its own channel. The MMS tab only points to the SMS tab.
- The "Simulate inbound" demo menu, and the inbox status footer ("Sending as…" / "Unified inbox · Email · Messages · Calls").
- Call recordings, transcripts, talk ratio, voicemail and Hold.

## Known gaps
- The app shows a decorative "AI sort" label and a Sort button that does nothing (`EmailList.tsx`). Both are cut from the design. The app should drop them as well.
- The app still renders disabled "coming soon" controls: Templates, AI draft and Schedule send in compose; Snooze and More in the reader; Mark all read; label rows. These are cut from the design.
- A feed row for an email outside the loaded Inbox pages opens an empty reader. This needs `starred` on the feed item, which is a backend follow-up.
- The Starred folder has no live count. Labels have no counts.
- No draft list or management exists beyond Save draft (drafts appear in the provider's Drafts folder after sync).
- SMS list previews lack message text, and `contactName` is not yet filled by the backend, so rows show the number with a "CRM" badge.
- The `inbox_labels` cap is unlimited on every tier today, so the label cap gate never triggers.
- The app has screens the prototype does not: the All-tab Compose menu, the "New SMS message" modal, label create/edit/delete dialogs, the Reconnect flow, the plan-limit step in the connect modal, empty-wallet and storage-cap notices, the sequence chip and line, and the unsubscribe page.
- The prototype defaults to the All tab; the app defaults to Email.
- Per-IP rate limiting on unsubscribe sits behind the frontend proxy. It is unconfirmed whether the backend reads the real client IP, so recipients could share one bucket.
