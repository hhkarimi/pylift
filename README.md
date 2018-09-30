# pylift

This package simplifies uplift model creation, tuning, and evaluation using the
Transformed Outcome Tree proxy method (Athey 2015).

# Usage

## Installation
```
git clone git@git.csnzoo.com:ds-uplift/pylift.git
cd pylift
pip install .
```

To upgrade, `git pull origin master` in the repo folder, and
then run `pip install --upgrade --no-cache-dir .`.

## Quick start
To start, you simply need a `pandas.DataFrame` with a treatment column of 0s
and 1s (0 for control, 1 for test) and a outcome column of 0s and 1s.
Implementation can be as simple as follows:

```
from pylift.proxymethods import TransformedOutcome
up = TransformedOutcome(df1, col_treatment='Test', col_outcome='Converted')

up.randomized_search()
up.fit(**up.rand_search_.best_params_)

up.plot(plot_type='aqini', show_theoretical_max=True)
print(up.test_results_.Q_aqini)
```

`up.fit()` can also be passed a flag `productionize=True`, which when `True`
will create a productionizable model
trained over the entire data set, stored in `self.model_final`. This can then
be pickled with `self.model_final.to_pickle(PATH)`, as usual.

## Passing custom parameters

Anything that can be taken by `RandomizedSearchCV()`, `GridSearchCV()`, or
`Regressor()` can be similarly passed to `up.randomized_search`,
`up.grid_search`, or `up.fit`, respectively.

```
up.fit(max_depth=2, nthread=-1)
```

`XGBRegressor` is the default regressor, but a different `Regressor` object can
also be used. To do this, pass the object to the keyword argument
`sklearn_model` during `TransformedOutcome` instantiation.

```
up = TransformedOutcome(df, col_treatment='Test', col_outcome='Converted', sklearn_model=RandomForestRegressor)

grid_search_params = {
    'estimator': RandomForestRegressor(),
    'param_grid': {'min_samples_split': [2,3,5,10,30,100,300,1000,3000,10000]},
    'verbose': True,
    'n_jobs': 35,
}
up.grid_search(**grid_search_params)
```

Regardless of what regressor you use, the `RandomizedSearchCV` default params
are contained in `up.randomized_search_params`, and the `GridSearchCV` params
are located in `up.grid_search_params`. These can be manually replaced, but
doing so will remove the scoring functions, so it is highly recommended that
any alterations to these class attributes be done as an update, or that
alterations be simply passed as arguments to `randomized_search` or
`grid_search`, as shown above.


## Accessing sklearn objects

The class objects produced by the sklearn classes, `RandomizedSearchCV`,
`GridSearchCV`, `XGBRegressor`, etc. are preserved in the `TransformedOutcome`
class as class attributes.

`up.randomized_search` -> `up.rand_search_`

`up.grid_search` -> `up.grid_search_`

`up.fit` -> `up.model`

`up.fit(productionize=True)` -> `up.model_final`

## Evaluation

All curves can be plotted using

```
up.plot(plot_type='qini')
```

Where `plot_type` can be any of the following values. In the formulaic representations

* `qini`: typical Qini curve (see Radcliffe 2007), except we normalize by the total number of people in treatment. The typical definition is `n_{t,1} - n_{c,1}*N_t/N_c`.
* `aqini`: adjusted Qini curve, calculated as `n_{t,1}/N_t - n_{c,1}*n_t/(n_c*N_t)`.
* `cuplift`: cumulative uplift curve, calculated as `n_{t,1}/n_t - n_{c,1}/n_c`.
* `uplift`: typical uplift curve, calculated the same as cuplift but only returning the average value within the bin, rather than cumulatively.
* `cgains`: cumulative gains curve (see Gutierrez, Gerardy 2016), defined as `((n_{t,1}/n_t - n_{c,1}/n_c)*phi`.
* `balance`: ratio of treatment group size to total group size within each bin, `n_t/(n_c + n_t)`.

Above, `phi` corresponds to the fraction of individuals targeted -- the
x-axis of these curves. `n` and `N` correspond to counts up to `phi`
(except for the uplift curve, which is only within the bin at the `phi`
position) or within the entire group, respectively. The subscript `t`
indicates the treatment group, and `c`, the control. The subscript `1`
indicates the subset of the count for which individuals had a positive outcome.

