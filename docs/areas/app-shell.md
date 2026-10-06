---
area: App shell
status: built
verified_against: frontend@1bb6aa5d, backend@577bae44
verified_on: 2026-10-06
---
# App shell

## What it does
The app shell is the frame around every signed-in page. It has a fixed 56px icon rail on the left (a bottom tab bar on mobile), a quick-add "+" menu, global search (Cmd/Ctrl+K), an "All destinations" flyout, the notification bell, a light/dark toggle, Settings, and the profile avatar. Each section supplies its own top bar. The shell does not own one. Above everything sits one workspace-wide banner strip, used today for trial and read-only notices. The shell also runs a few headless jobs: it tracks background CSV imports, keeps the live notification stream open, loads workspace settings, and asks before you leave a page with unsaved work.

## Screens
| Route (app) | Prototype page id | Purpose |
|---|---|---|
| every authenticated route (`AppShell`) | `AppSidebar` (component, not a page) | Rail, quick add, search, flyout, bell, theme, settings, profile |
| n/a (overlay) | `GlobalSearchOverlay` (component) | Cmd/Ctrl+K search across records, pages and settings |
| n/a (popover) | `QuickAddMenu` (component) | Create one record, several records (sheet), bulk import, Build & automate |
| n/a (flyout) | `NavMoreDrawer` + `DestinationGrid` (components) | "All destinations": a grouped, searchable card grid |
| n/a (drawer, mobile only) | `MobileNavDrawer` (component) | Full navigation on phones |
| `/dashboard` top bar | `AppTopbar` (component) | Each page's header. Only the dashboard uses the shared `TopNav` |

## Behaviour and rules
- **Rail order (desktop)** (`src/components/layout/sidebar.tsx` `PRIMARY_ITEMS`): brand mark (opens `/dashboard`), Quick add, Search, divider, then **Dashboard, Contacts, Pipeline, Tasks, Inbox**. The bottom cluster is All destinations, Notifications, Theme toggle, Settings, Profile avatar (opens `/profile`). The code also lists Calendar and Reports, but as disabled "Soon" buttons, so they are cut here.
- The rail is fixed. Users cannot pin, reorder, expand or collapse it, and cannot add saved views or custom links.
- **Rail badges.** Tasks shows a number: due today plus overdue (`GET /tasks/counts`). The bell shows the unread count (`GET /notifications/counts`). Both cap at "99+" and hide at 0. There are no severity "needs fix" dots.
- **All destinations flyout** (`destinations.ts`, `all-destinations-drawer.tsx`). It opens on hover after a 120ms delay and closes 200ms after the pointer leaves. A click pins it open, and Esc or an outside click closes it. It has a search box that matches label, description or group name. Built groups:
  - Overview: Dashboard
  - Daily work: Inbox, Tasks
  - Records & sales: Contacts, Companies, Leads, Pipeline
  - Marketing & insight: Engage
  - Revenue: Billing (`/settings?tab=billing`)
  - Workspace: Settings, Integrations (`?tab=integrations`), Team (`?tab=team`)

  There is no footer, so no "Explore all features" and no "Customize" buttons.
- **Mobile.** Below `md` the rail becomes a bottom bar of the pinned links plus Settings (Dashboard, Contacts, Pipeline, Tasks, Inbox, Settings), and the "+" becomes a floating button. A hamburger opens `MobileNavDrawer`. It has four tiles at the top (Add, Search, Alerts, Theme), one "Navigation" list built from the full destination catalogue, and a profile footer.
- **Quick add** (`features/quick-add`). It opens on click (no hover-open) and has two modes:
  - Single record: Lead, Contact, Company, Deal, Task, Email, Call. Email is disabled, with a reason, when no mailbox is connected. Call opens "log call".
  - Multiple records: a spreadsheet `RecordSheet` for Contacts, Leads, Companies or Deals.

  "Bulk import" opens the CSV wizard for contacts, companies and deals. Lead import is not wired here. The "Build & automate" column has Automation (`/automations/new`) and Sequence (`/sequences/email/new`), both owner/admin only. Its Form, Email template and Create intake form entries are "Coming soon", so they are cut. When the plan's record cap blocks creating records, the dashboard "New Contact" button is disabled with the reason as a tooltip.
- **Global search** (`sidebar-search.tsx`, `features/search`). Cmd/Ctrl+K opens the search overlay. There is no separate command palette. Record search is one server call (`GET /search`) across contacts, companies, leads, deals and tasks:
  - At least 3 characters, at most 128.
  - Results are grouped by type and never merged, because scores are not comparable across types.
  - Each group has a "See all in X" link.
  - The server rate-limits at 120 requests per window per user.

  The overlay also has two local, fuzzy-matched groups, "Pages" and "Settings". The Settings group links General, Duplicates and Billing only. With an empty query the overlay shows Recent searches (the last 6, kept in localStorage, and clearable) and "Jump to" for built pages. Navigation from the overlay goes through the unsaved-changes guard.
