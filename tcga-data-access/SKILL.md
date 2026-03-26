---
name: tcga-data-access
description: TCGA Data Access & Processing
---

# TCGA Data Access & Processing

Comprehensive guide for accessing, downloading, and organizing TCGA (The Cancer Genome Atlas) data through the GDC (Genomic Data Commons) API and related tools.

## When to Use This Skill

- Querying TCGA patient/sample metadata from GDC
- Downloading expression, mutation, CNV, methylation data
- Parsing TCGA barcodes and mapping UUIDs
- Building clinical data tables for survival or association analysis
- Understanding Open vs Controlled access data
- Integrating multi-omics TCGA data
- Filtering samples by cancer type, sample type, or experimental strategy

## TCGA Barcode System

### Overview

TCGA barcode 是 TCGA 專案中生物標本資料的主要識別碼。由 BCR (Biospecimen Core Resource) 統一分配，用於在 TCGA 網路中串接不同資料產生中心 (GCC, GSC, GDAC) 的結果。目前 BCR 同時分配 barcode 和 UUID，UUID 為主要識別碼。

### Barcode Structure

```
TCGA-XX-XXXX-XXX-XXX-XXXX-XX
│    │  │    │   │   │    │
│    │  │    │   │   │    └── Center code (sequencing/characterization center)
│    │  │    │   │   └── Plate ID (4-digit alphanumeric, 96-well plate order)
│    │  │    │   └── Portion (01-99) + Analyte (D=DNA, R=RNA, T=Total RNA, W/X=WGA)
│    │  │    └── Sample (01-29) + Vial (A-Z)
│    │  └── Participant ID (alphanumeric)
│    └── Tissue Source Site (TSS), 2-char (see Code Tables)
└── Project prefix (TCGA)
```

### Barcode Hierarchy (由上到下衍生)

```
TSS              TCGA-02
  └─ Participant TCGA-02-0001
       ├─ Sample      TCGA-02-0001-01
       │    └─ Vial        TCGA-02-0001-01C
       │         └─ Portion     TCGA-02-0001-01C-01
       │              └─ Analyte     TCGA-02-0001-01C-01D
       │                   └─ Plate       TCGA-02-0001-01C-01D-0182
       │                        └─ Aliquot     TCGA-02-0001-01C-01D-0182-01
       ├─ Drug        TCGA-02-0001-C1       (Drug ID: C/D/H/I/T + number)
       ├─ Examination TCGA-02-0001-E3124    (Exam ID: E + number)
       ├─ Radiation   TCGA-02-0001-R2       (Radiation ID: R + number)
       ├─ Surgery     TCGA-02-0001-S145     (Surgery ID: S + number)
       └─ Slide       TCGA-02-0001-01C-01-TS1  (TS=Top, BS=Bottom, MS=Middle)
```

**重要：** 父層 barcode 是子層 barcode 的前綴，反映生物標本的衍生關係。

### Barcode Component Details

| Label | Identifier For | Example Value | Description | Possible Values |
|-------|---------------|---------------|-------------|-----------------|
| Project | Project name | TCGA | TCGA project | TCGA |
| TSS | Tissue source site | 02 | GBM sample from MD Anderson | See Code Tables |
| Participant | Study participant | 0001 | First participant from site | Alphanumeric |
| Sample | Sample type | 01 | Primary solid tumor | 01-09 tumor, 10-19 normal, 20-29 control |
| Vial | Vial order | C | Third vial | A-Z |
| Portion | Portion order | 01 | First 100-120mg portion | 01-99 |
| Analyte | Molecular type | D | DNA sample | D, R, T, W, X (see below) |
| Plate | Plate order | 0182 | 182nd plate | 4-digit alphanumeric |
| Center | Analysis center | 01 | Broad Institute GCC | See Code Tables |

### Sample Type Codes (最重要)

| Code | Definition | Short |
|------|------------|-------|
| 01 | Primary Solid Tumor | TP |
| 02 | Recurrent Solid Tumor | TR |
| 03 | Primary Blood Derived Cancer - Peripheral Blood | TB |
| 05 | Additional - New Primary | TAP |
| 06 | Metastatic | TM |
| 10 | Blood Derived Normal | NB |
| 11 | Solid Tissue Normal | NT |
| 14 | Bone Marrow Normal | NBM |
| 20 | Cell Line Derived Tumor | CELLC |

