---
title: "NewRT (Cleaned Data)"
output: html_document
date: "2026-03-08"
---


``` r
# Load necessary libraries
library(lme4)
```

```
## 载入需要的程序包：Matrix
```

``` r
library(dplyr)
```

```
## 
## 载入程序包：'dplyr'
```

```
## The following objects are masked from 'package:stats':
## 
##     filter, lag
```

```
## The following objects are masked from 'package:base':
## 
##     intersect, setdiff, setequal, union
```

``` r
library(afex)
```

```
## ************
## Welcome to afex. For support visit: http://afex.singmann.science/
```

```
## - Functions for ANOVAs: aov_car(), aov_ez(), and aov_4()
## - Methods for calculating p-values with mixed(): 'S', 'KR', 'LRT', and 'PB'
## - 'afex_aov' and 'mixed' objects can be passed to emmeans() for follow-up tests
## - Get and set global package options with: afex_options()
## - Set sum-to-zero contrasts globally: set_sum_contrasts()
## - For example analyses see: browseVignettes("afex")
## ************
```

```
## 
## 载入程序包：'afex'
```

```
## The following object is masked from 'package:lme4':
## 
##     lmer
```

``` r
library(emmeans)
```

```
## Welcome to emmeans.
## Caution: You lose important information if you filter this package's results.
## See '? untidy'
```

``` r
library(ggplot2)
library(tools)

afex_options(type = 3)

# Define cleaned-data folder paths
partial_pass_folder_path <- "C:/Users/Jack/OneDrive/Desktop/follow up/cleaned_no4060_batch/PartialPass4060"
full_pass_folder_path    <- "C:/Users/Jack/OneDrive/Desktop/follow up/cleaned_no4060_batch/Pass4060"

interaction_data <- data.frame()

process_files_rt <- function(folder_path, feedback_type) {
  files <- list.files(folder_path, pattern = "_cleaned\\.csv$", full.names = TRUE)

  for (file_path in files) {
    df <- read.csv(file_path, stringsAsFactors = FALSE)
    participant <- file_path_sans_ext(basename(file_path))
    participant <- sub("_cleaned$", "", participant)

    if (all(c("trial_index", "task_name", "frame", "rt_final") %in% colnames(df))) {
      tmp <- df %>%
        mutate(
          trial_index = suppressWarnings(as.numeric(trial_index)),
          rt_final = suppressWarnings(as.numeric(rt_final)),
          Participant = participant,
          Feedback_Type = feedback_type,
          Time_Pressure = ifelse(grepl("_TP", task_name, ignore.case = TRUE), "Yes", "No"),
          Choice_Frame = frame
        ) %>%
        filter(!is.na(trial_index), is.finite(rt_final), rt_final > 0) %>%
        arrange(trial_index) %>%
        group_by(Participant, Feedback_Type, Time_Pressure, Choice_Frame) %>%
        mutate(Blocks = (row_number() - 1L) %/% 10L + 1L) %>%
        ungroup() %>%
        select(Participant, Feedback_Type, Time_Pressure, Choice_Frame, Blocks, trial_index, rt_final)

      interaction_data <<- bind_rows(interaction_data, tmp)
    }
  }
}

process_files_rt(partial_pass_folder_path, "Partial")
process_files_rt(full_pass_folder_path, "Full")

interaction_data <- interaction_data %>%
  mutate(
    Participant = as.factor(Participant),
    Feedback_Type = factor(Feedback_Type, levels = c("Full", "Partial")),
    Time_Pressure = factor(Time_Pressure, levels = c("No", "Yes")),
    Choice_Frame = factor(Choice_Frame, levels = c("Accept", "Reject")),
    Blocks = as.factor(Blocks),
    log_rt = log(rt_final)
  ) %>%
  filter(is.finite(log_rt))

print(head(interaction_data))
```

