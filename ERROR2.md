# Resume after the error2 screenshot

## What failed

The screenshot shows 36 committed MTA partitions, followed by a JSON parsing
exception in the next partition. The old parser aborted the entire extraction
on a single unusable payload. Previously committed VPN days and MTA partitions
remain in `source.sqlite`; the failing partition was rolled back.

The screenshot alone cannot distinguish malformed source data from a driver
transfer/type problem. This update does not attempt to reconstruct JSON or
infer missing usernames/IPs.

## Update and resume

Run the following from the `r5-kit` repository root after the old process has
exited, as it already had in the screenshot. Use your existing corporate Python
interpreter in `R5_PY`. Run each command only if the previous one succeeds.

```sh
git pull --ff-only
sha256sum -c SHA256SUMS
umask 077
age --decrypt --output r5-error2-update.zip r5-toolkit.zip.age
python3 -m zipfile -e r5-error2-update.zip .
cd analise_agencia_toolkit
"$R5_PY" run_analysis.py compare \
  --start 2026-09-07 --end 2026-09-14 --windows 15 240 \
  --env-file .env --out output/contrast_sep07_14 --resume
```

Enter the previously supplied passphrase when age prompts. If the installation
is under `work/analise_agencia_toolkit`, extract into `work` instead of `.` and
change directory there. Use the **same output directory and extraction options
as your interrupted run**. If it used `--mta-scope full`, retain that option;
do not change scopes, dates, filters or configuration during resume.

The archive contains no `.env`, credentials or corporate output, so updating
its files preserves those. SQL, configuration defaults and cache signatures
are unchanged. An additive **local SQLite** table is created for diagnostics;
no corporate table is created or modified. Completed partitions are reused,
and only the rolled-back/pending work is fetched again.

PDF generation is automatic if `pdflatex` is available. Without it, both TeX
reports are generated. Add `--compile-pdf` only when you want missing TeX
software to be treated as an error.

## New behavior and interpretation

- Valid rows in a partition continue to be analyzed. Invalid JSON/type values
  and rows without usable usernames/IPs get a local diagnostic record.
- Diagnostics, counters, normalized rows and the completed checkpoint commit
  in the same transaction. A failure rolls that transaction back.
- There is no per-reject retry, extra DB2 round trip, regex repair or inferred
  field. Database/query failures and invalid ID contracts still stop the run.
- `coverage` and `coverage_audit.json` include `payload_quality`: malformed JSON
  count, missing-field count, unusable total and diagnostic count.
- `summary.json`, `detailed.tex`/PDF and `relatorio.tex`/PDF explicitly disclose
  unusable rows and the resulting coverage limitation. Such rows may hide
  alerts or affect classification; the remaining counts are not certified as
  complete. Finishing all partitions does not prove all rows were analyzable.
- `rejected_mta.jsonl` in the final report contains ID, timestamp, reason,
  Python value type, byte length and SHA-256 where available. It contains no
  raw JSON. The same metadata lives in the local `rejected_mta` SQLite table.
  Treat these identifiers as corporate information despite the omitted payload.
- Previous versions counted missing fields without storing their IDs. These
  old counts remain explicit as `legacy_rows_without_diagnostics`; resuming
  does not reread completed partitions merely to reconstruct those diagnostics.
- Source tables are live. Neither this fix nor full ID coverage guarantees a
  consistent historical snapshot or recovery of purged/concurrently changed rows.

For an offline progress check:

```sh
"$R5_PY" run_analysis.py coverage \
  --cache output/contrast_sep07_14/source.sqlite
```

After successful completion, the command prints a `report_TIMESTAMP` directory
containing the two reports and diagnostics. To rebuild presentation only:

```sh
"$R5_PY" run_analysis.py report \
  --directory output/contrast_sep07_14/report_ACTUAL_TIMESTAMP --compile-pdf
```

## What to return if it still fails

Return the exact command (without secrets), final error and relevant `run.log`
lines. Include `coverage` output. For another database query error include the
matching `queries.jsonl` entry; for PDF failure include `report_status.json`
and the end of the failing `.build.log`. Review all files for corporate data
before sharing. For rejected payloads, start with aggregate `payload_quality`
counts and value types/lengths; inspect the identified source rows locally.
Do not publish the cache, raw JSON, credentials or complete diagnostic IDs.

## Validation scope

Local regression tests exercise malformed/truncated JSON, invalid encoding,
unexpected transport types, missing fields, transactional rollback, legacy
cache upgrade, resume without double counting, both report warnings and
manifest verification. These tests use synthetic adapters, not corporate DB2.
The actual source value, transport behavior and resumed corporate execution
still need validation in your environment.
