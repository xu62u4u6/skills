---
name: bulk-rna-seq-analysis
description: Bulk RNA-Seq Analysis Pipeline (GDC)
---

# Bulk RNA-Seq Analysis Pipeline (GDC)

Foundation knowledge for bulk RNA-Seq analysis following the GDC (Genomic Data Commons) mRNA quantification pipeline. Covers alignment, quantification, normalization, and fusion detection.

## When to Use This Skill

- Performing bulk RNA-Seq alignment and quantification
- Running STAR two-pass alignment for RNA-Seq
- Computing gene expression metrics (FPKM, FPKM-UQ, TPM)
- Detecting gene fusions (STAR-Fusion, Arriba)
- Working with GDC harmonized RNA-Seq data
- Understanding RNA-Seq normalization methods
- Debugging RNA-Seq alignment or quantification issues

## Pipeline Overview

```
FASTQ/BAM (raw reads)
    │
    ▼
[FASTQC] ── Pre-alignment QC
    │
    ▼
[STAR Two-Pass Alignment] ── Align to GRCh38
    │
    ├── Genomic BAM (aligned + unaligned reads)
    ├── Transcriptomic BAM (transcript coordinates)
    ├── Chimeric BAM (fusion alignments)
    └── Splice Junction file (SJ.out.tab)
    │
    ▼
[STAR Quantification] ── Raw read counts per gene
    │
    ▼
[Expression Normalization] ── FPKM, FPKM-UQ, TPM
    │
    ▼
[Fusion Detection] ── STAR-Fusion / Arriba
```

**Key Design Decisions:**
- Reference genome: **GRCh38**
- Gene annotation: **GENCODE v36** (as of DR32)
- All RNA-Seq reads treated as **unstranded** for harmonization
- Counting by **STAR** (HTSeq discontinued as of DR32)

## RNA-Seq Alignment Workflow (STAR Two-Pass)

### Concept

STAR two-pass alignment improves splice junction detection:
1. **1st pass:** Align reads, discover novel splice junctions
2. **Intermediate:** Rebuild genome index with discovered junctions
3. **2nd pass:** Re-align reads using the improved junction database

This method follows the ICGC (International Cancer Genome Consortium) approach.

### Output Files

| File | Description |
|------|-------------|
| Genomic BAM | Aligned + unaligned reads in genomic coordinates |
| Transcriptomic BAM | Aligned reads in transcript coordinates (not BAM-sliceable) |
| Chimeric BAM | Reads mapped to different chromosomes/strands (fusions) |
| SJ.out.tab | Splice junction file |

### Step-by-Step Commands

#### Step 1: Build STAR Index

```bash
STAR \
  --runMode genomeGenerate \
  --genomeDir <star_index_path> \
  --genomeFastaFiles <reference.fa> \
  --sjdbOverhang 100 \
  --sjdbGTFfile <gencode.v36.annotation.gtf> \
  --runThreadN 8
```

**Note:** Pre-built indices are available from GDC and do not need to be rebuilt.

#### Step 2: Alignment 1st Pass

```bash
STAR \
  --genomeDir <star_index_path> \
  --readFilesIn <fastq_left_1>,<fastq_left2>,... <fastq_right_1>,<fastq_right_2>,... \
  --runThreadN <runThreadN> \
  --outFilterMultimapScoreRange 1 \
  --outFilterMultimapNmax 20 \
  --outFilterMismatchNmax 10 \
  --alignIntronMax 500000 \
  --alignMatesGapMax 1000000 \
  --sjdbScore 2 \
  --alignSJDBoverhangMin 1 \
  --genomeLoad NoSharedMemory \
  --readFilesCommand <bzcat|cat|zcat> \
  --outFilterMatchNminOverLread 0.33 \
  --outFilterScoreMinOverLread 0.33 \
  --sjdbOverhang 100 \
  --outSAMstrandField intronMotif \
  --outSAMtype None \
  --outSAMmode None
```

**Purpose:** Discover splice junctions only; no BAM output.

#### Step 3: Intermediate Index Generation

```bash
STAR \
  --runMode genomeGenerate \
  --genomeDir <output_path> \
  --genomeFastaFiles <reference.fa> \
  --sjdbOverhang 100 \
  --runThreadN <runThreadN> \
  --sjdbFileChrStartEnd <SJ.out.tab from Step 2>
```

**Purpose:** Rebuild index incorporating novel junctions from 1st pass.

#### Step 4: Alignment 2nd Pass

```bash
STAR \
  --genomeDir <output_path from Step 3> \
  --readFilesIn <fastq_left_1>,<fastq_left2>,... <fastq_right_1>,<fastq_right_2>,... \
  --runThreadN <runThreadN> \
  --outFilterMultimapScoreRange 1 \
  --outFilterMultimapNmax 20 \
  --outFilterMismatchNmax 10 \
  --alignIntronMax 500000 \
  --alignMatesGapMax 1000000 \
  --sjdbScore 2 \
  --alignSJDBoverhangMin 1 \
  --genomeLoad NoSharedMemory \
  --limitBAMsortRAM 0 \
  --readFilesCommand <bzcat|cat|zcat> \
  --outFilterMatchNminOverLread 0.33 \
  --outFilterScoreMinOverLread 0.33 \
  --sjdbOverhang 100 \
  --outSAMstrandField intronMotif \
  --outSAMattributes NH HI NM MD AS XS \
  --outSAMunmapped Within \
  --outSAMtype BAM SortedByCoordinate \
  --outSAMheaderHD @HD VN:1.4 \
  --outSAMattrRGline <formatted RG line>
```

