# ClinVar Variant Pathogenicity Classifier

Predicting whether genomic variants are pathogenic or benign
using machine learning on 3.4 million real clinical records
from the NCBI ClinVar database.

Built as a self-study project during the first semester of
Bioinformatics (TUM/LMU Munich).

---

## Biological question

Given a genomic variant described only by its chromosomal
position, gene, and mutation type — can we predict whether
it is pathogenic or benign?

This mirrors a real clinical scenario: a patient's genome
is sequenced, a novel variant is found, and no prior
classification exists. The model must predict from sequence
features alone.

---

## Data

- Source: NCBI ClinVar, variant_summary.txt
- Download: https://ftp.ncbi.nlm.nih.gov/pub/clinvar/tab_delimited/
- Size: 8.9M variants, filtered to 3.4M with clear labels
- Labels: Pathogenic (704k) and Benign (2.7M)

---

## Methodological decisions

**Features used (all available from sequencing):**

| Feature | Description |
|---|---|
| Chromosome | Chromosome number (X=23, Y=24, MT=25) |
| Start | Genomic position |
| GeneSymbol | Gene name (label encoded) |
| Type | Mutation type (SNV, Deletion, Indel...) |
| Origin | Germline or somatic |

**Two features explicitly excluded:**

`PhenotypeList` — lists diseases associated with the variant.
Excluded because a novel unclassified variant has no disease
associations yet. Using it would inflate performance but make
the model inapplicable to real clinical cases (data leakage).

`ReviewStatus` — quality of the ClinVar record. Rather than
using it as a predictive feature, we used it as **sample weights**
during training. High-quality expert-reviewed records contribute
more to learning, but the model predicts without this information —
consistent with the real-world scenario where new variants
have no review status.

---

## Model

Random Forest (100 trees) with:
- `class_weight='balanced'` to handle 4:1 Benign:Pathogenic imbalance
- `sample_weight` from ReviewStatus during training
- 80/20 stratified train/test split

---

## Results

| Metric | Value |
|---|---|
| ROC-AUC | 0.865 |
| Accuracy | 0.86 |
| Precision (Pathogenic) | 0.65 |
| Recall (Pathogenic) | 0.69 |

**Feature importance:**

| Feature | Importance |
|---|---|
| Start (genomic position) | 0.540 |
| Type (mutation type) | 0.228 |
| GeneSymbol | 0.100 |
| Origin | 0.063 |
| Chromosome | 0.029 |

Genomic position is by far the most informative feature,
consistent with the concept of functional constraint —
certain genomic positions are conserved across evolution
and intolerant of variation.

---

## Error analysis

Evaluated on test set only (685,749 variants unseen during training).

**By mutation type:**

| Type | Error rate | FN rate | FP rate |
|---|---|---|---|
| Microsatellite | 0.218 | 0.264 | 0.182 |
| Duplication | 0.205 | 0.137 | 0.341 |
| Deletion | 0.169 | 0.101 | 0.379 |
| SNV | 0.140 | 0.484 | 0.091 |
| Copy number gain | 0.084 | 0.122 | 0.059 |

SNVs have the highest false negative rate (48%) despite being
the most common type — single nucleotide effects depend on
protein context, not position alone. Structural variants
show high false positive rates as the model conflates their
coordinates with nearby pathogenic hotspots.

**By chromosome:**

Chromosome X has the highest error rate (18.5%, FN 22.6%),
consistent with the complexity of hemizygous X-linked
inheritance. Mitochondrial DNA has the lowest error rate
(2.7%) — mitochondrial pathogenicity follows cleaner
position-based patterns.

**By gene:**

Genes with highest false negative rate: RELN (81%), C2CD3 (81%),
DOCK7 (80%) — large neurodevelopmental genes where pathogenicity
depends on protein domain context beyond genomic position.

Genes with highest false positive rate: GBA1 (77%), TWIST1 (69%),
GJB1 (65%) — well-studied disease genes where benign variants
cluster near known pathogenic hotspots.

**Conclusion:** Both error patterns point to the same limitation —
genomic position is necessary but not sufficient. Protein domain
annotation, population frequency (gnomAD), and evolutionary
conservation (PhyloP) would directly address the systematic
errors identified here.

---

## Limitations

- No protein structure features (domain, active site)
- No evolutionary conservation scores (PhyloP, GERP)
- No population frequency data (gnomAD)
- Label quality varies — some ClinVar entries are single
  unreviewed lab submissions

---

## Future work

- Add gnomAD allele frequency as a feature
- Add PhyloP conservation scores
- Add UniProt protein domain annotation
- Train separate models for germline vs somatic variants
- Compare with Logistic Regression and XGBoost baselines

---

## Reproduce
```bash
# Clone repository
git clone https://github.com/dariasof/clinvar-ml-classifier

# Create environment
conda env create -f environment.yml
conda activate clinvar-ml

# Download data
cd data
wget https://ftp.ncbi.nlm.nih.gov/pub/clinvar/tab_delimited/variant_summary.txt.gz

# Run analysis
jupyter notebook notebooks/02_final_analysis.ipynb
```

---

## Project structure
```
clinvar-ml-classifier/
├── README.md
├── environment.yml
├── notebooks/
│   ├── analysis.ipynb  ← full pipeline from loading to error analysis
├── results/
│   └── figures/
│       ├── model_results.png
│       └── error_analysis.png
└── data/
    └── variant_summary.txt.gz   ← not tracked by git
```

---

## Tech stack

Python 3.11 · pandas · scikit-learn · matplotlib · numpy
```

---

Two things to do after copying this in.

First, create a `.gitignore` file in the root folder so the large data file doesn't get uploaded to GitHub:
```
data/
__pycache__/
.ipynb_checkpoints/
*.gz