```
##                                                Participant Feedback_Type
## 1 Mega_project_PARTICIPANT_SESSION_2025-11-24_10h18.02.511       Partial
## 2 Mega_project_PARTICIPANT_SESSION_2025-11-24_10h18.02.511       Partial
## 3 Mega_project_PARTICIPANT_SESSION_2025-11-24_10h18.02.511       Partial
## 4 Mega_project_PARTICIPANT_SESSION_2025-11-24_10h18.02.511       Partial
## 5 Mega_project_PARTICIPANT_SESSION_2025-11-24_10h18.02.511       Partial
## 6 Mega_project_PARTICIPANT_SESSION_2025-11-24_10h18.02.511       Partial
##   Time_Pressure Choice_Frame Blocks trial_index rt_final   log_rt
## 1           Yes       Reject      1          17      662 6.495266
## 2           Yes       Reject      1          22     1447 7.277248
## 3           Yes       Reject      1          27      649 6.475433
## 4           Yes       Reject      1          31      362 5.891644
## 5           Yes       Reject      1          36      561 6.329721
## 6           Yes       Reject      1          40      383 5.948035
```

``` r
cat("Rows:", nrow(interaction_data), " Participants:", n_distinct(interaction_data$Participant), "\n")
```

```
## Rows: 19356  Participants: 121
```


``` r
rt_mixed <- mixed(
  log_rt ~ Blocks * Time_Pressure * Choice_Frame * Feedback_Type + (1 | Participant),
  data   = interaction_data,
  method = "KR",
  type   = 3,
  progress = TRUE,
  control  = lmerControl(optimizer = "bobyqa", optCtrl = list(maxfun = 100000))
)
```

```
## Contrasts set to contr.sum for the following variables: Blocks, Time_Pressure, Choice_Frame, Feedback_Type, Participant
```

```
## Fitting one lmer() model. [DONE]
## Calculating p-values. [DONE]
```

``` r
print(summary(rt_mixed))
```

