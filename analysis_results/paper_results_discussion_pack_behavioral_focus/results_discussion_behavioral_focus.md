# Results

## Behavioral Overview

The analyses focused on eight non-4060 follow-up bandit tasks. The behavioral summaries showed that risky choice was higher under Full feedback than Partial feedback (Full = 0.511; Partial = 0.405) and higher in Reject than Accept frames (Reject = 0.476; Accept = 0.448). Repetition also varied by condition: choices were repeated more often under time pressure than no time pressure (TP = 0.566; NoTP = 0.538) and more often under Partial than Full feedback (Partial = 0.609; Full = 0.504). These descriptive results motivated the computational analyses by showing that feedback, frame, and time pressure were behaviorally relevant, but they did not by themselves identify the underlying mechanism.

## Mixed-Effect Behavioral Validation

The mixed-effect analyses provided direct behavioral evidence that choice history was important. After controlling for task structure, previous risky choice significantly predicted current risky choice (estimate = 0.204, 95% CI [0.080, 0.328], p = 0.00125). This supports the interpretation that repetition or stickiness was not merely a computational-model artifact; it was visible in the behavioral data.

However, reliable non-stickiness effects also remained. The Full - Partial contrast in the relative-uncertainty slope was significant after accounting for previous choice (estimate = -8.084, 95% CI [-13.231, -2.936], p = 0.00209). Choice frame also predicted risky-choice bias (Accept - Reject estimate = -0.053, 95% CI [-0.091, -0.016], p = 0.00521). These results indicate that feedback and frame effects were not reducible to simple repetition, although their strongest expression was behavioral rather than a stable learning-rate shift.

Switching behavior further supported a feedback-dependent strategy account. Regret or loss after the previous trial increased switching (estimate = 0.428, 95% CI [0.378, 0.478], p = 1.51e-62). Within Full feedback, counterfactual regret predicted next-trial switching (estimate = 0.408, 95% CI [0.140, 0.677], p = 0.00289). In Partial feedback, early relative uncertainty predicted risky choice (estimate = 0.253, 95% CI [0.203, 0.303], p = 6.02e-23). Thus, after controlling for stickiness, the strongest remaining behavioral signals involved uncertainty and feedback-dependent outcome evaluation.

## Three Focus Models

The main model comparison focused on three interpretable models. The Original BMT / hierarchical softmax model served as the theory-linked baseline, Model B represented a dynamic Decay-RL model with stickiness, and the feedback-aware WSLS model represented a simple behavioral strategy account. In same-data weighted top-1 accuracy, Model B achieved 0.686, the feedback-aware WSLS model achieved 0.653, and the Original BMT benchmark achieved 0.642. Model B therefore improved prediction relative to the original benchmark, while WSLS remained useful as a simple behavioral explanation despite lower accuracy than Model B.

The BMT-family integrated parameter analysis showed that gamma was the clearest factor-sensitive parameter. The Time Pressure contrast for gamma was credibly positive (TP - NoTP = 0.141, 95% HDI [0.082, 0.200], HDI excluded zero), consistent with stronger perseveration under time pressure. The Feedback contrast for gamma also excluded zero (Full - Partial = -0.352, 95% HDI [-0.544, -0.156]), consistent with stronger stickiness under Partial than Full feedback in this coding. By comparison, alpha and beta showed feedback-related contrasts, but these learning/value-sensitivity effects were less consistent across model variants and should be interpreted more cautiously.

The feedback-aware WSLS model clarified the direction of the feedback effect in strategic terms. Win-stay probability was lower under Full than Partial feedback (Full = 0.600; Partial = 0.700; Full - Partial = -0.100, 95% HDI [-0.168, -0.030]). Lose-shift probability was higher under Full than Partial feedback (Full = 0.575; Partial = 0.474; Full - Partial = 0.100, 95% HDI [0.033, 0.166]). This pattern suggests that Partial feedback promoted repetition following favorable chosen outcomes, whereas Full feedback promoted switching when counterfactual information indicated that the unchosen option was better.

## Supplementary Predictive Validation

Neural-network and synthetic-Bayesian analyses were treated as supplementary predictive checks rather than central behavioral models. Under strict participant-level held-out validation, the GRU achieved 0.654 top-1 accuracy, the MLP achieved 0.626, and Model B trained on synthetic neural-network choices achieved 0.600. These results suggest that sequence models captured additional predictive history structure, but synthetic choices did not replace real participant data for fitting an interpretable Bayesian RL model.


# Discussion

The behavioral and computational results converge on a simple conclusion: the most robust structure in the data was choice-history dependence combined with feedback-dependent strategy adjustment. The results do not support a single-mechanism story in which feedback, time pressure, and choice frame all map cleanly onto one learning-rate parameter. Instead, participants' behavior was best understood as a combination of reinforcement learning, repetition, uncertainty sensitivity, and response bias.

The clearest computational signal was stickiness. Both the mixed-effect behavioral analyses and the BMT-family parameter contrasts indicated that recent choice history strongly shaped current choice. This is consistent with reinforcement-learning approaches in which choices are not determined only by expected value, but also by action tendencies, perseveration, or response propensities learned through experience (Erev & Roth, 1998; Sutton & Barto, 2018). In the present data, time pressure increased the gamma/stickiness parameter and descriptively increased repetition. This pattern suggests that time pressure made participants rely more on recent actions, consistent with behavioral speed-accuracy tradeoff accounts in which time constraints reduce the opportunity for slower comparison processes (Wickelgren, 1977).

