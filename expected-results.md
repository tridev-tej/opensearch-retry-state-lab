# Expected results

These expectations follow from the supplied synthetic fixture. Compare your responses with the checks below.

## Setup and load expectations

| Request | Expected check |
| :--- | :--- |
| 01 | Root request succeeds and `version.number` is `3.9.0`; retain actual build metadata |
| 02 | Index existence check returns 404, not an authorization error |
| 03 | Index creation succeeds with `acknowledged: true` and the supplied mapping |
| 04 | `errors: false`; 32 ordered `create` items, each status 201 and `_id` equal to its fixture `event_id` |
| 05 | `count: 32`, zero failed shards |
| 15 | `errors: false`; one `create`, status 201, `_id: J07-02` |
| 16 | `count: 33`, zero failed shards |

For every search, require `timed_out: false` and `_shards.failed: 0`. With `track_total_hits: true`, require exact `hits.total.relation: "eq"`. The aggregation searches in requests 06-24 have outer `size: 0`, so their ordinary `hits.hits` array is empty. The states used below come from aggregation sub-hits, not the outer array. Shard counts and timing values depend on deployment defaults.

## Wrong A: failure-filtered grouping

Request 06 is expected to return `hits.total: {"value":7,"relation":"eq"}` and these six buckets, in ascending job order:

| Job | `doc_count` | Matching failure event IDs |
| :--- | ---: | :--- |
| J01 | 1 | J01-02 |
| J02 | 1 | J02-02 |
| J03 | 1 | J03-02 |
| J05 | 2 | J05-02, J05-04 |
| J06 | 1 | J06-02 |
| J08 | 1 | J08-02 |

This is a count of jobs with failure records. Calling all six jobs currently failed is the deliberately incorrect interpretation. Composite size 12 exceeds this fixture's six matching groups. The query has removed successful and running snapshots before grouping.

## Correct reconstruction: complete selected events

Requests 07-10 use application sequence at cutoff C0 before the delayed load. Requests 17-20 repeat C0 after that load. Both are expected to have matching document total 32 and exactly the same selected sources. Requests 21-24 use C1 after the load and match 33 documents. The table covers all populated pages. Each selected hit's `_id` equals its `event_id`.

| Page | Job | C0 bucket `doc_count` | C0 selected event | Attempt | Sequence | State | C1 change |
| ---: | :--- | ---: | :--- | ---: | ---: | :--- | :--- |
| 1 | J01 | 4 | J01-04 | 2 | 4 | succeeded | None |
| 1 | J02 | 4 | J02-04 | 2 | 4 | succeeded | None |
| 1 | J03 | 4 | J03-04 | 2 | 4 | succeeded | None |
| 1 | J04 | 2 | J04-02 | 1 | 2 | succeeded | None |
| 2 | J05 | 4 | J05-04 | 2 | 4 | failed | None |
| 2 | J06 | 3 | J06-03 | 2 | 3 | running | None |
| 2 | J07 | 1 | J07-01 | 1 | 1 | running | doc_count 2; J07-02; attempt 1; sequence 2; succeeded |
| 2 | J08 | 2 | J08-02 | 1 | 2 | failed | None |
| 3 | J09 | 2 | J09-02 | 1 | 2 | succeeded | None |
| 3 | J10 | 2 | J10-02 | 1 | 2 | succeeded | None |
| 3 | J11 | 2 | J11-02 | 1 | 2 | succeeded | None |
| 3 | J12 | 2 | J12-02 | 1 | 2 | succeeded | None |

For source comparison, `latest.hits.hits[0]._source` must equal the corresponding document in `data/initial.ndjson` (or J07-02 in `data/delayed.ndjson`) projected onto the request's seven included fields: `job_id`, `event_id`, `state`, `attempt`, `event_seq`, `event_time`, and `ingested_at`. The omitted `batch_id` is the constant `exports-20261010`. Compare full timestamp values, not just the state. For numeric sort values, the primary selected `event_seq` must match the table and the secondary value must be the selected `event_id`.

Expected selected-state totals:

| Traversal | Succeeded | Failed | Running | Jobs |
| :--- | ---: | ---: | ---: | ---: |
| Correct C0 before delayed load, 07-10 | 8 | 2 | 2 | 12 |
| Correct C0 after delayed load, 17-20 | 8 | 2 | 2 | 12 |
| Correct C1 after delayed load, 21-24 | 9 | 2 | 1 | 12 |

J05 and J08 are the only latest failed jobs. J06 remains running. At C0, J07 remains running even after J07-02 has been physically loaded because its receipt timestamp exceeds C0.

## Wrong B: receipt ordering

Requests 11-14 keep all 32 matching snapshots but sort sub-hits by `ingested_at desc, event_id asc`. Their bucket counts and job pages match 07-10. Every selected event is identical to the C0 table except **J02 selects J02-02, attempt 1, sequence 2, state failed**, with receipt timestamp `2026-10-10T10:08:00.000+04:00`. The expected totals are seven succeeded, three failed, and two running. The incorrect failed set is J02, J05, J08.

## Pagination and exhaustion

| Page | Expected bucket keys | Prepared `after` in next request, conditional on actual response |
| ---: | :--- | :--- |
| 1 | J01, J02, J03, J04 | `{"job_id":"J04"}` |
| 2 | J05, J06, J07, J08 | `{"job_id":"J08"}` |
| 3 | J09, J10, J11, J12 | `{"job_id":"J12"}` |
| 4 | Empty `buckets`, when reached using a returned final cursor | Stop |

