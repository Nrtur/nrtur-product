---
title: Duplicates — instant open, background scan, saved dismiss, bulk merge
status: approved        # proposed | approved | in-progress | built | dropped
owner: Qamar Ul Islam
area: docs/areas/workspace-settings.md (Duplicates), docs/areas/contacts.md, docs/areas/companies.md, docs/areas/leads.md (duplicates mode)
screens: [settings-duplicates, contacts, companies, leads]
tickets: [SCRUM-1285 (BE phase 1), SCRUM-1287 (FE phase 1), SCRUM-1288 (BE phase 2), SCRUM-1289 (FE phase 2), SCRUM-1290 (BE phase 3), SCRUM-1291 (FE phase 3); blocked by SCRUM-1286 (merge orphans tasks) for phase 3]
created: 2026-10-15
---

# Duplicates: instant open, background scan, saved dismiss, bulk merge

## Problem
Opening **Settings › Data management › Duplicates** is slow (SCRUM-1285, found in QA). There are two causes.

**Backend.** The scan runs **inside the page request**.
- The results are cached per workspace, but the cache goes stale on **any** record change, or after 15 minutes.
- When it's stale, the next person to open the page waits for a full rescan, up to the 20s limit, then gets a 504.
- Every merge makes the cache stale again.
- On large workspaces (Pro 500k, Business 2M records per type) the scan can exceed the time limits and keep failing.
- Code: `dupescan.go`, migration 0109.

**Frontend.** It downloads **everything** when the page opens.
- It fetches all three objects at once, page after page: up to 50 × 100 = **5,000 pairs per object**, so up to **150 requests**.
- It renders every card at once.
- After each merge it reloads all of it.

**Smaller gaps on the same screen:**
- "Not a duplicate" isn't saved; the pair comes back on reload.
- There's no "last scanned" time.
- The app shows **pairs**, so 3 copies of one person appear as separate cards.
- Bulk merge is half-built and hidden.

## Proposal
**Screens** (design.nrtur.io): `settings-duplicates`, plus the in-list duplicates mode on `contacts`, `companies` and `leads`, which uses the same view. A **Duplicates** preview toggle on `settings-duplicates` switches between the scan states below.

1. **The page opens instantly.**
   - Opening it only **reads saved results**. It never waits for a scan.
   - Only the open tab loads, and only its first page.
   - The tab counts come from one cheap summary call.
2. **The scan runs in the background, automatically.**
   - A status line reads "✓ Up to date · Last scanned 12 min ago · Rescans run automatically after changes, after imports and nightly."
   - **Scan now** stays as a secondary button, to force a fresh scan.
3. **The list is paged:** 25 groups per page, highest confidence first, with "Showing 1–25 of 312 · Previous · Page 1 of 13 · Next". The confidence filter stays.
4. **Groups instead of pairs.** One card per person, e.g. "Sarah Chen · 3 records", merged in one go. The design already shows groups.
5. **"Not a duplicate" is saved** for the whole workspace.
   - A line reads "N marked not a duplicate · Show". It opens a list where each one can be **restored**.
6. **After a merge,** the group disappears immediately, the counts update, and nothing reloads.
7. **Bulk merge (owner/admin):** "Merge all near-certain (N)" merges every near-certain group into its oldest record, after a confirm.

## Rules

**Scan states** (what the screen shows):

| State | When | What the user sees |
|---|---|---|
| Up to date | Saved results match the current records | Results, plus "✓ Up to date · Last scanned …" |
| Updating | Records changed since the last scan, or Scan now was pressed | The **previous results stay**, plus "Updating in the background…" with a spinner. The list refreshes when the scan finishes. Scan now is disabled meanwhile. |
| First scan | The workspace was never scanned | "Scanning your contacts for duplicates… This runs in the background. You can leave this page; results appear here when it's done." |
| Scan failed | The last background scan failed | The previous results, plus the amber notice "Couldn't finish the last scan. These results are from …" with **Try again** |
| Top matches only | The scan hit the 250,000-pair cap | Info line: "Showing the top 250,000 matches, highest confidence first. Merge some, then rescan to see the rest." |

**When scans run.** All of these are background jobs on the existing worker, never inside a page request.
- **On open, when stale:** opening the page when the saved results are out of date queues a scan. The page doesn't wait.
- **After a CSV import** of contacts, companies or leads finishes.
- **Nightly**, for workspaces whose records changed that day.
- **Scan now.**
- **No piling up:** at most one scan per workspace and object at a time. Asking again while one is running does nothing.
- The job keeps today's in-database method (blocking plus trigram matching, migration 0109), with a longer time limit than a page request.

**Paging and order**
- 25 groups per page, ordered by confidence descending, then by age.
- The confidence filter (40% / 60% / 80% / 95%) keeps working and resets to page 1.
- The tab counts and the "possible duplicates" totals leave out dismissed groups.

