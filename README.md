# R5 Kit

An age-encrypted ZIP containing a portable R5 analysis toolkit, including the
DB2 authentication and ASUTIME fixes (17 September 2026). This guide covers download, decryption, installation,
configuration, the 15-minute/four-hour comparison, pause/resume, reports and
diagnostics.

The public repository contains the encrypted package, this guide and a checksum.
The decryption passphrase is supplied separately; it is not stored here. The
archive contains executable source, tests, SQL templates and configuration
examples. It contains no real credentials, corporate extracts or prior results.
Keep decrypted files and all generated results in approved local storage.

## Required enrichment and rule audit

Organizational enrichment is now required for `compare`, `r5` and `report`.
They use Curio by default; provide `--profiles` for fully offline mappings.
`--with-agencies` is no longer necessary. Final reports include daily distinct
employees, same-day/cross-day pair counts and candidate-IP checks against
saved VPN authentications. Read [RULE_AUDIT.md](RULE_AUDIT.md) for the rule's
limits and how to audit/enrich an existing run without querying DB2 again.

## Recovery from the error2 JSON failure

A single malformed MTA payload no longer aborts extraction. The updated toolkit
records unusable rows locally, continues valid rows, and discloses the analysis
coverage gap in both reports. Existing checkpoints remain resumable without
repeating completed partitions. Follow [ERROR2.md](ERROR2.md) to update and
resume your interrupted run, preserving `.env` and `output/`.

## Detailed dossier and executive summary

The reference-based outputs are now **`detailed.tex` / `detailed.pdf`** and
**`relatorio.tex` / `relatorio.pdf`**. PDFs are generated automatically when
`pdflatex` exists; `--compile-pdf` requires them and `--no-pdf` skips compilation.
Organizational results are required before finalization, using Curio or
an existing `--profiles` file. Full findings remain in CSV/JSONL; the dossier
states its evidence-display limit.

After your current run finishes and you update the toolkit, rebuild its
reports **without rerunning DB2**:

```sh
R5_REPORT=output/contrast_sep07_14/report_REPLACE_WITH_ACTUAL_TIMESTAMP
"$R5_PY" run_analysis.py report --directory "$R5_REPORT" --compile-pdf
```

Use the actual report directory printed by your execution. Read [REPORTS.md](REPORTS.md)
for both layouts, offline UOR enrichment, dependencies and complete commands.

## Future runs: exhaustive accounting and performance

**Leave the current running process alone; update its files after it finishes.**
This release keeps the current SQL/config signatures and default estimated
mode compatible. For a future exhaustive run, explicitly pass `--mta-scope full`
and choose a new output directory. Reuse its complete cache for subsequent
window comparisons without querying DB2 again.

New work includes learned query sizes after resource/cap failures, skipping
only ID gaps confirmed by an unfiltered DB2 lookup, faster local correlation,
and an offline interval audit:

```sh
"$R5_PY" run_analysis.py coverage --cache output/contrast_sep07_14/source.sqlite
```

`coverage_audit.json` distinguishes completed intervals from a verified database
snapshot. **Only live tables are available, so historical completeness and a
consistent DB2 snapshot cannot be guaranteed.** The audit checks interval
accounting and VPN-day completion; it never turns those into a snapshot claim.
Read [COVERAGE.md](COVERAGE.md) for the algorithm, conditional proof, synthetic
performance measurements, limitations and exact commands for full runs.

## Quick recovery: 17 September ASUTIME error

The `global MTA ID extent` failure came from a combined MIN/MAX query introduced
by the toolkit. This release restores separate endpoint queries and the
historical date-to-ID estimation approach. It also avoids scanning the entire
retained ID range by default and removes the redundant final extent query.

For the layout shown in your screenshot, open the **r5-kit repository root**
(the parent of your existing `analise_agencia_toolkit` directory) and run:

```sh
git pull --ff-only
sha256sum -c SHA256SUMS
umask 077
age --decrypt --output r5-update.zip r5-toolkit.zip.age
python3 -m zipfile -e r5-update.zip .
cd analise_agencia_toolkit
"$R5_PY" run_analysis.py compare \
  --start 2026-09-07 --end 2026-09-14 --windows 15 240 \
  --env-file .env --out output/contrast_sep07_14 --resume
```

Enter the same separately supplied passphrase. Stop if download verification
or decryption fails; do not extract a partial ZIP. The bundle contains no
`.env`, credentials, `source.sqlite` or generated results, so extraction replaces
code/docs while leaving your configured access and previous run intact.
For the layout with `work/analise_agencia_toolkit`, extract into `work` instead
of `.` and then `cd work/analise_agencia_toolkit`. If `R5_PY` is unset in a new
terminal, set it to the existing corporate virtual-environment Python as in
section 5. An optional local check is `"$R5_PY" -m unittest discover -s tests -q`.

A checkpoint stopped before MTA planning is backed up and upgraded automatically;
completed VPN days are reused. Keep the same `--out`, dates and filters. A cache
with MTA work already started is deliberately not migrated: preserve it and
use a new `--out` if the tool reports that mismatch.

**The estimate is a performance tradeoff, not proof of complete coverage.**
`mta_plan.json`, summary JSON and both TeX reports record its limits. Query
performance still needs confirmation in your corporate DB2. If another query
fails, return its `queries.jsonl` entry, the last lines of `run.log`, and
`mta_plan.json` if present, after reviewing them for sensitive identifiers.

## 1. Download

Run these commands in a WSL/Linux terminal:

```sh
git clone https://github.com/SaganGromov/r5-kit.git
cd r5-kit
```

If Git is unavailable, open <https://github.com/SaganGromov/r5-kit>, choose
**Code > Download ZIP**, and extract that GitHub download. Change into the
directory containing `README.md`, `SHA256SUMS` and `r5-toolkit.zip.age`.

The ZIP downloaded from GitHub is only a repository wrapper. The toolkit itself
is inside `r5-toolkit.zip.age`; decrypt that file in step 3.

## 2. Check tools and the download

```sh
age --version
python3 --version
sha256sum -c SHA256SUMS
```

The checksum command must report `r5-toolkit.zip.age: OK`. If it does not,
download the repository again before decrypting. This checksum detects damaged
or mismatched downloads; it is not a signature from an independent authority.

### If age is not installed

Use an already approved installation where possible. On a personal Ubuntu/WSL
machine with administrator access, the official project lists `apt install age`
as an installation option:

```sh
sudo apt update
sudo apt install age
```

