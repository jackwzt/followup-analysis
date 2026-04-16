---
title: "Risk Preference (Cleaned Data)"
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

# Initialize an empty data frame to store data for analysis
interaction_data <- data.frame()

# Function to load and process cleaned files from a folder
process_files <- function(folder_path, feedback_type) {
  files <- list.files(folder_path, pattern = "_cleaned\\.csv$", full.names = TRUE)

  for (file_path in files) {
    df <- read.csv(file_path, stringsAsFactors = FALSE)
    participant <- file_path_sans_ext(basename(file_path))
    participant <- sub("_cleaned$", "", participant)

    # cleaned files already carry final_choice + risky_choice + rt_final + task_name
    if (all(c("trial_index", "task_name", "frame", "risky_choice") %in% colnames(df))) {

      df <- df %>%
        mutate(
          trial_index = suppressWarnings(as.numeric(trial_index)),
          risky_choice = suppressWarnings(as.numeric(risky_choice)),
          Participant = participant,
          Feedback_Type = feedback_type,
          Time_Pressure = ifelse(grepl("_TP", task_name, ignore.case = TRUE), "Yes", "No"),
          Choice_Frame = frame
        ) %>%
        filter(!is.na(trial_index), !is.na(risky_choice)) %>%
        arrange(trial_index) %>%
        group_by(Participant, Feedback_Type, Time_Pressure, Choice_Frame) %>%
        mutate(Blocks = (row_number() - 1L) %/% 10L + 1L) %>%
        ungroup()

      avg_condition_data <- df %>%
        group_by(Participant, Time_Pressure, Choice_Frame, Feedback_Type, Blocks) %>%
        summarise(
          Average_Risky_Choice = mean(risky_choice, na.rm = TRUE),
          n_trials = n(),
          .groups = "drop"
        )

      interaction_data <<- bind_rows(interaction_data, avg_condition_data)
    }
  }
}

# Process partial and full feedback data
process_files(partial_pass_folder_path, "Partial")
process_files(full_pass_folder_path, "Full")

# Convert variables to factors for analysis
interaction_data$Blocks        <- as.factor(interaction_data$Blocks)
interaction_data$Feedback_Type <- as.factor(interaction_data$Feedback_Type)
interaction_data$Participant   <- as.factor(interaction_data$Participant)
interaction_data$Time_Pressure <- as.factor(interaction_data$Time_Pressure)
interaction_data$Choice_Frame  <- as.factor(interaction_data$Choice_Frame)
interaction_data$weights       <- interaction_data$n_trials