Feedback changed behavior primarily by changing the information available for learning and switching. Full feedback allowed participants to compare chosen and unchosen outcomes, whereas Partial feedback only provided chosen-outcome information. This distinction is central in bandit tasks because feedback determines what can be learned from unchosen options and how uncertainty is resolved (Steyvers et al., 2009; Speekenbrink & Konstantinidis, 2015). The mixed-effect analyses showed that relative uncertainty and counterfactual regret remained predictive after controlling for previous choice, and the WSLS model showed higher lose-shift under Full feedback but higher win-stay under Partial feedback. Thus, Full feedback appears to support counterfactual switching, whereas Partial feedback supports stronger repetition after favorable chosen outcomes.

Choice frame was better explained as a risk-bias or response-bias effect than as a stable shift in learning. Classic framing work shows that equivalent choice problems can produce different risk preferences depending on how outcomes are described (Kahneman & Tversky, 1979; Tversky & Kahneman, 1981). In this dataset, frame effects appeared clearly in risky-choice bias in the mixed-effect analyses, but they were not consistently recovered as robust beta or gamma shifts in the computational models. This suggests that Accept/Reject framing may alter the mapping between task representation and risky response rather than fundamentally changing how participants update values.

The uncertainty results support a behavioral bandit interpretation rather than a purely accuracy-based model-comparison interpretation. Human bandit behavior often reflects a mixture of directed exploration, random exploration, and heuristic strategies rather than a single optimal policy (Gershman, 2018; Wilson et al., 2014). Here, uncertainty effects were most visible after controlling for previous choice and separating feedback conditions. This means that uncertainty-based behavior was present, but it was partly masked by strong repetition. Future models should therefore include both uncertainty-sensitive choice terms and explicit choice-history terms, rather than treating stickiness as a nuisance parameter only.

Model comparison was useful, but the highest-accuracy model should not automatically be treated as the best psychological explanation. Model B improved prediction over the Original BMT benchmark and retained interpretable parameters for learning, value sensitivity, decay, and stickiness. The WSLS model was less accurate but provided a transparent account of how feedback changed simple strategy use. This illustrates why model comparison should balance predictive performance with interpretability, especially in cognitive bandit tasks. Predictive criteria such as LOO/WAIC are valuable for model evaluation, but they do not by themselves identify the true psychological mechanism (Vehtari et al., 2017).

The main limitation is that several parameter-level effects remain model-dependent. Feedback effects on alpha and beta appeared in the BMT-family model, but their interpretation was less stable than the gamma/stickiness effects. In addition, some task-level parameter comparisons were descriptive rather than full posterior contrasts, so they should not be overinterpreted. A second limitation is that time pressure was analyzed behaviorally and computationally, but the present task was not optimized to isolate speed-accuracy mechanisms. Future follow-up studies should strengthen the time-pressure manipulation, include explicit manipulation checks, and estimate participant-level random slopes for repetition, uncertainty sensitivity, and frame bias.

Overall, the results suggest that the follow-up bandit task successfully captured meaningful behavioral structure. The strongest pattern was not a general increase or decrease in learning, but a shift in how participants combined recent choice history, feedback information, uncertainty, and frame-dependent response tendencies. A concise interpretation is that feedback shaped what participants could learn from outcomes, time pressure increased reliance on repeated actions, and choice frame shifted risky-choice bias.


# References

Erev, I., & Roth, A. E. (1998). Predicting how people play games: Reinforcement learning in experimental games with unique, mixed strategy equilibria. *American Economic Review, 88*(4), 848-881.

Gershman, S. J. (2018). Deconstructing the human algorithms for exploration. *Cognition, 173*, 34-42. https://doi.org/10.1016/j.cognition.2017.12.014

Kahneman, D., & Tversky, A. (1979). Prospect theory: An analysis of decision under risk. *Econometrica, 47*(2), 263-291. https://doi.org/10.2307/1914185

Rescorla, R. A., & Wagner, A. R. (1972). A theory of Pavlovian conditioning: Variations in the effectiveness of reinforcement and nonreinforcement. In A. H. Black & W. F. Prokasy (Eds.), *Classical conditioning II: Current research and theory* (pp. 64-99). Appleton-Century-Crofts.

Speekenbrink, M., & Konstantinidis, E. (2015). Uncertainty and exploration in a restless bandit problem. *Topics in Cognitive Science, 7*(2), 351-367. https://doi.org/10.1111/tops.12145

Steyvers, M., Lee, M. D., & Wagenmakers, E.-J. (2009). A Bayesian analysis of human decision-making on bandit problems. *Journal of Mathematical Psychology, 53*(3), 168-179. https://doi.org/10.1016/j.jmp.2008.11.002

Sutton, R. S., & Barto, A. G. (2018). *Reinforcement learning: An introduction* (2nd ed.). MIT Press.

Tversky, A., & Kahneman, D. (1981). The framing of decisions and the psychology of choice. *Science, 211*(4481), 453-458. https://doi.org/10.1126/science.7455683

Vehtari, A., Gelman, A., & Gabry, J. (2017). Practical Bayesian model evaluation using leave-one-out cross-validation and WAIC. *Statistics and Computing, 27*, 1413-1432. https://doi.org/10.1007/s11222-016-9696-4

Wickelgren, W. A. (1977). Speed-accuracy tradeoff and information processing dynamics. *Acta Psychologica, 41*(1), 67-85. https://doi.org/10.1016/0001-6918(77)90012-9

Wilson, R. C., Geana, A., White, J. M., Ludvig, E. A., & Cohen, J. D. (2014). Humans use directed and random exploration to solve the explore-exploit dilemma. *Journal of Experimental Psychology: General, 143*(6), 2074-2081. https://doi.org/10.1037/a0038199
