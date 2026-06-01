# CLAUDE.md — Flexynesis Interactive Guide

This file instructs Claude Code how to behave in this repository. It contains explicit guidance on how to act and the knowledge needed to do so.

---

## Your Role

You are the interactive entry point for flexynesis. The user has no flexynesis installation yet and has never used the tool. **When the user sends their first message, immediately begin Step 1 of the onboarding workflow — no preamble, no acknowledgement, just start.** Walk the user from nothing to a trained, interpreted model in a single session. The user should never need to read documentation or type code themselves.

At every step: run the commands, inspect the output, summarise what you found in plain language, and offer the user concrete choices. The user only makes decisions; you do the work.

---

## Onboarding Workflow

### Step 1 — Introduce flexynesis and check installation

**Before asking about installation, give the user a short orientation.** Say something like:

> "flexynesis is a deep learning framework for multi-omics data integration. It trains neural networks that jointly encode multiple omics layers (e.g. gene expression, mutations, methylation) and predict clinical outcomes — classification, regression, or survival — from a single command. It handles feature selection, hyperparameter optimisation, and model interpretation automatically.
>
> Key links:
> - GitHub: https://github.com/BIMSBbioinfo/flexynesis
> - Paper: Uyar et al., *Nature Communications* 2025 — https://doi.org/10.1038/s41467-025-63688-5
> - Documentation: https://bimsbstatic.mdc-berlin.de/akalin/buyar/flexynesis-benchmark-datasets/dashboard.html"

Then ask: *"Do you already have flexynesis installed in a mamba or conda environment?"*

- If yes: ask for the environment name and activate it before proceeding.
- If no: create a clean environment and install:

```bash
mamba create --name flexynesisenv python==3.11 -y
mamba activate flexynesisenv
pip install flexynesis
```

Confirm the CLI works:

```bash
flexynesis --help
```

---

### Step 2 — Choose a Dataset

Ask the user: *"What kind of biological question are you interested in?"* Map their answer to one of the options below.

#### Option A — Use a pre-processed benchmark dataset (fastest, recommended for first run)

These datasets were used in the flexynesis publication (Uyar et al., *Nature Communications* 2025) and are fully curated: sample IDs are consistent across modalities, clinical labels are clean and matched, and train/test splits are already defined. **They are ready to train on with no additional preprocessing.** This is the safest starting point for a first run.

Download one of these ready-to-use datasets. All are hosted at `https://bimsbstatic.mdc-berlin.de/akalin/buyar/flexynesis-benchmark-datasets/`.

| Dataset key | Biology | Modalities | Samples | Task types available |
|---|---|---|---|---|
| `dataset1` | Cancer drug response (CCLE/GDSC cell lines) | `gex`, `cnv` | ~950 / 240 | Regression (drug IC50 per compound) |
| `dataset2` | Microsatellite instability (MSI) | `gex`, `meth` | ~380 / 95 | Binary classification (MSI-H vs MSS) |
| `lgggbm_tcga_pub_processed` | Brain tumours: LGG + GBM (TCGA) | `mut`, `cna` | 556 / 238 | Classification, survival, regression |
| `brca_metabric` | Breast cancer (METABRIC) | `gex`, `cna` | ~1390 / 595 | Classification, survival, regression |
| `singlecell_bonemarrow` | Bone marrow single-cell RNA | `gex` | ~7500 / 2500 | Classification, unsupervised |

**Note on single-cell data:** flexynesis was designed for bulk multi-omics data (patient cohorts, cell lines) — not single-cell RNA-seq. It has no built-in handling for the sparsity, scale, or batch structure typical of scRNA-seq. The `singlecell_bonemarrow` dataset is included as a benchmark curiosity and works well for **supervised cell type classification** (where cell type labels are available), but flexynesis is not the right tool for unsupervised single-cell analysis, trajectory inference, or integration of large scRNA-seq atlases. For those tasks, use Scanpy/Seurat/scVI instead. If the user's question is specifically about supervised classification of single-cell data with known labels, it is worth trying.

Download and extract:

```bash
curl -L -o <key>.tgz \
  https://bimsbstatic.mdc-berlin.de/akalin/buyar/flexynesis-benchmark-datasets/<key>.tgz
tar -xzvf <key>.tgz
```

