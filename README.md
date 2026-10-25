# EnMAP + Sentinel-2 fusion for urban land cover (Potsdam)

A reproducible Python workflow that classifies land cover and estimates sub-pixel vegetation cover from **EnMAP hyperspectral** imagery, **Sentinel-2** multispectral imagery, and a **feature-level fusion** of both. The three are compared under **spatial cross-validation**.

**Author:** Pallavi · **Status:** learning project / method demonstrator · **Runs in:** Google Colab or Jupyter (≈ 10 min)

---

## Aim

1. Build an end-to-end hyperspectral workflow: load the data, pre-process it, derive features, apply machine learning, and export the results.
2. Bring a second mission onto the same analysis-ready grid and check how consistent the two sensors are.
3. Measure whether fusing EnMAP with Sentinel-2 improves on either sensor alone, for (a) land-cover classification and (b) vegetation-fraction regression.

## Data

| Dataset | Details | Source / licence |
|---|---|---|
| EnMAP L2A surface reflectance | Potsdam, 24 Jul 2022 10:45 UTC · 224 bands (418–2446 nm) · 30 m · 330 × 200 px | EnMAP-Box example data, © DLR |
| Land-cover reference polygons | 9,677 polygons digitised on 1 m aerial imagery (2019) · 6 classes: roof, pavement, low vegetation, soil, tree, water | EnMAP-Box example data; GeoBasis-DE/LGB, dl-de/by-2-0 |
| Sentinel-2 L2A | `S2A_33UUU_20220720_0_L2A` · 20 Jul 2022 (−5 days from EnMAP) · tile cloud cover 1.6 %, **100 % clear over the study area** | Copernicus Sentinel data 2022, via Element84 Earth Search (AWS open data) |

All data is accessed without credentials.

## Method

```
EnMAP cube ─► bad-band & water-vapour removal (210/224 bands kept) ─► Savitzky–Golay smoothing
          ─► features: all bands │ PCA (10 components) │ narrow-band indices (NDVI, NDRE, NDWI, NDBI, PRI, red-edge position)

Sentinel-2 ─► STAC search ±25 days ─► SCL cloud mask ─► 10 bands resampled to the EnMAP 30 m grid (area average)
          ─► features: 10 bands + NDVI, NDRE, NDWI, NDBI

Reference polygons ─► rasterised at 3 m ─► class fractions per 30 m pixel
          ├─► classification samples: pixels ≥ 95 % covered and ≥ 70 % one class  (2,769 px)
          └─► regression target: vegetation fraction (low vegetation + tree)       (5,469 px)

Random Forest ─► 4-fold spatial cross-validation (600 m × 600 m blocks; whole blocks held out)
```

**Why spatial blocks?** Neighbouring pixels are strongly autocorrelated, so a random pixel split puts near-duplicates into both training and test sets and inflates accuracy. Holding out whole blocks gives an estimate closer to performance on unseen areas.

**Fusion** here is feature-level stacking: EnMAP PCA components and indices are concatenated with the Sentinel-2 bands and indices. All four feature sets use the same samples and the same folds, so they can be compared directly.

## Results

### Sensor consistency

 band      r   bias   RMSE
0  B02  0.873 -0.100  0.101
1  B03  0.878 -0.099  0.101
2  B04  0.909 -0.098  0.100
3  B05  0.912 -0.097  0.098
4  B06  0.955 -0.103  0.106
5  B07  0.963 -0.109  0.112
6  B08  0.956 -0.114  0.118
7  B8A  0.968 -0.111  0.115
8  B11  0.950 -0.105  0.108
9  B12  0.941 -0.099  0.101
estimated shift S2 vs EnMAP: +0.30 rows, -0.30 cols (= +9 m, -9 m)

### Land-cover classification (6 classes, spatial CV)

S2 (bands+indices)     OA=0.913 kappa=0.867 macroF1=0.630
EnMAP PCA              OA=0.867 kappa=0.793 macroF1=0.570
EnMAP all bands        OA=0.876 kappa=0.809 macroF1=0.595
Fused EnMAP+S2         OA=0.916 kappa=0.871 macroF1=0.616

Per-class F1:

                S2 (bands+indices)  EnMAP PCA  EnMAP all bands  Fused EnMAP+S2
low vegetation                0.91       0.82             0.85            0.91
pavement                      0.74       0.64             0.70            0.74
roof                          0.07       0.00             0.07            0.05
soil                          0.11       0.09             0.07            0.04
tree                          0.98       0.95             0.95            0.98
water                         0.98       0.92             0.93            0.98



