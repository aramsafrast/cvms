
<!-- README.md is generated from README.Rmd. Please edit that file -->

# cvms <a href='https://github.com/LudvigOlsen/cvms'><img src='man/figures/cvms_logo_242x280_250dpi.png' align="right" height="140" /></a>

**Cross-Validation for Model Selection**  
**Authors:** [Ludvig R. Olsen](http://ludvigolsen.dk/) (
<r-pkgs@ludvigolsen.dk> ), Hugh Benjamin Zachariae <br/> **License:**
[MIT](https://opensource.org/licenses/MIT) <br/> **Started:** October
2016

[![CRAN\_Status\_Badge](https://www.r-pkg.org/badges/version/cvms)](https://cran.r-project.org/package=cvms)
[![metacran
downloads](https://cranlogs.r-pkg.org/badges/cvms)](https://cran.r-project.org/package=cvms)
[![minimal R
version](https://img.shields.io/badge/R%3E%3D-3.5-6666ff.svg)](https://cran.r-project.org/)
[![Codecov test
coverage](https://codecov.io/gh/ludvigolsen/cvms/branch/master/graph/badge.svg)](https://codecov.io/gh/ludvigolsen/cvms?branch=master)
[![Travis build
status](https://travis-ci.org/LudvigOlsen/cvms.svg?branch=master)](https://travis-ci.org/LudvigOlsen/cvms)
[![AppVeyor build
status](https://ci.appveyor.com/api/projects/status/github/LudvigOlsen/cvms?branch=master&svg=true)](https://ci.appveyor.com/project/LudvigOlsen/cvms)
[![DOI](https://zenodo.org/badge/71063931.svg)](https://zenodo.org/badge/latestdoi/71063931)

## Overview

R package: Cross-validate one or multiple regression or classification
models and get relevant evaluation metrics in a tidy format. Validate
the best model on a test set and compare it to a baseline evaluation.
Perform hyperparameter tuning with (sampled) grid search. Evaluate
predictions from an external model. Currently supports regression
(`'gaussian'`), binary classification (`'binomial'`), and (some
functions only) multiclass classification (`'multinomial'`).

Main functions:

  - `cross_validate()`  
  - `cross_validate_fn()`  
  - `validate()`  
  - `validate_fn()`  
  - `evaluate()`
  - `baseline()`  
  - `combine_predictors()`  
  - `cv_plot()`  
  - `select_metrics()`  
  - `reconstruct_formulas()`

## Table of Contents

  - [cvms](#cvms)
      - [Overview](#overview)
          - [The difference between `cross_validate()` and
            `cross_validate_fn()`](#diff-cv-fn)
      - [Important News](#news)
      - [Installation](#installation)
  - [Examples](#examples)
      - [Attach packages](#packages)
      - [Load data](#load-data)
      - [Fold data](#fold)
      - [Cross-validate a single model](#cv-single)
          - [Gaussian](#cv-single-gaussian)
          - [Binomial](#cv-single-binomial)
      - [Cross-validate multiple models](#cv-multi)
          - [Create model formulas](#cv-multi-formulas)
          - [Cross-validate fixed effects models](#cv-multi-fixed)
          - [Cross-validate mixed effects models](#cv-multi-mixed)
      - [Repeated cross-validation](#cv-repeated)
      - [Cross-validating custom model functions](#cv-custom)
          - [SVM](#cv-custom-svm)
          - [Naive Bayes](#cv-custom-naive)
      - [Evaluating predictions](#evaluate)
          - [Multinomial](#evaluate-multinomial)
      - [Baseline evaluations](#baseline)
          - [Gaussian](#baseline-gaussian)
          - [Binomial](#baseline-binomial)
          - [Multinomial](#baseline-multinomial)
      - [Plot results](#plot)
          - [Gaussian](#plot-gaussian)
      - [Generate model formulas](#generate-formulas)

### The difference between `cross_validate()` and `cross_validate_fn()`

Originally, `cvms` only provided the option to cross-validate Gaussian
and binomial regression models, fitting the models internally with the
`lm()`, `lmer()`, `glm()`, and `glmer()` functions. The
`cross_validate()` function has thus been designed specifically to work
with those functions.

To allow cross-validation of custom model functions like support-vector
machines, neural networks, etc., the `cross_validate_fn()` function has
been added. You provide a model function and a predict function, and it
does the rest (see examples below). Additionally, you can provide a
preprocess function and a list or data frame with hyperparameter values
to test.

## Important News

Note: Check NEWS.md for the full list of changes.

  - In `cross_validate()` and `validate()`, the `models` argument is
    renamed to `formulas` and the `model_verbose` argument is renamed to
    `verbose`. Further, the `link` argument is hard-deprecated and will
    throw an error if used.

  - `Multinomial` AUC is now calculated with `pROC::multiclass.roc`.

  - `cross_validate_fn()` and `validate_fn()` now take the
    `preprocess_fn`, `preprocess_once`, and `hyperparameters` arguments.
    The `predict_type` argument has been removed.

  - `cross_validate_fn()` and `validate_fn()` are added.
    (Cross-)validate custom model functions.

  - In `evaluate()`, when `type` is `multinomial`, the output is now a
    single tibble. The `Class Level Results` are included as a nested
    tibble.

  - Adds `'multinomial'` family to `baseline()` and `evaluate()`.

  - `evaluate()` is added. Evaluate your model’s predictions with the
    same metrics as used in `cross_validate()`.

  - `Binomial` AUC calculation has changed. Now explicitly sets the
    direction in `pROC::roc`. (27th of May 2019)

  - Argument `positive` now defaults to `2`. If a dependent variable has
    the values 0 and 1, 1 is now the default positive class, as that’s
    the second smallest value. If the dependent variable is of type
    `character`, it’s in alphabetical order.

## Installation

CRAN:

> install.packages(“cvms”)

Development version:

> install.packages(“devtools”)
> 
> devtools::install\_github(“LudvigOlsen/groupdata2”)
> 
> devtools::install\_github(“LudvigOlsen/cvms”)

# Examples

## Attach packages

``` r
library(cvms)
library(groupdata2) # fold() partition()
library(knitr) # kable()
library(dplyr) # %>% arrange()
library(ggplot2)
```

## Load data

The dataset `participant.scores` comes with cvms.

``` r
data <- participant.scores
```

## Fold data

Create a grouping factor for subsetting of folds using
`groupdata2::fold()`. Order the dataset by the folds.

``` r
# Set seed for reproducibility
set.seed(7)

# Fold data 
data <- fold(data, k = 4,
             cat_col = 'diagnosis',
             id_col = 'participant') %>% 
  arrange(.folds)

# Show first 15 rows of data
data %>% head(15) %>% kable()
```

| participant | age | diagnosis | score | session | .folds |
| :---------- | --: | --------: | ----: | ------: | :----- |
| 9           |  34 |         0 |    33 |       1 | 1      |
| 9           |  34 |         0 |    53 |       2 | 1      |
| 9           |  34 |         0 |    66 |       3 | 1      |
| 8           |  21 |         1 |    16 |       1 | 1      |
| 8           |  21 |         1 |    32 |       2 | 1      |
| 8           |  21 |         1 |    44 |       3 | 1      |
| 2           |  23 |         0 |    24 |       1 | 2      |
| 2           |  23 |         0 |    40 |       2 | 2      |
| 2           |  23 |         0 |    67 |       3 | 2      |
| 1           |  20 |         1 |    10 |       1 | 2      |
| 1           |  20 |         1 |    24 |       2 | 2      |
| 1           |  20 |         1 |    45 |       3 | 2      |
| 6           |  31 |         1 |    14 |       1 | 2      |
| 6           |  31 |         1 |    25 |       2 | 2      |
| 6           |  31 |         1 |    30 |       3 | 2      |

## Cross-validate a single model

### Gaussian

``` r
CV1 <- cross_validate(data, 
                      formulas = "score~diagnosis",
                      fold_cols = '.folds',
                      family = 'gaussian',
                      REML = FALSE)

# Show results
CV1
#> # A tibble: 1 x 19
#>    RMSE   MAE   r2m   r2c   AIC  AICc   BIC Predictions Results
#>   <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <list>      <list> 
#> 1  16.4  13.8 0.271 0.271  195.  196.  198. <tibble [3… <tibbl…
#> # … with 10 more variables: Coefficients <list>, Folds <int>, `Fold
#> #   Columns` <int>, `Convergence Warnings` <int>, `Singular Fit
#> #   Messages` <int>, `Other Warnings` <int>, `Warnings and
#> #   Messages` <list>, Family <chr>, Dependent <chr>, Fixed <chr>

# Let's take a closer look at the different parts of the output 

# Results metrics
CV1 %>% select_metrics() %>% kable()
```

|     RMSE |      MAE |      r2m |      r2c |      AIC |     AICc |      BIC | Dependent | Fixed     |
| -------: | -------: | -------: | -------: | -------: | -------: | -------: | :-------- | :-------- |
| 16.35261 | 13.75772 | 0.270991 | 0.270991 | 194.6218 | 195.9276 | 197.9556 | score     | diagnosis |

``` r

# Nested predictions 
# Note that [[1]] picks predictions for the first row
CV1$Predictions[[1]] %>% head() %>% kable()
```

| Fold Column | Fold | Target | Prediction |
| :---------- | ---: | -----: | ---------: |
| .folds      |    1 |     33 |   51.00000 |
| .folds      |    1 |     53 |   51.00000 |
| .folds      |    1 |     66 |   51.00000 |
| .folds      |    1 |     16 |   30.66667 |
| .folds      |    1 |     32 |   30.66667 |
| .folds      |    1 |     44 |   30.66667 |

``` r

# Nested results from the different folds
CV1$Results[[1]] %>% kable()
```

| Fold Column | Fold |     RMSE |      MAE |       r2m |       r2c |      AIC |     AICc |      BIC |
| :---------- | ---: | -------: | -------: | --------: | --------: | -------: | -------: | -------: |
| .folds      |    1 | 12.56760 | 10.72222 | 0.2439198 | 0.2439198 | 209.9622 | 211.1622 | 213.4963 |
| .folds      |    2 | 16.60767 | 14.77778 | 0.2525524 | 0.2525524 | 182.8739 | 184.2857 | 186.0075 |
| .folds      |    3 | 15.97355 | 12.87037 | 0.2306104 | 0.2306104 | 207.9074 | 209.1074 | 211.4416 |
| .folds      |    4 | 20.26162 | 16.66049 | 0.3568816 | 0.3568816 | 177.7436 | 179.1554 | 180.8772 |

``` r

# Nested model coefficients
# Note that you have the full p-values, 
# but kable() only shows a certain number of digits
CV1$Coefficients[[1]] %>% kable()
```

| Fold Column | Fold | term        |   estimate | std.error |  statistic |   p.value |
| :---------- | ---: | :---------- | ---------: | --------: | ---------: | --------: |
| .folds      |    1 | (Intercept) |   51.00000 |  5.901264 |   8.642216 | 0.0000000 |
| .folds      |    1 | diagnosis   | \-20.33333 |  7.464574 | \-2.723978 | 0.0123925 |
| .folds      |    2 | (Intercept) |   53.33333 |  5.718886 |   9.325826 | 0.0000000 |
| .folds      |    2 | diagnosis   | \-19.66667 |  7.565375 | \-2.599563 | 0.0176016 |
| .folds      |    3 | (Intercept) |   49.77778 |  5.653977 |   8.804030 | 0.0000000 |
| .folds      |    3 | diagnosis   | \-18.77778 |  7.151778 | \-2.625610 | 0.0154426 |
| .folds      |    4 | (Intercept) |   49.55556 |  5.061304 |   9.791065 | 0.0000000 |
| .folds      |    4 | diagnosis   | \-22.30556 |  6.695476 | \-3.331437 | 0.0035077 |

``` r

# Additional information about the model
# and the training process
CV1 %>% select(11:19) %>% kable()
```

| Folds | Fold Columns | Convergence Warnings | Singular Fit Messages | Other Warnings | Warnings and Messages                                                                                                       | Family   | Dependent | Fixed     |
| ----: | -----------: | -------------------: | --------------------: | -------------: | :-------------------------------------------------------------------------------------------------------------------------- | :------- | :-------- | :-------- |
|     4 |            1 |                    0 |                     0 |              0 | list(`Fold Column` = character(0), Fold = integer(0), Function = character(0), Type = character(0), Message = character(0)) | gaussian | score     | diagnosis |

### Binomial

``` r
CV2 <- cross_validate(data, 
                      formulas = "diagnosis~score",
                      fold_cols = '.folds',
                      family = 'binomial')

# Show results
CV2
#> # A tibble: 1 x 28
#>   `Balanced Accur…    F1 Sensitivity Specificity `Pos Pred Value`
#>              <dbl> <dbl>       <dbl>       <dbl>            <dbl>
#> 1            0.736 0.821       0.889       0.583            0.762
#> # … with 23 more variables: `Neg Pred Value` <dbl>, AUC <dbl>, `Lower
#> #   CI` <dbl>, `Upper CI` <dbl>, Kappa <dbl>, MCC <dbl>, `Detection
#> #   Rate` <dbl>, `Detection Prevalence` <dbl>, Prevalence <dbl>,
#> #   Predictions <list>, ROC <list>, `Confusion Matrix` <list>,
#> #   Results <list>, Coefficients <list>, Folds <int>, `Fold
#> #   Columns` <int>, `Convergence Warnings` <int>, `Singular Fit
#> #   Messages` <int>, `Other Warnings` <int>, `Warnings and
#> #   Messages` <list>, Family <chr>, Dependent <chr>, Fixed <chr>

# Let's take a closer look at the different parts of the output 
# We won't repeat the parts too similar to those in Gaussian

# Results metrics
CV2 %>% select(1:9) %>% kable()
```

| Balanced Accuracy |        F1 | Sensitivity | Specificity | Pos Pred Value | Neg Pred Value |       AUC |  Lower CI |  Upper CI |
| ----------------: | --------: | ----------: | ----------: | -------------: | -------------: | --------: | --------: | --------: |
|         0.7361111 | 0.8205128 |   0.8888889 |   0.5833333 |      0.7619048 |      0.7777778 | 0.7685185 | 0.5962701 | 0.9407669 |

``` r
CV2 %>% select(10:14) %>% kable()
```

|     Kappa |       MCC | Detection Rate | Detection Prevalence | Prevalence |
| --------: | --------: | -------------: | -------------------: | ---------: |
| 0.4927536 | 0.5048268 |      0.5333333 |                  0.7 |        0.6 |

``` r

# Confusion matrix
CV2$`Confusion Matrix`[[1]] %>% kable()
```

| Fold Column | Prediction | Target | Pos\_0 | Pos\_1 |  N |
| :---------- | :--------- | :----- | :----- | :----- | -: |
| .folds      | 0          | 0      | TP     | TN     |  7 |
| .folds      | 1          | 0      | FN     | FP     |  5 |
| .folds      | 0          | 1      | FP     | FN     |  2 |
| .folds      | 1          | 1      | TN     | TP     | 16 |

## Cross-validate multiple models

### Create model formulas

``` r
model_formulas <- c("score~diagnosis", "score~age")
mixed_model_formulas <- c("score~diagnosis+(1|session)", "score~age+(1|session)")
```

### Cross-validate fixed effects models

``` r
CV3 <- cross_validate(data, 
                      formulas = model_formulas,
                      fold_cols = '.folds',
                      family = 'gaussian',
                      REML = FALSE)

# Show results
CV3
#> # A tibble: 2 x 19
#>    RMSE   MAE    r2m    r2c   AIC  AICc   BIC Predictions Results
#>   <dbl> <dbl>  <dbl>  <dbl> <dbl> <dbl> <dbl> <list>      <list> 
#> 1  16.4  13.8 0.271  0.271   195.  196.  198. <tibble [3… <tibbl…
#> 2  22.4  18.9 0.0338 0.0338  201.  202.  204. <tibble [3… <tibbl…
#> # … with 10 more variables: Coefficients <list>, Folds <int>, `Fold
#> #   Columns` <int>, `Convergence Warnings` <int>, `Singular Fit
#> #   Messages` <int>, `Other Warnings` <int>, `Warnings and
#> #   Messages` <list>, Family <chr>, Dependent <chr>, Fixed <chr>
```

### Cross-validate mixed effects models

``` r
CV4 <- cross_validate(data, 
                      formulas = mixed_model_formulas,
                      fold_cols = '.folds',
                      family = 'gaussian',
                      REML = FALSE)

# Show results
CV4
#> # A tibble: 2 x 20
#>    RMSE   MAE    r2m   r2c   AIC  AICc   BIC Predictions Results
#>   <dbl> <dbl>  <dbl> <dbl> <dbl> <dbl> <dbl> <list>      <list> 
#> 1  7.95  6.41 0.290  0.811  176.  178.  180. <tibble [3… <tibbl…
#> 2 17.5  16.2  0.0366 0.526  194.  196.  198. <tibble [3… <tibbl…
#> # … with 11 more variables: Coefficients <list>, Folds <int>, `Fold
#> #   Columns` <int>, `Convergence Warnings` <int>, `Singular Fit
#> #   Messages` <int>, `Other Warnings` <int>, `Warnings and
#> #   Messages` <list>, Family <chr>, Dependent <chr>, Fixed <chr>,
#> #   Random <chr>
```

## Repeated cross-validation

Let’s first add some extra fold columns. We will use the num\_fold\_cols
argument to add 3 unique fold columns. We tell `fold()` to keep the
existing fold column and simply add three extra columns. We could also
choose to remove the existing fold column, if for instance we were
changing the number of folds (k). Note, that the original fold column
will be renamed to “.folds\_1”.

``` r
# Set seed for reproducibility
set.seed(2)

# Fold data 
data <- fold(data, k = 4,
             cat_col = 'diagnosis',
             id_col = 'participant',
             num_fold_cols = 3,
             handle_existing_fold_cols = "keep")

# Show first 15 rows of data
data %>% head(10) %>% kable()
```

| participant | age | diagnosis | score | session | .folds\_1 | .folds\_2 | .folds\_3 | .folds\_4 |
| :---------- | --: | --------: | ----: | ------: | :-------- | :-------- | :-------- | :-------- |
| 10          |  32 |         0 |    29 |       1 | 4         | 4         | 3         | 1         |
| 10          |  32 |         0 |    55 |       2 | 4         | 4         | 3         | 1         |
| 10          |  32 |         0 |    81 |       3 | 4         | 4         | 3         | 1         |
| 2           |  23 |         0 |    24 |       1 | 2         | 3         | 1         | 2         |
| 2           |  23 |         0 |    40 |       2 | 2         | 3         | 1         | 2         |
| 2           |  23 |         0 |    67 |       3 | 2         | 3         | 1         | 2         |
| 4           |  21 |         0 |    35 |       1 | 3         | 2         | 4         | 4         |
| 4           |  21 |         0 |    50 |       2 | 3         | 2         | 4         | 4         |
| 4           |  21 |         0 |    78 |       3 | 3         | 2         | 4         | 4         |
| 9           |  34 |         0 |    33 |       1 | 1         | 1         | 2         | 3         |

``` r
CV5 <- cross_validate(data, 
                      formulas = "diagnosis ~ score",
                      fold_cols = paste0(".folds_", 1:4),
                      family = 'binomial',
                      REML = FALSE)

# Show results
CV5
#> # A tibble: 1 x 28
#>   `Balanced Accur…    F1 Sensitivity Specificity `Pos Pred Value`
#>              <dbl> <dbl>       <dbl>       <dbl>            <dbl>
#> 1            0.729 0.813       0.875       0.583            0.759
#> # … with 23 more variables: `Neg Pred Value` <dbl>, AUC <dbl>, `Lower
#> #   CI` <dbl>, `Upper CI` <dbl>, Kappa <dbl>, MCC <dbl>, `Detection
#> #   Rate` <dbl>, `Detection Prevalence` <dbl>, Prevalence <dbl>,
#> #   Predictions <list>, ROC <list>, `Confusion Matrix` <list>,
#> #   Results <list>, Coefficients <list>, Folds <int>, `Fold
#> #   Columns` <int>, `Convergence Warnings` <int>, `Singular Fit
#> #   Messages` <int>, `Other Warnings` <int>, `Warnings and
#> #   Messages` <list>, Family <chr>, Dependent <chr>, Fixed <chr>

# The binomial output now has a nested 'Results' tibble
# Let's see a subset of the columns
CV5$Results[[1]] %>% select(1:8) %>%  kable()
```

| Fold Column | Balanced Accuracy |        F1 | Sensitivity | Specificity | Pos Pred Value | Neg Pred Value |       AUC |
| :---------- | ----------------: | --------: | ----------: | ----------: | -------------: | -------------: | --------: |
| .folds\_1   |         0.7361111 | 0.8205128 |   0.8888889 |   0.5833333 |      0.7619048 |      0.7777778 | 0.7685185 |
| .folds\_2   |         0.7361111 | 0.8205128 |   0.8888889 |   0.5833333 |      0.7619048 |      0.7777778 | 0.7777778 |
| .folds\_3   |         0.7083333 | 0.7894737 |   0.8333333 |   0.5833333 |      0.7500000 |      0.7000000 | 0.7476852 |
| .folds\_4   |         0.7361111 | 0.8205128 |   0.8888889 |   0.5833333 |      0.7619048 |      0.7777778 | 0.7662037 |

## Cross-validating custom model functions

`cross_validate_fn()` works with regression (`gaussian`), binary
classification (`binomial`), and multiclass classification
(`multinomial`).

### SVM

Let’s cross-validate a support-vector machine using the `svm()` function
from the `e1071` package. First, we will create a model function. You
can do anything you want in it, as long as it takes the arguments
`train_data` and `formula` and returns the fitted model object.

``` r
# Create model function
#
# train_data : tibble with the training data
# formula : a formula object
# hyperparameters : a named list of hyparameters

svm_model_fn <- function(train_data, formula, hyperparameters){
  
  # Note that `formula` must be specified first
  # when calling svm(), otherwise it fails
  e1071::svm(formula = formula,
             data = train_data, 
             kernel = "linear",
             type = "C-classification",
             probability = TRUE)
}
```

``` r
# Create predict function
#
# test_data : tibble with the test data
# model : fitted model object
# formula : a formula object
# hyperparameters : a named list of hyparameters

svm_predict_fn <- function(test_data, model, formula, hyperparameters){
      predictions <- stats::predict(object = model,
                     newdata = test_data,
                     allow.new.levels = TRUE,
                     probability = TRUE)

      # Extract probabilities
      probabilities <- dplyr::as_tibble(
        attr(predictions, "probabilities")
        )

      # Return second column
      probabilities[[2]]
    }
```

<!-- For the `svm()` function, the default predict function and settings within `cross_validate_fn()` works, so we don't have to specify a predict function. In many cases, it's probably safer to supply a predict function anyway, so you're sure everything is correct. We will see how in the naive Bayes example below, but first, let's cross-validate the model function. Note, that some of the arguments have changed names (`models -> formulas`, `family -> type`). -->

``` r
# Cross-validate svm_model_fn
CV6 <- cross_validate_fn(data = data,
                         model_fn = svm_model_fn,
                         predict_fn = svm_predict_fn,
                         formulas = c("diagnosis~score", "diagnosis~age"),
                         fold_cols = '.folds_1', 
                         type = 'binomial')
#> Will cross-validate 2 models. This requires fitting 8 model instances.

CV6
#> # A tibble: 2 x 27
#>   `Balanced Accur…    F1 Sensitivity Specificity `Pos Pred Value`
#>              <dbl> <dbl>       <dbl>       <dbl>            <dbl>
#> 1            0.653 0.780       0.889       0.417            0.696
#> 2            0.458 0.615       0.667       0.25             0.571
#> # … with 22 more variables: `Neg Pred Value` <dbl>, AUC <dbl>, `Lower
#> #   CI` <dbl>, `Upper CI` <dbl>, Kappa <dbl>, MCC <dbl>, `Detection
#> #   Rate` <dbl>, `Detection Prevalence` <dbl>, Prevalence <dbl>,
#> #   Predictions <list>, ROC <list>, `Confusion Matrix` <list>,
#> #   Results <list>, Coefficients <list>, Folds <int>, `Fold
#> #   Columns` <int>, `Convergence Warnings` <int>, `Other Warnings` <int>,
#> #   `Warnings and Messages` <list>, Family <chr>, Dependent <chr>,
#> #   Fixed <chr>
```

### Naive Bayes

The naive Bayes classifier requires us to supply a predict function, so
we will go through that next. First, let’s create the model function.

``` r
# Create model function
#
# train_data : tibble with the training data
# formula : a formula object

nb_model_fn <- function(train_data, formula, hyperparameters){
  e1071::naiveBayes(formula = formula, 
                    data = train_data)
}
```

Now, we will create a predict function. This will usually wrap
`stats::predict()` and just make sure, the predictions have the correct
format. When `type` is `binomial`, the predictions should be a vector,
or a one-column matrix / data frame, with the probabilities of the
second class (alphabetically). That is, if we have the classes `0` and
`1`, it should be the probabilities of the observations being in class
`1`. The help file, `?cross_validate_fn`, describes the formats for the
other types (`gaussian` and `multinomial`).

The predict function should take the arguments `test_data`, `model`, and
`formula`. You do not need to use the `formula` within your function.

``` r
# Create predict function
#
# test_data : tibble with the test data
# model : fitted model object
# formula : a formula object
nb_predict_fn <- function(test_data, model, formula, hyperparameters){
    stats::predict(object = model, newdata = test_data, 
                   type = "raw", allow.new.levels = TRUE)[,2]
  }
```

With both functions specified, we are ready to cross-validate our naive
Bayes classifier.

``` r
CV7 <- cross_validate_fn(data,
                         model_fn = nb_model_fn,
                         predict_fn = nb_predict_fn,
                         formulas = c("diagnosis~score", "diagnosis~age"),
                         type = 'binomial',
                         fold_cols = '.folds_1')
#> Will cross-validate 2 models. This requires fitting 8 model instances.

CV7
#> # A tibble: 2 x 27
#>   `Balanced Accur…    F1 Sensitivity Specificity `Pos Pred Value`
#>              <dbl> <dbl>       <dbl>       <dbl>            <dbl>
#> 1            0.736 0.821       0.889       0.583            0.762
#> 2            0.25  0.462       0.5         0                0.429
#> # … with 22 more variables: `Neg Pred Value` <dbl>, AUC <dbl>, `Lower
#> #   CI` <dbl>, `Upper CI` <dbl>, Kappa <dbl>, MCC <dbl>, `Detection
#> #   Rate` <dbl>, `Detection Prevalence` <dbl>, Prevalence <dbl>,
#> #   Predictions <list>, ROC <list>, `Confusion Matrix` <list>,
#> #   Results <list>, Coefficients <list>, Folds <int>, `Fold
#> #   Columns` <int>, `Convergence Warnings` <int>, `Other Warnings` <int>,
#> #   `Warnings and Messages` <list>, Family <chr>, Dependent <chr>,
#> #   Fixed <chr>
```

## Evaluating predictions

Evaluate predictions from a model trained outside cvms. Works with
regression (`gaussian`), binary classification (`binomial`), and
multiclass classification (`multinomial`). The following is an example
of multinomial evaluation.

### Multinomial

Create a dataset with 3 predictors and a target column. Partition it
with `groupdata2::partition()` to create a training set and a validation
set. `multiclass_probability_tibble()` is a simple helper function for
generating random tibbles.

``` r
# Set seed
set.seed(1)

# Create class names
class_names <- paste0("class_", 1:4)

# Create random dataset with 100 observations 
# Partition into training set (75%) and test set (25%)
multiclass_partitions <- multiclass_probability_tibble(
  num_classes = 3, # Here, number of predictors
  num_observations = 100,
  apply_softmax = FALSE,
  FUN = rnorm,
  class_name = "predictor_") %>%
  dplyr::mutate(class = sample(
    class_names,
    size = 100,
    replace = TRUE)) %>%
  partition(p = 0.75,
            cat_col = "class")

# Extract partitions
multiclass_train_set <- multiclass_partitions[[1]]
multiclass_test_set <- multiclass_partitions[[2]]

multiclass_test_set
#> # A tibble: 26 x 4
#>    predictor_1 predictor_2 predictor_3 class  
#>          <dbl>       <dbl>       <dbl> <chr>  
#>  1      1.60         0.158     -0.331  class_1
#>  2     -1.99        -0.180     -0.341  class_1
#>  3      0.418       -0.324      0.263  class_1
#>  4      0.398        0.450      0.136  class_1
#>  5      0.0743       1.03      -1.32   class_1
#>  6      0.738        0.910      0.541  class_2
#>  7      0.576        0.384     -0.0134 class_2
#>  8     -0.305        1.68       0.510  class_2
#>  9     -0.0449      -0.393      1.52   class_2
#> 10      0.557       -0.464     -0.879  class_2
#> # … with 16 more rows
```

Train multinomial model using the `nnet` package and get the predicted
probabilities.

``` r
# Train multinomial model
multiclass_model <- nnet::multinom(
   "class ~ predictor_1 + predictor_2 + predictor_3",
   data = multiclass_train_set)
#> # weights:  20 (12 variable)
#> initial  value 102.585783 
#> iter  10 value 98.124010
#> final  value 98.114250 
#> converged

# Predict the targets in the test set
predictions <- predict(multiclass_model, 
                       multiclass_test_set,
                       type = "probs") %>%
  dplyr::as_tibble()

# Add the targets
predictions[["target"]] <- multiclass_test_set[["class"]]

head(predictions, 10)
#> # A tibble: 10 x 5
#>    class_1 class_2 class_3 class_4 target 
#>      <dbl>   <dbl>   <dbl>   <dbl> <chr>  
#>  1   0.243   0.214   0.304   0.239 class_1
#>  2   0.136   0.371   0.234   0.259 class_1
#>  3   0.230   0.276   0.264   0.230 class_1
#>  4   0.194   0.218   0.262   0.326 class_1
#>  5   0.144   0.215   0.302   0.339 class_1
#>  6   0.186   0.166   0.241   0.407 class_2
#>  7   0.201   0.222   0.272   0.305 class_2
#>  8   0.117   0.131   0.195   0.557 class_2
#>  9   0.237   0.264   0.215   0.284 class_2
#> 10   0.216   0.310   0.303   0.171 class_2
```

Perform the evaluation. This will create one-vs-all binomial evaluations
and summarize the results.

``` r
# Evaluate predictions
ev <- evaluate(data = predictions,
               target_col = "target",
               prediction_cols = class_names,
               type = "multinomial")

ev
#> # A tibble: 1 x 17
#>   `Overall Accura… `Balanced Accur…    F1 Sensitivity Specificity
#>              <dbl>            <dbl> <dbl>       <dbl>       <dbl>
#> 1            0.154            0.427   NaN       0.143       0.712
#> # … with 12 more variables: `Pos Pred Value` <dbl>, `Neg Pred
#> #   Value` <dbl>, AUC <dbl>, Kappa <dbl>, MCC <dbl>, `Detection
#> #   Rate` <dbl>, `Detection Prevalence` <dbl>, Prevalence <dbl>,
#> #   Predictions <list>, ROC <list>, `Confusion Matrix` <list>, `Class
#> #   Level Results` <list>
```

The class level results (i.e., the one-vs-all evaluations) are also
included, and would usually be reported alongside the above results.

``` r
ev$`Class Level Results`
#> [[1]]
#> # A tibble: 4 x 14
#>   Class `Balanced Accur…      F1 Sensitivity Specificity `Pos Pred Value`
#>   <chr>            <dbl>   <dbl>       <dbl>       <dbl>            <dbl>
#> 1 clas…            0.476 NaN           0           0.952            0    
#> 2 clas…            0.380   0.211       0.286       0.474            0.167
#> 3 clas…            0.474 NaN           0           0.947            0    
#> 4 clas…            0.380   0.211       0.286       0.474            0.167
#> # … with 8 more variables: `Neg Pred Value` <dbl>, Kappa <dbl>, MCC <dbl>,
#> #   `Detection Rate` <dbl>, `Detection Prevalence` <dbl>,
#> #   Prevalence <dbl>, Support <int>, `Confusion Matrix` <list>
```

## Baseline evaluations

Create baseline evaluations of a test set.

### Gaussian

Approach: The baseline model (y ~ 1), where 1 is simply the intercept
(i.e. mean of y), is fitted on n random subsets of the training set and
evaluated on the test set. We also perform an evaluation of the model
fitted on the entire training set.

Start by partitioning the dataset.

``` r
# Set seed for reproducibility
set.seed(1)

# Partition the dataset 
partitions <- groupdata2::partition(participant.scores,
                                    p = 0.7,
                                    cat_col = 'diagnosis',
                                    id_col = 'participant',
                                    list_out = TRUE)
train_set <- partitions[[1]]
test_set <- partitions[[2]]
```

Create the baseline evaluations:

``` r
baseline(test_data = test_set, train_data = train_set,
         n = 100, dependent_col = "score", family = "gaussian")
#> $summarized_metrics
#>    Measure      RMSE        MAE r2m r2c       AIC      AICc       BIC
#> 1     Mean 19.694703 15.8383731   0   0  86.98349  89.47709  87.39436
#> 2   Median 19.192945 15.5000000   0   0  83.28857  85.28857  83.68302
#> 3       SD  1.045639  0.7592939   0   0  28.94671  27.55333  29.64196
#> 4      IQR  1.161372  0.2638889   0   0  45.91832  44.25165  46.99631
#> 5      Max 24.147119 19.4166667   0   0 136.81149 137.81149 138.22759
#> 6      Min 18.923354 15.5000000   0   0  41.97455  47.97455  41.19342
#> 7      NAs  0.000000  0.0000000   0   0   0.00000   0.00000   0.00000
#> 8     INFs  0.000000  0.0000000   0   0   0.00000   0.00000   0.00000
#> 9 All_rows 19.142336 15.5000000   0   0 160.91539 161.71539 162.69613
#>   Training Rows
#> 1      9.630000
#> 2      9.000000
#> 3      3.215037
#> 4      5.000000
#> 5     15.000000
#> 6      5.000000
#> 7      0.000000
#> 8      0.000000
#> 9     18.000000
#> 
#> $random_evaluations
#> # A tibble: 100 x 13
#>     RMSE   MAE   r2m   r2c   AIC  AICc   BIC Predictions Coefficients
#>    <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <list>      <list>      
#>  1  20.0  16.3     0     0  72.5  74.9  72.7 <tibble [1… <tibble [1 …
#>  2  19.0  15.5     0     0 137.  138.  138.  <tibble [1… <tibble [1 …
#>  3  20.2  15.7     0     0  61.3  64.3  61.2 <tibble [1… <tibble [1 …
#>  4  20.0  15.7     0     0  97.7  99.2  98.5 <tibble [1… <tibble [1 …
#>  5  19.3  15.6     0     0  73.3  75.7  73.5 <tibble [1… <tibble [1 …
#>  6  20.4  15.9     0     0  44.4  50.4  43.6 <tibble [1… <tibble [1 …
#>  7  19.0  15.5     0     0 118.  120.  119.  <tibble [1… <tibble [1 …
#>  8  19.4  15.5     0     0  93.3  95.1  94.0 <tibble [1… <tibble [1 …
#>  9  20.7  16.2     0     0  71.2  73.6  71.3 <tibble [1… <tibble [1 …
#> 10  20.8  17.1     0     0  43.7  49.7  42.9 <tibble [1… <tibble [1 …
#> # … with 90 more rows, and 4 more variables: `Training Rows` <int>,
#> #   Family <chr>, Dependent <chr>, Fixed <chr>
```

### Binomial

Approach: n random sets of predictions are evaluated against the
dependent variable in the test set. We also evaluate a set of all 0s and
a set of all 1s.

Create the baseline evaluations:

``` r
baseline(test_data = test_set, n = 100, 
         dependent_col = "diagnosis", family = "binomial")
#> $summarized_metrics
#>    Measure Balanced Accuracy        F1 Sensitivity Specificity
#> 1     Mean         0.5016667 0.4953379   0.4783333   0.5250000
#> 2   Median         0.5000000 0.5000000   0.5000000   0.5000000
#> 3       SD         0.1469766 0.1590983   0.2153678   0.2097176
#> 4      IQR         0.1666667 0.2517483   0.3333333   0.3333333
#> 5      Max         0.8333333 0.8333333   0.8333333   1.0000000
#> 6      Min         0.1666667 0.1818182   0.0000000   0.0000000
#> 7      NAs         0.0000000 4.0000000   0.0000000   0.0000000
#> 8     INFs         0.0000000 0.0000000   0.0000000   0.0000000
#> 9    All_0         0.5000000        NA   0.0000000   1.0000000
#> 10   All_1         0.5000000 0.6666667   1.0000000   0.0000000
#>    Pos Pred Value Neg Pred Value       AUC   Lower CI  Upper CI
#> 1       0.4975635      0.5004026 0.4955556 0.14787921 0.8428463
#> 2       0.5000000      0.5000000 0.4722222 0.09192543 0.8571955
#> 3       0.1935538      0.1597369 0.1590928 0.16052808 0.1379743
#> 4       0.2000000      0.2000000 0.2013889 0.21398031 0.2222222
#> 5       1.0000000      0.8333333 0.8888889 0.69410610 1.0000000
#> 6       0.0000000      0.0000000 0.1666667 0.00000000 0.4933273
#> 7       0.0000000      0.0000000 0.0000000 0.00000000 0.0000000
#> 8       0.0000000      0.0000000 0.0000000 0.00000000 0.0000000
#> 9             NaN      0.5000000 0.5000000 0.50000000 0.5000000
#> 10      0.5000000            NaN 0.5000000 0.50000000 0.5000000
#>           Kappa          MCC Detection Rate Detection Prevalence
#> 1   0.003333333  0.001150867      0.2391667           0.47666667
#> 2   0.000000000  0.000000000      0.2500000           0.50000000
#> 3   0.293953278  0.310390217      0.1076839           0.15355861
#> 4   0.333333333  0.384900179      0.1666667           0.25000000
#> 5   0.666666667  0.666666667      0.4166667           0.83333333
#> 6  -0.666666667 -0.707106781      0.0000000           0.08333333
#> 7   0.000000000  0.000000000      0.0000000           0.00000000
#> 8   0.000000000  0.000000000      0.0000000           0.00000000
#> 9   0.000000000  0.000000000      0.0000000           0.00000000
#> 10  0.000000000  0.000000000      0.5000000           1.00000000
#>    Prevalence
#> 1         0.5
#> 2         0.5
#> 3         0.0
#> 4         0.0
#> 5         0.5
#> 6         0.5
#> 7         0.0
#> 8         0.0
#> 9         0.5
#> 10        0.5
#> 
#> $random_evaluations
#> # A tibble: 100 x 19
#>    `Balanced Accur…    F1 Sensitivity Specificity `Pos Pred Value`
#>               <dbl> <dbl>       <dbl>       <dbl>            <dbl>
#>  1            0.417 0.364       0.333       0.5              0.4  
#>  2            0.5   0.5         0.5         0.5              0.5  
#>  3            0.417 0.364       0.333       0.5              0.4  
#>  4            0.667 0.6         0.5         0.833            0.75 
#>  5            0.583 0.667       0.833       0.333            0.556
#>  6            0.667 0.6         0.5         0.833            0.75 
#>  7            0.25  0.308       0.333       0.167            0.286
#>  8            0.5   0.4         0.333       0.667            0.500
#>  9            0.25  0.182       0.167       0.333            0.20 
#> 10            0.417 0.222       0.167       0.667            0.333
#> # … with 90 more rows, and 14 more variables: `Neg Pred Value` <dbl>,
#> #   AUC <dbl>, `Lower CI` <dbl>, `Upper CI` <dbl>, Kappa <dbl>, MCC <dbl>,
#> #   `Detection Rate` <dbl>, `Detection Prevalence` <dbl>,
#> #   Prevalence <dbl>, Predictions <list>, ROC <list>, `Confusion
#> #   Matrix` <list>, Family <chr>, Dependent <chr>
```

### Multinomial

Approach: Creates one-vs-all (binomial) baseline evaluations for n sets
of random predictions against the dependent variable, along with sets of
“all class x,y,z,…” predictions.

Create the baseline evaluations:

``` r
multiclass_baseline <- baseline(
  test_data = multiclass_test_set, n = 100,
  dependent_col = "class", family = "multinomial")

# Summarized metrics
multiclass_baseline$summarized_metrics
#>        Measure Overall Accuracy Balanced Accuracy          F1 Sensitivity
#> 1         Mean       0.25038462        0.50134085  0.28310485  0.25250000
#> 2       Median       0.23076923        0.49360902  0.28013925  0.24285714
#> 3           SD       0.08406884        0.05671680  0.07374332  0.08529464
#> 4          IQR       0.11538462        0.07951128  0.09196935  0.12142857
#> 5          Max       0.53846154        0.78571429  0.66666667  1.00000000
#> 6          Min       0.07692308        0.26190476  0.11111111  0.00000000
#> 7          NAs               NA        0.00000000 61.00000000  0.00000000
#> 8         INFs               NA        0.00000000  0.00000000  0.00000000
#> 9  All_class_1       0.19230769        0.50000000          NA  0.25000000
#> 10 All_class_2       0.26923077        0.50000000          NA  0.25000000
#> 11 All_class_3       0.26923077        0.50000000          NA  0.25000000
#> 12 All_class_4       0.26923077        0.50000000          NA  0.25000000
#>    Specificity Pos Pred Value Neg Pred Value        AUC        Kappa
#> 1   0.75018170     0.24928523     0.75030214 0.49851701  0.001983840
#> 2   0.74624060     0.23819444     0.74604533 0.49863946 -0.009196691
#> 3   0.02837058     0.09327301     0.02956503 0.08005267  0.111210834
#> 4   0.03853383     0.13272682     0.04058776 0.09991497  0.158289366
#> 5   1.00000000     0.80000000     1.00000000 0.72346939  0.570247934
#> 6   0.47368421     0.00000000     0.58823529 0.31938776 -0.434482759
#> 7   0.00000000     1.00000000     0.00000000         NA  0.000000000
#> 8   0.00000000     0.00000000     0.00000000         NA  0.000000000
#> 9   0.75000000            NaN            NaN 0.50000000  0.000000000
#> 10  0.75000000            NaN            NaN 0.50000000  0.000000000
#> 11  0.75000000            NaN            NaN 0.50000000  0.000000000
#> 12  0.75000000            NaN            NaN 0.50000000  0.000000000
#>             MCC Detection Rate Detection Prevalence Prevalence
#> 1   0.001487716     0.06259615            0.2500000  0.2500000
#> 2  -0.011866913     0.05769231            0.2500000  0.2500000
#> 3   0.115086597     0.02101721            0.0000000  0.0000000
#> 4   0.163024146     0.02884615            0.0000000  0.0000000
#> 5   0.583886751     0.19230769            0.5384615  0.2692308
#> 6  -0.441640623     0.00000000            0.0000000  0.1923077
#> 7   0.000000000     0.00000000            0.0000000  0.0000000
#> 8   0.000000000     0.00000000            0.0000000  0.0000000
#> 9   0.000000000     0.04807692            0.2500000  0.2500000
#> 10  0.000000000     0.06730769            0.2500000  0.2500000
#> 11  0.000000000     0.06730769            0.2500000  0.2500000
#> 12  0.000000000     0.06730769            0.2500000  0.2500000

# Summarized class level results for class 1
multiclass_baseline$summarized_class_level_results %>% 
  dplyr::filter(Class == "class_1") %>%
  tidyr::unnest(Results)
#> # A tibble: 10 x 13
#>    Class Measure `Balanced Accur…     F1 Sensitivity Specificity
#>    <chr> <chr>              <dbl>  <dbl>       <dbl>       <dbl>
#>  1 clas… Mean               0.514  0.284       0.28       0.748 
#>  2 clas… Median             0.529  0.286       0.2        0.762 
#>  3 clas… SD                 0.102  0.106       0.191      0.0979
#>  4 clas… IQR                0.124  0.182       0.2        0.0952
#>  5 clas… Max                0.786  0.526       1          0.952 
#>  6 clas… Min                0.262  0.125       0          0.524 
#>  7 clas… NAs                0     18           0          0     
#>  8 clas… INFs               0      0           0          0     
#>  9 clas… All_0              0.5   NA           0          1     
#> 10 clas… All_1              0.5    0.323       1          0     
#> # … with 7 more variables: `Pos Pred Value` <dbl>, `Neg Pred Value` <dbl>,
#> #   Kappa <dbl>, MCC <dbl>, `Detection Rate` <dbl>, `Detection
#> #   Prevalence` <dbl>, Prevalence <dbl>

# Random evaluations
# Note, that the class level results for each repetition
# is available as well
multiclass_baseline$random_evaluations
#> # A tibble: 100 x 20
#>    Repetition `Overall Accura… `Balanced Accur…      F1 Sensitivity
#>         <int>            <dbl>            <dbl>   <dbl>       <dbl>
#>  1          1            0.154            0.445 NaN           0.171
#>  2          2            0.269            0.518 NaN           0.279
#>  3          3            0.192            0.460   0.195       0.193
#>  4          4            0.385            0.591   0.380       0.386
#>  5          5            0.154            0.430 NaN           0.143
#>  6          6            0.154            0.438 NaN           0.157
#>  7          7            0.154            0.445 NaN           0.171
#>  8          8            0.346            0.574   0.341       0.364
#>  9          9            0.308            0.541   0.315       0.314
#> 10         10            0.308            0.536   0.322       0.3  
#> # … with 90 more rows, and 15 more variables: Specificity <dbl>, `Pos Pred
#> #   Value` <dbl>, `Neg Pred Value` <dbl>, AUC <dbl>, Kappa <dbl>,
#> #   MCC <dbl>, `Detection Rate` <dbl>, `Detection Prevalence` <dbl>,
#> #   Prevalence <dbl>, Predictions <list>, ROC <list>, `Confusion
#> #   Matrix` <list>, `Class Level Results` <list>, Family <chr>,
#> #   Dependent <chr>
```

## Plot results

There are currently a small set of plots for quick visualization of the
results. It is supposed to be easy to extract the needed information to
create your own plots. If you lack access to any information or have
other requests or ideas, feel free to open an issue.

### Gaussian

``` r
cv_plot(CV1, type = "RMSE") +
  theme_bw()
```

<img src="man/figures/README-unnamed-chunk-26-1.png" width="644" />

``` r
cv_plot(CV1, type = "r2") +
  theme_bw()
```

<img src="man/figures/README-unnamed-chunk-26-2.png" width="644" />

``` r
cv_plot(CV1, type = "IC") +
  theme_bw()
```

<img src="man/figures/README-unnamed-chunk-26-3.png" width="644" />

``` r
cv_plot(CV1, type = "coefficients") +
  theme_bw()
```

<img src="man/figures/README-unnamed-chunk-26-4.png" width="644" />

## Generate model formulas

Instead of manually typing all possible model formulas for a set of
fixed effects (including the possible interactions),
`combine_predictors()` can do it for you (with some constraints).

When including interactions, \>200k formulas have been precomputed for
up to 8 fixed effects, with a maximum interaction size of 3, and a
maximum of 5 fixed effects per formula. It’s possible to further limit
the generated formulas.

We can also append a random effects structure to the generated formulas.

``` r
combine_predictors(dependent = "y",
                   fixed_effects = c("a","b","c"),
                   random_effects = "(1|d)")
#>  [1] "y ~ a + (1|d)"                    
#>  [2] "y ~ b + (1|d)"                    
#>  [3] "y ~ c + (1|d)"                    
#>  [4] "y ~ a * b + (1|d)"                
#>  [5] "y ~ a * c + (1|d)"                
#>  [6] "y ~ a + b + (1|d)"                
#>  [7] "y ~ a + c + (1|d)"                
#>  [8] "y ~ b * c + (1|d)"                
#>  [9] "y ~ b + c + (1|d)"                
#> [10] "y ~ a * b * c + (1|d)"            
#> [11] "y ~ a * b + c + (1|d)"            
#> [12] "y ~ a * c + b + (1|d)"            
#> [13] "y ~ a + b * c + (1|d)"            
#> [14] "y ~ a + b + c + (1|d)"            
#> [15] "y ~ a * b + a * c + (1|d)"        
#> [16] "y ~ a * b + b * c + (1|d)"        
#> [17] "y ~ a * c + b * c + (1|d)"        
#> [18] "y ~ a * b + a * c + b * c + (1|d)"
```

If two or more fixed effects should not be in the same formula, like an
effect and its log-transformed version, we can provide them as sublists.

``` r
combine_predictors(dependent = "y",
                   fixed_effects = list("a", list("b","log_b")),
                   random_effects = "(1|d)")
#> [1] "y ~ a + (1|d)"         "y ~ b + (1|d)"         "y ~ log_b + (1|d)"    
#> [4] "y ~ a * b + (1|d)"     "y ~ a * log_b + (1|d)" "y ~ a + b + (1|d)"    
#> [7] "y ~ a + log_b + (1|d)"
```