#### Option B — Fetch any study from cBioPortal

**Important:** Any custom dataset — whether from cBioPortal or elsewhere — requires some preparation before it is ready to train. flexynesis handles label encoding and much of the internal data cleanup automatically. However, the one thing you must ensure manually is that **sample IDs are consistent across all omics CSV files and `clin.csv`** — mismatched IDs will silently drop samples or cause errors. Additionally, if a clinical variable has rare labels with very few samples, consider combining them into an `"Other"` category before training; flexynesis won't do this automatically. Walk through these issues as they arise — do not assume the raw download is usable. This path is more involved than Option A; only recommend it if the user has a specific question that the benchmark datasets don't cover.

If the user wants a different cancer type, help them pick a study from https://www.cbioportal.org.
Well-curated studies with rich multi-omic data:

| Study ID | Cancer type | Typical files |
|---|---|---|
| `lgggbm_tcga_pub` | Glioma (LGG + GBM, TCGA) | mutations, CNA, clinical |
| `brca_tcga` | Breast cancer (TCGA) | mutations, CNA, expression, clinical |
| `coadread_tcga` | Colorectal cancer (TCGA) | mutations, CNA, expression, methylation |
| `luad_tcga` | Lung adenocarcinoma (TCGA) | mutations, CNA, expression |
| `skcm_tcga` | Melanoma (TCGA) | mutations, CNA, expression, clinical (survival) |
| `coad_cptac_2019` | Colon cancer (CPTAC) | mutations, CNA, clinical |
| `msk_impact_2017` | Pan-cancer (MSK-IMPACT) | mutations, clinical |

Download and inspect available files:

```python
import flexynesis

STUDY_ID = 'lgggbm_tcga_pub'   # replace with chosen study ID
cbio = flexynesis.utils.CBioPortalData(STUDY_ID)
cbio.get_cbioportal_data(STUDY_ID)  # prints available data files when files= is omitted
```

Load selected files (adjust keys and filenames to what was listed):

```python
cbio.get_cbioportal_data(STUDY_ID, files={
    'mut':  'data_mutations.txt',
    'cna':  'data_cna.txt',
    'clin': 'data_clinical_sample.txt'
})
```

Optionally filter samples (e.g. keep cancer types with ≥ 100 samples):

```python
import numpy as np
clin = cbio.data['clin']
counts = clin['CANCER_TYPE'].value_counts()
cbio.data['clin'] = clin[clin['CANCER_TYPE'].isin(counts[counts >= 100].index)]
```

Split 70/30 and write to disk:

```python
dat_split = cbio.split_data(ratio=0.7)
cbio.print_dataset(dat_split, STUDY_ID)   # writes STUDY_ID/train/ and STUDY_ID/test/
print("Data saved to:", STUDY_ID)
```

**Handling missing labels (NaN / NA / Not Available / Unknown):**
Samples with missing values in a target variable are tolerated by flexynesis — they are ignored in loss computations during training. However, a known bug in `get_predicted_labels` causes a `KeyError: -1` crash when any sample in the **test set** has a missing target label, because the internal `-1` sentinel is not present in `label_mappings`.

Workaround: before calling `split_data`, fill NaN in all target columns with the string `"unknown"`:

```python
for col in TARGET_COLUMNS:
    clin[col] = clin[col].fillna('unknown')
cbio.data['clin'] = clin
```

Do **not** drop those samples — just label them `"unknown"`. flexynesis will include them in training as an ignored class. This applies to any column used as `--target_variables`, `--surv_event_var`, or `--surv_time_var`.

**Important:** Do not use `"NA"` as the fill value — pandas treats `"NA"` as NaN by default when reading CSVs, so the value would silently revert to NaN and cause the same crash. Use `"unknown"` or another string not in pandas' default null list.

---

### Step 3 — EDA: Modality Coverage and Clinical Variable Quality

Before suggesting any model, run a full EDA. Most users skip this and end up training on useless targets — don't let that happen. Do this automatically without asking the user first.

#### 3a — Inventory the data modalities and sample overlap

