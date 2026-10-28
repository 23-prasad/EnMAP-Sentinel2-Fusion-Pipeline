# EnMAP hyperspectral processing + Sentinel-2 fusion

**Urban land-cover classification and vegetation-fraction estimation | Potsdam, Germany**

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/23-prasad/EnMAP-Sentinel2-Fusion-Pipeline/blob/main/EnMAP_S2_fusion_pipeline_v1.ipynb)

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
## Issues encountered and solutions

### Setting up EnMAP-Box in QGIS (Windows)

| Problem | Cause | Solution |
|---|---|---|
| `Permission denied … arrow.dll is in use` while installing EnMAP-Box dependencies with the OSGeo4W installer, even with QGIS closed and no process holding the file | QGIS 3.40.5 was installed with the **standalone .msi** into `C:\Program Files\QGIS 3.40.5`. That folder is write-protected, and the OSGeo4W package installer is meant for OSGeo4W network installs (`C:\OSGeo4W`), not standalone ones. The `OSGeo4W.bat` in the QGIS folder only opens a configured shell; it is not an installer. | Don't run `osgeo4w-setup.exe` against a standalone QGIS. Install the Python dependencies with pip from the **OSGeo4W Shell** instead: `python -m pip install --upgrade --user -r https://raw.githubusercontent.com/EnMAP-Box/enmap-box/main/.env/osgeo4w/requirements_osgeo4w.txt` |
| `error: ssl-missing` from pip, then `ImportError: DLL load failed while importing _ssl: The specified procedure could not be found` | The aborted OSGeo4W run had replaced some libraries but not others, leaving mismatched OpenSSL DLLs in the QGIS folder. | Uninstall QGIS, **delete the leftover `C:\Program Files\QGIS 3.40.5` folder** (the uninstaller only removes files it installed itself), reinstall, check with `python -c "import ssl; print(ssl.OPENSSL_VERSION)"`, then repeat the pip command. |
| pip warnings that scripts such as `f2py.exe` are "not on PATH" | `--user` installs go to `%APPDATA%\Python\Python312`. | Harmless: EnMAP-Box doesn't need those command-line tools. |

### Working with the EnMAP-Box example data

| Problem | Cause | Solution |
|---|---|---|
| `enmap_potsdam_srf.geojson` can't be added to the QGIS map | The file holds spectral response functions with `"geometry": null`: there is nothing to draw on a map. | Open it in EnMAP-Box (*Project → Load Example Data*, or add it in the Data Sources panel) and view it in a **Spectral Library** window. |
| The library shows a dense "comb" of curves instead of the vegetation/water spectra in the tutorial figure | The SRF file contains 224 narrow band-sensitivity curves, not reflectance spectra. The tutorial's `speclib_potsdam` belongs to an older version of the example data. | Build a reflectance library instead: collect profiles from `enmap_potsdam` with the profile tool, or import profiles from `enmap_potsdam` + `landcover_potsdam_point`. |
| Label field `level_3_id` not found | In the current example data, classes are text fields (`level_1`, `level_2`, `level_3`). | The notebook maps class names to integer codes itself (section 4). |

### Developing and running the notebook

| Problem | Cause | Solution |
|---|---|---|
| `ValueError: array must not contain infs or NaNs` in Savitzky–Golay smoothing | No-data pixels are NaN and `savgol_filter` requires finite values. | Fill NaNs before filtering, then restore them afterwards (section 3). |
| Smoothing across the VNIR/SWIR detector overlap and the water-vapour gaps | EnMAP's wavelength list isn't monotonic (the SWIR detector starts at 902 nm, before the VNIR detector ends at 993 nm). | Split bands into contiguous segments, breaking at gaps and where the wavelength drops, and smooth each segment separately. |
| Sentinel-2 STAC search failed in a network-restricted environment | Outbound access to the Earth Search endpoint was blocked. | The notebook falls back to Sentinel-2 bands **simulated** from EnMAP and prints a warning; fusion results are then only a code test. On Colab the real scene loads (this run: `S2A_33UUU_20220720_0_L2A`). |
| 99 % of variance captured by only 3 PCs, too few as ML features | Brightness dominates the variance; class-relevant detail sits in smaller components. | Keep components up to 99.9 % variance, with a minimum of 10. |
| Very few pure pixels for roof and soil | Small urban objects rarely fill a 30 m pixel. | Purity threshold lowered from 80 % to 70 % (more samples); the vegetation-fraction regression added as a better fit for mixed pixels. |
| Section 11 (regression CV) took more than 6 minutes on Colab | `RandomForestRegressor` defaults to `max_features=1.0`, so every split tests all 210 bands; it trains 17 forests on Colab's 2 CPU cores. | `max_features=0.3` (≈ 1–2 min, similar accuracy). |
| `bash: syntax error near unexpected token` when downloading outputs in Colab | A line starting with `!` is run entirely by the shell, including the Python part after `&&`. | Put the `!zip …` shell command and the Python `files.download(…)` call on separate lines, or copy `outputs/` to Google Drive. |

## Repository structure

```text
EnMAP_S2_fusion_pipeline.ipynb   # executable tutorial / experimental workflow
README.md                        # research summary and reproduction notes
requirements.txt                 # dependency list
.gitignore                       # excluded large inputs and transient files
outputs/outputs/                 # optional small example CSV/JSON/model artifacts
```

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

## License and Data Attribution

The original Python code in this repository is released under the [MIT License](LICENSE).

The Earth Observation datasets used in this project are subject to their respective providers' licences and attribution requirements and are not relicensed under MIT.

## Data licences
The MIT licence applies to the code only. Data used or derived in this repository keep their original terms:
- EnMAP example data © DLR, distributed with the EnMAP-Box (https://www.enmap.org/)
- Land-cover reference derived from aerial imagery: GeoBasis-DE/LGB, dl-de/by-2-0
- Contains modified Copernicus Sentinel data (2022)

Third-party software, datasets and other externally sourced material retain their respective licences.

## Development note
The pipeline was developed with the help of an AI assistant (Claude, Anthropic). I ran, tested and adapted the workflow, checked the results, and am responsible for the interpretation.

This repository is intended for educational, research and methodological demonstration purposes.
