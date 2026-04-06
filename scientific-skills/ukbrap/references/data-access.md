# UKB RAP Data Access

## Phenotypic Data Architecture

### Database and Dataset Naming

Phenotypic data is dispensed as a Parquet-format Spark SQL database (not the old `.enc_ukb` format).

```
Database: app<APPLICATION-ID>_<CREATION-TIME>
Dataset:  app<APPLICATION-ID>_<CREATION-TIME>.dataset
```
Example: `app12345_20210101123456` / `app12345_20210101123456.dataset`

Both live in the root folder (`/`) of the project.

### Column Naming Convention

Participant table columns follow: `p<FIELD-ID>_i<INSTANCE-ID>_a<ARRAY-ID>`

| Pattern | Example | Description |
|---------|---------|-------------|
| `p<FIELD>` | `p21022` | Non-instanced, non-arrayed (age at recruitment) |
| `p<FIELD>_i<N>` | `p53_i0` | Instanced, not arrayed (assessment centre date, visit 0) |
| `p<FIELD>` | `p41270` | Multi-select/embedded array (ICD10 diagnoses, single column) |
| `p<FIELD>_i<N>_a<N>` | `p93_i0_a0` | Both instanced and arrayed |

### EID (Participant ID)

- 7-digit number, range 1,000,000–6,000,000
- Each access application gets a different pseudonymized EID set
- EIDs are consistent across pVCF headers, FAM files, and phenotype columns within the same application
- To extract EID, specify `<entity>.eid` explicitly (e.g. `participant.eid`)

### Database Tables

All tables require specific field approvals:

| Table | Required Field ID |
|-------|-------------------|
| `participant_0001`, `participant_0002`, ... | (auto-split for scalability) |
| `hesin` | #41259 |
| `hesin_critical` | #41290 |
| `hesin_delivery` | #41264 |
| `hesin_diag` | #41234 |
| `hesin_maternity` | #41261 |
| `hesin_oper` | #41149 |
| `hesin_psych` | #41289 |
| `death`, `death_cause` | #40023 |
| `gp_clinical` | #42040 |
| `gp_registrations` | #42038 |
| `gp_scripts` | #42039 |
| `covid19_tpp_gp_clinical` | #40101 |
| `covid19_tpp_gp_scripts` | #40102 |
| `covid19_emis_gp_clinical` | #40103 |
| `covid19_emis_gp_scripts` | #40104 |
| `covid19_result_england/scotland/wales` | #40100 |
| `covid19_vaccination` | #32040 |
| `olink_instance_0`, `olink_instance_2`, `olink_instance_3` | #30900 |
| OMOP tables (15 tables) | #20142 |
| Allele/genotype tables | #23146, #23148, #23157 |

**Important**: Tables appear twice in SQL listings — as a regular name (e.g. `gp_clinical`) and a versioned name (e.g. `gp_clinical_v4_0_9b7a7f3`). The regular name is a VIEW pointing to the versioned table. Always use the regular name; versioned tables are deleted during data updates.

---

## Three Approaches to Extracting Phenotypic Data

### Approach A: Cohort Browser (GUI, small field counts)

1. Open project → click dataset name → Cohort Browser launches
2. Data Preview tab → grid icon → Add Columns
3. Save as a named View
4. Run **Table Exporter** app on that View

Table Exporter options:
- **Coding Option: RAW** — original UKB coded values (e.g. "0" for Female)
- **Header Style: UKB-FORMAT** — headers like `123-4.5` matching original UKB format
- TSV preferred over CSV (many field values contain commas)

### Approach B: `dx extract_dataset` (CLI/JupyterLab, < 30 fields)

**Step 1 — Get data dictionary:**
```bash
dx extract_dataset project-xxxx:record-yyyy -ddd --delimiter ","
```
Generates three CSVs:
- `*.dataset.data_dictionary.csv` — field names and entities
- `*.entity_dictionary.csv`
- `*.codings.csv` — lookup for coded values (ICD-10 etc.)