- **Banner strip** (`features/notifications/components/GlobalBannerHost.tsx`). The strip is one row, fixed at the top above the rail, and pushes the rail and content down by its height. Its sources are `TrialBanner` (dismissible: info above 3 days left, warning at 3 days or fewer). Dismissal is stored per workspace **per exact message** (`nrtur:trial-banner-dismissed:{workspaceId}:{text}`). The text carries the days-left count, so a dismissed banner comes back the next day and `ReadOnlyBanner` (critical, cannot be dismissed). `ReadOnlyBanner` renders only while the workspace is read-only, and its clause before "— this workspace is read-only" depends on the payment state (trial ended, plan ended, subscription canceled). A past-due workspace that is not yet read-only gets no strip. It shows the most severe notice, and "+N more" opens a 440px list of the rest. Only the owner gets the "choose a plan" link. Everyone else is told to ask the owner.
- **Toasts.** Sonner sits at bottom-center (`app/providers.tsx`).
- **Theme.** next-themes with a `class` attribute. Dark is the default and system preference is honoured. The rail button toggles light and dark. The rail follows the theme: it is dark in dark mode and light (`--paper`) in light mode (`html:not(.dark) .app-sidebar` in `globals.css`). Accent colour is `#2fa877` in dark mode and `#1d7c53` in light mode (`globals.css`). Users cannot change accent, font, density or rail colour.
- **Unsaved-changes guard** (`navigation-guard.tsx`). `NavigationGuardProvider` wraps the shell. Today only the automation builder registers a blocker (`features/automations/hooks/useNavigationGuard.ts`). That blocker intercepts in-app link clicks, browser back and forward, navigation the shell triggers in code (search, notification rows), and tab close or refresh (the native prompt). After a 401 session expiry the guard is suppressed.
- **Import tracker** (`features/import/components/ImportBatchTracker.tsx`). It has no UI. It keeps polling `GET /imports/{id}` for any accepted (202) import after the wizard closes. The active batches are kept in sessionStorage. When an import finishes, it raises one toast (success, warning, error or info) and refreshes the list. If the plan refused rows, the owner's toast carries an "Upgrade" action.
- **Workspace switcher** (`features/workspace/components/WorkspaceSwitcher.tsx`). The component exists, but nothing mounts it, and the backend allows one workspace per user, so it never shows.
- **Dashboard top bar** (`top-nav.tsx`). It shows a greeting ("Good morning/afternoon/evening, {first name}") and the date, followed by "— Here is where things stand today." (hidden below 432px). The **New Contact** button is hidden below `md`.
- The softphone (calls) and the notification popup host are also mounted by the shell. They are covered in their own areas.

## Permissions
- Every member sees the same rail and flyout.
- Quick add: Automation and Sequence need owner/admin (`config:manage`). The API enforces this regardless of the UI. Record creates are gated by plan caps, not by role.
- Banner: only the owner gets the checkout link (`POST /billing/checkout` is owner-only).
- Search: any active member (`GET /search`, no capability gate).
- The legacy backend role `manager` is treated as admin (`internal/platform/authz/policy.go`). Only owner, admin and member are assignable.

## API
- `GET /search`
- `GET /tasks/counts`
- `GET /notifications/counts`
- `GET /imports/{id}`
- `GET /users/me` (role, for the quick-add gates)
- `GET /workspaces/{id}` (workspace bootstrap: name, industry, timezone)
- `GET /workspaces` (switcher; not mounted)

## API only (no UI yet)
- Switching workspace: the BFF `app/api/auth/switch-workspace` plus `GET /workspaces`. Dormant because a user can belong to only one workspace.

## Not built (lives in the vision design)
- Rail customisation: pinning, reordering, a 12-item cap, saved-view pins, custom links, per-user and admin nav defaults (`NavCustomizeDrawer`, `settings-nav-customize`).
- The expandable rail with labels (`nav-expand`).
- Rail entries for Calendar, Reports and Payments.
- Severity nav dots (`navBadgeMap` / `NavDot`) on rail items, flyout cards and Engage tabs.
- The Explore page (`explore`) and the flyout footer ("Explore all features", "Customize").
- The command palette (`CommandPalette`, `CMD_ITEMS`) with Navigate / Create / Recent groups and shortcuts.
- Search over custom-object records, and search entries for cut pages.
- Quick add entries: Form, Email template, Create intake form, and Import leads.
- "View as role" (`ViewAsSwitcher`, `RolePreviewBanner`). Roles are owner, admin and member only.
- The "What's new" announcement spotlight (`AnnouncementSpotlight`).
- Workspace display settings: accent, rail colour, font, density (`TweaksLayer`, `settings-themes`).
- The footer's "All systems operational" dot and "N team members online" (`AppFooter`).
- The dev page jumper (`DevNav`). This is a design tool.

## Known gaps
- The search "Pages" list in the app is stale. Tasks and Inbox are labelled "Coming soon" even though both routes are built, and Engage is missing. The Settings group lists only General, Duplicates and Billing, though Team, Compliance, Tasks & reminders, Custom fields, Tags, Statuses, Integrations, Phone numbers and Notifications are built.
- The dashboard top bar has a **Customize** button with no handler. It does nothing, so the trimmed design drops it.
- `QuickAddMenu` "Bulk import" lists leads as "coming soon", but the backend has a live lead importer.
- The app's flyout describes Engage as "Sequences, automations, funnels, forms & templates" (`destinations.ts`). Funnels, forms and templates are not built, so the design says "Sequences, automations & rules".
- Only the automation builder uses the unsaved-changes guard. Other forms (record sheets, settings) use their own close confirmations or none.
