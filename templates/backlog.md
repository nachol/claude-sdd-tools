# Backlog

<!--
Backlog file read by the `agentic-backlog` skill (`/loop /agentic-backlog`).

Format
- Each item is a level-2 section: `## <title>`.
- The first fenced `yaml` block of the section holds the item metadata:
    id:      unique, stable identifier (used in branch names: agent/<id>, agent/<id>-O-NN). Letters, digits, dashes.
             SPLIT children get loop-assigned IDs <parent>-1, <parent>-2, ...
    title:   short title (used in commit and PR titles).
    links:   optional list of URLs or repository paths to supporting documents (read by triage and the delivery skills).
    status:  optional — `ready` (default when missing) or `hold` (the loop skips the item).
- Everything after the yaml block, up to the next `## `, is the item description. Write the problem
  and at least one observable success criterion; items without them are routed to NEEDS_INPUT.
- Order matters: the loop takes eligible items top to bottom.

This file is human-authored only. The loop never writes it. Loop state (triage verdict, method, branch,
PR, status) lives in `.agentic-backlog/state.json` (excluded via `.git/info/exclude`) and reports in
`.agentic-backlog/reports/`.
To unblock a `blocked` item, set its `status` in `state.json` to `ready` (keep its other fields) and optionally
write `guidance` (free text). The loop resumes it on its existing branch: a fix pass for code/test failures, a map
refine for refine blocks, or a re-run of the blocked spec/map/card skill with your guidance — it never re-triages it.
To resume a `needs-input` item:
- ODD item with a map (no eligible outcome): fix the map or write `guidance`, then set `status` to `ready`; the loop
  resumes it as an ODD item between outcomes (refine first).
- Triage NEEDS_INPUT (no map yet): update the item in this file, then delete its `state.json` entry; it is re-triaged.
To restart an item from scratch, delete its `agent/<id>*` branches (local and remote) and its `state.json` entry.
-->

## Rate-limit the public search API per tenant

```yaml
id: BL-001
title: Rate-limit the public search API per tenant
links:
  - docs/api/search.md
status: ready
```

Tenants on the free plan can saturate `/v1/search` and degrade latency for everyone. Add per-tenant
rate limiting to the public search endpoint.

Success criteria:
- A tenant exceeding its limit receives `429 Too Many Requests` with a `Retry-After` header.
- Tenants under their limit see no change in behavior or in p99 latency (< 5 ms added).
- Limits are configurable per plan without a deploy.

## Version the webhook payload format

```yaml
id: BL-002
title: Version the webhook payload format
links:
  - docs/webhooks.md
status: ready
```

External integrators parse our webhook payloads, and every field change breaks someone. Introduce an
explicit payload version (`v1`, `v2`) that integrators pin per subscription, with `v1` being today's format.

Success criteria:
- Existing subscriptions keep receiving the exact current payload (`v1`), byte for byte.
- A subscription pinned to `v2` receives the new format, documented in `docs/webhooks.md`.
- The version is validated on subscription create/update; unknown versions are rejected with `400`.

## Export the reports table as CSV

```yaml
id: BL-003
title: Export the reports table as CSV
status: hold
```

Users copy the reports table into spreadsheets by hand. Add a "Download CSV" action that exports the
rows currently visible (respecting active filters).

Success criteria:
- The downloaded file contains exactly the filtered rows and the visible columns, with a header row.
- Exports over 50,000 rows are refused with a clear message instead of timing out.
