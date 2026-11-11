# eminonu_ve_ilkerden_sonra — Re-analysis of the artificial intelligence prediction module

This folder is entirely independent of the previous `keskin_*` files; it contains its own modules (`eis_*.py`) and its own output directory (`ciktilar_eminonu_ve_ilkerden_sonra`).

## Number of replicates: 3 or 8 (version 2.1)

- The number of replicates is read automatically from the data (`BEKLENEN_TEKRAR = None` in `eis_config.py`). No manual changes are required.
- **3 replicates (90 observations):** follows the previous "2 + 1" configuration: 63 Train / 14 Validation / 13 Test.
- **8 replicates (240 observations):** 1 Validation and 1 Test sample per condition, with the remaining 6 used for training: 186 Train / 27 Validation / 27 Test. Control samples are always included in the training set.
- Notes regarding data quality checks (e.g., high CV, potential outliers) are recorded in `EIS_VERI_KONTROL.txt`; the analysis will proceed without interruption, and no observations are removed.
- The Excel file format must remain consistent: the sheet must be named `mtt_doz_time`, with columns labeled `ilac`, `time`, `doz`, and `mtt`. If you place the file in the code directory and name it `mtt_verisi.xlsx`, the `--veri` argument is not required.
- Models will take longer to fit with 240 observations. For an initial trial, consider using `--mc 10 --ga_populasyon 20 --ga_nesil 20`, or specify a subset of models using the `--modeller` argument. ## Models and model selection (version 2.0)

- **Candidates:** All models from the previous code are supported: MLR, Ridge, Lasso, Elastic Net, SVR, SVR-RFE, GPR, KRR, kNN, MLP, Random Forest, Extra Trees, XGBoost, monotonic XGBoost, LightGBM, CatBoost, T1-FRF, IT2-FRF, and ANFIS.
- **New recommended model, Spline-GAM** (`eis_yeni_modeller.py`): A penalized regression spline. It establishes a smooth dose-response curve for each treatment and incorporates a time interaction. It continues linearly beyond the highest dose and does not enforce monotonicity.
- **Mechanistic reference:** A time-dependent 4PL surface is always reported. Additionally, curve-based 4PL, isotonic regression, and linear interpolation are provided as references.
- **Selection ("best-performing models for the paper") is now an integral part of the validated procedure.** In each partition (fixed, 30 random assignments, 3 LODO folds, and the final set), only the training data for that specific partition is used:
1. A genetic algorithm (GA) is run for each candidate model. 
2. The top `k` models (default: 4) are selected based on the RMSE from an internal "dose-out" (inner LODO) cross-validation. 
3. The selected models are combined in two ways: by averaging them ("Consensus of the selected models") and by creating an NNLS-weighted mixture of the top 6 models ("Stacking"). 

Consequently, the selection process never "sees" the held-out wells or doses; thus, any optimism bias inherent in the selection is accounted for in the reported error. - **Comparison:** The consensus of the fixed quartet (GPR, KRR, SVR, IT2-FRF) from the article is also reported.
- **Location:** Details on which model was selected for each fold can be found in the `T13` and `T14` sheets; a table of all candidate models is provided in the `MAKALE_TableS2_tum_modeller` sheet and in the `FigS` figures.

Duration: 35 folds × 20 candidates = 700 GA (Genetic Algorithm) runs. With full settings, this may take up to a day using 8 workers. For a shorter trial, you can use, for example, `--mc 10` or `--modeller "GPR,KernelRidge,SVM,Type2-FRF,Lasso,ElasticNet,Spline-GAM"`. Thanks to the caching mechanism, an interrupted run can resume from where it left off.

## Execution

```
conda activate mtt_analiz
cd /d "C:\...\eminonu_ve_ilkerden_sonra"
chcp 65001
python eminonu_ve_ilkerden_sonra.py --ornek --hizli          (synthetic trial, ~1 min; figures are NOT real)
python eminonu_ve_ilkerden_sonra.py --veri "C:\...\mtt_verisi.xlsx" --isci 8      (actual analysis)
```

- `--mc 30`: Number of random "2 + 1" assignments (default is 30).
- `--bootstrap 500`: Number of bootstrap iterations for the 95% confidence interval (CI) of the mean response.
- `--isci 8`: Number of parallel workers.

Duration: The GA runs repeatedly across 35 folds × 4 models (1 fixed fold + 30 random assignments + 3 LODO [Leave-One-Dataset-Out] folds + 1 final run). It may take several hours with 8 workers. If interrupted, re-run the same command; completed runs will be retrieved from the `cache` folder.

## Feedback → implemented changes

| Item | Action taken |
|---|---|
| Dose unit | µM (verified against raw data; values ​​remain the same). |
| LODO | Each tested dose (0.5, 1, 2 µM) is entirely excluded from the training set across all treatments and durations. Results for 0.5 and 1 µM are reported as interpolation, while 2 µM is reported as extrapolation. The Genetic Algorithm (GA) is re-run for each fold. |
| Leakage | SPXY has been removed; random splitting with a fixed seed is used. The GA is re-run for each of the 30 assignments (the previous version used hyperparameters selected on a fixed training set). Scaling is integrated into the model pipeline. The test set is never used for any selection process. Detail: `T11`. |
| Prediction intervals | 95% confidence intervals (CI) for mean viability (using bias-corrected bootstrapping) and 95% prediction intervals (PI) for an individual new well are provided separately. The PI is calculated using a conformal method and calibrated separately for interpolation and extrapolation based on LODO residuals. Model dispersion (s) is no longer incorporated into the PI. The lower bound is truncated at 0; the upper bound is not truncated. Coverage and width are reported separately (`T5`). |
| Monotonicity | For each model and the observed means, the maximum increase, noise threshold, and Kendall’s trend test are reported. Monotonicity is not enforced; monotonic reference models (isotonic, 4PL) are compared using the same validation procedures. |
| Model comparison | Standard R² (1 − SSE/SST), RMSE, MAE, and MBE are reported. Paired differences are presented with 95% CIs: Nadeau–Bengio correction for MC, bootstrap with clustered conditions for LODO; Holm correction is applied. |
| Consensus | Average of the four models Each validation is compared against models fitted specifically to that validation set. |
| ATI and ATI+PLA | Differences and 95% CIs are calculated for each dose and duration (`T6`, Fig. C). |
| IC50 | Reported only if within the 0.5–2 µM range; values ​​outside this range are denoted as "> 2 µM". Bootstrap stability is also provided (`T8`). |
| Independence | Summarized at the level of the 30 assignments, using adjusted variance. Replicates are treated as technical replicates (wells on the same plate). |
| Reproducibility | `T12` and `EIS_REPRODUCIBILITY.json`: versions, random seeds, data SHA-256 hashes, and hyperparameters. |

## Outputs

- `EIS_RESULTS.xlsx`: `ARTICLE_Table1`, `ARTICLE_Table2`, and `T1`–`T12`. Descriptions for each sheet can be found in the `README` (or `GUIDE`) sheet. |
- `EIS_RAW_PREDICTIONS.xlsx`: Predictions for each held-out well across every run. |
- `EIS_SUMMARY.txt`: Key metrics. |
- `figures/`:
- Fig. A: Performance across four validations. 
- Fig. B: CIs and PIs for new doses. 
- Fig. C: Difference between ATI+PLA and ATI.