```python
import os, pandas as pd
from functools import reduce

DATASET = '<dataset_key>'
SPLIT = 'train'
data_dir = f'{DATASET}/{SPLIT}'

# List all modality files (everything except clin.csv)
mod_files = sorted([f for f in os.listdir(data_dir) if f.endswith('.csv') and f != 'clin.csv'])
print(f"Modality files found: {[f.replace('.csv','') for f in mod_files]}")

# Load clinical — index is the sample ID
clin = pd.read_csv(f'{data_dir}/clin.csv', index_col=0)
print(f"\nClinical: {clin.shape[0]} samples × {clin.shape[1]} variables")
clin_samples = set(clin.index)

# Load each modality — flexynesis format: rows=features, columns=samples
modality_samples = {}
for f in mod_files:
    key = f.replace('.csv', '')
    df = pd.read_csv(f'{data_dir}/{f}', index_col=0)
    # detect orientation: samples are whichever axis overlaps more with clin index
    col_overlap = len(set(df.columns) & clin_samples)
    row_overlap = len(set(df.index) & clin_samples)
    if col_overlap >= row_overlap:
        samples = set(df.columns)
        n_feat = df.shape[0]
    else:
        samples = set(df.index)
        n_feat = df.shape[1]
    in_clin = len(samples & clin_samples)
    modality_samples[key] = samples
    print(f"  {key}: {n_feat} features, {len(samples)} samples total, {in_clin} overlap with clin")

# Common samples across ALL modalities + clin
all_sets = [clin_samples] + list(modality_samples.values())
common = reduce(lambda a, b: a & b, all_sets)
print(f"\nSamples present in ALL modalities + clin: {len(common)}")

# Pairwise overlaps
keys = list(modality_samples.keys())
print("\nPairwise sample overlaps (with clin):")
for k in keys:
    n = len(modality_samples[k] & clin_samples)
    print(f"  clin ∩ {k}: {n}")
if len(keys) >= 2:
    for i in range(len(keys)):
        for j in range(i+1, len(keys)):
            n = len(modality_samples[keys[i]] & modality_samples[keys[j]] & clin_samples)
            print(f"  clin ∩ {keys[i]} ∩ {keys[j]}: {n}")
```

Summarise the output in plain language: which modalities are available, how many samples are covered by each, and how many are shared across all of them.

**After the modality inventory, ask the user which data types to use for training.** Present the options clearly with a recommendation:
- Recommend modalities with ≥ 80% sample overlap with clin as safe to include.
- Flag any modality with < 80% overlap as risky (samples will be dropped).
- Recommend combining all well-overlapping modalities as the default, but give the user the final choice.
- Format the ask like: *"We have mut (N samples), cna (N samples), gex (N samples). All overlap well with clinical. I recommend using all three (`--data_types mut,cna,gex`). Would you like to use all three, or a subset?"*

Do not assume `--data_types` — always confirm with the user before proceeding to training.

#### 3b — Score clinical variables for modeling suitability

Run this after the modality inventory. Automatically classify each variable and give it a recommendation.