```
## Linear mixed model fit by REML. t-tests use Satterthwaite's method [
## lmerModLmerTest]
## Formula: log_rt ~ Blocks * Time_Pressure * Choice_Frame * Feedback_Type +  
##     (1 | Participant)
##    Data: data
## Control: lmerControl(optimizer = "bobyqa", optCtrl = list(maxfun = 1e+05))
## 
## REML criterion at convergence: 37107.9
## 
## Scaled residuals: 
##     Min      1Q  Median      3Q     Max 
## -9.0829 -0.4715  0.0035  0.5037  7.1092 
## 
## Random effects:
##  Groups      Name        Variance Std.Dev.
##  Participant (Intercept) 0.07777  0.2789  
##  Residual                0.38506  0.6205  
## Number of obs: 19356, groups:  Participant, 121
## 
## Fixed effects:
##                                                       Estimate Std. Error
## (Intercept)                                          5.912e+00  2.581e-02
## Blocks1                                              2.335e-01  7.746e-03
## Blocks2                                             -4.747e-02  7.746e-03
## Blocks3                                             -7.951e-02  7.746e-03
## Time_Pressure1                                       4.663e-01  4.473e-03
## Choice_Frame1                                        2.547e-03  4.473e-03
## Feedback_Type1                                       4.619e-02  2.581e-02
## Blocks1:Time_Pressure1                               4.731e-02  7.746e-03
## Blocks2:Time_Pressure1                               1.778e-02  7.746e-03
## Blocks3:Time_Pressure1                              -3.546e-02  7.746e-03
## Blocks1:Choice_Frame1                               -4.625e-02  7.746e-03
## Blocks2:Choice_Frame1                               -8.610e-04  7.746e-03
## Blocks3:Choice_Frame1                                1.749e-02  7.746e-03
## Time_Pressure1:Choice_Frame1                         2.627e-02  4.473e-03
## Blocks1:Feedback_Type1                               8.549e-03  7.746e-03
## Blocks2:Feedback_Type1                               7.498e-03  7.746e-03
## Blocks3:Feedback_Type1                              -3.554e-03  7.746e-03
## Time_Pressure1:Feedback_Type1                        1.943e-02  4.473e-03
## Choice_Frame1:Feedback_Type1                        -8.003e-03  4.473e-03
## Blocks1:Time_Pressure1:Choice_Frame1                -4.895e-03  7.746e-03
## Blocks2:Time_Pressure1:Choice_Frame1                 1.083e-02  7.746e-03
## Blocks3:Time_Pressure1:Choice_Frame1                 1.783e-03  7.746e-03
## Blocks1:Time_Pressure1:Feedback_Type1                7.034e-03  7.746e-03
## Blocks2:Time_Pressure1:Feedback_Type1                3.641e-03  7.746e-03
## Blocks3:Time_Pressure1:Feedback_Type1                4.143e-03  7.746e-03
## Blocks1:Choice_Frame1:Feedback_Type1                -4.002e-03  7.746e-03
## Blocks2:Choice_Frame1:Feedback_Type1                -1.710e-03  7.746e-03
## Blocks3:Choice_Frame1:Feedback_Type1                 9.019e-03  7.746e-03
## Time_Pressure1:Choice_Frame1:Feedback_Type1          6.029e-03  4.473e-03
## Blocks1:Time_Pressure1:Choice_Frame1:Feedback_Type1  2.553e-03  7.746e-03
## Blocks2:Time_Pressure1:Choice_Frame1:Feedback_Type1 -1.710e-02  7.746e-03
## Blocks3:Time_Pressure1:Choice_Frame1:Feedback_Type1 -5.376e-03  7.746e-03
##                                                             df t value Pr(>|t|)
## (Intercept)                                          1.190e+02 229.049  < 2e-16
## Blocks1                                              1.921e+04  30.144  < 2e-16
## Blocks2                                              1.921e+04  -6.128 9.06e-10
## Blocks3                                              1.921e+04 -10.265  < 2e-16
## Time_Pressure1                                       1.921e+04 104.250  < 2e-16
## Choice_Frame1                                        1.921e+04   0.569   0.5691
## Feedback_Type1                                       1.190e+02   1.789   0.0761
## Blocks1:Time_Pressure1                               1.921e+04   6.108 1.03e-09
## Blocks2:Time_Pressure1                               1.921e+04   2.295   0.0218
## Blocks3:Time_Pressure1                               1.921e+04  -4.578 4.73e-06
## Blocks1:Choice_Frame1                                1.921e+04  -5.971 2.41e-09
## Blocks2:Choice_Frame1                                1.921e+04  -0.111   0.9115
## Blocks3:Choice_Frame1                                1.921e+04   2.258   0.0239
## Time_Pressure1:Choice_Frame1                         1.921e+04   5.873 4.36e-09
## Blocks1:Feedback_Type1                               1.921e+04   1.104   0.2698
## Blocks2:Feedback_Type1                               1.921e+04   0.968   0.3331
## Blocks3:Feedback_Type1                               1.921e+04  -0.459   0.6464
## Time_Pressure1:Feedback_Type1                        1.921e+04   4.345 1.40e-05
## Choice_Frame1:Feedback_Type1                         1.921e+04  -1.789   0.0736
## Blocks1:Time_Pressure1:Choice_Frame1                 1.921e+04  -0.632   0.5274
## Blocks2:Time_Pressure1:Choice_Frame1                 1.921e+04   1.398   0.1621
## Blocks3:Time_Pressure1:Choice_Frame1                 1.921e+04   0.230   0.8179
## Blocks1:Time_Pressure1:Feedback_Type1                1.921e+04   0.908   0.3639
## Blocks2:Time_Pressure1:Feedback_Type1                1.921e+04   0.470   0.6383
## Blocks3:Time_Pressure1:Feedback_Type1                1.921e+04   0.535   0.5927
## Blocks1:Choice_Frame1:Feedback_Type1                 1.921e+04  -0.517   0.6055
## Blocks2:Choice_Frame1:Feedback_Type1                 1.921e+04  -0.221   0.8253
## Blocks3:Choice_Frame1:Feedback_Type1                 1.921e+04   1.164   0.2443
## Time_Pressure1:Choice_Frame1:Feedback_Type1          1.921e+04   1.348   0.1777
## Blocks1:Time_Pressure1:Choice_Frame1:Feedback_Type1  1.921e+04   0.330   0.7417
## Blocks2:Time_Pressure1:Choice_Frame1:Feedback_Type1  1.921e+04  -2.207   0.0273
## Blocks3:Time_Pressure1:Choice_Frame1:Feedback_Type1  1.921e+04  -0.694   0.4877
##                                                        
## (Intercept)                                         ***
## Blocks1                                             ***
## Blocks2                                             ***
## Blocks3                                             ***
## Time_Pressure1                                      ***
## Choice_Frame1                                          
## Feedback_Type1                                      .  
## Blocks1:Time_Pressure1                              ***
## Blocks2:Time_Pressure1                              *  
## Blocks3:Time_Pressure1                              ***
## Blocks1:Choice_Frame1                               ***
## Blocks2:Choice_Frame1                                  
## Blocks3:Choice_Frame1                               *  
## Time_Pressure1:Choice_Frame1                        ***
## Blocks1:Feedback_Type1                                 
## Blocks2:Feedback_Type1                                 
## Blocks3:Feedback_Type1                                 
## Time_Pressure1:Feedback_Type1                       ***
## Choice_Frame1:Feedback_Type1                        .  
## Blocks1:Time_Pressure1:Choice_Frame1                   
## Blocks2:Time_Pressure1:Choice_Frame1                   
## Blocks3:Time_Pressure1:Choice_Frame1                   
## Blocks1:Time_Pressure1:Feedback_Type1                  
## Blocks2:Time_Pressure1:Feedback_Type1                  
## Blocks3:Time_Pressure1:Feedback_Type1                  
## Blocks1:Choice_Frame1:Feedback_Type1                   
## Blocks2:Choice_Frame1:Feedback_Type1                   
## Blocks3:Choice_Frame1:Feedback_Type1                   
## Time_Pressure1:Choice_Frame1:Feedback_Type1            
## Blocks1:Time_Pressure1:Choice_Frame1:Feedback_Type1    
## Blocks2:Time_Pressure1:Choice_Frame1:Feedback_Type1 *  
## Blocks3:Time_Pressure1:Choice_Frame1:Feedback_Type1    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
```

