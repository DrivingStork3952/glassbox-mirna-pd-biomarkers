# glassbox-mirna-pd-biomarkers
**From Black-Box to Glass-Box: A Transparent Explainable AI Approach to Serum Exosome microRNA Biomarker Discovery in Early Parkinson’s Disease**

[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
![Python](https://img.shields.io/badge/Python-3.9%2B-blue)
![Status](https://img.shields.io/badge/Status-Active-brightgreen)

## Overview

This project develops a fully transparent **"Glass-Box"** machine learning pipeline to discover reliable microRNA (miRNA) biomarkers for the early, non-invasive diagnosis of Parkinson’s Disease (PD) using serum exosomes.

Unlike traditional black-box AI models, this pipeline prioritizes **interpretability**, **statistical robustness**, and **reproducibility** — making it more trustworthy for potential clinical translation.

## Key Features

- Analysis of the large, recently released **2024 GSE269781** serum exosome miRNA sequencing dataset (~397 samples)
- Advanced preprocessing and quality control
- Boruta stability selection + Random Forest & XGBoost models
- Full SHAP (Shapley Additive exPlanations) interpretability
- Nested cross-validation, permutation testing, and FDR correction
- Biological validation of discovered miRNA signatures
- Open-source and fully reproducible

## Results

- **AUC-ROC**: 82.73%
- **Sensitivity**: 85.39%
- **F1-Score**: 84.23%
- Identified **21 core miRNAs** using Boruta filtering
- Top biomarkers include:
  - **hsa-miR-486-5p** (Mean |SHAP| = 0.509)
  - **hsa-miR-599** (Mean |SHAP| = 0.425) — known LRRK2 regulator
  - **hsa-miR-199a-5p** (Mean |SHAP| = 0.376)
