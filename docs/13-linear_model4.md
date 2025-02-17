# Linear model 4

## Load packages and set plotting theme  


```r
library("knitr")      # for knitting RMarkdown 
library("kableExtra") # for making nice tables
library("janitor")    # for cleaning column names
library("broom")      # for tidying up linear models 
library("afex")       # for running ANOVAs
library("emmeans")    # for calculating contrasts
library("car")        # for calculating ANOVAs
library("tidyverse")  # for wrangling, plotting, etc.
```


```r
theme_set(
  theme_classic() + #set the theme 
    theme(text = element_text(size = 20)) #set the default text size
)

# these options here change the formatting of how comments are rendered
opts_chunk$set(comment = "",
               fig.show = "hold")

# include references for used packages
write_bib(.packages(), "packages.bib") 
```

## Load data sets

Read in the data:


```r
df.poker = read_csv("data/poker.csv") %>% 
  mutate(skill = factor(skill,
                        levels = 1:2,
                        labels = c("expert", "average")),
         skill = fct_relevel(skill, "average", "expert"),
         hand = factor(hand,
                       levels = 1:3,
                       labels = c("bad", "neutral", "good")),
         limit = factor(limit,
                        levels = 1:2,
                        labels = c("fixed", "none")),
         participant = 1:n()) %>% 
  select(participant, everything())
```

## Linear contrasts 

Here is a linear contrast that assumes that there is a linear relationship between the quality of one's hand, and the final balance.  


```r
df.poker = df.poker %>% 
  mutate(hand_contrast = factor(hand,
                                levels = c("bad", "neutral", "good"),
                                labels = c(-1, 0, 1)),
         hand_contrast = hand_contrast %>% as.character() %>% as.numeric())

fit.contrast = lm(formula = balance ~ hand_contrast,
                  data = df.poker)
```

Here is a visualization of the model prediction together with the residuals. 


```r
df.plot = df.poker %>% 
  mutate(hand_jitter = hand %>% as.numeric(),
         hand_jitter = hand_jitter + runif(n(), min = -0.4, max = 0.4))

df.tidy = fit.contrast %>% 
  tidy() %>% 
  select_if(is.numeric) %>% 
  mutate_all(~ round(., 2))

df.augment = fit.contrast %>% 
  augment() %>%
  clean_names() %>% 
  bind_cols(df.plot %>% select(hand_jitter))

ggplot(data = df.plot,
       mapping = aes(x = hand_jitter,
                       y = balance,
                       color = as.factor(hand_contrast))) + 
  geom_point(alpha = 0.8) +
  geom_segment(data = NULL,
               aes(x = 0.6,
                   xend = 1.4,
                   y = df.tidy$estimate[1]-df.tidy$estimate[2],
                   yend = df.tidy$estimate[1]-df.tidy$estimate[2]),
               color = "red",
               size = 1) +
  geom_segment(data = NULL,
               aes(x = 1.6,
                   xend = 2.4,
                   y = df.tidy$estimate[1],
                   yend = df.tidy$estimate[1]),
               color = "orange",
               size = 1) +
  geom_segment(data = NULL,
               aes(x = 2.6,
                   xend = 3.4,
                   y = df.tidy$estimate[1] + df.tidy$estimate[2],
                   yend = df.tidy$estimate[1] + df.tidy$estimate[2]),
               color = "green",
               size = 1) +
  geom_segment(data = df.augment,
               aes(xend = hand_jitter,
                   y = balance,
                   yend = fitted),
               alpha = 0.3) +
  labs(y = "balance") + 
  scale_color_manual(values = c("red", "orange", "green")) + 
  scale_x_continuous(breaks = 1:3, labels = c("bad", "neutral", "good")) + 
  theme(legend.position = "none",
        axis.title.x = element_blank())
```

<img src="13-linear_model4_files/figure-html/unnamed-chunk-4-1.png" width="672" />

### Hypothetical data 

Here is some code to generate a hypothetical developmental data set. 


