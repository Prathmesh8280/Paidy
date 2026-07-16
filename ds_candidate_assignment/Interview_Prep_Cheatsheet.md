# Spendy Interview — Technical Prep Cheatsheet

> Read once. Say answers out loud. Close the doc.

---

## The 5 Numbers to Know Cold

| Number | What it is |
|---|---|
| **4.6%** | Overall loyalty rate — 6,273 users |
| **0.912** | ROC-AUC |
| **83.9%** | Observed loyalty rate in the High segment |
| **3.2%** | Positive rate in training data (1,108 loyal users) |
| **75%** | Users who bought from only one shop — the Brand Loyalist Problem |

---

## PPT Final Slide Status

| Slide | Status | Quick fix |
|---|---|---|
| 1 — Title | Ready | — |
| 2 — Business Problem | Ready | — |
| 3 — Dataset | Ready | — |
| 4 — Loyalty Definition | Ready | Bold the AND / OR connectors, bump to 26pt |
| 5 — Feature Engineering | Ready | — |
| 6 — Model Selection | Ready | RF reason titles: check they read at 13pt |
| 7 — Model Performance | Ready | — |
| 8 — Segmentation | Ready | — |
| 9 — Business Impact | Ready | — |
| 10 — Limitations | Ready | Card titles: bump 1pt if cramped |
| 11 — Conclusion | Ready | — |
| A / B / C — Appendix | Ready (Q&A ref) | — |

---

## Interview Q&A — Technical Questions

---

### LOYALTY DEFINITION

---

**Q1. Why 2 purchases per month and not 1.5 or 3?**

There is a natural elbow in the frequency distribution between 1.5 and 2.0 — below that, occasional users cluster densely. At 2.0 we are above the noise. The exact threshold is a business calibration question; 2.0 is the data-driven starting point.

---

**Q2. The loyalty rate is 4.6% — did you make the definition too strict?**

No — it is conservative by design. The primary use case is a rewards programme, and when real money is attached to a label, false positives are expensive. A programme covering 30% of users is not a loyalty programme — it is a blanket discount. Every user in that 4.6% cleared four simultaneous hard requirements, so the label actually means something.

---

**Q3. Why is the OR condition on platform loyalty principled and not just inflating numbers?**

The two paths represent fundamentally different expressions of the same construct. Breadth shows platform affinity spatially — the user chose Spendy at a new merchant. Depth shows it temporally — the user returned every month. A customer active in 6 of 7 months demonstrates stronger platform commitment than one who visited 2 shops but was active for only 2 months. The combined OR rate is only 28.5% — it is not inflating anything.

---

**Q4. Why does the registration minimum (3 months) exist?**

It is a business rule, not a data threshold. Without it, a user who joins in December and Christmas-shops can clear the frequency and spend thresholds on one month of seasonal behaviour. I found 419 December joiners who would have been classified as loyal with at most 1 month of data — that is clearly wrong. Three months gives a minimum observation window to confirm sustained behaviour.

---

**Q5. Could you have defined loyalty differently?**

Yes — I considered three alternatives. Top-N% by spend is simple but does not solve the Brand Loyalist Problem. RFM scoring is established but equal weighting is not appropriate here, and a composite score still allows a merchant-loyal user to qualify. Engagement tiers based on raw purchase counts have the seasonal spike problem. The five-component AND/OR definition is the only approach that directly distinguishes platform loyalty from merchant dependency.

---

### FEATURE ENGINEERING

---

**Q6. Why exactly 60 days? Why not 30 or 90?**

60 days is the prediction horizon — the earliest point at which marketing can act. At 30 days there is too little signal; many users have not made a second purchase yet. At 90 days the intervention is too late. The 60-day window also lets me compute a month-over-month spend trend, which a 30-day window cannot provide.

---

**Q7. How did you prevent data leakage?**

Three layers. First, every feature is computed strictly from transactions within days 0 to 60 per user. Second, labels are evaluated at December 31 — the feature window and label window never overlap. Third, I restricted training to users who joined before September 30, so every example has at least 3 months between the feature window and the label date.

---

**Q8. You cannot capture the "6+ active months" signal in 60 days — does that hurt the model?**

Yes, and I flag it explicitly under Limitations. The `spend_month_comparison` feature partially compensates by capturing early momentum — a user whose spend is growing from month 1 to month 2 is on a trajectory consistent with sustained engagement. It is an imperfect proxy, which is why unique_shops has the lowest feature importance at 6% — the 60-day window is simply too short to observe cross-merchant behaviour for most users.

---

**Q9. Why these 5 features and not more?**

The dataset gives us only price, timestamp, user ID, and shop ID — so these 5 features represent the most informative signals extractable from 3 raw columns. Each one maps directly to a loyalty component. With richer data — categories, demographics, campaign interactions — the feature set would expand. The 5-feature model achieves 0.912 AUC, which suggests the signal is there even with limited inputs.

---

### MODEL

---

**Q10. Why Random Forest and not XGBoost or Logistic Regression?**

