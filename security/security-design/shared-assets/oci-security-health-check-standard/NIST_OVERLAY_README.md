# NIST Overlay for OCI CIS Reports

This branch adds NIST SP 800-53 mapping output to the OCI CIS Security Health Check report while preserving the existing CIS report behavior.

The same mapping data supports two workflows:

- Live report generation against OCI.
- Offline overlay generation from an existing CIS report directory or `.zip`.

## Disclaimer

This branch is not official Oracle software, is not supported by Oracle Support, and has not been reviewed or approved as an Oracle release. It is an experimental working copy intended for evaluation, discussion, and validation of a possible NIST mapping overlay workflow. Review the code, outputs, and mappings before using them for audit, compliance, ATO, or customer-facing reporting.

## Get Started / Basic Usage

Download this branch as a zip:

```bash
wget https://github.com/cgpavlakos/technology-engineering/archive/refs/heads/cis-nist-overlay.zip \
  -O technology-engineering-cis-nist-overlay.zip
```

Unzip it:

```bash
unzip -q technology-engineering-cis-nist-overlay.zip
cd technology-engineering-cis-nist-overlay
```

Run online and generate a new NIST-only report for all subscribed regions:

```bash
cd security/security-design/shared-assets/oci-security-health-check-standard/files/oci-security-health-check-standard
chmod +x standard.sh
./standard.sh --cis '--nist-mappings'
```

Run offline and overlay an existing Ashburn CIS report zip:

```bash
cd security/security-design/shared-assets/oci-security-health-check-standard/files/oci-security-health-check-standard/scripts/cis_reports

python3 nist_reports.py overlay \
  --input /path/to/DEFAULT_YYYYMMDDHHMMSS_standard_us-ashburn-1.zip \
  --nist-mappings \
  --output /path/to/DEFAULT_YYYYMMDDHHMMSS_standard_us-ashburn-1_nist_overlay.zip
```

The offline overlay does not call OCI APIs or change the source zip. For an Ashburn overlay, use an existing CIS report package that was originally generated for `us-ashburn-1`.

## Files

From the repository root, the implementation files are:

```text
security/security-design/shared-assets/oci-security-health-check-standard/files/oci-security-health-check-standard/scripts/cis_reports/cis_reports.py
security/security-design/shared-assets/oci-security-health-check-standard/files/oci-security-health-check-standard/scripts/cis_reports/nist_reports.py
security/security-design/shared-assets/oci-security-health-check-standard/files/oci-security-health-check-standard/scripts/cis_reports/nist_mappings.json
```

## Mapping Modes

Default CIS behavior remains unchanged unless a mapping flag is used.

| Mode | Flag | Summary mapping columns | Output prefix |
| --- | --- | --- | --- |
| CIS/default | none | `CIS v8`, `CCCS Guard Rail` | `cis_*` |
| NIST-only | `--nist-mappings` | `NIST Controls` | `nist_*` |
| Combined | `--all-mappings` | `CIS v8`, `NIST Controls`, `CCCS Guard Rail` | `combined_*` |

NIST IDs are intentionally preserved in workbook order. Repeated NIST IDs are not collapsed in the current output because duplicate source rows may matter for audit evidence.

## Live OCI Run

Run live reports from the report home directory:

```bash
cd security/security-design/shared-assets/oci-security-health-check-standard/files/oci-security-health-check-standard
chmod +x standard.sh
```

Run a NIST-only live report for Ashburn:

```bash
./standard.sh -r us-ashburn-1 --cis '--nist-mappings'
```

Run a combined live report for Ashburn:

```bash
./standard.sh -r us-ashburn-1 --cis '--all-mappings'
```

The `--cis '<options>'` wrapper passes options through to `cis_reports.py`. A live run requires OCI config credentials and network access to OCI APIs.

Expected live NIST-only outputs are written under a timestamped report directory:

