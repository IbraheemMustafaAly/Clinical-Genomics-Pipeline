# 🧬 Clinical Genomics Pipeline — WES Variant Calling

![Pipeline](https://img.shields.io/badge/Pipeline-WES%20Variant%20Calling-blue?style=for-the-badge)
![GATK](https://img.shields.io/badge/GATK-4.6.2.0-orange?style=for-the-badge)
![BWA](https://img.shields.io/badge/BWA-0.7.19-green?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-Linux%20Ubuntu-purple?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen?style=for-the-badge)

> A complete end-to-end **Whole Exome Sequencing (WES)** variant calling pipeline — from raw FASTQ reads to clinical variant interpretation using **GATK4 Best Practices** and **Franklin by Genoox**.

---

## 🗺️ Pipeline Overview

```
FASTQ (Raw Reads)
      │
      ▼
  FastQC / MultiQC          ← Quality Control
      │
      ▼
  Trimmomatic                ← Adapter Trimming & Quality Filtering
      │
      ▼
  BWA-MEM                    ← Alignment to hg38 Reference Genome
      │
      ▼
  SAMtools sort + index      ← BAM Processing
      │
      ▼
  GATK MarkDuplicates        ← Duplicate Marking
      │
      ▼
  GATK HaplotypeCaller       ← Variant Calling (GVCF mode)
      │
      ▼
  GATK GenotypeGVCFs         ← Genotyping
      │
      ▼
  VCF File
      │
      ▼
  Franklin by Genoox         ← Clinical Variant Interpretation (ACMG)
```

---

## 🛠️ Tools & Versions

| Tool | Version | Purpose |
|------|---------|---------|
| FastQC | 0.12.1 | Raw reads quality control |
| MultiQC | 1.19 | Aggregate QC reports |
| Trimmomatic | 0.40 | Adapter trimming |
| BWA-MEM | 0.7.19 | Short-read alignment |
| SAMtools | 1.23.1 | BAM processing |
| GATK4 | 4.6.2.0 | Variant calling |
| BCFtools | 1.23.1 | VCF manipulation |
| Franklin by Genoox | Web | Clinical interpretation |

---

## 📦 Installation

```bash
# Create conda environment
conda create -n genomics -c bioconda -c conda-forge \
    fastqc multiqc trimmomatic bwa samtools gatk4 bcftools sra-tools -y

conda activate genomics
```

---

## 📁 Project Structure

```
clinical-genomics-pipeline/
├── pipeline.sh              # Full automated pipeline script
├── README.md                # This file
├── data/
│   ├── raw/                 # Raw FASTQ files
│   ├── trimmed/             # Trimmed FASTQ files
│   └── ref/                 # Reference genome (chr22)
├── results/
│   ├── qc/                  # FastQC / MultiQC reports
│   ├── aligned/             # BAM files
│   └── variants/            # VCF files
└── docs/
    └── interpretation.md    # Variant interpretation notes
```

---

## 🚀 Usage

### Step 1: Generate Synthetic Test Data

```bash
# Download chr22 reference
wget https://hgdownload.soe.ucsc.edu/goldenPath/hg38/chromosomes/chr22.fa.gz
gunzip chr22.fa.gz

# Index reference
bwa index chr22.fa
samtools faidx chr22.fa
gatk CreateSequenceDictionary -R chr22.fa -O chr22.dict

# Generate synthetic reads (100k reads, 150bp, Illumina PE)
wgsim -N 100000 -1 150 -2 150 -e 0.01 \
    chr22.fa sample_R1.fastq sample_R2.fastq
```

### Step 2: Quality Control

```bash
fastqc sample_R1.fastq sample_R2.fastq -o results/qc/ -t 4
multiqc results/qc/ -o results/qc/multiqc_report
```

### Step 3: Trimming

```bash
trimmomatic PE \
    sample_R1.fastq sample_R2.fastq \
    results/trimmed/R1_paired.fastq.gz results/trimmed/R1_unpaired.fastq.gz \
    results/trimmed/R2_paired.fastq.gz results/trimmed/R2_unpaired.fastq.gz \
    LEADING:3 TRAILING:3 SLIDINGWINDOW:4:15 MINLEN:36
```

### Step 4: Alignment

```bash
bwa mem -t 4 \
    -R "@RG\tID:sample\tSM:sample\tPL:ILLUMINA\tLB:lib1" \
    chr22.fa \
    results/trimmed/R1_paired.fastq.gz \
    results/trimmed/R2_paired.fastq.gz \
    > results/aligned/sample.sam
```

### Step 5: BAM Processing

```bash
samtools sort -@ 4 -o results/aligned/sample_sorted.bam results/aligned/sample.sam
samtools index results/aligned/sample_sorted.bam
rm results/aligned/sample.sam
```

### Step 6: Mark Duplicates

```bash
gatk MarkDuplicates \
    -I results/aligned/sample_sorted.bam \
    -O results/aligned/sample_markdup.bam \
    -M results/aligned/markdup_metrics.txt

samtools index results/aligned/sample_markdup.bam
```

### Step 7: Variant Calling

```bash
gatk HaplotypeCaller \
    -I results/aligned/sample_markdup.bam \
    -R chr22.fa \
    -O results/variants/sample_raw.g.vcf.gz \
    -ERC GVCF
```

### Step 8: Genotyping

```bash
gatk GenotypeGVCFs \
    -R chr22.fa \
    -V results/variants/sample_raw.g.vcf.gz \
    -O results/variants/sample_genotyped.vcf.gz
```

---

## 📊 Results Summary

| Metric | Value |
|--------|-------|
| Total Reads | 200,000 (100K pairs) |
| Mapping Rate | 100% |
| Properly Paired | 99.999% |
| Duplicates | 2 (0.001%) |
| Total SNPs Called | 2,065 |
| Pipeline Runtime | ~10 minutes (i5-4200M, 6GB RAM) |

---

## 🔬 Variant Interpretation

After generating the VCF, variants were uploaded to **Franklin by Genoox** for clinical interpretation:

- **Platform:** [franklin.genoox.com](https://franklin.genoox.com)
- **Analysis Type:** Inherited Disease — Single Case
- **Reference Genome:** hg38
- **Classification Standard:** ACMG/AMP 2015 Guidelines

Franklin provides:
- ACMG variant classification (Pathogenic / Likely Pathogenic / VUS / Benign)
- ClinVar cross-referencing
- gnomAD population frequencies
- OMIM disease associations
- Functional impact predictions

---

## 📚 References

- GATK Best Practices: [https://gatk.broadinstitute.org](https://gatk.broadinstitute.org)
- BWA: Li H. & Durbin R. (2009) *Bioinformatics*
- ACMG Guidelines: Richards et al. (2015) *Genetics in Medicine*
- Franklin by Genoox: [https://franklin.genoox.com](https://franklin.genoox.com)

---

## 👤 Author

**Ibraheem Mustafa**

Bioinformatician & Field Application Specialist — Life Tech Co.

MSc Bioinformatics — University of Sadat City

**Co-Founder — OmicX Labs - Bioinformatics Freelancing Company**


<br/>

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?style=flat&logo=linkedin)](https://linkedin.com/in/ibraheemmustafaaly)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-black?style=flat&logo=github)](https://github.com/IbraheemMustafaAly)
