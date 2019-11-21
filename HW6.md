HW6
================
Stephen Powers
11/21/2019

## Problem 1

#### Loading data

``` r
birthweight = 
  read_csv("./data/birthweight.csv") %>% 
  janitor::clean_names()
```

    ## Parsed with column specification:
    ## cols(
    ##   .default = col_double()
    ## )

    ## See spec(...) for full column specifications.

## Problem 2
