Parallelizing *tidyverse* workflows
================

[multidplyr](https://github.com/hadley/multidplyr) allows to split the analisyis of evenly-sized subsamples of large datasets (~10m obs) by dispatching across multiple cores. See the *multidplyr* [vignette](https://github.com/hadley/multidplyr/blob/master/vignettes/multidplyr.md) for details and examples.

Installation
------------

*multidplyr* is a library under development: installation requires the [devtools](https://cran.r-project.org/web/packages/devtools/index.html) library to compile from source – and [rtools](https://cran.r-project.org/bin/windows/Rtools/) on Windows.

``` r
devtools::install_github("hadley/multidplyr")
```

Example: country-level regression
---------------------------------

From cross-country firm-level data, we want to fit country-wise regression models of average wages on firm controls.

``` r
library(tidyverse)
library(multidplyr)
```

Create dummy data:

``` r
countries <- c("Italy", "France", "Germany", "Spain", "UK") %>% as.factor()

# Number of observations for each country
n <- 1000000 

firm_data <- tibble(
  country = rep(countries, each = n),
  profits = runif(n = n*length(countries), min = 15, max = 50),
  unionized = sample(c(TRUE, FALSE), size = n*length(countries), replace = TRUE),
  share_tertiary = runif(n*length(countries)),
  employees = runif(n = n*length(countries), min = 1, max = 300) %>% round()
)

firm_data
```

    ## # A tibble: 5,000,000 x 5
    ##    country profits unionized share_tertiary employees
    ##    <fct>     <dbl> <lgl>              <dbl>     <dbl>
    ##  1 Italy      20.1 FALSE             0.193         31
    ##  2 Italy      47.3 TRUE              0.206        155
    ##  3 Italy      40.0 TRUE              0.528        126
    ##  4 Italy      48.7 FALSE             0.0860        69
    ##  5 Italy      43.8 TRUE              0.402        192
    ##  6 Italy      34.9 FALSE             0.367         93
    ##  7 Italy      18.3 TRUE              0.212        271
    ##  8 Italy      18.8 TRUE              0.319        278
    ##  9 Italy      34.5 TRUE              0.0215         6
    ## 10 Italy      25.8 TRUE              0.307         37
    ## # ... with 4,999,990 more rows

Initialize a cluster and split the data by country into “shards” using `partition()`. Similar to `group_by()`, but will send each group to a different cluster.

``` r
firm_data_part <- partition(firm_data, country)
```

    ## Initialising 3 core cluster.

    ## Warning: group_indices_.grouped_df ignores extra arguments

Then, fit a model by groups using a `do()` expression:

``` r
firm_data_part %>% 
  do(model_wage = lm(profits ~ employees + unionized +employees + share_tertiary,  data = .))
```

    ## Source: party_df [5 x 2]
    ## Groups: country
    ## Shards: 3 [1--2 rows]
    ## 
    ## # S3: party_df
    ##   country model_wage
    ##   <fct>   <list>    
    ## 1 Italy   <S3: lm>  
    ## 2 Germany <S3: lm>  
    ## 3 UK      <S3: lm>  
    ## 4 France  <S3: lm>  
    ## 5 Spain   <S3: lm>