```
## 
## Correlation matrix not shown by default, as p = 32 > 12.
## Use print(summary(rt_mixed), correlation=TRUE)  or
##     vcov(summary(rt_mixed))        if you need it
```

``` r
print(rt_mixed)
```

```
## Mixed Model Anova Table (Type 3 tests, KR-method)
## 
## Model: log_rt ~ Blocks * Time_Pressure * Choice_Frame * Feedback_Type + 
## Model:     (1 | Participant)
## Data: interaction_data
##                                             Effect          df            F
## 1                                           Blocks 3, 19205.00   310.15 ***
## 2                                    Time_Pressure 1, 19205.00 10868.06 ***
## 3                                     Choice_Frame 1, 19205.00         0.32
## 4                                    Feedback_Type   1, 119.00       3.20 +
## 5                             Blocks:Time_Pressure 3, 19205.00    19.54 ***
## 6                              Blocks:Choice_Frame 3, 19205.00    13.84 ***
## 7                       Time_Pressure:Choice_Frame 1, 19205.00    34.49 ***
## 8                             Blocks:Feedback_Type 3, 19205.00         1.24
## 9                      Time_Pressure:Feedback_Type 1, 19205.00    18.88 ***
## 10                      Choice_Frame:Feedback_Type 1, 19205.00       3.20 +
## 11               Blocks:Time_Pressure:Choice_Frame 3, 19205.00         0.85
## 12              Blocks:Time_Pressure:Feedback_Type 3, 19205.00         1.25
## 13               Blocks:Choice_Frame:Feedback_Type 3, 19205.00         0.46
## 14        Time_Pressure:Choice_Frame:Feedback_Type 1, 19205.00         1.82
## 15 Blocks:Time_Pressure:Choice_Frame:Feedback_Type 3, 19205.00       3.02 *
##    p.value
## 1    <.001
## 2    <.001
## 3     .569
## 4     .076
## 5    <.001
## 6    <.001
## 7    <.001
## 8     .293
## 9    <.001
## 10    .074
## 11    .466
## 12    .291
## 13    .708
## 14    .178
## 15    .029
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '+' 0.1 ' ' 1
```


