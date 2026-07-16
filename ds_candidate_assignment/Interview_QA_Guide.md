# Spendy Case Study — Interview Q&A Preparation Guide

---

## PART 1 — THE 20 MOST LIKELY INTERVIEW QUESTIONS

---

### Q1. Walk me through your overall approach to this problem.

**Ideal answer:**
"I treated this as a two-stage problem. First, I needed to define what loyalty actually means for Spendy — and that's harder than it sounds, because raw purchase counts would reward the wrong people. Once I had a rigorous definition, I translated it into a supervised learning problem: can I predict who will become loyal after just two months, so marketing can act early?

I started with EDA to find the structural constraints in the data — the skewed purchase distribution and the 75% single-shop concentration were the two findings that shaped every downstream decision. The loyalty definition came from those insights, not from a textbook. The ML model was chosen specifically because the loyalty label is constructed from hard thresholds, which means the feature-to-label relationship is non-linear by design."

**Why this answer works:** It demonstrates end-to-end thinking, shows that EDA was purposeful, and connects problem structure to solution choices.

---

### Q2. Why is predicting loyalty at month 2 specifically valuable?

**Ideal answer:**
"Timing is the core business insight here. A marketing intervention in month 2 — when a user is still forming habits around Spendy — is fundamentally different from an intervention at month 6. In month 2, the customer hasn't yet committed to a pattern. They might be using Spendy occasionally and could be nudged into habitual use with a well-timed incentive. By month 6, the pattern is largely set. You're either catching up to an outcome or reinforcing it.

There's also a cost argument. Early interventions are cheaper per conversion than re-acquisition campaigns. A spend cashback offer to a high-potential month-2 user is orders of magnitude cheaper than trying to re-engage a churned customer 12 months later."

---

### Q3. Why did you choose 4.6% as your loyalty rate? Isn't that too low?

**Ideal answer:**
"The 4.6% isn't a target — it's an outcome. I designed the definition to be conservative, and here's why: the primary use case I had in mind was a rewards programme — cashback, exclusive offers, early access. When real monetary value is attached to a label, the cost of a false positive is real. A rewards programme covering 30% of users isn't a loyalty programme — it's a blanket discount scheme, and the unit economics of the two are very different.

Every customer in that 4.6% has cleared four simultaneous hard requirements and a platform loyalty check. That label means something. If the business determines the thresholds are too strict — perhaps because the reward value is low — they can be relaxed. But that's a business decision, not a data science decision."

---

### Q4. What is the Brand Loyalist Problem and why does it matter?

**Ideal answer:**
"The Brand Loyalist Problem is the core business risk in BNPL: 75% of Spendy's users in this dataset bought from exactly one merchant. A user who buys frequently at a single shop might appear loyal on any frequency-based metric, but they may actually be loyal to that merchant — not to Spendy. If that merchant stops offering Spendy or switches to a competitor, Spendy loses the customer entirely.

This matters enormously for a loyalty programme. If you reward merchant-loyal users with cashback or exclusive access, you're spending money on users who have no particular affinity for Spendy as a platform. The ROI on that investment is much lower than investing in genuinely platform-loyal users, who will use Spendy across multiple merchants regardless of any individual store's decisions."

---

### Q5. How did you handle class imbalance?

**Ideal answer:**
"I used `class_weight='balanced'` in the Random Forest classifier, which scales the class weights inversely proportional to their frequencies. This effectively increases the penalty for misclassifying a loyal user during training — the model becomes more aggressive in identifying potential loyalists, improving recall at the cost of some precision.

The trade-off is intentional: for this business problem, missing a future loyal customer — a false negative — is more costly than occasionally flagging a user who doesn't become loyal — a false positive. The cost of sending a targeted campaign to a non-loyal user is small. The cost of missing a high-LTV user entirely is much larger.

I also used stratified train-test splitting, so the loyal-to-non-loyal ratio is preserved in both the training and test sets. This ensures the evaluation metrics reflect the true class distribution."

---

### Q6. Your ROC-AUC is 0.912 — what does that actually mean in business terms?

**Ideal answer:**
"ROC-AUC of 0.912 means that if I take one random loyal user and one random non-loyal user from the test set, the model will correctly rank the loyal user higher 91.2% of the time. In business terms: the model's probability scores are reliable for ranking and segmenting customers.

