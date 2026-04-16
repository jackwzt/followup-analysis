---
title: "Repetition (Cleaned Data)"
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

process_files_rep <- function(folder_path, feedback_type) {
  files <- list.files(folder_path, pattern = "_cleaned\\.csv$", full.names = TRUE)

  for (file_path in files) {
    df <- read.csv(file_path, stringsAsFactors = FALSE)
    participant <- file_path_sans_ext(basename(file_path))
    participant <- sub("_cleaned$", "", participant)

    if (all(c("trial_index", "task_name", "frame", "final_choice") %in% colnames(df))) {
      tmp <- df %>%
        mutate(
          trial_index = suppressWarnings(as.numeric(trial_index)),
          final_choice = tolower(trimws(as.character(final_choice))),
          Participant = participant,
          Feedback_Type = feedback_type,
          Time_Pressure = ifelse(grepl("_TP", task_name, ignore.case = TRUE), "Yes", "No"),
          Choice_Frame = frame
        ) %>%
        filter(!is.na(trial_index), final_choice %in% c("left", "right")) %>%
        arrange(trial_index) %>%
        group_by(Participant, Feedback_Type, Time_Pressure, Choice_Frame) %>%
        mutate(
          Blocks = (row_number() - 1L) %/% 10L + 1L,
          next_choice = lead(final_choice),
          comparison_block = lead(Blocks),
          is_repeated = as.integer(final_choice == next_choice)
        ) %>%
        ungroup() %>%
        filter(!is.na(next_choice), !is.na(comparison_block)) %>%
        group_by(Participant, Feedback_Type, Time_Pressure, Choice_Frame, comparison_block) %>%
        summarise(
          Repetition_Rate = mean(is_repeated, na.rm = TRUE),
          .groups = "drop"
        ) %>%
        rename(Blocks = comparison_block)

      interaction_data <<- bind_rows(interaction_data, tmp)
    }
  }
}

process_files_rep(partial_pass_folder_path, "Partial")
process_files_rep(full_pass_folder_path, "Full")