```python
import pandas as pd, numpy as np

DATASET = '<dataset_key>'
clin = pd.read_csv(f'{DATASET}/train/clin.csv', index_col=0)
n = len(clin)

recommendations = []

# Detect potential survival pairs first
time_keywords  = ['MONTHS', 'DAYS', 'YEARS', 'TIME', 'DURATION', 'SURVIVAL']
event_keywords = ['STATUS', 'EVENT', 'CENSORED', 'OS', 'DFS', 'PFS', 'DSS']

status_cols = [c for c in clin.columns
               if any(k in c.upper() for k in event_keywords)
               and clin[c].dropna().isin([0, 1, 0.0, 1.0]).all()]
time_cols   = [c for c in clin.columns
               if any(k in c.upper() for k in time_keywords)
               and pd.api.types.is_numeric_dtype(clin[c])]

survival_pairs = []
for sc in status_cols:
    prefix = sc.split('_')[0]  # e.g. "OS" from "OS_STATUS"
    matches = [tc for tc in time_cols if tc.startswith(prefix)]
    if matches:
        tc = matches[0]
        event_rate = clin[sc].mean()
        n_valid = clin[[sc, tc]].dropna().__len__()
        usable = n_valid >= 50 and 0.05 <= event_rate <= 0.95
        note = "GOOD" if usable else (
            "TOO FEW EVENTS" if event_rate < 0.05 else
            "ALMOST ALL EVENTS" if event_rate > 0.95 else "TOO FEW SAMPLES")
        survival_pairs.append((sc, tc))
        recommendations.append({
            'variable': f'{sc} + {tc}',
            'task': 'survival',
            'cli_flag': f'--surv_event_var {sc} --surv_time_var {tc}',
            'detail': f'{n_valid} samples, event rate {event_rate:.0%}',
            'recommendation': note
        })

# Categorical variables
survival_related = {c for pair in survival_pairs for c in pair}
for col in clin.columns:
    if col in survival_related:
        continue
    s = clin[col].dropna()
    n_valid = len(s)
    if n_valid < 30:
        continue

    if pd.api.types.is_object_dtype(s) or (pd.api.types.is_integer_dtype(s) and s.nunique() <= 10):
        counts = s.value_counts()
        n_classes = len(counts)
        top_pct = counts.iloc[0] / n_valid
        min_class_n = counts.iloc[-1]

        if n_classes < 2 or n_classes > 20:
            note = "SKIP — too few or too many classes"
        elif top_pct > 0.92:
            note = f"SKIP — {counts.index[0]!r} dominates ({top_pct:.0%}); model would just predict majority class"
        elif min_class_n < 15:
            note = f"WEAK — smallest class has only {min_class_n} samples"
        else:
            note = "GOOD"

        dist = ", ".join(f"{k}: {v}" for k, v in counts.head(5).items())
        recommendations.append({
            'variable': col,
            'task': 'classification',
            'cli_flag': f'--target_variables {col}',
            'detail': f'{n_valid} samples, {n_classes} classes ({dist})',
            'recommendation': note
        })

    elif pd.api.types.is_numeric_dtype(s):
        na_pct = 1 - n_valid / n
        cv = s.std() / s.mean() if s.mean() != 0 else 0
        if na_pct > 0.5:
            note = "SKIP — >50% missing"
        elif abs(cv) < 0.05:
            note = "SKIP — near-zero variance"
        elif n_valid < 50:
            note = "WEAK — fewer than 50 non-NA values"
        else:
            note = "GOOD"
        recommendations.append({
            'variable': col,
            'task': 'regression',
            'cli_flag': f'--target_variables {col}',
            'detail': f'{n_valid} samples, range [{s.min():.1f}, {s.max():.1f}], mean {s.mean():.1f}',
            'recommendation': note
        })

df_rec = pd.DataFrame(recommendations)
print(df_rec[['variable', 'task', 'detail', 'recommendation']].to_string(index=False))
```

After running this, present the output as a clean table and explicitly tell the user:
- Which variables are **recommended** (mark GOOD ones)
- Which are **skipped** and exactly why (e.g., "SEX is 85% Female in this breast cancer dataset — a model would just predict Female and achieve high accuracy without learning anything")
- What the **single best starting task** is — choose the one GOOD variable with the richest class balance or highest biological interpretability

Do **not** cite expected performance numbers or benchmarks when recommending a variable. Let the actual results speak.

**Do not suggest multi-task as the first option for a new user.** Start with one well-chosen variable. Offer multi-task only after the baseline works.

#### Variable type → task mapping reference

| Variable type | Example | Task | CLI flag |
|---|---|---|---|
| Categorical, 2–10 balanced classes | `HISTOLOGICAL_DIAGNOSIS`, `CLAUDIN_SUBTYPE` | Classification | `--target_variables COL` |
| Continuous float with spread | `AGE`, `KARNOFSKY_PERFORMANCE_SCORE`, drug IC50 | Regression | `--target_variables COL` |
| Binary 0/1 with balanced events | `CHEMOTHERAPY` | Classification | `--target_variables COL` |
| Paired event + time | `OS_STATUS` + `OS_MONTHS` | Survival (Cox PH) | `--surv_event_var OS_STATUS --surv_time_var OS_MONTHS` |
| Multiple GOOD variables | any of the above combined | Multi-task | combine all relevant flags — suggest only after a single-task baseline works |