# View data
print(head(interaction_data))
```

```
##                                                Participant Time_Pressure
## 1 Mega_project_PARTICIPANT_SESSION_2025-11-24_10h18.02.511            No
## 2 Mega_project_PARTICIPANT_SESSION_2025-11-24_10h18.02.511            No
## 3 Mega_project_PARTICIPANT_SESSION_2025-11-24_10h18.02.511            No
## 4 Mega_project_PARTICIPANT_SESSION_2025-11-24_10h18.02.511            No
## 5 Mega_project_PARTICIPANT_SESSION_2025-11-24_10h18.02.511            No
## 6 Mega_project_PARTICIPANT_SESSION_2025-11-24_10h18.02.511            No
##   Choice_Frame Feedback_Type Blocks Average_Risky_Choice n_trials weights
## 1       Accept       Partial      1                  0.4       10      10
## 2       Accept       Partial      2                  0.3       10      10
## 3       Accept       Partial      3                  0.5       10      10
## 4       Accept       Partial      4                  0.5       10      10
## 5       Reject       Partial      1                  0.3       10      10
## 6       Reject       Partial      2                  0.5       10      10
```

``` r
cat("Rows:", nrow(interaction_data), " Participants:", n_distinct(interaction_data$Participant), "\n")
```

```
## Rows: 1936  Participants: 121
```


``` r
# Fit mixed effects model using afex::mixed
blocksmixed <- mixed(
  Average_Risky_Choice ~ Blocks * Time_Pressure * Choice_Frame * Feedback_Type + (1 | Participant),
  family = binomial(),
  data = interaction_data,
  method = "LRT",
  progress = TRUE,
  control = glmerControl(optimizer = "bobyqa", optCtrl = list(maxfun = 100000)),
  weights = weights
)
```

```
## Contrasts set to contr.sum for the following variables: Blocks, Time_Pressure, Choice_Frame, Feedback_Type, Participant
```

```
## Fitting 16 (g)lmer() models:
## [................]
```

``` r
# Summary + ANOVA table
print(summary(blocksmixed))
```

```
## Generalized linear mixed model fit by maximum likelihood (Laplace
##   Approximation) [glmerMod]
##  Family: binomial  ( logit )
## Formula: Average_Risky_Choice ~ Blocks * Time_Pressure * Choice_Frame *  
##     Feedback_Type + (1 | Participant)
##    Data: data
## Weights: weights
## Control: glmerControl(optimizer = "bobyqa", optCtrl = list(maxfun = 1e+05))
## 
##       AIC       BIC    logLik -2*log(L)  df.resid 
##    8013.2    8197.0   -3973.6    7947.2      1903 
## 
## Scaled residuals: 
##     Min      1Q  Median      3Q     Max 
## -4.7900 -0.6748 -0.0044  0.6589  3.6597 
## 
## Random effects:
##  Groups      Name        Variance Std.Dev.
##  Participant (Intercept) 0.07009  0.2648  
## Number of obs: 1936, groups:  Participant, 121
## 
## Fixed effects:
##                                                      Estimate Std. Error
## (Intercept)                                         -0.177376   0.028289
## Blocks1                                              0.058392   0.025399
## Blocks2                                             -0.006395   0.025491
## Blocks3                                              0.034521   0.025501
## Time_Pressure1                                      -0.017641   0.014728
## Choice_Frame1                                       -0.066442   0.014730
## Feedback_Type1                                       0.223672   0.028292
## Blocks1:Time_Pressure1                               0.038599   0.025399
## Blocks2:Time_Pressure1                              -0.015298   0.025491
## Blocks3:Time_Pressure1                              -0.007032   0.025501
## Blocks1:Choice_Frame1                               -0.005428   0.025398
## Blocks2:Choice_Frame1                                0.007732   0.025491
## Blocks3:Choice_Frame1                                0.011231   0.025501
## Time_Pressure1:Choice_Frame1                         0.035465   0.014728
## Blocks1:Feedback_Type1                              -0.077851   0.025399
## Blocks2:Feedback_Type1                              -0.005302   0.025491
## Blocks3:Feedback_Type1                               0.030087   0.025501
## Time_Pressure1:Feedback_Type1                        0.018435   0.014728
## Choice_Frame1:Feedback_Type1                         0.077347   0.014730
## Blocks1:Time_Pressure1:Choice_Frame1                -0.012113   0.025399
## Blocks2:Time_Pressure1:Choice_Frame1                 0.004897   0.025491
## Blocks3:Time_Pressure1:Choice_Frame1                 0.027791   0.025501
## Blocks1:Time_Pressure1:Feedback_Type1               -0.012954   0.025399
## Blocks2:Time_Pressure1:Feedback_Type1                0.002067   0.025491
## Blocks3:Time_Pressure1:Feedback_Type1                0.023416   0.025501
## Blocks1:Choice_Frame1:Feedback_Type1                 0.014743   0.025398
## Blocks2:Choice_Frame1:Feedback_Type1                -0.009310   0.025491
## Blocks3:Choice_Frame1:Feedback_Type1                -0.008073   0.025501
## Time_Pressure1:Choice_Frame1:Feedback_Type1         -0.036994   0.014728
## Blocks1:Time_Pressure1:Choice_Frame1:Feedback_Type1  0.033864   0.025399
## Blocks2:Time_Pressure1:Choice_Frame1:Feedback_Type1 -0.003369   0.025491
## Blocks3:Time_Pressure1:Choice_Frame1:Feedback_Type1  0.006492   0.025501
##                                                     z value Pr(>|z|)    
## (Intercept)                                          -6.270 3.61e-10 ***
## Blocks1                                               2.299  0.02151 *  
## Blocks2                                              -0.251  0.80190    
## Blocks3                                               1.354  0.17584    
## Time_Pressure1                                       -1.198  0.23100    
## Choice_Frame1                                        -4.511 6.46e-06 ***
## Feedback_Type1                                        7.906 2.66e-15 ***
## Blocks1:Time_Pressure1                                1.520  0.12858    
## Blocks2:Time_Pressure1                               -0.600  0.54841    
## Blocks3:Time_Pressure1                               -0.276  0.78273    
## Blocks1:Choice_Frame1                                -0.214  0.83078    
## Blocks2:Choice_Frame1                                 0.303  0.76163    
## Blocks3:Choice_Frame1                                 0.440  0.65963    
## Time_Pressure1:Choice_Frame1                          2.408  0.01604 *  
## Blocks1:Feedback_Type1                               -3.065  0.00218 ** 
## Blocks2:Feedback_Type1                               -0.208  0.83523    
## Blocks3:Feedback_Type1                                1.180  0.23808    
## Time_Pressure1:Feedback_Type1                         1.252  0.21069    
## Choice_Frame1:Feedback_Type1                          5.251 1.51e-07 ***
## Blocks1:Time_Pressure1:Choice_Frame1                 -0.477  0.63341    
## Blocks2:Time_Pressure1:Choice_Frame1                  0.192  0.84766    
## Blocks3:Time_Pressure1:Choice_Frame1                  1.090  0.27580    
## Blocks1:Time_Pressure1:Feedback_Type1                -0.510  0.61002    
## Blocks2:Time_Pressure1:Feedback_Type1                 0.081  0.93538    
## Blocks3:Time_Pressure1:Feedback_Type1                 0.918  0.35851    
## Blocks1:Choice_Frame1:Feedback_Type1                  0.580  0.56159    
## Blocks2:Choice_Frame1:Feedback_Type1                 -0.365  0.71496    
## Blocks3:Choice_Frame1:Feedback_Type1                 -0.317  0.75157    
## Time_Pressure1:Choice_Frame1:Feedback_Type1          -2.512  0.01201 *  
## Blocks1:Time_Pressure1:Choice_Frame1:Feedback_Type1   1.333  0.18243    
## Blocks2:Time_Pressure1:Choice_Frame1:Feedback_Type1  -0.132  0.89484    
## Blocks3:Time_Pressure1:Choice_Frame1:Feedback_Type1   0.255  0.79904    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
```

```
## 
## Correlation matrix not shown by default, as p = 32 > 12.
## Use print(summary(blocksmixed), correlation=TRUE)  or
##     vcov(summary(blocksmixed))        if you need it
```

``` r
print(blocksmixed)
```

```
## Mixed Model Anova Table (Type 3 tests, LRT-method)
## 
## Model: Average_Risky_Choice ~ Blocks * Time_Pressure * Choice_Frame * 
## Model:     Feedback_Type + (1 | Participant)
## Data: interaction_data
## Df full model: 33
##                                             Effect df     Chisq p.value
## 1                                           Blocks  3  13.92 **    .003
## 2                                    Time_Pressure  1      1.43    .231
## 3                                     Choice_Frame  1 20.36 ***   <.001
## 4                                    Feedback_Type  1 50.54 ***   <.001
## 5                             Blocks:Time_Pressure  3      2.37    .500
## 6                              Blocks:Choice_Frame  3      0.46    .928
## 7                       Time_Pressure:Choice_Frame  1    5.80 *    .016
## 8                             Blocks:Feedback_Type  3   11.34 *    .010
## 9                      Time_Pressure:Feedback_Type  1      1.57    .211
## 10                      Choice_Frame:Feedback_Type  1 27.60 ***   <.001
## 11               Blocks:Time_Pressure:Choice_Frame  3      1.57    .666
## 12              Blocks:Time_Pressure:Feedback_Type  3      1.01    .799
## 13               Blocks:Choice_Frame:Feedback_Type  3      0.44    .932
## 14        Time_Pressure:Choice_Frame:Feedback_Type  1    6.31 *    .012
## 15 Blocks:Time_Pressure:Choice_Frame:Feedback_Type  3      2.95    .399
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '+' 0.1 ' ' 1
```


``` r
# Plot fixed-effect coefficients
sjPlot::plot_model(blocksmixed, type = "est")
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

