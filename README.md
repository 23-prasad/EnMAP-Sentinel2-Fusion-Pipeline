# EnMAP hyperspectral processing + Sentinel-2 fusion

**Urban land-cover classification and vegetation-fraction estimation | Potsdam, Germany**

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/YOUR-USERNAME/YOUR-REPOSITORY/blob/main/EnMAP_S2_fusion_pipeline.ipynb)

> **Before publishing:** replace `YOUR-USERNAME` and `YOUR-REPOSITORY` in the badge URL with your exact GitHub account and repository name. The linked notebook must be named `EnMAP_S2_fusion_pipeline.ipynb` at the repository root. See **Open in Colab** below.

## Overview

This self-contained **learning project and research-method demonstrator** explores whether feature-level integration of **EnMAP hyperspectral** data and **Sentinel-2 multispectral** data improves six-class urban land-cover classification or 30 m vegetation-fraction estimation. Processing, feature engineering and modelling are implemented in Python using publicly accessible sample data. It is **not an operational or independently validated land-cover product**.

**Key question:** Does stacking EnMAP spectral information with Sentinel-2 features outperform either sensor alone on this Potsdam example?

## Main findings (recorded example run)

| Feature set | Classification overall accuracy | Macro-F1 | Vegetation fraction R² | Vegetation fraction RMSE |
|---|---:|---:|---:|---:|
| Sentinel-2 bands + indices | 0.913 | **0.630** | 0.875 | 0.126 |
| EnMAP PCA | 0.867 | 0.570 | 0.786 | 0.165 |
| EnMAP retained bands | 0.876 | 0.595 | 0.799 | 0.160 |
| EnMAP + Sentinel-2 feature stack | **0.916** | 0.616 | **0.880** | **0.124** |

**Interpretation:** Fusion raised overall accuracy by only **0.003** and R² by **0.005** versus Sentinel-2, while lowering macro-F1 by **0.014**. Without fold-wise uncertainty, these modest numerical differences **do not establish a meaningful gain**. Roof and soil have particularly poor F1 scores (fusion approximately 0.05 and 0.04); overall accuracy conceals severe class imbalance / scale-related weaknesses. These results reflect a single scene and specific training labels.

**Important validation note:** Spatial-group CV is used, but PCA is fit on image data *before* the cross-validation folds. For a rigorous inductive assessment, PCA must be fitted within each training fold. Findings here should be presented as exploratory.

## Data and provenance

| Input | Description | Provenance |
|---|---|---|
| EnMAP L2A | Potsdam scene, 24 July 2022; hyperspectral surface reflectance, nominal 30 m | [EnMAP-Box](https://github.com/EnMAP-Box/enmap-box) example data; EnMAP / DLR |
| Land-cover reference | EnMAP-Box polygons derived from 2019 high-resolution aerial interpretation; six classes | EnMAP-Box sample data, with GeoBasis-DE/LGB attribution (`dl-de/by-2-0` as identified in the original project notes) |
| Sentinel-2 L2A | Example selected item: `S2A_33UUU_20220720_0_L2A`, 20 July 2022 | [Element84 Earth Search](https://earth-search.aws.element84.com/v1) STAC, Copernicus Sentinel data |

The example data is requested programmatically, without credentials. **Check the provider terms and attribution when redistributing source imagery**; this repository does not include the original imagery. The Sentinel-2 catalogue may select a different scene if available items or search conditions change.

## Processing methods

1. Download EnMAP reflectance and polygons; examine spatial and spectral metadata.
2. Mask flagged bands and water-vapour regions; smooth valid spectral segments with Savitzky–Golay filtering.
3. Aggregate labelled polygons through 3 m rasterisation into 30 m class fractions; select relatively pure pixels for classification.
4. Build EnMAP spectral indices and PCA representations.
5. Query Sentinel-2 L2A with STAC, mask invalid/cloudy pixels and resample selected bands to the common EnMAP grid.
6. Diagnose cross-sensor spectral agreement and potential residual shifts.
7. Compare Sentinel-2, EnMAP PCA, EnMAP spectral and stacked-feature Random Forest models using four spatial folds (approximately 600 m blocks).
8. Estimate vegetation-cover fraction and export maps, model and evaluation tables.

**Fusion definition:** Feature-level stacking at 30 m. **No claim is made that the notebook produces sharpened 10 m EnMAP imagery.**

## Reproduce in Google Colab

1. Click **Open in Colab** above (after replacing the placeholder account/repository names), or in Colab select **File → Open notebook → GitHub** and paste the repository URL.
2. Run cells from top to bottom; dependencies are installed in the setup cell. The notebook uses network access to fetch public data.
3. Inspect `CFG` in section 0 to change folders, search parameters, thresholds and model options.
4. **Verify `s2_is_real` is `True`.** If STAC retrieval fails, the notebook intentionally uses synthetic Sentinel-2 bands made from EnMAP as an offline *smoke test*. Results from that fallback are **not scientific evidence for multisensor fusion**.
5. Inspect classification and regression outputs. Download `outputs.zip` from the export section or persist results to Drive. A typical Colab run may take several minutes; runtime varies by instance and network.

**Local execution:** Jupyter with a compatible Python environment and the packages listed in `requirements.txt`. Because package versions and services change, rerun dependency checks before relying on exact reproducibility.

## Repository structure

```text
EnMAP_S2_fusion_pipeline.ipynb   # executable tutorial / experimental workflow
README.md                        # research summary and reproduction notes
requirements.txt                 # dependency list
.gitignore                       # excluded large inputs and transient files
outputs/outputs/                 # optional small example CSV/JSON/model artifacts
```

Large generated GeoTIFF imagery is generally **not committed**; rerun the notebook to generate full-resolution maps. If your local repository uses a different folder arrangement, update this section to match.

## Limitations

- Single-location example with reference polygons from 2019 and satellite observations from 2022.
- Small numeric differences between model configurations were not tested for statistical significance, and fold-wise variation has not been reported.
- Pre-CV PCA uses image features from held-out locations; a fold-contained preprocessing pipeline is needed for stronger validation claims.
- Fine-grained urban roof and soil classes are often mixed at 30 m and perform poorly.
- Results may depend on acquisition conditions, temporal mismatch, spatial alignment, model choices and STAC availability.
- The saved `.joblib` classifier should only be loaded from a trusted source; model files may be dependency-version sensitive.

## Research extensions

- Fit preprocessing within spatial folds; report per-fold distributions and confidence intervals.
- Test across additional scenes, dates and locations, using an independent external test area.
- Investigate spectral-sensitive questions (urban materials, vegetation traits, soils) where hyperspectral bands might add value.
- Compare spectral unmixing, time-series features and controlled spatial sharpening approaches.

## Acknowledgements

EnMAP / DLR; EnMAP-Box contributors; Copernicus Sentinel programme; Element84 Earth Search; and providers of the high-resolution land-cover reference data. This repository is an independent learning exercise, not an official EnMAP, Copernicus or DLR product.

## Licence

**Code licence not yet selected.** Before publishing, choose a suitable licence for *your code* separately from the licences / attribution conditions for the input datasets. Do not assume that open access to data grants unrestricted redistribution.