**常用判斷：**
- `01-09` → 腫瘤樣本
- `10-19` → 正常樣本
- `20-29` → 細胞株/控制樣本

### Analyte Codes

| Code | Analyte Type |
|------|-------------|
| D | DNA |
| R | RNA |
| T | Total RNA |
| W | Whole Genome Amplified DNA (GenomePlex) |
| X | Whole Genome Amplified DNA (alternative) |

**Note:** W 和 X 都指 whole genome amplification 衍生的分析物。

### Special Barcode Types

| Type | Format Example | Note |
|------|---------------|------|
| Shipped Portion | TCGA-CM-5341-01A-21-1933-20 | 僅用於 MDA_RPPA_CORE 平台 |
| Slide | TCGA-02-0001-01C-01-TS1 | TS=Top Slide, BS=Bottom Slide, MS=Middle Slide |
| Drug | TCGA-02-0001-C1 | ID prefix: C, D, H, I, or T |
| Examination | TCGA-02-0001-E3124 | ID prefix: E |
| Radiation | TCGA-02-0001-R2 | ID prefix: R |
| Surgery | TCGA-02-0001-S145 | ID prefix: S |

### Biospecimen Processing Flow

```
Tissue Source Site (TSS)
    → Participant donates tissue
        → Sample collected (assigned sample barcode)
            → Split into Vials
                → Divided into Portions (100-120 mg)
                    → Analytes extracted (DNA/RNA)
                        → Distributed across Plates (96-well)
                            → Each well = Aliquot
                                → Sent to GCC/GSC for sequencing
```

```python
# 從 barcode 解析 sample type
def get_sample_type(barcode: str) -> str:
    parts = barcode.split("-")
    sample_code = int(parts[3][:2])
    if sample_code < 10:
        return "Tumor"
    elif sample_code < 20:
        return "Normal"
    else:
        return "Cell Line"

# 解析完整 barcode 各層級
def parse_barcode(barcode: str) -> dict:
    parts = barcode.split("-")
    result = {"project": parts[0], "tss": parts[1], "participant": parts[2]}
    if len(parts) > 3:
        result["sample_type"] = int(parts[3][:2])
        result["vial"] = parts[3][2:] if len(parts[3]) > 2 else None
    if len(parts) > 4:
        result["portion"] = parts[4][:2]
        result["analyte"] = parts[4][2:] if len(parts[4]) > 2 else None
    if len(parts) > 5:
        result["plate"] = parts[5]
    if len(parts) > 6:
        result["center"] = parts[6]
    return result
```

### Analyte Codes

| Code | Analyte Type |
|------|-------------|
| D | DNA |
| R | RNA |
| T | Total RNA |
| W | Whole Genome Amplified DNA |
| X | GenomePlex amplified DNA |

## GDC API Reference

### Base URLs

```
Cases:     https://api.gdc.cancer.gov/cases
Files:     https://api.gdc.cancer.gov/files
Projects:  https://api.gdc.cancer.gov/projects
Data:      https://api.gdc.cancer.gov/data
```

### Query Patterns

#### 1. Fetch Cases (Patient Clinical Data)

```python
import json, requests

filters = {
    "op": "=",
    "content": {
        "field": "project.project_id",
        "value": "TCGA-LUAD",
    },
}

fields = [
    "submitter_id",
    "project.project_id",
    "demographic.gender",
    "demographic.vital_status",
    "demographic.days_to_death",
    "demographic.age_at_index",
    "diagnoses.ajcc_pathologic_stage",
    "diagnoses.days_to_last_follow_up",
    "diagnoses.primary_diagnosis",
]

params = {
    "filters": json.dumps(filters),
    "fields": ",".join(fields),
    "format": "JSON",
    "size": 500,
    "from": 0,
}

resp = requests.get("https://api.gdc.cancer.gov/cases", params=params)
hits = resp.json()["data"]["hits"]
```

#### 2. Fetch File Metadata (Data Inventory)

