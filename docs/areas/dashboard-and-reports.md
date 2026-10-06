---
area: Dashboard and reports
status: built
verified_against: frontend@1bb6aa5d, backend@577bae44
verified_on: 2026-10-06
---
# Dashboard and reports

## What it does
The dashboard is the home page after sign-in. It shows a fixed set of live widgets: three pipeline KPIs (pipeline value, win rate, weighted forecast), a "Deals by Stage" bar list for the default pipeline, and, for owners and admins, a "Recent Activity" feed. The layout cannot be customised. Users cannot add, remove, resize or configure widgets, and roles do not get different layouts. There is no analytics or reports module. The `/reports` route hosts only an **Activity Feed** page ("My Activity" / "Team"), which owners and admins reach from the dashboard's "View all" link.

## Screens
| Route (app) | Prototype page id | Purpose |
|---|---|---|
| `/dashboard` | `dashboard` (`DashboardPage`) | Greeting top bar and a fixed widget grid |
| `/reports` | `reports` (`ReportsPage`) | Activity Feed: My Activity / Team, newest first, "Load more" |

## Behaviour and rules
- **Grid** (`features/dashboard/components/DashboardDefaultGrid.tsx`). Dense CSS grid: 1 column on mobile, 2 at `md`, 3 at `lg`. Rows are at least 140px. Max width is 1480px. Spans: S = 1×1, M = 2×1.
- **Live widgets, in order:**
  1. **Pipeline Value** (S). Open pipeline value in the forecast currency (default USD). Sub-line: "N open deals", or "Open deal count unavailable".
  2. **Win Rate** (S). `win_rate` as a %. Sub-line: "W won / N closed", or "No closed deals yet" with the value shown as "—".
  3. **Weighted Forecast** (S). Sub-line: "Open value × probability".
  4. **Deals by Stage** (M). The default pipeline's stages, each with total value, deal count and a bar scaled to the largest stage (not a share of the total). The header sub-line is "{total} · pipeline", and there is no link to the board. Won is green and lost is red.
  5. **Recent Activity** (M). The workspace's 5 newest activities: a type chip, the summary (or the type label if there is none), and "actor · channel" with a relative time. A "View all" footer goes to `/reports`. Rows do not link anywhere because the activity carries no entity link. **Only owners and admins can see it.** For members it is not rendered and not fetched.
- The three KPI tiles are not clickable; their only control is Retry on error.
- All three KPIs share one `GET /deals/forecast` query. Every live widget has loading, error-with-Retry and empty states.
- **The widgets do not show period-over-period deltas.** The prototype's "+12%" style trend chips do not exist.
- **Placeholders that are cut.** The app renders five "Coming soon" tiles from `widgetCatalog.ts` `DEFERRED_WIDGETS`: Needs Follow-up, New Leads, Lead Funnel, Top Contacts and Top Accounts. They have no data source, so they are not built and are removed from the design.
- **Top bar** (`components/layout/top-nav.tsx`). The greeting plus the date, then "New Contact" (opens the quick-add contact sheet). It shows for every role and is disabled, with the reason as a tooltip, when the contacts cap blocks creation (`useCapGate("contacts")`, including a read-only workspace). The Customize button has no handler and is cut.
- **Activity Feed** (`/reports`, `features/activity/components/GlobalActivityFeed.tsx`):
  - Title "Activity Feed", subtitle "Real-time activity across your CRM".
  - A toggle switches between **My Activity** (`actor_id` = me) and **Team** (whole workspace).
  - It loads 50 rows at a time, and "Load more" adds 50 until the 100-row ceiling. A "Load more" button shows while more rows exist; at the ceiling the footer reads "Showing the first 100 activities." On the backend, a `page_size` outside 1–100 falls back to 20 (it is not clamped).
  - On Team, a member sees "Owner or Admin access required" and nothing is fetched.
  - There is no link to this page from the rail, search or flyout. The rail's "Reports" entry is a disabled "Soon" item, which is cut.

## Permissions
- **Forecast KPIs and Deals by Stage:** any member. `GET /deals/forecast` has no role gate. The deal lists follow the deals area's rules.
- **Recent Activity widget and Team feed:** owner and admin in the UI only. `GET /activities` has **no** role gate on the server, so any member who calls the API gets the whole workspace feed (see Known gaps).
- **My Activity:** any member, filtered by `actor_id`.

## API
- `GET /deals/forecast`
- `GET /pipelines` (default pipeline)
- `GET /deals` (stage rows for the default pipeline)
- `GET /activities` (`page_size`, `actor_id`; it also accepts `type`, `entity_type`, `since`, `until` and paging)

## API only (no UI yet)
- `GET /activity-feed`: the dashboard global/team feed with subject descriptors for deep-linking and a server-side `scope=me|team`. The frontend uses `GET /activities` instead.
- `GET /deals/forecast` filters (`owner_id`, `pipeline_id`, date range). The dashboard does not use them.

## Not built (lives in the vision design)
- Dashboard customisation: edit mode, drag to reorder, snap resize, widget config drawer (source, filters, date range, chart type), the widget library, layout templates, and Reset / Save as workspace default.
- Role-based starting layouts (`DASH_ROLE_PRESETS`, `DASH_ARCHETYPES`, `DASH_TEMPLATES`) and role-restricted widgets (`WIDGET_ROLE_RULES`).
- Every widget except the five live ones (around 40 definitions in `WIDGET_DEFS`), including the five placeholders.
- Delta and trend chips on stat widgets.
- A Reports analytics page: per-role report presets, the widget engine and charts. In the design, `ReportsPage` is already the Activity Feed.
- Settings › Appearance › Dashboards (`settings-dash-customize`).
- The dashboard footer ("N widgets · hidden for your role").

## Known gaps
- **Security:** `GET /activities` is not role-gated on the server. Hiding the team feed from members is done only in the frontend. If team-wide activity must be admin-only, the backend needs the check.
- The app still renders the five "Coming soon" placeholder tiles. Once they are removed, the app grid will have holes until it is re-tiled. The trimmed design shows only the five live widgets.
- The **Customize** button in the dashboard top bar is inert.
- The `/reports` route name does not match its content (an activity feed). The Reports rail entry stays "Soon" while the page is reachable through "View all".
- `RecentActivityWidget`'s doc comment says there is no "View all" link, but the code renders one. The code is correct.
- Deals by Stage covers the default pipeline only.
- Activity rows do not deep-link, because `GET /activities` returns no subject linkage.
