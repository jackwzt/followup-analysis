---
title: "Original Data Plots: Risk, Repetition, RT"
output: html_document
date: "2026-03-08"
---


``` r
library(dplyr)
library(ggplot2)
library(tools)

partial_pass_folder_path <- "C:/Users/Jack/OneDrive/Desktop/follow up/cleaned_no4060_batch/PartialPass4060"
full_pass_folder_path    <- "C:/Users/Jack/OneDrive/Desktop/follow up/cleaned_no4060_batch/Pass4060"

read_cleaned_folder <- function(folder_path, feedback_type) {
  files <- list.files(folder_path, pattern = "_cleaned\\.csv$", full.names = TRUE)
  if (length(files) == 0) return(data.frame())

  bind_rows(lapply(files, function(f) {
    d <- read.csv(f, stringsAsFactors = FALSE)
    d$Participant <- file_path_sans_ext(basename(f))
    d$Participant <- sub("_cleaned$", "", d$Participant)
    d$Feedback_Type <- feedback_type
    d
  }))
}

all_trials <- bind_rows(
  read_cleaned_folder(partial_pass_folder_path, "Partial"),
  read_cleaned_folder(full_pass_folder_path, "Full")
)

all_trials <- all_trials %>%
  mutate(
    trial_index = suppressWarnings(as.numeric(trial_index)),
    rt_final = suppressWarnings(as.numeric(rt_final)),
    risky_choice = suppressWarnings(as.numeric(risky_choice)),
    Time_Pressure = ifelse(grepl("_TP", task_name, ignore.case = TRUE), "Yes", "No"),
    Choice_Frame = frame,
    Feedback_Type = factor(Feedback_Type, levels = c("Full", "Partial")),
    Time_Pressure = factor(Time_Pressure, levels = c("No", "Yes")),
    Choice_Frame = factor(Choice_Frame, levels = c("Accept", "Reject")),
    Participant = factor(Participant)
  ) %>%
  arrange(Participant, Feedback_Type, Time_Pressure, Choice_Frame, trial_index) %>%
  group_by(Participant, Feedback_Type, Time_Pressure, Choice_Frame) %>%
  mutate(Blocks = (row_number() - 1L) %/% 10L + 1L) %>%
  ungroup()

cat("Rows:", nrow(all_trials), " Participants:", n_distinct(all_trials$Participant), "\n")
```

```
## Rows: 19360  Participants: 121
```

## Risk Preference (Original Data)


``` r
# 1) ?? participant-level ? block ??
risk_participant <- all_trials %>%
  filter(!is.na(risky_choice)) %>%
  group_by(Participant, Feedback_Type, Choice_Frame, Time_Pressure, Blocks) %>%
  summarise(
    Average_Risky_Choice = mean(risky_choice, na.rm = TRUE),
    .groups = "drop"
  )

# 2) ?????????? SE
risk_plot_df <- risk_participant %>%
  group_by(Feedback_Type, Choice_Frame, Time_Pressure, Blocks) %>%
  summarise(
    N = sum(!is.na(Average_Risky_Choice)),
    MeanRisky = mean(Average_Risky_Choice, na.rm = TRUE),
    SE = sd(Average_Risky_Choice, na.rm = TRUE) / sqrt(N),
    .groups = "drop"
  ) %>%
  mutate(
    RiskyPct = MeanRisky * 100,
    SE_pct = SE * 100,
    Blocks = factor(Blocks, levels = sort(unique(Blocks)))
  )

# 3) ????????????
ggplot(risk_plot_df, aes(x = Blocks, y = RiskyPct, color = Time_Pressure, group = Time_Pressure)) +
  geom_point(position = position_dodge(0.2), size = 2) +
  geom_line(position = position_dodge(0.2), linewidth = 1) +
  geom_errorbar(aes(ymin = RiskyPct - SE_pct, ymax = RiskyPct + SE_pct),
                width = 0.2, position = position_dodge(0.2)) +
  facet_grid(Choice_Frame ~ Feedback_Type, scales = "fixed") +
  theme_minimal() +
  labs(
    title = "Observed Mean Risky Choice (Percentage)",
    x = "Block Number",
    y = "Average % Risky Choice",
    color = "Time Pressure"
  ) +
  scale_color_manual(values = c("No" = "#00BFC4", "Yes" = "#F8766D")) +
  theme(
    legend.position = "bottom",
    strip.text = element_text(size = 12, face = "bold"),
    axis.text = element_text(size = 10),
    axis.title = element_text(size = 12, face = "bold")
  ) +
  scale_y_continuous(limits = c(30, 75), breaks = seq(30, 75, 10))
```

![plot of chunk risk_plot](figure/risk_plot-1.png)

## Repetition (Original Data)