aov_tab <- as.data.frame(blocksmixed$anova_table)
aov_tab$Effect <- rownames(aov_tab)

p_col <- if ("Pr(>Chisq)" %in% names(aov_tab)) "Pr(>Chisq)" else if ("p.value" %in% names(aov_tab)) "p.value" else "p.value"
x_col <- if ("Chisq" %in% names(aov_tab)) "Chisq" else "Chisq"
df_col <- if ("df" %in% names(aov_tab)) "df" else if ("Df" %in% names(aov_tab)) "Df" else NA_character_

aov_tab$p <- aov_tab[[p_col]]
aov_tab$X2 <- aov_tab[[x_col]]
aov_tab$stars <- vapply(aov_tab$p, p_to_stars, character(1))

sig_tab <- aov_tab %>% filter(!is.na(p) & p < .05)

display_cols <- c("Effect", if (!is.na(df_col)) df_col, "X2", "p", "stars")
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
##                                    Effect Df        X2            p stars
##                                    Blocks 30 13.917537 3.019575e-03    **
##                              Choice_Frame 32 20.360358 6.414493e-06   ***
##                             Feedback_Type 32 50.536135 1.169908e-12   ***
##                Time_Pressure:Choice_Frame 32  5.798297 1.604170e-02     *
##                      Blocks:Feedback_Type 30 11.339055 1.002689e-02     *
##                Choice_Frame:Feedback_Type 32 27.598475 1.492964e-07   ***
##  Time_Pressure:Choice_Frame:Feedback_Type 32  6.309341 1.201034e-02     *
```

``` r
sig_effects <- sig_tab$Effect
m_glmer <- blocksmixed$full_model