This matters because the business deliverable isn't a binary prediction — it's a ranked list. Marketing doesn't need to know exactly which users will be loyal; they need to know which users are more likely to be loyal than others, so they can prioritise their budget accordingly. An AUC of 0.912 means the ranking is very reliable.

I deliberately avoided using accuracy as the headline metric. A model that predicts 'not loyal' for everyone would score 96.8% accuracy — which is completely useless. Accuracy is a misleading metric whenever classes are highly imbalanced."

---

### Q7. Recall for the loyal class is only 0.45. Isn't that a big problem?

**Ideal answer:**
"It's a known limitation and the primary area for improvement. With 1,108 positive training examples out of 34,420 total — a 3.2% positive rate — the model is learning a signal from very few examples. Recall of 0.45 means we're catching 45% of future loyal users at the default threshold.

However, two things contextualise this. First, the High likelihood segment — users scoring above 66% — has an observed loyalty rate of 83.9%. The model's confident predictions are very accurate. Second, many of the missed loyal users likely fall into the Medium segment where campaigns can still drive conversion.

The fix is a combination of: (1) calibrating the classification threshold — lowering it from 0.5 increases recall at a precision cost, (2) trying Gradient Boosting models that often handle imbalance better, (3) adding more positive training examples as more loyal users accumulate over time."

---

### Q8. Why didn't you try other models like XGBoost or a neural network?

**Ideal answer:**
"Random Forest was chosen as the primary model because it matches the problem structure — non-linear thresholds, mixed-scale features, correlated predictors — without requiring extensive tuning. It's also transparent enough that feature importances give a clear explanation to business stakeholders.

XGBoost or LightGBM would be the natural next step. Gradient Boosting typically outperforms Random Forest on imbalanced datasets because its sequential error correction focuses more aggressively on the misclassified minority class. I'd expect a 1–3% AUC improvement. The cost is more hyperparameters to tune and reduced interpretability.

Neural networks are inappropriate here. With 34K training examples and 5 features, a deep learning model would overfit and provide no interpretability advantage. Tree-based models consistently outperform deep learning on structured tabular data at this scale."

---

### Q9. How did you prevent data leakage?

**Ideal answer:**
"Data leakage was the central engineering concern in feature construction. The risk is that any feature using data from beyond day 60 per user would give the model information that isn't available at prediction time — the model would be 'cheating'.

I addressed this in three ways. First, all features are computed exclusively from transactions within the first 60 days of each user's history. Second, the labels are computed from the full dataset up to December 31 — completely separate from the feature window. Third, I restricted the training population to new users who joined before September 30, ensuring every training example has at least 3 months between the end of the feature window and the label evaluation date.

The feature importance output is a useful sanity check for leakage. Spend rate and frequency rate dominate at 42% and 28% respectively — exactly what we'd expect from the loyalty definition. If an unexpected feature like a timestamp component had dominated, that would be a red flag."

---

### Q10. How would you put this model into production?

**Ideal answer:**
"The pipeline has three layers. Data ingestion: transactions flow into a data warehouse (BigQuery or S3). Feature pipeline: a scheduled job — triggered monthly by EventBridge — computes the 60-day feature window for every new user who crossed the day-60 mark since the last run. The model scores them, and probabilities plus segment labels are written to the CRM within the same day.

The loyalty definition itself runs as a separate monthly batch — computing the current loyal user count for the KPI dashboard, and generating ground truth labels that validate the model's predictions over time.

Model governance: quarterly retraining with a strict AUC gate before promoting to production. Score distribution monitoring using KS tests on each feature to detect data drift. And threshold governance — segment cutoffs and loyalty thresholds reviewed jointly with marketing and finance stakeholders each quarter."

---

## PART 2 — QUESTIONS ABOUT THE LOYALTY DEFINITION

---

### Q11. Why did you choose an AND/OR structure rather than a composite score?

**Ideal answer:**
"A composite score — like RFM — would allow a high value on one dimension to compensate for a low value on another. That's exactly what I wanted to avoid. A user who spends 50,000 JPY a month but bought from a single shop twice and hasn't purchased in four months should not be classified as loyal — but a composite score might give them a passing grade.

