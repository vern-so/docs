# Migration API — foreseeable integrator issues

A prioritized memo of friction points and gaps in the Migration API docs, written
from the perspective of a developer integrating Vern as a headless migration
engine. Based on a full read of the 14 `migration-api/*.mdx` pages (plus the
older `webhooks.mdx`) as of 2026-06-11.

Most items are framed as **"document this"** — the API behavior is assumed
largely decided and the docs just need to catch up. Items that can't be resolved
by writing alone are tagged **[DESIGN]**. Two items are tagged **[SAFETY]**
because the decision UI is rendered to the *end-customer*, which widens the blast
radius.

Severity: **P0** = blocks or breaks the happy path · **P1** = correctness/safety
or major confusion · **P2** = polish, consistency, completeness.

---

## P0 — Blocks the happy path

### 1. The signed-upload flow has no page
- **Where:** `import-files.mdx:30`, `quickstart.mdx:77` (`"upload_ref": "<from-signed-upload>"`)
- **Problem:** "Request a signed upload URL, `PUT` the file to it, then pass the
  returned reference as `upload_ref`" — but no endpoint documents how to request
  that URL. This is the first thing a developer does with real files and it's a
  black hole.
- **Document:** the request/response for obtaining a signed URL; the `PUT`
  contract (method, content-type, required headers); `upload_ref` lifetime and
  whether it's single-use; one ref per file vs. batch; max file size.
- **Type:** doc fix (new page, e.g. `upload-files.mdx`).

### 2. `needs_attention` is a dead end today
- **Where:** imports are "Preview" (`import-files.mdx`), but the `decisions`
  endpoint is "**Planned**" (`resolve-flagged-rows.mdx:7`).
- **Problem:** a Preview import can park at `needs_attention`, but there is no
  shipped API to resume it. A developer following the quickstart can reach a
  state they can't exit programmatically.
- **Document:** a clear "what works today vs. later" availability matrix, and —
  if `decisions` truly isn't shipped — what a developer should do *today* when an
  import parks (reject? fix source + re-import? dashboard?).
- **Type:** doc fix + **[DESIGN]** if the resolution path for Preview users isn't settled.

### 3. The inline-`rows` vs. signed-upload threshold is undefined
- **Where:** `import-files.mdx:44`, `quickstart.mdx:71`
- **Problem:** "small tabular sets can be sent inline as `rows`" — "small" is
  unquantified. Developers will build against `rows`, then hit an unknown wall.
- **Document:** the row/byte cutoff at which `rows` is rejected and signed upload
  is required.
- **Type:** doc fix.

---

## P1 — Correctness, safety, major confusion

### 4. "cells" vs "rows" ambiguity → silent row loss
- **Where:** `poll-an-import.mdx:38-53`, `export.mdx:35`
- **Problem:** report shows `inserted: 1450, invalidCellCount: 12`. `filter=valid`
  (the default) "returns only rows that passed validation." If those 12 are
  *cells*, the default CSV silently drops the whole *rows* containing them — fewer
  rows than `inserted` implies, with no signal. A row lost over one bad cell is a
  big deal.
- **Document:** define `inserted` precisely (total vs. valid), state explicitly
  whether `filter=valid` excludes whole rows that contain any invalid cell, and
  cross-reference the count fields so the math is unambiguous.
- **Type:** doc fix.

### 5. [SAFETY] `proposed_patch` can delete data the customer was never asked about
- **Where:** `resolve-flagged-rows.mdx:63-71`
- **Problem:** the patch summary is "Map 12 Status values… **drop the rows
  missing Email**," but there's only one question (fq0, Status). Approving with no
  `answers` (the "Apply suggested fix" button the doc recommends) silently applies
  a `delete_invalid_rows` on Email — data loss the end-customer's rendered UI
  never surfaced as a choice.
- **Document / [DESIGN]:** clarify the mapping between `questions` and
  `proposed_patch.actions`; ensure every destructive action is representable as a
  question; warn integrators not to wire "Apply suggested fix" to a one-click
  flow without showing the full action list. Since the **end-customer** sees this,
  recommend rendering `actions[]` (especially `delete_invalid_rows`) explicitly
  before approve.
- **Type:** doc fix + design clarification.

### 6. [SAFETY] `promote: true` is a global side effect triggered from one run
- **Where:** `resolve-flagged-rows.mdx:134`
- **Problem:** one customer's fix can be promoted to the source's default logic
  for *all* future customers. Because the decision UI is rendered to the
  **end-customer**, an end-customer's one-off answer could reshape every other
  customer's migration.
- **Document / [DESIGN]:** loud warning on blast radius; strong guidance that
  `promote` should be an integrator-staff action, never exposed to the
  end-customer; consider whether `promote` should require a separate
  capability/permission rather than a request flag.
- **Type:** doc fix + design guidance.

### 7. Destination duplicate-writes hinge on an invisible prerequisite
- **Where:** `export-to-destination.mdx:77-82`, `errors.mdx:39-44`
- **Problem:** `idempotency_key` doesn't dedupe writes — a "natural key" in the
  writer does. But there's no API to check whether a writer *has* one. A developer
  finds out by double-writing into a customer's live system. Also `hollowWrites`
  appears in the report (`export-to-destination.mdx:68`) and is never defined.
