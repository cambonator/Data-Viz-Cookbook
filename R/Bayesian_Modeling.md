Bayesian Modeling
================
Jesse Cambon
25 April, 2020

References: \* <http://appliedpredictivemodeling.com/data> \*
<http://faculty.marshall.usc.edu/gareth-james/ISL/data.html>

Todo: \* HDI \* Sigma Term \* References

## Setup

``` r
#library(AppliedPredictiveModeling) # datasets
library(ISLR) # datasets
library(skimr)
library(tidyverse)
library(wesanderson)
library(rstanarm)
library(bayestestR)
library(insight)
library(bayesplot)
library(broom)
library(rsample)
library(knitr)
library(jcolors)
library(patchwork)
library(ggrepel)

num_cores <-  parallel::detectCores()
options(mc.cores = num_cores)

set.seed(42) # for reproducibility
```

## Set input data and formula

Datasets and formulas: \* ISLR::Carseats : Sales \~ Advertising + Price
\* ISLR::Credit : Limit \~ Income + Rating

``` r
### Set input dataset here ################
split <- initial_split(Credit, prop = 1/2)
############################################

### Set model equation here ##########################
model_formula = as.formula(Limit ~ Income + Rating + Age)
######################################################
```

``` r
library(DataExplorer)
plot_histogram(Credit)
```

![](../rmd_images/Bayesian_Modeling/unnamed-chunk-3-1.png)<!-- -->

C/V split

``` r
train <- training(split) %>% as_tibble()
test  <- testing(split) %>% as_tibble()

train_small <- train %>% sample_n(50)
train_tiny <- train %>% sample_n(10)
```

Fit models

``` r
lm_model <- glm(model_formula, data = train)
stan_model <- stan_glm(model_formula, data = train)
stan_model_small <- stan_glm(model_formula, data = train_small)
stan_model_tiny <- stan_glm(model_formula, data = train_tiny)
```

``` r
tidy(lm_model) %>% kable()
```

| term        |      estimate |  std.error |    statistic |   p.value |
| :---------- | ------------: | ---------: | -----------: | --------: |
| (Intercept) | \-525.0002320 | 50.8812765 | \-10.3181419 | 0.0000000 |
| Income      |     0.5665139 |  0.6271243 |    0.9033519 | 0.3674481 |
| Rating      |    14.7966546 |  0.1345476 |  109.9733565 | 0.0000000 |
| Age         |   \-0.1527329 |  0.7346700 |  \-0.2078932 | 0.8355282 |

``` r
tidy(stan_model) %>% kable()
```

| term        |      estimate |  std.error |
| :---------- | ------------: | ---------: |
| (Intercept) | \-524.6240780 | 53.8400159 |
| Income      |     0.5746042 |  0.6161314 |
| Rating      |    14.7958603 |  0.1380327 |
| Age         |   \-0.1343952 |  0.7565537 |

``` r
prior_summary(stan_model)
```

    ## Priors for model 'stan_model' 
    ## ------
    ## Intercept (after predictors centered)
    ##   Specified prior:
    ##     ~ normal(location = 0, scale = 10)
    ##   Adjusted prior:
    ##     ~ normal(location = 0, scale = 22646)
    ## 
    ## Coefficients
    ##   Specified prior:
    ##     ~ normal(location = [0,0,0], scale = [2.5,2.5,2.5])
    ##   Adjusted prior:
    ##     ~ normal(location = [0,0,0], scale = [173.52, 37.34,318.82])
    ## 
    ## Auxiliary (sigma)
    ##   Specified prior:
    ##     ~ exponential(rate = 1)
    ##   Adjusted prior:
    ##     ~ exponential(rate = 0.00044)
    ## ------
    ## See help('prior_summary.stanreg') for more details