**Purpose:** Final alignment with improved junction database, outputs sorted BAM.

### Key STAR Parameters Explained

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `--outFilterMultimapNmax` | 20 | Max number of loci a read can map to |
| `--outFilterMismatchNmax` | 10 | Max mismatches per read |
| `--alignIntronMax` | 500000 | Max intron size (500 kb) |
| `--alignMatesGapMax` | 1000000 | Max gap between paired mates (1 Mb) |
| `--sjdbOverhang` | 100 | Overhang for splice junction database (ideally = read_length - 1) |
| `--outFilterMatchNminOverLread` | 0.33 | Min matched bases as fraction of read length |
| `--outSAMstrandField` | intronMotif | Add strand info from intron motifs (for unstranded data) |
| `--outSAMunmapped` | Within | Keep unmapped reads in BAM |
| `--twopass1readsN` | -1 | Use all reads in 1st pass (ICGC pipeline mode) |

## mRNA Expression Quantification

### Raw Counts (STAR)

As of DR32, STAR produces gene-level counts concurrently with alignment. Output includes:
- Gene ID
- Unstranded counts
- Stranded counts (forward/reverse)

**Important:** STAR does NOT count reads mapped to more than one different gene. Genes completely encompassed by other genes will likely show zero counts.

### Normalization Methods

#### FPKM (Fragments Per Kilobase of transcript per Million mapped reads)

Controls for transcript length and sequencing depth.

```
FPKM = (read_count * 10^9) / (gene_length * total_mapped_reads)
```

#### FPKM-UQ (Upper Quartile FPKM)

More stable than FPKM; uses the 75th percentile protein-coding gene count instead of total reads.

```
FPKM-UQ = (read_count * 10^9) / (gene_length * 75th_percentile_count * num_protein_coding_genes)
```

- `num_protein_coding_genes` on autosomes: **19,029**
- More robust to highly expressed outlier genes

#### TPM (Transcripts Per Million)

Normalizes for length first, then for sequencing depth. Better for cross-sample comparison than FPKM.

```
TPM = (read_count * 1000 / gene_length) * 1,000,000 / sum_of_length_normalized_counts
```

### Worked Example

**Gene A in Sample 1:**
- Gene length: 3,000 bp
- Reads mapped to Gene A: 1,000
- Total reads mapped to protein-coding regions: 50,000,000
- 75th percentile gene count: 2,000
- Protein-coding genes on autosomes: 19,029
- Sum of length-normalized transcript counts: 9,000,000

| Metric | Formula | Result |
|--------|---------|--------|
| FPKM | 1,000 * 10^9 / (3,000 * 50,000,000) | **6.67** |
| FPKM-UQ | 1,000 * 10^9 / (3,000 * 2,000 * 19,029) | **8.76** |
| TPM | (1,000 * 1,000 / 3,000) * 1,000,000 / 9,000,000 | **37.04** |

### When to Use Which Metric

| Metric | Best For | Limitation |
|--------|----------|------------|
| Raw Counts | Differential expression (DESeq2, edgeR) | Not comparable across samples/genes |
| FPKM | Within-sample gene comparison | Not ideal for cross-sample comparison |
| FPKM-UQ | More stable cross-sample comparison | Still affected by composition bias |
| TPM | Cross-sample comparison | Assumes similar total RNA per cell |

**Best practice:** Use **raw counts** for differential expression analysis; use normalized values (TPM/FPKM) for visualization and reporting.

## Fusion Detection Pipelines

### STAR-Fusion (v1.6)

Processes chimeric junction files from STAR aligner to detect gene fusions.

**Input:** Chimeric junction file from STAR alignment
**Output:** Tab-delimited fusion prediction file

**Output includes:**
- Fused gene names
- Junction read count
- Spanning read count
- Breakpoint information

### Arriba (v1.1.0)

Alternative fusion detection pipeline applied to tumor RNA-Seq data.

**Input:** RNA-Seq aligned reads
**Output:** Gene fusion predictions

## I/O Summary

| Pipeline Step | Input Format | Output Format |
|---------------|-------------|---------------|
| Alignment | FASTQ or BAM | BAM (genomic, transcriptomic, chimeric) |
| Quantification | BAM | TSV (counts + normalized values) |
| Fusion Detection | Chimeric BAM / junction file | TSV |

## Quality Control Checkpoints

1. **Pre-alignment:** FASTQC on raw reads
2. **Post-alignment:** Picard Tools on BAM
3. **Mapping rate:** Expect 70-85% for RNA-Seq
4. **Multi-mapping:** Check `--outFilterMultimapNmax` impact
5. **Gene body coverage:** Assess 3'/5' bias
6. **Strandedness:** GDC treats all as unstranded for harmonization

## GDC Data Portal File Types

| Type | Description | Format |
|------|-------------|--------|
| RNA-Seq Alignment | Aligned reads to GRCh38 (includes unaligned) | BAM |
| STAR Read Counts | Per-gene counts + FPKM/FPKM-UQ/TPM | TSV |
| Gene Fusion | Fusion predictions | TSV |

## Related Skills

- **bioinformatics-fundamentals** - SAM/BAM format, sequencing technologies, filtering patterns
- **scRNA-seq** (future) - Single-cell/single-nuclei RNA-Seq with Cell Ranger + Seurat

## Version History

- **v1.0.0:** Initial release based on GDC mRNA Analysis Pipeline documentation (GENCODE v36, STAR quantification, DR32+)
