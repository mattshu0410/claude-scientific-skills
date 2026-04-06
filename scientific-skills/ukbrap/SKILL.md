---
name: ukbrap
description: UK Biobank Research Analysis Platform (RAP). Access phenotypic/bulk data, Spark queries, Table Exporter, GWAS/PheWAS/burden testing workflows, WES/WGS pipelines, and UKB-specific apps on the DNAnexus-hosted RAP.
license: Unknown
compatibility: Requires a UK Biobank RAP account and approved access application
metadata:
    skill-author: Matthew Shu
---

# UK Biobank Research Analysis Platform (RAP)

## Overview

The UK Biobank RAP is a cloud research environment hosted on DNAnexus (AWS eu-west-2) for analyzing UK Biobank's ~500,000 participant dataset. It provides access to phenotypic (tabular) and bulk (genomic/imaging) data with integrated analysis tools.

This skill covers UKB RAP-specific concepts. For generic DNAnexus platform operations (app development, dxpy SDK, data operations, job execution, configuration), use the `dnanexus-integration` skill.

## When to Use This Skill

- Accessing UK Biobank phenotypic data (field IDs, Spark SQL, Table Exporter)
- Navigating UKB RAP bulk data folder structure (genotypes, WES, WGS, imaging)
- Running GWAS, PheWAS, or burden testing on UKB data
- Working with UKB-specific apps (Swiss Army Knife, REGENIE, OQFE, Table Exporter)
- Querying the dispensed database via JupyterLab or Spark
- Understanding UKB data versioning, participant withdrawals, and EID pseudonymization

## Core Capabilities

### 1. Data Access — Phenotypic (Tabular)

**Purpose**: Query participant-level phenotypic data stored as Spark SQL databases.

**Key Operations**:
- Query via Cohort Browser (GUI), `dx extract_dataset` (CLI), or Spark JupyterLab
- Navigate field IDs and column naming conventions (`p<FIELD>_i<INSTANCE>_a<ARRAY>`)
- Access linked health records (HES, GP, death, COVID, Olink, OMOP)

**Reference**: See `references/data-access.md` for:
- Database/dataset architecture and naming
- Column naming conventions
- All available database tables and required field IDs
- Three approaches to extracting phenotypic data
- Spark initialization and query patterns
- Table Exporter usage

### 2. Data Access — Bulk (Genomic/Imaging Files)

**Purpose**: Navigate and access per-participant and cohort-wide bulk data files.

**Key Operations**:
- Locate files via `/Bulk/` folder structure and field IDs
- Search by EID using file properties
- Access genotype arrays, imputed data, WES, WGS, MRI, proteomics

**Reference**: See `references/data-access.md` for:
- Bulk folder structure and file naming conventions
- Key field IDs and paths for all major data types
- pVCF pseudonymization details
- Proteomics (Olink) data structure

### 3. Scientific Workflows

**Purpose**: Run end-to-end genomic analyses on UKB data.

**Key Workflows**:
- GWAS with REGENIE (array + imputed data)
- Burden testing with WES (REGENIE gene-level tests)
- PheWAS (phenome-wide association)
- WES OQFE pipeline (alignment, variant calling, joint calling)
- Sample and variant QC

**Reference**: See `references/scientific-workflows.md` for:
- Complete GWAS pipeline (sample QC → variant QC → REGENIE → LD clumping)
- Burden testing with annotation files and masks
- WES OQFE protocol (BWA-MEM → DeepVariant → GLnexus)
- Quality control (90pct10dp filter)
- Instance type recommendations per step

### 4. Platform Setup and Administration

**Purpose**: Account setup, billing, project management, and data updates.

**Key Operations**:
- Link AMS and RAP accounts
- Manage billing wallets and spending limits
- Update dispensed data and handle participant withdrawals

**Reference**: See `references/platform-admin.md` for:
- Account and access setup
- Billing and cost management
- Data update procedures
- Participant withdrawal handling
- UKB-specific apps and tools