interaction_data <- interaction_data %>%
  mutate(
    Participant = as.factor(Participant),
    Feedback_Type = factor(Feedback_Type, levels = c("Full", "Partial")),
    Time_Pressure = factor(Time_Pressure, levels = c("No", "Yes")),
    Choice_Frame = factor(Choice_Frame, levels = c("Accept", "Reject")),
    Blocks = as.factor(Blocks)
  )

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
##   Time_Pressure Choice_Frame Blocks Repetition_Rate
## 1            No       Accept      1       0.2222222
## 2            No       Accept      2       0.5000000
## 3            No       Accept      3       0.3000000
## 4            No       Accept      4       0.7000000
## 5            No       Reject      1       0.4444444
## 6            No       Reject      2       0.3000000
```

``` r
cat("Rows:", nrow(interaction_data), " Participants:", n_distinct(interaction_data$Participant), "\n")
```

```
## Rows: 1936  Participants: 121
```


``` r
repetition_mixed <- mixed(
  Repetition_Rate ~ Blocks * Time_Pressure * Choice_Frame * Feedback_Type + (1 | Participant),
  data = interaction_data,
  method = "KR",
  type = 3,
  progress = TRUE,
  control = lmerControl(optimizer = "bobyqa", optCtrl = list(maxfun = 100000))
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
print(summary(repetition_mixed))
```

```
## Linear mixed model fit by REML. t-tests use Satterthwaite's method [
## lmerModLmerTest]
## Formula: 
## Repetition_Rate ~ Blocks * Time_Pressure * Choice_Frame * Feedback_Type +  
##     (1 | Participant)
##    Data: data
## Control: lmerControl(optimizer = "bobyqa", optCtrl = list(maxfun = 1e+05))
## 
## REML criterion at convergence: -704.8
## 
## Scaled residuals: 
##     Min      1Q  Median      3Q     Max 
## -3.8741 -0.6193  0.0336  0.6361  3.3396 
## 
## Random effects:
##  Groups      Name        Variance Std.Dev.
##  Participant (Intercept) 0.01278  0.1130  
##  Residual                0.03178  0.1783  
## Number of obs: 1936, groups:  Participant, 121
## 
## Fixed effects:
##                                                       Estimate Std. Error
## (Intercept)                                          5.564e-01  1.108e-02
## Blocks1                                             -4.271e-02  7.037e-03
## Blocks2                                              2.076e-02  7.037e-03
## Blocks3                                              1.230e-02  7.037e-03
## Time_Pressure1                                      -1.392e-02  4.063e-03
## Choice_Frame1                                        9.595e-04  4.063e-03
## Feedback_Type1                                      -5.281e-02  1.108e-02
## Blocks1:Time_Pressure1                               8.867e-03  7.037e-03
## Blocks2:Time_Pressure1                              -1.074e-03  7.037e-03
## Blocks3:Time_Pressure1                               7.600e-04  7.037e-03
## Blocks1:Choice_Frame1                                1.174e-02  7.037e-03
## Blocks2:Choice_Frame1                               -4.830e-03  7.037e-03
## Blocks3:Choice_Frame1                               -8.909e-03  7.037e-03
## Time_Pressure1:Choice_Frame1                        -1.906e-03  4.063e-03
## Blocks1:Feedback_Type1                               2.034e-02  7.037e-03
## Blocks2:Feedback_Type1                               1.825e-03  7.037e-03
## Blocks3:Feedback_Type1                              -6.637e-03  7.037e-03
## Time_Pressure1:Feedback_Type1                       -3.935e-04  4.063e-03
## Choice_Frame1:Feedback_Type1                         1.220e-03  4.063e-03
## Blocks1:Time_Pressure1:Choice_Frame1                -7.095e-03  7.037e-03
## Blocks2:Time_Pressure1:Choice_Frame1                -5.474e-03  7.037e-03
## Blocks3:Time_Pressure1:Choice_Frame1                 7.239e-03  7.037e-03
## Blocks1:Time_Pressure1:Feedback_Type1                9.723e-03  7.037e-03
## Blocks2:Time_Pressure1:Feedback_Type1               -7.638e-04  7.037e-03
## Blocks3:Time_Pressure1:Feedback_Type1               -8.751e-03  7.037e-03
## Blocks1:Choice_Frame1:Feedback_Type1                 4.034e-03  7.037e-03
## Blocks2:Choice_Frame1:Feedback_Type1                -1.965e-03  7.037e-03
## Blocks3:Choice_Frame1:Feedback_Type1                 2.884e-03  7.037e-03
## Time_Pressure1:Choice_Frame1:Feedback_Type1         -1.150e-03  4.063e-03
## Blocks1:Time_Pressure1:Choice_Frame1:Feedback_Type1 -4.379e-03  7.037e-03
## Blocks2:Time_Pressure1:Choice_Frame1:Feedback_Type1  3.145e-03  7.037e-03
## Blocks3:Time_Pressure1:Choice_Frame1:Feedback_Type1 -1.107e-03  7.037e-03
##                                                             df t value Pr(>|t|)
## (Intercept)                                          1.190e+02  50.226  < 2e-16
## Blocks1                                              1.785e+03  -6.070 1.56e-09
## Blocks2                                              1.785e+03   2.950 0.003216
## Blocks3                                              1.785e+03   1.748 0.080669
## Time_Pressure1                                       1.785e+03  -3.427 0.000624
## Choice_Frame1                                        1.785e+03   0.236 0.813322
## Feedback_Type1                                       1.190e+02  -4.767 5.33e-06
## Blocks1:Time_Pressure1                               1.785e+03   1.260 0.207807
## Blocks2:Time_Pressure1                               1.785e+03  -0.153 0.878733
## Blocks3:Time_Pressure1                               1.785e+03   0.108 0.914009
## Blocks1:Choice_Frame1                                1.785e+03   1.668 0.095562
## Blocks2:Choice_Frame1                                1.785e+03  -0.686 0.492587
## Blocks3:Choice_Frame1                                1.785e+03  -1.266 0.205645
## Time_Pressure1:Choice_Frame1                         1.785e+03  -0.469 0.639028
## Blocks1:Feedback_Type1                               1.785e+03   2.891 0.003888
## Blocks2:Feedback_Type1                               1.785e+03   0.259 0.795428
## Blocks3:Feedback_Type1                               1.785e+03  -0.943 0.345736
## Time_Pressure1:Feedback_Type1                        1.785e+03  -0.097 0.922855
## Choice_Frame1:Feedback_Type1                         1.785e+03   0.300 0.764001
## Blocks1:Time_Pressure1:Choice_Frame1                 1.785e+03  -1.008 0.313465
## Blocks2:Time_Pressure1:Choice_Frame1                 1.785e+03  -0.778 0.436749
## Blocks3:Time_Pressure1:Choice_Frame1                 1.785e+03   1.029 0.303745
## Blocks1:Time_Pressure1:Feedback_Type1                1.785e+03   1.382 0.167246
## Blocks2:Time_Pressure1:Feedback_Type1                1.785e+03  -0.109 0.913579
## Blocks3:Time_Pressure1:Feedback_Type1                1.785e+03  -1.244 0.213793
## Blocks1:Choice_Frame1:Feedback_Type1                 1.785e+03   0.573 0.566523
## Blocks2:Choice_Frame1:Feedback_Type1                 1.785e+03  -0.279 0.780076
## Blocks3:Choice_Frame1:Feedback_Type1                 1.785e+03   0.410 0.681999
## Time_Pressure1:Choice_Frame1:Feedback_Type1          1.785e+03  -0.283 0.777249
## Blocks1:Time_Pressure1:Choice_Frame1:Feedback_Type1  1.785e+03  -0.622 0.533804
## Blocks2:Time_Pressure1:Choice_Frame1:Feedback_Type1  1.785e+03   0.447 0.655007
## Blocks3:Time_Pressure1:Choice_Frame1:Feedback_Type1  1.785e+03  -0.157 0.875058
##                                                        
## (Intercept)                                         ***
## Blocks1                                             ***
## Blocks2                                             ** 
## Blocks3                                             .  
## Time_Pressure1                                      ***
## Choice_Frame1                                          
## Feedback_Type1                                      ***
## Blocks1:Time_Pressure1                                 
## Blocks2:Time_Pressure1                                 
## Blocks3:Time_Pressure1                                 
## Blocks1:Choice_Frame1                               .  
## Blocks2:Choice_Frame1                                  
## Blocks3:Choice_Frame1                                  
## Time_Pressure1:Choice_Frame1                           
## Blocks1:Feedback_Type1                              ** 
## Blocks2:Feedback_Type1                                 
## Blocks3:Feedback_Type1                                 
## Time_Pressure1:Feedback_Type1                          
## Choice_Frame1:Feedback_Type1                           
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
## Blocks2:Time_Pressure1:Choice_Frame1:Feedback_Type1    
## Blocks3:Time_Pressure1:Choice_Frame1:Feedback_Type1    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
```

```
## 
## Correlation matrix not shown by default, as p = 32 > 12.
## Use print(summary(repetition_mixed), correlation=TRUE)  or
##     vcov(summary(repetition_mixed))        if you need it
```

``` r
print(repetition_mixed)
```

```
## Mixed Model Anova Table (Type 3 tests, KR-method)
## 
## Model: Repetition_Rate ~ Blocks * Time_Pressure * Choice_Frame * Feedback_Type + 
## Model:     (1 | Participant)
## Data: interaction_data
##                                             Effect      df         F p.value
## 1                                           Blocks 3, 1785 12.62 ***   <.001
## 2                                    Time_Pressure 1, 1785 11.74 ***   <.001
## 3                                     Choice_Frame 1, 1785      0.06    .813
## 4                                    Feedback_Type  1, 119 22.73 ***   <.001
## 5                             Blocks:Time_Pressure 3, 1785      0.78    .508
## 6                              Blocks:Choice_Frame 3, 1785      1.23    .296
## 7                       Time_Pressure:Choice_Frame 1, 1785      0.22    .639
## 8                             Blocks:Feedback_Type 3, 1785    3.55 *    .014
## 9                      Time_Pressure:Feedback_Type 1, 1785      0.01    .923
## 10                      Choice_Frame:Feedback_Type 1, 1785      0.09    .764
## 11               Blocks:Time_Pressure:Choice_Frame 3, 1785      0.81    .486
## 12              Blocks:Time_Pressure:Feedback_Type 3, 1785      0.87    .457
## 13               Blocks:Choice_Frame:Feedback_Type 3, 1785      0.27    .849
## 14        Time_Pressure:Choice_Frame:Feedback_Type 1, 1785      0.08    .777
## 15 Blocks:Time_Pressure:Choice_Frame:Feedback_Type 3, 1785      0.18    .910
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '+' 0.1 ' ' 1
```


``` r
sjPlot::plot_model(repetition_mixed, type = "est")
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

aov_tab <- as.data.frame(repetition_mixed$anova_table)
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
##                Effect num Df      stat            p stars
##                Blocks      3 12.621928 3.641418e-08   ***
##         Time_Pressure      1 11.743742 6.243288e-04   ***
##         Feedback_Type      1 22.728032 5.333439e-06   ***
##  Blocks:Feedback_Type      3  3.546335 1.403044e-02     *
```

``` r
sig_effects <- sig_tab$Effect
m_lmer <- repetition_mixed$full_model

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
      y = "Estimated marginal mean (Repetition Rate)"
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
      x = A, y = "Estimated marginal mean (Repetition Rate)", fill = B
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
      x = A, y = "Estimated marginal mean (Repetition Rate)", fill = B
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
        x = "Blocks", y = "Estimated marginal mean (Repetition Rate)", color = others[1]
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
        x = "Blocks", y = "Estimated marginal mean (Repetition Rate)", color = others[1]
      ) + theme_minimal() + theme(axis.title = element_text(face = "bold"), legend.position = "bottom")
  } else {
    ggplot(df, aes(x = Blocks, y = emmean, group = 1)) +
      geom_point(size = 2) + geom_line(linewidth = 1) +
      geom_errorbar(aes(ymin = emmean - SE, ymax = emmean + SE), width = 0.15) +
      labs(
        title = paste0("Interaction: ", effect_name, " ", stars_for(effect_name)),
        x = "Blocks", y = "Estimated marginal mean (Repetition Rate)"
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
## NOTE: Results may be misleading due to involvement in interactions
## NOTE: Results may be misleading due to involvement in interactions
## NOTE: Results may be misleading due to involvement in interactions
## NOTE: Results may be misleading due to involvement in interactions
```

``` r
for (eff in sig_tab$Effect) {
  if (!is.null(plots[[eff]])) print(plots[[eff]])
}
```

![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-1.png)![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-2.png)![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-3.png)![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-4.png)

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
## Blocks: stat(3) = 12.62, p = 3.64e-08 ***
## Time_Pressure: stat(1) = 11.74, p = 0.000624 ***
## Feedback_Type: stat(1) = 22.73, p = 5.33e-06 ***
## Blocks:Feedback_Type: stat(3) = 3.55, p = 0.014 *
```
