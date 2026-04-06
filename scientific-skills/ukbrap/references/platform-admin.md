# UKB RAP Platform Administration

## Account Setup

### Prerequisites
1. UK Biobank AMS account at `ams.ukbiobank.ac.uk`
2. Approval from UK Biobank Access Management Team (AMT)
3. Listed as collaborator on an approved UKB access application

### Account Linking
- RAP account (at `ukbiobank.dnanexus.com`) links 1:1 with AMS account via OAuth redirect
- One-time setup: follow redirect from AMS to RAP

### Project Constraints
- Every project linked to exactly one UKB access application
- Can only share with users on the same application
- File copying between projects requires both on the same application
- Project ownership transfers require recipient to be on the application

### Region
- Hosted in `aws:eu-west-2` (Europe London)

## Two-Factor Authentication

2FA is required for all UKB RAP access. Setup via the RAP login page.

## Billing and Costs

### Initial Credit
- £40 credit on signup, expires after 3 months
- Must add a billing wallet within 1 month or project data is deleted
- Default spending limit: £250 (can be raised via Billing Portal)

### What's Charged
- **Compute**: per-minute charges based on instance type
- **Stored data**: data YOU create (analysis outputs, uploaded files)
- **Egress**: data transferred out of the platform

### What's Free
- UKB-dispensed data storage (sponsored by AWS)
- Cohort Browser usage (browsing, exporting sample IDs)

### Cost Optimization
- Use **Smart Reuse** to avoid recomputing identical steps
- Set job timeouts to prevent runaway costs
- Use spot instances (Low priority) for fault-tolerant jobs
- Archive old analysis outputs
- Monitor usage via Billing Portal

### Job Priority and Pricing
| Priority | Behavior | Cost |
|----------|----------|------|
| High | On-demand instances, immediate | Most expensive |
| Normal | Tries spot 15 min, falls back to on-demand | Default |
| Low | Spot instances only, may be interrupted | Cheapest |

### Rate Card
https://www.dnanexus.com/ratecards/ukbiobank_ratecard_current

### Contacts
- Billing: billing@dnanexus.com
- Platform support: ukbiobank-support@dnanexus.com
- UKB data issues: access@ukbiobank.ac.uk

## Data Updates

### Checking for Updates
Project Settings → "Check for Updates" in UK Biobank section

### Update Process
- **Tabular data**: in-place schema + row changes; new dataset generated, old dataset persists
- **Bulk data**: new files dispensed; `.fam`, `.sample`, `ukb_rel.dat` regenerated (old ones deleted even if copied)
- Database naming: `app<APP-ID>_<TIME>` — each update creates a new dataset with the same database

### Data Release Versions
New data releases may add fields, update existing data, or change file formats. Check the UKB data showcase for release notes.

### Rebase Cohorts After Update
After a data update, run **Rebase Cohorts and Dashboards** app to migrate saved cohorts:
```
https://ukbiobank.dnanexus.com/app/rebase_cohorts_and_dashboards
```

## Participant Withdrawals

### Timeline
1. Email notification from UK Biobank with anonymized EIDs
2. 90-day grace period — project data still accessible
3. After grace period: RAP auto-removes files and databases containing withdrawn participant data

### Your Responsibilities
- Manually remove withdrawn participant records from downloaded or derived files (GWAS results, Table Exporter CSVs, etc.)
- If a newer data version is available, do a data refresh (includes withdrawal processing) instead of standalone withdrawal update

### pVCF Impact
Withdrawn EIDs in pVCF headers become `W000001`, `W000002`, etc.

### Warning
Do not run large jobs or copy operations while withdrawal is in progress.

## UKB RAP-Specific Apps

| App | URL | Purpose |
|-----|-----|---------|
| Table Exporter | `app/table-exporter` | Extract phenotypic fields to CSV/TSV |
| Dataset Extender | `app/dataset-extender` | Ingest new data, create superset Dataset |
| Rebase Cohorts | `app/rebase_cohorts_and_dashboards` | Migrate cohorts between datasets |
| Swiss Army Knife | `app/swiss-army-knife` | bcftools, plink, plink2, bedtools, bgenix, qctool, regenie |
| REGENIE | `app/regenie` | GWAS and burden testing |
| OQFE | `app/oqfe` | WES alignment |
| DeepVariant | `app/pbdeepvariant_ukb` | WES variant calling (GPU) |
| GLnexus | `app/glnexus` | gVCF aggregation → pVCF |
| LocusZoom | `app/locuszoom` | GWAS visualization |
| SAIGE | GRM, sparse GRM, SVAT, GBAT | UKB-specific GWAS configs |
| DRAGEN | `app/dragen_germline_ukb` | Germline variant calling (FPGA) |
| 3D Slicer + MONAI | — | Radiology imaging |
| PHESANT | — | Phenome-wide association |
| Cloud Workstation | `app/cloud_workstation` | Interactive shell |
| ttyd | `app/ttyd` | Web-based terminal |

## JupyterLab

### Two Variants
- Standard: `ukbiobank.dnanexus.com/app/dxjupyterlab` (Python/R)
- Spark: `ukbiobank.dnanexus.com/app/dxjupyterlab_spark_cluster` (includes Hail, VEP, MLlib)

### Troubleshooting
- `502 Bad Gateway` after launch → wait 10-15 minutes; try appending `:8080` or `:8081` to URL
- Cannot open from protected project → set "Delete Access" to "Contributors & Admins"
- `Py4JJavaError: Could not execute broadcast` → upgrade to latest JupyterLab version
- SparkContext double-init → restart kernel

## RStudio

- Backup workspace: `dx-backup-folder` (saves to `/.Backups/<name>.<user>.tar.gz`)
- Restore: `dx-restore-folder /.Backups/filename.tar.gz`
- `/mnt/project` is read-only for streaming large files
- Session state is NOT persistent — upload outputs before terminating
- Closing browser does NOT stop session (charges continue)

## Project Management

### Adding Users
Users must be on the same UKB access application. Invite via Project Settings.

### Permission Levels
- VIEW — read-only
- UPLOAD — can add files
- CONTRIBUTE — can add, modify, run jobs
- ADMINISTER — full control including user management

### Copying Files Between Projects
Both projects must be on the same access application. Use `dx cp` or the web UI.

### Using Metadata
Tag files with properties for organization and search:
```bash
dx set_properties file-xxxx key1=value1 key2=value2
dx find data --property key1=value1
```
