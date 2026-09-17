# Detailed dossier and executive report

## Reference files and scope

The supplied `detailed.tex` is a multi-rule dossier with a cover, contents,
plain-language explanation, rule definitions, occurrence evidence, profiles,
chronology, parameters, methodology, limitations and glossary. The supplied
`relatorio.pdf` is a two-page executive R5 summary with a decision overview,
organizational coverage, UOR ranking, daily counts and caveats.

The toolkit now follows those two structures for the **R5 analysis actually
computed**. It uses the reference's restrained blue title, tables, restricted
information header and Brazilian number formatting. Other rules, device
profiles, severity scores and organizational color policies in the detailed
reference are not inferred from R5 inputs. No historical people, findings or
counts from the supplied documents are used as current-run results. The
original reference documents remain unchanged and are not distributed.

## Files produced by compare and r5

| File | Contents |
| --- | --- |
| `detailed.tex` / `detailed.pdf` | Cover, contents, plain-language summary, population, per-window counts, R5 definition, UOR recurrence, detailed evidence, user profiles, chronology, full parameters, methodology, coverage, limitations, glossary and review guidance |
| `relatorio.tex` / `relatorio.pdf` | Decision summary, counts/means by window, comparison, UOR coverage and rankings, daily counts, caveats and supporting-file pointers |
| `perfis_r5_<window>min.csv` | Every flagged user, occurrence count, current UOR/dependency if available, first and last VPN anchor |
| `funcionarios_r5_curio_<window>min.csv` | Every flagged user with organizational assignment, when enrichment is available |
| `resumo_r5_por_uor_<window>min.csv` | Complete organizational aggregation, including UNKNOWN separately |
| `report_status.json` | Whether TeX and PDFs were generated, omitted or failed |

Existing `backtesting_dossier.*` and `relatorio_executivo_r5.*` filenames remain
as identical compatibility copies. Full findings remain in each window's
original CSV/JSONL. The default `--sample-limit 20` limits occurrence examples
and displayed user profiles per window, **never aggregate counts or CSV data**.
Examples follow the saved user/time order; no severity ranking is invented.
The dossier states how many occurrences were displayed and omitted from its
body. Set `--sample-limit 0` to keep analytical sections without examples.

The executive ranking shows up to 15 units for one window, or five per window
for a comparison, sorted by alert count. Unknown units remain in overall
counts and have an explicit unresolved count. Organizational coverage uses
flagged employees as its denominator; zero flagged employees means coverage
is not applicable. Page counts depend on the period, evidence limit and text
length; the reference's two-page summary is a design target, not truncation.

## PDF behavior and dependencies

PDFs are now generated **automatically when `pdflatex` is installed**. Without
it, the tool writes TeX and a clear status/warning that no PDFs were produced.
`--compile-pdf` makes PDFs mandatory and fails clearly if compilation cannot
succeed. `--no-pdf` generates TeX only. These two flags are mutually exclusive.

The templates use `article`, `inputenc`, `fontenc`, `geometry`, `longtable`,
`array`, `booktabs`, `xcolor`, `fancyhdr` and `hyperref`; `lmodern` is optional.
These are a subset of the packages in the supplied detailed reference.
No new Python dependency is required. The compiler runs twice, with shell
escape disabled and a five-minute timeout per pass. Source values are escaped
and long IDs/paths get line-break opportunities. Inspect `detailed.build.log`
or `relatorio.build.log` if compilation fails. TeX and analytical files remain
available. A regeneration removes prior generated PDFs before compilation so
an obsolete PDF cannot be mistaken for a new successful build.

## Generate reports without querying DB2 again

Let the current run finish before updating its source files. Set the actual
completed report directory printed by your run:

```sh
R5_REPORT=output/contrast_sep07_14/report_REPLACE_WITH_ACTUAL_TIMESTAMP
"$R5_PY" run_analysis.py report --directory "$R5_REPORT" --compile-pdf
```

This verifies the existing report manifest and regenerates presentation using
its saved findings and summary. It does not query DB2 or Curio, and rewrites
the generated report files in that directory while preserving the findings.
A saved `enrichment/profiles.json` in that directory is reused automatically.
Otherwise supply an existing profile file to include the organizational tables:

```sh
"$R5_PY" run_analysis.py report --directory "$R5_REPORT" \
  --profiles /path/to/existing/profiles.json --compile-pdf
```

If only a complete source cache exists, regenerate the analysis and reports
entirely offline in a new output directory:

```sh
"$R5_PY" run_analysis.py compare \
  --cache output/contrast_sep07_14/source.sqlite --windows 15 240 \
  --out output/reformatted_reports --compile-pdf
```

Add `--profiles /path/to/existing/profiles.json` for saved UOR assignments.
With no organizational data, both reports explicitly state that enrichment
was not performed. They do not invent dependency names or a UOR ranking.

## Live organizational enrichment

To query the existing Curio sidecar after a cached comparison:

```sh
"$R5_PY" run_analysis.py compare \
  --cache output/contrast_sep07_14/source.sqlite --windows 15 240 \
  --env-file .env --with-agencies --compile-pdf \
  --out output/reports_with_uor
```

This makes Curio requests, but no DB2 requests. `--with-agencies` and
`--profiles` are alternatives. On a new online extraction, the same flags
apply to the normal dated `compare`/`r5` command. The final dossier and
executive PDF are built **after** organizational enrichment, so their UOR
figures match the exported aggregations. Current assignment is identified as
such, not presented as historical placement.

## Validation

The report regression tests use synthetic evidence. They check structural
sections, Brazilian formatting, totals independent of example limits, missing
and zero-denominator organizational data, offline regeneration, enrichment
before PDF compilation, stale-PDF removal, missing-compiler behavior, TeX
escaping of hostile strings, PDF text content and manifest verification.
The local TeX integration check compiles both PDFs with existing tools; it
skips only when TeX or Poppler is unavailable. No DB2 query is needed for it.
