HW6
================
Stephen Powers
11/21/2019

## Problem 1

#### Loading and tidying data

``` r
birthweight = 
  read_csv("./data/birthweight.csv") %>% 
  janitor::clean_names() %>% 
  drop_na() %>% 
  mutate(
    babysex = factor(babysex, 
                     levels = c(1, 2), 
                     labels = c("male", "female")),
    frace = factor(frace, 
                   levels = c(1, 2, 3, 4, 8, 9), 
                   labels = c("white", "black", "asian", "puerto rican", "other", "unknown")),
    mrace = factor(mrace, 
                   levels = c(1, 2, 3, 4, 8, 9), 
                   labels = c("white", "black", "asian", "puerto rican", "other", "unknown")),
    malform = factor(malform, 
                     levels = c(0, 1), 
                     labels = c("absent", "present")))
```

    ## Parsed with column specification:
    ## cols(
    ##   .default = col_double()
    ## )

    ## See spec(...) for full column specifications.

#### Propose a regression model for birthweight

``` r
birthweight_base_model = 
  lm(bwt ~ gaweeks + malform + blength + bhead + babysex + wtgain + smoken, data = birthweight)
```

The model above was based on on a hypothesized structure for the factors
that underly birthweight. It was not based on a data-driven
model-building process.

#### Creating residual plot

``` r
bw_plot = modelr::add_residuals(birthweight, birthweight_base_model) 
bw_plot = modelr::add_predictions(bw_plot, birthweight_base_model)

bw_plot %>% 
  ggplot(aes(x = pred, y = resid)) + 
  geom_point() +
  theme_minimal()
```

![](HW6_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

#### Comparing the model above to two others

``` r
bw_model1 =
  lm(bwt ~ blength + gaweeks, data = birthweight)
  
bw_model2 = 
  lm(bwt ~ bhead + blength + babysex + bhead*blength + bhead*babysex + blength*babysex + bhead*blength*babysex, data = birthweight)
```

#### Comparison in terms of the cross-validated prediction error

``` r
cv_df = 
  crossv_mc(birthweight, 100)

cv_df = 
  cv_df %>% 
    mutate(base_model = 
             map(train, ~lm(bwt ~ gaweeks + malform + blength + bhead + babysex + wtgain + smoken, data = .x)), 
           model1 = 
             map(train, ~lm(bwt ~ blength + gaweeks, data = .x)), 
           model2 = 
             map(train, ~lm(bwt ~ bhead + blength + babysex + bhead*blength + bhead*babysex + blength*babysex + bhead*blength*babysex, data = .x))) %>% 
    mutate(rmse_base = 
              map2_dbl(base_model, test, ~rmse(model = .x, data = .y)),
          rmse_model1 = 
              map2_dbl(model1, test, ~rmse(model = .x, data = .y)),
          rmse_model2 = 
              map2_dbl(model2, test, ~rmse(model = .x, data = .y)))

cv_df %>% 
  select(starts_with("rmse")) %>% 
  pivot_longer(
    everything(),
    names_to = "model", 
    values_to = "rmse",
    names_prefix = "rmse_") %>% 
  mutate(model = fct_inorder(model)) %>% 
  ggplot(aes(x = model, y = rmse)) + 
    geom_violin() +
    theme_minimal()
```

![](HW6_files/figure-gfm/unnamed-chunk-5-1.png)<!-- -->

## Problem 2

#### Loading in data

``` r
weather_df = 
  rnoaa::meteo_pull_monitors(
    c("USW00094728"),
    var = c("PRCP", "TMIN", "TMAX"), 
    date_min = "2017-01-01",
    date_max = "2017-12-31") %>%
  mutate(
    name = recode(id, USW00094728 = "CentralPark_NY"),
    tmin = tmin / 10,
    tmax = tmax / 10) %>%
  select(name, id, everything())
```

    ## Registered S3 method overwritten by 'crul':
    ##   method                 from
    ##   as.character.form_file httr

    ## Registered S3 method overwritten by 'hoardr':
    ##   method           from
    ##   print.cache_info httr

    ## file path:          /Users/StephenPowers/Library/Caches/rnoaa/ghcnd/USW00094728.dly

    ## file last updated:  2019-09-03 11:49:56

    ## file min/max dates: 1869-01-01 / 2019-09-30

#### Creating bootstrap function and sample

``` r
set.seed(1)

boot_sample = function(df) {
  sample_frac(df, replace = TRUE)
}

boot_straps = 
  data_frame(
    strap_number = 1:5000,
    strap_sample = rerun(5000, boot_sample(weather_df))
  )
```

    ## Warning: `data_frame()` is deprecated, use `tibble()`.
    ## This warning is displayed once per session.

#### Estimated log(β0∗β1)

``` r
bootstrap_results = 
  boot_straps %>% 
    mutate(
      model = map(strap_sample, ~lm(tmax ~ tmin, data = .x)),
      results = map(model, broom::tidy)) %>% 
    unnest(cols = c(results)) %>%
    select(strap_number, model:estimate) %>% 
    pivot_wider(names_from = term, values_from = estimate) %>%
    janitor::clean_names() %>%
    rename(b0 = intercept, b1 = tmin) %>%
    mutate(log_b0b1 = log(b0*b1)) %>%
    select(-b0,-b1)
```

#### Estimated r^2

``` r
bootstrap_results =
  bootstrap_results %>%
  mutate(results2 = map(model, broom::glance)) %>%
  unnest(cols = c(results2)) %>%
  select(strap_number:r.squared) %>%
  rename(r2 = r.squared)
```

#### Creating plots

``` r
log_plot =
  bootstrap_results %>%
    ggplot(aes(x = log_b0b1)) +
    geom_histogram() +
    theme_minimal()

r2_plot = 
  bootstrap_results %>%
    ggplot(aes(x = r2)) + 
    geom_histogram() +
    theme_minimal()

log_plot + r2_plot
```

    ## `stat_bin()` using `bins = 30`. Pick better value with `binwidth`.
    ## `stat_bin()` using `bins = 30`. Pick better value with `binwidth`.

![](HW6_files/figure-gfm/unnamed-chunk-10-1.png)<!-- -->

The distributions of both of the plots above appears to be normal. The
median of the log(β0∗β1) value is 2.01, and the mean is 2.01. The median
of the r^2 value is 0.91, and the mean is 0.91.

The 95% confidence interval of log(β0∗β1) is (1.96, 2.06). The 95%
confidence interval of r^2 is (0.89, 0.93).
