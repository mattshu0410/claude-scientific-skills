# UKB RAP Scientific Workflows

## End-to-End GWAS Pipeline

### Overview

A typical GWAS on UKB RAP follows these steps:
1. Cohort creation (Cohort Browser)
2. Sample QC (JupyterLab)
3. LiftOver if needed (GRCh37 → GRCh38)
4. Array variant QC (Swiss Army Knife / PLINK2)
5. Imputed variant QC (WDL workflow)
6. Association testing (REGENIE app)
7. LD clumping (JupyterLab)
8. Visualization (LocusZoom)

### Step 1: Cohort Creation (Cohort Browser)

1. Click dataset from Manage tab → Cohort Browser
2. Add Filter → search disease/phenotype → select field (e.g. 41270 for ICD10)
3. Select inclusion criteria → name cohort (e.g. "cases")
4. Use "+" → "not in cases" → name "controls"

Cohort Browser is free — no charges for browsing or exporting sample IDs.

### Step 2: Sample QC (JupyterLab)

Instance: `mem1_ssd1_v2_x16`, single node, ~5 minutes

Standard QC filters:
```python
pdf_qced = pdf[
    (pdf['p31'] == pdf['p22001']) &         # sex matches genetic sex
    (pdf['p22006'] == 1) &                   # White British ancestry
    (pdf['p22019'].isnull()) &               # no sex chromosome aneuploidy
    (pdf['p22020'] == 1) &                   # used in PCA (non-relatives)
    (pdf['p22027'].isnull())                  # not het/missing outlier
]
```

Common covariates: sex, age (21022), BMI (23104), ever_smoked (20160), HDL (30760), LDL (30780), PC1-PC10 (22009)

Upload phenotype file:
```bash
dx upload my_phenotype.phe -p --path /output/my_gwas/ --brief
```

### Step 3: LiftOver (GRCh37 → GRCh38)

Required when using array genotype data (GRCh37) with WES/WGS data (GRCh38).

**WDL approach** (~34 hours):
```bash
java -jar dxCompiler-2.10.8.jar compile liftover_plink_beds.wdl \
  -project project-xxxx -folder <directory>
```

Inputs:
- `plink_beds`: `/Bulk/Genotype Results/Genotype calls/*.bed` (22 files)
- `plink_bims`: corresponding `.bim` files
- `plink_fams`: corresponding `.fam` files
- `reference_fastagz`: `/Bulk/Exome sequences/Exome OQFE CRAM files/helper_files/GRCh38_full_analysis_set_plus_decoy_hla.fa`
- `ucsc_chain`: `b37ToHg38.over.chain`

### Step 4: Array Variant QC (Swiss Army Knife)

Instance: `mem1_ssd1_v2_x36`, ~10 minutes

```bash
plink2 --bfile "/mnt/project/<path>/ukb_c1-22_merged" \
  --keep "/mnt/project/<path>/phenotype.phe" \
  --autosome \
  --maf 0.01 \
  --mac 100 \
  --geno 0.1 \
  --hwe 1e-15 \
  --mind 0.1 \
  --write-snplist --write-samples --no-id-header \
  --out array_snps_qc_pass
```

QC thresholds:
- `--maf 0.01` — minor allele frequency > 1%
- `--mac 100` — minor allele count > 100
- `--geno 0.1` — variant missingness < 10%
- `--mind 0.1` — sample missingness < 10%
- `--hwe 1e-15` — HWE p-value > 1e-15

### Step 5: Imputed Variant QC (WDL)

~10 hours at normal priority.

```bash
# Compile WDL
java -jar dxCompiler-2.10.8.jar compile bgens_qc.wdl \
  -project project-xxxx \
  -inputs bgens_qc_input.json \
  -archive -folder '<path>'

# Run
dx run <path>/bgens_qc -f bgens_qc_input.dx.json
```

Inputs:
- `geno_bgen_files`: `/Bulk/Imputation/Imputation from genotype (GEL)/*.bgen` (22 files)
- `geno_sample_files`: corresponding `.sample` files
- `plink2_options`: `--mac 10 --maf 0.0001 --hwe 1e-15 --mind 0.1 --geno 0.1`

### Step 6: GWAS with REGENIE

Instance: `mem1_ssd1_v2_x36`, Step 2 ~7 hours

```bash
dx run app-regenie \
  -iwgr_genotype_bed="<path>/ukb_merged.bed" \
  -iwgr_genotype_bim="<path>/ukb_merged.bim" \
  -iwgr_genotype_fam="<path>/ukb_merged.fam" \
  -igenotype_bgens="/Bulk/Imputation/Imputation from genotype (GEL)/ukb21008_c1_b0_v1.bgen" \
  -igenotype_bgens="...ukb21008_c22_b0_v1.bgen" \
  -igenotype_bgis="...ukb21008_c1_b0_v1.bgen.bgi" \
  -igenotype_samples="...ukb21008_c1_b0_v1.sample" \
  -ipheno_txt="<path>/phenotype.phe" \
  -iquant_traits=False \
  -ipheno_names=case_control \
  -icovar_names=sex,age,pc1,pc2,pc3,pc4,pc5,pc6,pc7,pc8,pc9,pc10 \
  -istep1_block_size=1000 \
  -istep2_block_size=200 \
  -imin_mac=3 \
  -istep1_extract_txt="<path>/array_snps_qc_pass.snplist" \
  -istep2_extract_txt="<path>/imputed_snps_qc_pass.snplist" \
  -istep1_ref_first=1 \
  -icovar_txt="<path>/phenotype.phe" \
  -iuse_firth_approx=True \
  --destination "<output_path>"
```