**Auto-discover dataset in JupyterLab:**
```python
import dxpy, subprocess

dispensed_dataset_id = dxpy.find_one_data_object(
    typename='Dataset', name='app*.dataset', folder='/', name_mode='glob')['id']
project_id = dxpy.find_one_project()["id"]
dataset = ':'.join([project_id, dispensed_dataset_id])

cmd = ["dx", "extract_dataset", dataset, "-ddd", "--delimiter", ","]
subprocess.check_call(cmd)
```

**Step 2 — Collect field names (must be per-entity):**
```python
import glob, os, pandas as pd

data_dict_csv = glob.glob(os.path.join(os.getcwd(), "*.data_dictionary.csv"))[0]
data_dict_df = pd.read_csv(data_dict_csv)

# Example: all olink proteins from a specific entity
field_names = list(data_dict_df.loc[
    data_dict_df["entity"] == "olink_instance_0", "name"].values)
field_names_str = [f"olink_instance_0.{f}" for f in field_names]
field_names_protein = ",".join(field_names_str)
```

**Step 3 — Extract:**
```python
cmd = ['dx', 'extract_dataset', dataset,
       '--fields', field_names_protein,
       '--delimiter', ',', '--output', 'filename.csv']
subprocess.check_call(cmd)
```

**Programmatic with field list file:**
1. Write field names to `field_name.txt` (one per line, same entity only)
2. Upload: `dx upload field_name.txt --destination project-xxxx:/path/`
3. Run Table Exporter with that file + specify entity in Advanced Options

### Approach C: Spark JupyterLab (30+ fields or complex queries)

Launch JupyterLab with **Spark Cluster** configuration (not standard).

**Spark initialization (run exactly once per session):**
```python
import pyspark

config = pyspark.SparkConf().setAll([
    ('spark.kryoserializer.buffer.max', '128'),
    ('spark.sql.execution.arrow.pyspark.enabled', 'true')
])
sc = pyspark.SparkContext(conf=config)
spark = pyspark.sql.SparkSession(sc)
```

**Query tables:**
```python
spark.sql("USE " + dispensed_database_name)
spark.sql("SHOW TABLES").show(truncate=False)

# Example: GP clinical records
df = spark.sql("SELECT * FROM gp_clinical WHERE read_2 = '44P5.00' OR read_3 = '44P5.'")
```

**Extract via SQL dump then Spark:**
```python
cmd = ["dx", "extract_dataset", dataset,
       "--fields", field_names,
       "--delimiter", ",", "--output", "extracted_data.sql", "--sql"]
subprocess.check_call(cmd)

with open('extracted_data.sql', 'r') as file:
    retrieve_sql = file.read()
temp_df = spark.sql(retrieve_sql.strip(";"))
pdf = temp_df.toPandas()
pdf.to_csv('extracted_data.tsv', sep='\t', index=False)
```

**Spark gotchas:**
- Only initialize SparkContext once per session — multiple initializations cause hangs. Restart kernel if this happens.
- Shut down unused notebook kernels before running a second notebook.
- Kryo overflow fix: `('spark.kryoserializer.buffer.max', '128')` in SparkConf
- Broadcast timeout: set `spark.sql.broadcastTimeout` or disable via `spark.sql.autoBroadcastJoinThreshold`
- Spark UI: `https://job-xxxx.dnanexus.cloud:8081/jobs/`

**Common Table Exporter errors:**
- `Out of memory` → increase instance type or use Spark JupyterLab
- `Job aborted (SparkException)` → missing `entity` specification
- `unrecognized arguments: activity` → spaces in output filename (use underscores)
- EID not extracted → add `eid` to field names and specify `entity`

---

## Bulk Data

### Folder Structure

```
/Bulk/<Category>/<Field or group of fields>/
```

Per-participant files are grouped in 2-digit EID-prefix subfolders (e.g. "10", "23").

### File Naming Conventions

**Per-participant:**
```
<EID>_<FIELD-ID>_<INSTANCE-ID>_<ARRAY-ID>.<SUFFIX>
```
Example: `1234567_23193_0_0.cram`

Companion files use the main field's ID:
```
1234567_23193_0_0.cram.crai   # index file
```

**Cohort-wide (PLINK, BGEN, pVCF):**
```
ukb<FIELD-ID>_c<CHROM>_b<BLOCK>_v<VERSION>.<SUFFIX>
```
Example: `ukb23158_c1_b0_v1.bed`

### File Properties (searchable)