```text
DEFAULT_YYYYMMDDHHMMSS_standard_us-ashburn-1/
  DEFAULT_..._nist_summary_report.csv
  DEFAULT_..._nist_summary_report.json
  DEFAULT_..._nist_summary_report.html
  DEFAULT_..._nist_summary_compliance.png
  DEFAULT_..._nist_summary_compliance_by_focus_area.png
  DEFAULT_..._nist_Consolidated_Report.xlsx
```

The standard wrapper also creates a downloadable `.zip` package for the report directory unless its normal zip behavior is disabled.

Charts are generated during live runs when the report runtime can import Matplotlib.

## Offline Overlay From Existing Zip

Use offline overlay when OCI data has already been collected and you want to add NIST or combined mapping artifacts without rerunning OCI collection.

Run from the CIS reports script directory:

```bash
cd security/security-design/shared-assets/oci-security-health-check-standard/files/oci-security-health-check-standard/scripts/cis_reports
```

Create a NIST-only overlay from an existing zip:

```bash
python3 nist_reports.py overlay \
  --input /path/to/existing-cis-report.zip \
  --nist-mappings
```

Create a combined overlay from an existing zip:

```bash
python3 nist_reports.py overlay \
  --input /path/to/existing-cis-report.zip \
  --all-mappings
```

For zip input, the original zip is not modified. A new zip is created next to the input by default:

```text
existing-cis-report_nist_overlay.zip
existing-cis-report_combined_overlay.zip
```

Use `--output` to choose the destination:

```bash
python3 nist_reports.py overlay \
  --input /path/to/existing-cis-report.zip \
  --nist-mappings \
  --output /path/to/report_nist_overlay.zip
```

Use `--force` only when the output path already exists and should be replaced.

## Offline Overlay From Existing Directory

For an unpacked report directory, the overlay writes files into that directory by default:

```bash
python3 nist_reports.py overlay \
  --input /path/to/existing-cis-report-directory \
  --nist-mappings
```

To avoid modifying the original directory, provide an output directory:

```bash
python3 nist_reports.py overlay \
  --input /path/to/existing-cis-report-directory \
  --all-mappings \
  --output /path/to/report_combined_directory
```

Offline overlay reads the existing `*_cis_summary_report.csv` first. If no CSV is present, it falls back to `*_cis_summary_report.json`.

Expected offline overlay artifacts:

```text
*_nist_summary_report.csv
*_nist_summary_report.json
*_nist_summary_report.html
*_nist_summary_compliance.png
*_nist_summary_compliance_by_focus_area.png
*_nist_Consolidated_Report.xlsx

*_combined_summary_report.csv
*_combined_summary_report.json
*_combined_summary_report.html
*_combined_summary_compliance.png
*_combined_summary_compliance_by_focus_area.png
*_combined_Consolidated_Report.xlsx
```

Offline chart behavior:

- If the source CIS report has `*_cis_summary_compliance.png` and `*_cis_summary_compliance_by_focus_area.png`, the overlay copies them to the selected mode prefix and embeds them in the overlay HTML.
- If those CIS chart PNGs are missing, the overlay still creates CSV, JSON, HTML, and XLSX outputs, but reports that charts were skipped.

## Smoke Tests

Syntax check:

```bash
python3 -m py_compile \
  cis_reports.py \
  nist_reports.py
```

Mapping lookup check from `scripts/cis_reports`:

```bash
python3 - <<'PY'
from nist_reports import NISTMappings

mappings = NISTMappings()
assert mappings.get_control_identifiers(['5.4']) == ['AC-6(2)', 'AC-6(5)']
assert mappings.get_control_identifiers(['6.7']) == ['AC-2(1)', 'AC-3']
assert mappings.get_control_identifiers(['1.1']) == ['CM-8', 'CM-8(1)', 'PM-5']
assert mappings.get_control_identifiers(['5.4', '6.7']) == ['AC-6(2)', 'AC-6(5)', 'AC-2(1)', 'AC-3']
print('mapping assertions passed')
PY
```
