# More-Steps-More-Calories-
Exploratory data analysis and predictive modeling on data from food.com's recipes, examining how recipe complexity relates to nutritional content. Final project for DSC 80 at UCSD. 

# More Steps, More Calories?

Exploratory data analysis and predictive modeling on data from food.com's recipes, examining how recipe complexity relates to nutritional content. Final project for DSC 80 at UCSD.

## Step 1: Introduction

The recipes data is a collection of 83,782 recipes posted to the Food.com website since 2008. The columns relevant to this analysis are:

- `name` — recipe name
- `minutes` — minutes to prepare
- `nutrition` — calories (absolute), and total fat, sugar, sodium, protein, saturated fat, and carbohydrates (all as % of daily value)
- `n_steps` — number of steps
- `n_ingredients` — number of ingredients

The Interactions dataset holds more than 700,000 reviews and ratings for the recipes in the recipes dataset.

**Question:** Does the number of steps in a recipe relate to its nutritional content, and does it explain the pattern?

Coming home after work or school can leave you exhausted and ready to eat. Cooking can involve many steps, so people often gravitate toward easier recipes, although some go the extra mile and choose more involved recipes. This project explores the number of steps in a recipe and how that relates to its nutritional profile, focusing specifically on main-dish recipes and their calorie and fat content.

## Step 2: Data Cleaning and Exploratory Data Analysis

### Univariate Analysis

<iframe src="assets/univariate-steps.html" width="800" height="600" frameborder="0"></iframe>

The distribution of recipe steps is strongly right-skewed. It peaks between 5 and 10 steps, which indicates that most recipes in the dataset are simple and concise, with only a tiny fraction demanding complex processes exceeding 20 to 30 steps.

<iframe src="assets/univariate-ingredients.html" width="800" height="600" frameborder="0"></iframe>

### Bivariate Analysis

<iframe src="assets/bivariate-calories-steps.html" width="800" height="600" frameborder="0"></iframe>

The median calorie count shows a slight upward trend as recipe complexity increases, rising from around 350 calories for 1–5 step recipes to about 500 calories for those with 21+ steps. While more involved recipes tend to be slightly higher in calories, the bulk of recipes across all step categories remain under 1,000 calories, with a significant number of high-calorie outliers present in every group.

<iframe src="assets/bivariate-fat-steps.html" width="800" height="600" frameborder="0"></iframe>

### Interesting Aggregates

[Paste your `ingredient_summary.to_markdown()` output here]

Across all ingredient buckets, every nutritional metric — including calories, sugar, fat, sodium, and carbs — consistently increases alongside ingredient count. Most recipes cluster within the 6–15 ingredient range, with very simple (1–5) or highly complex (21+) recipes being less common.

## Step 3: Assessment of Missingness

The Interactions dataset contains a column, `rating`, that uses a 1–5 star scale. However, some reviews have a rating of 0, which makes up about 51,000 of the 822,681 total ratings. The 0 cannot represent "0 stars" — it represents a review left without a star rating at all. Since the reason for the missing rating relates to the value that would have been reported, we classify these missing values as **MNAR**. This can happen when reviewers leave comments or questions without actually rating the recipe, and we treat those missing values as `NaN` rather than as a rating of 0, since treating them as 0 would incorrectly decrease a recipe's average rating.

We tested whether the missingness of `avg_rating` — which occurs for main-dish recipes where every review left a rating of 0 (treated as "no rating given," not "0 stars") — depends on other recipe characteristics, using permutation tests with the difference in group means as our test statistic.

**Does missingness depend on `n_steps`?** We found an observed difference in mean `n_steps` of 0.611 between recipes with a missing `avg_rating` and those without, with a p-value of 0.013. Since this is below our significance threshold of 0.05, we reject the null hypothesis that missingness is unrelated to `n_steps` — recipes with a higher number of steps are somewhat more likely to have a missing average rating.

**Does missingness depend on `n_ingredients`?** We found an observed difference of 0.219, with a p-value of 0.159. Since this is well above 0.05, we fail to reject the null hypothesis — the number of ingredients in a recipe does not appear to meaningfully predict whether its `avg_rating` will be missing.

<iframe src="assets/missingness-permutation.html" width="800" height="600" frameborder="0"></iframe>