Administrator access is not required to run age. If you cannot install packages,
obtain the correct Linux binary from the [official age releases](https://github.com/FiloSottile/age/releases)
or follow the [official installation instructions](https://github.com/FiloSottile/age#installation).
Choose the architecture shown by `uname -m`; unpack the binary into a directory
you control and invoke it by its full path. Use an approved transfer route if
the corporate machine cannot access the release site. Do not change corporate
security settings or run an unapproved installer.

Decryption can also be performed on your personal machine, followed by transfer
of the resulting toolkit directory through an approved route. The analysis
itself runs in corporate WSL/Linux because it requires corporate DB2 access.

## 3. Decrypt the package

```sh
umask 077
age --decrypt --output r5-toolkit.zip r5-toolkit.zip.age
```

Enter the supplied passphrase at the prompt. Characters are normally hidden
while you type. Do not add the passphrase to this command, shell history or a
configuration file. This is **age passphrase encryption around a ZIP**, not a
ZIP password understood by 7-Zip or `unzip`.

Successful decryption exits with code zero and produces `r5-toolkit.zip`.
If age reports a wrong passphrase or decryption error, check the supplied
passphrase and the encrypted-file checksum. Do not use a partial output file.

The age command syntax is documented in the [official manual](https://filippo.io/age/age.1).

## 4. Extract into a new directory

```sh
mkdir work
python3 -m zipfile -e r5-toolkit.zip work
cd work/analise_agencia_toolkit
```

If `work` already exists, use a different empty directory instead. Extraction
into a fresh directory avoids accidentally overwriting a previous run.

The extracted layout is:

```text
analise_agencia_toolkit/
    README.md
    BUNDLE.md
    COVERAGE.md
    REPORTS.md
    MANIFEST.json
    THIRD_PARTY_NOTICE.md
    DIAGNOSIS.md
    .env.example
    .gitignore
    run_analysis.py
    config/analysis.example.json
    scripts/source.py
    scripts/r5.py
    scripts/reports.py
    scripts/report_layout.py
    sql/vpn.sql
    sql/mta.sql
    sql/id_min.sql
    sql/id_max.sql
    sql/id_sample.sql
    tests/test_toolkit.py
    tests/test_exhaustive.py
    tests/test_reports.py
    output/.gitkeep
```

No original diagnostic archive, session transcript, corporate dataset or
previous result is included. `DIAGNOSIS.md` inside the encrypted package
documents the prior connection failure and the previously used corporate
interpreter/connection-file paths. These are references to existing corporate
configuration, not bundled credentials.

### Verify the extracted files

Run from the extracted toolkit directory:

```sh
python3 - <<'PY'
import hashlib, json
from pathlib import Path
manifest = json.loads(Path('MANIFEST.json').read_text())
for name, expected in manifest['files'].items():
    data = Path(name).read_bytes()
    assert len(data) == expected['bytes'], name
    assert hashlib.sha256(data).hexdigest() == expected['sha256'], name
print('All packaged files match MANIFEST.json')
PY
```

The package manifest covers every delivered file except itself. Keep it for
provenance; expected local edits such as creating `.env` are outside it.

## 5. Select the existing corporate Python environment

The toolkit requires **Python 3.11 or newer**. Source extraction additionally
requires the existing **ibm_db** package and native DB2 client configuration.
Prefer the virtual environment which already connected successfully in the
previous corporate analysis. No dashboard, Java, pandas or Docker installation
is needed to run the toolkit. Native Windows execution is not supported; cache
locking uses Linux `fcntl`, which works in WSL.

Replace the placeholder below with your existing corporate interpreter path.
The previously recorded path is in the encrypted `DIAGNOSIS.md`:

```sh
export R5_PY="/absolute/path/to/your/existing/.venv/bin/python"
test -x "$R5_PY"
"$R5_PY" --version
"$R5_PY" run_analysis.py doctor
"$R5_PY" -m unittest discover -s tests -v
```

The test suite should report **65 tests, OK**. It uses synthetic fixtures and
does not connect to DB2 or Curio. Plain `doctor` checks local dependencies and
configuration without connecting. A successful test suite or `ibm_db` import
does **not** establish that authentication or a SELECT query works.

If `doctor` reports `ibm_db` unavailable, choose the correct existing virtual
environment. Do not assume that the global `python3` has the same driver.
The historical project declared `ibm-db>=3.2.0`; the actual corporate version
is reported by `doctor` when package metadata is available. If no suitable
environment exists, have the corporate support team provision the existing
approved driver/native-client setup. This bundle does not contain installers,
native libraries, licenses or credentials.

## 6. Configure the endpoint and credentials locally

For a fresh installation only:

```sh
cp .env.example .env
chmod 600 .env
```

Edit `.env` in your local editor. Do not overwrite an existing configured `.env`.
Use your real corporate values for either:

```text
DB2_HOST=your-actual-db2-host
DB2_PORT=your-actual-port
DB2_DATABASE=your-actual-database
```

or the already used JDBC-shaped endpoint setting:

```text
DB2_JDBC_URL=jdbc:db2://your-actual-db2-host:your-actual-port/your-actual-database
```

The URL describes the endpoint; Python still connects using `ibm_db`, not Java.
Use one endpoint configuration consistently. Do not put authentication values
or extra options in `DB2_JDBC_URL`.

Prefer a reference to your **existing** corporate credential file:

```text
DB2_CREDENTIALS_FILE=/absolute/path/to/existing/db2_credentials.json
```

That file must contain a JSON object with nonempty string keys `username` and
`password`. Its values and path are supplied by your corporate environment.
If you already use exported `DB2_USERNAME` and `DB2_PASSWORD`, those can be used
instead. Configure both together, or leave both unset when using the credential
file. An environment variable exported in the terminal takes precedence over
the `.env` file. The toolkit loads `.env` only when `--env-file .env` is passed.
The `.env` format is simple `KEY=VALUE`; it is read as data, not executed.

The fixed client uses the historical `ibm_db.connect(dsn, "", "")` pattern,
with credentials inserted into an in-memory DSN and excluded from toolkit
logs. Semicolons, braces, control characters or surrounding whitespace in
credential values are rejected rather than ambiguously parsed. If this affects
your actual credentials, report the character category without disclosing the
credential. Do not change authentication or TLS settings to bypass an error.

## 7. Test the actual DB2 connection first

```sh
"$R5_PY" run_analysis.py doctor \
  --env-file .env --connect --out output/doctor
```

This connects to DB2 and performs zero-row SELECTs referencing the documented
columns. Proceed only when it reports successful connection and column checks.
Success confirms authentication, SELECT access and acceptance of those column
names. It does not confirm source retention, timestamp semantics, full JSON
transfer or the performance of a full extraction.

On failure, the tool prints the diagnostic directory and retains `doctor.json`.
If connection was attempted, it also writes a redacted `connection.jsonl`.
Do not delete checkpoints or change server authentication settings. Section 14
explains what to inspect and return.

## 8. Preview the interrupted analysis without querying

The pending request is to compare **15-minute and four-hour windows over the
same period, 7–14 September 2026**, with technical and executive TeX reports.

```sh
"$R5_PY" run_analysis.py compare \
  --start 2026-09-07 --end 2026-09-14 --windows 15 240 --dry-run
```

Dry-run prints SQL templates, bindings and the dependent extraction steps,
without connecting or importing the DB2 driver. It does not invent ID bounds
which can only be obtained from DB2.

Dates are **inclusive local calendar dates**. The technical interval for this
command is `[2026-09-07 00:00, 2026-09-15 00:00)`. No weekends or zero-alert
days are silently removed. The primary mean divides by all eight calendar
days. The windows are symmetric: `15` means plus/minus 15 minutes; `240` means
plus/minus four hours.

## 9. Run the complete comparison

```sh
"$R5_PY" run_analysis.py compare \
  --start 2026-09-07 --end 2026-09-14 --windows 15 240 \
  --env-file .env --out output/contrast_sep07_14 --resume
```

`--resume` works for a new directory, an interrupted compatible extraction, or
a completed extraction. It prevents the need to delete an existing checkpoint.

The command automatically:

1. Reads eligible VPN events and stores them in a local SQLite cache.
2. Reads the minimum and maximum MTA IDs in separate queries, samples three
   timestamps, estimates the requested ID range, and scans bounded partitions
   of that range. It subdivides on resource limits or a result-cap sentinel.
3. Parses source JSON locally and commits completed partitions durably.
4. Reclassifies each window independently using the same cached source rows.
5. Writes full evidence, daily counts, comparison statistics, technical TeX and
   executive TeX into a new `report_TIMESTAMP` directory.

You do not copy query results manually. The default `--mta-scope estimated`
follows the historical three-sample ID estimate with a 20% plus 1,000-ID
margin. `mta_plan.json` records the selected range and samples. Late or
out-of-order events may lie outside it: complete coverage is not guaranteed.
This limitation also appears in the JSON summary and both TeX reports.
Invalid samples or dates outside the estimate stop with an error; no full
scan is started automatically. `--mta-scope full` explicitly scans every
retained ID and can cost much more. Use a new output directory to switch scope.
Neither mode has a two-hour completion guarantee.

The source data needed for a past period may have expired. Missing retained
data cannot be reconstructed by rerunning a query. Treat coverage warnings as
limitations, not as evidence that there were no historical alerts.

### PDF output

PDFs are automatic when `pdflatex` is available. Add `--compile-pdf` to require them:

```sh
"$R5_PY" run_analysis.py compare \
  --start 2026-09-07 --end 2026-09-14 --windows 15 240 \
  --env-file .env --out output/contrast_sep07_14 --resume --compile-pdf
```

The templates require `article`, `inputenc`, `fontenc`, `geometry`, `longtable`,
`array`, `booktabs`, `xcolor`, `fancyhdr` and `hyperref`; `lmodern` is optional.
Shell escape is disabled. Use `--no-pdf` for TeX only. See REPORTS.md.

## 10. Pause and resume safely

To stop after the current query boundary, create a file named `STOP`:

```sh
touch output/contrast_sep07_14/STOP
```

Alternatively press Ctrl+C, or pass `--max-seconds 2100` to request a pause
after roughly 35 minutes. A native driver call can delay interruption until
it returns, so this is not a hard SQL timeout.

Completed partitions are committed with their progress marker. An unfinished
query/day is rolled back and repeated. Remove the STOP file before resuming:

```sh
rm -f output/contrast_sep07_14/STOP
"$R5_PY" run_analysis.py compare \
  --start 2026-09-07 --end 2026-09-14 --windows 15 240 \
  --env-file .env --out output/contrast_sep07_14 --resume
```

Do not remove `source.sqlite`. Parameter, endpoint or SQL-signature changes
require a separate output directory. The connection fix is compatible with
the checkpoint from the previous failed authentication attempt.

Extraction is checkpointed. Offline classification and report generation can
be restarted without additional DB2 access. A report directory without its
final `manifest.json` must not be treated as complete.

## 11. Reuse captured sources without DB2

Once extraction completes, the DB2 connection is unnecessary for new report
runs at supported windows:

```sh
"$R5_PY" run_analysis.py compare \
  --cache output/contrast_sep07_14/source.sqlite \
  --windows 15 240 --out output/offline_contrast --compile-pdf

"$R5_PY" run_analysis.py r5 \
  --cache output/contrast_sep07_14/source.sqlite \
  --delta 60 --out output/offline_60min
```

Enrichment still uses Curio: add `--env-file .env` for its settings or
`--profiles /path/to/profiles.json` for fully offline mappings.
Use `--no-pdf` when TeX is unavailable. With `--cache`, dates and filters
come from the saved signature. Do not add `--start`, `--end`, `--config`,
`--resume` or `--max-seconds`. A requested window cannot exceed the maximum
window supported by the extraction. A cache collected for 240 minutes supports
15, 60 and 240 minutes, but not 300 minutes.

For extraction without immediately generating reports:

```sh
"$R5_PY" run_analysis.py extract \
  --start 2026-09-07 --end 2026-09-14 --max-delta 240 \
  --env-file .env --out output/source_only --resume
```

## 12. Other periods, filters and organizational analysis

### Repeat a single-window analysis

```sh
"$R5_PY" run_analysis.py r5 \
  --start 2026-08-18 --end 2026-08-31 --delta 15 \
  --env-file .env --out output/august_15min --resume
```

Change the dates/window as needed, respecting source retention. Comparing
means from different periods does not isolate the effect of changing the
window; use `compare` for an equal-period contrast.

### Change analysis configuration

```sh
cp config/analysis.example.json config/analysis.local.json
```

Edit the local JSON, then add `--config config/analysis.local.json` to an
extraction/online analysis command with a fresh output directory. Supported
keys are:

| Key | Meaning |
|---|---|
| `vpn_schema`, `vpn_table` | Documented VPN source location |
| `mta_schema`, `mta_table` | Documented OpenAM source location |
| `decision_nodes` | Successful authentication nodes to include |
| `physical_ip_cidrs` | Networks eligible for classification |
| `users` | Restricted user list; empty means all eligible users |
| `excluded_users` | Users excluded from the analysis |
| `mta_scope` | `estimated` (default) or `full`; CLI `--mta-scope` overrides this setting |
| `id_span` | Initial ID partition width, default 50,000 |
| `row_limit` | Accepted rows per query, default 10,000; an extra row detects truncation |

Do not invent alternate column names or joins. Use the supplied defaults unless
there is evidence to change them. `--sample-limit` changes the number of report
examples, not the counts. No maximum-record option silently truncates findings.

### Enrich all users and aggregate by UOR

Enrichment is required and uses the already configured corporate Curio sidecar
unless you supply `--profiles` for saved mappings.
Set `CURIO_BASE_URL` in `.env` to its actual URL. Local HTTP and trusted HTTPS
are supported; HTTPS certificate verification remains enabled. Configure
`CURIO_CA_FILE` only if your existing setup needs a trusted CA file.
The toolkit does not create containers, load sidecar credentials or modify
operation registration.

`compare` and `r5` always run the organizational pipeline. The legacy
`--with-agencies` flag is accepted but unnecessary. The pipeline collects
the union of users from both comparison windows:

```sh
"$R5_PY" run_analysis.py compare \
  --cache output/contrast_sep07_14/source.sqlite \
  --windows 15 240 --env-file .env \
  --out output/contrast_with_uor --with-agencies
```

You can also run each step independently. Replace the report-directory
placeholder below with the exact path printed by the analysis command:

```sh
export R5_REPORT="output/contrast_sep07_14/report_TIMESTAMP"

"$R5_PY" run_analysis.py enrich \
  --findings "$R5_REPORT/r5_240min.jsonl" \
  --env-file .env --out output/current_org

"$R5_PY" run_analysis.py agencies \
  --findings "$R5_REPORT/r5_240min.jsonl" \
  --profiles output/current_org/profiles.json --out output/uor_summary
```

Reusing `output/current_org` resumes successful lookups. Use a fresh directory
to refresh current assignments. Add `--uor YOUR_NUMERIC_UOR` to filter a known
unit after enrichment, or `--min-occurrences 3` to change the recurrence flag
threshold. There is no invented DB2 agency column. Unknown assignments remain
separate. A UOR is not necessarily a retail agency, and current assignments
do not establish historical placement on the event date.

## 13. Locate and verify results

The command prints the exact report directory. Each report run creates a new
timestamped directory so previous deliverables are preserved.

| File | Purpose |
|---|---|
| `source.sqlite` | Sensitive local source cache and extraction checkpoints |
| `run.log` | Extraction milestones and failures |
| `connection.jsonl` | Redacted connection outcomes, including failures before any query |
| `queries.jsonl` | Actual SELECTs, ordered bindings, row counts, timings and errors |
| `r5_15min.jsonl/.csv`, `r5_240min.jsonl/.csv` | Complete three-event findings for each window |
| `daily.csv` | Every calendar day, including zero-alert days |
| `coverage_audit.json` | Interval/VPN-day accounting; explicit unverified snapshot and historical-completeness flags |
| `summary.json` | Means, counts, overlap/window-only findings, source coverage and provenance |
| `detailed.tex` / `detailed.pdf` | Detailed technical dossier; old `backtesting_dossier.*` names retained |
| `relatorio.tex` / `relatorio.pdf` | Executive report; old `relatorio_executivo_r5.*` names retained |
| `manifest.json` | Report file sizes and SHA-256 checksums |
| `profiles.json` | Current organizational mappings and lookup status |
| `funcionarios_r5_curio.csv` | All counted employees with available UOR data |
| `resumo_r5_por_uor.csv`, `relatorio_uor.tex` | Recurrence by organizational unit |

Check a generated report:

```sh
"$R5_PY" run_analysis.py verify --directory "$R5_REPORT"
```

Findings are event combinations, not distinct people or proven infringements.
One employee can generate many findings. Widening the time window can also
remove findings by reclassifying a supposedly physical login as VPN; the
smaller-window results need not be a subset of the larger-window results.
Source gaps are explicitly flagged. A zero alert count does not establish
complete source retention. Ratios are undefined when the smaller count is zero.

CSV files use UTF-8 with BOM. Formula-like text is neutralized in CSV; JSON
preserves original text. DB2 timestamps are expected without offsets and are
interpreted using the historical local-time convention. Microseconds are kept.
The local extraction is not a single DB2 transaction snapshot; source changes
or expiry during a long run can affect completeness.

## 14. Troubleshooting and what to return

| Symptom | Action |
|---|---|
| Decryption fails | Check the supplied passphrase and `sha256sum -c SHA256SUMS`; use age, not ZIP-password software. |
| `ibm_db` import fails | Select the working corporate virtual environment; check the approved native client setup. |
| Connection settings missing | Configure the real endpoint and existing credential-file reference; exported variables override `.env`. |
| SQL30082N reason 17 on the earlier toolkit | Update to this package, run `doctor --connect`, then use `--resume`. The DSN authentication bug is fixed here; server authentication changes are not part of the fix. |
| Connection still fails | Return `doctor.json` and the relevant redacted `connection.jsonl` entry. |
| Object/column error | Return the SQLCODE/SQLSTATE and failed `queries.jsonl` entry. Do not guess replacement columns. |
| Malformed JSON or unexpected payload type | Inspect the indicated row locally and describe the structure without sharing raw corporate content by default. The current partition is rolled back. |
| Resource-limit failures | This release removes the combined MIN/MAX discovery query. Return the failing query label, SQLSTATE and timings. Extraction partitions split; failed endpoint/sample probes stop without an automatic full scan. |
| Existing cache error | Use identical settings and `--resume`, or a separate output directory. Do not delete the checkpoint as a routine recovery step. |
| Curio 404 despite healthy service | Confirm that both organizational operations were loaded when the existing sidecar started. |
| Missing UOR | Keep UNKNOWN; inspect lookup status. Missing dependency name and missing UOR are distinct conditions. |
| PDF build fails | Inspect the generated `.build.log`; TeX and analytical outputs remain available. Run offline with `--no-pdf` if only `.tex` is required. |

Exit codes: `0` completed; `1` configuration/execution error; `2` invalid
command-line usage; `130` interruption/pause. Add `--verbose` for more toolkit
logging. Do not enable unreviewed native driver traces, which may contain
authentication material.

If something fails, return:

- The command, with credentials removed.
- `doctor.json` and relevant `connection.jsonl` lines for connection failures.
- The relevant `run.log` excerpt and failed `queries.jsonl` entry for query failures.
- SQLCODE/SQLSTATE and coverage/parameters from `summary.json`, when available.
- Relevant `.build.log` lines for a TeX failure.

Do not upload `.env`, the credentials file, full source caches, raw findings,
employee profiles or complete native traces. Review even redacted toolkit logs
before sharing: they can contain corporate identifiers or configured user
filters. The public repository is a distribution channel, not a diagnostics
or corporate-results store.

## 15. Updating an existing installation

Download and decrypt the new package into a separate empty directory. Retain
the old toolkit directory as a recoverable copy. Transfer your existing `.env`
and the relevant `output/` run directory into the new toolkit using your local
file manager or approved copy procedure. Do not replace configured files with
examples. Run tests and `doctor --connect`, then use the same analysis command
with `--resume`.

Only caches with compatible extraction parameters/signatures can be resumed.
The 17 September correction automatically upgrades the exact original
checkpoint only if MTA planning/data never started and the other parameters
match. It first saves `source.sqlite.before-mta-fix.sqlite`, then preserves
completed VPN days. If MTA extraction had already started, preserve that
cache and choose a new output directory. Complete old caches remain usable
for offline analysis with `--cache`.

## Validation and limits

The bundle is checked locally by encrypting/decrypting it, verifying ZIP/file
integrity, and running its 54 synthetic regression tests from the extracted
copy. Earlier offline tests also covered relocation, report verification,
historical pure-correlation parity and TeX compilation. The corporate
diagnostic confirmed Python 3.13.1, driver import and local test execution,
but authentication failed in the initial implementation. A later screenshot
reached query execution and failed at the combined MTA MIN/MAX statement with
SQL0905N / ASUTIME. This release addresses that specific regression.

Live performance of the revised SELECTs, real query compatibility,
source retention, runtime and organizational service availability still need
corporate validation. No new corporate result is included or claimed.