``` r
sjPlot::plot_model(rt_mixed, type = "est")
```

![plot of chunk unnamed-chunk-3](figure/unnamed-chunk-3-1.png)


``` r
# Significant effects only + plots
library(afex)
library(emmeans)
library(dplyr)
library(ggplot2)

p_to_stars <- function(p) {
  if (is.na(p)) return("")
  if (p < .001) return("***")
  if (p < .01)  return("**")
  if (p < .05)  return("*")
  return("")
}

aov_tab <- as.data.frame(rt_mixed$anova_table)
aov_tab$Effect <- rownames(aov_tab)

p_col <- if ("Pr(>F)" %in% names(aov_tab)) "Pr(>F)" else if ("p.value" %in% names(aov_tab)) "p.value" else "p.value"
F_col <- if ("F" %in% names(aov_tab)) "F" else NA_character_
df_col <- if ("df" %in% names(aov_tab)) "df" else if ("num Df" %in% names(aov_tab)) "num Df" else NA_character_

aov_tab$p <- aov_tab[[p_col]]
aov_tab$stat <- if (!is.na(F_col)) aov_tab[[F_col]] else NA_real_
aov_tab$stars <- vapply(aov_tab$p, p_to_stars, character(1))

sig_tab <- aov_tab %>% filter(!is.na(p) & p < .05)

display_cols <- c("Effect", if (!is.na(df_col)) df_col, "stat", "p", "stars")
cat("\nSignificant effects (p < .05):\n")
```

```
## 
## Significant effects (p < .05):
```

``` r
print(sig_tab[, display_cols, drop = FALSE], row.names = FALSE)
```

```
##                                           Effect num Df        stat
##                                           Blocks      3   310.15118
##                                    Time_Pressure      1 10868.06436
##                             Blocks:Time_Pressure      3    19.53714
##                              Blocks:Choice_Frame      3    13.84271
##                       Time_Pressure:Choice_Frame      1    34.48978
##                      Time_Pressure:Feedback_Type      1    18.87950
##  Blocks:Time_Pressure:Choice_Frame:Feedback_Type      3     3.01807
##              p stars
##  1.182826e-196   ***
##   0.000000e+00   ***
##   1.215027e-12   ***
##   5.161558e-09   ***
##   4.355608e-09   ***
##   1.399559e-05   ***
##   2.860341e-02     *
```