Key REGENIE parameters:
- `quant_traits`: False for binary, True for quantitative
- `use_firth_approx`: True for binary traits (SPA approximation)
- `step1_ref_first`: 1 for GEL imputed data
- Step 2 is parallelized across chromosomes automatically

### Step 7: LD Clumping (JupyterLab)

Instance: `mem2_ssd1_v2_x32`, ~20 minutes

Uses PLINK to clump significant variants from REGENIE output, extracting per-chromosome from imputed BGEN files.

### Step 8: Visualization

**LocusZoom app** (available in Tools Library):
- Input: REGENIE output `lmm.tsv.gz`
- Generates: Manhattan plots, QQ plots, locus-level plots

---

## Burden Testing with WES

For rare variant gene-level association testing using REGENIE.

### Helper Files

Located at:
```
/Bulk/Exome sequences/Population level exome OQFE variants, PLINK format - final release/helper_files/
```

Three required files:
1. `ukb23158_500k_OQFE.sets.txt.gz` — gene set list (~18,985 genes)
2. `ukb23158_500k_OQFE.annotations.txt.gz` — variant annotations
3. `ukb23158_500k_OQFE.masks` — mask definitions

### Annotation Labels

| Label | Description |
|-------|-------------|
| `LoF` | Loss-of-function |
| `missense(0/5)` | Missense, REVEL score 0/5 |
| `missense(5/5)` | Missense, REVEL score 5/5 |
| `missense(>=1/5)` | Missense, REVEL score ≥1/5 |
| `synonymous` | Synonymous |

Default mask M1 = LoF only.

### Run Burden Test (REGENIE)

Install regenie in JupyterLab:
```bash
wget https://github.com/rgcgithub/regenie/releases/download/v2.2.4/regenie_v2.2.4.gz_x86_64_Linux_mkl.zip
unzip regenie_v2.2.4.gz_x86_64_Linux_mkl.zip
```

Run (example: PCSK9 gene, LDL phenotype):
```bash
./regenie \
  --step 2 \
  --bgen ${genotype_prefix}.bgen \
  --ref-first \
  --sample ${genotype_prefix}.sample \
  --phenoFile $phenotype_file \
  --covarFile $phenotype_file \
  --phenoCol LDL \
  --covarColList Sex,Age,Age_Sq,Age_x_Sex,PC{1:10} \
  --set-list "${helper_path}/ukb23158_500k_OQFE.sets.txt.gz" \
  --anno-file "${helper_path}/ukb23158_500k_OQFE.annotations.txt.gz" \
  --mask-def "${helper_path}/ukb23158_500k_OQFE.masks" \
  --nauto 23 \
  --aaf-bins 0.01,0.001 \
  --bsize 200 \
  --extract-setlist "PCSK9(ENSG00000169174)" \
  --out ldl_pcsk9_test
```

**Important**: In real analyses, always run Step 1 first (`--ignore-pred` skips this — only for testing).

### Custom Masks

```bash
echo "M2 LoF,missense(0/5),missense(5/5),missense(>=1/5)" > custom_masks.txt
```

### Key Options

| Option | Description |
|--------|-------------|
| `--aaf-bins 0.05,0.01` | AAF cutoffs for burden masks |
| `--extract-setlist "GENE"` | Test specific gene(s) |
| `--extract-sets genes.txt` | Test gene list from file |
| `--write-mask` | Save masks to PLINK BED |
| `--build-mask sum` | Sum ALT alleles (vs default max) |
| `--build-mask comphet` | Identify compound heterozygotes |
| `--mask-lovo "GENE,MASK,AAF"` | Leave-One-Variant-Out analysis |

### LOVO (Leave-One-Variant-Out)

Identifies which single variant drives a significant mask signal:
```bash
./regenie \
  --step 2 \
  --bgen ${genotype_prefix}.bgen \
  --ref-first \
  --sample ${genotype_prefix}.sample \
  --phenoFile $phenotype_file \
  --covarFile $phenotype_file \
  --phenoCol LDL \
  --covarColList Sex,Age,Age_Sq,Age_x_Sex,PC{1:10} \
  --set-list "${helper_path}/ukb23158_500k_OQFE.sets.txt.gz" \
  --anno-file "${helper_path}/ukb23158_500k_OQFE.annotations.txt.gz" \
  --mask-def "${helper_path}/ukb23158_500k_OQFE.masks" \
  --nauto 23 \
  --bsize 200 \
  --mask-lovo "PCSK9(ENSG00000169174),M1,0.01" \
  --out ldl_pcsk9_lovo
```

