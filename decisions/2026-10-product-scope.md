# Product scope reset (2026-10-06)

Context: the full design prototype had grown about 2–3× larger than the built product, and three different documents described three different permission models. Developers and AI agents were building towards screens that had never been approved.

## Decisions

1. **The full prototype is archived as the "vision".** The repo `nrtur-design` was tagged and renamed to `nrtur-design-vision`, made read-only, and is served at vision.nrtur.io. It is an idea bank, not a spec.
2. **Design and product docs live together in `nrtur-product`.** A feature is one PR holding design, docs and pricing, so it is one approval. design.nrtur.io serves this repo's `design/`.
3. **The synced design shows only what is built or approved.** Features shown as "Soon" or disabled in the app are left out completely, not shown greyed out.
4. **Roles are owner / admin / member**, as built. The 6-role model with own/team/all record scope and "View as role" stays in the vision archive.
5. **The trial is 21 days.** See also [2026-09-billing-and-limits.md](2026-09-billing-and-limits.md).
6. **Automated sends per month: Team 1,000 · Pro 7,500 · Business 35,000.** These replace the old 1,000 / 25,000 / 100,000.
7. **The repo is public** so GitHub Pages can serve the design on the free plan. As a result, no secrets, internal hosts or customer data are ever committed.
8. **`nrtur-docs` is retired.** Its current content (pricing v4, personas) moved here, corrected. Everything else is archived.
9. **Mobile is out of scope** for this reset and will be brought in later.
10. **Mailboxes: Gmail only, for now** (2026-10-09). Microsoft 365 waits on SCRUM-1043. IMAP (Business email, Other) is untested end to end. Yahoo and iCloud are not accepted by the backend. Any of these comes back through a proposal once it has been verified.