**Groups**
- Records that match each other, directly or through another record, form one group. Groups are worked out when the scan runs. A group shows all its records side by side, with "Set as primary".
- A group's confidence is its strongest match. Its "Matched on" lists every signal: Email, Phone, Name.
- **Merge** merges every other record in the group into the primary, using the existing merge.

**Not a duplicate**
- Saved per workspace and per pair of records, with who and when. It applies to everyone.
- A dismissed pair never comes back, even after a rescan, unless it's restored. Restoring puts it back in the list.
- Merging or deleting either record removes its dismissals.
- Every member can dismiss and restore, the same as merging.

**After a merge**
- The merged records' pairs are removed from the saved results right away, so the list and counts stay right without a full rescan.
- Any other change is picked up by the next background scan.

**Bulk merge**
- **Owner or admin only.** Shown only when near-certain groups exist.
- **Near-certain = an exact match on a strong identifier.** This is not the confidence score: a same-name match scores 100% but is not enough on its own.
  - **Leads:** the same email, or the same phone digits.
  - **Companies:** the same phone digits, or the same normalized name **and** the same website domain.
  - **Contacts:** the same full name **and** the same company. Contacts never share an email or phone (both are unique per workspace), so a name alone would merge two different people with the same name.
  - **Name-only matches are never bulk-merged.** They stay in the list for a person to review and merge one by one.
- Each group is merged into its **oldest** record. Fields keep the oldest record's values, tags are combined, and everything linked moves over, as a normal merge does.
- A confirm says "Merges can't be undone". The toast reports how many groups were merged.

**Unchanged:**
- The plan feature `duplicate_detection`: off on Solo.
- What counts as a match, and the 250,000-pair cap.
- The merge drawer.
- Write-time duplicate checks (409 `DUPLICATE_LEAD` and so on).

## Permissions

| | Owner | Admin | Member |
|---|:-:|:-:|:-:|
| Open Duplicates, see results, Scan now | ✓ | ✓ | ✓ |
| Merge one group, Not a duplicate, Restore | ✓ | ✓ | ✓ |
| Bulk merge ("Merge all near-certain") | ✓ | ✓ | — |

Single merges stay open to every member, as today. Bulk merge is new and changes many records at once, so it needs owner or admin (`config:manage`).

## Plan and pricing impact
None. `duplicate_detection` stays on Trial, Team, Pro and Business, and off on Solo.

## Out of scope
- New matching signals (e.g. fuzzy email or address), or tuning confidence.
- Undoing a merge.
- Automatic merging without anyone clicking.
- Duplicates across object types (a lead that matches a contact). The lead-to-contact check at convert stays as it is.

## Phases
Each phase ships on its own.

1. **Fix the slow open.** This alone fixes what QA hit.
   - FE: load only the active tab and page 1, with server paging and no 50-page loop. Remove merged groups locally.
   - BE: return `scan` metadata (`computed_at`, `truncated`, `pairs_count`) with the results.
2. **Background scan and saved dismiss.**
   - BE: the worker job, auto-rescans, never computing in the request, the summary endpoint, the dismiss table and endpoints, pruning the cache after a merge.
   - FE: scan states, Scan now with polling, saved dismiss and restore.
3. **Groups and bulk merge.**
   - BE: groups at scan time and paging by group, plus the bulk-merge endpoint.
   - FE: group cards and Merge all near-certain.

## Decisions (owner, 2026-10-15)
Accepted as recommended:
1. Background scan, started automatically on open when stale, plus after imports and nightly. Scan now stays as an option.
2. Groups instead of pairs, in phase 3.
3. Bulk merge in phase 3, for owner and admin only.
4. Single merge stays open to every member.
5. SCRUM-1285 becomes the phase-1 BE ticket.

## Follow-up (owner, 2026-10-15)
- **Bulk merge "near-certain" means an exact match on a strong identifier, not 95%+ confidence.** The first version would have bulk-merged every pair of contacts with the same full name, because contacts only match on name. The rule is now: leads on email or phone; companies on phone, or name plus website domain; contacts on name plus company; never on name alone.

## Tickets (to create after approval)
- **Phase 1**
  - BE: SCRUM-1285, reworded. Add `scan` metadata to the duplicates responses.
  - FE: load only the active tab and page 1 with server paging; remove merged groups locally.
- **Phase 2**
  - BE: background scan job (on open when stale, after import, nightly, Scan now), reads never compute, `GET /duplicates/summary`, `POST /{object}/duplicates/scan`, the dismissals table with dismiss and restore endpoints, pruning pairs after a merge.
  - FE: scan states and polling, saved Not a duplicate and Restore.
- **Phase 3**
  - BE: groups (connected records) at scan time and paging by group; `POST /{object}/duplicates/bulk-merge` (owner/admin).
  - FE: group cards, Merge all near-certain (the existing hidden `useBulkMerge`).
