---
area: Workspace settings
status: built
verified_against: frontend@1bb6aa5d, backend@577bae44
verified_on: 2026-10-06
---
# Workspace settings

## What it does
Settings is where a workspace is configured and where each person manages their own account. `/settings` is one screen with a grouped rail on the left, a per-page title bar, and the active page in the body. The page is chosen with `?tab=`. Owners and admins can rename the workspace and set its defaults, and they can invite and manage teammates. They also set the legal mailing address that automation emails need, set task defaults, and manage custom fields, tags and statuses. Owners and admins can also merge duplicates and connect email accounts. Members can open every page, but workspace pages are read-only for them, with a "Managed by your admin" notice, with three exceptions: on **Tags** members can create (not delete); **Duplicates** is fully usable by members (scan, dismiss, merge), with no notice; and **Integrations** shows no notice but hides Connect and Disconnect from members (a member can still connect a mailbox from the Inbox; `POST /inbox/accounts/connect` has no role check). Each user's own profile, password and sign-out live on `/profile`. Billing, phone numbers, pipeline/deal settings and notification settings sit in the same menu but are documented in their own area docs.

## Screens
The Settings menu, exactly as the app ships it. Every item not listed here is cut.

| Route (app) | Prototype page id | Purpose |
|---|---|---|
| **My account** › Profile & account: `/profile` | `profile` | Name, phone, job title, language, avatar, change password, sign out. |
| **My account** › Notifications: `/settings/notifications` | `settings-notifications` | See notifications area doc. |
| **Workspace** › General: `/settings?tab=general` | `settings-general` | Workspace name, industry, company size, timezone, currency. Danger zone: delete workspace. |
| **Workspace** › Team: `/settings?tab=team` | `settings-team` | Members and pending invites in one table: invite, resend, revoke, change role, remove. |
| **Workspace** › Compliance: `/settings?tab=compliance` | `settings-compliance` | Postal mailing address and unsubscribe base URL for automation emails. |
| **Workspace** › Tasks & reminders: `/settings?tab=tasks` | `settings-tasks` | Workspace defaults for new tasks and reminders. |
| **Objects & fields** › Custom fields: `/settings?tab=custom-fields[&object=]` | `settings-properties` | List, create and delete custom fields per object. |
| **Objects & fields** › Tags: `/settings?tab=tags` | `settings-tags` | Search, create and delete workspace tags. |
| **Objects & fields** › Statuses: `/settings?tab=statuses` | `settings-statuses` | Lead and contact status lists, plus a pointer to deal stages. |
| **Data management** › Duplicates: `/settings?tab=data` | `settings-duplicates` | Find and merge duplicate contacts, companies and leads. |
| **Integrations** › Integrations: `/settings?tab=integrations` | `settings-integrations` | Email accounts (connect, reconnect and disconnect mailboxes). |
| **Integrations** › Phone numbers: `/settings?tab=phone-numbers` | `settings-phone-numbers` | See phone-and-sms-setup area doc. |
| **Billing** › Billing & usage: `/settings?tab=billing` | `settings-billing` | See billing area doc. |

The menu has 6 groups: My account, Workspace, Objects & fields, Data management, Integrations, Billing. The prototype's Appearance group has no built item and is cut.

## Behaviour and rules
**Shell**
- `SettingsPage` / `SettingsNav` in `src/features/settings/`.
- An unknown `?tab=` falls back to General.
- Every group is shown to every role, with no lock icons and no role-based hiding. Gating happens inside each page.
- Hovering or clicking a group opens a flyout: a full-height panel on desktop, a short drop-down under 768px.

**General** (`features/workspace/WorkspaceGeneralSettings`)
- Fields:
  - Name is required. A 409 means the name is taken, shown on the field.
  - Industry and Company size are optional.
  - Timezone uses a searchable picker.
  - Currency offers USD only.
- Options come from `GET /workspaces/settings/options`. Language is kept in the save payload unchanged, because it is a personal setting.
- **Danger zone** (owner only): delete workspace with a type-to-confirm dialog. The name must match exactly (`confirm_name`).
  - The backend **soft-deletes** the workspace.
  - The UI then shows "Workspace deleted" with a Sign out button. It does not log out on its own.