- `eid` — participant EID
- `field_id` — data-field ID
- `instance_id` — visit/instance
- `array_id` — array index
- `resource_id` — UK Biobank resource ID (auxiliary files)

**Search by EID:**
```bash
dx find data --property eid=1234567
```

### Key Bulk Data Paths and Field IDs

| Field | Description | Path | Formats |
|-------|-------------|------|---------|
| 22418 | Genotype calls (array) | `/Bulk/Genotype Results/Genotype calls/` | .bed, .bim, .fam |
| 22828 | Imputed genotypes (WTCHG) | `/Bulk/Imputation/UKB imputation from genotype/` | .bgen, .bgi, .sample |
| 21007 | Imputed genotypes (TOPmed) | `/Bulk/Imputation/Imputation from genotype (TOPmed)/` | .bgen, .bgi, .sample |
| 21008 | Imputed genotypes (GEL) | `/Bulk/Imputation/Imputation from genotype (GEL)/` | .bgen, .bgi, .sample |
| 23143 | WES OQFE CRAM | `/Bulk/Exome sequences/Exome OQFE CRAM files/` | .cram |
| 23158 | WES OQFE PLINK (500k) | `/Bulk/Exome sequences/Population level exome OQFE variants, PLINK format - 500k release/` | .bed, .bim, .fam |
| 23159 | WES OQFE BGEN (500k) | `/Bulk/Exome sequences/Population level exome OQFE variants, BGEN format - 500k release/` | .bgen, .bgi, .sample |
| 23157 | WES OQFE pVCF (500k) | `/Bulk/Exome sequences/Population level exome OQFE variants, pVCF format - 500k release/` | .vcf.gz, .tbi |
| 23193 | WGS CRAM files | `/Bulk/Whole genome sequences/Whole genome CRAM files/` | .cram |
| 23196 | WGS GATK pVCF | `/Bulk/Whole genome sequences/Whole genome GATK joint call pVCF/` | .vcf.gz, .tbi |
| 23374 | WGS pVCF (500k) | `/Bulk/Whole genome sequences/Population level WGS variants, pVCF format - 500k release/` | .vcf.gz, .tbi |
| 20209 | Heart MRI short axis | `/Bulk/Heart MRI/Short axis/` | .zip |
| 20216 | Brain MRI T1 | `/Bulk/Brain MRI/T1/` | .zip |
| 30900 | Olink protein biomarkers | `/Bulk/Protein biomarkers/Olink/helper_files/` | .pdf, .dat |

### pVCF Pseudonymization

pVCF sample headers are pseudonymized with application-specific EIDs for fields: 20278, 20279, 23146, 23148, 23156, 23157, 23195, 23196, 23352, 23353, 23354, 23374, 24068, 24304, 24310, 24311, 30108.

Withdrawn participants appear as `W000001`, `W000002`, etc.

### Proteomics Data (Olink, Field #30900)

Raw field 30900 is transformed before dispensing:
1. Protein ID codes replaced with abbreviated names (e.g. code "3" → "aarsd1")
2. Split by instance: `olink_instance_0`, `olink_instance_2`, `olink_instance_3`
3. Pivoted: one column per protein, values are NPX values

Column order: `eid` first, then proteins sorted alphabetically.

Protein field names list: https://github.com/dnanexus/UKB_RAP/blob/main/proteomics/field_names.txt

### Accessing Bulk Data via Swiss Army Knife

**Stream from mounted project (no disk write):**
```bash
plink --bfile "/mnt/project/Bulk/Genotype Results/Genotype calls/ukb22418_c21_b0_v2" \
  --maf 0.1 --out filtered_chr21
```

**Loop across chromosomes:**
```bash
for chr in {1..22}; do
   dx run app-swiss-army-knife --instance-type mem1_ssd1_v2_x8 -y \
   -iin="project-xxxx:/Bulk/Genotype Results/Genotype calls/ukb22418_c${chr}_b0_v2.bed" \
   -icmd="plink --bfile ukb22418_c${chr}_b0_v2 --maf 0.1 --out filtered_chr${chr}"
done
```

**Read from mounted project in R:**
```r
fields <- read.csv("/mnt/project/path/to/file.csv", sep="\t")
```