These are expected bucket pages and example cursor values, not captured `after_key` objects. Always take the continuation object from the actual response. The prepared fourth requests (10, 14, 20, 24) are conditional: execute only if the preceding nonempty page supplies a cursor, using its returned value. If the cursor is absent, the traversal has already ended. Do not require a particular empty-page cursor shape.

At page one, the correct traversal has four succeeded and zero failed jobs. After page two, it has four succeeded, two failed, and two running. Page three adds four successes. `hits.total` remains the matching snapshot total on each page; it is not a per-page or distinct-job count.

## Full fixture inventory

This is the literal initial fixture projected into a compact table. All dates are **10 October 2026 GST**. Times below expand to `2026-10-10T<time>.000+04:00`. The batch is always `exports-20261010`, `_id` equals `event_id`, and the job is the prefix before the hyphen. NDJSON remains the complete source of bytes.

| Initial load position | Event ID | Attempt | Sequence | State | Event time | Receipt time |
| ---: | :--- | ---: | ---: | :--- | :--- | :--- |
| 1 | J01-01 | 1 | 1 | running | 10:00:00 | 10:00:01 |
| 2 | J01-02 | 1 | 2 | failed | 10:01:00 | 10:01:01 |
| 3 | J01-03 | 2 | 3 | running | 10:02:00 | 10:02:01 |
| 4 | J01-04 | 2 | 4 | succeeded | 10:03:00 | 10:03:01 |
| 5 | J02-01 | 1 | 1 | running | 10:00:00 | 10:00:01 |
| 6 | J02-03 | 2 | 3 | running | 10:02:00 | 10:02:01 |
| 7 | J02-04 | 2 | 4 | succeeded | 10:03:00 | 10:03:01 |
| 8 | J02-02 | 1 | 2 | failed | 10:01:00 | 10:08:00 |
| 9 | J03-01 | 1 | 1 | running | 10:00:00 | 10:00:01 |
| 10 | J03-02 | 1 | 2 | failed | 10:01:00 | 10:01:01 |
| 11 | J03-04 | 2 | 4 | succeeded | 10:03:00 | 10:03:02 |
| 12 | J03-03 | 2 | 3 | running | 10:03:00 | 10:03:01 |
| 13 | J04-01 | 1 | 1 | running | 10:00:00 | 10:00:01 |
| 14 | J04-02 | 1 | 2 | succeeded | 10:02:00 | 10:02:01 |
| 15 | J05-01 | 1 | 1 | running | 10:00:00 | 10:00:01 |
| 16 | J05-02 | 1 | 2 | failed | 10:01:00 | 10:01:01 |
| 17 | J05-03 | 2 | 3 | running | 10:02:00 | 10:02:01 |
| 18 | J05-04 | 2 | 4 | failed | 10:04:00 | 10:04:01 |
| 19 | J06-01 | 1 | 1 | running | 10:00:00 | 10:00:01 |
| 20 | J06-02 | 1 | 2 | failed | 10:01:00 | 10:01:01 |
| 21 | J06-03 | 2 | 3 | running | 10:04:00 | 10:04:01 |
| 22 | J07-01 | 1 | 1 | running | 10:00:00 | 10:00:01 |
| 23 | J08-01 | 1 | 1 | running | 10:00:00 | 10:00:01 |
| 24 | J08-02 | 1 | 2 | failed | 10:04:00 | 10:04:01 |
| 25 | J09-01 | 1 | 1 | running | 10:00:00 | 10:00:01 |
| 26 | J09-02 | 1 | 2 | succeeded | 10:02:00 | 10:02:01 |
| 27 | J10-01 | 1 | 1 | running | 10:00:00 | 10:00:01 |
| 28 | J10-02 | 1 | 2 | succeeded | 10:02:00 | 10:02:01 |
| 29 | J11-01 | 1 | 1 | running | 10:00:00 | 10:00:01 |
| 30 | J11-02 | 1 | 2 | succeeded | 10:02:00 | 10:02:01 |
| 31 | J12-01 | 1 | 1 | running | 10:00:00 | 10:00:01 |
| 32 | J12-02 | 1 | 2 | succeeded | 10:02:00 | 10:02:01 |

The single delayed source is J07-02: attempt 1, sequence 2, state `succeeded`, event time `2026-10-10T10:04:00.000+04:00`, receipt time `2026-10-10T10:12:00.000+04:00`.

Request 25 must return 33 ordinary hits with `hits.total: {"value":33,"relation":"eq"}`. Their `_source` objects must equal the 32 initial documents plus the delayed document, sorted by `job_id asc, event_seq asc, event_id asc`. Compare every field, including timestamps and `batch_id`. J02-02 and J03-03 intentionally occur out of sequence in the original bulk body; stored insertion order must not become application order.

## Ordering checks and failure expectations

- J02: receipt sorting selects J02-02; sequence sorting selects J02-04.
- J03: J03-03 and J03-04 both have event time 10:03:00 GST. Sequence 4 must win. `event_id` is only a deterministic secondary sort; it cannot make conflicting equal sequences valid.
- Reversed input: `data/initial-reversed.ndjson` contains the exact same action/source pairs in reverse order. On a separate fresh index, every selected source, bucket count, and total above is expected to stay unchanged.
- Partial bulk success, unexpected counts or sources, duplicate job keys, nonadvancing cursors, timeouts, or failed shards invalidate the corresponding result. A success HTTP status alone is insufficient. A full-fixture count alone cannot prove which sources or latest events were retained.

Latest indexed scheduler state is not proof of report correctness, delivery, retry exhaustion, or any external side effect.