**Team** (`features/workspace/TeamSettings`)
- One table merges active members and live pending invites.
  - Role pill: Admin or Member are editable on member rows. Owner and Manager show as read-only badges, and a pending invite always shows a read-only badge (there is no endpoint to change an invite's role).
  - Status: Active or Pending.
- Header: "N members · M pending". Pending invites and their count are shown to owners and admins only.
- `?member=<id>` highlights that member's row (the link from the member-joined notification).
- **Invite dialog:** one email plus a role (Admin or Member), and a seat line such as "2 of 3 seats in use on the Trial plan".
  - Send is disabled at an enforced seat cap. The cap counts accepted members plus live pending invites.
  - At the cap the dialog adds a remedy line that depends on tier and role (`features/billing/lib/inviteSeatLine.ts`). The owner sees "Add a user" (paid plans; plus "or revoke a pending invitation" when one exists), "Choose a plan to add more" (Trial) or "Move to Team to add users" (Solo), with a link. Everyone else sees "Ask your workspace owner …" with no link.
  - Errors: `ALREADY_MEMBER` shows on the email field.
  - Re-inviting someone with a live pending invite returns the existing invite.
  - Each email can receive at most 10 invites per hour across all workspaces.
- Resend fires at once (a spinner, no confirmation), with a 60s cooldown and at most 10 resends per invite. Revoke and Remove each ask for confirmation.
- **Role matrix** (`workspaces/service/members.go`):
  - The owner manages everyone.
  - An admin manages members (and legacy managers), but never owners or other admins.
  - You cannot change your own role from the table.
  - The last owner can never be demoted or removed (`LAST_OWNER`).
- Removing a member is a soft remove, and their records are un-owned. The API supports `reassign_to`, but the UI doesn't use it.

**Compliance** (`features/workspace/WorkspaceComplianceSettings`)
- Postal mailing address: up to 500 chars.
- Unsubscribe base URL: optional, up to 300 chars. It must be a bare http(s) URL with no query, fragment or credentials. A trailing slash is trimmed.
- A live warning appears while the address is empty. In that state every automation email node refuses to send, because the backend fails closed.
- A preview line shows what recipients will see.
- Saving sends both fields every time (full replacement).

**Tasks & reminders** (`features/tasks/SettingsTaskDefaults`)
- Default reminder: None / 5 / 15 / 30 min / 1 hour / 1 day before.
- "Also send reminders by email" is a toggle.
- Default priority: Low / Medium / High.
- Default assignee: Me (task creator) or Unassigned only.
- Default due date: Today / Tomorrow / In 3 days / In 1 week / No due date.
- The defaults pre-fill the task drawer and quick-add.

**Custom fields** (`features/custom-fields/CustomFieldsSettingsPage`)
- Tabs: Contacts / Companies / Deals / Leads, with deep links via `?object=`. Search plus an "N fields" count.
- Rows show type, name, the Required badge, the options count and the permanent `field_key`, plus Delete.
- **New field drawer**: name, type, options (for selects) and Required.
  - The 9 types are Text, Number, Date, Single select, Multi select, Boolean, URL, Email, Phone.
  - The key is slugified from the name, shown before save, and checked for duplicates on the client.
- There is no edit, reorder, sections or placement. The page lists custom fields only, not built-in ones.
- Limit: the backend allows 100 per object type (`CUSTOM_FIELD_LIMIT_REACHED`). Plan caps are read for a pre-emptive notice.

**Tags** (`features/tags/SettingsTagsPage`)
- The list has search and a count. "New tag" takes a name (up to 100 chars) and a colour from 3 quick swatches or a custom picker.
- Delete asks for confirmation.
- There is no rename, recolour, merge or per-tag usage count; the delete confirmation shows no count either. The UI shows Delete only to owner, admin or manager (`canManageWorkspaceConfig`). Members still see "New tag" and a notice that starts "Deleting tags needs an admin." and ends "You can still create tags."

**Statuses** (`features/status/StatusesSettingsPage`)
- **Lead status card:** add, rename, recolour, drag to reorder, set default, delete.
  - Delete archives the status and **reassigns** its leads to the default.
  - Deleting the default is refused (409).
- **Contact status card:** add, rename, recolour, reorder, delete.
  - Delete archives the status. Contacts keep pointing at it and are **not** moved.
- Both cards show "N records" usage counts.
- A "Deal stages" pointer row links to the Pipeline board.

**Duplicates** (`features/settings/DuplicatesSettings` over `features/duplicates`)
- Header: "We found N possible duplicates…".
- Object tabs Contacts / Companies / Leads, each with a count.
- Toolbar: a **Minimum match confidence** select with four presets (All matches 40%+, Likely 60%+, High confidence 80%+, Near-certain 95%+), driving `min_confidence` (backend default and floor 0.4), the Cards/List toggle and "Scan now".
- A merge drawer for choosing field values. Activity timelines are always combined (fixed text, not a toggle); "Combine tags" is the only toggle, on by default.
- Dismiss lasts for the session only. The toast reads "Hidden for now": "Not a duplicate" isn't saved yet, and the pair comes back on reload.
- No bulk merge.
- The plan feature `duplicate_detection` can refuse with 422.

**Integrations** (`features/settings/SettingsIntegrationsPage`)
- The only wired content is the Email accounts card (from `features/inbox`):
  - It lists mailboxes with their sync state: Synced, Syncing…, Sync error, Reconnect needed or Disconnected.
  - Owners and admins get Connect account or Add another account, and Disconnect. Members see the list only, with no notice; the buttons are simply hidden.
  - A "Reconnect needed" row gets a **Reconnect** button for owners and admins. The plan's mailbox cap gates Connect only, never Reconnect.
- See Known gaps for the other items on the page.

**Profile** (`/profile`, `features/profile`)
- Identity header with avatar upload (`POST /users/me/avatar`).
- Fields: First name, Last name, Work email (read-only), Phone, Job title, Language.
  - Language options come from the backend, which lists English only.
  - Timezone is hidden and its value is round-tripped.
- **Change password** card: current, new and confirm.
- **Sign out** card:
  - "Sign out" is local only.
  - "Sign out of all devices" asks for confirmation, revokes every session, then signs out locally.

## Permissions
These rules are enforced by `internal/platform/authz/policy.go` plus service checks. The legacy `manager` role passes capability checks but is refused wherever a service checks for owner/admin literally: team management, General edits and mailbox disconnect (see [permissions.md](../permissions.md)).

| Area | Owner | Admin | Member |
|---|---|---|---|
| View any settings page | yes | yes | yes (most read-only; see the exceptions above) |
| General: edit (owner/admin literal check, not `workspace:manage`) | yes | yes | no |
| Delete workspace (`workspace:delete`) | yes | no | no |
| Team: invite, resend, revoke, list invites | yes | yes | no |
| Team: change role, remove | everyone | members only | no |
| Compliance: edit (`config:manage`) | yes | yes | no |
| Tasks defaults: edit (`workspace:manage`) | yes | yes | no |
| Custom fields, statuses: create/edit/delete (`config:manage`) | yes | yes | no |
| Tags: create/delete | yes | yes | yes at the API (the UI hides Delete for members) |
| Duplicates: scan and merge (not role-gated; plan feature `duplicate_detection`) | yes | yes | yes |
| Billing (`billing:manage`) | yes | no | no |
| Own profile, password, sessions | yes | yes | yes |

## API
All paths sit under `/api/v1`.

General
- `GET /workspaces/settings/options`
- `GET /workspaces/{id}`
- `PUT /workspaces/{id}`
- `DELETE /workspaces/{id}?confirm_name=`

Team
- `GET /workspaces/{id}/members`
- `PATCH /workspaces/{id}/members/{userId}`
- `DELETE /workspaces/{id}/members/{userId}`
- `GET /workspaces/{id}/invitations`
- `POST /workspaces/{id}/invite`
- `POST /workspaces/{id}/invitations/{inviteId}/resend`
- `POST /workspaces/{id}/invitations/{inviteId}/revoke`
- `GET /settings/usage` (for the seat line)

Compliance
- `GET /workspaces/{id}/compliance`
- `PUT /workspaces/{id}/compliance`

Tasks & reminders
- `GET /settings/tasks`
- `PATCH /settings/tasks`

Custom fields
- `GET /custom-fields/definitions`
- `POST /custom-fields/definitions`
- `DELETE /custom-fields/definitions/{id}`

Tags
- `GET /tags`
- `POST /tags`
- `DELETE /tags/{id}`

Statuses
- `GET /lead-statuses`
- `POST /lead-statuses`
- `PATCH /lead-statuses/{id}`
- `DELETE /lead-statuses/{id}`
- `PUT /lead-statuses/{id}/default`
- `GET /contact-statuses`
- `POST /contact-statuses`
- `PUT /contact-statuses/{id}`
- `DELETE /contact-statuses/{id}`

Duplicates
- `GET /{contacts|companies|leads}/duplicates`
- `POST /{object}/{id}/merge`

Integrations
- `GET /inbox/accounts`
- `POST /inbox/accounts/connect`
- `DELETE /inbox/accounts/{id}`

Profile
- `GET /users/me`
- `PATCH /users/me`
- `PUT /users/me/password`
- `POST /users/me/avatar`
- `POST /users/me/sessions/sign-out-all`

## API only (no UI yet)
- Suspend or reactivate a member: `POST /workspaces/{id}/members/{userId}/suspend|reactivate`. Reactivation takes a seat.
- Reassign a removed member's records: `reassign_to` on the member DELETE.
- Change your email address: `POST /users/me/email`, then `POST /users/me/email/verify`.
- Active sessions and login history: `GET /users/me/sessions`, `DELETE /users/me/sessions/{id}`, `GET /users/me/login-history`.
- Switching or listing workspaces: `POST /auth/switch-workspace`, `GET /workspaces`. The switcher is unmounted.

## Not built (lives in the vision design)
**Menu items** (all "Soon" in the app, so cut)
- My account: My calendar connection, My integrations, My preferences, Close my account.
- Workspace: Permissions, Permission matrix, Notification defaults.
- Objects & fields: Custom objects, Record layouts, List views.
- Data management: Import data, Export data, SLA rules, Approvals, Privacy & data retention, Recycle bin, Delete workspace (as a separate page).
- Appearance: Navigation, Dashboards, Themes & display, Branding, White-label.

**Role model**
- The 6-role model (Owner, Admin, Sales Manager, Team Lead, Sales Rep, Read Only).
- Custom roles, per-object CRUD and Own/Team/All scopes.
- The permission editor and comparison matrix.
- Sensitive-field masking.
- The "View as role" preview switcher and banner.

**In-page cuts**
- Team: activity audit log (a "Coming soon" card in the app), the "Preview invite page" button, the per-row "open invite link" button.
- Integrations: the app catalog (Slack, calendars, ad sources and others), the API keys panel, webhooks, the integration detail drawer, and the Twilio BYO connect.
- Custom fields: sections, drag reorder, placement ("In form" / "Column"), edit, hide, built-in field rows, and 7 extra field types (Long text, Rich text, Currency, Date range, Lookup, Autocomplete, Attachment).
- Tags: edit, rename, recolour, merge, usage counts.
- Duplicates: "Bulk merge high-confidence".
- Profile: two-factor authentication, GDPR data download, delete account, an editable email, and a timezone field.

## Known gaps
- **Fake Twilio connect.** Integrations still renders a "Twilio — advanced (BYO)" card whose connect flow is a UI simulation (`TwilioSetupModal.onSubmit` waits on a timer, then logs a TODO). It is not wired, so it is cut from the design. The app should hide it.
- **"Coming soon" banner.** Integrations shows an "Integrations catalog coming soon" banner, and Team shows a "Coming soon" audit-log card with a disabled Export CSV button. Both are cut from the design, but they are still in the app.
- **Tag delete not role-gated in the API.** `DELETE /tags/{id}` has no role gate, so any member can delete a tag through the API. The UI hides Delete from members.
- **Gate widths differ by page.** UI gates differ in how wide they are:
  - Statuses and custom fields allow owner/admin only (`isWorkspaceManager`).
  - Compliance and tag delete also allow the legacy `manager` (`canManageWorkspaceConfig`).
  - The backend is inconsistent too: manager passes capability checks but is refused by team management, General edits, mailbox disconnect and pipelines.
- **Delete-workspace copy.** The copy says data is "permanently removed", but the backend soft-deletes. Nothing handles a workspace-less account after deletion.
- **Statuses pointer.** The "Deal stages" row links to the board. Stage editing is documented in the deals area.
- **Custom field limit.** The backend enforces a fixed 100 per type. The per-plan field caps are reported but not enforced on this route.
- **Stale settings search.** Global search's Settings group lists only General, Duplicates and Billing (`components/layout/sidebar-search.tsx`).
- **Member role badge.** The Team table can show a legacy "Manager" badge for old rows. Managers cannot be assigned.
- **Admins are shown Team controls the backend refuses.** In `TeamMemberRow.tsx`, `roleEditable` and Remove check only `canManage && !isSelf`, so an admin sees the role select on other admins' rows and Remove on the owner's row. The backend (`members.go` `canManageMember`) refuses both with 403. The design correctly hides them.