These results suggest the missingness of `avg_rating` is best described as **Missing at Random (MAR)** with respect to `n_steps`, but not with respect to `n_ingredients`. This means we cannot treat `avg_rating`'s missingness as purely random noise, since it's tied to at least one observable recipe characteristic — more complex, multi-step recipes are somewhat more likely to receive comment-only reviews without a star rating. Reviewers of more involved recipes may ask questions or leave troubleshooting comments (perhaps because something didn't go as expected) without actually rating the dish.

## Step 4: Hypothesis Testing

**Null hypothesis (H₀):** Among main-dish recipes, there is no difference in mean calorie content between recipes with more steps (above the median of 10) and recipes with fewer steps (at or below the median); any observed difference is due to random chance.

**Alternative hypothesis (H₁):** Among main-dish recipes, recipes with more steps have a higher mean calorie content than recipes with fewer steps.

**Test statistic:** Difference in mean calorie content between the two groups (high-step minus low-step).

**Significance level:** α = 0.05

We split recipes into two groups using the median number of steps (10) as the cutoff, then ran a permutation test with 1,000 trials, shuffling the group labels and recomputing the difference in means each time to build an empirical null distribution. Our observed difference — recipes with more steps averaged 80.35 more calories than recipes with fewer steps — did not occur in any of the 1,000 random permutations, giving a p-value of approximately 0 (p < 0.001).

Since this p-value is far below our significance threshold of 0.05, we reject the null hypothesis. This provides strong evidence that recipes with more steps tend to have higher calorie content than recipes with fewer steps, among main-dish recipes. It's important to note that this test doesn't prove causation or establish this as an absolute fact — it shows the observed pattern is very unlikely to have arisen by random chance alone, which supports (but does not conclusively prove) a genuine relationship between recipe complexity and calorie content.

## Step 5: Framing a Prediction Problem

Our prediction problem is to predict the calorie content of a main-dish recipe using its structural characteristics. This is a **regression** problem, since calories is a continuous numeric value rather than a category.

**Response variable:** `calories`. We chose this variable because it was the nutrition measure most strongly associated with recipe complexity throughout our earlier analysis — it had the strongest correlation with both `n_steps` (0.182) and `n_ingredients` (0.177) among all the nutrition variables we examined, and our Step 4 hypothesis test confirmed a statistically significant relationship between step count and calorie content. Predicting calories directly builds on the relationship we spent the entire project investigating.

**Evaluation metric:** We use RMSE (Root Mean Squared Error) rather than R² or MAE alone. We chose RMSE because it is measured in the same units as our target variable (calories), making model performance directly interpretable — a reader can understand "the model's predictions are off by roughly X calories on average" far more intuitively than a unitless R² score. RMSE also penalizes large prediction errors more heavily than MAE would, which is appropriate for this problem: seriously underestimating a very calorie-dense recipe is a more meaningful failure than being slightly off on a typical one. (Note: accuracy and F1-score, mentioned as example metrics in the assignment, don't apply here since those are classification metrics, and this is a regression problem.)

## Step 6: Baseline Model

**Model description:** The baseline model is a linear regression predicting calories from two features drawn from the original dataset: `n_steps` and `n_ingredients`.

**Features and their types:** Both features are quantitative — `n_steps` (number of preparation steps) and `n_ingredients` (number of ingredients) are both counts, not categories or rankings, so no ordinal or nominal features are present in this baseline.

**Encodings performed:** Since both features are quantitative, no categorical encoding was necessary. We applied `StandardScaler` to both features, transforming them to have mean 0 and standard deviation 1. This isn't an encoding in the categorical sense, but a numerical transform necessary because linear regression coefficients are sensitive to the scale of input features, and `n_steps` and `n_ingredients` have different natural ranges.

**Performance:** We evaluated the model using RMSE, computed separately on training data and a held-out test set (an 80/20 split) to check how well the model generalizes to unseen recipes:
- Train RMSE: 259.07
- Test RMSE: 258.73

The near-identical train and test RMSE values indicate the model is not overfitting — it performs consistently on data it hasn't seen, which is a good sign for its reliability, even though its predictions aren't very precise.

**Is this a "good" model?** No — we don't consider this baseline strong. An average error of roughly 259 calories is large relative to a typical main-dish recipe (medians across our earlier analysis ranged from about 340 to 540 calories depending on complexity), meaning the model's predictions are often off by more than half a typical recipe's calorie count. This isn't surprising: our EDA found that `n_steps` and `n_ingredients` individually have only weak correlations with calories (0.18 each), so two features alone were unlikely to explain most of the variation. We view this baseline as a reasonable, honest starting point — it captures a real, statistically significant relationship (confirmed in Step 4), but there's clear room for improvement, which we address in the Final Model.

## Step 7: Final Model

**Features added and why they're good for this task:** We added two new features beyond our baseline: `minutes` and `contains_meat`.

We included `minutes`, transformed with a `QuantileTransformer` to normalize its skewed distribution, because it captures a different dimension of "effort" than `n_steps` and `n_ingredients` do. A recipe can involve many discrete steps and ingredients while cooking quickly (a stir-fry), or involve very few steps while taking a long time (a slow-braised roast). Cooking duration often reflects the technique being used — roasting, reducing, slow-cooking — and these techniques can concentrate fat and calories (through moisture loss or fat rendering) in ways that step or ingredient count alone wouldn't capture. This is a data-generating-process reason to expect `minutes` to add distinct information, not something we noticed only after it improved our score.

We also added `contains_meat`, a binary feature engineered from each recipe's tags, because protein source is one of the most direct drivers of calorie and fat density in a dish, independent of how complex the recipe is to prepare. A meat-based main dish is inherently more likely to be calorie-dense than a vegetable- or legume-based one, regardless of how many steps or ingredients it involves — this reflects a real property of the ingredients themselves, not an artifact of our modeling process.

**Modeling algorithm and hyperparameter selection:** We compared three algorithms on the same features and the same train/test split as our baseline: Random Forest, Gradient Boosting, and Ridge Regression. For the tree-based models, we tuned `max_depth` via `GridSearchCV` with 5-fold cross-validation, searching over `[3, 5, 10, 15, 20, None]` for Random Forest and `[2, 3, 5, 7]` for Gradient Boosting; for Ridge, we tuned `alpha` over `[0.1, 1, 10, 100]`. We chose this search approach because it evaluates each candidate hyperparameter value using cross-validation on the training set only, keeping the test set completely untouched until final evaluation.

The best-performing configurations were:
- Random Forest: `max_depth=5` — test RMSE 256.22
- Gradient Boosting: `max_depth=2` — test RMSE 256.24
- Ridge: `alpha=10` — test RMSE 256.71

Our final model is the Random Forest Regressor with `max_depth=5`, since it achieved the lowest test RMSE among the three, though the margin over the alternatives was small. This suggests our chosen features — not model complexity — are the primary constraint on predictive performance.

**Improvement over baseline:** Our baseline model (linear regression on `n_steps` and `n_ingredients` alone) achieved a test RMSE of 258.73. Our final model improves on this, achieving a test RMSE of 256.22 — a modest but real improvement, consistent with the generally weak correlations found throughout our EDA. The close match between our final model's train RMSE (254.54) and test RMSE (256.22) also confirms it isn't overfitting despite the added feature complexity.

## Step 8: Fairness Analysis

**Group X:** Vegetarian main-dish recipes (n = 2,627, tagged `vegetarian`)
**Group Y:** Non-vegetarian main-dish recipes (n = 21,691)

**Evaluation metric:** RMSE (since our final model is a regression model, precision/recall don't apply, per the assignment's own note).

**Null hypothesis (H₀):** Our model is fair. Its RMSE for vegetarian recipes and non-vegetarian recipes are roughly the same, and any observed difference is due to random chance.

**Alternative hypothesis (H₁):** Our model is unfair. Its RMSE is higher for non-vegetarian recipes than for vegetarian recipes.

**Test statistic:** Difference in RMSE between groups (non-vegetarian RMSE − vegetarian RMSE), evaluated via permutation test (shuffling group labels on the test set's squared residuals, 1,000 trials).

**Significance level:** α = 0.05

**Results:**
- RMSE (vegetarian): 227.63
- RMSE (non-vegetarian): 259.53
- Observed difference: 31.91
- p-value: 0.003

Since p = 0.003 is well below α = 0.05, we reject the null hypothesis. This provides strong evidence that our model performs worse (higher error) on non-vegetarian recipes than on vegetarian ones. This could be because non-vegetarian dishes likely have more variable calorie content — different meats, preparation methods (fried vs. grilled), and portion styles introduce more nutritional variability than the more similar plant-based dishes typically seen in the vegetarian category, making calories genuinely harder to predict from step/ingredient/time/meat-presence features alone. As with our earlier hypothesis test, we frame this as evidence of a real disparity, not proof of an absolute, unchanging fact about the model.