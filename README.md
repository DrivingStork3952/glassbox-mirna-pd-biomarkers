# MicroRNA Biomarker Discovery for Parkinson's Disease
![Python 3.9+](https://img.shields.io/badge/python-3.9+-blue.svg)
![scikit-learn](https://img.shields.io/badge/scikit--learn-Active-orange.svg)
![Status](https://img.shields.io/badge/Status-In_Progress-success.svg)

 >A comprehensive machine learning pipeline built to identify specific miRNA biomarkers to distinguish Parkinson's Disease (PD) Patients from healthy controls, using multi-cohort serum/exosome small RNA-sequencing data from NCBI GEO.

## Overview

Parkinson's Disease is currently diagnosed clinically, after substantial neurodegenerative damage has already occured. This project investigates whether circulating blood-based miRNAs(which are stable, minimally invasive to collect, and reflect systemic biological state) can serve as an early molecular signal for PD, using publicly available transcriptomic data and leakage-aware machine learning pipeline.

**Final Model:** L1-regularized logistic regression, 14-miRNA panel

**Held-out test performance:** AUC = 0.821, Average Precision = 0.613

**Repeated holdout performance (n=20 resamples):** AUC = 0.80 ± 0.056

## Repository Structure

```
│   .gitignore         #git exclusions
│   README.md          #project overview and documentation
│   requirements.txt   #python package dependencies
│   
├───data
│   ├───processed
│   │       clean_normalized_counts.csv #merged, cpm normalized, log2 transformed matrix (320 samples X 2792 miRNAs)
│   │       final_aligned_labels.csv #binary control/pd values aligned with the csv file above
│   │       
│   └───raw
│           GSE269775_raw_counts.xlsx #2020 n=100
│           GSE269776_raw_counts.xlsx #2021 excluded file corrupted
│           GSE269777_raw_counts.xlsx #2022 n=120
│           GSE269779_raw_counts.xlsx #2023 n=100
│           GSE269781_family.soft.gz #SuperSeries SOFT metadata file from GEO 
│           SraRunTable.csv #NCBI SRA Sample metadata - required for label alignment
│           
├───figures
│       combat_verification.png #pre/post combat batch correction distribution check
│       pca_biomarker_space.png #2d pca projection of final selected miRNA panel
│       roc_pr_performace.png #ROC and Precision Recall Curve, final model
│       
└───src
        01_data_download.ipynb #Downloads raw GEO count matrices
        02_preprocessing.ipynb #Normalization, filtering, label alignment
        03_model_training.ipynb #Batch correction, modeling, evaluation, SHAP interpretation
```
## Data Sources

Three sub-series of the GEO SuperSeries [GSE269781](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE269781)

| Cohort | Geo Accession | Year | n |
|--|--|--|--|
| Cohort 1 | GSE269775 | 2020 | 100 | 
| Cohort 2 | GSE269777 | 2022 | 120 | 
| Cohort 1 | GSE269779 | 2023 | 100 | 

The fourth sub-series (GSE269776, 2021 cohort) was excluded after discovering corruption in its raw count file during preprocessing (see `02_preprocessing.ipynb`). 

**External Validation:** An independent Parkinson's cohort is planned as a held-out validation set to assess cross-cohort generalization. This step is in progress and not yet shown in the results below.

## Methodology

### 1. Preprocessing (`02_preprocessing.ipynb`)
1. Per-cohort CPM normalization (library-size correction applied independently per cohort before merging)
2. Low-expression filtering (miRNA must reach CPM >= 1 in at least 10% of all samples)
3. Inner-join merge across all three cohorts on shared miRNA identifiers
4. log2(CPM + 1) transformation
5. Label alignment against GEO sample metadata (`SraRunTable.csv`)

### 2. Modeling (`03_model_training.ipynb`)
1. **Stratified 80/20 train test split** performed *before* any batch correction, scaling, or feature selection
2. **Batch Correction (ComBat)** fit exclusively on training data; test samples corrected via an offset projection derived from training statistics only - the test set never influences the correction parameters
3. **Feature filtering** (constant and low-variance removal) fit on training data only
4. **Two feature-selection/modeling strategies were systematically compared:**
   - *Boruta (Random Forest-based) feature selection* → Random Forest / XGBoost (GridSearchCV)
   - *L1-regularized logistic regression*, where feature selection happens intrinsically via the L1 penalty
5. **Decision threshold** selected via the F2-score-optimal point on out-of-fold cross-validated training predictions (`cross_val_predict`), so test labels are never used to tune the operating threshold
6. **Interpretation** using SHAP (`LinearExplainer`) on the final model's selected features

### 3. Model Selection: Why choose Logistic Regression over traditional Tree Ensembles?

An initial naive 5-fold cross-validation on the Boruta + XGBoost pipeline reported an AUC of approximately 0.95. This number did not hold up under scrutiny:

| Diagnostic | Finding |
| --- |--- |
| Heldout test AUC (single split)| 0.72-0.76 (**far** below the CV estimate) |
| Cause identified | Boruta selected features using the ***entire*** training set before cross-validation, so each CV fold's "held-out" data had already influenced which features existed in the model|
|**Honest CV AUC** (with features re-selected per fold)|0.882 - ~0.06 AUC ofthe original estimate was optimism bias|
|Repeated holdout (20 resamples, Boruta + XGBoost) |***0.69 ± 0.08***|
|Repeated holdout (20 resamples, l1 logistic regression)|***0.802 ± 0.056***|

The instability stemmed back to a small-N, large-P problem: ~256 training samples against ~2400 candidate miRNA features is a regime where Boruta's discrete, permutation-based feature selection is high-variance (confirmed features per re-sample ranged from 28-55). L1-regularized logistic regression performs feature selection continuously as part of model fitting rather than as a separate combinatorial step, and was both more accurate and substantially more stable across resamples. It was selected as the final model because of this.

## RESULTS

**Final model:** L1-regularized logistic regression (`LogisticRegressionCv`, `penalty='l1'`, `solver='liblinear'`)
**Selected features:** 14 miRNAs (from an initial candidate pool of ~2400)

| Metric | Value |
|---|---|
| Test AUC-ROC | 0.821 |
| Test Average Precision | 0.613 (vs. ~0.3 baseline at this class prevalence)|
| Repeated holdout AUC (mean ± SD, n=20)| 0.8 ± 0.056|
|Operating threshold|F2-optimal, selected via cross-validated training predictions|

### Top Biomarkers (according to SHAP impact)

| miRNA | Direction in PD | Literature Context |
|---|---|---|
| hsa-miR-1290 | Down-regulated | No prior PD-specific biomarker literature identified; mechanistically linked to neuronal cell-cycle exit, a pathway implicated in PD pathogenesis. Flagged as a candidate for further investigation rather than a confirmed marker. |
| hsa-let-7b-5p | Up-regulated | Mixed evidence across the let-7 family in PD literature; direction varies by isoform and study. |
| hsa-miR-195-5p | Down-regulated | Appears in independent published serum-based PD biomarker panels. |
| hsa-miR-130b-3p | Down-regulated | Concordant direction with a 2018 PD plasma profiling study. |
| hsa-miR-15a-5p | Up-regulated | Concordant direction with a 2026 PD blood biomarker study. |


Full ranking and SHAP values can be seen in `03_model_training.ipynb`

## Setup

```bash
pip install -r requirements.txt
```
### Run Order
```
01_data_download.ipynb -> 02_preprocessing.ipynb -> 03_model_training.ipynb
```

Each notebook can be run independently if its required input files already exist in `data/raw/` or `data/processed/`.

## Limitations

- **Sample Size.** With 320 total samples and ~2400 candidate features after filtering, feature selection is inherently unstable; this was characterized explicitly via repeated holdout rather than masked by a single train/test split.
- **Batch correction approximation.** Test-set ComBat correction uses an offset-projection approximation (rather than re-fitting ComBat on test data) to strictly prevent test-set leakage into correction parameters. This is a deliberate methodological trade-off, not an oversight.
- **No external cohort validation yet.** All current results are internal to the GSE269781 SuperSeries. Cross-cohort validation is planned but not yet complete.
- **hsa-miR-1290** is the model's strongest driver but has no confirmed prior association with PD in the literature reviewed for this project.  It should be treated as a hypothesis-generating, not as a validated biomarker.
