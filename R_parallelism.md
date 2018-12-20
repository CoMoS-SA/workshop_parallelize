Parallelism in R – Tutorials for methods for complexity modelling
================

Parallelizing *tidyverse* workflows
-----------------------------------

[multidplyr](https://github.com/hadley/multidplyr) allows to split the analisyis of evenly-sized subsamples of large datasets (~10m obs) by dispatching across multiple cores. See the *multidplyr* [vignette](https://github.com/hadley/multidplyr/blob/master/vignettes/multidplyr.md) for details and examples.

*multidplyr* is a library under development: installation requires the [devtools](https://cran.r-project.org/web/packages/devtools/index.html) library to compile from source – and [rtools](https://cran.r-project.org/bin/windows/Rtools/) on Windows.

### Installation

*multidplyr* is a library under development: installation requires the [devtools](https://cran.r-project.org/web/packages/devtools/index.html) library to compile from source – and [rtools](https://cran.r-project.org/bin/windows/Rtools/) on Windows.

``` r
devtools::install_github("hadley/multidplyr")
```

    ## Skipping install of 'multidplyr' from a github remote, the SHA1 (0085ded4) has not changed since last install.
    ##   Use `force = TRUE` to force installation

### Example: country-level regression

From cross-country firm-level data, we want to fit country-wise regression models of average wages on firm controls.

First, create dummy data:

``` r
library(tidyverse)
```

    ## -- Attaching packages ------------------------------------------------------------------------------------------------------------------------------------- tidyverse 1.2.1 --

    ## v ggplot2 3.1.0     v purrr   0.2.5
    ## v tibble  1.4.2     v dplyr   0.7.8
    ## v tidyr   0.8.2     v stringr 1.3.1
    ## v readr   1.3.0     v forcats 0.3.0

    ## -- Conflicts ---------------------------------------------------------------------------------------------------------------------------------------- tidyverse_conflicts() --
    ## x dplyr::filter() masks stats::filter()
    ## x dplyr::lag()    masks stats::lag()

``` r
library(multidplyr)

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
    ##  1 Italy      47.4 FALSE             0.966        289
    ##  2 Italy      28.1 TRUE              0.201         77
    ##  3 Italy      19.7 TRUE              0.728        251
    ##  4 Italy      34.8 TRUE              0.0282       234
    ##  5 Italy      26.0 TRUE              0.261          8
    ##  6 Italy      23.9 TRUE              0.257         31
    ##  7 Italy      24.1 TRUE              0.149        162
    ##  8 Italy      46.3 TRUE              0.418        201
    ##  9 Italy      48.1 TRUE              0.573        214
    ## 10 Italy      25.4 FALSE             0.707         85
    ## # ... with 4,999,990 more rows

``` r
pryr::object_size(firm_data)
```

    ## 160 MB

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
    ## 1 UK      <S3: lm>  
    ## 2 Germany <S3: lm>  
    ## 3 Italy   <S3: lm>  
    ## 4 France  <S3: lm>  
    ## 5 Spain   <S3: lm>
