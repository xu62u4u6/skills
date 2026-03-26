---
name: adni-cohort
description: Explore the ADNI cohort HDF5 file - check available labels, sample counts, column names, and diagnoses. Use when the user asks about cohort structure, label names, sMCI/pMCI, grade columns, or wants to know what labels are available before running experiments.
allowed-tools: Bash
---

# ADNI Cohort Explorer

Load and inspect the ADNI cohort using pymaftools.

## How to load

```python
import sys; sys.path.insert(0, '.')
from pymaftools.core.Cohort import Cohort

cohort = Cohort.read_hdf5("outputs/cohorts/adni.h5")
# or subset file:
# cohort = Cohort.read_hdf5("outputs/cohorts/adni_subset.h5")
```

## Key attributes

```python
cohort.name                          # cohort name
list(cohort.tables.keys())           # available tables: ['stats', 'pyradiomics', ...]
cohort.sample_metadata               # DataFrame (N samples × M columns)
cohort.sample_metadata.columns       # all column names
cohort.sample_IDs                    # list of sample IDs
```

## Find label columns

```python
df = cohort.sample_metadata
# Grade / diagnosis related columns
for col in df.columns:
    if any(x in col.lower() for x in ['grade', 'mci', 'dx', 'diag', 'label']):
        print(col, sorted(df[col].dropna().astype(str).unique().tolist()))
```

## Common label columns in ADNI

- `grade_baseline` — diagnosis at baseline visit (e.g. CN, MCI, D, sMCI, pMCI)
- `grade` — current visit diagnosis
- `DIAGNOSIS` — raw ADNI diagnosis field

## Filter by diagnosis

```python
# sMCI vs pMCI subset
mask = df['grade_baseline'].isin(['sMCI', 'pMCI'])
subset_ids = df.loc[mask].index.tolist()
print(df.loc[mask, 'grade_baseline'].value_counts())
```

## Project root

All paths relative to: `/home/dingyang/code/mriresearch/ADNI-study/`

Run with venv: `.venv/bin/python`

---

When the user asks about cohort labels or wants to know the exact label strings, run the inspection script above in the ADNI-study directory using `.venv/bin/python`.