The four hard requirements are all necessary conditions for what I mean by loyalty: genuine, sustained, recent, platform-level engagement. None of them is optional. The OR condition is applied specifically to the platform loyalty component because there are two genuinely different ways to demonstrate platform affinity — breadth across merchants, or depth across months — and it would be unfair to exclude a consistently engaged single-shop user purely on diversity grounds."

---

### Q12. How would you respond if the business wanted to change the thresholds?

**Ideal answer:**
"I'd welcome it — the thresholds are heuristics, not ground truth. The right process is: define what a loyal customer is worth (expected LTV uplift), define the cost of a false positive (wasted reward spend), then back-solve to find the threshold combination that maximises expected programme ROI.

In practice I'd run an A/B experiment: deploy the current thresholds as the control, test a relaxed version in treatment, and measure actual loyalty rates, reward redemption rates, and LTV 6 months out. The data should drive threshold decisions, not intuition."

---

### Q13. You said the 3-month registration requirement is a 'business rule, not a data threshold' — what's the difference?

**Ideal answer:**
"A data threshold is set by looking at a distribution and finding a meaningful breakpoint — like the 2 purchases/month threshold, where there's a natural elbow in the frequency distribution. A business rule is set by logic regardless of what the distribution looks like.

The 3-month minimum is justified by the argument that you cannot assess sustained behaviour with less than 3 months of data. It doesn't matter what the distribution of registration tenures looks like — a December joiner with 1 month of Christmas spending has not demonstrated sustained platform loyalty by definition. No amount of high frequency or high spend in that single month can substitute for time."

---

### Q14. Could you validate the loyalty definition externally?

**Ideal answer:**
"Ideally yes. The strongest validation would be to compare the loyalty label against actual churn data: do users classified as loyal have significantly lower 12-month churn rates than non-loyal users? If LTV data exists, do loyal users generate significantly higher revenue per user over 12+ months?

Without external validation data, the definition can be partially validated through internal consistency: do the loyalty thresholds produce a stable loyalty rate over time (which they do — stabilising around 4.5–4.6% from September onwards), and do the feature importances in the ML model reflect the expected loyalty signals (which they do — spend and frequency dominate)."

---

## PART 3 — QUESTIONS ABOUT RANDOM FOREST

---

### Q15. Can you explain intuitively why Random Forest is better than Logistic Regression here?

**Ideal answer:**
"Think about how the loyalty label was constructed. A user is loyal if their frequency crosses a threshold AND their spend crosses a threshold AND they meet the recency gate. The label is fundamentally a series of if-then conditions — it's a set of decision rules.

Random Forest literally learns by building decision trees — it learns if-then rules from data. So the model's internal representation mirrors how the label was constructed. Logistic Regression, by contrast, learns a straight line through the feature space. It assumes that a one-unit increase in frequency always produces the same change in loyalty probability, regardless of spend level. That's not true for this label — the relationship is threshold-based, not smooth.

A simpler way to put it: Random Forest speaks the same language as the loyalty definition. Logistic Regression doesn't."

---

### Q16. What would happen to your model if you added more features?

**Ideal answer:**
"Adding more features could help significantly, particularly in two areas. First, category-level data — knowing what types of merchants a user shops at would improve the merchant diversity signal. A user shopping at fashion and electronics is very different from one buying at two shops in the same category. Second, temporal behavioural features — regularity of purchase timing, day-of-week patterns, or response to promotions.

The risk with more features in Random Forest is increased training time and potential noise from uninformative features. I'd address this with feature importance screening and potentially a feature selection step before training. The model would also need re-evaluation for data leakage with each new feature — are we introducing anything from beyond the 60-day window?"

---

## PART 4 — QUESTIONS ABOUT FEATURE ENGINEERING

---

### Q17. Why use a 60-day feature window instead of 30 or 90 days?

**Ideal answer:**
"60 days — two months — is the natural prediction horizon for this problem. Marketing needs to intervene early enough to shape behaviour, but late enough to have meaningful signal. A 30-day window would give very little data per user, especially for users who purchase infrequently in their first month. A 90-day window would delay the intervention by an extra month, reducing the lead time for campaigns.

The 60-day window also aligns with the loyalty label structure. Frequency and spend are computed per month — so 60 days gives exactly two months of data, which allows for both a level estimate and a trend estimate (spend_month_comparison). A 30-day window would give only a level estimate with no trend."

