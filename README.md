# nrtur-product

The single source of truth for **what nrtur is**: the design and the product docs. Everything here matches the code that is actually built, plus anything that has been approved to be built next.

> **The rule:** if a feature is not in this repo, it is not approved.
> If the app and this repo disagree, one of them is a bug. File it; don't guess.

## What lives here

| Path | What it is |
|---|---|
| [`design/`](design/) | The clickable prototype, served at **design.nrtur.io**. It shows only screens that are built or approved. |
| [`docs/`](docs/) | Product docs: one file per product area, plus cross-cutting docs (permissions, glossary). |
| [`pricing/`](pricing/) | Plans, prices and limits. This is the one pricing source. |
| [`decisions/`](decisions/) | Owner decisions, dated. These explain *why* things are the way they are. |
| [`proposals/`](proposals/) | Features under review or approved but not yet built. |
| [`process/`](process/) | How a feature goes from idea to code, and the ticket template. |

## What does NOT live here

| Thing | Where it lives |
|---|---|
| API contract (endpoints, request/response shapes) | `nrtur-backend/api/openapi.yaml`, which CI keeps in sync with the code |
| How the code is built (architecture, testing, CI) | The `docs/` folder of each code repo |
| Ideas we have not approved | The vision archive, **vision.nrtur.io** (repo `nrtur-design-vision`). It is read-only and **not a spec**. |
| Mobile app | Not covered yet |

## How a feature gets built

1. **Propose:** open a PR here with a proposal, the design change and the doc change together.
2. **Approve:** the owner approves the PR, and merging it means the feature is approved.
3. **Ticket:** create tickets that link to the proposal and the screens.
4. **Build:** write the code against what was merged here.
5. **Close out:** when it ships, fold the proposal into the area doc and mark it `built`.

Full detail: [process/feature-workflow.md](process/feature-workflow.md).