stars_for <- function(effect_name) {
  s <- sig_tab$stars[sig_tab$Effect == effect_name]
  if (length(s) == 0) "" else s[[1]]
}

plot_main_effect_binom <- function(effect_name) {
  emm <- emmeans(m_glmer, as.formula(paste0("~ ", effect_name)), type = "response")
  df  <- as.data.frame(emm) %>% mutate(Pct = prob * 100, SE_pct = SE * 100)
  names(df)[names(df) == effect_name] <- "X"

  ggplot(df, aes(x = X, y = Pct, fill = X)) +
    geom_col(width = 0.65) +
    geom_errorbar(aes(ymin = Pct - SE_pct, ymax = Pct + SE_pct), width = 0.2) +
    labs(
      title = paste0("Main effect: ", effect_name, " ", stars_for(effect_name)),
      x = effect_name,
      y = "Predicted % risky choice"
    ) +
    theme_minimal() +
    theme(legend.position = "none", axis.title = element_text(face = "bold"))
}

plot_two_way_binom <- function(effect_name) {
  parts <- unlist(strsplit(effect_name, ":", fixed = TRUE))
  A <- parts[1]; B <- parts[2]

  emm <- emmeans(m_glmer, as.formula(paste0("~ ", A, " * ", B)), type = "response")
  df  <- as.data.frame(emm) %>% mutate(Pct = prob * 100, SE_pct = SE * 100)

  ggplot(df, aes_string(x = A, y = "Pct", fill = B)) +
    geom_col(position = position_dodge(0.7), width = 0.6) +
    geom_errorbar(aes(ymin = Pct - SE_pct, ymax = Pct + SE_pct),
                  position = position_dodge(0.7), width = 0.2) +
    labs(
      title = paste0("Interaction: ", effect_name, " ", stars_for(effect_name)),
      x = A,
      y = "Predicted % risky choice",
      fill = B
    ) +
    theme_minimal() +
    theme(axis.title = element_text(face = "bold"), legend.position = "bottom")
}

plot_three_way_binom <- function(effect_name) {
  parts <- unlist(strsplit(effect_name, ":", fixed = TRUE))
  A <- parts[1]; B <- parts[2]; C <- parts[3]

  emm <- emmeans(m_glmer, as.formula(paste0("~ ", A, " * ", B, " * ", C)), type = "response")
  df  <- as.data.frame(emm) %>% mutate(Pct = prob * 100, SE_pct = SE * 100)

  ggplot(df, aes_string(x = A, y = "Pct", fill = B)) +
    geom_col(position = position_dodge(0.7), width = 0.6) +
    geom_errorbar(aes(ymin = Pct - SE_pct, ymax = Pct + SE_pct),
                  position = position_dodge(0.7), width = 0.2) +
    facet_wrap(as.formula(paste0("~", C))) +
    labs(
      title = paste0("Interaction: ", effect_name, " ", stars_for(effect_name)),
      x = A,
      y = "Predicted % risky choice",
      fill = B
    ) +
    theme_minimal() +
    theme(axis.title = element_text(face = "bold"),
          legend.position = "bottom",
          strip.text = element_text(face = "bold"))
}