---

## WES OQFE Protocol

The standard pipeline for all UKB WES data (category 170).

### Step 1: Read Alignment (OQFE)

BWA-MEM alt-aware mapping to GRCh38, marks duplicates, retains original quality scores.

```bash
dx run oqfe \
  -ireads_fastqgzs=<first_end_reads> \
  -ireads2_fastqgzs=<second_end_reads> \
  -igenome_fastagz=<genome>.fa.gz \
  -isample=<sample_name>
```

### Step 2: Variant Calling (DeepVariant)

~5 minutes per sample including upload/download.

```bash
dx run pbdeepvariant_ukb \
  -igenome_fastagz=<genome>.fa.gz \
  -iinterval_file=<interval>.bed \
  -imappings_sorted_bam <oqfe_cram>
```

Output: gVCF `.g.vcf.gz` + index

### Step 3: Joint Calling (GLnexus → pVCF)

Generate manifest:
```bash
dx find data --project=<project> --folder=<folder> \
  --name="*.g.vcf.gz" --brief | cut -d\: -f2 > manifest.csv
dx upload --path <project>:/<folder>/ manifest.csv --brief
```

Run GLnexus:
```bash
dx run glnexus -y --brief \
  -ivariants_gvcfgz_manifest_csv=manifest.csv \
  -igenotype_range_bed=calling_regions.bed \
  -ioutput_prefix=output_name \
  --folder=<output_folder>
```

### Step 4: pVCF → PLINK → BGEN Conversion

```bash
# Normalize
bcftools norm -f references.fa -m -any -Oz -o pvcf.norm.vcf.gz pvcf.vcf.gz

# pVCF → PLINK
plink --vcf pvcf.norm.vcf.gz \
  --keep-allele-order --vcf-idspace-to _ --double-id \
  --allow-extra-chr 0 --make-bed --vcf-half-call m \
  --out pvcf.norm

# PLINK → BGEN (zlib)
plink2 --bfile pvcf.norm \
  --export bgen-1.2 bits=8 id-paste=iid ref-first \
  --out pvcf.norm_zlib

# BGEN zlib → zstd (via qctool)
qctool -g pvcf.norm_zlib.bgen -s pvcf.norm_zlib.sample \
  -og pvcf.norm.bgen -os pvcf.norm.sample \
  -ofiletype bgen -bgen-bits 8 -bgen-compression zstd \
  -bgen-omit-sample-identifier-block

# Index
bgenix -g pvcf.norm.bgen -index -clobber
```

### BED Region Preparation (Swiss Army Knife)

```bash
# Add 100bp buffer to targets
bedtools flank -i targets.bed -g references.fa.fai -b 100 > buffers.bed
cat targets.bed buffers.bed | sort -k1,1 -k2,2n > target_buffers.bed
bedtools merge -i target_buffers.bed > calling_regions.bed
```

---

## WES Quality Control: 90pct10dp Filter

Removes sites where <90% of genotypes have read depth DP≥10. Critical for avoiding batch-effect artifacts between Phase 1 (50k) and Phase 2 WES data.

**Apply with bcftools:**
```bash
bcftools norm -m - -f <reference> -Oz -o <normVCF> <inputVCF>
bcftools view -i 'F_PASS(DP>=10 & GT!="mis") > 0.9' -Oz -o <filtered> <normVCF>
```

**Or use precomputed helper file:**
```bash
plink --bfile <original> --out <filtered> \
  --exclude ukb23145_300k_OQFE.90pct10dp_qc_variants.txt \
  --keep-allele-order
```

Impact: ~1.5% SNPs filtered, ~5.2–5.5% indels filtered. Recommend also including a batch covariate (Phase 1 vs Phase 2).

---

## PheWAS (Phenome-Wide Association)

Instance: `mem2_ssd1_v2_x32`, data prep ~20 min, PheWAS ~6 hours

Uses the PheWAS R package with ICD-10 field 41270 as input.

```bash
dx upload get-phewas-data.ipynb --destination <dir>
dx upload run-phewas.ipynb --destination <dir>
```

Key R options: `significance.threshold = c("p-value", "bonferroni", "fdr")`, `additive.genotypes = FALSE`

---

## Instance Type Recommendations

| Task | Instance | Time |
|------|----------|------|
| Sample QC notebook | `mem1_ssd1_v2_x16` | ~5 min |
| Array variant QC | `mem1_ssd1_v2_x36` | ~10 min |
| LiftOver (WDL) | varies | ~34 hrs |
| Imputed variant QC | varies | ~10 hrs |
| REGENIE Step 2 | `mem1_ssd1_v2_x36` | ~7 hrs |
| LD clumping | `mem2_ssd1_v2_x32` | ~20 min |
| PheWAS | `mem2_ssd1_v2_x32` | ~6 hrs |
| Spark JupyterLab | `mem1_hdd1_v2_x4` (2 nodes) | varies |
| DRAGEN variant calling | `mem3_ssd2_dragen_fpga1_x16` | varies |
