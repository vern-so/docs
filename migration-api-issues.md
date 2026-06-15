# Migration API — foreseeable integrator issues

A prioritized memo of friction points and gaps in the Migration API docs, written
from the perspective of a developer integrating Vern as a headless migration
engine.

> **Updated 2026-06-15 for the Managed-Agent pivot.** The API is now a managed
> agent (reuse-or-author at run time, self-heal, `ask_user`), not a frozen-recipe
> replay. That retired a batch of earlier issues — the `needs_attention` dead
> end, the `proposed_patch` blast radius, the `promote` global side effect, and
> the version-drift ambiguity are all moot under the new model and have been
> pruned. The error-body shape, ID-format inconsistency, rate-limit numbers,
> one-active-run rule, and `email`/`created_by` friction are now documented and
> have been pruned too. Items below are the ones still open.

Most items are framed as **"document this"** — the API behavior is assumed
largely decided and the docs just need to catch up. Items that can't be resolved
by writing alone are tagged **[DESIGN]**.

Severity: **P0** = blocks or breaks the happy path · **P1** = correctness/safety
or major confusion · **P2** = polish, consistency, completeness.

---

## P0 — Blocks the happy path

### 1. The signed-upload flow has no dedicated page
- **Where:** `import-files.mdx` (signed-upload section), `quickstart.mdx`
  (`"upload_ref": "<from-signed-upload>"`)
- **Problem:** the handshake is documented inline inside `import-files.mdx`, but
  there's no standalone `POST /v1/companies/{id}/uploads` reference page. It's the
  first thing a developer does with real files and it's easy to miss buried in the
  import page.
- **Document:** consider a dedicated `upload-files.mdx`; spell out `upload_ref`
  lifetime/single-use, one ref per file vs. batch, and max file size.
- **Type:** doc fix.

### 2. The inline-`rows` vs. signed-upload threshold is undefined
- **Where:** `import-files.mdx`, `quickstart.mdx`
- **Problem:** "small tabular sets can be sent inline as `rows`" — "small" is
  unquantified. Developers will build against `rows`, then hit an unknown wall.
- **Document:** the row/byte cutoff at which `rows` is rejected and signed upload
  is required.
- **Type:** doc fix.

---

## P1 — Correctness, safety, major confusion

### 3. Destination duplicate-writes hinge on an invisible prerequisite (v2)
- **Where:** `export-to-destination.mdx`, `errors.mdx`
- **Problem:** `idempotency_key` doesn't dedupe writes — a "natural key" in the
  writer does. But there's no API to check whether a writer *has* one. A developer
  finds out by double-writing into a customer's live system.
- **Document:** how to confirm a writer has a natural key (UI affordance or API
  field); recommend always running `canary` first. Lands with the v2 writer
  export.
- **Type:** doc fix + **[DESIGN]** (v2).

### 4. No `blocked`-run expiry / max run duration is loose
- **Where:** `poll-an-import.mdx` (~1 hour, stale → `failed`)
- **Problem:** run timeout is given as "~1 hour" but a `blocked` run waits on a
  *human* answer via `messages`. Does a `blocked` run expire if the end-customer
  never answers? Unbounded HITL waits with no stated ceiling.
- **Document:** the exact max run duration, the stale threshold, and whether/when
  a `blocked` run times out while awaiting an answer.
- **Type:** doc fix + **[DESIGN]**.

---

## P2 — Consistency, completeness, polish

### 5. [DESIGN] Re-migrating the same customer is undefined
- **Where:** `provision-a-company.mdx` (`external_id` unique → 409)
- **Problem:** `external_id` uniqueness means you can't re-provision the same
  customer. The reuse/pinning model now answers *version* drift, but the
  re-migration story (start a fresh migration for a customer you've already
  provisioned) isn't documented — reuse the company and re-import? new company?
- **Document:** the re-migration path given the `external_id` constraint.
- **Type:** **[DESIGN]**.

### 6. [DESIGN] One account key addresses every company; magic-link auth undescribed
- **Where:** `provision-a-company.mdx`, `authentication.mdx`
- **Problem:** no per-customer key scoping — a leaked key exposes all customers'
  data. `magic_link_url` is returned on every provision with no described auth
  model (who can use it, is it an ingress-abuse vector).
- **Document / [DESIGN]:** scoped/restricted keys (or state the single-key model
  explicitly as a security note); the magic link's auth/expiry model.

### Smaller notes
- `template_ids` omitted silently provisions *every* template
  (`provision-a-company.mdx`) — call out the default loudly.
- The `webhooks.mdx` page is now hidden from the Migration API nav (its
  workbook-export webhook model predates the API and the spec says delivery is
  polling, no webhooks yet). The page file still exists and is still linked from
  `authentication.mdx` — decide whether to keep it reachable, fold it into the
  help center, or remove it.
- No documented way to add/remove sheets after provisioning, or (v2) to retry
  *only* the failed rows of a partial destination export.
- (v2) `canary` exports "one row per template" (`export-to-destination.mdx`) —
  document which row and whether it's deterministic, so a passing canary is
  meaningful.