```python
filters = {
    "op": "=",
    "content": {
        "field": "cases.project.project_id",
        "value": "TCGA-LUAD",
    },
}

# Use facets for aggregated counts
params = {
    "filters": json.dumps(filters),
    "facets": "data_category,data_type,experimental_strategy",
    "size": 0,
    "format": "JSON",
}

resp = requests.get("https://api.gdc.cancer.gov/files", params=params)
buckets = resp.json()["data"]["aggregations"]
```

#### 3. Download Files by Data Type

```python
filters = {
    "op": "and",
    "content": [
        {"op": "=", "content": {"field": "cases.project.project_id", "value": "TCGA-LUAD"}},
        {"op": "=", "content": {"field": "data_type", "value": "Gene Expression Quantification"}},
        {"op": "=", "content": {"field": "data_format", "value": "TSV"}},
    ],
}

# Get file UUIDs and associated case barcode
fields = "file_id,file_name,cases.submitter_id,cases.samples.sample_type,cases.samples.submitter_id"
params = {
    "filters": json.dumps(filters),
    "fields": fields,
    "size": 1000,
    "format": "JSON",
}

resp = requests.get("https://api.gdc.cancer.gov/files", params=params)
```

#### 4. Download File Content

```python
# Single file
resp = requests.get(f"https://api.gdc.cancer.gov/data/{file_uuid}")

# Multiple files (POST with payload)
payload = {"ids": [uuid1, uuid2, uuid3]}
resp = requests.post("https://api.gdc.cancer.gov/data", json=payload)
# Returns tar.gz containing all files
```

#### 5. Pagination Pattern

```python
def fetch_all(endpoint, filters, fields, page_size=500):
    all_hits = []
    from_idx = 0
    while True:
        params = {
            "filters": json.dumps(filters),
            "fields": ",".join(fields),
            "format": "JSON",
            "size": page_size,
            "from": from_idx,
        }
        resp = requests.get(endpoint, params=params, timeout=60)
        resp.raise_for_status()
        data = resp.json()["data"]
        hits = data["hits"]
        if not hits:
            break
        all_hits.extend(hits)
        from_idx += page_size
        if from_idx >= data["pagination"]["total"]:
            break
    return all_hits
```

### Complex Filters

```python
# AND filter
{"op": "and", "content": [filter1, filter2]}

# OR filter
{"op": "or", "content": [filter1, filter2]}

# IN filter (multiple values)
{"op": "in", "content": {"field": "project.project_id", "value": ["TCGA-LUAD", "TCGA-LUSC"]}}

# Range filter
{"op": ">=", "content": {"field": "demographic.age_at_index", "value": 50}}

# NOT filter
{"op": "not", "content": {"op": "=", "content": {"field": "...", "value": "..."}}}
```

## Data Access Levels

### Open Access (直接下載)

| Data Type | Format | 說明 |
|-----------|--------|------|
| Gene Expression Quantification | TSV | STAR counts + FPKM/TPM |
| miRNA Expression Quantification | TXT | miRNA 表達量 |
| Isoform Expression Quantification | TXT | miRNA isoform |
| Copy Number Segment | TXT | 片段層級 CNV |
| Masked Copy Number Segment | TXT | 遮蔽後 CNV |
| Gene Level Copy Number | TSV | 基因層級 CNV |
| Allele-specific Copy Number | TXT | 等位基因特異 CNV |
| Methylation Beta Value | TXT | DNA 甲基化 beta 值 |
| Masked Intensities (IDAT) | IDAT | 甲基化原始強度 |
| Masked Somatic Mutation | MAF | 遮蔽後體細胞突變 |
| Protein Expression (RPPA) | TSV | 蛋白質表達 |
| Clinical / Biospecimen | XML/Biotab | 臨床與生物標本 |
| Pathology Reports | PDF | 病理報告 |
| Slide Images | SVS | 組織切片影像 |

### Controlled Access (需 dbGaP 申請)

| Data Type | Format | 說明 |
|-----------|--------|------|
| Aligned Reads (BAM) | BAM | 所有定序策略的比對檔 |
| Annotated Somatic Mutation | MAF/VCF | 完整突變註釋 |
| Raw Somatic Mutation | VCF | 原始突變 |
| Transcript Fusion | BEDPE/TSV | 融合基因 |
| Splice Junction Quantification | TSV | 剪接位點定量 |
| Simple Germline Variation | TSV | 種系變異 |