``` r
prior_summary(stan_model_small)
```

    ## Priors for model 'stan_model_small' 
    ## ------
    ## Intercept (after predictors centered)
    ##   Specified prior:
    ##     ~ normal(location = 0, scale = 10)
    ##   Adjusted prior:
    ##     ~ normal(location = 0, scale = 20644)
    ## 
    ## Coefficients
    ##   Specified prior:
    ##     ~ normal(location = [0,0,0], scale = [2.5,2.5,2.5])
    ##   Adjusted prior:
    ##     ~ normal(location = [0,0,0], scale = [201.26, 37.83,307.24])
    ## 
    ## Auxiliary (sigma)
    ##   Specified prior:
    ##     ~ exponential(rate = 1)
    ##   Adjusted prior:
    ##     ~ exponential(rate = 0.00048)
    ## ------
    ## See help('prior_summary.stanreg') for more details

``` r
prior_summary(stan_model_tiny)
```

    ## Priors for model 'stan_model_tiny' 
    ## ------
    ## Intercept (after predictors centered)
    ##   Specified prior:
    ##     ~ normal(location = 0, scale = 10)
    ##   Adjusted prior:
    ##     ~ normal(location = 0, scale = 22951)
    ## 
    ## Coefficients
    ##   Specified prior:
    ##     ~ normal(location = [0,0,0], scale = [2.5,2.5,2.5])
    ##   Adjusted prior:
    ##     ~ normal(location = [0,0,0], scale = [213.55, 36.76,315.40])
    ## 
    ## Auxiliary (sigma)
    ##   Specified prior:
    ##     ~ exponential(rate = 1)
    ##   Adjusted prior:
    ##     ~ exponential(rate = 0.00044)
    ## ------
    ## See help('prior_summary.stanreg') for more details

Draw from the prior and posterior distributions

``` r
# Function for simulating prior and posterior distributions from stan model
sim_post_prior <- function(model) {
  # Simulate prior with bayestestR package
  prior <- simulate_prior(model) %>%
  pivot_longer(everything(),names_to='Parameter')

  # Simulate Posterior with insight package
  posterior <- get_parameters(model,iterations=10000) %>% 
  pivot_longer(everything(),names_to='Parameter')

  # Combine into one dataset
  combined <- prior %>% mutate(Distribution='Prior') %>% 
  bind_rows(posterior %>% mutate(Distribution='Posterior'))
  
  return(combined)
}

prior_posterior <- sim_post_prior(stan_model)
prior_posterior_small <- sim_post_prior(stan_model_small)
prior_posterior_tiny <- sim_post_prior(stan_model_tiny)
```

Plot our parameter prior and posterior distributions

``` r
# Find the x,y coordinates for peak density in a sample
find_peak_density <- function(x_sample) {
  density_x <- density(x_sample)
  # Find coordinates for peak density
  x_max <- density_x$x[which.max(density_x$y)]
  y_max <- max(density_x$y)
  
  return(tibble(x=x_max,y=y_max))
}

# Function for plotting 
plot_parameters <- function(distribution_sample,train_data,plot_peaks=FALSE) {
    
  # data to plot - exclude intercept term
  plot_data <- distribution_sample %>% filter(!str_detect(Parameter,'Intercept'))
    
  # Points for labeling max density 
  # based loosely on: https://stackoverflow.com/questions/56520287/how-to-add-label-to-each-geom-density-line)
 density_coordinates <- plot_data %>% 
  group_by(Distribution,Parameter) %>%
  do(find_peak_density(.$value))
    
  base_plot <- ggplot(data=plot_data,
         aes(x=value,fill=Parameter)) +
    facet_wrap(~fct_rev(Distribution),scales='free') +
    theme_minimal() +
    scale_y_continuous(expand =c(0,0,0.15,0)) + # add spacing for labels
    geom_vline(xintercept=0,color='red',size=0.25,linetype='dashed') +
    theme(legend.position='top',
          legend.title=element_blank(),
          plot.title = element_text(hjust = 0.5)) +
    geom_density(alpha=0.4,size=0.05) + ggtitle(str_c('n = ',as.character(nrow(train_data)))) +
    xlab('') + ylab('') + scale_fill_jcolors('pal6') + 
    guides(color = guide_legend(reverse=T))
  
  if (plot_peaks == TRUE) {
    return(base_plot +
    geom_point(data=density_coordinates, aes(x=x, y=y),show.legend = F) +
    geom_text_repel(data=density_coordinates, aes(label=round(x,2),x=x, y=y),
                     force=1.5,size=4,show.legend = F))
  }
  else {
    return(base_plot)
  }
}
```