```r
# make example reproducible 
set.seed(1)

# means = c(5, 10, 5)
means = c(3, 5, 20)
# means = c(3, 5, 7)
# means = c(3, 7, 12)
sd = 2
sample_size = 20

# generate data 
df.development = tibble(
  group = rep(c("3-4", "5-6", "7-8"), each = sample_size),
  performance = NA) %>% 
  mutate(performance = ifelse(group == "3-4",
                              rnorm(sample_size,
                                    mean = means[1],
                                    sd = sd),
                              performance),
         performance = ifelse(group == "5-6",
                              rnorm(sample_size,
                                    mean = means[2],
                                    sd = sd),
                              performance),
         performance = ifelse(group == "7-8",
                              rnorm(sample_size,
                                    mean = means[3],
                                    sd = sd),
                              performance),
         group = factor(group, levels = c("3-4", "5-6", "7-8")),
         group_contrast = group %>% 
           fct_recode(`-1` = "3-4",
                      `0` = "5-6",
                      `1` = "7-8") %>% 
           as.character() %>%
           as.numeric())
```

Let's define a linear contrast using the `emmeans` package, and test whether it's significant. 


```r
fit = lm(formula = performance ~ group,
         data = df.development)

fit %>% 
  emmeans("group",
          contr = list(linear = c(-0.5, 0, 0.5)),
          adjust = "bonferroni") %>% 
  pluck("contrasts")
```

```
 contrast estimate    SE df t.ratio p.value
 linear       8.45 0.274 57 30.856  <.0001 
```

Yes, we see that there is a significant positive linear contrast with an estimate of 8.45. This means, it predicts a difference of 8.45 in performance between each of the consecutive age groups. For a visualization of the predictions of this model, see Figure \@ref{fig:linear-contrast-model}. 

### Visualization

Total variance: 


```r
set.seed(1)

fit_c = lm(formula = performance ~ 1,
           data = df.development)

df.plot = df.development %>% 
  mutate(group_jitter = 1 + runif(n(), min = -0.25, max = 0.25))

df.augment = fit_c %>% 
  augment() %>% 
  clean_names() %>% 
  bind_cols(df.plot %>% select(group, group_jitter))

ggplot(data = df.plot, 
       mapping = aes(x = group_jitter,
                       y = performance,
                       fill = group)) + 
  geom_hline(yintercept = mean(df.development$performance)) +
  geom_point(alpha = 0.5) + 
  geom_segment(data = df.augment,
               aes(xend = group_jitter,
                   yend = fitted),
               alpha = 0.2) +
  labs(y = "performance") + 
  theme(legend.position = "none",
        axis.text.x = element_blank(),
        axis.title.x = element_blank())
```

<img src="13-linear_model4_files/figure-html/unnamed-chunk-7-1.png" width="672" />

With contrast


```r
# make example reproducible 
set.seed(1)

fit = lm(formula = performance ~ group_contrast,
         data = df.development)

df.plot = df.development %>% 
  mutate(group_jitter = group %>% as.numeric(),
         group_jitter = group_jitter + runif(n(), min = -0.4, max = 0.4))

df.tidy = fit %>% 
  tidy() %>% 
  select(where(is.numeric)) %>% 
  mutate(across(.fns = ~ round(. , 2)))

df.augment = fit %>% 
  augment() %>%
  clean_names() %>% 
  bind_cols(df.plot %>% select(group_jitter))

ggplot(data = df.plot,
       mapping = aes(x = group_jitter,
                       y = performance,
                       color = as.factor(group_contrast))) + 
  geom_point(alpha = 0.8) +
  geom_segment(data = NULL,
               aes(x = 0.6,
                   xend = 1.4,
                   y = df.tidy$estimate[1]-df.tidy$estimate[2],
                   yend = df.tidy$estimate[1]-df.tidy$estimate[2]),
               color = "red",
               size = 1) +
  geom_segment(data = NULL,
               aes(x = 1.6,
                   xend = 2.4,
                   y = df.tidy$estimate[1],
                   yend = df.tidy$estimate[1]),
               color = "orange",
               size = 1) +
  geom_segment(data = NULL,
               aes(x = 2.6,
                   xend = 3.4,
                   y = df.tidy$estimate[1] + df.tidy$estimate[2],
                   yend = df.tidy$estimate[1] + df.tidy$estimate[2]),
               color = "green",
               size = 1) +
  geom_segment(data = df.augment,
               aes(xend = group_jitter,
                   y = performance,
                   yend = fitted),
               alpha = 0.3) +
  labs(y = "performance") + 
  scale_color_manual(values = c("red", "orange", "green")) + 
  scale_x_continuous(breaks = 1:3, labels = levels(df.development$group)) +
  theme(legend.position = "none",
        axis.title.x = element_blank())
```