``` r
sig_effects <- sig_tab$Effect
m_lmer <- rt_mixed$full_model

stars_for <- function(effect_name) {
  s <- sig_tab$stars[sig_tab$Effect == effect_name]
  if (length(s) == 0) "" else s[[1]]
}

plot_main_effect <- function(effect_name) {
  emm <- emmeans(m_lmer, as.formula(paste0("~ ", effect_name)))
  df  <- as.data.frame(emm)
  names(df)[names(df) == effect_name] <- "X"

  ggplot(df, aes(x = X, y = emmean, fill = X)) +
    geom_col(width = 0.65) +
    geom_errorbar(aes(ymin = emmean - SE, ymax = emmean + SE), width = 0.2) +
    labs(
      title = paste0("Main effect: ", effect_name, " ", stars_for(effect_name)),
      x = effect_name,
      y = "Estimated marginal mean (log RT)"
    ) +
    theme_minimal() +
    theme(legend.position = "none", axis.title = element_text(face = "bold"))
}

plot_two_way <- function(effect_name) {
  parts <- unlist(strsplit(effect_name, ":", fixed = TRUE))
  A <- parts[1]; B <- parts[2]
  emm <- emmeans(m_lmer, as.formula(paste0("~ ", A, " * ", B)))
  df <- as.data.frame(emm)

  ggplot(df, aes_string(x = A, y = "emmean", fill = B)) +
    geom_col(position = position_dodge(0.7), width = 0.6) +
    geom_errorbar(aes(ymin = emmean - SE, ymax = emmean + SE),
                  position = position_dodge(0.7), width = 0.2) +
    labs(
      title = paste0("Interaction: ", effect_name, " ", stars_for(effect_name)),
      x = A, y = "Estimated marginal mean (log RT)", fill = B
    ) +
    theme_minimal() + theme(axis.title = element_text(face = "bold"), legend.position = "bottom")
}

plot_three_way <- function(effect_name) {
  parts <- unlist(strsplit(effect_name, ":", fixed = TRUE))
  A <- parts[1]; B <- parts[2]; C <- parts[3]
  emm <- emmeans(m_lmer, as.formula(paste0("~ ", A, " * ", B, " * ", C)))
  df <- as.data.frame(emm)

  ggplot(df, aes_string(x = A, y = "emmean", fill = B)) +
    geom_col(position = position_dodge(0.7), width = 0.6) +
    geom_errorbar(aes(ymin = emmean - SE, ymax = emmean + SE),
                  position = position_dodge(0.7), width = 0.2) +
    facet_wrap(as.formula(paste0("~", C))) +
    labs(
      title = paste0("Interaction: ", effect_name, " ", stars_for(effect_name)),
      x = A, y = "Estimated marginal mean (log RT)", fill = B
    ) +
    theme_minimal() + theme(axis.title = element_text(face = "bold"), legend.position = "bottom")
}

plot_blocks_interaction <- function(effect_name) {
  parts <- unlist(strsplit(effect_name, ":", fixed = TRUE))
  others <- setdiff(parts, "Blocks")

  fml <- if (length(others) == 0) {
    ~ Blocks
  } else {
    as.formula(paste0("~ Blocks | ", paste(others, collapse = " * ")))
  }

  emm <- emmeans(m_lmer, fml)
  df <- as.data.frame(emm)

  if (length(others) == 1) {
    ggplot(df, aes(x = Blocks, y = emmean, color = .data[[others[1]]], group = .data[[others[1]]])) +
      geom_point(position = position_dodge(0.15), size = 2) +
      geom_line(position = position_dodge(0.15), linewidth = 1) +
      geom_errorbar(aes(ymin = emmean - SE, ymax = emmean + SE),
                    width = 0.15, position = position_dodge(0.15)) +
      labs(
        title = paste0("Interaction: ", effect_name, " ", stars_for(effect_name)),
        x = "Blocks", y = "Estimated marginal mean (log RT)", color = others[1]
      ) + theme_minimal() + theme(axis.title = element_text(face = "bold"), legend.position = "bottom")
  } else if (length(others) == 2) {
    ggplot(df, aes(x = Blocks, y = emmean, color = .data[[others[1]]], group = .data[[others[1]]])) +
      geom_point(position = position_dodge(0.15), size = 2) +
      geom_line(position = position_dodge(0.15), linewidth = 1) +
      geom_errorbar(aes(ymin = emmean - SE, ymax = emmean + SE),
                    width = 0.15, position = position_dodge(0.15)) +
      facet_wrap(as.formula(paste0("~", others[2]))) +
      labs(
        title = paste0("Interaction: ", effect_name, " ", stars_for(effect_name)),
        x = "Blocks", y = "Estimated marginal mean (log RT)", color = others[1]
      ) + theme_minimal() + theme(axis.title = element_text(face = "bold"), legend.position = "bottom")
  } else {
    ggplot(df, aes(x = Blocks, y = emmean, group = 1)) +
      geom_point(size = 2) + geom_line(linewidth = 1) +
      geom_errorbar(aes(ymin = emmean - SE, ymax = emmean + SE), width = 0.15) +
      labs(
        title = paste0("Interaction: ", effect_name, " ", stars_for(effect_name)),
        x = "Blocks", y = "Estimated marginal mean (log RT)"
      ) + theme_minimal() + theme(axis.title = element_text(face = "bold"))
  }
}

plots <- list()
for (eff in sig_effects) {
  n_way <- length(unlist(strsplit(eff, ":", fixed = TRUE)))
  has_blocks <- grepl("Blocks", eff)
  if (n_way == 1) {
    plots[[eff]] <- plot_main_effect(eff)
  } else if (has_blocks) {
    plots[[eff]] <- plot_blocks_interaction(eff)
  } else if (n_way == 2) {
    plots[[eff]] <- plot_two_way(eff)
  } else if (n_way == 3) {
    plots[[eff]] <- plot_three_way(eff)
  }
}
```

