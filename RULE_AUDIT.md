# R5 rule audit and required organizational enrichment

## Business facts and the remaining inference

The user confirms that remote access requires VPN, every VPN authentication is
recorded, and the VPN table records **authentication/connection establishment**.
A typical workday has a morning authentication and another after lunch;
reconnections can create additional records. These are user-confirmed operating
facts, not independently measured DB2 findings. No verified physical/VPN IP
range inventory is available.

Under the exclusive physical-or-VPN model, an access established as non-VPN
is physical-network access. The old rule does not establish that premise:
it tests absence of a matching authentication within a short time window.
An authentication at 08:00 can support activity at 10:00 without another
authentication. A 15-minute window will not associate that activity with the
08:00 record. Four hours moves the cutoff; it does not describe a session.
No session-end column or session-lifetime contract is evidenced in this toolkit.
We do not invent logout timestamps or assume that the next authentication
terminates the previous session.

## What the historical rule actually computes

1. Associate an OpenAM event with the nearest VPN authentication for the same
   employee/IP within +/-delta.
2. Treat another private-IP OpenAM event with no such association as a physical
   candidate if its IP differs and it is within +/-delta of the first event.
3. Emit every qualifying pair. Ten events on one side and ten on the other
   can generate 100 pairs for one employee-day, even after flow deduplication.

It is not a same-day transition detector: it can cross midnight, miss valid
same-day activity separated by more than delta, and label an IP as a physical
candidate even though the same employee authenticated to VPN on that IP earlier
that day. The VPN-to-candidate gap can reach 2*delta because the rule uses two
separate comparisons. Larger windows can also remove findings by reclassifying
candidates as VPN-associated. Pair totals are not monotone in delta.

A high pair count does not establish a high employee count. Pair multiplication
alone cannot explain a genuinely high **distinct employee** count. Nor is a
same-day network switch by itself proof of misconduct: it can match the desired
behavioral definition while still being legitimate.

The historical source and classification remain unchanged for comparison.
New outputs audit that rule; they are not advertised as a corrected,
session-aware classifier or as a complete detector of all same-day switches.

## New audit outputs

- `rule_audit.json`: per-window pair totals, same-day/cross-day totals,
  employee-days, maximum pairs per employee-day and daily distinct employees.
- `rule_audit_daily.csv`: daily pair and distinct-employee counts, including
  the same-day subset. Zero days remain explicit.
- `employee_days_<window>min.csv`: one row per employee/VPN-anchor date,
  current UOR/dependency, pair counts, same-day directional candidate counts,
  minimum absolute VPN-to-candidate gap, and a representative occurrence ID.
  This aggregates pairs; it does not count actual physical movements.
- Both reports distinguish pairs, people over the period and people per day,
  and show these audit results. Existing `_physical` CSV field names remain
  compatible but their display labels say **candidate**.

The audit additionally checks each candidate IP against VPN authentications
for that employee in the **entire saved VPN cache**, independently of delta:

- `candidate_ip_vpn_same_day_pairs`: any authentication on the candidate's day;
- `candidate_ip_vpn_earlier_same_day_pairs`: authentication earlier than or at
  the candidate event;
- `candidate_ip_vpn_any_cached_day_pairs`: authentication on any cached day.

These overlapping counters measure conflicting evidence for the old proxy.
They do not prove the session was still open, and zero does not prove physical
presence. The cache contains the requested period and configured filters, not
necessarily every row in the corporate VPN table.

The audit reads only a cache matching the original report's SHA-256. By default
it tries the saved source path and `source.sqlite` beside the report directory.
If neither matches, these counters are `null`, with an explicit unavailable
status. They are never filled with zero to imply an absence of VPN records.
Use `report --source-cache /new/path/source.sqlite` after relocating files.

## Organizational enrichment is required

`compare`, `r5` and `report` now run organizational enrichment before finalizing
the reports. `--with-agencies` remains accepted for compatibility but is no
longer necessary. Configure the existing Curio sidecar through `.env` as before.
Use `--profiles /path/to/profiles.json` to apply saved mappings without Curio.
Unknown assignments remain UNKNOWN and organizational coverage is quantified;
a successful lookup does not guarantee every employee has a usable mapping.

A Curio error returns failure and preserves findings and any completed profile
lookups. `pipeline_status.json` identifies the failed stage and report directory.
Run `report` on that directory to resume enrichment without DB2 or reclassification.
Prior TeX/PDF files are kept under `previous_reports/TIMESTAMP/` during regeneration;
they are not left at the canonical paths as if the new generation succeeded.

## Update and audit an existing completed analysis

From the distribution repository root, after any current process stops, run
each command only if the preceding one succeeds:

```sh
git pull --ff-only
sha256sum -c SHA256SUMS
umask 077
age --decrypt --output r5-rule-update.zip r5-toolkit.zip.age
python3 -m zipfile -e r5-rule-update.zip .
cd analise_agencia_toolkit
R5_REPORT=output/contrast_sep07_14/report_REPLACE_WITH_ACTUAL_TIMESTAMP
"$R5_PY" run_analysis.py report --directory "$R5_REPORT" --env-file .env --compile-pdf
```

For a `work/analise_agencia_toolkit` installation, extract into `work` and change
to that directory. Use the existing passphrase and corporate Python. The update
preserves `.env`, output and source caches. `--compile-pdf` requires pdflatex;
omit it for automatic PDF generation when installed, otherwise TeX only.

The last command uses Curio for enrichment, reads local findings/cache and makes
**no DB2 query**. For fully offline operation add `--profiles` with saved mappings.
If extraction completed but no report directory exists, use:

```sh
"$R5_PY" run_analysis.py compare \
  --cache output/contrast_sep07_14/source.sqlite --windows 15 240 \
  --env-file .env --out output/rule_audit --compile-pdf
```

This reclassifies locally, then enriches. Never delete a useful source cache
just to change the presentation or obtain these diagnostics.

## What to inspect and return

Start with `rule_audit.json` and `rule_audit_daily.csv`: compare pair counts with
daily distinct employees and the earlier-authentication counter. Inspect the
employee-day examples locally. Return aggregate counts and the exact error/
`pipeline_status.json` if a stage fails; redact identifying information. Keep
raw IPs, names, profiles, employee IDs and source caches within the corporate
environment. The provided historical counts cannot substitute for this latest
run's actual outputs.

## Corrected detector design and validation boundary

The intended primary unit should be employee/local-date, with chronological
network transitions as supporting evidence. VPN classification must be separate
from any threshold used to rank rapid transitions. VPN session intervals or
verified address-ownership semantics would support that classifier. With only
authentication starts, an earlier same-day match can support a possible VPN
association; an unmatched event should remain unresolved until the matching
and retention assumptions establish non-VPN access. Simply shrinking delta
or assuming exactly two authentications per day does not fix the inference.

Local synthetic tests reproduce pair multiplication, a same-day earlier VPN
record missed by delta, midnight crossings, tied timestamps, and missed distant
same-day activity. They also check mandatory enrichment, retry without DB2,
explicit offline mappings and unavailable-cache handling. No corporate counts,
classification accuracy or production performance are claimed.
