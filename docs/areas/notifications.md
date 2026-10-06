---
area: Notifications
status: built
verified_against: frontend@1bb6aa5d, backend@577bae44
verified_on: 2026-10-06
---
# Notifications

## What it does
Notifications tell a user about things that concern them personally: mentions, task assignments and reminders, assigned leads, missed calls and SMS replies, sequence replies, team changes, billing problems, failed automations and finished imports. The backend writes one row per recipient. The app shows the unread count on the rail bell, lists the rows in a drawer, and can show a short popup card top-right. Each user picks, per notification type, whether they get it in the app and whether it is pushed to their phone. A live stream (SSE) can make the bell update in under a second, but it sits behind a build flag. When the flag is off, the stream is down, or the tab is hidden, the app falls back to polling.

## Screens
| Route (app) | Prototype page id | Purpose |
|---|---|---|
| n/a (drawer from the rail bell) | `NotifDrawer` (component) | The list: All / Unread / Mentions / Assigned to me, grouped Today / Earlier, with mark read |
| n/a (top-right cards) | `NotifPopupHost` (component) | Live popup cards for new notifications |
| `/settings/notifications` | `settings-notifications` | The user's own notification preferences. A standalone page with its own top bar (like `/profile`), not a tab inside the settings rail. Has loading and error ("Couldn't load notification settings", Try again) states |
| `/inbox` Push and In-app tabs | (Inbox area) | Read-only history for one channel. Reuses the drawer's row (`NotificationChannelPane`). The backend only writes `channel='inapp'` rows, so the Push tab is always empty and `counts.push` is always 0 |

## Behaviour and rules
- **Types** (`internal/platform/notify/notify.go`). There are 22 declared types. The settings page groups them into 8 categories (`features/notifications/lib/notificationTypeCatalog.ts`). Rows render in the server's declared order (`notify.DeclaredTypes`), not the catalog's:
  - Tasks: `task.mentioned`, `task.assigned`, `task.reminder`, `task.due`, `task.overdue`
  - Leads: `lead.assigned`
  - Inbox & email: `inbox.mailbox_disconnected`
  - Calls & SMS: `call.missed`, `sms.received`, `sms.opted_out`
  - Sequences & automations: `sequence.replied`, `flow.errored`, `automation.sends_paused`, `automation.sends_resumed`
  - Team: `note.mentioned` ("Mentioned in a note"), `member.joined`, `member.role_changed`
  - Billing: `wallet.low_balance`, `billing.payment_failed`, `billing.payment_recovered`
  - Imports: `import.completed`, `import.failed`

  On the settings page, a type the app has no copy for still renders in a trailing "Other" section, with a generic Bell icon and the raw type string. The drawer's fallback is different: a Zap icon and the server's title.
- **Bell badge.** It shows the number from `GET /notifications/counts`, which returns `{push, inapp, total}`. It hides at 0 and caps at 99+.
  - Polling is gated on visibility: no poll while the tab is hidden, every **30s** while the stream is down or off, and every **5 min** while the stream is connected.
  - The stream raises the count from the event's own `count`. Reconnecting, refocusing the tab or opening the drawer brings it back in line with the server.
- **Drawer** (`components/NotificationDrawer.tsx`):
  - Tabs: All, Unread (filtered on the server), Mentions, Assigned to me (both filtered in the client over the pages already loaded).
  - Groups: Today and Earlier. The list loads more as you scroll.
  - **Task stacks** (`lib/taskStacks.ts`): rows that share a task id (for example `task.reminder`, `task.due` and `task.overdue` for one task) are drawn together, anchored at the newest. Each row keeps its own unread state and mark-read.
  - The drawer has no close button (Esc or an outside click closes it). The header shows the raw unread count, not capped.
  - "Assigned to me" matches `task.assigned`, `task.reminder`, `task.due`, `task.overdue` and `lead.assigned`.
  - It has loading and error states.
  - Clicking a row marks it read and opens its target (`lib/deepLink.ts`):
    - contact, company, lead or deal events: that record
    - task events: the Tasks list (no single-task route)
    - `lead.assigned`: `/leads/{id}`
    - `member.*`: Settings › Team; `wallet.low_balance` and `billing.*`: Settings › Billing
    - `inbox.mailbox_disconnected`: Settings › Integrations
    - `sequence.replied`: the inbox thread (else the contact)
    - `call.missed`, `sms.received`, `sms.opted_out`: the inbox call or SMS thread (else the contact)
    - `flow.errored`: the automation (`/automations/{id}`), or Engage › Sequences for a sequence
    - `automation.sends_paused`: Settings › Integrations (email) or Phone numbers (SMS)
    - imports: the entity list
    - `automation.sends_resumed`: no target
  - `sequence.replied` rows have a second button, **View sequence**, shown only when the row carries both `flow_id` and `enrollment_id`. It resolves `GET /flows/{id}` and opens that sequence builder or automation, falling back to `/engage?tab=sequences`.
  - "Mark all read" is optimistic and rolls back on error. It is disabled when nothing is unread or while a request is pending.
  - The footer links **Notification settings**.
  - Every navigation from the drawer goes through the unsaved-changes guard.
