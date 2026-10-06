# Ticket template

Use this for every ticket that comes from an approved proposal.

```
Summary: <verb> <thing> (<area>)

Spec: nrtur-product/proposals/<file>.md  (approved in PR #<n>)
Screens: design.nrtur.io → <page id>[, <page id>]
Area doc: nrtur-product/docs/areas/<area>.md

What to build
- <the slice of the proposal this ticket covers>

Acceptance criteria
- <testable statements, copied or narrowed from the proposal's Rules>

Out of scope
- <what the next ticket covers, or what the proposal excludes>

API
- <new or changed endpoints, or "none"> (the contract change goes in nrtur-backend/api/openapi.yaml)
```

Rules:
- A ticket links to an **approved** proposal. No proposal means no ticket, except for bugs and drift fixes.
- Refer to screens by page id, never by line number.
- If the ticket needs something the proposal doesn't say, update the proposal first.
