# Coverage, performance and the limits of live tables

## What this release can and cannot guarantee

Only live corporate tables are available. The toolkit does **not** promise a
historically complete snapshot, a fixed runtime, or that DB2 will never return
SQL0905N. It prevents the combined MIN/MAX regression, reduces repeated work,
and refuses to report a planned extraction as complete when its local interval
ledger has gaps, overlaps, unfinished partitions or missing VPN days.

Three different statements must not be confused:

| Statement | Evidence / limitation |
| --- | --- |
| Every planned interval was processed | Checked locally against the exact partition union and completed VPN days |
| Every captured MTA ID interval was considered | True for completed `--mta-scope full`; intervals are queried or excluded by an actual successor lookup proving an empty numeric gap |
| Every historical event in the period was captured in one consistent database state | **Not established** on the available live tables; updates, late inserts and purges can occur during extraction |

The emitted `coverage_audit.json` and the report summary separate these claims:
`planned_extraction_accounted_for`, `global_id_domain_accounted_for`,
`database_snapshot_verified`, and `historical_completeness_verified`.
The last two remain **false**. The audit is an accounting check, not an
independent database reconciliation or a DBA-certified snapshot.

Example of an unavoidable live-table race: DB2 reports the next existing ID
as 10,000; a delayed event with ID 5,000 arrives after that query. A prior gap
observation cannot include the later event. The tests explicitly exercise this
case and require snapshot/completeness flags to remain false. A normal completed
partition has the same issue if it receives a late insert after being read.
Repeating the scan can discover changes, but two equal scans do not prove that
nothing was inserted and deleted between them. Min/max IDs and row counts alone
cannot certify unchanged content either.

