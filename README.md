# Latest state of retried jobs with OpenSearch

Reconstruct the latest recorded state of 12 example report-export jobs from 32 scheduler snapshots, then add one delayed completion. Compare three questions:

- Which jobs have failure records?
- Which jobs are latest failed when ordered by producer sequence?
- What goes wrong when receipt time determines the latest event?

Compare your responses with the fixture-derived results in [expected-results.md](expected-results.md).

## Prerequisites

Use an existing, authenticated **OpenSearch 3.9.0** test cluster and its configured **Dashboards Dev Tools** console. Your account must be allowed to read the version, check index existence, create an index, bulk-create documents, count them, and search. This package supplies no environment setup.

Use an unused, authorized index name. Request 02 must return **404** for `retry-job-events`; an authorization error does not prove absence. If the index exists, stop. To choose another unused name, replace `retry-job-events` in request paths only. Bulk actions contain no index names. Keep other writers, transformations, backfills, and retention policies away from the lab index. Resolve incompatible prerequisites before loading data.

## Files

| File | Purpose |
| :--- | :--- |
| [requests.http](requests.http) | 15 numbered Dev Tools templates: mapping, loads, counts, searches, and readback |
| [data/initial.ndjson](data/initial.ndjson) | Exact request 04 body: 32 action/source pairs |
| [data/initial-reversed.ndjson](data/initial-reversed.ndjson) | The same pairs in reverse order |
| [data/delayed.ndjson](data/delayed.ndjson) | Exact request 11 body: delayed completion J07-02 |
| [expected-results.md](expected-results.md) | Expected selected events, bucket counts, totals, and full fixture inventory |

## Load and run

Open `requests.http` as text. Copy one request at a time into Dev Tools and inspect its response. It uses relative paths without headers. Do not execute the whole file: continuations contain a placeholder that must be replaced before sending.

Follow this table in order. Each traversal runs its first-page template once, then reuses its continuation template through exhaustion as described below. Template 07 is intentionally run twice.

| Stage | Run | Expected check |
| :--- | :--- | :--- |
| Setup | 01, 02, 03, 04, 05 | Version `3.9.0`; absence **404**; creation **200** with `acknowledged: true`; 32 creates; count **32** |
| Wrong A | 06 | Failure-filtered comparison; Freeze writes before searching |
| Sequence C0 before delayed load | 07 then reuse 08 | Complete all 12 jobs; 8 succeeded, 2 failed, 2 running |
| Receipt C0 before delayed load | 09 then reuse 10 | Complete all 12 jobs; 7 succeeded, 3 failed, 2 running |
| Delayed load | 11, 12 | Only after BOTH initial traversals finish; one create J07-02; count **33**; Freeze writes again |
| Sequence C0 after delayed load | 07 then reuse 08 | Restart unchanged C0 with empty tallies and no starting `after`; same selected sources as before load |
| Sequence C1 after delayed load | 13 then reuse 14 | Start a fresh traversal; 9 succeeded, 2 failed, 1 running |
| Full readback | 15 | All 33 complete sources and IDs equal the combined fixture in job/sequence/event-ID order |

Run 04 once. Its embedded body equals `data/initial.ndjson`. Preserve action/source pairs and the final newline. Require `errors: false`, exactly 32 `create` items, every status **201**, and exact fixture IDs. The explicit mapping uses `dynamic: strict`. For 11, require `errors: false` and exactly one **201** create for J07-02. Stop if either count check fails.

For every search, require `timed_out: false`, zero failed shards, and exact total hits (`relation: "eq"`). A successful HTTP status or total count alone does not prove correct contents. Stop on any rejected mapping, partial bulk success, conflict, missing source, unexpected count, timeout, or failed shard. Do not overwrite existing records to repair a run.

## Page with the actual response cursor

Start each traversal without `composite.after` and with empty tallies. For each bucket under `aggregations.jobs.buckets`, record `key.job_id` and `latest.hits.hits[0]._source`. Count the selected state once per job. Reject duplicate job keys.

An empty bucket array ends traversal. After a nonempty page, copy the entire returned `aggregations.jobs.after_key` into the next request's `composite.after`. If no cursor is returned, traversal is complete. **Never construct a cursor from the last bucket.**

Templates 08, 10, and 14 contain the unquoted token `{{AFTER_KEY}}`. It is a placeholder, not valid request JSON. Replace that entire token with the complete JSON object from the immediately preceding response's `aggregations.jobs.after_key`. Do not add quotes or another pair of braces. On each reuse, replace the entire previous `composite.after` value with the newly returned object. Never carry a cursor from another traversal.

Send continuation requests only while a cursor exists, including after the last populated page if it supplies one. Reuse the same continuation template for every subsequent page, including a possible terminal empty page. Reject repeated or nonadvancing cursors. At exhaustion, require all 12 jobs and compare every projected field with the fixture. Keep filters and sort unchanged during a traversal. Restart the C0-after traversal from pristine template 07, never from an edited continuation.

## Expected comparison

| Query | Succeeded | Failed | Running |
| :--- | ---: | ---: | ---: |
| Sequence order, C0, before or after delayed load | 8 | 2 | 2 |
| Receipt order, C0 | 7 | 3 | 2 |
| Sequence order, C1, after delayed load | 9 | 2 | 1 |

Failure-only request 06 matches **7 records across 6 jobs**. Correct reconstruction selects **J05 and J08** as failed; receipt order incorrectly adds **J02**. Page one has zero latest failures, so partial-page totals mislead. J03's tied event timestamps require producer sequence. J07 changes to succeeded only at C1 after its delayed completion is loaded.

Every query retains batch `exports-20261010` and event horizon **10 October 2026, 10:05 GST**. C0 is **10:10 GST**; C1 is **10:15 GST**. Requests use explicit `+04:00` timestamps.

## Repeat and limits

Repeat on a second fresh index with all index paths changed consistently. For request 04, replace the entire body with `data/initial-reversed.ndjson`; do not load both initial bodies. Repeat the remaining requests and response-driven traversals. Selected sources, bucket counts, and totals should match the first index.

These are immutable scheduler snapshots. A trustworthy, unique, increasing producer-assigned `event_seq` orders each job across attempts; it is unrelated to OpenSearch `_seq_no`. The secondary event-ID sort cannot repair conflicting sequences. Strict mapping does not enforce required fields, allowed states, or unique sequences.

Receipt cutoffs describe the fixture's available evidence, not historical search visibility. Missing or expired events, bad clocks, conflicting writers, and concurrent backfills limit reconstruction. Freeze writes while paging; a timestamp cutoff alone cannot freeze a dataset. Latest recorded failure does not establish retry exhaustion, report delivery, or business correctness. This small fixture checks correctness, not performance. Grouping and paging still process relevant histories.