``` r
# 1) trial-level ?? repetition??? trial ???
rep_trial <- all_trials %>%
  mutate(final_choice = tolower(trimws(as.character(final_choice)))) %>%
  filter(final_choice %in% c("left", "right")) %>%
  arrange(Participant, Feedback_Type, Choice_Frame, Time_Pressure, trial_index) %>%
  group_by(Participant, Feedback_Type, Choice_Frame, Time_Pressure) %>%
  mutate(
    next_choice = lead(final_choice),
    comparison_block = lead(Blocks),
    is_repeated = as.integer(final_choice == next_choice)
  ) %>%
  ungroup() %>%
  filter(!is.na(next_choice), !is.na(comparison_block))

# 2) participant-level repetition rate per block
rep_participant <- rep_trial %>%
  group_by(Participant, Feedback_Type, Choice_Frame, Time_Pressure, comparison_block) %>%
  summarise(
    Repetition_Rate = mean(is_repeated, na.rm = TRUE),
    .groups = "drop"
  ) %>%
  rename(Blocks = comparison_block)

# 3) ??????? SE
rep_plot_df <- rep_participant %>%
  group_by(Feedback_Type, Choice_Frame, Time_Pressure, Blocks) %>%
  summarise(
    N = sum(!is.na(Repetition_Rate)),
    MeanRep = mean(Repetition_Rate, na.rm = TRUE),
    SE = sd(Repetition_Rate, na.rm = TRUE) / sqrt(N),
    .groups = "drop"
  ) %>%
  mutate(
    RepPct = MeanRep * 100,
    SE_pct = SE * 100,
    Blocks = factor(Blocks, levels = sort(unique(Blocks)))
  )

# 4) ??
ggplot(rep_plot_df, aes(x = Blocks, y = RepPct, color = Time_Pressure, group = Time_Pressure)) +
  geom_point(position = position_dodge(0.2), size = 2) +
  geom_line(position = position_dodge(0.2), linewidth = 1) +
  geom_errorbar(aes(ymin = RepPct - SE_pct, ymax = RepPct + SE_pct),
                width = 0.2, position = position_dodge(0.2)) +
  facet_grid(Choice_Frame ~ Feedback_Type, scales = "fixed") +
  theme_minimal() +
  labs(
    title = "Observed Mean Repetition Rate (Percentage)",
    x = "Block Number",
    y = "Average % Repetition",
    color = "Time Pressure"
  ) +
  scale_color_manual(values = c("No" = "#00BFC4", "Yes" = "#F8766D")) +
  theme(
    legend.position = "bottom",
    strip.text = element_text(size = 12, face = "bold"),
    axis.text = element_text(size = 10),
    axis.title = element_text(size = 12, face = "bold")
  ) +
  scale_y_continuous(limits = c(30, 75), breaks = seq(30, 75, 10))
```

![plot of chunk repetition_plot](figure/repetition_plot-1.png)

## RT (Original Data)


``` r
# 1) participant-level mean RT per block
rt_participant <- all_trials %>%
  filter(is.finite(rt_final), rt_final > 0) %>%
  group_by(Participant, Feedback_Type, Choice_Frame, Time_Pressure, Blocks) %>%
  summarise(
    MeanRT = mean(rt_final, na.rm = TRUE),
    .groups = "drop"
  )

# 2) ??????? SE
rt_plot_df <- rt_participant %>%
  group_by(Feedback_Type, Choice_Frame, Time_Pressure, Blocks) %>%
  summarise(
    N = sum(!is.na(MeanRT)),
    Mean_RT = mean(MeanRT, na.rm = TRUE),
    SE_RT = sd(MeanRT, na.rm = TRUE) / sqrt(N),
    .groups = "drop"
  ) %>%
  mutate(
    Blocks = factor(Blocks, levels = sort(unique(Blocks)))
  )

# 3) ??
ggplot(rt_plot_df, aes(x = Blocks, y = Mean_RT, color = Time_Pressure, group = Time_Pressure)) +
  geom_point(position = position_dodge(0.2), size = 2) +
  geom_line(position = position_dodge(0.2), linewidth = 1) +
  geom_errorbar(aes(ymin = Mean_RT - SE_RT, ymax = Mean_RT + SE_RT),
                width = 0.2, position = position_dodge(0.2)) +
  facet_grid(Choice_Frame ~ Feedback_Type, scales = "fixed") +
  theme_minimal() +
  labs(
    title = "Observed Mean Reaction Time (ms)",
    x = "Block Number",
    y = "Average RT (ms)",
    color = "Time Pressure"
  ) +
  scale_color_manual(values = c("No" = "#00BFC4", "Yes" = "#F8766D")) +
  theme(
    legend.position = "bottom",
    strip.text = element_text(size = 12, face = "bold"),
    axis.text = element_text(size = 10),
    axis.title = element_text(size = 12, face = "bold")
  ) +
  scale_y_continuous(limits = c(200, 1400), breaks = seq(200, 1400, 200))
```

![plot of chunk rt_plot](figure/rt_plot-1.png)