## Clinical Data Fields

### Key Fields for Survival Analysis

```python
# 計算 Overall Survival (OS)
def compute_os(case):
    vital = case["demographic"]["vital_status"]
    if vital == "Dead":
        os_time = case["demographic"]["days_to_death"]
        os_event = 1
    else:
        os_time = case["diagnoses"][0].get("days_to_last_follow_up")
        os_event = 0
    return os_time, os_event
```

### Demographic Fields

| Field | 說明 | Example |
|-------|------|---------|
| `demographic.gender` | 性別 | male, female |
| `demographic.race` | 人種 | white, asian, black or african american |
| `demographic.ethnicity` | 族裔 | hispanic or latino, not hispanic or latino |
| `demographic.vital_status` | 存活狀態 | Alive, Dead |
| `demographic.days_to_death` | 到死亡天數 | 365 |
| `demographic.age_at_index` | 診斷年齡 | 67 |
| `demographic.year_of_birth` | 出生年 | 1945 |

### Diagnosis Fields

| Field | 說明 | Example |
|-------|------|---------|
| `diagnoses.primary_diagnosis` | 主要診斷 | Adenocarcinoma, NOS |
| `diagnoses.ajcc_pathologic_stage` | AJCC 病理分期 | Stage IIA |
| `diagnoses.ajcc_pathologic_t` | T 分期 | T2 |
| `diagnoses.ajcc_pathologic_n` | N 分期 | N0 |
| `diagnoses.ajcc_pathologic_m` | M 分期 | M0 |
| `diagnoses.tumor_stage` | 腫瘤分期 | stage iia |
| `diagnoses.days_to_last_follow_up` | 最後追蹤天數 | 1523 |
| `diagnoses.tissue_or_organ_of_origin` | 原發器官 | Lower lobe, lung |
| `diagnoses.morphology` | ICD-O-3 形態碼 | 8140/3 |

## Common Pitfalls

### 1. UUID vs Barcode 對應

GDC 內部用 UUID，但研究者習慣用 TCGA barcode。下載檔案後需要對應：

```python
# 查詢 file UUID 對應的 TCGA barcode
fields = "file_id,file_name,cases.submitter_id,cases.samples.submitter_id"
```

**注意：** 一個 file UUID 可能對應多個 sample（技術重複）。

### 2. 重複樣本處理

同一個病人可能有：
- 多個 aliquot（技術重複）→ 選最新的或品質最好的
- 多個 sample type（01=腫瘤, 11=正常）→ 要分開處理
- 多筆 diagnosis record → 通常取第一筆

```python
# 去重策略：每個病人只保留一個腫瘤樣本
seen = set()
unique_samples = []
for s in samples:
    patient = "-".join(s["barcode"].split("-")[:3])
    if patient not in seen:
        seen.add(patient)
        unique_samples.append(s)
```

### 3. 正常樣本數量不平衡

| Project | Tumor (01) | Normal (11) | Ratio |
|---------|-----------|-------------|-------|
| TCGA-LUAD | ~535 | ~59 | ~9:1 |
| TCGA-LUSC | ~502 | ~51 | ~10:1 |

**影響：** 差異表達分析統計力下降。考慮補充 GTEx 正常肺組織資料（但需處理批次效應）。

### 4. 臨床資料缺失值

- `days_to_death` 只有 Dead 的病人有值
- `days_to_last_follow_up` 可能為 0 或 null
- `ajcc_pathologic_stage` 部分病人缺失（尤其早期資料）
- 有些欄位回傳 `"Not Reported"` 而非 null

### 5. 表達量檔案整合

GDC 下載的是每個樣本一個 TSV，需合併成矩陣：

```python
import pandas as pd
import os

counts = {}
for fname in os.listdir(download_dir):
    if fname.endswith(".tsv"):
        df = pd.read_csv(os.path.join(download_dir, fname), sep="\t", comment="#")
        sample_id = fname_to_barcode[fname]
        counts[sample_id] = df.set_index("gene_id")["unstranded"]

matrix = pd.DataFrame(counts)
# 過濾前 4 行 metadata (N_unmapped, N_multimapping, etc.)
matrix = matrix.iloc[4:]
```

## Download Methods

### Method 1: GDC API (Python, 推薦小量資料)