plot_blocks_interaction_binom <- function(effect_name) {
  parts <- unlist(strsplit(effect_name, ":", fixed = TRUE))
  others <- setdiff(parts, "Blocks")

  fml <- if (length(others) == 0) {
    ~ Blocks
  } else {
    as.formula(paste0("~ Blocks | ", paste(others, collapse = " * ")))
  }

  emm <- emmeans(m_glmer, fml, type = "response")
  df  <- as.data.frame(emm) %>% mutate(Pct = prob * 100, SE_pct = SE * 100)

  if (length(others) == 1) {
    ggplot(df, aes(x = Blocks, y = Pct, color = .data[[others[1]]], group = .data[[others[1]]])) +
      geom_point(position = position_dodge(0.15), size = 2) +
      geom_line(position = position_dodge(0.15), linewidth = 1) +
      geom_errorbar(aes(ymin = Pct - SE_pct, ymax = Pct + SE_pct),
                    width = 0.15, position = position_dodge(0.15)) +
      labs(
        title = paste0("Interaction: ", effect_name, " ", stars_for(effect_name)),
        x = "Blocks",
        y = "Predicted % risky choice",
        color = others[1]
      ) +
      theme_minimal() +
      theme(axis.title = element_text(face = "bold"), legend.position = "bottom")
  } else if (length(others) == 2) {
    ggplot(df, aes(x = Blocks, y = Pct, color = .data[[others[1]]], group = .data[[others[1]]])) +
      geom_point(position = position_dodge(0.15), size = 2) +
      geom_line(position = position_dodge(0.15), linewidth = 1) +
      geom_errorbar(aes(ymin = Pct - SE_pct, ymax = Pct + SE_pct),
                    width = 0.15, position = position_dodge(0.15)) +
      facet_wrap(as.formula(paste0("~", others[2]))) +
      labs(
        title = paste0("Interaction: ", effect_name, " ", stars_for(effect_name)),
        x = "Blocks",
        y = "Predicted % risky choice",
        color = others[1]
      ) +
      theme_minimal() +
      theme(axis.title = element_text(face = "bold"),
            legend.position = "bottom",
            strip.text = element_text(face = "bold"))
  } else {
    ggplot(df, aes(x = Blocks, y = Pct, group = 1)) +
      geom_point(size = 2) +
      geom_line(linewidth = 1) +
      geom_errorbar(aes(ymin = Pct - SE_pct, ymax = Pct + SE_pct), width = 0.15) +
      labs(
        title = paste0("Interaction: ", effect_name, " ", stars_for(effect_name)),
        x = "Blocks",
        y = "Predicted % risky choice"
      ) +
      theme_minimal() +
      theme(axis.title = element_text(face = "bold"))
  }
}

plots <- list()
for (eff in sig_effects) {
  n_way <- length(unlist(strsplit(eff, ":", fixed = TRUE)))
  has_blocks <- grepl("Blocks", eff)

  if (n_way == 1) {
    plots[[eff]] <- plot_main_effect_binom(eff)
  } else if (has_blocks) {
    plots[[eff]] <- plot_blocks_interaction_binom(eff)
  } else if (n_way == 2) {
    plots[[eff]] <- plot_two_way_binom(eff)
  } else if (n_way == 3) {
    plots[[eff]] <- plot_three_way_binom(eff)
  }
}
```

```
## NOTE: Results may be misleading due to involvement in interactions
## NOTE: Results may be misleading due to involvement in interactions
## NOTE: Results may be misleading due to involvement in interactions
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
## NOTE: Results may be misleading due to involvement in interactions
## NOTE: Results may be misleading due to involvement in interactions
## NOTE: Results may be misleading due to involvement in interactions
```

``` r
for (eff in sig_tab$Effect) {
  if (!is.null(plots[[eff]])) print(plots[[eff]])
}
```

![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-1.png)![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-2.png)![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-3.png)![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-4.png)![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-5.png)![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-6.png)![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-7.png)

``` r
cat("\nReporting-ready lines (Type III LRT):\n")
```

```
## 
## Reporting-ready lines (Type III LRT):
```

``` r
for (i in seq_len(nrow(sig_tab))) {
  df_text <- if (!is.na(df_col)) as.character(sig_tab[[df_col]][i]) else "NA"
  cat(sprintf("%s: ChiSq(%s) = %.2f, p = %.3g %s\n",
              sig_tab$Effect[i], df_text, sig_tab$X2[i], sig_tab$p[i], sig_tab$stars[i]))
}
```

```
## Blocks: ChiSq(30) = 13.92, p = 0.00302 **
## Choice_Frame: ChiSq(32) = 20.36, p = 6.41e-06 ***
## Feedback_Type: ChiSq(32) = 50.54, p = 1.17e-12 ***
## Time_Pressure:Choice_Frame: ChiSq(32) = 5.80, p = 0.016 *
## Blocks:Feedback_Type: ChiSq(30) = 11.34, p = 0.01 *
## Choice_Frame:Feedback_Type: ChiSq(32) = 27.60, p = 1.49e-07 ***
## Time_Pressure:Choice_Frame:Feedback_Type: ChiSq(32) = 6.31, p = 0.012 *
```