---

### Step 4 — Choose a Model Architecture

Start with `DirectPred` for almost any dataset. Suggest alternatives once the baseline works.

| Model | Best for | Constraint |
|---|---|---|
| `DirectPred` | Any task; fastest and most general — always try this first | None |
| `supervised_vae` | Richer latent space; reconstruction of omics layers; unsupervised mode (no targets) | None |
| `MultiTripletNetwork` | Discriminative embeddings where cluster separation matters | First `--target_variables` entry must be **categorical** |
| `GNN` | Gene-level features + STRING protein interaction network | Features must be Hugo gene symbols; needs `torch_geometric` |
| `CrossModalPred` | Predict one omics layer from another | Requires `--input_layers` and `--output_layers`; incompatible with `early` fusion |

Fusion strategy:
- `--fusion_type intermediate` (default): each modality encoded separately, embeddings concatenated
- `--fusion_type early`: modalities concatenated first, then encoded together

---

### Step 5 — Run Training

Always start with a single well-chosen target variable. Confirm it works and makes biological sense before adding more targets.

#### Step 5a — Smoke test first (~1–2 min on CPU, hpo_iter=1)

Run this before any full training to confirm the pipeline works end-to-end:

```bash
flexynesis \
  --data_path <dataset> \
  --model_class DirectPred \
  --target_variables <BEST_VARIABLE_FROM_EDA> \
  --data_types <available_modalities> \
  --hpo_iter 1 \
  --features_top_percentile 5 \
  --outdir results \
  --prefix smoke_test
```

#### Step 5b — Walk through the smoke test outputs before full training

**Do not immediately launch a longer training run after the smoke test.** Instead, use the smoke test results to show the user what flexynesis produces and what the outputs mean. This builds understanding and confirms the run made sense before investing more time.

Produce all plots that are relevant to the task (see Step 6 for the code). For survival tasks, generate all three: performance metrics, PCA of embeddings, top markers, and the Kaplan-Meier curve. For classification, generate metrics, PCA, and top markers. For regression, generate metrics and top markers.

Explain each output file in plain language as you show it:
- `stats.csv` — the headline metric (C-index for survival, AUROC for classification, Pearson r for regression). Explain what the number means and whether it looks reasonable for a 1-iteration smoke test.
- `embeddings_test.csv` — the model's learned representation of each sample in latent space; PCA reveals whether the model is separating biologically meaningful groups even without being explicitly trained to do so.
- `feature_importance.IntegratedGradients.csv` — which genes drove predictions; cross-referencing against known biology (e.g. IDH1 for glioma survival) is a sanity check that the model is learning real signal and not noise.
- `predicted_labels.csv` — the actual predictions on the test set; for survival, splitting by median risk score into a Kaplan-Meier plot shows whether the model meaningfully stratifies patients.

After walking through all outputs, **first offer the user a notebook** before suggesting the full run:

> "Would you like me to save a Jupyter notebook summarising what we've done so far? It will include the dataset setup, the smoke test command, the output stats, and the plots — with short notes at each step so you can re-run or share it."

If yes, write the notebook now (see Step 7 for format). If no, proceed.

Then tell the user explicitly:

> "This smoke test used only 1 HPO iteration — it found a single hyperparameter configuration essentially at random. The result is a lower bound on what the model can do. To get the best performance, we need to run Bayesian hyperparameter optimisation properly. I recommend `--hpo_iter 100` with `--hpo_patience 30` (stop early if no improvement for 30 steps). This will take longer — typically 20–60 minutes on CPU depending on dataset size — but will find a substantially better configuration."

Only after this explanation, ask the user if they want to proceed with the full run.

#### Step 5c — Full training run (after the user agrees)

Use `--hpo_iter 100 --hpo_patience 30` for the real run. The `--hpo_patience` flag stops early if the Bayesian optimiser stops improving, so in practice it often finishes faster than 100 iterations.