## Quick Start Examples

### Discover dataset and extract fields (JupyterLab)

```python
import dxpy, subprocess

dispensed_dataset_id = dxpy.find_one_data_object(
    typename='Dataset', name='app*.dataset', folder='/', name_mode='glob')['id']
project_id = dxpy.find_one_project()["id"]
dataset = ':'.join([project_id, dispensed_dataset_id])

# Extract age and sex
cmd = ['dx', 'extract_dataset', dataset,
       '--fields', 'participant.eid,participant.p21022,participant.p31',
       '--delimiter', ',', '--output', 'age_sex.csv']
subprocess.check_call(cmd)
```

### Query GP records via Spark

```python
import pyspark
sc = pyspark.SparkContext(conf=pyspark.SparkConf().setAll([
    ('spark.kryoserializer.buffer.max', '128'),
    ('spark.sql.execution.arrow.pyspark.enabled', 'true')
]))
spark = pyspark.sql.SparkSession(sc)
spark.sql("USE " + dispensed_database_name)
df = spark.sql("SELECT * FROM gp_clinical WHERE read_2 = '44P5.00'")
```

### Search bulk files by participant EID

```bash
dx find data --property eid=1234567
```

### Run variant QC via Swiss Army Knife

```bash
plink2 --bfile "/mnt/project/Bulk/Genotype Results/Genotype calls/ukb22418_c1_b0_v2" \
  --maf 0.01 --mac 100 --geno 0.1 --hwe 1e-15 --mind 0.1 \
  --write-snplist --write-samples --no-id-header \
  --out array_snps_qc_pass
```

## Workflow Decision Tree

1. **Need to access phenotypic (tabular) data?**
   - < 30 fields → `dx extract_dataset` (CLI/JupyterLab)
   - 30+ fields or complex joins → Spark JupyterLab
   - GUI exploration → Cohort Browser + Table Exporter

2. **Need to access bulk genomic files?**
   - Per-participant files → `/Bulk/<Category>/<Field>/` by EID
   - Cohort-wide files → PLINK/BGEN/pVCF in `/Bulk/` subfolders
   - Stream without download → `/mnt/project/Bulk/...`

3. **Running a GWAS?**
   - Sample QC → JupyterLab notebook
   - Variant QC → Swiss Army Knife (PLINK2)
   - Association testing → REGENIE app
   - Visualization → LocusZoom

4. **Running burden/rare-variant tests?**
   - Use REGENIE with UKB helper files (sets, annotations, masks)

5. **Processing raw WES/WGS?**
   - Alignment → OQFE app
   - Variant calling → DeepVariant (pbdeepvariant_ukb)
   - Joint calling → GLnexus

## Key Field IDs Quick Reference

| Field | Description |
|-------|-------------|
| 21022 | Age at recruitment |
| 31 | Sex |
| 22001 | Genetic sex |
| 22006 | Genetic ethnic grouping |
| 22009 | Genetic principal components (PCs 1-40) |
| 22019 | Sex chromosome aneuploidy |
| 22020 | Used in PCA calculation |
| 22021 | Genetic kinship to other participants |
| 22027 | Outliers for heterozygosity or missing rate |
| 41270 | Diagnoses — ICD10 |
| 30900 | Olink protein biomarkers |
| 22418 | Genotype calls (array) |
| 22828 | Imputed genotypes (WTCHG) |
| 23158 | WES OQFE PLINK (500k) |
| 23159 | WES OQFE BGEN (500k) |
| 23193 | WGS CRAM files |

## Resources

- Official docs: https://dnanexus.gitbook.io/uk-biobank-rap
- Rate card: https://www.dnanexus.com/ratecards/ukbiobank_ratecard_current
- Code examples: https://github.com/dnanexus/UKB_RAP/
- UKB data showcase: https://biobank.ndph.ox.ac.uk/showcase/
- RAP support: ukbiobank-support@dnanexus.com
- UKB data issues: access@ukbiobank.ac.uk
