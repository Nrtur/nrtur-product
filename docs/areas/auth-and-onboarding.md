---
area: Auth and onboarding
status: built
verified_against: frontend@1bb6aa5d, backend@577bae44
verified_on: 2026-10-06
---
# Auth and onboarding

## What it does
This area covers everything a person goes through before they reach the CRM. They sign up with name, email and password (or with Google), confirm their email with a 6-digit code, and then walk through a short setup wizard: name the workspace, see the starter pipeline, and optionally add or import contacts. Returning users sign in with email and password or Google, and anyone can reset a forgotten password with an emailed code. Teammates join through an emailed invite link; they can accept with an existing session or create an account on the spot. One public page lives here too: the unsubscribe page that email recipients reach from an automation email's footer. There is no marketing landing page in the app. The root URL `/` always redirects to sign-in, and the marketing site lives in a separate repo.

## Screens
| Route (app) | Prototype page id | Purpose |
|---|---|---|
| `/` | (none; was `landing`) | Redirects to `/auth/login`. |
| `/auth/signup` | `signup` | Create an account: full name, work email, password, confirm password; or Google. |
| `/auth/verify-email?email=` | `verify-email` | Enter the 6-digit email code, with a 60s resend cooldown. |
| `/auth/login` | `signin` | Email and password, or Google. Links to forgot password and sign up. |
| `/auth/forgot-password` | `forgot-password` (and `reset-expired` as its expired state) | Three steps on one route (email, code, new password), plus an "expired" state. |
| `/auth/accept-invite?token=` | `accept-invite` | Validate an invite, then accept, decline, or create an account and accept. |
| `/onboarding` | `onboarding-1` … `onboarding-4` | Four-step wizard: Workspace, Pipeline, Contacts, Done. |
| `/unsubscribe/{token}` | (none, new) | Public page: confirm an email unsubscribe. |

## Behaviour and rules
**Sign up** (`POST /auth/register`)
- Client validation: name is 2–100 chars, email must be valid, password 8–100 chars, and the confirmation must match. The backend enforces name required, password ≥ 8, and a unique email (`users/service/service.go`).
- In one transaction, the backend creates the user (unverified), a placeholder workspace named "<name>'s Workspace (<random hex>)", and an owner membership. It also seeds the default contact statuses, lead statuses, the default pipeline and a wallet.
- The trial deadline is set to created_at + 21 days (`sqlc/models.go`). The signup side panel says "21 days Free trial".
- It sends a 6-digit OTP, then redirects to `/auth/verify-email?email=`.

**Verify email** (`POST /auth/verify-otp`, `POST /auth/resend-otp`)
- The code expires after 15 min, and 5 wrong attempts invalidate it. Resend has a 60s cooldown (429 `COOLDOWN_ACTIVE`).
- The UI auto-submits once all 6 digits are entered. On success the session cookie is set and the user goes to `/onboarding`.
- Opening the page without `?email=` redirects to `/auth/signup`.

**Sign in** (`POST /auth/login`, rate-limited per IP)
- A wrong email or password always returns "Invalid email or password." The error renders under the password field, so the layout does not shift.
- An unverified account gets 403 `EMAIL_NOT_VERIFIED`. The form shows "Email verification is required before sign in." It does **not** route to the verify screen; see Known gaps.
- A Google-only account gets 401 `NO_PASSWORD`.
- `?redirect=` is honoured for same-origin paths only (`sanitizeRedirect`).

**Google** (`POST /auth/google` with the GIS `id_token`)
- New users go to `/onboarding`; existing users go to the dashboard.
- Google must report the email as verified (`GOOGLE_EMAIL_UNVERIFIED` otherwise).
- If the client ID is missing or the GIS script fails to load, the button falls back to an "unavailable" message.
- There is no Microsoft sign-in.

**Forgot password**
- Step 1 (`POST /auth/forgot-password`): enter the email.
- Step 2: enter the 6-digit code. It goes through `POST /auth/verify-otp` with `purpose=password_reset`. The code lives 30 min and allows 5 attempts.
- Step 3: set a new password (8–100 chars, with confirmation) via `POST /auth/reset-password`. The reset token is held in a short-lived httpOnly cookie by the BFF and expires after 15 min.
- On success the user sees the toast "Password updated." and is sent to `/auth/login`. There is no confirmation screen.
- If the reset token has lapsed, the "expired" state offers "Request a new code", which restarts at step 1 on the same route.

**Accept invite**
- On load the page validates the token (`GET /auth/invitations/validate`), which returns valid, email, role, workspace name and expiry. No inviter name is available.
- **Signed in:** one-click Accept.
- **Signed out:** a name + password form (name 2–100 chars, trimmed; password 8–100) that creates the account and accepts in one step. A "Sign in to accept" link round-trips back to the invite.
- Decline (`POST /auth/decline-invite`) is terminal.
- Full-card terminal states: invalid token, expired, no longer pending (revoked or used), already a member, and already has a workspace. The last one exists because of the one-workspace-per-account rule.
- Invites expire after 7 days. The role is Admin or Member.