適合下載 metadata、expression counts (TSV)、mutation (MAF) 等小檔案。

```python
# 下載單一檔案
resp = requests.get(f"https://api.gdc.cancer.gov/data/{file_uuid}")
with open(output_path, "wb") as f:
    f.write(resp.content)
```

### Method 2: gdc-client (大量/大檔案)

適合下載 BAM 檔或大量檔案，支援斷點續傳。

```bash
# 1. 從 Data Portal 下載 manifest
# 2. 下載（open access 不需 token）
gdc-client download -m manifest.txt -d output_dir/

# Controlled access 需要 token
gdc-client download -m manifest.txt -t gdc_token.txt -d output_dir/
```

### Method 3: TCGAbiolinks (R, 推薦整合分析)

最方便的一站式方案，自動處理 barcode 對應和資料整理。

```r
library(TCGAbiolinks)

# 查詢
query <- GDCquery(
  project = c("TCGA-LUAD", "TCGA-LUSC"),
  data.category = "Transcriptome Profiling",
  data.type = "Gene Expression Quantification",
  workflow.type = "STAR - Counts"
)

# 下載
GDCdownload(query)

# 整理成 SummarizedExperiment 物件
data <- GDCprepare(query)
# data 包含表達矩陣 + 完整臨床資料
```

## LUAD/LUSC Data Summary

### TCGA-LUAD (肺腺癌)

- **Patients:** 585
- **Open data:** ~14,053 files
- **Controlled data:** ~22,171 files
- **Key open data:** Gene Expression (601), miRNA (567), CNV (1,554), Methylation (710), Protein (365), Mutation MAF (618)

### TCGA-LUSC (肺鱗狀細胞癌)

- **Patients:** 504
- **Open data:** ~13,139 files
- **Controlled data:** ~18,926 files
- **Key open data:** Gene Expression (562), miRNA (533), CNV (1,502), Methylation (573), Protein (328), Mutation MAF (549)
- **Extra:** Expression Array (CEL, 135 files)

## Multi-Omics Sample Alignment (跨資料類型樣本對齊)

### 概念

GDC 各 data type 的樣本覆蓋率不同，做 multi-omics 整合分析前必須取交集（inner join），確保每個樣本在所有 data type 都有資料。

### 對齊 Key：`submitter_id`（Case Level）

所有 GDC 檔案都帶有 `cases.submitter_id`（如 `TCGA-44-2655`），這是 **patient-level** 的 ID。用這個做跨 data type 的 set intersection。

**注意：** 不要用 file UUID 或 sample barcode 對齊——同一 patient 可能有多個 sample（tumor + normal）、多個 aliquot。對齊在 case level 做，之後再挑需要的 sample type。

### 對齊流程

```python
import json
import requests

def get_cases_by_datatype(project_id: str, data_type: str, workflow_type: str = None) -> set:
    """查詢某 project + data_type 的所有 case submitter_id"""
    filters = {
        "op": "and",
        "content": [
            {"op": "=", "content": {"field": "cases.project.project_id", "value": project_id}},
            {"op": "=", "content": {"field": "data_type", "value": data_type}},
        ]
    }
    if workflow_type:
        filters["content"].append(
            {"op": "=", "content": {"field": "analysis.workflow_type", "value": workflow_type}}
        )

    params = {
        "filters": json.dumps(filters),
        "fields": "cases.submitter_id",
        "size": 5000,
        "format": "json",
    }

    r = requests.get("https://api.gdc.cancer.gov/files", params=params)
    cases = set()
    for hit in r.json()["data"]["hits"]:
        for c in hit.get("cases", []):
            cases.add(c["submitter_id"])
    return cases


def align_multiomics(project_id: str, data_types: dict) -> set:
    """
    取多個 data type 的 case 交集。

    data_types: {label: {"data_type": str, "workflow_type": str (optional)}}
    Returns: set of aligned submitter_ids
    """
    case_sets = {}
    for label, dt in data_types.items():
        cases = get_cases_by_datatype(project_id, dt["data_type"], dt.get("workflow_type"))
        case_sets[label] = cases
        print(f"  {label}: {len(cases)} cases")

    aligned = set.intersection(*case_sets.values())
    print(f"  >>> Aligned ({len(case_sets)}-way): {len(aligned)} cases")
    return aligned
```