A number of scores are stored in both the `test_results_` and `train_results_`
objects, containing scores calculated over the test set and train set,
respectively. Namely, there are three important scores:
* `Q`: unnormalized area between the qini curve and the random selection line.
* `q1`: `Q`, normalized by the theoretical maximum value of `Q`.
* `q2`: `Q`, normalized by the practical maximum value of `Q`.

Each of these can be accesses as attributes of `test_results_` or
`train_results_`. Either `_qini`, `_aqini`, or `_cgains` can be appended to obtain the
same calculation for the qini curve, adjusted qini
curve, or the cumulative gains curve, respectively. The score most unaffected by
anomalous treatment/control ordering, without any bias to treatment or control
(i.e. if you're looking at lift between two equally viable treatments) is the
`q1_cgains` score, but if you are looking at a simple treatment vs. control
situation, `q1_aqini` is preferred. Because this only really has meaning over
an independent holdout [test] set, the most valuable value to access, then,
would likely be `up.test_results_.q1_aqini`.

```
up.test_results_.q1_aqini # Over training set.
```

Maximal curves can also be toggled by passing flags into `up.plot()`.
* `show_theoretical_max`
* `show_practical_max`
* `show_no_dogs`
* `show_random_selection`

Each of these curves satisfies shows the maximally attainable curve given
different assumptions about the underlying data. The `show_theoretical_max`
curve corresponds to a sorting in which we assume that an individual is
persuadable (uplift = 1) if and only if they respond in the treatment group
(and the same reasoning applies to the control group, for sleeping dogs). The
`show_practical_max` curve assumes that all individuals that have a positive
outcome in the treatment group must also have a counterpart (relative to the
proportion of individuals in the treatment and control group) in the control
group that did not respond. This is a more conservative, realistic curve. The
former can only be attained through overfitting, while the latter can only be
attained under very generous circumstances. Within the package, we also
calculate the `show_no_dogs` curve, which simply precludes the possibility of
negative effects.

The random selection line is shown by default, but the option to
toggle it off is included in case you'd like to plot multiple plots on top of
each other.

The below code plots the practical max over the aqini curve of a model
contained in the TransformedOutcome object `up`, then overlays the aqini curve
of a second model contained in `up1`, also changing the line color.

```
ax = up.plot(show_practical_max=True, show_random_selection=False, label='Model 1')
up1.plot(ax=ax, label='Model 2', color=[0.7,0,0])
```

### Error bars
It is often useful to obtain error bars on your qini curves. We've implemented two ways to do this:

1. `up.shuffle_fit()`: Seeds the `train_test_split`, fit the model over the new training data, and evaluate on the new test data. Average these curves.
1. `up.noise_fit()`: Randomly shuffle the labels independently of the features and fit a model. This can help distinguish your evaluation curves from noise.

```
up.shuffle_fit()
up.plot(plot_type='aqini', show_shuffle_fits=True)
```

Adjustments can also be made to the aesthetics of these curves by passing in dictionaries that pass down to plot elements. For example, `shuffle_band_kwargs` is a dictionary of kwargs that modifies the `fill_between` shaded error bar region.

# UpliftEval

The `UpliftEval` class can also independently be used to apply the above evaluation visualizations and calculations.

```
from pylift.eval import UpliftEval
upev = UpliftEval(treatment, outcome, predictions)
upev.plot_aqini()
```

# Appendix

## Raw data

Raw data for the `TransformedOutcome` method are stored as class attributes:
```
up.randomized_search_params
up.grid_search_params
up.transform                # Outcome transform function.
up.untransform              # Reverse of outcome transform function.

# Data (`y` in any of these can be replaced with `tc` for treatment or `x`).
up.transformed_y_train  # The predicted uplift.
up.y_train
up.y_test
up.y                    # All the `y` data.
up.df
up.df_train
up.df_test

# Once a model has been created...
up.model
up.model_final
up.frost_score_test (or train)
```

## Qini information

`up.test_results_` and `up.train_results_` are `UpliftEval` class
objects, and consequently contain all data about your qini curves, which can be
accessed as follows.

```
up.test_results_.qini_x  # percentile
up.test_results_.qini_y
# Best theoretical qini curve.
up.test_results_.qini_max_x  # percentile
up.test_results_.qini_max_y
# Uplift curve.
up.test_results_.uplift_x  # percentile
up.test_results_.uplift_y
```

`up.train_results_` can be used to plot the qini performance on the training
data, as follows: `up.train_results_.plot_qini()`.