**Onboarding** (`/onboarding`, Zustand store persisted to sessionStorage and scoped to the user)
1. **Workspace**
   - Company name (2–100 chars) with a live, debounced availability check (`GET /workspaces/check-name`).
   - Industry: one of the 10 backend options, e.g. Agency / Consulting, SaaS / Technology, … Other.
   - Team size: Just me / 2–3 / 4–5 / 5+.
   - Timezone is auto-detected from the device and sent silently.
   - Saved with `POST /workspaces/{id}/setup`. The wizard advances only on success.
2. **Pipeline:** shows the single "B2B Sales" template as static content. Nothing is sent, and the backend has already seeded its own default pipeline.
3. **Contacts**
   - A grid with First / Last / Email / Phone / Title / Status columns and no Company column. A row is "ready" when it has a first name or a valid email.
   - Ready rows are created with `POST /contacts`, at most 3 in parallel.
   - "Import from CSV" opens the shared bulk import wizard, which runs in the background.
   - "Add manually later" skips ahead.
4. **Done:** a success screen with a single "Go to Dashboard" tile.
- "I'll do this later" (shown on steps 1–3) goes straight to the dashboard.
- Nothing marks onboarding complete on the server.

**Unsubscribe** (`/unsubscribe/{token}`, public)
- The page never submits on load, because mail scanners follow links. The opt-out is written only after an explicit button press (`POST /unsubscribe/{token}`).
- It never shows the address, because there is no GET for the token.
- States: confirm, in progress, unsubscribed (idempotent), invalid link (400, no retry), too many attempts (429), generic error, and incomplete link.
- Branding is nrtur only. The response carries no workspace identity.

## Permissions
- Every screen here is pre-auth or public, except onboarding, which needs a session.
- The sign-up user becomes **owner** of their new workspace.
- An invited user gets the invite's role, **admin** or **member**. Owner is never invitable. The legacy `manager` role is treated as admin by the backend.

## API
All paths sit under `/api/v1`.
- `POST /auth/register`
- `POST /auth/login`
- `POST /auth/google`
- `POST /auth/verify-otp`
- `POST /auth/resend-otp`
- `POST /auth/forgot-password`
- `POST /auth/reset-password`
- `GET /auth/invitations/validate`
- `POST /auth/accept-invite`
- `POST /auth/decline-invite`
- `GET /workspaces/check-name`
- `POST /workspaces/{id}/setup`
- `GET /workspaces/settings/options`
- `GET /contact-statuses`
- `POST /contacts`
- `POST /imports/contacts/analyze` · `/validate` · `/import`, and `GET /imports/{id}`
- `POST /unsubscribe/{token}`
- `GET /users/me` (role lookup after sign-in)

Session handling is a Next.js BFF under `/api/auth/*`. It sets httpOnly cookies and rebuilds the user from them. The backend has no `/auth/me` or logout endpoint, so logout only clears cookies.

## API only (no UI yet)
- `POST /auth/switch-workspace` and `GET /workspaces`: the `WorkspaceSwitcher` component exists but is unmounted, because the product allows one workspace per account.

## Not built (lives in the vision design)
- The marketing **landing page** (`landing`), with its hero, features and pricing. It lives in the separate marketing repo.
- "Continue with Microsoft" on sign-up and sign-in.
- Separate first-name and last-name fields on sign-up. The app uses a single full-name field.
- A standalone `reset-password` page. The reset happens inside `forgot-password`.
- The inviter's name on the invite page. The API does not return it.
- The demo-state switcher on the invite page.
- Onboarding: the Integrations step (Gmail/Slack/Calendar), the Invite-team step, multiple pipeline templates, and the Done tiles "Import contacts" and "Connect your tools".

## Known gaps
- **Unverified sign-in.** Signing in with an unverified email only shows an error. There is no link or route to `/auth/verify-email` to finish verification.
- **Onboarding re-entry.** Nothing stops an onboarded user from opening `/onboarding` again, and no completion flag exists server-side.
- **Pipeline step is presentational.** The template is never sent.
- **No Company column** in the onboarding contact grid. The create contract takes only `company_id`.
- **Stale session name.** The BFF rebuilds the user from cookies, so the name can be stale until the next sign-in (the profile save refreshes it).
- **Accept-invite card styling.** It still uses an always-dark card colour; the other auth screens follow the theme.
- **Unsubscribe rate limiting.** The page goes through the BFF, so per-IP limits may group all recipients into one bucket unless the backend trusts forwarded headers. This needs confirming with the backend.
- **Prototype trial copy.** The prototype's signup panel says "14 days Free trial"; the app and backend use 21 days.