```
## Note: D.f. calculations have been disabled because the number of observations exceeds 3000.
## To enable adjustments, add the argument 'pbkrtest.limit = 19356' (or larger)
## [or, globally, 'set emm_options(pbkrtest.limit = 19356)' or larger];
## but be warned that this may result in large computation time and memory use.
```

```
## Note: D.f. calculations have been disabled because the number of observations exceeds 3000.
## To enable adjustments, add the argument 'lmerTest.limit = 19356' (or larger)
## [or, globally, 'set emm_options(lmerTest.limit = 19356)' or larger];
## but be warned that this may result in large computation time and memory use.
```

```
## NOTE: Results may be misleading due to involvement in interactions
```

```
## Note: D.f. calculations have been disabled because the number of observations exceeds 3000.
## To enable adjustments, add the argument 'pbkrtest.limit = 19356' (or larger)
## [or, globally, 'set emm_options(pbkrtest.limit = 19356)' or larger];
## but be warned that this may result in large computation time and memory use.
```

```
## Note: D.f. calculations have been disabled because the number of observations exceeds 3000.
## To enable adjustments, add the argument 'lmerTest.limit = 19356' (or larger)
## [or, globally, 'set emm_options(lmerTest.limit = 19356)' or larger];
## but be warned that this may result in large computation time and memory use.
```

```
## NOTE: Results may be misleading due to involvement in interactions
```

```
## Note: D.f. calculations have been disabled because the number of observations exceeds 3000.
## To enable adjustments, add the argument 'pbkrtest.limit = 19356' (or larger)
## [or, globally, 'set emm_options(pbkrtest.limit = 19356)' or larger];
## but be warned that this may result in large computation time and memory use.
```

```
## Note: D.f. calculations have been disabled because the number of observations exceeds 3000.
## To enable adjustments, add the argument 'lmerTest.limit = 19356' (or larger)
## [or, globally, 'set emm_options(lmerTest.limit = 19356)' or larger];
## but be warned that this may result in large computation time and memory use.
```

```
## NOTE: Results may be misleading due to involvement in interactions
```

```
## Note: D.f. calculations have been disabled because the number of observations exceeds 3000.
## To enable adjustments, add the argument 'pbkrtest.limit = 19356' (or larger)
## [or, globally, 'set emm_options(pbkrtest.limit = 19356)' or larger];
## but be warned that this may result in large computation time and memory use.
```

```
## Note: D.f. calculations have been disabled because the number of observations exceeds 3000.
## To enable adjustments, add the argument 'lmerTest.limit = 19356' (or larger)
## [or, globally, 'set emm_options(lmerTest.limit = 19356)' or larger];
## but be warned that this may result in large computation time and memory use.
```