Logistic Regression assumes a linear relationship between features and the outcome — but the loyalty label is constructed from hard thresholds, which is a step function, not a line. Random Forest learns via decision splits that mirror this threshold logic naturally, requires no feature scaling, and handles correlated features through sub-sampling.

XGBoost is the obvious next step. Gradient boosting typically handles class imbalance better through sequential error correction, and I would expect a 1–3% AUC improvement. The trade-off is more hyperparameters to tune and reduced interpretability for stakeholders.

---

**Q11. Recall is 0.45 — you are missing more than half of loyal users. How do you defend that?**

With a 3.2% positive rate and only 1,108 loyal examples in training, the model is learning from a small signal. `class_weight='balanced'` already improves recall at the cost of precision. The practical fixes are: lower the classification threshold below 0.5, try SMOTE oversampling, or switch to Gradient Boosting. More importantly — missed loyal users do not disappear. Many land in the Medium segment where targeted campaigns can still drive conversion.

---

**Q12. Why is ROC-AUC the right metric here and not accuracy?**

Because accuracy is meaningless with a 96.8% negative class — a model predicting "not loyal" for everyone scores 96.8% and is completely useless. ROC-AUC measures ranking quality: does the model correctly rank a loyal user above a non-loyal one? At 0.912, it does this 91% of the time. That is what the segmentation needs — a reliable ranked list, not a binary split.

---

**Q13. How did you handle class imbalance?**

`class_weight='balanced'` in the Random Forest, which scales the misclassification penalty inversely to class frequency. I also used stratified train-test splitting to preserve the 3.2% positive rate in both sets. The trade-off is explicit: recall improves to 0.45 at the cost of precision dropping to 0.34, which is the right trade-off when missing a high-LTV user is more costly than a false positive.

---

**Q14. Did you tune hyperparameters?**

I used 100 estimators with default settings as a strong baseline. Given the 0.912 AUC, the baseline is solid. A grid search over `max_depth`, `min_samples_split`, and `n_estimators` is the natural next step, alongside comparing against Gradient Boosting models.

---

### BUSINESS & PRODUCTION

---

**Q15. How would you measure whether this system actually works after deployment?**

A/B experiment: randomly split new users into control (standard marketing) and treatment (segment-differentiated campaigns based on model score). After 6 months, compare loyalty rates and revenue per user between groups. The difference is the causal impact of the model. Secondary monitoring: track score distribution over time — drift of more than 5% from baseline triggers a model review.

---

**Q16. How would you set the 33% and 66% segment thresholds in production?**

These are prototype thresholds. In production, they should be set based on campaign economics: at what probability does the expected revenue uplift from a converted loyal user exceed the reward cost? That is a decision-theory calculation, not a data science one. The business and finance teams own the inputs; data science optimises the cutoffs given those inputs.

---

**Q17. What does the production pipeline look like?**

Monthly batch triggered by a scheduler (e.g. EventBridge). A feature pipeline in dbt or Apache Beam computes the 60-day feature window for every user who crossed day 60 since the last run. The model scores them, output goes to the CRM with segment labels. The loyalty definition runs as a separate monthly snapshot for the KPI dashboard. Quarterly retraining with an AUC ≥ 0.90 gate before promoting to production.

---

**Q18. What is the biggest risk of deploying this model?**

Model drift. If the user base composition shifts — for example, a new merchant category brings in a different type of user — the feature distributions change and the model's predictions degrade silently. Monitoring score distribution and observed loyalty rates per segment as ground truth (at 6 months post-scoring) is the safeguard.

---

**Q19. What would you do differently with more time?**

Three things. First, compare Gradient Boosting models — I expect a meaningful recall improvement. Second, bring in Spendy's actual LTV data to back-solve the loyalty thresholds from business value rather than data inspection. Third, run a proper A/B experiment to validate the segmentation strategy causally before committing marketing budget.

---

**Q20. If a senior stakeholder asks why only 4.6% of users are loyal — what do you say?**

"That 4.6% represents customers who have genuinely earned the label across four dimensions simultaneously. When we attach real rewards to a label, the cost of getting it wrong goes up. A higher number would mean a weaker definition — and a weaker rewards programme. If we want to broaden coverage, we can calibrate the thresholds once we have LTV data to justify it."

---

## Phrases to Use Proactively

- *"The Brand Loyalist Problem is why a simple frequency definition fails for Spendy."*
- *"The 60-day window is not arbitrary — it is the earliest point at which marketing can act."*
- *"Accuracy is a misleading metric here. 96.8% accuracy requires no model at all."*
- *"The OR condition reflects two genuinely different expressions of platform loyalty — not a compromise."*
- *"The 4.6% rate is conservative by design. When rewards have real cost, false positives are expensive."*
- *"Feature importances are my leakage sanity check — spend and frequency dominating is exactly what I expected."*

---

## Phrases to Avoid

- "I just used Random Forest because it usually works well" — always give the structural reason
- "The threshold felt right" — always reference the distribution elbow or business logic
- "I didn't have time to try other models" — say "RF was the right baseline; GBM is the obvious next step"
- "The model is pretty accurate" — never use accuracy as a metric without caveating

---

*Good luck — you have done the work. Now go explain it.*
