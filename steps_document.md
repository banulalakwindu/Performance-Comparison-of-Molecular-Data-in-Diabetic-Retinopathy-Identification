# Steps Document

This document walks through the pipeline implemented in each notebook in
[`Notebooks/`](Notebooks/), so that the work can be followed without opening
every notebook. It records what each notebook does, which files it reads and
writes, and where the results reported in the published paper appear in this
repository.

> **Note on provenance.** The notebooks and their saved outputs are the
> computational record of the study and are intentionally left unmodified.
> This document was written *from* those notebooks; nothing in them was
> re-run or edited.

## 1. Overall Pipeline

Every dataset notebook follows the same five-stage pipeline:

1. **Preprocessing** — import the molecular data, orient it as
   *samples × features*, align it with the phenotype/diagnosis table, encode
   categorical values numerically, and (where required) normalise to [0, 1].
2. **Feature selection** — rank features with four algorithms:
   - Information Gain (Mutual Information) — `mutual_info_classif`
   - Correlation Coefficient — `f_classif`
   - Chi-Square — `chi2`
   - Feature Importance — `RandomForestClassifier.feature_importances_`
3. **Best-selector screening** — build a Random Forest on the top
   25 / 50 / 100 / 150 / 200 features of each selector and compare
   cross-validation scores.
4. **Model building** — generate candidate feature sets of increasing size,
   then evaluate models with 5-fold cross-validation:
   - linear SVM, Logistic Regression, Random Forest, Naïve Bayes
   (the GSE221521 notebooks additionally test other classifiers).
5. **Comparison** — pick the (feature set, model) pair with the best CV
   accuracy. Per-combination results are written to
   `Datasets/DatasetNN/Feature_Select/results.csv`, and the candidate feature
   sets themselves to `Datasets/DatasetNN/Feature_Select/dataset_*.csv`.

The same framework is described in [`Includes/Framework.txt`](Includes/Framework.txt).

## 2. Datasets Used in the Published Study

| Paper role | GEO accession | Repo location | Raw file |
|---|---|---|---|
| DNA methylation (5hmC, cfDNA) | **GSE140842** | `Datasets/Dataset01` | `GSE140842_5hmC_NormCount.csv` |
| small RNA (retina) | **GSE160310** (subseries GSE160308) | `Datasets/Dataset02` | `GSE160308_human_retina_DR_smallRNA_counts.txt` |
| total RNA (retina) | **GSE160310** (subseries GSE160306) | `Datasets/Dataset03` | `GSE160306_human_retina_DR_totalRNA_counts.txt` |
| smallRNA + totalRNA combined | as above | `Datasets/Dataset04` | derived from Dataset02 + Dataset03 |

> The two transcriptome raw files carry the subseries accessions **GSE160308**
> (small RNA) and **GSE160306** (total RNA); the paper cites the parent
> accession **GSE160310**, which links both subseries.

## 3. Notebook Walkthrough (Paper Datasets)

### Dataset_01.ipynb — DNA methylation (GSE140842)

1. Import `GSE140842_5hmC_NormCount.csv`, transpose to *samples × features*,
   join the case/control diagnosis, and encode values numerically
   (no imputation was needed; data were already normalised). Save `data2.csv`.
2. Screen the four feature-selection algorithms with Random Forest and pick
   the best (correlation coefficient won at every feature count tested).
3. Re-run feature importance with Logistic Regression to sanity-check the
   selector, plot the selected features, and inspect their distributions.
4. Build models with different numbers of correlation-selected features and
   compare cross-validation accuracies.

**Headline result (paper §3):** top **48** correlation-coefficient-selected
methylation features + **Logistic Regression** → accuracy
**0.9571 ± 0.035**. This row is preserved in
`Datasets/Dataset01/Feature_Select/results.csv`
(`Logistic Regression,48,0.957142857,0.034992711`) and matches Fig. 2 of the
paper.

### Dataset_02.ipynb — small RNA (GSE160308 / parent GSE160310)

1. Convert the raw `GSE160308_human_retina_DR_smallRNA_counts.txt` to CSV,
   transpose to *samples × features*, join the diagnosis table, and
   min–max-normalise to [0, 1]. Save `data2.csv`.
2. Screen the four feature-selection algorithms with Random Forest — feature
   importance performed best consistently.
3. Build models over increasing feature-set sizes and compare CV accuracies.

**Headline result (paper §3):** top **48** feature-importance-selected
small-RNA features + **SVM** → accuracy **0.92 ± 0.07** (5-fold CV). The
supporting rows are in `Datasets/Dataset02/Feature_Select/results.csv`
(e.g. `SVM,48,0.925,0.0728…`).

### Dataset_03.ipynb — total RNA (GSE160306 / parent GSE160310)

Same pipeline as Dataset_02, applied to
`GSE160306_human_retina_DR_totalRNA_counts.txt`.