```
## NOTE: Results may be misleading due to involvement in interactions
```

```
## Note: D.f. calculations have been disabled because the number of observations exceeds 3000.
## To enable adjustments, add the argument 'pbkrtest.limit = 19356' (or larger)
## [or, globally, 'set emm_options(pbkrtest.limit = 19356)' or larger];
## but be warned that this may result in large computation time and memory use.
```

```
## Note: D.f. calculations have been disabled because the number of observations exceeds 3000.
## To enable adjustments, add the argument 'lmerTest.limit = 19356' (or larger)
## [or, globally, 'set emm_options(lmerTest.limit = 19356)' or larger];
## but be warned that this may result in large computation time and memory use.
```

```
## NOTE: Results may be misleading due to involvement in interactions
```

```
## Warning: `aes_string()` was deprecated in ggplot2 3.0.0.
## ℹ Please use tidy evaluation idioms with `aes()`.
## ℹ See also `vignette("ggplot2-in-packages")` for more information.
## This warning is displayed once every 8 hours.
## Call `lifecycle::last_lifecycle_warnings()` to see where this warning was
## generated.
```

```
## Note: D.f. calculations have been disabled because the number of observations exceeds 3000.
## To enable adjustments, add the argument 'pbkrtest.limit = 19356' (or larger)
## [or, globally, 'set emm_options(pbkrtest.limit = 19356)' or larger];
## but be warned that this may result in large computation time and memory use.
```

```
## Note: D.f. calculations have been disabled because the number of observations exceeds 3000.
## To enable adjustments, add the argument 'lmerTest.limit = 19356' (or larger)
## [or, globally, 'set emm_options(lmerTest.limit = 19356)' or larger];
## but be warned that this may result in large computation time and memory use.
```

```
## NOTE: Results may be misleading due to involvement in interactions
```

```
## Note: D.f. calculations have been disabled because the number of observations exceeds 3000.
## To enable adjustments, add the argument 'pbkrtest.limit = 19356' (or larger)
## [or, globally, 'set emm_options(pbkrtest.limit = 19356)' or larger];
## but be warned that this may result in large computation time and memory use.
```

```
## Note: D.f. calculations have been disabled because the number of observations exceeds 3000.
## To enable adjustments, add the argument 'lmerTest.limit = 19356' (or larger)
## [or, globally, 'set emm_options(lmerTest.limit = 19356)' or larger];
## but be warned that this may result in large computation time and memory use.
```

``` r
for (eff in sig_tab$Effect) {
  if (!is.null(plots[[eff]])) print(plots[[eff]])
}
```

![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-1.png)![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-2.png)![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-3.png)![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-4.png)![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-5.png)![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-6.png)![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-7.png)

``` r
cat("\nReporting-ready lines:\n")
```

```
## 
## Reporting-ready lines:
```

``` r
for (i in seq_len(nrow(sig_tab))) {
  df_text <- if (!is.na(df_col)) as.character(sig_tab[[df_col]][i]) else "NA"
  cat(sprintf("%s: stat(%s) = %.2f, p = %.3g %s\n",
              sig_tab$Effect[i], df_text, sig_tab$stat[i], sig_tab$p[i], sig_tab$stars[i]))
}
```

```
## Blocks: stat(3) = 310.15, p = 1.18e-196 ***
## Time_Pressure: stat(1) = 10868.06, p = 0 ***
## Blocks:Time_Pressure: stat(3) = 19.54, p = 1.22e-12 ***
## Blocks:Choice_Frame: stat(3) = 13.84, p = 5.16e-09 ***
## Time_Pressure:Choice_Frame: stat(1) = 34.49, p = 4.36e-09 ***
## Time_Pressure:Feedback_Type: stat(1) = 18.88, p = 1.4e-05 ***
## Blocks:Time_Pressure:Choice_Frame:Feedback_Type: stat(3) = 3.02, p = 0.0286 *
```