- **Popups** (`NotificationPopupHost`, `lib/notificationPopup.ts`):
  - They appear only when a live `created` event arrives, so they need the stream to be on.
  - They sit top-right, newest on top, at most 3. Each lasts 7s. Hovering or focusing holds a card, and leaving restarts its timer at 3s.
  - No card shows for the page the user is already on, or while the drawer is open.
  - The card's content comes from re-reading the newest unread notification. If that read fails, or finds no unread row, the card shows a count/type summary instead, with no Open link.
  - A short sound plays (Web Audio) on every live `created` event, even when popups are off or the drawer is open. There is no mute setting for it.
  - The **Popup cards** switch is stored in this browser only (localStorage), because the preferences API has no field for it.
- **Realtime stream** (SSE):
  - The client opens `GET /realtime/stream` only when the build flag `NEXT_PUBLIC_REALTIME_NOTIFICATIONS_ENABLED=true` is set. It is **off by default**.
  - The connection goes through the BFF proxy, so the browser never holds a token.
  - Events are content-free hints. The client re-reads the normal endpoints.
  - A `503` (backend kill switch, or Redis down) or a `429` keeps the client on polling. A `401` or `403` stops it for good.
  - The stream is closed while the tab is hidden.
- **Preferences** (`/settings/notifications`, `GET`/`PATCH /settings/notifications`). The page has these cards:
  - **Live updates.** Sets `realtime_enabled`. When off, the notifier stops sending this user live `created` hints, but the stream itself stays connected (the hub never reads the flag). So popups and the sound stop, and the badge, which polls only every 5 min while the stream is connected, can lag by up to 5 min. Rows and history are unaffected. The card's copy still says it "only stops the instant in-page nudge".
  - **Popup cards.** Per device, as above.
  - **One section per category.** Each type has an **In-app** switch and a **Push** switch.
    - Turning In-app off suppresses the row entirely (no badge, no history) and also suppresses push. The UI does not show this: the Push switch stays enabled and keeps showing its own value.
    - Turning Push off stops only phone delivery.
    - Each switch shows the stored override if there is one, otherwise the server default (from `defaults` in the GET response), otherwise on.
    - Setting a switch back to its default **deletes** the override (PATCH with `null`) only when both surfaces are then at their defaults. Otherwise the switched surface is stored, even at its default value.
    - These push types are off by default: `task.due`, `task.overdue`, `inbox.mailbox_disconnected`, `sms.received`, `sms.opted_out`, `billing.payment_recovered`, `automation.sends_resumed`.
- **Push devices.** Mobile only. The mobile app registers an FCM token with `POST /notifications/devices`. Only Android can currently receive pushes. iOS and web registrations are stored but not delivered to.
- **Member events refresh caches.** `member.role_changed` makes the role and capabilities reload (`GET /users/me`) and the member list refresh. `member.joined` refreshes the member list.

## Permissions
- Every endpoint here is for any active member and acts only on the caller's own data. There is no capability gate, so an admin cannot change a colleague's preferences.
- Marking read is limited to the caller's own rows. An unknown id, or another member's, returns not found.

## API
- `GET /notifications` (`channel`, `unread`, `cursor`, `limit`: 0 or less becomes 25, over 100 becomes 100; never rejected)
- `GET /notifications/counts`
- `POST /notifications/{id}/read`
- `POST /notifications/read-all`
- `GET /settings/notifications`
- `PATCH /settings/notifications`
- `GET /realtime/stream` (SSE, behind the frontend flag)
- `GET /flows/{id}` (resolves the "View sequence" target)

## API only (no UI yet)
- `GET`/`POST`/`DELETE /notifications/devices`: push device registration, used by the mobile app only. The web app has no device list.

## Not built (lives in the vision design)
- Workspace notification defaults for admins (`settings-notification-defaults`).
- Email notifications: the "Email" column in the prototype's event matrix, the "Emailed" row tag, and the daily digest. The real columns are In-app and Push.
- "Reset to workspace default" on the personal page.
- The prototype's own event list (email opened, deal stage changed, deal won, automation completed). These are not backend types.
- Announcements / "What's new": the drawer tab, the bottom-right spotlight card, the unseen-update dot on the bell, and the Announcements card in settings.
- The bell items the prototype builds in the browser: today's meetings and calendar events, and its own overdue / due-today task rows. In the app, task reminders are real backend `task.*` notifications.
- The banner-states preview page (`settings-notification-states`) and its "In-app notices · Preview every state" card. These are design tools.
- Workspace banners other than trial and read-only (maintenance, export ready, SMS blocked, payment failed or cancelled as their own strip rows).

## Known gaps
- Popups never fire unless the realtime flag is on, and it ships off. The **Live updates** and **Popup cards** switches therefore do nothing visible in a build without the flag.
- There is no single-task deep link. Task notifications open the Tasks list.
- The keys inside `data` have undocumented casing (`additionalProperties: true`), so the deep-link code reads both snake_case and camelCase.
- Billing notifications open the top of the Billing tab, not a section of it.
- The stream sends no `read` events yet, so marking read in another tab is only picked up at the next refresh.
- The notification sound cannot be muted.
- The `/inbox` Push tab is always empty: every notification, including one pushed to a phone, is stored as `channel='inapp'`.
- Turning **Live updates** off makes the badge lag by up to 5 min (the stream stays connected, so polling stays at 5 min), which the card's copy does not say.
- The web app has no way to register a push device. Push reaches users only through the mobile app.
