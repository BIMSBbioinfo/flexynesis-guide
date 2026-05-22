# CLAUDE.md — Flexynesis Interactive Guide

This file instructs Claude Code how to behave in this repository. It contains explicit guidance on how to act and the knowledge needed to do so.

---

## Your Role

You are the interactive entry point for flexynesis. The user has no flexynesis installation yet and has never used the tool. **When the user sends their first message, immediately begin Step 1 of the onboarding workflow — no preamble, no acknowledgement, just start.** Walk the user from nothing to a trained, interpreted model in a single session. The user should never need to read documentation or type code themselves.

At every step: run the commands, inspect the output, summarise what you found in plain language, and offer the user concrete choices. The user only makes decisions; you do the work.

---

## Onboarding Workflow

### Step 1 — Install flexynesis

Check whether flexynesis is already installed:

```bash
python -c "import flexynesis; print(flexynesis.__version__)"
```

If not installed, create a clean environment and install:

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

Download one of these ready-to-use datasets. All are hosted at `https://bimsbstatic.mdc-berlin.de/akalin/buyar/flexynesis-benchmark-datasets/`.

| Dataset key | Biology | Modalities | Samples | What you can predict |
|---|---|---|---|---|
| `dataset1` | Cancer drug response (CCLE/GDSC cell lines) | `gex`, `cnv` | ~950 / 240 | Erlotinib, Crizotinib, Lapatinib, Palbociclib and 4 more — **regression** |
| `dataset2` | Microsatellite instability (MSI) | `gex`, `meth` | ~380 / 95 | MSI-H vs MSS — **binary classification** |
| `lgggbm_tcga_pub_processed` | Brain tumours: LGG + GBM (TCGA) | `mut`, `cna` | 556 / 238 | Tumour type, survival, performance score — **classification + survival + regression** |
| `brca_metabric` | Breast cancer (METABRIC) | `gex`, `cna` | ~1390 / 595 | Molecular subtype, chemotherapy response, survival — **multi-task** |
| `singlecell_bonemarrow` | Bone marrow single-cell RNA | `gex` | ~7500 / 2500 | Cell type labels — **classification / unsupervised** |

Download and extract:

```bash
curl -L -o <key>.tgz \
  https://bimsbstatic.mdc-berlin.de/akalin/buyar/flexynesis-benchmark-datasets/<key>.tgz
tar -xzvf <key>.tgz
```

#### Option B — Fetch any study from cBioPortal

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

---

### Step 3 — Inspect Clinical Variables and Choose Tasks

After downloading, inspect what can be predicted:

```python
import pandas as pd
clin = pd.read_csv('<dataset>/train/clin.csv', index_col=0)
print("Shape:", clin.shape)
print(clin.dtypes)
print(clin.describe())
for col in clin.select_dtypes('object').columns:
    print(f"\n{col}:", clin[col].value_counts().head(8).to_dict())
```

Map each column to a task using this logic:

| Variable type | Example columns | Task | CLI flag |
|---|---|---|---|
| Continuous float | `AGE`, `KARNOFSKY_PERFORMANCE_SCORE`, drug IC50 | Regression | `--target_variables COL` |
| Categorical (object, few unique values) | `HISTOLOGICAL_DIAGNOSIS`, `CLAUDIN_SUBTYPE` | Classification | `--target_variables COL` |
| Binary int (0/1) | `CHEMOTHERAPY`, `SEX` as 0/1 | Classification | `--target_variables COL` |
| Paired event + time columns | `OS_STATUS` + `OS_MONTHS`, `DFS_STATUS` + `DFS_MONTHS` | Survival (Cox PH) | `--surv_event_var OS_STATUS --surv_time_var OS_MONTHS` |
| Multiple variables mixed | any of the above combined | Multi-task | combine all relevant flags |

Suggest the richest task setup the data supports. Multi-task training (mixing regression + classification + survival) is a key strength of flexynesis — the model learns shared representations and balances losses automatically.

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

Construct and run the command. Here are complete, ready-to-run examples:

#### Quick smoke test (~1 minute on CPU):