Compare parameter distributions by sample size of training dataset

``` r
plot_parameters(prior_posterior,train,plot_peaks=T) 
```

![](../rmd_images/Bayesian_Modeling/unnamed-chunk-12-1.png)<!-- -->

``` r
plot_parameters(prior_posterior_small,train_small,plot_peaks=T) 
```

![](../rmd_images/Bayesian_Modeling/unnamed-chunk-12-2.png)<!-- -->

``` r
plot_parameters(prior_posterior_tiny,train_tiny,plot_peaks=T)
```

![](../rmd_images/Bayesian_Modeling/unnamed-chunk-12-3.png)<!-- -->

Posterior Prediction Check

``` r
# Function that adds size of training dataset to pp_check
pp_check_info <- function(model) {
  pp_check(model) + ggtitle(str_c('n = ',as.character(nrow(model$data)))) +
      theme_minimal() +
      theme(plot.title = element_text(hjust = 0.5))
}

pp_check_info(stan_model)
```

![](../rmd_images/Bayesian_Modeling/unnamed-chunk-13-1.png)<!-- -->

``` r
pp_check_info(stan_model_small)
```

![](../rmd_images/Bayesian_Modeling/unnamed-chunk-13-2.png)<!-- -->

``` r
pp_check_info(stan_model_tiny)
```

![](../rmd_images/Bayesian_Modeling/unnamed-chunk-13-3.png)<!-- -->

Manually plot the outcome distribution to compare to the posterior check
plot above

``` r
# Extract variables from formula
all_model_vars <- all.vars(model_formula)
outcome_var <- sym(all_model_vars[1])
predictors <- all_model_vars[-1]

ggplot(aes(x=!!outcome_var),data=train) + geom_density() + theme_minimal()
```

![](../rmd_images/Bayesian_Modeling/unnamed-chunk-14-1.png)<!-- -->

Make predictions using the posterior distribution

``` r
post_pred <- posterior_predict(stan_model,new_data = test,draws = 1000) %>%
  as_tibble()
```

``` r
# Function that adds size of training dataset to mcmc_areas
mcmc_areas_info <- function(model,variables) {
  mcmc_areas(model,pars=variables) + ggtitle(str_c('n = ',as.character(nrow(model$data)))) +
      theme_minimal() +
      theme(plot.title = element_text(hjust = 0.5))
}

mcmc_areas_info(stan_model,predictors)
```

    ## Warning: `expand_scale()` is deprecated; use `expansion()` instead.

![](../rmd_images/Bayesian_Modeling/unnamed-chunk-16-1.png)<!-- -->

``` r
mcmc_areas_info(stan_model_small,predictors)
```

    ## Warning: `expand_scale()` is deprecated; use `expansion()` instead.

![](../rmd_images/Bayesian_Modeling/unnamed-chunk-16-2.png)<!-- -->

``` r
mcmc_areas_info(stan_model_tiny,predictors) 
```

    ## Warning: `expand_scale()` is deprecated; use `expansion()` instead.

![](../rmd_images/Bayesian_Modeling/unnamed-chunk-16-3.png)<!-- -->

``` r
#mcmc_intervals(stan_model,pars=predictors) + theme_bw()
#posterior_vs_prior(stan_model)
```

Look at the posterior prediction distribution for a single observation

``` r
row_num <- quo(`25`)
ggplot(aes(x=!!row_num),data=post_pred) + geom_density() + theme_minimal()

# Take a look at that same row number
print(test %>% select(Sales, Advertising, Price) %>% slice(as.numeric(as_label(row_num))))
```