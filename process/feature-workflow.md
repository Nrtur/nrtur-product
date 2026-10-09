# Feature workflow

The flow is: design and docs first, approval second, code third. The goal is that nobody, human or AI, builds from a guess or from the vision archive.

## 1. Propose

- Branch off `main` in this repo.
- Copy [`proposals/_template.md`](../proposals/_template.md) to `proposals/YYYY-MM-<slug>.md` and fill it in. Set `status: proposed`.
- Change the design in `design/index.html`.
  - If the idea exists in the vision archive, copy the relevant screen from `nrtur-design-vision` and **trim it to what you actually want**. Do not bring the rest of the vision along with it.
- If a rule, limit or price changes, update the matching file in `docs/areas/` or `pricing/` in the **same PR**.
- Add a row for the feature to the "Screens from proposals" table in `design/README.md`, so nobody mistakes the new screens for built ones.
- `docs/areas/` describes what is **built**. A not-yet-built feature's rules live in its proposal until close-out. Only cross-cutting files (pricing, permissions) get a short pointer to the proposal.
- Open the PR using the template. Screenshots of the changed screens help reviewers.

## 2. Approve

- The owner reviews the PR. Comments get resolved in the PR.
- **Merging means approved.** The proposal's status moves to `approved` in the merge commit.
- Anything not merged is not approved, however good it looks on a branch.

## 3. Ticket

- Create Jira tickets from the merged proposal using [ticket-template.md](ticket-template.md).
- Each ticket links to the proposal file, and to the screens by **prototype page id** (for example `contact-detail`), never by line number. Line numbers shift on every edit.
- Write the ticket keys back into the proposal's `tickets:` field.

## 4. Build

- Code PRs reference their ticket.
- If, while building, the design turns out to be wrong or impossible, change it **here** first with a small PR. Then build the corrected version. Don't let the code quietly diverge.

## 5. Close out

When the feature ships to production:

- Move the content of the proposal into the relevant `docs/areas/<area>.md`.
- Update its `verified_against` and `verified_on` stamps.
- Set the proposal to `status: built`. The proposal stays as history.
- Remove its row from the "Screens from proposals" table in `design/README.md`.

## Keeping it in sync

- **Drift found** (the app differs from this repo with no proposal behind it): open an issue labelled `drift`. Then either fix the code or open a sync PR here. The owner picks which.
- **Bug fixes** that don't change behaviour need nothing here.
- **Each release**, someone re-checks the area docs they touched and bumps `verified_on`.

## Status values

| Status | Meaning |
|---|---|
| `proposed` | Open PR, under discussion |
| `approved` | Merged; tickets may be created and work may start |
| `in-progress` | Tickets are being worked |
| `built` | Shipped; content folded into `docs/areas/` |
| `dropped` | Decided against; kept for the record |