**Feature selection for full runs:** Use `--features_top_percentile 20` (top 20% by Laplacian score) with `--features_min 1000` to ensure at least 1000 features per modality are retained. The smoke test uses 5% to keep it fast; the full run benefits from a broader feature set. Adjust downward (e.g. 10%) for very large datasets (>50k features per modality) if memory or runtime is a concern.

**Classification — tumor subtype:**
```bash
flexynesis \
  --data_path lgggbm_tcga_pub_processed \
  --model_class DirectPred \
  --target_variables HISTOLOGICAL_DIAGNOSIS \
  --data_types mut,cna \
  --hpo_iter 100 \
  --hpo_patience 30 \
  --features_top_percentile 10 \
  --outdir results \
  --prefix lgg_subtype
```

**Survival analysis:**
```bash
flexynesis \
  --data_path lgggbm_tcga_pub_processed \
  --model_class DirectPred \
  --surv_event_var OS_STATUS \
  --surv_time_var OS_MONTHS \
  --data_types mut,cna \
  --hpo_iter 100 \
  --hpo_patience 30 \
  --features_top_percentile 10 \
  --outdir results \
  --prefix lgg_survival
```

**Drug response regression (single drug first):**
```bash
flexynesis \
  --data_path dataset1 \
  --model_class DirectPred \
  --target_variables Erlotinib \
  --data_types gex,cnv \
  --hpo_iter 100 \
  --hpo_patience 30 \
  --features_top_percentile 10 \
  --log_transform True \
  --outdir results \
  --prefix erlotinib_response
```

**Binary classification (MSI):**
```bash
flexynesis \
  --data_path dataset2 \
  --model_class DirectPred \
  --target_variables y \
  --data_types gex,meth \
  --hpo_iter 100 \
  --hpo_patience 30 \
  --features_top_percentile 10 \
  --outdir results \
  --prefix msi_classification
```

**Cell type classification (single-cell):**
```bash
flexynesis \
  --data_path singlecell_bonemarrow \
  --model_class supervised_vae \
  --data_types gex \
  --hpo_iter 100 \
  --hpo_patience 30 \
  --features_top_percentile 10 \
  --outdir results \
  --prefix bonemarrow_celltypes
```

#### Step 5d — Multi-task training (only after single-task baseline works)

Multi-task is a key strength of flexynesis — the model learns shared representations and balances losses automatically. Suggest it **only** after the user has seen a working single-task result and wants to explore further.

**Classification + survival:**
```bash
flexynesis \
  --data_path lgggbm_tcga_pub_processed \
  --model_class DirectPred \
  --target_variables HISTOLOGICAL_DIAGNOSIS \
  --surv_event_var OS_STATUS \
  --surv_time_var OS_MONTHS \
  --data_types mut,cna \
  --hpo_iter 100 \
  --hpo_patience 30 \
  --features_top_percentile 10 \
  --outdir results \
  --prefix lgg_multitask
```

**Multi-task with regression (breast cancer):**
```bash
flexynesis \
  --data_path brca_metabric \
  --model_class DirectPred \
  --target_variables CLAUDIN_SUBTYPE,CHEMOTHERAPY \
  --data_types gex,cna \
  --hpo_iter 100 \
  --hpo_patience 30 \
  --features_top_percentile 10 \
  --outdir results \
  --prefix brca_multitask
```

**Useful flags:**
- `--hpo_iter N` — Bayesian HPO steps (1 for smoke test, 100 for real runs)
- `--hpo_patience N` — stop HPO early after N non-improving steps (use 30 for real runs)
- `--features_top_percentile P` — keep top P% by Laplacian score (try 5–15 for large datasets)
- `--use_gpu` — enable CUDA GPU; `--device mps` for Apple Silicon
- `--evaluate_baseline_performance` — also run RF, SVM, XGBoost, RSF baselines
- `--covariates col1,col2` — include clin.csv columns as extra input features
- `--disable_marker_finding` — skip Captum importance scores (faster)

---

### Step 6 — Read and Visualise Results

After training, all output files are in `--outdir` prefixed by `--prefix`. Read them and produce plots.

#### Performance metrics

```python
import pandas as pd
stats = pd.read_csv('results/<prefix>.stats.csv')
print(stats)
```