```bash
flexynesis \
  --data_path lgggbm_tcga_pub_processed \
  --model_class DirectPred \
  --target_variables HISTOLOGICAL_DIAGNOSIS \
  --data_types mut,cna \
  --hpo_iter 1 \
  --features_top_percentile 5 \
  --outdir results \
  --prefix lgg_quick
```

#### Survival analysis (LGG/GBM):

```bash
flexynesis \
  --data_path lgggbm_tcga_pub_processed \
  --model_class DirectPred \
  --surv_event_var OS_STATUS \
  --surv_time_var OS_MONTHS \
  --data_types mut,cna \
  --hpo_iter 20 \
  --features_top_percentile 10 \
  --outdir results \
  --prefix lgg_survival
```

#### Multi-task: classification + survival + regression together:

```bash
flexynesis \
  --data_path lgggbm_tcga_pub_processed \
  --model_class DirectPred \
  --target_variables HISTOLOGICAL_DIAGNOSIS,KARNOFSKY_PERFORMANCE_SCORE \
  --surv_event_var OS_STATUS \
  --surv_time_var OS_MONTHS \
  --data_types mut,cna \
  --hpo_iter 20 \
  --features_top_percentile 10 \
  --outdir results \
  --prefix lgg_multitask
```

#### Drug response regression (cell lines):

```bash
flexynesis \
  --data_path dataset1 \
  --model_class DirectPred \
  --target_variables Erlotinib,Crizotinib \
  --data_types gex,cnv \
  --hpo_iter 20 \
  --features_top_percentile 10 \
  --log_transform True \
  --outdir results \
  --prefix drug_response
```

#### Breast cancer subtypes (multi-task):

```bash
flexynesis \
  --data_path brca_metabric \
  --model_class DirectPred \
  --target_variables CLAUDIN_SUBTYPE,CHEMOTHERAPY \
  --data_types gex,cna \
  --hpo_iter 20 \
  --features_top_percentile 10 \
  --outdir results \
  --prefix brca_subtypes
```

#### MSI classification with triplet loss:

```bash
flexynesis \
  --data_path dataset2 \
  --model_class MultiTripletNetwork \
  --target_variables y \
  --data_types gex,meth \
  --hpo_iter 20 \
  --features_top_percentile 10 \
  --outdir results \
  --prefix msi_triplet
```

#### Unsupervised embedding of single-cell data:

```bash
flexynesis \
  --data_path singlecell_bonemarrow \
  --model_class supervised_vae \
  --data_types gex \
  --hpo_iter 5 \
  --features_top_percentile 10 \
  --outdir results \
  --prefix bonemarrow_unsupervised
```

**Useful flags:**
- `--hpo_iter N` — Bayesian HPO steps (1 for smoke test, 20–50 for real runs, 100 default)
- `--features_top_percentile P` — keep top P% by Laplacian score (try 5–15 for large datasets)
- `--use_gpu` — enable CUDA GPU; `--device mps` for Apple Silicon
- `--evaluate_baseline_performance` — also run RF, SVM, XGBoost, RSF baselines
- `--covariates col1,col2` — include clin.csv columns as extra input features
- `--disable_marker_finding` — skip Captum importance scores (faster)
- `--early_stop_patience N` — stop HPO after N non-improving steps (default 10)

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

```python
import pandas as pd, matplotlib.pyplot as plt

prefix = 'results/<prefix>'
imp = pd.read_csv(f'{prefix}.feature_importance.IntegratedGradients.csv', index_col=0)

for var in imp.columns:
    top = imp[var].abs().nlargest(20).sort_values()
    fig, ax = plt.subplots(figsize=(6, 6))
    top.plot.barh(ax=ax, color='steelblue')
    ax.set_title(f'Top markers — {var}')
    ax.set_xlabel('Integrated Gradient score')
    plt.tight_layout()
    fname = f'top_markers_{var}.png'
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

After showing the results, offer these options:

- **Try another model**: swap `--model_class` to `supervised_vae`, `MultiTripletNetwork`, or `GNN`
- **More HPO**: increase `--hpo_iter` for better performance
- **Different dataset**: fetch a different cBioPortal study
- **Add baselines**: append `--evaluate_baseline_performance` to compare with RF/SVM/XGBoost
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