### Vegetation-cover fraction (regression, spatial CV)

                       R2   RMSE
features                        
S2 (bands+indices)  0.875  0.126
EnMAP PCA           0.786  0.165
EnMAP all bands     0.799  0.160
Fused EnMAP+S2      0.880  0.124

## Interpretation

**1. Fusion gave no measurable gain over Sentinel-2 alone.** The fused model improves overall accuracy by 0.3 percentage points and R² by 0.004, while its macro F1 is slightly lower. These differences are smaller than the variation expected between CV folds, which was not quantified in this run. They should be read as "no difference" rather than an improvement.

**2. In this setup, Sentinel-2 alone outperformed EnMAP alone** (OA +3.7 points, R² +0.08). The analysis doesn't isolate the cause. Plausible explanations to test:
- **Spatial quality at 30 m.** Sentinel-2's 10/20 m bands were area-averaged to 30 m, which gives a sharp, well-defined pixel footprint. EnMAP's native 30 m pixels have a broader point-spread function and lower signal-to-noise in parts of the SWIR. In a fine-grained urban mosaic, that blurring mixes neighbouring surfaces.
- **Geometric alignment with the reference.** The labels come from orthorectified aerial imagery. A sub-pixel offset of either sensor relative to the labels hurts that sensor's scores (see the shift estimate in section 9).
- **Task fit.** The six broad classes are separable from visible–NIR–SWIR contrasts that a multispectral sensor captures well. EnMAP's spectral detail matters most for questions this task doesn't ask, such as roof materials, vegetation traits or stress, soil properties, and crop types.
- **Feature-to-sample ratio.** 210 highly correlated bands with ~2,000 training pixels per fold. PCA did not help here, which suggests discriminative information sits in low-variance components.

**3. Roof and soil are effectively not mapped by any feature set** (F1 ≤ 0.11). This reflects the data rather than the model. Both classes have very few pure 30 m pixels (roofs and bare-soil patches are mostly smaller than a pixel), and both are spectrally close to pavement, which absorbs most of their predictions. Water, tree and low vegetation are mapped reliably (F1 0.82–0.98).

**4. Sub-pixel vegetation fraction is the strongest product** (R² 0.88, RMSE 0.12). Treating mixed 30 m urban pixels as fractions rather than forcing a single class is a better fit to the data. The scatter plot shows the typical Random Forest pattern of overestimating low fractions and underestimating high ones (shrinkage toward the mean).

## Limitations

- One EnMAP scene, one city, one date. The results do not generalise beyond this setting.
- The reference was digitised from 2019 aerial imagery, while the satellite data is from 2022, so some land-cover change is possible.
- No uncertainty across folds or repeated runs was reported, so small differences between feature sets cannot be called significant.
- Only feature-level fusion with Random Forest was tested. Pixel-level fusion (sharpening) and other models were not.

## Next steps

- Report fold-wise mean ± standard deviation and use a paired test to compare feature sets.
- Test tasks that use spectral detail: roof-material mapping, vegetation traits, and crop-type or crop-condition mapping.
- Add a Sentinel-2 **time series** (phenology) alongside the single EnMAP scene.
- Sharpen EnMAP to 10 m using Sentinel-2 (pixel-level fusion).
- Apply the workflow to other EnMAP scenes and regions.

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

## Repository contents

```
EnMAP_S2_fusion_pipeline.ipynb         end-to-end notebook (sections 0-13)
outputs/
  class_enmap.tif, class_fused.tif     classification maps (uint8, see classes.json)
  vegetation_fraction.tif              predicted vegetation cover (0-1)
  s2_on_enmap_grid.tif                 Sentinel-2 L2A resampled to the EnMAP grid
  enmap_pca.tif, enmap_indices.tif     derived EnMAP features
  rf_fused_classifier.joblib           trained model
  accuracy_*.csv                       result tables
```

## How to run

Open the notebook in Colab (*File → Upload notebook*) or Jupyter and run all cells. The data downloads automatically. All settings are in the `CFG` dictionary in section 0.

## Acknowledgements

EnMAP-Box team (Humboldt-Universität zu Berlin, GFZ) for the example dataset; DLR for EnMAP; ESA/Copernicus for Sentinel-2; Element84 for the Earth Search STAC catalogue. Built with rasterio, GeoPandas, scikit-learn and pystac-client.