IBM documents the concurrency/isolation tradeoff in its
[Db2 for z/OS concurrency guidance](https://www.ibm.com/docs/en/db2-for-zos/13.0.0?topic=users-improved-performance-through-concurrency-control).
This toolkit does not change transaction isolation, hold an extraction-wide
repeatable-read transaction, or introduce long-lived production locks. A stronger
source-level guarantee requires a separately approved stable extraction or
snapshot with known retention and a coordinated view of both source tables.
The initial historical population cannot be recovered once its rows expire.

## Exact traversal without a timestamp-order assumption

For future runs where exhaustive ID coverage matters, select `--mta-scope full`
in a **new output directory**. The default remains `estimated` so an existing
running command and its checkpoint stay compatible. An estimated run cannot
be converted into a full run merely by relabeling its results.

1. Read minimum and maximum IDs with separate read-only SELECTs. Capture the
   initial closed domain `[L,H]`. IDs inserted above H later are outside it.
2. Partition that domain into disjoint inclusive intervals. Apply the requested
   timestamp, success and node predicates inside each bounded ID query. There
   is no assumption that IDs are ordered by event timestamp.
3. Fetch `row_limit + 1` rows. A sentinel row or resource-limit error causes a
   split; that failed/capped result is never accepted as a complete partition.
4. Save the smaller ID span and use it for subsequent pending partitions and
   resumed runs. Slow network responses alone do not shrink queries: elapsed
   wall time is logged but is not a measurement of ASUTIME/CPU consumption.
5. After an empty filtered interval, optionally ask for the first actual ID
   at or after the next bound, with **no date/node/user filter**. Only IDs before
   that returned ID can be skipped as an observed empty gap. Dates outside the
   target period do not justify skipping populated ID ranges. If this optional
   probe hits a resource limit, disable the optimization and keep using bounded
   interval queries. Do not omit data or relax filters to make a failure pass.
6. Commit normalized rows, counters, the interval's done marker and its audit
   record together in local SQLite. Failed parsing, interrupted queries and
   out-of-range/duplicate IDs prevent completion. The existing checkpoint lock
   prevents two extraction processes from writing the same cache.
7. Verify that the sorted partition union is exactly `[L,H]`, with no gaps,
   overlaps or pending work, and that all requested VPN days were committed.
   The same check runs before a report accepts the cache as complete.

Conditional argument: on stable source tables, with the documented non-null
integer ID contract, every ID in `[L,H]` belongs to one completed interval.
Each interval has either an uncapped successful result or an unfiltered
successor proof that it contained no IDs. Thus no row satisfying the SQL
predicates in that domain is omitted by range estimation or result truncation.
That argument does not apply to concurrently changing tables as a common
snapshot. It also does not prove that retained rows include all historical
activity or that every JSON payload contains a usable user/IP.

The `partition_audit` table records successful query row counts/timings or gap
proofs. Earlier compatible checkpoints may have committed intervals without
these newer detailed records; their original done markers are still used.
Integrity hashes cover delivered reports, not the truth of the database.

## Performance improvements and practical limits

- Reuse completed source caches with `--cache`: subsequent supported-window
  analyses make **zero DB2 queries** and operate on the same collected rows.
- Learn smaller ID spans after actual resource/cap failures rather than retrying
  the initial oversized span in every subsequent partition.
- Skip confirmed numeric holes, not sampled timestamp regions. A synthetic
  domain from 1 to 1,000,000 with only two retained IDs needed two extraction
  queries plus one successor lookup, versus 1,000 fixed-width queries. This
  is a constructed example, not a corporate performance measurement.
- Index local pending partitions and timestamps, and avoid repeatedly counting
  every partition for progress reporting. These indexes exist only in SQLite.
- Classify VPN anchors once, then search only non-VPN candidates by timestamp.
  In a local synthetic case with 3,000 VPN-classified logins and no alerts,
  the previous correlator's median was 0.1644 s and the optimized median was
  0.0030 s over three runs (about 55x). This favorable case is not an overall
  runtime prediction. Independent brute-force checks cover 200 mixed scenarios.
- Use one DB2 query at a time. There is no added parallel load, schema change,
  index creation on DB2, or change to the bank's resource limits.

A dense retained table can still require many bounded queries. Without a
validated timestamp index, immutable block summaries, or another exact access
path, uninspected rows can contain arbitrary event dates. A sample cannot
rule that out. Full coverage has a real cost. An algorithm also cannot avoid
the time required to produce a genuinely large number of alert combinations.

[IBM's one-fetch MIN/MAX optimization](https://www.ibm.com/docs/en/db2-for-zos/13.0.0?topic=dx-one-fetch-access-accesstypei1)
requires a single aggregate. The toolkit keeps the historical separate
endpoint statements. [ASUTIME is enforced by DB2](https://www.ibm.com/docs/en/db2-for-zos/12.0.0?topic=facility-limiting-resources-sql-statements-reactively),
so even these or a single-ID query may still fail depending on the access
plan and source types. An irreducible failure stops with an incomplete cache;
it does not yield a fabricated successful report.

## Commands for subsequent runs

**Let the current run finish. Do not replace its files while it is running.**
After updating, use a new directory for an exhaustive run:

```sh
"$R5_PY" run_analysis.py compare \
  --start 2026-09-07 --end 2026-09-14 --windows 15 240 \
  --mta-scope full --env-file .env \
  --out output/contrast_sep07_14_full --resume
```

Audit a current or completed cache offline, without connecting to DB2:

```sh
"$R5_PY" run_analysis.py coverage \
  --cache output/contrast_sep07_14_full/source.sqlite
```

The command takes a short, consistent **local SQLite** read transaction. This
is not a DB2 snapshot. Exit code 0 means the audit was evaluated; inspect its
booleans and pending count to determine whether extraction is finished.

Repeat the analysis on the collected cache with no source queries:

```sh
"$R5_PY" run_analysis.py compare \
  --cache output/contrast_sep07_14_full/source.sqlite \
  --windows 15 240 --out output/reports_from_full_cache
```

Bring back `coverage_audit.json`, `mta_plan.json`, the final `run.log` lines,
and relevant sanitized `queries.jsonl` entries to evaluate actual costs. Keep
credentials and raw source data local. Query wall times help identify expensive
statements; they do not expose the optimizer's access plan or measure CPU time.