---

### Q18. Why can't you include the 'sustained activity' signal in the ML features?

**Ideal answer:**
"Sustained activity — being active in 6 of 7 months — is a 7-month phenomenon, but the feature window is only 60 days. You can't observe 7 months of activity in 2 months — it's a measurement horizon problem, not a feature engineering problem.

The spend_month_comparison feature partially compensates. If spend is growing from month 1 to month 2, that's a directional signal that points towards the kind of consistent, increasing engagement that defines sustained activity. But it's an imperfect proxy. This is why unique_shops has the lowest feature importance at 6% — in 60 days, many users who will eventually use multiple shops haven't yet done so. This is an inherent ceiling on the 60-day model, not a design flaw."

---

## PART 5 — QUESTIONS ABOUT BUSINESS IMPACT

---

### Q19. How would you measure whether this system actually works after deployment?

**Ideal answer:**
"The gold standard is a randomised controlled experiment. Split new users randomly: control group receives standard marketing, treatment group receives segment-differentiated campaigns based on model scores. After 6 months, measure:

1. Loyalty rate per segment in treatment vs control
2. Revenue per user per segment
3. Campaign ROI — cost per converted loyal user

If the treatment group shows significantly higher loyalty conversion in the High and Medium segments, the model is adding causal value. If not, the features aren't predictive enough for the campaign differentiation to matter.

Secondary monitoring: track score distribution over time. If the proportion of High-likelihood users changes significantly quarter-over-quarter, investigate whether it reflects genuine platform growth or model drift."

---

### Q20. If you could only keep one slide from this presentation to show a non-technical executive, which would you pick?

**Ideal answer:**
"Slide 7 — Business Segmentation. It answers the question executives actually care about: what do we do differently tomorrow because of this analysis?

The slide shows three distinct customer groups, each with a clear marketing playbook. The High segment has 83.9% observed loyalty — 4 in 5 are going to be loyal anyway, so reinforce rather than acquire. The Medium segment is the highest ROI opportunity — they're already interested. The Low segment is where to reallocate budget away from spray-and-pray marketing.

Everything else in the presentation — the loyalty definition, the model metrics, the feature engineering — is the justification for why these segments are trustworthy. But for an executive who has 5 minutes, slide 7 is the answer to 'so what?'"

---

## QUICK REFERENCE — CRITICAL NUMBERS TO MEMORISE

| Metric | Value |
|---|---|
| Total transactions | 759,425 |
| Total users | 136,830 |
| Unique shops | 711 |
| Dataset window | Jun–Dec 2020 (7 months) |
| New users (joined in window) | 72,841 |
| Users with single shop | 75% |
| Frequency threshold | 2 purchases/month → 18.4% qualify |
| Spend threshold | ¥10,000/month → 16.3% qualify |
| Shop diversity threshold | 2+ shops → 25.2% qualify |
| Sustained activity threshold | 6+ months → 8.0% qualify |
| Platform loyalty (either path) | 28.5% qualify |
| Overall loyalty rate | 4.6% (6,273 users) |
| Training users (ML) | 34,420 |
| Positive rate (ML) | 3.2% (1,108 loyal) |
| ROC-AUC | 0.912 |
| Recall (loyal class) | 0.45 |
| Precision (loyal class) | 0.34 |
| High segment users | 2,135 |
| High segment observed loyalty | 83.9% |
| High segment avg probability | 0.95 |
| Medium segment observed loyalty | 6.3% |
| Low segment users | 26,225 |
| Low segment observed loyalty | 1.0% |

---

## KEY PHRASES TO USE PROACTIVELY

- "The Brand Loyalist Problem is why a simple frequency definition fails for Spendy."
- "The 60-day window isn't arbitrary — it's the earliest point at which marketing can act."
- "Accuracy is a misleading metric here — 96.8% accuracy requires no model at all."
- "The OR condition isn't a compromise — it reflects two genuinely different expressions of platform loyalty."
- "The 4.6% loyalty rate is conservative by design. When rewards have real cost, false positives are expensive."
- "Feature importances are my leakage sanity check — if spend and frequency didn't dominate, I'd be worried."

---

*Prepared for Final Round Interview — Spendy Data Scientist Case Study*
