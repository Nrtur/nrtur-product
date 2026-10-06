# Instructions for AI agents

You are reading the nrtur product source of truth. Follow these rules.

1. **This repo is the spec.**
   - `design/index.html` shows only what is built or approved.
   - `docs/areas/*.md` describes how it behaves.
   - If the code you are working on disagrees with this repo, report the disagreement. Do not pick one side silently.
2. **The vision archive is not a spec.**
   - `nrtur-design-vision` (vision.nrtur.io) holds ideas that were never approved.
   - Never build from it.
   - Never use it to fill in gaps here. Ask instead.
3. **The API contract is not here.** It is `nrtur-backend/api/openapi.yaml`. The area docs only list endpoint paths, as pointers.
4. **Refer to screens by page id** (the `page==='…'` value in the router in `design/index.html`), not by line number. Line numbers change on every edit.
5. **Proposals:** a file in `proposals/` with `status: proposed` is under discussion and may change or be dropped. Only build `approved` (or later) proposals.
6. **Roles:** nrtur has exactly three: owner, admin and member. Any mention of more roles, record scopes or "view as role" belongs to the vision archive.
7. **Pricing and limits** come from `pricing/pricing.md` only.
8. **This repo is public.** Never commit secrets, tokens, internal hostnames, real people's contact details or customer data.