<div class="figure">
<img src="13-linear_model4_files/figure-html/linear-contrast-model-1.png" alt="Predictions of the linear contrast model" width="672" />
<p class="caption">(\#fig:linear-contrast-model)Predictions of the linear contrast model</p>
</div>

Results figure


```r
df.development %>% 
  ggplot(aes(x = group, y = performance)) + 
  geom_point(alpha = 0.3, position = position_jitter(width = 0.1, height = 0)) +
  stat_summary(fun.data = "mean_cl_boot",
               shape = 21, 
               fill = "white",
               size = 0.75)
```

<img src="13-linear_model4_files/figure-html/unnamed-chunk-8-1.png" width="672" />

Here we test some more specific hypotheses: the the two youngest groups of children are different from the oldest group, and that the 3 year olds are different from the 5 year olds. 


```r
#  fit the linear model 
fit = lm(formula = performance ~ group,
         data = df.development)

# check factor levels 
levels(df.development$group)
```

```
[1] "3-4" "5-6" "7-8"
```

```r
# define the contrasts of interest 
contrasts = list(young_vs_old = c(-0.5, -0.5, 1),
                 three_vs_five = c(-0.5, 0.5, 0))

# compute significance test on contrasts 
fit %>% 
  emmeans("group",
          contr = contrasts,
          adjust = "bonferroni") %>% 
  pluck("contrasts")
```

```
 contrast      estimate    SE df t.ratio p.value
 young_vs_old    16.094 0.474 57 33.936  <.0001 
 three_vs_five    0.803 0.274 57  2.933  0.0097 

P value adjustment: bonferroni method for 2 tests 
```

### Post-hoc tests

Post-hoc tests for a single predictor (using the poker data set). 


```r
fit = lm(formula = balance ~ hand,
         data = df.poker)

# post hoc tests 
fit %>% 
  emmeans(pairwise ~ hand,
          adjust = "bonferroni") %>% 
  pluck("contrasts")
```

```
 contrast       estimate    SE  df t.ratio p.value
 bad - neutral     -4.41 0.581 297  -7.576 <.0001 
 bad - good        -7.08 0.581 297 -12.185 <.0001 
 neutral - good    -2.68 0.581 297  -4.609 <.0001 

P value adjustment: bonferroni method for 3 tests 
```

Post-hoc tests for two predictors (:


```r
# fit the model
fit = lm(formula = balance ~ hand + skill,
         data = df.poker)

# post hoc tests 
fit %>% 
  emmeans(pairwise ~ hand + skill,
          adjust = "bonferroni") %>% 
  pluck("contrasts")
```

```
 contrast                         estimate    SE  df t.ratio p.value
 bad average - neutral average      -4.405 0.580 296  -7.593 <.0001 
 bad average - good average         -7.085 0.580 296 -12.212 <.0001 
 bad average - bad expert           -0.724 0.474 296  -1.529 1.0000 
 bad average - neutral expert       -5.129 0.749 296  -6.849 <.0001 
 bad average - good expert          -7.809 0.749 296 -10.427 <.0001 
 neutral average - good average     -2.680 0.580 296  -4.619 0.0001 
 neutral average - bad expert        3.681 0.749 296   4.914 <.0001 
 neutral average - neutral expert   -0.724 0.474 296  -1.529 1.0000 
 neutral average - good expert      -3.404 0.749 296  -4.545 0.0001 
 good average - bad expert           6.361 0.749 296   8.492 <.0001 
 good average - neutral expert       1.955 0.749 296   2.611 0.1424 
 good average - good expert         -0.724 0.474 296  -1.529 1.0000 
 bad expert - neutral expert        -4.405 0.580 296  -7.593 <.0001 
 bad expert - good expert           -7.085 0.580 296 -12.212 <.0001 
 neutral expert - good expert       -2.680 0.580 296  -4.619 0.0001 

P value adjustment: bonferroni method for 15 tests 
```



```r
fit = lm(formula = balance ~ hand,
         data = df.poker)

# comparing each to the mean 
fit %>% 
  emmeans(eff ~ hand) %>% 
  pluck("contrasts")
```

```
 contrast       estimate    SE  df t.ratio p.value
 bad effect       -3.830 0.336 297 -11.409 <.0001 
 neutral effect    0.575 0.336 297   1.713 0.0877 
 good effect       3.255 0.336 297   9.696 <.0001 

P value adjustment: fdr method for 3 tests 
```

```r
# one vs. all others 
fit %>% 
  emmeans(del.eff ~ hand) %>% 
  pluck("contrasts")
```

```
 contrast       estimate    SE  df t.ratio p.value
 bad effect       -5.745 0.504 297 -11.409 <.0001 
 neutral effect    0.863 0.504 297   1.713 0.0877 
 good effect       4.882 0.504 297   9.696 <.0001 

P value adjustment: fdr method for 3 tests 
```

### Understanding dummy coding 


```r
fit = lm(formula = balance ~ 1 + hand,
         data = df.poker)

fit %>% 
  summary()
```

```

Call:
lm(formula = balance ~ 1 + hand, data = df.poker)

Residuals:
     Min       1Q   Median       3Q      Max 
-12.9264  -2.5902  -0.0115   2.6573  15.2834 

Coefficients:
            Estimate Std. Error t value Pr(>|t|)    
(Intercept)   5.9415     0.4111  14.451  < 2e-16 ***
handneutral   4.4051     0.5815   7.576 4.55e-13 ***
handgood      7.0849     0.5815  12.185  < 2e-16 ***
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

Residual standard error: 4.111 on 297 degrees of freedom
Multiple R-squared:  0.3377,	Adjusted R-squared:  0.3332 
F-statistic:  75.7 on 2 and 297 DF,  p-value: < 2.2e-16
```

```r
model.matrix(fit) %>% 
  as_tibble() %>% 
  distinct()
```

```
# A tibble: 3 x 3
  `(Intercept)` handneutral handgood
          <dbl>       <dbl>    <dbl>
1             1           0        0
2             1           1        0
3             1           0        1
```

```r
df.poker %>% 
  select(participant, hand, balance) %>% 
  group_by(hand) %>% 
  top_n(3, wt = -participant) %>% 
  kable(digits = 2) %>% 
  kable_styling(bootstrap_options = "striped",
                full_width = F)
```

<table class="table table-striped" style="width: auto !important; margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:right;"> participant </th>
   <th style="text-align:left;"> hand </th>
   <th style="text-align:right;"> balance </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:left;"> bad </td>
   <td style="text-align:right;"> 4.00 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2 </td>
   <td style="text-align:left;"> bad </td>
   <td style="text-align:right;"> 5.55 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 3 </td>
   <td style="text-align:left;"> bad </td>
   <td style="text-align:right;"> 9.45 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 51 </td>
   <td style="text-align:left;"> neutral </td>
   <td style="text-align:right;"> 11.74 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 52 </td>
   <td style="text-align:left;"> neutral </td>
   <td style="text-align:right;"> 10.04 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 53 </td>
   <td style="text-align:left;"> neutral </td>
   <td style="text-align:right;"> 9.49 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 101 </td>
   <td style="text-align:left;"> good </td>
   <td style="text-align:right;"> 10.86 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 102 </td>
   <td style="text-align:left;"> good </td>
   <td style="text-align:right;"> 8.68 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 103 </td>
   <td style="text-align:left;"> good </td>
   <td style="text-align:right;"> 14.36 </td>
  </tr>
</tbody>
</table>

### Understanding effect coding 


```r
fit = lm(formula = balance ~ 1 + hand,
         contrasts = list(hand = "contr.sum"),
         data = df.poker)

fit %>% 
  summary()
```

```

Call:
lm(formula = balance ~ 1 + hand, data = df.poker, contrasts = list(hand = "contr.sum"))

Residuals:
     Min       1Q   Median       3Q      Max 
-12.9264  -2.5902  -0.0115   2.6573  15.2834 

Coefficients:
            Estimate Std. Error t value Pr(>|t|)    
(Intercept)   9.7715     0.2374  41.165   <2e-16 ***
hand1        -3.8300     0.3357 -11.409   <2e-16 ***
hand2         0.5751     0.3357   1.713   0.0877 .  
---
Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

Residual standard error: 4.111 on 297 degrees of freedom
Multiple R-squared:  0.3377,	Adjusted R-squared:  0.3332 
F-statistic:  75.7 on 2 and 297 DF,  p-value: < 2.2e-16
```

```r
model.matrix(fit) %>% 
  as_tibble() %>% 
  distinct() %>% 
  kable(digits = 2) %>% 
  kable_styling(bootstrap_options = "striped",
                full_width = F)
```

<table class="table table-striped" style="width: auto !important; margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:right;"> (Intercept) </th>
   <th style="text-align:right;"> hand1 </th>
   <th style="text-align:right;"> hand2 </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 0 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> -1 </td>
   <td style="text-align:right;"> -1 </td>
  </tr>
</tbody>
</table>

## Session info 

Information about this R session including which version of R was used, and what packages were loaded. 


```r
sessionInfo()
```

```
R version 4.0.3 (2020-10-10)
Platform: x86_64-apple-darwin17.0 (64-bit)
Running under: macOS Catalina 10.15.7

Matrix products: default
BLAS:   /Library/Frameworks/R.framework/Versions/4.0/Resources/lib/libRblas.dylib
LAPACK: /Library/Frameworks/R.framework/Versions/4.0/Resources/lib/libRlapack.dylib

locale:
[1] en_US.UTF-8/en_US.UTF-8/en_US.UTF-8/C/en_US.UTF-8/en_US.UTF-8

attached base packages:
[1] stats     graphics  grDevices utils     datasets  methods   base     

other attached packages:
 [1] forcats_0.5.1    stringr_1.4.0    dplyr_1.0.4      purrr_0.3.4     
 [5] readr_1.4.0      tidyr_1.1.2      tibble_3.0.6     ggplot2_3.3.3   
 [9] tidyverse_1.3.0  car_3.0-10       carData_3.0-4    emmeans_1.5.3   
[13] afex_0.28-1      lme4_1.1-26      Matrix_1.3-2     broom_0.7.3     
[17] janitor_2.1.0    kableExtra_1.3.1 knitr_1.31      

loaded via a namespace (and not attached):
  [1] TH.data_1.0-10      minqa_1.2.4         colorspace_2.0-0   
  [4] ellipsis_0.3.1      rio_0.5.16          htmlTable_2.1.0    
  [7] estimability_1.3    snakecase_0.11.0    base64enc_0.1-3    
 [10] fs_1.5.0            rstudioapi_0.13     farver_2.1.0       
 [13] fansi_0.4.2         mvtnorm_1.1-1       lubridate_1.7.9.2  
 [16] xml2_1.3.2          codetools_0.2-18    splines_4.0.3      
 [19] Formula_1.2-4       jsonlite_1.7.2      nloptr_1.2.2.2     
 [22] cluster_2.1.0       dbplyr_2.0.0        png_0.1-7          
 [25] compiler_4.0.3      httr_1.4.2          backports_1.2.1    
 [28] assertthat_0.2.1    cli_2.3.0           htmltools_0.5.1.1  
 [31] tools_4.0.3         lmerTest_3.1-3      coda_0.19-4        
 [34] gtable_0.3.0        glue_1.4.2          reshape2_1.4.4     
 [37] Rcpp_1.0.6          cellranger_1.1.0    vctrs_0.3.6        
 [40] nlme_3.1-151        xfun_0.21           ps_1.6.0           
 [43] openxlsx_4.2.3      rvest_0.3.6         lifecycle_1.0.0    
 [46] statmod_1.4.35      MASS_7.3-53         zoo_1.8-8          
 [49] scales_1.1.1        hms_1.0.0           parallel_4.0.3     
 [52] sandwich_3.0-0      RColorBrewer_1.1-2  yaml_2.2.1         
 [55] curl_4.3            gridExtra_2.3       rpart_4.1-15       
 [58] latticeExtra_0.6-29 stringi_1.5.3       highr_0.8          
 [61] checkmate_2.0.0     boot_1.3-26         zip_2.1.1          
 [64] rlang_0.4.10        pkgconfig_2.0.3     evaluate_0.14      
 [67] lattice_0.20-41     htmlwidgets_1.5.3   labeling_0.4.2     
 [70] tidyselect_1.1.0    plyr_1.8.6          magrittr_2.0.1     
 [73] bookdown_0.21       R6_2.5.0            generics_0.1.0     
 [76] Hmisc_4.4-2         multcomp_1.4-15     DBI_1.1.1          
 [79] pillar_1.4.7        haven_2.3.1         foreign_0.8-81     
 [82] withr_2.4.1         nnet_7.3-15         survival_3.2-7     
 [85] abind_1.4-5         modelr_0.1.8        crayon_1.4.1       
 [88] utf8_1.1.4          rmarkdown_2.6       jpeg_0.1-8.1       
 [91] grid_4.0.3          readxl_1.3.1        data.table_1.13.6  
 [94] reprex_1.0.0        digest_0.6.27       webshot_0.5.2      
 [97] xtable_1.8-4        numDeriv_2016.8-1.1 munsell_0.5.0      
[100] viridisLite_0.3.0  
```

## References
