# Injection Moulding Part-Weight Prediction

This interactive dashboard tests a simple idea: **when predicting the weight of a new plastic part, does it help to train a model on geometrically similar parts instead of every available part?** It compares those two approaches on artificial injection-moulding datasets.

## The problem

Injection moulding produces a plastic part by filling a mould and letting the material cool. The resulting part weight is one measure of whether the process produced the intended amount of material. Weight can depend on the part's shape and size, the polymer, the process settings, and the machine.

A machine-learning model needs examples with known weights. For a new geometry or a new geometry–material combination, there may be few such examples. Existing data from other parts can be reused, but not every old part is equally relevant. This project asks whether selecting similar source cases makes weight prediction more accurate in that situation.

In this repository, a **source row** is an available training example; a **target row** is a held-out example whose weight the model tries to predict. A **neighbor** is a source row that passes the similarity threshold for a particular target. “Transfer” means applying knowledge from the source rows to a target geometry or combination excluded from training.

## What is being compared?

| Approach | Training rows used for a target prediction |
| --- | --- |
| **Global Random Forest** | All source training rows |
| **Local Random Forest** | Only source rows sufficiently similar to that target |

A Random Forest combines many decision trees to predict a numeric value. Both models here use the same **fixed forest settings**—300 trees, maximum depth 8, and minimum leaf size 2. Holding those settings constant makes the source-row selection the main model difference. The local approach still has another choice to make: **the similarity threshold**. Both models predict the same held-out target rows and are compared using mean absolute error (MAE), root mean squared error (RMSE), and R². Lower MAE and RMSE, and higher R², indicate better fit to those held-out weights.

## Data and transfer scenarios

The dashboard contains saved results from two artificial-data benchmarks. `model_data` has **123,200 fully synthetic rows** with geometry, material, process, and machine predictors. `quality_model_inputs` is a **1,200-row Moldflow-derived expansion**: 240 original Moldflow outputs with some assumed descriptors, plus 960 generated rows for additional geometry and material combinations. It has geometry, material, and process predictors. The outcome in both is part weight in grams.

Each dataset is examined in two ways:

1. **New geometry:** ribbed-plate examples are withheld as the target. The model learns from other geometries.
2. **New geometry–material pair:** ribbed plate with HDPE is withheld as the target. Ribbed plates with other materials and HDPE with other geometries can still occur in the source training set.

The data are separated into source training, validation, and test roles. Source rows train the models. A separate validation geometry (or geometry–material pair) is used to choose the local similarity threshold by its prediction error. The test geometry or pair is then used to evaluate both methods. This keeps the test results out of the threshold-selection step. No target test rows are added to source training in these comparisons.

## How similarity selects training rows

The local model repeats these steps for each target row:

1. **Choose comparison measurements.** For `model_data`, use cavity volume. For `quality_model_inputs`, use projected area, cavity volume, and surface area.
2. **Put measurements on comparable scales.** Subtract each measurement's mean in the source training data and divide by its standard deviation in that same data.
3. **Calculate distance.** Use ordinary Euclidean distance between the target and each source row on those standardized measurements. The selected measurements are treated equally; the code does not multiply them by importance weights.
4. **Turn distance into a score.** For each target, `similarity = 1 − distance / distance_to_the_farthest_source_row`. The closest source rows receive the highest scores.
5. **Apply the threshold.** Keep every source row with `similarity ≥ threshold` and train a local Random Forest on those rows. If none qualifies, use the single closest source row and report that fallback.

The similarity score is **relative to the source pool for that target**. A score of 0.90 does not mean two parts are 90% physically identical. At a higher threshold, fewer rows normally qualify. In the volume-only dataset, differently shaped parts with similar volumes may be treated as neighbors.

Similarity chooses **rows**, not the Random Forest's input columns. After selection, the local model still receives all available prediction columns: 55 in `model_data` or 35 in `quality_model_inputs`. Thus material and process information can affect the weight prediction even though they do not choose neighbors here.

## Why geometry-only similarity?

This dashboard is a **focused test of one part of a broader thesis method**. In exploratory permutation-importance checks on these artificial datasets, geometry measurements dominated the model's weight predictions. Permutation importance measures how much prediction performance drops when one input column is shuffled. The selected geometry measurements accounted for the first 80% of positive individual importance in that analysis, motivating a simpler neighbor-selection experiment.

The full thesis method calculates separate similarities for **geometry, material, and process**, then combines those category scores using weights derived from permutation importance. The thesis presentation illustrates weights of 0.89, 0.10, and 0.01 for those categories. **Those weights and the full combined similarity are not used in this dashboard.** In particular, the “new geometry–material pair” scenario changes what is held out for testing; it does not add material to the similarity calculation.

Geometry-only selection is a research choice to test, not a claim that material, process, or machine conditions are unimportant. The artificial datasets and correlated inputs can make permutation importance misleading about physical causation or performance on truly new parts.

## How to use the dashboard

Choose a dataset and transfer scenario, then move the similarity-threshold control from **0.70 to 0.995**. The RMSE, MAE, and R² charts show the global and local results at each saved threshold. Neighbor counts show how much source data the local model used; fallback counts show where no row passed the threshold. Selected target examples display their actual weight and both predictions.

The threshold charts are a **post-hoc view of test results** at several thresholds, including settings other than the one chosen on validation. They help illustrate how neighbor count and prediction error changed, but picking the best point after seeing this chart would overstate performance. The formal comparison uses the threshold selected on validation before test scoring. Moving the control displays precomputed results; it does not retrain a model or predict a newly entered part.

## Scope and viewing

These datasets were created for **personal testing of the modelling method**. Their results are not independent physical experiments, evidence that a particular model will work on a factory line, or a replacement for measured validation data. This static package contains the dashboard and saved summary results, but not the training scripts, full synthetic datasets, or original row-level Moldflow and ProBayes files.

The interactive dashboard is `index.html`. Open it in a browser after downloading the repository, or use the GitHub Pages link if the repository has been published as a site. Some interface libraries and fonts load from public CDNs, so an internet connection is needed for the intended appearance and behavior.