- **Document:** define `hollowWrites`; document how to confirm a writer has a
  natural key (UI affordance or API field); recommend always running `canary`
  first.
- **Type:** doc fix.

### 8. Error response *body* shape is never shown
- **Where:** every "Errors" table; `errors.mdx` is the natural home.
- **Problem:** status codes are listed, but no page shows the JSON body — no
  `code` / `message` / `field`, no request/correlation ID for support. You can't
  program error handling against status codes alone.
- **Document:** the canonical error body schema and an example; a request ID
  header (e.g. `x-request-id`) for support escalation.
- **Type:** doc fix.

### 9. Rate limits are unquantified
- **Where:** every page ("429 — back off and retry"); `errors.mdx:48-50`,
  `poll-an-import.mdx:17`.
- **Problem:** no numeric limit, no `Retry-After` header, no reconciliation of
  "poll every ≈2s" against the limit. Polling many concurrent migrations at 2s
  could self-inflict 429s.
- **Document:** the actual limit(s), whether `Retry-After` is sent, and a
  recommended poll cadence/backoff that stays under it.
- **Type:** doc fix.

### 10. No run timeout / `needs_attention` expiry / stale threshold
- **Where:** `poll-an-import.mdx:126-130`
- **Problem:** "a run… is treated as `failed`… once it goes stale" — no number.
  How long can an import run? Does `needs_attention` expire if the end-customer
  never answers (likely, since a human is in the loop)? Unbounded waits with no
  ceiling.
- **Document:** max run duration, the stale threshold, and `needs_attention`
  expiry behavior.
- **Type:** doc fix.

---

## P2 — Consistency, completeness, polish

### 11. The webhooks page is from the old model and doesn't connect
- **Where:** `webhooks.mdx` (pre-restructure, dated May 13)
- **Problem:** describes a third delivery channel
  (`workbook.production.export` / `preview.export`, flat/nested payloads,
  batching) that appears nowhere in the intro flowchart, `concepts.mdx`, or the
  two export pages. Reads as UI-dialog language grafted onto an API. Unclear how a
  headless caller *triggers* a webhook export or what makes one "production" vs
  "preview."
- **Document / [DESIGN]:** reconcile webhooks into the delivery model — is it a
  third option alongside CSV download and destination push? How is it triggered
  via the API? Add it (or its absence) to the intro flowchart and `concepts.mdx`.
- **Type:** doc fix + design reconciliation.

### 12. ID formats are inconsistent across pages
- **Where:** `cmp_123` / `wb_123` / `sht_1` / `tmpl_contacts` (provision,
  quickstart) vs. UUIDs for `source` and template IDs (discovery pages; provision
  *request* mixes a UUID `source` with a `tmpl_contacts`-style `template_ids`) vs.
  `wb_abc123` / `sheet_001` (webhooks).
- **Problem:** a reader can't tell what format to expect and store.
- **Document:** make IDs consistent across examples, or state plainly they're
  illustrative and give the real format per resource.
- **Type:** doc fix.

### 13. [DESIGN] Re-migration and version drift are undefined
- **Where:** `provision-a-company.mdx:91` (external_id unique → 409),
  `concepts.mdx:70-72` ("new runs pick up the new version")
- **Problem:** a company is bound to a *source*, not a pinned *version*. (a) How
  do you re-migrate the same customer later given the `external_id` uniqueness
  constraint? (b) A company's second import months later can silently behave
  differently from its first, because new runs adopt the latest published version.
- **Document / [DESIGN]:** the re-migration story; whether a company pins the
  version at creation or always uses latest; how to opt into pinning if needed.

### 14. [DESIGN] One account key addresses every company; magic-link auth undescribed
- **Where:** `provision-a-company.mdx:18`, `:77` (`magic_link_url`)
- **Problem:** no per-customer key scoping — a leaked key exposes all customers'
  data. `magic_link_url` is returned on every provision with no described auth
  model (who can use it, is it an ingress-abuse vector).
- **Document / [DESIGN]:** scoped/restricted keys (or state the single-key model
  explicitly as a security note); the magic link's auth/expiry model.

### Smaller notes
- `email` must be an existing Vern teammate as `created_by` (`provision-a-company.mdx:33`)
  — odd friction for a "customers never see Vern" flow; document why, and whether
  a service-account pattern is supported.
- `template_ids` omitted silently provisions *every* template
  (`provision-a-company.mdx:47`) — call out the default loudly.
- No documented way to add/remove sheets after provisioning, or to retry *only*
  the failed rows of a partial destination export (`export-to-destination.mdx`
  report has `failures[]` but no "export failures only" path; `selection.row_filter`
  is too vague to rely on).
- One-active-run model is implied (`poll-an-import.mdx:116`) but never stated —
  document whether concurrent imports per company are allowed and what happens if
  you POST a second.
- `canary` exports "one row per template" (`export-to-destination.mdx:30`) —
  document which row and whether it's deterministic, so a passing canary is
  meaningful.