### 使用範例：5-way 對齊

```python
DATA_TYPES = {
    "Expression": {"data_type": "Gene Expression Quantification", "workflow_type": "STAR - Counts"},
    "Mutation":   {"data_type": "Masked Somatic Mutation"},
    "CNV":        {"data_type": "Masked Copy Number Segment"},
    "Methylation":{"data_type": "Methylation Beta Value"},
    "Clinical":   {"data_type": "Clinical Supplement"},
}

luad_aligned = align_multiomics("TCGA-LUAD", DATA_TYPES)  # ~506
lusc_aligned = align_multiomics("TCGA-LUSC", DATA_TYPES)  # ~486
all_aligned = luad_aligned | lusc_aligned                   # ~992
```

### 對齊後下載策略

對齊完成後，用 aligned case IDs 過濾下載：

```python
def get_files_for_aligned_cases(project_id: str, data_type: str, aligned_cases: set,
                                 workflow_type: str = None) -> list:
    """只下載對齊樣本的檔案"""
    filters = {
        "op": "and",
        "content": [
            {"op": "=",  "content": {"field": "cases.project.project_id", "value": project_id}},
            {"op": "=",  "content": {"field": "data_type", "value": data_type}},
            {"op": "in", "content": {"field": "cases.submitter_id", "value": list(aligned_cases)}},
        ]
    }
    if workflow_type:
        filters["content"].append(
            {"op": "=", "content": {"field": "analysis.workflow_type", "value": workflow_type}}
        )

    params = {
        "filters": json.dumps(filters),
        "fields": "file_id,file_name,cases.submitter_id,cases.samples.sample_type",
        "size": 5000,
        "format": "json",
    }
    r = requests.get("https://api.gdc.cancer.gov/files", params=params)
    return r.json()["data"]["hits"]
```

### 注意事項

1. **Sample type 過濾：** 對齊在 case level，但下載時要注意只取 Primary Tumor (01) 或根據需求篩選
2. **重複 aliquot：** 同一 case 同一 data type 可能有多個檔案（技術重複），需去重（通常取最新的 plate）
3. **Tumor purity：** 對齊後可進一步用 ESTIMATE/ABSOLUTE 過濾低 purity 樣本
4. **批次效應：** LUAD + LUSC 合併分析時需注意 batch effect，建議用 ComBat 或在模型中加入 project 作為 covariate

## Downloaded File Formats (gdc-client)

gdc-client 下載後的目錄結構：每個檔案放在 `{file_uuid}/` 子目錄裡。

```
data/raw/{data_type}/
├── {file_uuid_1}/
│   ├── {filename}
│   ├── annotations.txt
│   └── logs/
├── {file_uuid_2}/
│   └── ...
```

### 1. Gene Expression (STAR Counts)

**檔名格式：** `{uuid}.rna_seq.augmented_star_gene_counts.tsv`

```tsv
# gene-model: GENCODE v36
gene_id	gene_name	gene_type	unstranded	stranded_first	stranded_second	tpm_unstranded	fpkm_unstranded	fpkm_uq_unstranded
N_unmapped			2389002	2389002	2389002
N_multimapping			6317951	6317951	6317951
N_noFeature			3275231	36621386	36861817
N_ambiguous			6711313	1703524	1695291
ENSG00000000003.15	TSPAN6	protein_coding	4383	2223	2161	54.7095	15.6603	15.1026
```

**注意：**
- 前 1 行 `#` 開頭為 comment
- 前 4 筆資料行（N_unmapped, N_multimapping, N_noFeature, N_ambiguous）是 QC metadata，不是基因
- `gene_id` 帶版本號（如 `.15`），需決定是否去除
- 常用欄位：`unstranded`（raw counts）、`tpm_unstranded`（TPM）
- 每樣本一檔，需合併成 gene × sample 矩陣

### 2. Somatic Mutation (Masked MAF)

**檔名格式：** `{uuid}.wxs.aliquot_ensemble_masked.maf.gz`（gzip 壓縮）