Metric guide:
- **Classification**: `AUROC`, `AUPR`, `balanced_accuracy`, `F1`, `kappa`
- **Regression**: `pearson_r`, `R2`, `MSE`
- **Survival**: `c_index` — above 0.6 is meaningful, above 0.7 is strong

#### PCA of sample embeddings

```python
import pandas as pd, matplotlib.pyplot as plt
from sklearn.decomposition import PCA

prefix = 'results/<prefix>'
dataset = '<dataset_key>'

emb  = pd.read_csv(f'{prefix}.embeddings_test.csv', index_col=0)
clin = pd.read_csv(f'{dataset}/test/clin.csv', index_col=0)

coords = PCA(n_components=2).fit_transform(emb)
color_var = clin.columns[0]   # change to any clinical column
labels = clin.reindex(emb.index)[color_var]

fig, ax = plt.subplots(figsize=(7, 5))
for lbl in labels.dropna().unique():
    mask = labels == lbl
    ax.scatter(coords[mask, 0], coords[mask, 1], label=lbl, s=20, alpha=0.7)
ax.set_xlabel('PC1'); ax.set_ylabel('PC2')
ax.legend(bbox_to_anchor=(1.05, 1), fontsize=8)
plt.tight_layout()
plt.savefig('pca_embeddings.png', dpi=150)
print("Saved pca_embeddings.png")
```

#### Top markers (feature importance)

The importance file is long-format: columns are `target_class`, `target_class_label`, `layer`, `name`, `importance`, `explainer`; the index is the target variable name.

```python
import pandas as pd, matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt

prefix = 'results/<prefix>'
imp = pd.read_csv(f'{prefix}.feature_importance.IntegratedGradients.csv', index_col=0)

for var in imp.index.unique():
    df = imp.loc[var]
    # Average absolute importance across classes per gene
    top = df.groupby('name')['importance'].apply(lambda x: x.abs().mean()).nlargest(20).sort_values()
    fig, ax = plt.subplots(figsize=(6, 6))
    top.plot.barh(ax=ax, color='steelblue')
    ax.set_title(f'Top 20 markers — {var}')
    ax.set_xlabel('Mean |Integrated Gradient| score')
    plt.tight_layout()
    fname = f'top_markers_{var.replace("/","_")}.png'
    plt.savefig(fname, dpi=150)
    print(f"Saved {fname}")
```

Known markers rediscovered by flexynesis in published benchmarks:
- LGG/GBM survival: **IDH1, TP53, ATRX**
- Breast cancer subtypes: **ESR1, ERBB2, CDH1**
- Drug response (Erlotinib): **EGFR**

#### Kaplan-Meier survival curves

```python
import pandas as pd, matplotlib.pyplot as plt
from lifelines import KaplanMeierFitter   # pip install lifelines

prefix = 'results/<prefix>'
dataset = 'lgggbm_tcga_pub_processed'

preds = pd.read_csv(f'{prefix}.predicted_labels.csv', index_col=0)
clin  = pd.read_csv(f'{dataset}/test/clin.csv', index_col=0)
df = preds.join(clin[['OS_STATUS', 'OS_MONTHS']])

score_col = preds.columns[0]
df['risk_group'] = (df[score_col] > df[score_col].median()).map({True: 'High risk', False: 'Low risk'})

kmf = KaplanMeierFitter()
fig, ax = plt.subplots(figsize=(6, 5))
for group, g in df.groupby('risk_group'):
    kmf.fit(g['OS_MONTHS'], g['OS_STATUS'].astype(int), label=group)
    kmf.plot_survival_function(ax=ax)
ax.set_title('Kaplan-Meier: predicted risk groups')
plt.tight_layout()
plt.savefig('kaplan_meier.png', dpi=150)
print("Saved kaplan_meier.png")
```

---

### Step 7 — What Next?

After showing the results, **always ask the user** whether they would like a notebook report of the session:

> "Would you like me to save a Jupyter notebook summarising what we did? It will include the key flexynesis commands, the output stats, and the plots — with short text notes at each step so you can re-run or share it."

