# Flexynesis — Claude-Guided Quickstart

This repository contains a single `CLAUDE.md` file. When you open it with [Claude Code](https://claude.ai/code), Claude becomes an interactive guide that walks you from zero to a trained multi-omics deep learning model — without you reading any documentation or writing any code.

**You only need:**
- [mamba](https://mamba.readthedocs.io/en/latest/installation/mamba-installation.html) (or conda / pip)
- Claude Code CLI
- An Anthropic API key (free tier available)

Claude installs flexynesis, downloads a dataset, trains the model, and produces plots — all from a conversation.

---

## Quickstart

**1. Install Claude Code**

```bash
npm install -g @anthropic/claude-code
```

Requires Node.js ≥ 18. Install Node from https://nodejs.org if needed.

**2. Get an Anthropic API key**

Sign up at https://console.anthropic.com and create an API key. Then set it:

```bash
export ANTHROPIC_API_KEY=your_key_here
```

Add that line to your `~/.bashrc` or `~/.zshrc` to make it permanent.

**3. Clone this repository and start Claude**

```bash
git clone https://github.com/BIMSBbioinfo/flexynesis-claude.git
cd flexynesis-claude
claude
```

Claude reads `CLAUDE.md` on startup and begins the guided session immediately.

---

## What Claude will do

| Step | What happens |
|---|---|
| 1 | Installs flexynesis via pip into a fresh mamba environment |
| 2 | Asks what biology you want to explore, then suggests a matching dataset |
| 3 | Downloads and extracts the dataset; inspects available clinical variables |
| 4 | Proposes concrete modelling tasks (regression, classification, survival, or a mix) |
| 5 | Recommends a model architecture and explains the tradeoffs |
| 6 | Runs training with hyperparameter optimisation |
| 7 | Reads the output files and generates PCA plots, feature importance charts, Kaplan-Meier curves, and a performance table |

You make the choices; Claude does the typing.

---

## Available datasets

Claude can download any of these ready-to-use benchmark datasets or fetch any study directly from cBioPortal.

| Dataset | Biology | What you can predict |
|---|---|---|
| Cancer drug response (CCLE/GDSC) | Cell line pharmacogenomics | IC50/AUC for 8 compounds — **regression** |
| LGG + GBM (TCGA glioma) | Brain tumours | Survival, tumour type, performance score |
| METABRIC breast cancer | Breast tumours | Molecular subtype, chemotherapy, survival |
| MSI status | Microsatellite instability | MSI-H vs MSS — **binary classification** |
| Bone marrow single-cell RNA | Single-cell biology | Cell type labels — **unsupervised / classification** |
| Any cBioPortal study | Your choice | Depends on clinical metadata |

---

## Example session

```
$ claude

I see you've opened this repository. Let me get you started with flexynesis.

Step 1/7 — Checking installation...
  flexynesis not found. Installing into a new mamba environment...
  ✓ flexynesis 1.2.3 installed

Step 2/7 — Choose a dataset
  What kind of biological question interests you?
    a) Cancer drug response (cell lines)
    b) Brain tumour survival and subtyping     ← LGG + GBM, TCGA
    c) Breast cancer molecular subtypes        ← METABRIC
    d) Microsatellite instability (MSI)
    e) Single-cell unsupervised analysis
    f) Fetch a custom study from cBioPortal

> b

  Downloading lgggbm_tcga_pub_processed...
  556 training samples · 238 test samples
  Modalities: mutations (mut), copy number alterations (cna)

Step 3/7 — Clinical variables found:
  HISTOLOGICAL_DIAGNOSIS    categorical  (4 tumour subtypes)
  AGE                       numeric
  KARNOFSKY_PERFORMANCE_SCORE  numeric
  OS_STATUS + OS_MONTHS     survival pair  ✓
  SEX                       categorical

  Suggested tasks:
    Classify tumour subtype  +  predict survival  +  regress performance score
    (multi-task — flexynesis handles the loss balancing automatically)

> looks good

Step 4/7 — Model: DirectPred  |  Fusion: intermediate

Step 5/7 — Training...
  flexynesis --data_path lgggbm_tcga_pub_processed \
    --model_class DirectPred \
    --target_variables HISTOLOGICAL_DIAGNOSIS,KARNOFSKY_PERFORMANCE_SCORE \
    --surv_event_var OS_STATUS --surv_time_var OS_MONTHS \
    --data_types mut,cna --hpo_iter 20 --outdir results --prefix lgg_run

Step 6/7 — Results:
  HISTOLOGICAL_DIAGNOSIS     AUROC = 0.94   balanced_accuracy = 0.81
  KARNOFSKY_PERFORMANCE_SCORE  pearson_r = 0.43
  Survival (OS)              c_index = 0.71

Step 7/7 — Plots saved:
  pca_embeddings.png
  top_markers_HISTOLOGICAL_DIAGNOSIS.png
  top_markers_OS.png
  kaplan_meier.png

  Top survival markers: IDH1, TP53, ATRX
  (Well-established glioma drivers — the model learned real biology.)

What next?
  a) Try a different model (supervised_vae, GNN, MultiTripletNetwork ...)
  b) Run more HPO steps for better performance
  c) Fetch a different dataset from cBioPortal
  d) Run inference on new samples with this model
```

---

## About flexynesis

Flexynesis is a deep learning toolkit for multi-omics data integration and clinical outcome prediction. It supports fully connected networks, variational autoencoders, graph convolutional networks, and triplet-loss models, with automated feature selection, hyperparameter optimisation, and integrated-gradient marker discovery.

- Documentation: https://bimsbstatic.mdc-berlin.de/akalin/buyar/flexynesis/site/
- Benchmark results: https://bimsbstatic.mdc-berlin.de/akalin/buyar/flexynesis-benchmark-datasets/dashboard.html
- Source code: https://github.com/BIMSBbioinfo/flexynesis
- Paper: Uyar et al., *Nature Communications* 2025. https://doi.org/10.1038/s41467-025-63688-5

If you use flexynesis in your research, please cite the paper above.