```
#version gdc-1.0.0
#annotation.spec gdc-2.0.0-aliquot-merged-masked
#contigs chr1,chr2,...,chrX,chrY,chrM
#sort.order BarcodesAndCoordinate
#filedate 20220517
#normal.aliquot {uuid}
Hugo_Symbol  Entrez_Gene_Id  Center  NCBI_Build  Chromosome  Start_Position  End_Position  ...
TP53         7157            BCM     GRCh38      chr17       7675088         7675088       ...
```

**注意：**
- `#` 開頭行為 header metadata
- 標準 MAF 格式，可直接用 `pymaftools.MAF.read_maf()` 讀取
- 每個 case 一個 MAF 檔，需用 `MAF.merge_mutations()` 合併
- `Tumor_Sample_Barcode` 欄位包含完整 TCGA aliquot barcode

### 3. Copy Number (Masked Segment)

**檔名格式：** `{sample_name}.nocnv_grch38.seg.v2.txt`

```tsv
GDC_Aliquot	Chromosome	Start	End	Num_Probes	Segment_Mean
47ed8f8d-...	1	3301765	49242229	24819	-0.277
47ed8f8d-...	1	49249600	49536318	93	-0.5299
```

**注意：**
- `GDC_Aliquot` 是 UUID 不是 TCGA barcode，需透過 manifest 或 API 對應回 case_id
- `Segment_Mean` 是 log2(copy_number / 2)，0 = diploid, >0 = gain, <0 = loss
- Segment-level 資料，需轉換為 gene-level（mapping 到基因座標）或 arm-level
- 每 case 可能有多個 SEG 檔（tumor + normal）

### 4. DNA Methylation (450K Beta Value)

**檔名格式：** `{uuid}.methylation_array.sesame.level3betas.txt`

```
cg00000029	0.153012679159177
cg00000108	0.975289051698828
cg00000109	0.930274402814806
```

**注意：**
- **無 header**，兩欄：probe_id (cg*) + beta_value (0~1)
- Tab 分隔
- ~480K probes per sample，每檔約 13 MB
- 需過濾 cross-reactive probes 和 SNP-associated probes
- 合併後矩陣非常大（480K × N_samples），建議用稀疏格式或分批處理

### 5. Clinical (BCR XML)

**檔名格式：** `nationwidechildrens.org_clinical.TCGA-XX-XXXX.xml`（LUAD）或 `nationwidechildrens.org_biospecimen.*.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<luad:tcga_bcr ...>
  <admin:admin>...</admin:admin>
  <luad:patient>
    <clin_shared:tumor_tissue_site>Lung</clin_shared:tumor_tissue_site>
    <clin_shared:gender>MALE</clin_shared:gender>
    <clin_shared:vital_status>Dead</clin_shared:vital_status>
    <clin_shared:days_to_death>365</clin_shared:days_to_death>
    ...
  </luad:patient>
</luad:tcga_bcr>
```

**注意：**
- XML 有命名空間前綴（`clin_shared:`, `luad:`, `shared:` 等），解析需處理 namespace
- LUAD 和 LUSC 的 XML schema 不同（root tag 分別是 `luad:tcga_bcr` 和 `lusc:tcga_bcr`）
- 每個 case 有 clinical + biospecimen 兩種 XML
- **替代方案：** 用 GDC API `cases` endpoint 直接拉結構化 clinical 表格更方便

## Existing Scripts

以下腳本位於 `~/`：
- `fetch_tcga_lung_patients.py` — 抓取 LUAD/LUSC 所有病患臨床資料，輸出 `tcga_lung_patients.tsv`
- `fetch_tcga_lung_data_types.py` — 查詢 LUAD/LUSC 所有可用資料類型與檔案數

## Related Skills

- **bulk-rna-seq-analysis** — GDC RNA-Seq alignment 和 quantification pipeline 詳細流程
- **bioinformatics-fundamentals** — SAM/BAM 格式、定序技術、過濾模式

## Version History

- **v1.0.0:** Initial release covering GDC API patterns, TCGA barcode parsing, clinical data fields, access levels, download methods, and LUAD/LUSC data inventory
- **v1.1.0:** Added multi-omics sample alignment section — cross data-type case intersection workflow, aligned download strategy, and deduplication notes
- **v1.2.0:** Added downloaded file formats section — detailed format specs for all 5 data types (expression TSV, MAF.gz, SEG, methylation beta, clinical XML) with gdc-client directory structure