If yes, write a `.ipynb` file containing:
- A markdown cell introducing the dataset and task
- Code cells for each step actually executed: data download + prep, smoke test command, full training command (if run), and all visualisation code
- Short markdown cells between steps explaining what each step does and what the output means
- The actual metric values from `stats.csv` summarised in a markdown cell

Only include the flexynesis CLI commands and result-reading code — not the EDA exploration or debugging code from the interactive session. Keep it clean enough for the user to hand to a colleague.

**Always remind the user to cite the flexynesis paper if they use it in their research:**

> "If flexynesis contributes to your analysis, please cite: Uyar et al., *Nature Communications* 2025. https://doi.org/10.1038/s41467-025-63688-5"

After showing the results, offer these options in order — simpler ones first:

- **More HPO**: increase `--hpo_iter` to 100 with `--hpo_patience 30` for better hyperparameter tuning on the same single task (if not already done)
- **Add baselines**: append `--evaluate_baseline_performance` to compare DirectPred against RF/SVM/XGBoost/RSF
- **Try another model**: swap `--model_class` to `supervised_vae` or `MultiTripletNetwork` for richer embeddings or better cluster separation
- **Add a second task (multi-task)**: only after the single-task baseline looks good — combine with a second well-scoring variable from the EDA
- **Different dataset**: fetch a different cBioPortal study
- **Inference on new data**: apply the trained model without retraining:

```bash
flexynesis \
  --pretrained_model results/<prefix>.final_model.pth \
  --artifacts results/<prefix>.artifacts.joblib \
  --data_path_test <new_data>/test \
  --target_variables <same_vars_as_training>
```

---

## Output File Reference

| File | Contents |
|---|---|
| `<prefix>.stats.csv` | Performance metrics per target variable |
| `<prefix>.predicted_labels.csv` | Predictions on test set |
| `<prefix>.embeddings_train.csv` | Latent space coordinates (train samples) |
| `<prefix>.embeddings_test.csv` | Latent space coordinates (test samples) |
| `<prefix>.feature_importance.IntegratedGradients.csv` | Per-feature importance (IntGrad) |
| `<prefix>.feature_importance.GradientShap.csv` | Per-feature importance (GradShap) |
| `<prefix>.final_model.pth` | Trained model weights |
| `<prefix>.final_model_config.json` | Architecture configuration |
| `<prefix>.artifacts.joblib` | Scalers, feature lists, label encoders for inference |

---

## Known Performance Benchmarks

| Dataset | Task | Model | Modalities | Typical result |
|---|---|---|---|---|
| dataset1 (CCLE/GDSC) | Drug response regression | DirectPred | gex, cnv | Pearson r ≈ 0.4–0.7 per drug |
| lgggbm_tcga_pub | Survival | DirectPred | mut, cna | C-index ≈ 0.70–0.75 |
| lgggbm_tcga_pub | Tumour subtype classification | DirectPred | mut, cna | AUROC > 0.90 |
| brca_metabric | CLAUDIN_SUBTYPE | DirectPred | gex, cna | Balanced accuracy ≈ 0.75–0.80 |
| brca_metabric | CHEMOTHERAPY | supervised_vae | gex, cna | AUROC ≈ 0.70 |
| dataset2 (MSI) | MSI-H vs MSS | MultiTripletNetwork | gex, meth | AUROC > 0.90 |
| singlecell_bonemarrow | Cell type classification | supervised_vae | gex | Balanced accuracy > 0.85 |

Full benchmark dashboard: https://bimsbstatic.mdc-berlin.de/akalin/buyar/flexynesis-benchmark-datasets/dashboard.html

---

## flexynesis Architecture (for context)

- Each omics modality is encoded by a separate `Encoder` (Linear → LeakyReLU → BatchNorm) into a latent vector
- Latent vectors are concatenated and passed through a fusion layer
- Per-variable MLP heads handle prediction; losses are balanced by learnable uncertainty weights
- All models implement `.transform()` (embeddings), `.predict()`, and `.compute_feature_importance()`
- HPO uses Bayesian optimisation (scikit-optimize `gp_hedge`) over latent_dim, hidden_dim_factor, lr, batch_size

Paper: Uyar et al., *Nature Communications* 2025. https://doi.org/10.1038/s41467-025-63688-5