**Headline result (paper §3, final model of the study):** top **14**
feature-importance-selected total-RNA features + **Naïve Bayes**
(`GaussianNB`) → accuracy **0.9625 ± 0.05**. This is the best model of the
study and corresponds to Figs. 5–7 and Table 4 of the paper. Supporting rows
in `Datasets/Dataset03/Feature_Select/results.csv`
(`Naive Bayes,14,0.9625,0.05`, also 15/16 features at 0.9625 ± 0.03).

### Dataset_04.ipynb — combined smallRNA + totalRNA

1. Concatenate the processed small-RNA (`Dataset02/data2.csv`) and
   total-RNA (`Dataset03/data2.csv`) feature tables, keeping the shared
   diagnosis column. Save `Datasets/Dataset04/data2.csv`.
2. Screen the four feature-selection algorithms (Random Forest, feature
   counts 25–200).
3. Evaluate a wider model pool — SVM (linear), SVM (poly), Random Forest,
   Logistic Regression, Naïve Bayes, XGBoost, ANN (MLP) — across many
   candidate feature-set sizes (561 evaluated combinations are recorded in
   `Datasets/Dataset04/Feature_Select/results.csv`).

**Observed best:** Random Forest, 76 combined features →
0.9500 ± 0.0250 (`results.csv`), which did not surpass the total-RNA-only
model reported in the paper. This notebook corresponds to the extension
experiments beyond the three single-omics comparisons in the paper.

## 4. Notebook Walkthrough (GSE221521 Exploratory Datasets)

`Datasets/Raw_Data/GSE221521_gene_expression.csv` is a retina gene-expression
matrix with 193 samples. Notebooks `Dataset_05`–`Dataset_21` each isolate one
gene-type subset (filtering on the `gene_type` column, keeping the first 193
columns, then transposing), join the shared diagnosis table
(`Datasets/Dataset05/Diagnosis.csv`), remove non-diabetic controls
(`Diagnosis != 0`, shifting labels to 0/1), and re-run the same
feature-selection + model-comparison pipeline.

| Notebook | Gene type kept | Data folder |
|---|---|---|
| Dataset_05 | protein_coding | `Datasets/Dataset05` |
| Dataset_06 | lncRNA | `Datasets/Dataset06` |
| Dataset_07 | processed_pseudogene | `Datasets/Dataset07` |
| Dataset_08 | unprocessed_pseudogene | `Datasets/Dataset08` |
| Dataset_09 | misc_RNA | `Datasets/Dataset09` |
| Dataset_10 | snRNA | `Datasets/Dataset10` |
| Dataset_11 | miRNA | `Datasets/Dataset11` |
| Dataset_12 | TEC | `Datasets/Dataset12` |
| Dataset_13 | snoRNA | `Datasets/Dataset13` |
| Dataset_14 | transcribed_unprocessed_pseudogene | `Datasets/Dataset14` |
| Dataset_15 | transcribed_processed_pseudogene | `Datasets/Dataset15` |
| Dataset_16 | rRNA_pseudogene | `Datasets/Dataset16` |
| Dataset_17 | IG_V_pseudogene | `Datasets/Dataset17` |
| Dataset_18 | IG_V_gene | `Datasets/Dataset18` |
| Dataset_19 | transcribed_unitary_pseudogene | `Datasets/Dataset19` |
| Dataset_20 | TR_V_gene | `Datasets/Dataset20` |
| Dataset_21 | all gene types (no filter) | `Datasets/Dataset21` |

These are exploratory extensions of the published GSE140842/GSE160310
comparison and are not part of the paper's reported results.

## 5. Where Each Paper Result Lives

| Paper result | Location in this repo |
|---|---|
| Methylation: 0.9571 ± 0.035 (48 features, Logistic Regression) | `Datasets/Dataset01/Feature_Select/results.csv`; `Notebooks/Dataset_01.ipynb` outputs |
| smallRNA: 0.92 ± 0.07 (48 features, SVM) | `Datasets/Dataset02/Feature_Select/results.csv`; `Notebooks/Dataset_02.ipynb` outputs |
| totalRNA: 0.9625 ± 0.05 (14 features, Naïve Bayes) | `Datasets/Dataset03/Feature_Select/results.csv`; `Notebooks/Dataset_03.ipynb` outputs |
| Combined-omics exploration (best RF 0.95 ± 0.025) | `Datasets/Dataset04/Feature_Select/results.csv`; `Notebooks/Dataset_04.ipynb` outputs |

Raw GEO data files are excluded from git via `.gitignore` (`Raw_Data/`) and
should be downloaded from GEO using the accessions above.

## 6. Environment

The execution environment recorded in the notebook metadata is documented in
[`requirements-freeze.txt`](requirements-freeze.txt) (Python 3.10.9 / 3.12.4,
Anaconda). Library versions were not captured in the saved notebook outputs,
so the runtime dependencies there are listed without pins; the lightweight
list is in [`requirements.txt`](requirements.txt).
