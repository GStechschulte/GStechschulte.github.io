---
title: 'Advanced Interpret Usage in Bambi'
date: 2023-12-09
author: 'Gabriel Stechschulte'
draft: false
categories: ['probabilistic-programming']
---

<!--eofm-->

# Interpret Advanced Usage

The `interpret` module is inspired by the R package [marginaleffects](https://marginaleffects.com) and ports the core functionality of {marginaleffects} to Bambi. To close the gap of non-supported functionality in Bambi, `interpret` now provides a set of helper functions to aid the user in more advanced and complex analysis not covered within the `comparisons`, `predictions`, and `slopes` functions. 

These helper functions are `data_grid` and `select_draws`. The `data_grid` can be used to create a pairwise grid of data points for the user to pass to `model.predict`. Subsequently, `select_draws` is used to select the draws from the posterior (or posterior predictive) group of the InferenceData object returned by the predict method that correspond to the data points that "produced" that draw.

With access to the appropriately indexed draws, and data used to generate those draws, it enables for more complex analysis such as cross-comparisons and the choice of which model parameter to compute a quantity of interest for. Additionally, the user has more control over the data passed to `model.predict`. Below, it will be demonstrated how to use these helper functions. First, to reproduce the results from the standard `interpret` API, and then to compute cross-comparisons.


```python
import warnings

import arviz as az
import numpy as np
import pandas as pd

import bambi as bmb

from bambi.interpret.helpers import data_grid, select_draws

warnings.simplefilter(action='ignore', category=FutureWarning)
```

## Zero Inflated Poisson

We will adopt the zero inflated Poisson (ZIP) model from the [comparisons](https://bambinos.github.io/bambi/notebooks/plot_comparisons.html)  documentation to demonstrate the helper functions introduced above. 

The ZIP model will be used to predict how many fish are caught by visitors at a state park using survey [data](http://www.stata-press.com/data/r11/fish.dta). Many visitors catch zero fish, either because they did not fish at all, or because they were unlucky. We would like to explicitly model this bimodal behavior (zero versus non-zero) using a Zero Inflated Poisson model, and to compare how different inputs of interest $w$ and other covariate values $c$ are associated with the number of fish caught. The dataset contains data on 250 groups that went to a state park to fish. Each group was questioned about how many fish they caught (`count`), how many children were in the group (`child`), how many people were in the group (`persons`), if they used a live bait and whether or not they brought a camper to the park (`camper`).


```python
fish_data = pd.read_stata("http://www.stata-press.com/data/r11/fish.dta")
cols = ["count", "livebait", "camper", "persons", "child"]
fish_data = fish_data[cols]
fish_data["child"] = fish_data["child"].astype(np.int8)
fish_data["persons"] = fish_data["persons"].astype(np.int8)
fish_data["livebait"] = pd.Categorical(fish_data["livebait"])
fish_data["camper"] = pd.Categorical(fish_data["camper"])
```


```python
fish_model = bmb.Model(
    "count ~ livebait + camper + persons + child", 
    fish_data, 
    family='zero_inflated_poisson'
)

fish_idata = fish_model.fit(random_seed=1234)
```

    Auto-assigning NUTS sampler...
    Initializing NUTS using jitter+adapt_diag...
    Multiprocess sampling (4 chains in 4 jobs)
    NUTS: [count_psi, Intercept, livebait, camper, persons, child]




<style>
    /* Turns off some styling */
    progress {
        /* gets rid of default border in Firefox and Opera. */
        border: none;
        /* Needs to be in here for Safari polyfill so background images work as expected. */
        background-size: auto;
    }
    progress:not([value]), progress:not([value])::-webkit-progress-bar {
        background: repeating-linear-gradient(45deg, #7e7e7e, #7e7e7e 10px, #5c5c5c 10px, #5c5c5c 20px);
    }
    .progress-bar-interrupted, .progress-bar-interrupted::-webkit-progress-bar {
        background: #F44336;
    }
</style>





<div>
  <progress value='8000' class='' max='8000' style='width:300px; height:20px; vertical-align: middle;'></progress>
  100.00% [8000/8000 00:02&lt;00:00 Sampling 4 chains, 0 divergences]
</div>



    Sampling 4 chains for 1_000 tune and 1_000 draw iterations (4_000 + 4_000 draws total) took 3 seconds.


## Create a grid of data

`data_grid` allows you to create a pairwise grid, also known as a cross-join or cartesian product, of data using the covariates passed to the `conditional` and the optional `variable` parameter. Covariates not passed to `conditional`, but are terms in the Bambi model, are set to typical values (e.g., mean or mode). If you are coming from R, this function is partially inspired from the [data_grid](https://modelr.tidyverse.org/reference/data_grid.html) function in {modelr}. 

There are two ways to create a pairwise grid of data:

1. user-provided values are passed as a dictionary to `conditional` where the keys are the names of the covariates and the values are the values to use in the grid.
2. a list of covariates where the elements are the names of the covariates to use in the grid. As only the names of the covariates were passed, default values are computed to construct the grid.

Any unspecified covariates, i.e., covariates not passed to `conditional` but are terms in the Bambi model, are set to their "typical" values such as mean or mode depending on the data type of the covariate.

### User-provided values

To construct a pairwise grid of data for specific covariate values, pass a dictionary to `conditional`. The values of the dictionary can be of type `int`, `float`, `list`, or `np.ndarray`.


```python
conditional = {
    "camper": np.array([0, 1]),
    "persons": np.arange(1, 5, 1),
    "child": np.array([1, 2, 3]),
}
user_passed_grid = data_grid(fish_model, conditional)
user_passed_grid.query("camper == 0")
```

    Default computed for unspecified variable: livebait





<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>camper</th>
      <th>persons</th>
      <th>child</th>
      <th>livebait</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>0</td>
      <td>1</td>
      <td>2</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>0</td>
      <td>1</td>
      <td>3</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>0</td>
      <td>2</td>
      <td>1</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>0</td>
      <td>2</td>
      <td>2</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>5</th>
      <td>0</td>
      <td>2</td>
      <td>3</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>6</th>
      <td>0</td>
      <td>3</td>
      <td>1</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>7</th>
      <td>0</td>
      <td>3</td>
      <td>2</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>8</th>
      <td>0</td>
      <td>3</td>
      <td>3</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>9</th>
      <td>0</td>
      <td>4</td>
      <td>1</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>10</th>
      <td>0</td>
      <td>4</td>
      <td>2</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>11</th>
      <td>0</td>
      <td>4</td>
      <td>3</td>
      <td>1.0</td>
    </tr>
  </tbody>
</table>
</div>



Subsetting by `camper = 0`, it can be seen that a combination of all possible pairs of values from the dictionary (including the unspecified variable `livebait`) results in a dataframe containing every possible combination of values from the original sets. `livebait` has been set to 1 as this is the mode of the unspecified categorical variable.

### Default values

Alternatively, a list of covariates can be passed to `conditional` where the elements are the names of the covariates to use in the grid. By doing this, you are telling `interpret` to compute default values for these covariates. The psuedocode below outlines the logic and functions used to compute these default values:

```python
if is_numeric_dtype(x) or is_float_dtype(x):
    values = np.linspace(np.min(x), np.max(x), 50)

elif is_integer_dtype(x):
    values = np.quantile(x, np.linspace(0, 1, 5))

elif is_categorical_dtype(x) or is_string_dtype(x) or is_object_dtype(x):
    values = np.unique(x)
```


```python
conditional = ["camper", "persons", "child"]
default_grid = data_grid(fish_model, conditional)

default_grid.shape, user_passed_grid.shape
```

    Default computed for conditional variable: camper, persons, child
    Default computed for unspecified variable: livebait





    ((32, 4), (24, 4))



Notice how the resulting length is different between the user passed and default grid. This is due to the fact that values for `child` range from 0 to 3 for the default grid.


```python
default_grid.query("camper == 0")
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>camper</th>
      <th>persons</th>
      <th>child</th>
      <th>livebait</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0.0</td>
      <td>1</td>
      <td>0</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>0.0</td>
      <td>1</td>
      <td>1</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>0.0</td>
      <td>1</td>
      <td>2</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>0.0</td>
      <td>1</td>
      <td>3</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>0.0</td>
      <td>2</td>
      <td>0</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>5</th>
      <td>0.0</td>
      <td>2</td>
      <td>1</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>6</th>
      <td>0.0</td>
      <td>2</td>
      <td>2</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>7</th>
      <td>0.0</td>
      <td>2</td>
      <td>3</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>8</th>
      <td>0.0</td>
      <td>3</td>
      <td>0</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>9</th>
      <td>0.0</td>
      <td>3</td>
      <td>1</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>10</th>
      <td>0.0</td>
      <td>3</td>
      <td>2</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>11</th>
      <td>0.0</td>
      <td>3</td>
      <td>3</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>12</th>
      <td>0.0</td>
      <td>4</td>
      <td>0</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>13</th>
      <td>0.0</td>
      <td>4</td>
      <td>1</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>14</th>
      <td>0.0</td>
      <td>4</td>
      <td>2</td>
      <td>1.0</td>
    </tr>
    <tr>
      <th>15</th>
      <td>0.0</td>
      <td>4</td>
      <td>3</td>
      <td>1.0</td>
    </tr>
  </tbody>
</table>
</div>



## Compute comparisons

To use `data_grid` to help generate data in computing comparisons or slopes, additional data is passed to the optional `variable` parameter. The name `variable` is an abstraction for the comparisons parameter `contrast` and slopes parameter `wrt`. If you have used any of the `interpret` functions, these parameter names should be familiar and the use of `data_grid` should be analogous to `comparisons`, `predictions`, and `slopes`.

`variable` can also be passed user-provided data (as a dictionary), or a string indicating the name of the covariate of interest. If the latter, a default value will be computed. Additionally, if an argument is passed for `variable`, then the `effect_type` needs to be passed. This is because for `comparisons` and `slopes` an epsilon value `eps` needs to be determined to compute the centered and finite difference, respectively. You can also pass a value for `eps` as a kwarg.


```python
conditional = {
    "camper": np.array([0, 1]),
    "persons": np.arange(1, 5, 1),
    "child": np.array([1, 2, 3, 4])
}
variable = "livebait"

grid = data_grid(fish_model, conditional, variable, effect_type="comparisons")
```

    Default computed for contrast variable: livebait



```python
idata_grid = fish_model.predict(fish_idata, data=grid, inplace=False)
```

### Select draws conditional on data

The second helper function to aid in more advanced analysis is `select_draws`. This is a function that selects the posterior or posterior predictive draws from the ArviZ [InferenceData](https://python.arviz.org/en/stable/getting_started/WorkingWithInferenceData.html) object returned by `model.predict` given a conditional dictionary. The conditional dictionary represents the values that correspond to that draw. 

For example, if we wanted to select posterior draws where `livebait = [0, 1]`, then all we need to do is pass a dictionary where the key is the name of the covariate and the value is the value that we want to condition on (or select). The resulting InferenceData object will contain the draws that correspond to the data points where `livebait = [0, 1]`. Additionally, you must pass the InferenceData object returned by `model.predict`, the data used to generate the predictions, and the name of the data variable `data_var` you would like to select from the InferenceData posterior group. If you specified to return the posterior predictive samples by passing `model.predict(..., kind="pps")`, you can use this group instead of the posterior group by passing `group="posterior_predictive"`.

Below, it is demonstrated how to compute comparisons for `count_mean` for the contrast `livebait = [0, 1]` using the posterior draws.


```python
idata_grid
```





            <div>
              <div class='xr-header'>
                <div class="xr-obj-type">arviz.InferenceData</div>
              </div>
              <ul class="xr-sections group-sections">

            <li class = "xr-section-item">
                  <input id="idata_posteriorfc9a1fda-11ae-41ac-92f2-1b880679f029" class="xr-section-summary-in" type="checkbox">
                  <label for="idata_posteriorfc9a1fda-11ae-41ac-92f2-1b880679f029" class = "xr-section-summary">posterior</label>
                  <div class="xr-section-inline-details"></div>
                  <div class="xr-section-details">
                      <ul id="xr-dataset-coord-list" class="xr-var-list">
                          <div style="padding-left:2rem;"><div><svg style="position: absolute; width: 0; height: 0; overflow: hidden">
<defs>
<symbol id="icon-database" viewBox="0 0 32 32">
<path d="M16 0c-8.837 0-16 2.239-16 5v4c0 2.761 7.163 5 16 5s16-2.239 16-5v-4c0-2.761-7.163-5-16-5z"></path>
<path d="M16 17c-8.837 0-16-2.239-16-5v6c0 2.761 7.163 5 16 5s16-2.239 16-5v-6c0 2.761-7.163 5-16 5z"></path>
<path d="M16 26c-8.837 0-16-2.239-16-5v6c0 2.761 7.163 5 16 5s16-2.239 16-5v-6c0 2.761-7.163 5-16 5z"></path>
</symbol>
<symbol id="icon-file-text2" viewBox="0 0 32 32">
<path d="M28.681 7.159c-0.694-0.947-1.662-2.053-2.724-3.116s-2.169-2.030-3.116-2.724c-1.612-1.182-2.393-1.319-2.841-1.319h-15.5c-1.378 0-2.5 1.121-2.5 2.5v27c0 1.378 1.122 2.5 2.5 2.5h23c1.378 0 2.5-1.122 2.5-2.5v-19.5c0-0.448-0.137-1.23-1.319-2.841zM24.543 5.457c0.959 0.959 1.712 1.825 2.268 2.543h-4.811v-4.811c0.718 0.556 1.584 1.309 2.543 2.268zM28 29.5c0 0.271-0.229 0.5-0.5 0.5h-23c-0.271 0-0.5-0.229-0.5-0.5v-27c0-0.271 0.229-0.5 0.5-0.5 0 0 15.499-0 15.5 0v7c0 0.552 0.448 1 1 1h7v19.5z"></path>
<path d="M23 26h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
<path d="M23 22h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
<path d="M23 18h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
</symbol>
</defs>
</svg>
<style>/* CSS stylesheet for displaying xarray objects in jupyterlab.
 *
 */

:root {
  --xr-font-color0: var(--jp-content-font-color0, rgba(0, 0, 0, 1));
  --xr-font-color2: var(--jp-content-font-color2, rgba(0, 0, 0, 0.54));
  --xr-font-color3: var(--jp-content-font-color3, rgba(0, 0, 0, 0.38));
  --xr-border-color: var(--jp-border-color2, #e0e0e0);
  --xr-disabled-color: var(--jp-layout-color3, #bdbdbd);
  --xr-background-color: var(--jp-layout-color0, white);
  --xr-background-color-row-even: var(--jp-layout-color1, white);
  --xr-background-color-row-odd: var(--jp-layout-color2, #eeeeee);
}

html[theme=dark],
body[data-theme=dark],
body.vscode-dark {
  --xr-font-color0: rgba(255, 255, 255, 1);
  --xr-font-color2: rgba(255, 255, 255, 0.54);
  --xr-font-color3: rgba(255, 255, 255, 0.38);
  --xr-border-color: #1F1F1F;
  --xr-disabled-color: #515151;
  --xr-background-color: #111111;
  --xr-background-color-row-even: #111111;
  --xr-background-color-row-odd: #313131;
}

.xr-wrap {
  display: block !important;
  min-width: 300px;
  max-width: 700px;
}

.xr-text-repr-fallback {
  /* fallback to plain text repr when CSS is not injected (untrusted notebook) */
  display: none;
}

.xr-header {
  padding-top: 6px;
  padding-bottom: 6px;
  margin-bottom: 4px;
  border-bottom: solid 1px var(--xr-border-color);
}

.xr-header > div,
.xr-header > ul {
  display: inline;
  margin-top: 0;
  margin-bottom: 0;
}

.xr-obj-type,
.xr-array-name {
  margin-left: 2px;
  margin-right: 10px;
}

.xr-obj-type {
  color: var(--xr-font-color2);
}

.xr-sections {
  padding-left: 0 !important;
  display: grid;
  grid-template-columns: 150px auto auto 1fr 20px 20px;
}

.xr-section-item {
  display: contents;
}

.xr-section-item input {
  display: none;
}

.xr-section-item input + label {
  color: var(--xr-disabled-color);
}

.xr-section-item input:enabled + label {
  cursor: pointer;
  color: var(--xr-font-color2);
}

.xr-section-item input:enabled + label:hover {
  color: var(--xr-font-color0);
}

.xr-section-summary {
  grid-column: 1;
  color: var(--xr-font-color2);
  font-weight: 500;
}

.xr-section-summary > span {
  display: inline-block;
  padding-left: 0.5em;
}

.xr-section-summary-in:disabled + label {
  color: var(--xr-font-color2);
}

.xr-section-summary-in + label:before {
  display: inline-block;
  content: '►';
  font-size: 11px;
  width: 15px;
  text-align: center;
}

.xr-section-summary-in:disabled + label:before {
  color: var(--xr-disabled-color);
}

.xr-section-summary-in:checked + label:before {
  content: '▼';
}

.xr-section-summary-in:checked + label > span {
  display: none;
}

.xr-section-summary,
.xr-section-inline-details {
  padding-top: 4px;
  padding-bottom: 4px;
}

.xr-section-inline-details {
  grid-column: 2 / -1;
}

.xr-section-details {
  display: none;
  grid-column: 1 / -1;
  margin-bottom: 5px;
}

.xr-section-summary-in:checked ~ .xr-section-details {
  display: contents;
}

.xr-array-wrap {
  grid-column: 1 / -1;
  display: grid;
  grid-template-columns: 20px auto;
}

.xr-array-wrap > label {
  grid-column: 1;
  vertical-align: top;
}

.xr-preview {
  color: var(--xr-font-color3);
}

.xr-array-preview,
.xr-array-data {
  padding: 0 5px !important;
  grid-column: 2;
}

.xr-array-data,
.xr-array-in:checked ~ .xr-array-preview {
  display: none;
}

.xr-array-in:checked ~ .xr-array-data,
.xr-array-preview {
  display: inline-block;
}

.xr-dim-list {
  display: inline-block !important;
  list-style: none;
  padding: 0 !important;
  margin: 0;
}

.xr-dim-list li {
  display: inline-block;
  padding: 0;
  margin: 0;
}

.xr-dim-list:before {
  content: '(';
}

.xr-dim-list:after {
  content: ')';
}

.xr-dim-list li:not(:last-child):after {
  content: ',';
  padding-right: 5px;
}

.xr-has-index {
  font-weight: bold;
}

.xr-var-list,
.xr-var-item {
  display: contents;
}

.xr-var-item > div,
.xr-var-item label,
.xr-var-item > .xr-var-name span {
  background-color: var(--xr-background-color-row-even);
  margin-bottom: 0;
}

.xr-var-item > .xr-var-name:hover span {
  padding-right: 5px;
}

.xr-var-list > li:nth-child(odd) > div,
.xr-var-list > li:nth-child(odd) > label,
.xr-var-list > li:nth-child(odd) > .xr-var-name span {
  background-color: var(--xr-background-color-row-odd);
}

.xr-var-name {
  grid-column: 1;
}

.xr-var-dims {
  grid-column: 2;
}

.xr-var-dtype {
  grid-column: 3;
  text-align: right;
  color: var(--xr-font-color2);
}

.xr-var-preview {
  grid-column: 4;
}

.xr-index-preview {
  grid-column: 2 / 5;
  color: var(--xr-font-color2);
}

.xr-var-name,
.xr-var-dims,
.xr-var-dtype,
.xr-preview,
.xr-attrs dt {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  padding-right: 10px;
}

.xr-var-name:hover,
.xr-var-dims:hover,
.xr-var-dtype:hover,
.xr-attrs dt:hover {
  overflow: visible;
  width: auto;
  z-index: 1;
}

.xr-var-attrs,
.xr-var-data,
.xr-index-data {
  display: none;
  background-color: var(--xr-background-color) !important;
  padding-bottom: 5px !important;
}

.xr-var-attrs-in:checked ~ .xr-var-attrs,
.xr-var-data-in:checked ~ .xr-var-data,
.xr-index-data-in:checked ~ .xr-index-data {
  display: block;
}

.xr-var-data > table {
  float: right;
}

.xr-var-name span,
.xr-var-data,
.xr-index-name div,
.xr-index-data,
.xr-attrs {
  padding-left: 25px !important;
}

.xr-attrs,
.xr-var-attrs,
.xr-var-data,
.xr-index-data {
  grid-column: 1 / -1;
}

dl.xr-attrs {
  padding: 0;
  margin: 0;
  display: grid;
  grid-template-columns: 125px auto;
}

.xr-attrs dt,
.xr-attrs dd {
  padding: 0;
  margin: 0;
  float: left;
  padding-right: 10px;
  width: auto;
}

.xr-attrs dt {
  font-weight: normal;
  grid-column: 1;
}

.xr-attrs dt:hover span {
  display: inline-block;
  background: var(--xr-background-color);
  padding-right: 10px;
}

.xr-attrs dd {
  grid-column: 2;
  white-space: pre-wrap;
  word-break: break-all;
}

.xr-icon-database,
.xr-icon-file-text2,
.xr-no-icon {
  display: inline-block;
  vertical-align: middle;
  width: 1em;
  height: 1.5em !important;
  stroke-width: 0;
  stroke: currentColor;
  fill: currentColor;
}
</style><pre class='xr-text-repr-fallback'>&lt;xarray.Dataset&gt;
Dimensions:       (chain: 4, draw: 1000, livebait_dim: 1, camper_dim: 1,
                   count_obs: 64)
Coordinates:
  * chain         (chain) int64 0 1 2 3
  * draw          (draw) int64 0 1 2 3 4 5 6 7 ... 993 994 995 996 997 998 999
  * livebait_dim  (livebait_dim) &lt;U3 &#x27;1.0&#x27;
  * camper_dim    (camper_dim) &lt;U3 &#x27;1.0&#x27;
  * count_obs     (count_obs) int64 0 1 2 3 4 5 6 7 ... 56 57 58 59 60 61 62 63
Data variables:
    Intercept     (chain, draw) float64 -2.454 -2.31 -2.91 ... -2.652 -2.887
    livebait      (chain, draw, livebait_dim) float64 1.629 1.58 ... 1.799 1.967
    camper        (chain, draw, camper_dim) float64 0.7037 0.7089 ... 0.7128
    persons       (chain, draw) float64 0.8707 0.8369 0.9457 ... 0.8847 0.912
    child         (chain, draw) float64 -1.345 -1.412 -1.418 ... -1.293 -1.573
    count_psi     (chain, draw) float64 0.6311 0.6201 0.6342 ... 0.6768 0.5745
    count_mean    (chain, draw, count_obs) float64 0.05349 0.2728 ... 0.05777
Attributes:
    created_at:                  2023-12-09T06:01:42.032935
    arviz_version:               0.16.1
    inference_library:           pymc
    inference_library_version:   5.8.1
    sampling_time:               2.652681827545166
    tuning_steps:                1000
    modeling_interface:          bambi
    modeling_interface_version:  0.13.0.dev0</pre><div class='xr-wrap' style='display:none'><div class='xr-header'><div class='xr-obj-type'>xarray.Dataset</div></div><ul class='xr-sections'><li class='xr-section-item'><input id='section-32bde36f-5825-4fa3-a395-8fba2cac83f3' class='xr-section-summary-in' type='checkbox' disabled ><label for='section-32bde36f-5825-4fa3-a395-8fba2cac83f3' class='xr-section-summary'  title='Expand/collapse section'>Dimensions:</label><div class='xr-section-inline-details'><ul class='xr-dim-list'><li><span class='xr-has-index'>chain</span>: 4</li><li><span class='xr-has-index'>draw</span>: 1000</li><li><span class='xr-has-index'>livebait_dim</span>: 1</li><li><span class='xr-has-index'>camper_dim</span>: 1</li><li><span class='xr-has-index'>count_obs</span>: 64</li></ul></div><div class='xr-section-details'></div></li><li class='xr-section-item'><input id='section-83c44492-6998-4ea4-bb90-0f85e7666556' class='xr-section-summary-in' type='checkbox'  checked><label for='section-83c44492-6998-4ea4-bb90-0f85e7666556' class='xr-section-summary' >Coordinates: <span>(5)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>chain</span></div><div class='xr-var-dims'>(chain)</div><div class='xr-var-dtype'>int64</div><div class='xr-var-preview xr-preview'>0 1 2 3</div><input id='attrs-3186d4e4-573b-48bd-a785-11437686e6c4' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-3186d4e4-573b-48bd-a785-11437686e6c4' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-5a97ac99-ab9f-4b53-a774-53aa45f36ed3' class='xr-var-data-in' type='checkbox'><label for='data-5a97ac99-ab9f-4b53-a774-53aa45f36ed3' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([0, 1, 2, 3])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>draw</span></div><div class='xr-var-dims'>(draw)</div><div class='xr-var-dtype'>int64</div><div class='xr-var-preview xr-preview'>0 1 2 3 4 5 ... 995 996 997 998 999</div><input id='attrs-e5ed31dc-e2b6-489e-9dff-6677d4bc1a79' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-e5ed31dc-e2b6-489e-9dff-6677d4bc1a79' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-88a0c93c-f239-448d-9680-0f696d16debc' class='xr-var-data-in' type='checkbox'><label for='data-88a0c93c-f239-448d-9680-0f696d16debc' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([  0,   1,   2, ..., 997, 998, 999])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>livebait_dim</span></div><div class='xr-var-dims'>(livebait_dim)</div><div class='xr-var-dtype'>&lt;U3</div><div class='xr-var-preview xr-preview'>&#x27;1.0&#x27;</div><input id='attrs-f44b6b87-f03d-4b16-8547-908eb9c5ec3b' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-f44b6b87-f03d-4b16-8547-908eb9c5ec3b' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-a686494c-1302-48c9-92bc-8f993e862aac' class='xr-var-data-in' type='checkbox'><label for='data-a686494c-1302-48c9-92bc-8f993e862aac' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([&#x27;1.0&#x27;], dtype=&#x27;&lt;U3&#x27;)</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>camper_dim</span></div><div class='xr-var-dims'>(camper_dim)</div><div class='xr-var-dtype'>&lt;U3</div><div class='xr-var-preview xr-preview'>&#x27;1.0&#x27;</div><input id='attrs-accdb2ea-bbaa-4ad8-bed9-a1d6d4af2a2e' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-accdb2ea-bbaa-4ad8-bed9-a1d6d4af2a2e' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-0bf052e3-adc7-4c94-8aec-8b8eee81317b' class='xr-var-data-in' type='checkbox'><label for='data-0bf052e3-adc7-4c94-8aec-8b8eee81317b' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([&#x27;1.0&#x27;], dtype=&#x27;&lt;U3&#x27;)</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>count_obs</span></div><div class='xr-var-dims'>(count_obs)</div><div class='xr-var-dtype'>int64</div><div class='xr-var-preview xr-preview'>0 1 2 3 4 5 6 ... 58 59 60 61 62 63</div><input id='attrs-f997a288-50bd-4726-9f1f-07d4c8fb762e' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-f997a288-50bd-4726-9f1f-07d4c8fb762e' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-d093d05a-7484-443d-bd8d-eca2ed5e9414' class='xr-var-data-in' type='checkbox'><label for='data-d093d05a-7484-443d-bd8d-eca2ed5e9414' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([ 0,  1,  2,  3,  4,  5,  6,  7,  8,  9, 10, 11, 12, 13, 14, 15, 16, 17,
       18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35,
       36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53,
       54, 55, 56, 57, 58, 59, 60, 61, 62, 63])</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-e55d0a41-cc60-4590-86da-d6c74b8c5617' class='xr-section-summary-in' type='checkbox'  checked><label for='section-e55d0a41-cc60-4590-86da-d6c74b8c5617' class='xr-section-summary' >Data variables: <span>(7)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span>Intercept</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>-2.454 -2.31 ... -2.652 -2.887</div><input id='attrs-8e9f63f3-1cc7-45d5-a28f-a385d43ebf9d' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-8e9f63f3-1cc7-45d5-a28f-a385d43ebf9d' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-2cccd8c5-46a0-404f-8417-6cbd85162605' class='xr-var-data-in' type='checkbox'><label for='data-2cccd8c5-46a0-404f-8417-6cbd85162605' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[-2.45369792, -2.31017959, -2.91023957, ..., -2.12445232,
        -2.13810127, -2.40815127],
       [-2.21843503, -2.81570517, -2.81570517, ..., -3.23806838,
        -2.43778314, -2.40094452],
       [-2.57030767, -2.71197967, -3.13096258, ..., -2.75122519,
        -2.81845915, -2.77702878],
       [-2.27327526, -2.76201826, -2.72978474, ..., -2.43555099,
        -2.65150277, -2.88742403]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>livebait</span></div><div class='xr-var-dims'>(chain, draw, livebait_dim)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>1.629 1.58 1.858 ... 1.799 1.967</div><input id='attrs-f51f524f-364a-46f5-8df7-c4d7937a1c04' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-f51f524f-364a-46f5-8df7-c4d7937a1c04' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-fff5a5b8-1971-48dd-80fa-9db398f0393b' class='xr-var-data-in' type='checkbox'><label for='data-fff5a5b8-1971-48dd-80fa-9db398f0393b' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[[1.62921077],
        [1.57971577],
        [1.85814553],
        ...,
        [1.39320044],
        [1.31938529],
        [2.0303655 ]],

       [[1.44441014],
        [2.03180494],
        [2.03180494],
        ...,
        [2.00383874],
        [2.0219552 ],
        [1.77554631]],

       [[1.44183823],
        [2.12560898],
        [1.84301056],
        ...,
        [1.65875902],
        [1.69372851],
        [1.78166201]],

       [[1.64438961],
        [1.88379206],
        [1.95128032],
        ...,
        [1.73718918],
        [1.79946112],
        [1.96668979]]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>camper</span></div><div class='xr-var-dims'>(chain, draw, camper_dim)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>0.7037 0.7089 ... 0.6445 0.7128</div><input id='attrs-40aacba3-b34e-4919-ba91-e25972660932' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-40aacba3-b34e-4919-ba91-e25972660932' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-85cd9b3b-53a6-4915-9c2a-051d7cb5a285' class='xr-var-data-in' type='checkbox'><label for='data-85cd9b3b-53a6-4915-9c2a-051d7cb5a285' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[[0.70368097],
        [0.70888183],
        [0.6432408 ],
        ...,
        [0.64457622],
        [0.68265053],
        [0.59371655]],

       [[0.73004824],
        [0.63806292],
        [0.63806292],
        ...,
        [0.92181193],
        [0.44431118],
        [0.52257123]],

       [[0.67251782],
        [0.61970006],
        [0.80110109],
        ...,
        [0.71626836],
        [0.7386686 ],
        [0.80126267]],

       [[0.68695406],
        [0.66147853],
        [0.7129354 ],
        ...,
        [0.62523222],
        [0.64449304],
        [0.71282881]]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>persons</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>0.8707 0.8369 ... 0.8847 0.912</div><input id='attrs-4d32e7db-5fff-456c-8d3d-e320971b3676' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-4d32e7db-5fff-456c-8d3d-e320971b3676' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-0c0a2108-faa7-4803-91b4-d3a387204de5' class='xr-var-data-in' type='checkbox'><label for='data-0c0a2108-faa7-4803-91b4-d3a387204de5' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[0.87073326, 0.83691824, 0.94570525, ..., 0.86127639, 0.85807333,
        0.7827932 ],
       [0.85684603, 0.88279463, 0.88279463, ..., 0.93383647, 0.82809407,
        0.84387704],
       [0.93868593, 0.83511835, 0.9966305 , ..., 0.92864502, 0.9583216 ,
        0.9046778 ],
       [0.82278573, 0.89846761, 0.85979591, ..., 0.85493891, 0.88465389,
        0.91197904]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>child</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>-1.345 -1.412 ... -1.293 -1.573</div><input id='attrs-815b2853-643d-47e6-97f9-b2dd5b976e9b' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-815b2853-643d-47e6-97f9-b2dd5b976e9b' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-61f5f6e4-f31d-419f-8eea-8300adbf76f3' class='xr-var-data-in' type='checkbox'><label for='data-61f5f6e4-f31d-419f-8eea-8300adbf76f3' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[-1.34537519, -1.41232517, -1.41808649, ..., -1.28364874,
        -1.21311483, -1.343049  ],
       [-1.36022475, -1.38360737, -1.38360737, ..., -1.34137848,
        -1.37005354, -1.32761054],
       [-1.42361978, -1.45390876, -1.59921868, ..., -1.57170719,
        -1.60156825, -1.49976267],
       [-1.33972561, -1.29366616, -1.38679881, ..., -1.29405166,
        -1.29343353, -1.57284431]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>count_psi</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>0.6311 0.6201 ... 0.6768 0.5745</div><input id='attrs-5d4204bb-5565-470c-8864-35ae4cc474b3' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-5d4204bb-5565-470c-8864-35ae4cc474b3' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-035b0d61-a5e1-4bd1-8004-5213e07c89f9' class='xr-var-data-in' type='checkbox'><label for='data-035b0d61-a5e1-4bd1-8004-5213e07c89f9' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[0.63114698, 0.62012886, 0.63423439, ..., 0.57386427, 0.56121415,
        0.57771985],
       [0.61291154, 0.607627  , 0.607627  , ..., 0.63015121, 0.54061116,
        0.54315861],
       [0.60778057, 0.68015035, 0.57385421, ..., 0.75656914, 0.74951665,
        0.68320354],
       [0.61202421, 0.58032736, 0.57834964, ..., 0.65743098, 0.67681655,
        0.57454481]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>count_mean</span></div><div class='xr-var-dims'>(chain, draw, count_obs)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>0.05349 0.2728 ... 0.008082 0.05777</div><input id='attrs-f7f52342-9bbc-40b7-936b-20d24df4af63' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-f7f52342-9bbc-40b7-936b-20d24df4af63' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-51753934-d6ba-4512-8153-2c41892a936e' class='xr-var-data-in' type='checkbox'><label for='data-51753934-d6ba-4512-8153-2c41892a936e' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[[0.05348576, 0.27276925, 0.01392994, ..., 0.50966648,
         0.02602794, 0.13273855],
        [0.05582204, 0.27093652, 0.01359692, ..., 0.40216834,
         0.02018278, 0.09795866],
        [0.03395834, 0.21773528, 0.00822393, ..., 0.41466193,
         0.01566191, 0.10042158],
        ...,
        [0.07833   , 0.31549128, 0.02169934, ..., 0.61108678,
         0.04203026, 0.16928611],
        [0.08264981, 0.30920293, 0.0245693 , ..., 0.70955544,
         0.05638135, 0.21092947],
        [0.0513851 , 0.3913936 , 0.013414  , ..., 0.50558281,
         0.01732754, 0.13198164]],

       [[0.06575538, 0.27876013, 0.01687303, ..., 0.49794444,
         0.03014001, 0.12777409],
        [0.03627894, 0.27673   , 0.00909414, ..., 0.46511021,
         0.01528485, 0.11659041],
        [0.03627894, 0.27673   , 0.00909414, ..., 0.46511021,
         0.01528485, 0.11659041],
...
        [0.03356447, 0.17630703, 0.00697101, ..., 0.25240022,
         0.00997967, 0.05242108],
        [0.03137619, 0.17067787, 0.00632482, ..., 0.25730828,
         0.00953508, 0.05186824],
        [0.03431703, 0.20383353, 0.00765898, ..., 0.34140667,
         0.01282825, 0.07619621]],

       [[0.061408  , 0.31796133, 0.01608383, ..., 0.51172625,
         0.02588528, 0.13403007],
        [0.04254398, 0.27987149, 0.01166826, ..., 0.60418447,
         0.02518935, 0.16570571],
        [0.03851191, 0.271035  , 0.00962312, ..., 0.45530771,
         0.01616574, 0.11376952],
        ...,
        [0.05643511, 0.32062773, 0.01547212, ..., 0.5853596 ,
         0.02824695, 0.16048086],
        [0.04687446, 0.28342116, 0.01285894, ..., 0.57739214,
         0.02619653, 0.1583944 ],
        [0.02877382, 0.2056459 , 0.00596925, ..., 0.27844848,
         0.00808248, 0.05776533]]])</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-ce75c25d-2e30-4407-a6b6-f81e1ee59a12' class='xr-section-summary-in' type='checkbox'  ><label for='section-ce75c25d-2e30-4407-a6b6-f81e1ee59a12' class='xr-section-summary' >Indexes: <span>(5)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-index-name'><div>chain</div></div><div class='xr-index-preview'>PandasIndex</div><div></div><input id='index-4927113c-57f5-4fbe-81cd-619d63126ba1' class='xr-index-data-in' type='checkbox'/><label for='index-4927113c-57f5-4fbe-81cd-619d63126ba1' title='Show/Hide index repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-index-data'><pre>PandasIndex(Index([0, 1, 2, 3], dtype=&#x27;int64&#x27;, name=&#x27;chain&#x27;))</pre></div></li><li class='xr-var-item'><div class='xr-index-name'><div>draw</div></div><div class='xr-index-preview'>PandasIndex</div><div></div><input id='index-25f46950-96d5-455e-9476-dccc56f82839' class='xr-index-data-in' type='checkbox'/><label for='index-25f46950-96d5-455e-9476-dccc56f82839' title='Show/Hide index repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-index-data'><pre>PandasIndex(Index([  0,   1,   2,   3,   4,   5,   6,   7,   8,   9,
       ...
       990, 991, 992, 993, 994, 995, 996, 997, 998, 999],
      dtype=&#x27;int64&#x27;, name=&#x27;draw&#x27;, length=1000))</pre></div></li><li class='xr-var-item'><div class='xr-index-name'><div>livebait_dim</div></div><div class='xr-index-preview'>PandasIndex</div><div></div><input id='index-8088ad52-e239-4dc3-8b15-9a9ccaa3c911' class='xr-index-data-in' type='checkbox'/><label for='index-8088ad52-e239-4dc3-8b15-9a9ccaa3c911' title='Show/Hide index repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-index-data'><pre>PandasIndex(Index([&#x27;1.0&#x27;], dtype=&#x27;object&#x27;, name=&#x27;livebait_dim&#x27;))</pre></div></li><li class='xr-var-item'><div class='xr-index-name'><div>camper_dim</div></div><div class='xr-index-preview'>PandasIndex</div><div></div><input id='index-7d6d9322-d819-4baa-9522-281b6aba3650' class='xr-index-data-in' type='checkbox'/><label for='index-7d6d9322-d819-4baa-9522-281b6aba3650' title='Show/Hide index repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-index-data'><pre>PandasIndex(Index([&#x27;1.0&#x27;], dtype=&#x27;object&#x27;, name=&#x27;camper_dim&#x27;))</pre></div></li><li class='xr-var-item'><div class='xr-index-name'><div>count_obs</div></div><div class='xr-index-preview'>PandasIndex</div><div></div><input id='index-bbf959d3-5b51-44d8-a428-735b50c22a0a' class='xr-index-data-in' type='checkbox'/><label for='index-bbf959d3-5b51-44d8-a428-735b50c22a0a' title='Show/Hide index repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-index-data'><pre>PandasIndex(Index([ 0,  1,  2,  3,  4,  5,  6,  7,  8,  9, 10, 11, 12, 13, 14, 15, 16, 17,
       18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35,
       36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47, 48, 49, 50, 51, 52, 53,
       54, 55, 56, 57, 58, 59, 60, 61, 62, 63],
      dtype=&#x27;int64&#x27;, name=&#x27;count_obs&#x27;))</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-20dfa257-8b9c-40bf-a5e0-389c32f418ec' class='xr-section-summary-in' type='checkbox'  checked><label for='section-20dfa257-8b9c-40bf-a5e0-389c32f418ec' class='xr-section-summary' >Attributes: <span>(8)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><dl class='xr-attrs'><dt><span>created_at :</span></dt><dd>2023-12-09T06:01:42.032935</dd><dt><span>arviz_version :</span></dt><dd>0.16.1</dd><dt><span>inference_library :</span></dt><dd>pymc</dd><dt><span>inference_library_version :</span></dt><dd>5.8.1</dd><dt><span>sampling_time :</span></dt><dd>2.652681827545166</dd><dt><span>tuning_steps :</span></dt><dd>1000</dd><dt><span>modeling_interface :</span></dt><dd>bambi</dd><dt><span>modeling_interface_version :</span></dt><dd>0.13.0.dev0</dd></dl></div></li></ul></div></div><br></div>
                      </ul>
                  </div>
            </li>

            <li class = "xr-section-item">
                  <input id="idata_sample_stats726cc3b2-08d4-48b5-ac93-60ae1fe79973" class="xr-section-summary-in" type="checkbox">
                  <label for="idata_sample_stats726cc3b2-08d4-48b5-ac93-60ae1fe79973" class = "xr-section-summary">sample_stats</label>
                  <div class="xr-section-inline-details"></div>
                  <div class="xr-section-details">
                      <ul id="xr-dataset-coord-list" class="xr-var-list">
                          <div style="padding-left:2rem;"><div><svg style="position: absolute; width: 0; height: 0; overflow: hidden">
<defs>
<symbol id="icon-database" viewBox="0 0 32 32">
<path d="M16 0c-8.837 0-16 2.239-16 5v4c0 2.761 7.163 5 16 5s16-2.239 16-5v-4c0-2.761-7.163-5-16-5z"></path>
<path d="M16 17c-8.837 0-16-2.239-16-5v6c0 2.761 7.163 5 16 5s16-2.239 16-5v-6c0 2.761-7.163 5-16 5z"></path>
<path d="M16 26c-8.837 0-16-2.239-16-5v6c0 2.761 7.163 5 16 5s16-2.239 16-5v-6c0 2.761-7.163 5-16 5z"></path>
</symbol>
<symbol id="icon-file-text2" viewBox="0 0 32 32">
<path d="M28.681 7.159c-0.694-0.947-1.662-2.053-2.724-3.116s-2.169-2.030-3.116-2.724c-1.612-1.182-2.393-1.319-2.841-1.319h-15.5c-1.378 0-2.5 1.121-2.5 2.5v27c0 1.378 1.122 2.5 2.5 2.5h23c1.378 0 2.5-1.122 2.5-2.5v-19.5c0-0.448-0.137-1.23-1.319-2.841zM24.543 5.457c0.959 0.959 1.712 1.825 2.268 2.543h-4.811v-4.811c0.718 0.556 1.584 1.309 2.543 2.268zM28 29.5c0 0.271-0.229 0.5-0.5 0.5h-23c-0.271 0-0.5-0.229-0.5-0.5v-27c0-0.271 0.229-0.5 0.5-0.5 0 0 15.499-0 15.5 0v7c0 0.552 0.448 1 1 1h7v19.5z"></path>
<path d="M23 26h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
<path d="M23 22h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
<path d="M23 18h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
</symbol>
</defs>
</svg>
<style>/* CSS stylesheet for displaying xarray objects in jupyterlab.
 *
 */

:root {
  --xr-font-color0: var(--jp-content-font-color0, rgba(0, 0, 0, 1));
  --xr-font-color2: var(--jp-content-font-color2, rgba(0, 0, 0, 0.54));
  --xr-font-color3: var(--jp-content-font-color3, rgba(0, 0, 0, 0.38));
  --xr-border-color: var(--jp-border-color2, #e0e0e0);
  --xr-disabled-color: var(--jp-layout-color3, #bdbdbd);
  --xr-background-color: var(--jp-layout-color0, white);
  --xr-background-color-row-even: var(--jp-layout-color1, white);
  --xr-background-color-row-odd: var(--jp-layout-color2, #eeeeee);
}

html[theme=dark],
body[data-theme=dark],
body.vscode-dark {
  --xr-font-color0: rgba(255, 255, 255, 1);
  --xr-font-color2: rgba(255, 255, 255, 0.54);
  --xr-font-color3: rgba(255, 255, 255, 0.38);
  --xr-border-color: #1F1F1F;
  --xr-disabled-color: #515151;
  --xr-background-color: #111111;
  --xr-background-color-row-even: #111111;
  --xr-background-color-row-odd: #313131;
}

.xr-wrap {
  display: block !important;
  min-width: 300px;
  max-width: 700px;
}

.xr-text-repr-fallback {
  /* fallback to plain text repr when CSS is not injected (untrusted notebook) */
  display: none;
}

.xr-header {
  padding-top: 6px;
  padding-bottom: 6px;
  margin-bottom: 4px;
  border-bottom: solid 1px var(--xr-border-color);
}

.xr-header > div,
.xr-header > ul {
  display: inline;
  margin-top: 0;
  margin-bottom: 0;
}

.xr-obj-type,
.xr-array-name {
  margin-left: 2px;
  margin-right: 10px;
}

.xr-obj-type {
  color: var(--xr-font-color2);
}

.xr-sections {
  padding-left: 0 !important;
  display: grid;
  grid-template-columns: 150px auto auto 1fr 20px 20px;
}

.xr-section-item {
  display: contents;
}

.xr-section-item input {
  display: none;
}

.xr-section-item input + label {
  color: var(--xr-disabled-color);
}

.xr-section-item input:enabled + label {
  cursor: pointer;
  color: var(--xr-font-color2);
}

.xr-section-item input:enabled + label:hover {
  color: var(--xr-font-color0);
}

.xr-section-summary {
  grid-column: 1;
  color: var(--xr-font-color2);
  font-weight: 500;
}

.xr-section-summary > span {
  display: inline-block;
  padding-left: 0.5em;
}

.xr-section-summary-in:disabled + label {
  color: var(--xr-font-color2);
}

.xr-section-summary-in + label:before {
  display: inline-block;
  content: '►';
  font-size: 11px;
  width: 15px;
  text-align: center;
}

.xr-section-summary-in:disabled + label:before {
  color: var(--xr-disabled-color);
}

.xr-section-summary-in:checked + label:before {
  content: '▼';
}

.xr-section-summary-in:checked + label > span {
  display: none;
}

.xr-section-summary,
.xr-section-inline-details {
  padding-top: 4px;
  padding-bottom: 4px;
}

.xr-section-inline-details {
  grid-column: 2 / -1;
}

.xr-section-details {
  display: none;
  grid-column: 1 / -1;
  margin-bottom: 5px;
}

.xr-section-summary-in:checked ~ .xr-section-details {
  display: contents;
}

.xr-array-wrap {
  grid-column: 1 / -1;
  display: grid;
  grid-template-columns: 20px auto;
}

.xr-array-wrap > label {
  grid-column: 1;
  vertical-align: top;
}

.xr-preview {
  color: var(--xr-font-color3);
}

.xr-array-preview,
.xr-array-data {
  padding: 0 5px !important;
  grid-column: 2;
}

.xr-array-data,
.xr-array-in:checked ~ .xr-array-preview {
  display: none;
}

.xr-array-in:checked ~ .xr-array-data,
.xr-array-preview {
  display: inline-block;
}

.xr-dim-list {
  display: inline-block !important;
  list-style: none;
  padding: 0 !important;
  margin: 0;
}

.xr-dim-list li {
  display: inline-block;
  padding: 0;
  margin: 0;
}

.xr-dim-list:before {
  content: '(';
}

.xr-dim-list:after {
  content: ')';
}

.xr-dim-list li:not(:last-child):after {
  content: ',';
  padding-right: 5px;
}

.xr-has-index {
  font-weight: bold;
}

.xr-var-list,
.xr-var-item {
  display: contents;
}

.xr-var-item > div,
.xr-var-item label,
.xr-var-item > .xr-var-name span {
  background-color: var(--xr-background-color-row-even);
  margin-bottom: 0;
}

.xr-var-item > .xr-var-name:hover span {
  padding-right: 5px;
}

.xr-var-list > li:nth-child(odd) > div,
.xr-var-list > li:nth-child(odd) > label,
.xr-var-list > li:nth-child(odd) > .xr-var-name span {
  background-color: var(--xr-background-color-row-odd);
}

.xr-var-name {
  grid-column: 1;
}

.xr-var-dims {
  grid-column: 2;
}

.xr-var-dtype {
  grid-column: 3;
  text-align: right;
  color: var(--xr-font-color2);
}

.xr-var-preview {
  grid-column: 4;
}

.xr-index-preview {
  grid-column: 2 / 5;
  color: var(--xr-font-color2);
}

.xr-var-name,
.xr-var-dims,
.xr-var-dtype,
.xr-preview,
.xr-attrs dt {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  padding-right: 10px;
}

.xr-var-name:hover,
.xr-var-dims:hover,
.xr-var-dtype:hover,
.xr-attrs dt:hover {
  overflow: visible;
  width: auto;
  z-index: 1;
}

.xr-var-attrs,
.xr-var-data,
.xr-index-data {
  display: none;
  background-color: var(--xr-background-color) !important;
  padding-bottom: 5px !important;
}

.xr-var-attrs-in:checked ~ .xr-var-attrs,
.xr-var-data-in:checked ~ .xr-var-data,
.xr-index-data-in:checked ~ .xr-index-data {
  display: block;
}

.xr-var-data > table {
  float: right;
}

.xr-var-name span,
.xr-var-data,
.xr-index-name div,
.xr-index-data,
.xr-attrs {
  padding-left: 25px !important;
}

.xr-attrs,
.xr-var-attrs,
.xr-var-data,
.xr-index-data {
  grid-column: 1 / -1;
}

dl.xr-attrs {
  padding: 0;
  margin: 0;
  display: grid;
  grid-template-columns: 125px auto;
}

.xr-attrs dt,
.xr-attrs dd {
  padding: 0;
  margin: 0;
  float: left;
  padding-right: 10px;
  width: auto;
}

.xr-attrs dt {
  font-weight: normal;
  grid-column: 1;
}

.xr-attrs dt:hover span {
  display: inline-block;
  background: var(--xr-background-color);
  padding-right: 10px;
}

.xr-attrs dd {
  grid-column: 2;
  white-space: pre-wrap;
  word-break: break-all;
}

.xr-icon-database,
.xr-icon-file-text2,
.xr-no-icon {
  display: inline-block;
  vertical-align: middle;
  width: 1em;
  height: 1.5em !important;
  stroke-width: 0;
  stroke: currentColor;
  fill: currentColor;
}
</style><pre class='xr-text-repr-fallback'>&lt;xarray.Dataset&gt;
Dimensions:                (chain: 4, draw: 1000)
Coordinates:
  * chain                  (chain) int64 0 1 2 3
  * draw                   (draw) int64 0 1 2 3 4 5 ... 994 995 996 997 998 999
Data variables: (12/17)
    largest_eigval         (chain, draw) float64 nan nan nan nan ... nan nan nan
    max_energy_error       (chain, draw) float64 0.595 1.645 ... -0.2578 0.4279
    tree_depth             (chain, draw) int64 3 3 3 3 3 3 3 3 ... 3 3 3 3 3 4 4
    process_time_diff      (chain, draw) float64 0.000894 0.000888 ... 0.001654
    diverging              (chain, draw) bool False False False ... False False
    step_size_bar          (chain, draw) float64 0.4502 0.4502 ... 0.4541 0.4541
    ...                     ...
    acceptance_rate        (chain, draw) float64 0.7512 0.5492 ... 0.9919 0.8264
    n_steps                (chain, draw) float64 7.0 7.0 7.0 ... 7.0 11.0 15.0
    energy                 (chain, draw) float64 751.9 752.2 ... 755.7 756.8
    index_in_trajectory    (chain, draw) int64 6 4 -6 -5 -2 4 ... 7 1 4 4 -1 9
    lp                     (chain, draw) float64 -750.1 -750.2 ... -751.9 -754.0
    energy_error           (chain, draw) float64 0.04379 0.02705 ... 0.415
Attributes:
    created_at:                  2023-12-09T06:01:42.040761
    arviz_version:               0.16.1
    inference_library:           pymc
    inference_library_version:   5.8.1
    sampling_time:               2.652681827545166
    tuning_steps:                1000
    modeling_interface:          bambi
    modeling_interface_version:  0.13.0.dev0</pre><div class='xr-wrap' style='display:none'><div class='xr-header'><div class='xr-obj-type'>xarray.Dataset</div></div><ul class='xr-sections'><li class='xr-section-item'><input id='section-57817a28-46cb-4441-ade9-a0acbf7cb433' class='xr-section-summary-in' type='checkbox' disabled ><label for='section-57817a28-46cb-4441-ade9-a0acbf7cb433' class='xr-section-summary'  title='Expand/collapse section'>Dimensions:</label><div class='xr-section-inline-details'><ul class='xr-dim-list'><li><span class='xr-has-index'>chain</span>: 4</li><li><span class='xr-has-index'>draw</span>: 1000</li></ul></div><div class='xr-section-details'></div></li><li class='xr-section-item'><input id='section-665ce732-e5e6-41cc-992e-2370655207c2' class='xr-section-summary-in' type='checkbox'  checked><label for='section-665ce732-e5e6-41cc-992e-2370655207c2' class='xr-section-summary' >Coordinates: <span>(2)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>chain</span></div><div class='xr-var-dims'>(chain)</div><div class='xr-var-dtype'>int64</div><div class='xr-var-preview xr-preview'>0 1 2 3</div><input id='attrs-fbf03521-9c98-4b9b-843d-9cc5a2c32510' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-fbf03521-9c98-4b9b-843d-9cc5a2c32510' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-48084846-363a-415f-b6d6-394e3df3dc5e' class='xr-var-data-in' type='checkbox'><label for='data-48084846-363a-415f-b6d6-394e3df3dc5e' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([0, 1, 2, 3])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>draw</span></div><div class='xr-var-dims'>(draw)</div><div class='xr-var-dtype'>int64</div><div class='xr-var-preview xr-preview'>0 1 2 3 4 5 ... 995 996 997 998 999</div><input id='attrs-f3be30e6-5645-46ac-b578-db2ec6529972' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-f3be30e6-5645-46ac-b578-db2ec6529972' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-a3f2b253-9c3d-41ad-a22b-b7157412a76f' class='xr-var-data-in' type='checkbox'><label for='data-a3f2b253-9c3d-41ad-a22b-b7157412a76f' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([  0,   1,   2, ..., 997, 998, 999])</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-29be3e6e-8f04-4c13-926a-d0847548aa1e' class='xr-section-summary-in' type='checkbox'  ><label for='section-29be3e6e-8f04-4c13-926a-d0847548aa1e' class='xr-section-summary' >Data variables: <span>(17)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span>largest_eigval</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>nan nan nan nan ... nan nan nan nan</div><input id='attrs-78b6688f-519e-40a2-88d9-5a2511749ff6' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-78b6688f-519e-40a2-88d9-5a2511749ff6' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-c5f511fb-3461-49a9-ae3f-9cfad70bca18' class='xr-var-data-in' type='checkbox'><label for='data-c5f511fb-3461-49a9-ae3f-9cfad70bca18' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>max_energy_error</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>0.595 1.645 ... -0.2578 0.4279</div><input id='attrs-25c242ef-d957-433f-a600-0bb613c1b470' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-25c242ef-d957-433f-a600-0bb613c1b470' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-84ffae91-1044-4c53-b019-361a138f6889' class='xr-var-data-in' type='checkbox'><label for='data-84ffae91-1044-4c53-b019-361a138f6889' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[ 0.595031  ,  1.64520146,  0.21527509, ...,  0.78075972,
         5.80798519,  0.43782396],
       [-0.37301973, -0.20402685,  2.5627162 , ...,  0.36741839,
         3.6828103 , -0.50001872],
       [ 1.27621281, -0.68221199,  0.44120051, ...,  2.43620146,
        -0.08386685, -0.22408853],
       [ 0.14004462,  0.99694174, -0.07864808, ...,  0.22969662,
        -0.25775022,  0.42794799]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>tree_depth</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>int64</div><div class='xr-var-preview xr-preview'>3 3 3 3 3 3 3 3 ... 3 3 3 3 3 3 4 4</div><input id='attrs-492041f1-c472-4051-8757-a0e54e347075' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-492041f1-c472-4051-8757-a0e54e347075' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-e85847a0-6ef3-436e-bc71-0a91117a8298' class='xr-var-data-in' type='checkbox'><label for='data-e85847a0-6ef3-436e-bc71-0a91117a8298' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[3, 3, 3, ..., 3, 2, 3],
       [3, 3, 3, ..., 3, 4, 3],
       [3, 3, 3, ..., 3, 2, 3],
       [3, 3, 4, ..., 3, 4, 4]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>process_time_diff</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>0.000894 0.000888 ... 0.001654</div><input id='attrs-5f4324f7-f2a1-4cc8-b139-38a3e8a4627f' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-5f4324f7-f2a1-4cc8-b139-38a3e8a4627f' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-91470dc1-d897-4370-ac5b-07a213e22b43' class='xr-var-data-in' type='checkbox'><label for='data-91470dc1-d897-4370-ac5b-07a213e22b43' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[0.000894, 0.000888, 0.000882, ..., 0.00088 , 0.000451, 0.000883],
       [0.000893, 0.000889, 0.000886, ..., 0.000891, 0.001756, 0.000893],
       [0.000895, 0.000891, 0.000886, ..., 0.000868, 0.00044 , 0.000865],
       [0.000894, 0.000895, 0.001761, ..., 0.000882, 0.001246, 0.001654]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>diverging</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>bool</div><div class='xr-var-preview xr-preview'>False False False ... False False</div><input id='attrs-4c456609-5e73-4577-a989-1d5333ef622f' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-4c456609-5e73-4577-a989-1d5333ef622f' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-9cdb35cb-c0a2-414b-9073-882ceb421bc3' class='xr-var-data-in' type='checkbox'><label for='data-9cdb35cb-c0a2-414b-9073-882ceb421bc3' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[False, False, False, ..., False, False, False],
       [False, False, False, ..., False, False, False],
       [False, False, False, ..., False, False, False],
       [False, False, False, ..., False, False, False]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>step_size_bar</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>0.4502 0.4502 ... 0.4541 0.4541</div><input id='attrs-7da496c9-e27e-4056-91d3-96b489096645' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-7da496c9-e27e-4056-91d3-96b489096645' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-c8377341-e431-4dbb-b169-db47c14a709f' class='xr-var-data-in' type='checkbox'><label for='data-c8377341-e431-4dbb-b169-db47c14a709f' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[0.45015324, 0.45015324, 0.45015324, ..., 0.45015324, 0.45015324,
        0.45015324],
       [0.42774415, 0.42774415, 0.42774415, ..., 0.42774415, 0.42774415,
        0.42774415],
       [0.44696235, 0.44696235, 0.44696235, ..., 0.44696235, 0.44696235,
        0.44696235],
       [0.45405445, 0.45405445, 0.45405445, ..., 0.45405445, 0.45405445,
        0.45405445]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>perf_counter_start</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>1.067e+03 1.067e+03 ... 1.069e+03</div><input id='attrs-43bf1afb-a2ce-48f0-8a8f-870882b27d43' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-43bf1afb-a2ce-48f0-8a8f-870882b27d43' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-284e52b8-b594-45d1-a21a-deb477dfeacf' class='xr-var-data-in' type='checkbox'><label for='data-284e52b8-b594-45d1-a21a-deb477dfeacf' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[1067.40390204, 1067.40486017, 1067.40580821, ..., 1068.46292929,
        1068.46387517, 1068.46438762],
       [1067.41511467, 1067.41606821, 1067.41704075, ..., 1068.53776558,
        1068.53871471, 1068.54055937],
       [1067.46033633, 1067.461301  , 1067.46227354, ..., 1068.56037796,
        1068.56130604, 1068.56180675],
       [1067.44745625, 1067.44842704, 1067.44939542, ..., 1068.58570321,
        1068.58666521, 1068.58797033]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>step_size</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>0.415 0.415 0.415 ... 0.3918 0.3918</div><input id='attrs-8df6da90-b64b-4546-ace6-452bf648b04b' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-8df6da90-b64b-4546-ace6-452bf648b04b' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-89d01ea3-714b-4bfa-8015-843be8c9452b' class='xr-var-data-in' type='checkbox'><label for='data-89d01ea3-714b-4bfa-8015-843be8c9452b' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[0.41500359, 0.41500359, 0.41500359, ..., 0.41500359, 0.41500359,
        0.41500359],
       [0.42993239, 0.42993239, 0.42993239, ..., 0.42993239, 0.42993239,
        0.42993239],
       [0.55799239, 0.55799239, 0.55799239, ..., 0.55799239, 0.55799239,
        0.55799239],
       [0.39177214, 0.39177214, 0.39177214, ..., 0.39177214, 0.39177214,
        0.39177214]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>perf_counter_diff</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>0.000894 0.0008885 ... 0.001654</div><input id='attrs-9c130fc2-b91e-4067-98be-34b0c70a7caf' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-9c130fc2-b91e-4067-98be-34b0c70a7caf' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-bd40b533-5dca-4ccf-866d-c39e3abc89be' class='xr-var-data-in' type='checkbox'><label for='data-bd40b533-5dca-4ccf-866d-c39e3abc89be' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[0.000894  , 0.00088854, 0.00088317, ..., 0.00087933, 0.00045042,
        0.00088333],
       [0.00089242, 0.00088767, 0.00088663, ..., 0.00089146, 0.00178467,
        0.0008925 ],
       [0.00089392, 0.00089162, 0.00088608, ..., 0.00086804, 0.00044117,
        0.00086496],
       [0.00089838, 0.00089992, 0.00178875, ..., 0.00090033, 0.00124583,
        0.00165379]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>reached_max_treedepth</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>bool</div><div class='xr-var-preview xr-preview'>False False False ... False False</div><input id='attrs-e6f84e79-89e8-464a-9b05-12d1ce302257' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-e6f84e79-89e8-464a-9b05-12d1ce302257' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-64730413-8bf4-4168-833c-ca1d92af4dc9' class='xr-var-data-in' type='checkbox'><label for='data-64730413-8bf4-4168-833c-ca1d92af4dc9' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[False, False, False, ..., False, False, False],
       [False, False, False, ..., False, False, False],
       [False, False, False, ..., False, False, False],
       [False, False, False, ..., False, False, False]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>smallest_eigval</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>nan nan nan nan ... nan nan nan nan</div><input id='attrs-384cc264-f265-4883-920d-82832e2d70f1' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-384cc264-f265-4883-920d-82832e2d70f1' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-8e5dfa67-cde7-4f0b-8cb5-34891c1d4eb2' class='xr-var-data-in' type='checkbox'><label for='data-8e5dfa67-cde7-4f0b-8cb5-34891c1d4eb2' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan],
       [nan, nan, nan, ..., nan, nan, nan]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>acceptance_rate</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>0.7512 0.5492 ... 0.9919 0.8264</div><input id='attrs-d68fb00b-0e01-41e8-b561-9a2dba0b1990' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-d68fb00b-0e01-41e8-b561-9a2dba0b1990' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-8bbc02fd-70f4-467d-a278-63f12d1fe2a8' class='xr-var-data-in' type='checkbox'><label for='data-8bbc02fd-70f4-467d-a278-63f12d1fe2a8' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[0.75118358, 0.54917103, 0.90600545, ..., 0.64640509, 0.24651666,
        0.84703996],
       [1.        , 1.        , 0.18921988, ..., 0.81098596, 0.34035628,
        0.99696537],
       [0.56560841, 0.95846614, 0.85780695, ..., 0.50105863, 0.99947793,
        1.        ],
       [0.9683953 , 0.61509461, 0.99814479, ..., 0.9439049 , 0.99189283,
        0.82635188]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>n_steps</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>7.0 7.0 7.0 7.0 ... 7.0 11.0 15.0</div><input id='attrs-8b2eb680-adb5-4087-a2da-9056f089d417' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-8b2eb680-adb5-4087-a2da-9056f089d417' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-11af0f56-ec98-4fdd-b46b-432e00b86286' class='xr-var-data-in' type='checkbox'><label for='data-11af0f56-ec98-4fdd-b46b-432e00b86286' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[ 7.,  7.,  7., ...,  7.,  3.,  7.],
       [ 7.,  7.,  7., ...,  7., 15.,  7.],
       [ 7.,  7.,  7., ...,  7.,  3.,  7.],
       [ 7.,  7., 15., ...,  7., 11., 15.]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>energy</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>751.9 752.2 751.9 ... 755.7 756.8</div><input id='attrs-c43f6a87-608e-4722-8472-32316acfc953' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-c43f6a87-608e-4722-8472-32316acfc953' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-cc7506f8-1bf1-4902-b7bb-0624f7bc7fcf' class='xr-var-data-in' type='checkbox'><label for='data-cc7506f8-1bf1-4902-b7bb-0624f7bc7fcf' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[751.94493597, 752.16775838, 751.86589668, ..., 757.02411223,
        757.59305462, 755.76371432],
       [754.28875075, 752.65123698, 755.84607135, ..., 759.92489206,
        761.1929198 , 755.68207248],
       [755.10969541, 756.38178297, 761.18044623, ..., 762.22995695,
        756.69732237, 756.35418424],
       [753.32406793, 752.80673891, 752.03157068, ..., 756.22896884,
        755.6715562 , 756.78221912]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>index_in_trajectory</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>int64</div><div class='xr-var-preview xr-preview'>6 4 -6 -5 -2 4 -5 ... 7 1 4 4 -1 9</div><input id='attrs-2653162f-a146-4e56-b13c-c472d82a876f' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-2653162f-a146-4e56-b13c-c472d82a876f' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-97a9940a-00a7-4557-baca-7ec9b641b2c3' class='xr-var-data-in' type='checkbox'><label for='data-97a9940a-00a7-4557-baca-7ec9b641b2c3' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[ 6,  4, -6, ...,  4,  2,  5],
       [ 4,  7,  0, ..., -6,  9, -2],
       [ 3,  5,  5, ..., -4, -2,  4],
       [-3, -4, -3, ...,  4, -1,  9]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>lp</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>-750.1 -750.2 ... -751.9 -754.0</div><input id='attrs-5c0c6f30-2306-47f0-8d5f-c5a58904f836' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-5c0c6f30-2306-47f0-8d5f-c5a58904f836' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-080482d6-60db-4a66-9442-ebf1fb09f432' class='xr-var-data-in' type='checkbox'><label for='data-080482d6-60db-4a66-9442-ebf1fb09f432' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[-750.0635191 , -750.19379111, -751.09897583, ..., -751.87190426,
        -752.95441117, -754.23172742],
       [-751.17173283, -750.81535676, -750.81535676, ..., -755.33966632,
        -755.44844073, -752.33208106],
       [-754.06157469, -753.90911474, -759.1519945 , ..., -755.85090606,
        -755.67682047, -752.00709578],
       [-750.85175653, -751.04878911, -750.6370331 , ..., -751.95509665,
        -751.89343509, -753.95045712]])</pre></div></li><li class='xr-var-item'><div class='xr-var-name'><span>energy_error</span></div><div class='xr-var-dims'>(chain, draw)</div><div class='xr-var-dtype'>float64</div><div class='xr-var-preview xr-preview'>0.04379 0.02705 ... -0.1224 0.415</div><input id='attrs-0efc5f1d-3a6a-4d41-9f43-e8e20724c014' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-0efc5f1d-3a6a-4d41-9f43-e8e20724c014' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-14052c16-319b-4e41-85ef-a561d5184522' class='xr-var-data-in' type='checkbox'><label for='data-14052c16-319b-4e41-85ef-a561d5184522' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([[ 4.37855069e-02,  2.70547404e-02, -5.88277115e-04, ...,
         8.64705248e-02,  3.20765941e-01,  4.37823957e-01],
       [-9.78110746e-02, -4.02245905e-02,  0.00000000e+00, ...,
         2.05920633e-01,  2.91692817e-01, -2.47079734e-01],
       [ 1.14647984e+00, -6.82211987e-01,  4.41200505e-01, ...,
        -3.52379476e-01,  1.56742695e-03, -1.73449488e-01],
       [ 1.45574755e-02, -4.02199593e-03, -5.80109339e-02, ...,
        -9.46547980e-03, -1.22382092e-01,  4.15048801e-01]])</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-e5067510-ad2a-4eed-8fd6-94a6dc7c415a' class='xr-section-summary-in' type='checkbox'  ><label for='section-e5067510-ad2a-4eed-8fd6-94a6dc7c415a' class='xr-section-summary' >Indexes: <span>(2)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-index-name'><div>chain</div></div><div class='xr-index-preview'>PandasIndex</div><div></div><input id='index-82115c2b-204a-448b-8ff4-8ce63a5cca6b' class='xr-index-data-in' type='checkbox'/><label for='index-82115c2b-204a-448b-8ff4-8ce63a5cca6b' title='Show/Hide index repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-index-data'><pre>PandasIndex(Index([0, 1, 2, 3], dtype=&#x27;int64&#x27;, name=&#x27;chain&#x27;))</pre></div></li><li class='xr-var-item'><div class='xr-index-name'><div>draw</div></div><div class='xr-index-preview'>PandasIndex</div><div></div><input id='index-b585c361-a252-4e4c-8107-6be14585160c' class='xr-index-data-in' type='checkbox'/><label for='index-b585c361-a252-4e4c-8107-6be14585160c' title='Show/Hide index repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-index-data'><pre>PandasIndex(Index([  0,   1,   2,   3,   4,   5,   6,   7,   8,   9,
       ...
       990, 991, 992, 993, 994, 995, 996, 997, 998, 999],
      dtype=&#x27;int64&#x27;, name=&#x27;draw&#x27;, length=1000))</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-e5e2b511-8ade-4ef5-bf13-6a424791e3ee' class='xr-section-summary-in' type='checkbox'  checked><label for='section-e5e2b511-8ade-4ef5-bf13-6a424791e3ee' class='xr-section-summary' >Attributes: <span>(8)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><dl class='xr-attrs'><dt><span>created_at :</span></dt><dd>2023-12-09T06:01:42.040761</dd><dt><span>arviz_version :</span></dt><dd>0.16.1</dd><dt><span>inference_library :</span></dt><dd>pymc</dd><dt><span>inference_library_version :</span></dt><dd>5.8.1</dd><dt><span>sampling_time :</span></dt><dd>2.652681827545166</dd><dt><span>tuning_steps :</span></dt><dd>1000</dd><dt><span>modeling_interface :</span></dt><dd>bambi</dd><dt><span>modeling_interface_version :</span></dt><dd>0.13.0.dev0</dd></dl></div></li></ul></div></div><br></div>
                      </ul>
                  </div>
            </li>

            <li class = "xr-section-item">
                  <input id="idata_observed_data1b746639-c161-454d-863d-bcf33dc59f06" class="xr-section-summary-in" type="checkbox">
                  <label for="idata_observed_data1b746639-c161-454d-863d-bcf33dc59f06" class = "xr-section-summary">observed_data</label>
                  <div class="xr-section-inline-details"></div>
                  <div class="xr-section-details">
                      <ul id="xr-dataset-coord-list" class="xr-var-list">
                          <div style="padding-left:2rem;"><div><svg style="position: absolute; width: 0; height: 0; overflow: hidden">
<defs>
<symbol id="icon-database" viewBox="0 0 32 32">
<path d="M16 0c-8.837 0-16 2.239-16 5v4c0 2.761 7.163 5 16 5s16-2.239 16-5v-4c0-2.761-7.163-5-16-5z"></path>
<path d="M16 17c-8.837 0-16-2.239-16-5v6c0 2.761 7.163 5 16 5s16-2.239 16-5v-6c0 2.761-7.163 5-16 5z"></path>
<path d="M16 26c-8.837 0-16-2.239-16-5v6c0 2.761 7.163 5 16 5s16-2.239 16-5v-6c0 2.761-7.163 5-16 5z"></path>
</symbol>
<symbol id="icon-file-text2" viewBox="0 0 32 32">
<path d="M28.681 7.159c-0.694-0.947-1.662-2.053-2.724-3.116s-2.169-2.030-3.116-2.724c-1.612-1.182-2.393-1.319-2.841-1.319h-15.5c-1.378 0-2.5 1.121-2.5 2.5v27c0 1.378 1.122 2.5 2.5 2.5h23c1.378 0 2.5-1.122 2.5-2.5v-19.5c0-0.448-0.137-1.23-1.319-2.841zM24.543 5.457c0.959 0.959 1.712 1.825 2.268 2.543h-4.811v-4.811c0.718 0.556 1.584 1.309 2.543 2.268zM28 29.5c0 0.271-0.229 0.5-0.5 0.5h-23c-0.271 0-0.5-0.229-0.5-0.5v-27c0-0.271 0.229-0.5 0.5-0.5 0 0 15.499-0 15.5 0v7c0 0.552 0.448 1 1 1h7v19.5z"></path>
<path d="M23 26h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
<path d="M23 22h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
<path d="M23 18h-14c-0.552 0-1-0.448-1-1s0.448-1 1-1h14c0.552 0 1 0.448 1 1s-0.448 1-1 1z"></path>
</symbol>
</defs>
</svg>
<style>/* CSS stylesheet for displaying xarray objects in jupyterlab.
 *
 */

:root {
  --xr-font-color0: var(--jp-content-font-color0, rgba(0, 0, 0, 1));
  --xr-font-color2: var(--jp-content-font-color2, rgba(0, 0, 0, 0.54));
  --xr-font-color3: var(--jp-content-font-color3, rgba(0, 0, 0, 0.38));
  --xr-border-color: var(--jp-border-color2, #e0e0e0);
  --xr-disabled-color: var(--jp-layout-color3, #bdbdbd);
  --xr-background-color: var(--jp-layout-color0, white);
  --xr-background-color-row-even: var(--jp-layout-color1, white);
  --xr-background-color-row-odd: var(--jp-layout-color2, #eeeeee);
}

html[theme=dark],
body[data-theme=dark],
body.vscode-dark {
  --xr-font-color0: rgba(255, 255, 255, 1);
  --xr-font-color2: rgba(255, 255, 255, 0.54);
  --xr-font-color3: rgba(255, 255, 255, 0.38);
  --xr-border-color: #1F1F1F;
  --xr-disabled-color: #515151;
  --xr-background-color: #111111;
  --xr-background-color-row-even: #111111;
  --xr-background-color-row-odd: #313131;
}

.xr-wrap {
  display: block !important;
  min-width: 300px;
  max-width: 700px;
}

.xr-text-repr-fallback {
  /* fallback to plain text repr when CSS is not injected (untrusted notebook) */
  display: none;
}

.xr-header {
  padding-top: 6px;
  padding-bottom: 6px;
  margin-bottom: 4px;
  border-bottom: solid 1px var(--xr-border-color);
}

.xr-header > div,
.xr-header > ul {
  display: inline;
  margin-top: 0;
  margin-bottom: 0;
}

.xr-obj-type,
.xr-array-name {
  margin-left: 2px;
  margin-right: 10px;
}

.xr-obj-type {
  color: var(--xr-font-color2);
}

.xr-sections {
  padding-left: 0 !important;
  display: grid;
  grid-template-columns: 150px auto auto 1fr 20px 20px;
}

.xr-section-item {
  display: contents;
}

.xr-section-item input {
  display: none;
}

.xr-section-item input + label {
  color: var(--xr-disabled-color);
}

.xr-section-item input:enabled + label {
  cursor: pointer;
  color: var(--xr-font-color2);
}

.xr-section-item input:enabled + label:hover {
  color: var(--xr-font-color0);
}

.xr-section-summary {
  grid-column: 1;
  color: var(--xr-font-color2);
  font-weight: 500;
}

.xr-section-summary > span {
  display: inline-block;
  padding-left: 0.5em;
}

.xr-section-summary-in:disabled + label {
  color: var(--xr-font-color2);
}

.xr-section-summary-in + label:before {
  display: inline-block;
  content: '►';
  font-size: 11px;
  width: 15px;
  text-align: center;
}

.xr-section-summary-in:disabled + label:before {
  color: var(--xr-disabled-color);
}

.xr-section-summary-in:checked + label:before {
  content: '▼';
}

.xr-section-summary-in:checked + label > span {
  display: none;
}

.xr-section-summary,
.xr-section-inline-details {
  padding-top: 4px;
  padding-bottom: 4px;
}

.xr-section-inline-details {
  grid-column: 2 / -1;
}

.xr-section-details {
  display: none;
  grid-column: 1 / -1;
  margin-bottom: 5px;
}

.xr-section-summary-in:checked ~ .xr-section-details {
  display: contents;
}

.xr-array-wrap {
  grid-column: 1 / -1;
  display: grid;
  grid-template-columns: 20px auto;
}

.xr-array-wrap > label {
  grid-column: 1;
  vertical-align: top;
}

.xr-preview {
  color: var(--xr-font-color3);
}

.xr-array-preview,
.xr-array-data {
  padding: 0 5px !important;
  grid-column: 2;
}

.xr-array-data,
.xr-array-in:checked ~ .xr-array-preview {
  display: none;
}

.xr-array-in:checked ~ .xr-array-data,
.xr-array-preview {
  display: inline-block;
}

.xr-dim-list {
  display: inline-block !important;
  list-style: none;
  padding: 0 !important;
  margin: 0;
}

.xr-dim-list li {
  display: inline-block;
  padding: 0;
  margin: 0;
}

.xr-dim-list:before {
  content: '(';
}

.xr-dim-list:after {
  content: ')';
}

.xr-dim-list li:not(:last-child):after {
  content: ',';
  padding-right: 5px;
}

.xr-has-index {
  font-weight: bold;
}

.xr-var-list,
.xr-var-item {
  display: contents;
}

.xr-var-item > div,
.xr-var-item label,
.xr-var-item > .xr-var-name span {
  background-color: var(--xr-background-color-row-even);
  margin-bottom: 0;
}

.xr-var-item > .xr-var-name:hover span {
  padding-right: 5px;
}

.xr-var-list > li:nth-child(odd) > div,
.xr-var-list > li:nth-child(odd) > label,
.xr-var-list > li:nth-child(odd) > .xr-var-name span {
  background-color: var(--xr-background-color-row-odd);
}

.xr-var-name {
  grid-column: 1;
}

.xr-var-dims {
  grid-column: 2;
}

.xr-var-dtype {
  grid-column: 3;
  text-align: right;
  color: var(--xr-font-color2);
}

.xr-var-preview {
  grid-column: 4;
}

.xr-index-preview {
  grid-column: 2 / 5;
  color: var(--xr-font-color2);
}

.xr-var-name,
.xr-var-dims,
.xr-var-dtype,
.xr-preview,
.xr-attrs dt {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  padding-right: 10px;
}

.xr-var-name:hover,
.xr-var-dims:hover,
.xr-var-dtype:hover,
.xr-attrs dt:hover {
  overflow: visible;
  width: auto;
  z-index: 1;
}

.xr-var-attrs,
.xr-var-data,
.xr-index-data {
  display: none;
  background-color: var(--xr-background-color) !important;
  padding-bottom: 5px !important;
}

.xr-var-attrs-in:checked ~ .xr-var-attrs,
.xr-var-data-in:checked ~ .xr-var-data,
.xr-index-data-in:checked ~ .xr-index-data {
  display: block;
}

.xr-var-data > table {
  float: right;
}

.xr-var-name span,
.xr-var-data,
.xr-index-name div,
.xr-index-data,
.xr-attrs {
  padding-left: 25px !important;
}

.xr-attrs,
.xr-var-attrs,
.xr-var-data,
.xr-index-data {
  grid-column: 1 / -1;
}

dl.xr-attrs {
  padding: 0;
  margin: 0;
  display: grid;
  grid-template-columns: 125px auto;
}

.xr-attrs dt,
.xr-attrs dd {
  padding: 0;
  margin: 0;
  float: left;
  padding-right: 10px;
  width: auto;
}

.xr-attrs dt {
  font-weight: normal;
  grid-column: 1;
}

.xr-attrs dt:hover span {
  display: inline-block;
  background: var(--xr-background-color);
  padding-right: 10px;
}

.xr-attrs dd {
  grid-column: 2;
  white-space: pre-wrap;
  word-break: break-all;
}

.xr-icon-database,
.xr-icon-file-text2,
.xr-no-icon {
  display: inline-block;
  vertical-align: middle;
  width: 1em;
  height: 1.5em !important;
  stroke-width: 0;
  stroke: currentColor;
  fill: currentColor;
}
</style><pre class='xr-text-repr-fallback'>&lt;xarray.Dataset&gt;
Dimensions:    (count_obs: 250)
Coordinates:
  * count_obs  (count_obs) int64 0 1 2 3 4 5 6 7 ... 243 244 245 246 247 248 249
Data variables:
    count      (count_obs) int64 0 0 0 0 1 0 0 0 0 1 0 ... 4 1 1 0 1 0 0 0 0 0 0
Attributes:
    created_at:                  2023-12-09T06:01:42.043721
    arviz_version:               0.16.1
    inference_library:           pymc
    inference_library_version:   5.8.1
    modeling_interface:          bambi
    modeling_interface_version:  0.13.0.dev0</pre><div class='xr-wrap' style='display:none'><div class='xr-header'><div class='xr-obj-type'>xarray.Dataset</div></div><ul class='xr-sections'><li class='xr-section-item'><input id='section-364be521-072e-45e7-a293-8840b64eafdf' class='xr-section-summary-in' type='checkbox' disabled ><label for='section-364be521-072e-45e7-a293-8840b64eafdf' class='xr-section-summary'  title='Expand/collapse section'>Dimensions:</label><div class='xr-section-inline-details'><ul class='xr-dim-list'><li><span class='xr-has-index'>count_obs</span>: 250</li></ul></div><div class='xr-section-details'></div></li><li class='xr-section-item'><input id='section-fde04a8e-67f3-4fa0-b962-28e52f58e9d7' class='xr-section-summary-in' type='checkbox'  checked><label for='section-fde04a8e-67f3-4fa0-b962-28e52f58e9d7' class='xr-section-summary' >Coordinates: <span>(1)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span class='xr-has-index'>count_obs</span></div><div class='xr-var-dims'>(count_obs)</div><div class='xr-var-dtype'>int64</div><div class='xr-var-preview xr-preview'>0 1 2 3 4 5 ... 245 246 247 248 249</div><input id='attrs-a9b7fd36-6934-4ea1-bdeb-25428f1e6f16' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-a9b7fd36-6934-4ea1-bdeb-25428f1e6f16' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-d2fbb151-b2e0-4ec9-9514-8ee260812c87' class='xr-var-data-in' type='checkbox'><label for='data-d2fbb151-b2e0-4ec9-9514-8ee260812c87' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([  0,   1,   2, ..., 247, 248, 249])</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-58f557a1-67e5-4302-a8e8-4530aa455194' class='xr-section-summary-in' type='checkbox'  checked><label for='section-58f557a1-67e5-4302-a8e8-4530aa455194' class='xr-section-summary' >Data variables: <span>(1)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-var-name'><span>count</span></div><div class='xr-var-dims'>(count_obs)</div><div class='xr-var-dtype'>int64</div><div class='xr-var-preview xr-preview'>0 0 0 0 1 0 0 0 ... 0 1 0 0 0 0 0 0</div><input id='attrs-3e23f24d-0d37-4010-9c26-f1f7311a8cb0' class='xr-var-attrs-in' type='checkbox' disabled><label for='attrs-3e23f24d-0d37-4010-9c26-f1f7311a8cb0' title='Show/Hide attributes'><svg class='icon xr-icon-file-text2'><use xlink:href='#icon-file-text2'></use></svg></label><input id='data-18850491-bdc3-4dbd-9e74-57f92de2b93d' class='xr-var-data-in' type='checkbox'><label for='data-18850491-bdc3-4dbd-9e74-57f92de2b93d' title='Show/Hide data repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-var-attrs'><dl class='xr-attrs'></dl></div><div class='xr-var-data'><pre>array([  0,   0,   0,   0,   1,   0,   0,   0,   0,   1,   0,   0,   1,
         2,   0,   1,   0,   0,   1,   0,   1,   5,   0,   3,  30,   0,
        13,   0,   0,   0,   0,   0,  11,   5,   0,   1,   1,   7,   0,
        14,   0,  32,   0,   1,   0,   0,   0,   1,   5,   0,   1,   0,
        22,   0,  15,   0,   0,   0,   5,   4,   2,   0,   2,  32,   0,
         0,   1,   0,   0,   0,   7,   0,   0,   0,   0,   0,   0,   0,
         0,   2,   3,   1,   5,   0,   2,   1,   0,   1, 149,   0,   1,
         0,   0,   1,   0,   0,   0,   2,   2,  29,   3,   0,   0,   5,
         0,   0,   0,   0,   0,   1,   7,   1,   0,   2,   0,   2,   0,
         0,   0,   1,   0,   0,   0,   0,   0,   3,   4,   3,   3,   8,
         2,   1,   6,   0,   0,   5,   3,  31,   0,   2,   0,   0,   0,
         0,   0,   0,   6,   9,   0,   0,   0,   0,   0,   2,  15,   1,
         2,   3,   0,  65,   5,   0,   0,   0,   0,   1,   8,   0,   0,
         0,   2,   4,   5,   9,   0,   0,   0,   0,  21,   0,   6,   0,
         0,   0,   0,  16,   0,   0,   4,   2,  10,   0,   0,   0,   2,
         1,   3,   0,   0,  21,   0,   0,   2,   0,   3,   0,  38,   0,
         0,   0,   1,   3,   0,   1,   0,   0,   0,   0,   5,   0,   0,
         2,   0,   0,   0,   1,   4,   0,   0,   2,   3,   0,   0,   0,
         0,   1,   2,   0,   6,   4,   1,   1,   0,   1,   0,   0,   0,
         0,   0,   0])</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-62846d01-e068-4a46-a926-c2b1e4b023b5' class='xr-section-summary-in' type='checkbox'  ><label for='section-62846d01-e068-4a46-a926-c2b1e4b023b5' class='xr-section-summary' >Indexes: <span>(1)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><ul class='xr-var-list'><li class='xr-var-item'><div class='xr-index-name'><div>count_obs</div></div><div class='xr-index-preview'>PandasIndex</div><div></div><input id='index-46e637b1-fa3b-4c42-b886-c2455e45ae0e' class='xr-index-data-in' type='checkbox'/><label for='index-46e637b1-fa3b-4c42-b886-c2455e45ae0e' title='Show/Hide index repr'><svg class='icon xr-icon-database'><use xlink:href='#icon-database'></use></svg></label><div class='xr-index-data'><pre>PandasIndex(Index([  0,   1,   2,   3,   4,   5,   6,   7,   8,   9,
       ...
       240, 241, 242, 243, 244, 245, 246, 247, 248, 249],
      dtype=&#x27;int64&#x27;, name=&#x27;count_obs&#x27;, length=250))</pre></div></li></ul></div></li><li class='xr-section-item'><input id='section-775477e3-7374-4fcc-8d00-5d4fe2cee254' class='xr-section-summary-in' type='checkbox'  checked><label for='section-775477e3-7374-4fcc-8d00-5d4fe2cee254' class='xr-section-summary' >Attributes: <span>(6)</span></label><div class='xr-section-inline-details'></div><div class='xr-section-details'><dl class='xr-attrs'><dt><span>created_at :</span></dt><dd>2023-12-09T06:01:42.043721</dd><dt><span>arviz_version :</span></dt><dd>0.16.1</dd><dt><span>inference_library :</span></dt><dd>pymc</dd><dt><span>inference_library_version :</span></dt><dd>5.8.1</dd><dt><span>modeling_interface :</span></dt><dd>bambi</dd><dt><span>modeling_interface_version :</span></dt><dd>0.13.0.dev0</dd></dl></div></li></ul></div></div><br></div>
                      </ul>
                  </div>
            </li>

              </ul>
            </div>
            <style> /* CSS stylesheet for displaying InferenceData objects in jupyterlab.
 *
 */

:root {
  --xr-font-color0: var(--jp-content-font-color0, rgba(0, 0, 0, 1));
  --xr-font-color2: var(--jp-content-font-color2, rgba(0, 0, 0, 0.54));
  --xr-font-color3: var(--jp-content-font-color3, rgba(0, 0, 0, 0.38));
  --xr-border-color: var(--jp-border-color2, #e0e0e0);
  --xr-disabled-color: var(--jp-layout-color3, #bdbdbd);
  --xr-background-color: var(--jp-layout-color0, white);
  --xr-background-color-row-even: var(--jp-layout-color1, white);
  --xr-background-color-row-odd: var(--jp-layout-color2, #eeeeee);
}

html[theme=dark],
body.vscode-dark {
  --xr-font-color0: rgba(255, 255, 255, 1);
  --xr-font-color2: rgba(255, 255, 255, 0.54);
  --xr-font-color3: rgba(255, 255, 255, 0.38);
  --xr-border-color: #1F1F1F;
  --xr-disabled-color: #515151;
  --xr-background-color: #111111;
  --xr-background-color-row-even: #111111;
  --xr-background-color-row-odd: #313131;
}

.xr-wrap {
  display: block;
  min-width: 300px;
  max-width: 700px;
}

.xr-text-repr-fallback {
  /* fallback to plain text repr when CSS is not injected (untrusted notebook) */
  display: none;
}

.xr-header {
  padding-top: 6px;
  padding-bottom: 6px;
  margin-bottom: 4px;
  border-bottom: solid 1px var(--xr-border-color);
}

.xr-header > div,
.xr-header > ul {
  display: inline;
  margin-top: 0;
  margin-bottom: 0;
}

.xr-obj-type,
.xr-array-name {
  margin-left: 2px;
  margin-right: 10px;
}

.xr-obj-type {
  color: var(--xr-font-color2);
}

.xr-sections {
  padding-left: 0 !important;
  display: grid;
  grid-template-columns: 150px auto auto 1fr 20px 20px;
}

.xr-sections.group-sections {
  grid-template-columns: auto;
}

.xr-section-item {
  display: contents;
}

.xr-section-item input {
  display: none;
}

.xr-section-item input + label {
  color: var(--xr-disabled-color);
}

.xr-section-item input:enabled + label {
  cursor: pointer;
  color: var(--xr-font-color2);
}

.xr-section-item input:enabled + label:hover {
  color: var(--xr-font-color0);
}

.xr-section-summary {
  grid-column: 1;
  color: var(--xr-font-color2);
  font-weight: 500;
}

.xr-section-summary > span {
  display: inline-block;
  padding-left: 0.5em;
}

.xr-section-summary-in:disabled + label {
  color: var(--xr-font-color2);
}

.xr-section-summary-in + label:before {
  display: inline-block;
  content: '►';
  font-size: 11px;
  width: 15px;
  text-align: center;
}

.xr-section-summary-in:disabled + label:before {
  color: var(--xr-disabled-color);
}

.xr-section-summary-in:checked + label:before {
  content: '▼';
}

.xr-section-summary-in:checked + label > span {
  display: none;
}

.xr-section-summary,
.xr-section-inline-details {
  padding-top: 4px;
  padding-bottom: 4px;
}

.xr-section-inline-details {
  grid-column: 2 / -1;
}

.xr-section-details {
  display: none;
  grid-column: 1 / -1;
  margin-bottom: 5px;
}

.xr-section-summary-in:checked ~ .xr-section-details {
  display: contents;
}

.xr-array-wrap {
  grid-column: 1 / -1;
  display: grid;
  grid-template-columns: 20px auto;
}

.xr-array-wrap > label {
  grid-column: 1;
  vertical-align: top;
}

.xr-preview {
  color: var(--xr-font-color3);
}

.xr-array-preview,
.xr-array-data {
  padding: 0 5px !important;
  grid-column: 2;
}

.xr-array-data,
.xr-array-in:checked ~ .xr-array-preview {
  display: none;
}

.xr-array-in:checked ~ .xr-array-data,
.xr-array-preview {
  display: inline-block;
}

.xr-dim-list {
  display: inline-block !important;
  list-style: none;
  padding: 0 !important;
  margin: 0;
}

.xr-dim-list li {
  display: inline-block;
  padding: 0;
  margin: 0;
}

.xr-dim-list:before {
  content: '(';
}

.xr-dim-list:after {
  content: ')';
}

.xr-dim-list li:not(:last-child):after {
  content: ',';
  padding-right: 5px;
}

.xr-has-index {
  font-weight: bold;
}

.xr-var-list,
.xr-var-item {
  display: contents;
}

.xr-var-item > div,
.xr-var-item label,
.xr-var-item > .xr-var-name span {
  background-color: var(--xr-background-color-row-even);
  margin-bottom: 0;
}

.xr-var-item > .xr-var-name:hover span {
  padding-right: 5px;
}

.xr-var-list > li:nth-child(odd) > div,
.xr-var-list > li:nth-child(odd) > label,
.xr-var-list > li:nth-child(odd) > .xr-var-name span {
  background-color: var(--xr-background-color-row-odd);
}

.xr-var-name {
  grid-column: 1;
}

.xr-var-dims {
  grid-column: 2;
}

.xr-var-dtype {
  grid-column: 3;
  text-align: right;
  color: var(--xr-font-color2);
}

.xr-var-preview {
  grid-column: 4;
}

.xr-var-name,
.xr-var-dims,
.xr-var-dtype,
.xr-preview,
.xr-attrs dt {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  padding-right: 10px;
}

.xr-var-name:hover,
.xr-var-dims:hover,
.xr-var-dtype:hover,
.xr-attrs dt:hover {
  overflow: visible;
  width: auto;
  z-index: 1;
}

.xr-var-attrs,
.xr-var-data {
  display: none;
  background-color: var(--xr-background-color) !important;
  padding-bottom: 5px !important;
}

.xr-var-attrs-in:checked ~ .xr-var-attrs,
.xr-var-data-in:checked ~ .xr-var-data {
  display: block;
}

.xr-var-data > table {
  float: right;
}

.xr-var-name span,
.xr-var-data,
.xr-attrs {
  padding-left: 25px !important;
}

.xr-attrs,
.xr-var-attrs,
.xr-var-data {
  grid-column: 1 / -1;
}

dl.xr-attrs {
  padding: 0;
  margin: 0;
  display: grid;
  grid-template-columns: 125px auto;
}

.xr-attrs dt, dd {
  padding: 0;
  margin: 0;
  float: left;
  padding-right: 10px;
  width: auto;
}

.xr-attrs dt {
  font-weight: normal;
  grid-column: 1;
}

.xr-attrs dt:hover span {
  display: inline-block;
  background: var(--xr-background-color);
  padding-right: 10px;
}

.xr-attrs dd {
  grid-column: 2;
  white-space: pre-wrap;
  word-break: break-all;
}

.xr-icon-database,
.xr-icon-file-text2 {
  display: inline-block;
  vertical-align: middle;
  width: 1em;
  height: 1.5em !important;
  stroke-width: 0;
  stroke: currentColor;
  fill: currentColor;
}
.xr-wrap{width:700px!important;} </style>




```python
draw_1 = select_draws(idata_grid, grid, {"livebait": 0}, "count_mean")
```


```python
draw_1 = select_draws(idata_grid, grid, {"livebait": 0}, "count_mean")
draw_2 = select_draws(idata_grid, grid, {"livebait": 1}, "count_mean")

comparison_mean = (draw_2 - draw_1).mean(("chain", "draw"))
comparison_hdi = az.hdi(draw_2 - draw_1)

comparison_df = pd.DataFrame(
    {
        "mean": comparison_mean.values,
        "hdi_low": comparison_hdi.sel(hdi="lower")["count_mean"].values,
        "hdi_high": comparison_hdi.sel(hdi="higher")["count_mean"].values,
    }
)
comparison_df.head(10)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>mean</th>
      <th>hdi_low</th>
      <th>hdi_high</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0.214363</td>
      <td>0.144309</td>
      <td>0.287735</td>
    </tr>
    <tr>
      <th>1</th>
      <td>0.053678</td>
      <td>0.029533</td>
      <td>0.077615</td>
    </tr>
    <tr>
      <th>2</th>
      <td>0.013558</td>
      <td>0.006332</td>
      <td>0.021971</td>
    </tr>
    <tr>
      <th>3</th>
      <td>0.003454</td>
      <td>0.001132</td>
      <td>0.006040</td>
    </tr>
    <tr>
      <th>4</th>
      <td>0.512709</td>
      <td>0.369741</td>
      <td>0.661034</td>
    </tr>
    <tr>
      <th>5</th>
      <td>0.128316</td>
      <td>0.077068</td>
      <td>0.181741</td>
    </tr>
    <tr>
      <th>6</th>
      <td>0.032392</td>
      <td>0.015553</td>
      <td>0.050690</td>
    </tr>
    <tr>
      <th>7</th>
      <td>0.008247</td>
      <td>0.003047</td>
      <td>0.014382</td>
    </tr>
    <tr>
      <th>8</th>
      <td>1.228708</td>
      <td>0.913514</td>
      <td>1.533121</td>
    </tr>
    <tr>
      <th>9</th>
      <td>0.307342</td>
      <td>0.192380</td>
      <td>0.426808</td>
    </tr>
  </tbody>
</table>
</div>



We can compare this comparison with `bmb.interpret.comparisons`.


```python
summary_df = bmb.interpret.comparisons(
    fish_model,
    fish_idata,
    contrast={"livebait": [0, 1]},
    conditional=conditional
)
summary_df.head(10)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>term</th>
      <th>estimate_type</th>
      <th>value</th>
      <th>camper</th>
      <th>persons</th>
      <th>child</th>
      <th>estimate</th>
      <th>lower_3.0%</th>
      <th>upper_97.0%</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>livebait</td>
      <td>diff</td>
      <td>(0, 1)</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>0.214363</td>
      <td>0.144309</td>
      <td>0.287735</td>
    </tr>
    <tr>
      <th>1</th>
      <td>livebait</td>
      <td>diff</td>
      <td>(0, 1)</td>
      <td>0</td>
      <td>1</td>
      <td>2</td>
      <td>0.053678</td>
      <td>0.029533</td>
      <td>0.077615</td>
    </tr>
    <tr>
      <th>2</th>
      <td>livebait</td>
      <td>diff</td>
      <td>(0, 1)</td>
      <td>0</td>
      <td>1</td>
      <td>3</td>
      <td>0.013558</td>
      <td>0.006332</td>
      <td>0.021971</td>
    </tr>
    <tr>
      <th>3</th>
      <td>livebait</td>
      <td>diff</td>
      <td>(0, 1)</td>
      <td>0</td>
      <td>1</td>
      <td>4</td>
      <td>0.003454</td>
      <td>0.001132</td>
      <td>0.006040</td>
    </tr>
    <tr>
      <th>4</th>
      <td>livebait</td>
      <td>diff</td>
      <td>(0, 1)</td>
      <td>0</td>
      <td>2</td>
      <td>1</td>
      <td>0.512709</td>
      <td>0.369741</td>
      <td>0.661034</td>
    </tr>
    <tr>
      <th>5</th>
      <td>livebait</td>
      <td>diff</td>
      <td>(0, 1)</td>
      <td>0</td>
      <td>2</td>
      <td>2</td>
      <td>0.128316</td>
      <td>0.077068</td>
      <td>0.181741</td>
    </tr>
    <tr>
      <th>6</th>
      <td>livebait</td>
      <td>diff</td>
      <td>(0, 1)</td>
      <td>0</td>
      <td>2</td>
      <td>3</td>
      <td>0.032392</td>
      <td>0.015553</td>
      <td>0.050690</td>
    </tr>
    <tr>
      <th>7</th>
      <td>livebait</td>
      <td>diff</td>
      <td>(0, 1)</td>
      <td>0</td>
      <td>2</td>
      <td>4</td>
      <td>0.008247</td>
      <td>0.003047</td>
      <td>0.014382</td>
    </tr>
    <tr>
      <th>8</th>
      <td>livebait</td>
      <td>diff</td>
      <td>(0, 1)</td>
      <td>0</td>
      <td>3</td>
      <td>1</td>
      <td>1.228708</td>
      <td>0.913514</td>
      <td>1.533121</td>
    </tr>
    <tr>
      <th>9</th>
      <td>livebait</td>
      <td>diff</td>
      <td>(0, 1)</td>
      <td>0</td>
      <td>3</td>
      <td>2</td>
      <td>0.307342</td>
      <td>0.192380</td>
      <td>0.426808</td>
    </tr>
  </tbody>
</table>
</div>



Albeit the other information in the `summary_df`, the columns `estimate`, `lower_3.0%`, `upper_97.0%` are identical.

### Cross comparisons

Computing a cross-comparison is useful for when we want to compare contrasts when two (or more) predictors change at the same time. Cross-comparisons are currently not supported in the `comparisons` function, but we can use `select_draws` to compute them. For example, imagine we are interested in computing the cross-comparison between the two rows below.


```python
summary_df.iloc[:2]
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>term</th>
      <th>estimate_type</th>
      <th>value</th>
      <th>camper</th>
      <th>persons</th>
      <th>child</th>
      <th>estimate</th>
      <th>lower_3.0%</th>
      <th>upper_97.0%</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>livebait</td>
      <td>diff</td>
      <td>(0, 1)</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>0.214363</td>
      <td>0.144309</td>
      <td>0.287735</td>
    </tr>
    <tr>
      <th>1</th>
      <td>livebait</td>
      <td>diff</td>
      <td>(0, 1)</td>
      <td>0</td>
      <td>1</td>
      <td>2</td>
      <td>0.053678</td>
      <td>0.029533</td>
      <td>0.077615</td>
    </tr>
  </tbody>
</table>
</div>



The cross-comparison amounts to first computing the comparison for row 0, given below, and can be verified by looking at the estimate in `summary_df`.


```python
cond_10 = {
    "camper": 0,
    "persons": 1,
    "child": 1,
    "livebait": 0 
}

cond_11 = {
    "camper": 0,
    "persons": 1,
    "child": 1,
    "livebait": 1
}

draws_10 = select_draws(idata_grid, grid, cond_10, "count_mean")
draws_11 = select_draws(idata_grid, grid, cond_11, "count_mean")

(draws_11 - draws_10).mean(("chain", "draw")).item()
```




    0.2143627093182434



Next, we need to compute the comparison for row 1.


```python
cond_20 = {
    "camper": 0,
    "persons": 1,
    "child": 2,
    "livebait": 0
}

cond_21 = {
    "camper": 0,
    "persons": 1,
    "child": 2,
    "livebait": 1
}

draws_20 = select_draws(idata_grid, grid, cond_20, "count_mean")
draws_21 = select_draws(idata_grid, grid, cond_21, "count_mean")
```


```python
(draws_21 - draws_20).mean(("chain", "draw")).item()
```




    0.053678256991883604



Next, we compute the "first level" comparisons (`diff_1` and `diff_2`). Subsequently, we compute the difference between these two differences to obtain the cross-comparison.


```python
diff_1 = (draws_11 - draws_10)
diff_2 = (draws_21 - draws_20)

cross_comparison = (diff_2 - diff_1).mean(("chain", "draw")).item()
cross_comparison
```




    -0.16068445232635978



To verify this is correct, we can check by performing the cross-comparison directly on the `summary_df`.


```python
summary_df.iloc[1]["estimate"] - summary_df.iloc[0]["estimate"]
```




    -0.16068445232635978



## Summary

In this notebook, the interpret helper functions `data_grid` and `select_draws` were introduced and it was demonstrated how they can be used to compute pairwise grids of data and cross-comparisons. With these functions, it is left to the user to generate their grids of data and quantities of interest allowing for more flexibility and control over the type of data passed to `model.predict` and the quantities of interest computed.


```python
%load_ext watermark
%watermark -n -u -v -iv -w
```

    Last updated: Tue Dec 05 2023
    
    Python implementation: CPython
    Python version       : 3.11.0
    IPython version      : 8.13.2
    
    numpy : 1.24.2
    pandas: 2.1.0
    bambi : 0.13.0.dev0
    arviz : 0.16.1
    
    Watermark: 2.3.1
    

