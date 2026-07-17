# Spendy Loyalty Analysis — Detailed Presentation Speaker Script

> **Audience:** Mixed (technical panel + business stakeholders)
> **Total time target:** 20–25 minutes (slides 1–11) + 5 min Q&A
> **Appendix A:** Production pipeline — reference only during Q&A
> **How to use this script:** Read it all once. Highlight what resonates. Cut what doesn't. The goal is to have more material than you need so you can speak naturally rather than rushing to fill time.

---

## SLIDE 1 — Title

*[Take a breath before you start. Stand, make eye contact with the room, pause for 2–3 seconds. Don't rush the opening.]*

"Thank you for the opportunity to present today. My name is Prathmesh, and I'm going to walk you through my analysis of customer loyalty for Spendy.

Before I jump in, let me tell you how I've structured the next 20 minutes.

I've split this into three parts that build on each other. The first part is understanding the data and the business problem — and I want to spend real time here because the problem turns out to be more nuanced than it first appears. The second part is defining what customer loyalty actually means for a BNPL platform — and I'll show you exactly how I got there, component by component, with the data to back each decision. The third part is the machine learning model — predicting which brand new customers are on a path to becoming loyal after just their first two months on the platform.

By the time we're done, you'll have three things: a rigorous, data-driven loyalty definition that solves a specific structural problem in the Spendy user base. A working predictive model with a ROC-AUC of 0.912. And three actionable customer segments with clear marketing playbooks for each.

Let's get into it."

*[Advance slide. Don't rush — let the room settle before moving on.]*

---

## SLIDE 2 — Business Problem

*[This is the most important slide in the deck. Everything that follows is justified by what you establish here. Spend 2.5–3 full minutes. Do not rush the Brand Loyalist Problem — it needs to land.]*

"Before I touched the data or ran a single line of code, I spent time understanding what Spendy actually is and what problem they're trying to solve.

**What is Spendy?**

Spendy is a Buy Now Pay Later platform — BNPL — operating across Japan. The business model is straightforward: customers use Spendy at checkout instead of a credit card, they pay in instalments, and Spendy makes money through two streams — merchant transaction fees on each purchase, and interest on consumer instalment repayments.

The dataset I worked with covers 759,000 transactions across 136,000 users and 711 shops, spanning June to December 2020 — seven months of data.

**Now — the challenge. And this is the core of the whole presentation.**

When I did my initial exploration of the data, one number jumped out at me immediately and it reframed how I thought about the entire problem.

**75% of Spendy's users bought from exactly one merchant.**

Three quarters of the user base. One shop. That's not a minor data quirk — that's a structural characteristic of the platform. And it creates a very specific problem that I'm calling the Brand Loyalist Problem.

Here's the issue. If I look at a user who makes 10 purchases at the same shop over six months, they look extremely loyal on any frequency-based metric. High purchase count. High spend. Regular behaviour. But here's the question I need to ask: are they loyal to *Spendy*, or are they loyal to that one shop that happens to offer Spendy as a payment option?

Because those are fundamentally different things. If the merchant stops accepting Spendy — maybe they switch to a competitor payment platform, maybe they go out of business, maybe they renegotiate their contract — that user disappears from Spendy's platform entirely. They weren't Spendy customers. They were customers of that shop who used Spendy incidentally.

A loyalty programme built on raw purchase counts rewards exactly these users — the single-shop users — and that is a misallocation of budget. You're spending rewards money on users who have no particular affinity for Spendy as a platform.

So the first design question I had to answer before building anything else was: how do I distinguish genuine platform loyalty from what is effectively just merchant loyalty?

**And the third part of this slide — why predict loyalty early?**

Once we know who is loyal and who isn't, we want to be able to predict that outcome as early as possible in a new customer's journey.

The logic is simple. By month two of a user's life on the platform, their habits are still forming. They haven't yet settled into a fixed pattern. A well-placed incentive at month two — a cashback offer, a merchant discovery prompt, an early access reward — can actively shape how that customer behaves going forward. It's a nudge at the moment when nudges are cheapest and most effective.

Contrast that with month twelve. By then the pattern is set. You're either reinforcing an outcome that's already happened, or you're trying to re-acquire someone who drifted away months ago. Both are expensive. Month two is not.

And not every new user has the same potential. Some will become loyal regardless of what marketing does. Some will never become loyal regardless of how much you spend. The model separates these groups at month two so that budget goes where it actually changes outcomes."

*[Pause. Give the room a moment to absorb the Brand Loyalist Problem before moving on. It's the conceptual anchor for the entire presentation.]*

*[Advance slide.]*

---

## SLIDE 3 — Exploratory Data Analysis

*[90 seconds to 2 minutes. The charts are on the slide — point to them specifically. This slide justifies the loyalty definition decisions you make on the next slide.]*

"Before defining loyalty, I did exploratory analysis to understand the shape of the data. Two findings came out of this that directly shaped every decision I made downstream.

**The chart on the left — purchase frequency per user.**

This is the distribution of total purchases per user across the seven-month window. I've clipped it at 30 on the x-axis to keep it readable, but the tail goes much further. The red dashed line is the median — just 2 purchases per user across the entire dataset. Fifty percent of users made two or fewer purchases total over seven months.

The distribution is extremely right-skewed. The vast majority of users are very low frequency. A small number of power users pull the mean up, but the median tells you the real story — most users on this platform are not frequent buyers.

This had an immediate practical implication for how I designed the loyalty definition. Raw purchase counts are useless — a user who joined in November and made 3 purchases looks worse than a user who joined in June and made 5, even though the November user is actually buying faster. I needed to normalise everything by tenure. Rates, not counts.

**The chart on the right — unique shops per user.**

This is the Brand Loyalist Problem visualised. 75% of users sit in the first bar — one shop only. The other 25% have used two or more merchants. That's it. That's the entire split.

This single chart is the reason the loyalty definition has a platform loyalty component at all. Without addressing this, any definition I built on frequency or spend alone would be rewarding single-shop users — merchant loyalists — not platform loyalists.

These two charts aren't decoration. They are the architectural inputs to everything that follows."

*[Advance slide.]*

---

## SLIDE 4 — Loyalty Definition

*[This is your most important technical slide. Walk through it slowly and clearly. 3–4 minutes. Don't rush the OR condition — it's the most nuanced part and will likely get questions.]*

"So — given those findings — how do I actually define a loyal customer?

I want to start by explaining what I decided *not* to do, because that choice shapes everything.

**I did not use a composite score.**

The textbook approach to customer loyalty scoring is RFM — Recency, Frequency, Monetary value. You score a customer on each dimension, add them up, and anyone above a threshold is 'loyal.' It's simple, it's well understood, and it has a critical flaw for this problem.

A composite score allows a high value on one dimension to compensate for a low value on another. Someone who spends enormous amounts of money — say ¥50,000 a month — but bought exclusively from one shop twice and hasn't been seen in four months would score well on spend, and a composite might classify them as loyal. That is exactly the wrong answer. That user is inactive and possibly merchant-dependent. They should not be in the loyal bucket.

I wanted a definition where every loyal customer has genuinely earned the label across *every* dimension simultaneously. So I used hard requirements — AND logic — not a composite.

Here's how it works.

**① Purchase Frequency — at least 2 purchases per month.**

I computed purchase rate per month for every user — total purchases divided by months registered. Then I looked at what percentage of users fall above each threshold level.

At 2 purchases per month, 18.4% of users qualify. Below that threshold — at 1.5 per month — you're still in the territory of users who buy roughly once a month. That's usage, not habit. At 2 per month, the user is making regular, repeated decisions to use Spendy. There's also a natural elbow in the distribution between 1.5 and 2.0 — the curve of 'users above threshold' flattens out in that range. That elbow is the data telling you where the meaningful break is.

**② Monthly Spend — at least ¥10,000 per month.**

Same approach. Median monthly spend across all users is about ¥3,000. The mean is around ¥6,000 because a small number of very high spenders pull it up. At ¥10,000 a month — 16.3% of users qualify — we're talking about the top quintile of spenders. Below ¥10,000, the spending levels are low enough that they could easily reflect occasional or incidental transactions rather than genuine, committed usage.

¥10,000 a month in Japan is roughly the cost of a meal out two or three times, or a few household purchases. It's a number that says: this person is actually using Spendy as part of their spending life, not just testing it out.

**③ Recency — last purchase within 2 months.**

Loyalty is, by definition, an ongoing relationship. A user who was active nine months ago and has since gone silent is a former customer — or at best, a lapsed one. They should not be receiving loyalty rewards.

I set the recency window at two months. That's strict enough to exclude genuinely inactive users — someone who stopped using Spendy in September should not be labelled loyal in December — but lenient enough to give a grace period for natural gaps. People travel. They have quiet months. Two months is the right balance.

60.3% of users have purchased within the last two months. This component is relatively permissive on its own — but it becomes a sharp filter when combined with the others.

**④ Minimum Registration — 3 months.**

This one is different from all the others. The previous three thresholds were derived by looking at distributions and finding where meaningful breaks exist. This one is a **business rule** — justified by logic, not by a percentile.

The argument is this: if a user joins Spendy in December and does all their Christmas shopping through the platform, they might buy 15 times in a single month and spend ¥30,000. On the frequency and spend components, they look like a perfect loyal customer. But they have one month of data. You cannot assess sustained behaviour on one month of Christmas shopping.

I ran the numbers to put a face on this problem. 17,594 users joined in December. Without the tenure gate, 419 of them would be classified as loyal — purely on the basis of seasonal spike behaviour. Three months is the minimum window to say with any confidence that what you're observing is a pattern rather than a one-off event.

So that's the four hard requirements. Every loyal customer must clear all four simultaneously.

**Then — the platform loyalty check. And this is where we directly address the Brand Loyalist Problem.**

I gave users two routes to demonstrate platform loyalty, and they only need to satisfy one.

**Path 1 — Shop Diversity: purchased from at least 2 different merchants.**

A user who makes a purchase at a second merchant has actively *chosen* to bring Spendy somewhere new. They didn't have to do that. They could have just kept buying from their existing shop. The fact that they tried Spendy at a new merchant is a deliberate signal that they are loyal to Spendy as a payment platform, not just loyal to a single store. 25.2% of users qualify via this path.

**Path 2 — Sustained Activity: active in at least 6 of 7 available months.**

But what about a user who genuinely loves Spendy, uses it every single month like clockwork, but happens to only shop at one merchant? Maybe that merchant is the only one in their area that offers Spendy. Or maybe they just have a very specific purchasing habit. This user is demonstrating platform loyalty through depth rather than breadth — consistent, sustained engagement over nearly the entire observation window. Excluding them purely on shop diversity grounds would be unfair and would misclassify genuinely loyal users. 8.0% of users qualify via this path.

The OR condition is principled, not a compromise. Both paths represent genuine platform affinity — just expressed differently. Combined, 28.5% of users clear at least one of the two paths.

**The final result — applying all five components together:**

6,273 users. 4.6% of the entire user base.

I want to address that number directly because it will raise eyebrows. 4.6% sounds low. But remember what these users have done — they've cleared four hard requirements simultaneously *and* demonstrated platform loyalty through one of two evidence paths. Every single one of them has genuinely earned the label. That's the point.

If Spendy attaches real value to this label — cashback, exclusive merchant access, early features — then the cost of a false positive is real and measurable. A loyalty programme covering 30% of users isn't a loyalty programme, it's a blanket discount scheme. The unit economics are completely different. 4.6% means the label is worth something. If the business wants to broaden it — maybe the reward value is low cost enough to justify it — the thresholds can be calibrated. But that's a business decision that should be made with LTV data, not a data science decision made by eyeballing a number."

*[Pause before advancing. This slide generates the most questions — be ready.]*

*[Advance slide.]*

---

## SLIDE 5 — Feature Engineering

*[2–3 minutes. Three things to land clearly: why 60 days, what the 5 features are, and how we prevented data leakage. The leakage section will be probed.]*

"The loyalty definition gives us a label — a binary outcome we can measure at December 31 for every eligible user. The question for the ML model is: can we predict that label using only information from the first 60 days of a user's life on the platform?

**Why 60 days specifically?**

This is the prediction horizon — the point at which marketing wants to act. Let me explain why 60 days is the right window and not 30 or 90.

At 30 days, the signal is too thin. Many users haven't even made a second purchase within their first month. You'd be trying to predict long-term loyalty from what is essentially a first impression. The model would have almost nothing to work with.

At 90 days, you've waited too long. By 90 days, the habits are already forming. The user's trajectory is becoming clearer on its own — but the window for a cheap, effective early intervention is closing. Marketing wants to act *before* the pattern sets, not after.

60 days — two calendar months — is the sweet spot. It's the earliest point at which a user has enough transaction history to meaningfully score. And it gives us two full months of data, which means we can compute both a level estimate (how much are they spending right now) and a *trend* (is their spending growing from month one to month two). That trend feature is something a 30-day window simply cannot provide.

**How we prevented data leakage.**

Data leakage is the central engineering concern here, and it's worth explaining clearly because it's a common failure point in exactly these kinds of predictive problems.

The risk is this: if any feature accidentally uses information from after day 60 per user, the model is effectively cheating at prediction time. It would perform brilliantly in training and testing — because it has access to future data — and then collapse in production because that data doesn't exist yet when you need to score new users.

I addressed leakage in three layers.

First — every single feature is computed exclusively from transactions within the first 60 days of each user's history. Not a single row of data from day 61 onward is included in the feature set.

Second — the loyalty labels are evaluated at December 31, 2020. The feature window and the label evaluation date never overlap. The features look at days 0 to 60 of a user's history. The label looks at the full dataset up to December 31. These two windows are completely separate.

Third — the training population is restricted to users who joined before September 30. This ensures that every training example has at least 90 days — three months — between the end of their feature window and the label evaluation date. A user who joined on October 1 would have their feature window end on November 30, leaving only one month until December 31. That's not enough time for the loyalty label to fully develop. We exclude them.

**The five features — and why each one was chosen.**

I designed exactly five features, and each one maps directly to a component of the loyalty definition. This was deliberate. The model should be learning the signals that the loyalty definition is built on. If the feature importances showed something completely different, that would be a red flag.

**Feature 1: freq_rate_2m — purchases per month in the first 60 days.**
Direct proxy for the frequency component. A user buying twice a month in their first 60 days is showing early evidence of the habit we're looking for.

**Feature 2: spend_rate_2m — monthly spend in the first 60 days.**
Direct proxy for the spend component. A user who is already at ¥10,000 a month in their first two months isn't experimenting — they're committing.

**Feature 3: unique_shops_2m — distinct merchants visited in the first 60 days.**
The early signal for platform loyalty. A user who has already visited two merchants by day 60 is demonstrating that their relationship is with Spendy, not just with one shop.

**Feature 4: days_since_last_purchase_in_window — how many days before the end of the 60-day window was the user's last purchase.**
This is the recency signal within the feature window itself. Computed as 60 minus the day number of the user's last purchase. A value of 0 means they purchased on day 60 — still active right up to the prediction date. A value of 60 means they made their only purchase on day 0 and never came back. It captures early churn risk before it becomes actual churn.

**Feature 5: spend_month_comparison — Month 2 spend divided by Month 1 spend.**
This is the momentum feature. Because we cannot observe 7 months of behaviour in 60 days, we use the trend between month one and month two as a proxy for sustained engagement. A user whose spend is growing month-over-month in their first two months is on a trajectory consistent with long-term loyalty. A user whose spend is falling is showing an early warning signal.

One technical note on this feature — the denominator is Month 1 spend plus 1. The plus 1 is a division-by-zero guard. Many users have zero spend in month one because all their purchases happened after day 30, or they only made one purchase right at the start. Without the plus 1, you'd get undefined values that would crash the model. Adding 1 to the denominator is negligible when month one spend is large — ¥20,000 becomes ¥20,001 — and it handles the zero case cleanly by simply returning the month two spend as the output."

*[Advance slide.]*

---

## SLIDE 6 — Model Selection

*[2 minutes. Be confident and specific — the RF choice will be challenged. Have the structural argument ready.]*

"I chose Random Forest as the model for this problem. I want to walk through exactly why, because I think the reasoning is important and it connects directly back to how the problem is set up.

**The core argument: the model architecture should match the problem structure.**

Think about what the loyalty label actually is. It's a set of if-then conditions.

A user is loyal *if* their purchase frequency is at or above 2 per month *AND if* their monthly spend is at or above ¥10,000 *AND if* they purchased within the last 2 months *AND if* they've been registered for at least 3 months *AND if* they've either visited 2 or more shops *OR* been active in 6 or more months.

That is literally a decision tree written in plain English. It's a hierarchy of conditions — threshold crossings — that determine an outcome.

Random Forest learns by building hundreds of decision trees from the data. Each tree is asking questions like: 'is this user's spend rate above some threshold?' 'Is their frequency above some threshold?' These are exactly the kinds of questions the loyalty definition asks. The model's internal logic naturally mirrors the label construction.

Logistic Regression, by contrast, learns a straight line through the feature space. It assumes that increasing spend by ¥1,000 always produces the same change in loyalty probability, regardless of what the user's frequency is. That's not true here. The relationship is threshold-based, not smooth. Below ¥10,000 a month, you're not loyal regardless of frequency. Above ¥10,000 and above 2 purchases per month, you might be. Logistic Regression simply cannot represent that structure.

Random Forest also has practical advantages for this specific dataset. The features are on wildly different scales — spend_rate_2m goes up to ¥50,000 while freq_rate_2m sits between 0 and 10. Random Forest doesn't care about scale — it splits on relative comparisons, not absolute values. And spend_rate_2m and spend_month_comparison are correlated, since both are derived from spending. Random Forest handles correlated features naturally through its random feature sub-sampling at each split. Logistic Regression would be unstable without careful preprocessing.

**What about XGBoost or LightGBM?**

Gradient Boosting would be my immediate next experiment. Gradient Boosting models typically handle class imbalance better than Random Forest because of how they work — they build trees sequentially, and each new tree focuses disproportionately on the examples the previous trees got wrong. In an imbalanced dataset where the positive class is rare, this sequential error-correction mechanism naturally pays more attention to the minority class.

I'd expect a 1 to 3 percentage point improvement in AUC from switching to Gradient Boosting, and potentially a more significant improvement in recall specifically. The trade-off is that Gradient Boosting has more hyperparameters to tune — learning rate, tree depth, sub-sampling rate — and the feature importances are slightly less interpretable than Random Forest's. For a production system where you need to explain predictions to business stakeholders, that interpretability cost matters.

For this analysis, Random Forest as a baseline is the right starting point. It requires minimal tuning, gives directly interpretable feature importances, and still achieves 0.912 AUC. That's a strong baseline to build from.

**Class imbalance — handling 3.2% positive rate.**

Only 1,108 out of 34,420 training users are loyal — a 3.2% positive rate. Without any correction, the model's natural tendency would be to predict 'not loyal' for everyone — which would be 96.8% accurate and completely useless.

I applied two corrections. First, `class_weight='balanced'` in the Random Forest. This tells the model to treat each loyal user as if they were approximately 30 times more important than a non-loyal user, scaled by the inverse of the class frequency. In practice, this means the model is penalised much more heavily for missing a loyal user than for incorrectly flagging a non-loyal one. The result is better recall at the cost of some precision — which is the right trade-off when the business cost of missing a high-LTV user is much higher than the cost of sending a campaign to someone who won't convert.

Second, I used stratified splitting when dividing the data into training and test sets. Stratified splitting ensures that the 3.2% loyalty rate is preserved in both the training set and the test set. Without stratification, random chance could produce a test set with very few loyal users, making evaluation metrics unreliable."

*[Advance slide.]*

---

## SLIDE 7 — Model Results

*[2.5 minutes. Lead with AUC and explain it clearly. Address recall proactively — don't wait to be challenged. The feature importance chart is your leakage sanity check — say that explicitly.]*

"Let's talk about what the model actually produced.

**ROC-AUC of 0.912.**

I want to explain what this number means, because I think it gets misused in presentations.

ROC-AUC — Area Under the Receiver Operating Characteristic Curve — is a measure of the model's ability to rank users correctly. Specifically: if I take one random user who turned out to be loyal and one random user who turned out to be not loyal, what is the probability that my model gives the loyal user a higher probability score?

At 0.912, the model does this correctly 91.2% of the time. That is a strong result. A random model would score 0.5 — no better than a coin flip. A perfect model would score 1.0. 0.912 means the model's probability scores are a reliable basis for ranking and segmenting customers.

And ranking is exactly what the business needs. Marketing doesn't need a binary 'yes this person will be loyal' — they need a ranked list so they can decide who gets the high-value campaign, who gets the medium-value campaign, and who gets deprioritised. AUC measures exactly that ranking quality.

**Why not accuracy?**

I want to address this because accuracy is the first metric most people reach for, and it would be deeply misleading here. A model that predicts 'not loyal' for every single user in the dataset would be 96.8% accurate — because 96.8% of users aren't loyal. That model has no predictive value whatsoever. It's a null model dressed up in a high accuracy number.

Whenever you have severe class imbalance — which we do here at 3.2% positive rate — accuracy is the wrong metric to report. AUC, precision, and recall tell you what's actually happening.

**Feature importances — and why this chart is a leakage sanity check.**

Looking at the feature importance chart on the right of the slide:

- spend_rate_2m: approximately 42% — the dominant signal
- freq_rate_2m: approximately 28%
- days_since_last_purchase_in_window: approximately 18%
- spend_month_comparison: approximately 6%
- unique_shops_2m: approximately 6%

Spend and frequency together account for about 70% of the model's decision-making. That is exactly what we would expect. These two features map most directly to the hard loyalty thresholds — ¥10,000 per month and 2 purchases per month. A user who is already at or near those thresholds in their first 60 days is far more likely to sustain them long term.

Here's why I describe this chart as a leakage sanity check. If the model had inadvertently used future information — information from after day 60 — the features that captured that future data would dominate the importance rankings. Something like a timestamp feature derived from the label evaluation period, or a shop count computed from the full dataset, would score extremely high. The fact that spend and frequency dominate — exactly as the loyalty definition would predict — is confirmation that the features are clean and the leakage prevention worked.

Unique shops at 6% is low and worth explaining. In the first 60 days, most users have only visited one merchant regardless of whether they eventually become multi-shop users. The 60-day window is simply too short to observe platform loyalty through breadth for most people. This is an inherent limitation of the feature window, not a flaw in the feature itself.

**The honest number — recall of 0.45.**

At the default 0.5 classification threshold, the model correctly identifies 45% of future loyal users. We are missing more than half.

I want to be upfront about this rather than glossing over it. It's the primary area for improvement.

The context: we have 1,108 loyal users in 34,420 training examples. The model is learning to identify a signal from a very small number of positive examples. class_weight='balanced' helps — it pushes recall up from what it would otherwise be — but there is a ceiling on what's achievable with this training set size.

Two things make this acceptable as a starting point. First, the High likelihood segment — users scoring above 66% — has an observed loyalty rate of 83.9%. When the model is confident, it is almost always right. The recall problem is concentrated in users who are on the borderline — users who have some positive signals but not enough to cross the confidence threshold. Many of these users land in the Medium segment, where targeted campaigns can still drive conversion.

Second, the fixes are clear and concrete. Lowering the classification threshold from 0.5 to, say, 0.3 would increase recall at the cost of precision — more users get flagged as likely loyal, but more of those flags are false positives. Switching to Gradient Boosting would likely improve recall without sacrificing as much precision. And as Spendy accumulates more data over time, the positive training set will grow, and the model will have more examples to learn from."

*[Advance slide.]*

---

## SLIDE 8 — Business Segmentation

*[2 minutes. This is the 'so what' slide. Be concrete about the playbooks. The numbers are compelling — let them speak.]*

"Everything we've done so far — the loyalty definition, the model, the features — is infrastructure. This slide is where it becomes a business tool.

The model outputs a probability score for every new user at the end of their second month. We translate those scores into three segments. The thresholds — 0.33 and 0.66 — divide the 0 to 1 probability range into thirds. Every user gets one of three labels: Low likelihood, Medium likelihood, or High likelihood.

Here's what that looks like in practice.

**High likelihood — probability score above 0.66.**

2,135 users fall into this segment. Their observed loyalty rate — meaning the percentage who actually became loyal by December 31 — is **83.9%**. Think about that number. Four out of every five users the model placed in this segment actually became loyal.

The average predicted probability for this group is 0.95 — meaning the model wasn't hedging. These are users the model was highly confident about, and that confidence was warranted.

The marketing playbook for this segment is reinforcement, not acquisition. These users are already on the right trajectory. They don't need to be convinced — they need to feel recognised. A premium tier badge, early access to a new merchant on the platform, a small cashback reward that signals Spendy has noticed their behaviour. The goal is to lock in what's already happening and make them feel like platform insiders. Spend is low per user because you're accelerating an inevitable outcome, not fighting an uphill battle.

**Medium likelihood — probability score between 0.33 and 0.66.**

Approximately 6,000 users. Observed loyalty rate of 6.3% — twice the overall base rate of 3.2%. Average predicted probability of 0.44.

This is the highest ROI opportunity in the whole segmentation. These users have real signals — spend, frequency, some engagement — but they haven't crossed the model's confidence threshold. Something is holding them back. The most common gap, given what we know about the loyalty definition, is the platform loyalty component. They're likely single-shop users who haven't yet used Spendy at a second merchant.

The targeted playbook: a cashback offer or discovery incentive tied specifically to a second merchant visit. 'Try Spendy at any of these three merchants and get ¥500 cashback.' That's not a generic promotion — it's a precisely targeted nudge that directly addresses the loyalty gap. If they make that second merchant visit, the probability of them ultimately qualifying as loyal jumps dramatically.

**Low likelihood — probability score below 0.33.**

26,000 users. Observed loyalty rate of just 1.0% — below even the base rate of 3.2%. Average predicted probability of 0.10.

The playbook here is restraint. In a world of unlimited marketing budget, you might try to move these users. But marketing budgets are not unlimited, and spending reward money on users who have shown almost no signal of platform loyalty in their first 60 days is a low-return use of that budget.

The better approach: redirect the budget from this segment to the High and Medium segments where the ROI is proven. There may be a subset of Low likelihood users worth a lightweight re-engagement message — a simple reminder that Spendy exists — but the heavy investment should not be here.

The segment distribution chart at the bottom of the slide confirms that the model is doing real work — it's not trivially dumping everyone into one category. There's meaningful spread across all three tiers. The scoring is working."

*[Advance slide.]*

---

## SLIDE 9 — Business Impact

*[60–90 seconds. Three big numbers, then the three-tier playbook. This is the 'so what' moment — speak it like you mean it.]*

"This slide answers the question every business stakeholder in the room is actually asking: what does this analysis give Spendy that it didn't have before?

Three things.

**6,273 loyal customers — identified for the first time.**

Spendy now has a formal answer to 'who are our loyal customers?' It didn't have one before. 6,273 users, named and scored, who cleared every component of the loyalty definition. That's a list you can action immediately.

**Month 2 — the earliest scoring window.**

The model scores new users after 60 days. That's before the habits are fixed, before the user has decided whether Spendy is part of their life or not. A well-placed incentive at month two is cheap and effective. Waiting until month twelve means you're either reinforcing an outcome that already happened or fighting churn that started six months ago.

**83.9% actual loyalty in the High segment.**

Four in five users the model was confident about turned out to be genuinely loyal. That's not a model metric — it's an observed real-world outcome. When the model says High, it's almost always right.

And the bottom half of the slide is the operational output — three tiers, three distinct actions.

High, above 66%: reinforce. Acknowledge loyalty early. Exclusive offers, status recognition, make them feel seen before a competitor does.

Medium, 33 to 66%: nudge. Targeted spend incentive or a merchant discovery prompt to close the loyalty gap. Usually the missing signal is a second merchant visit.

Low, below 33%: deprioritise. Investigate re-engagement or redirect budget to the segments where it moves outcomes."

*[Advance slide.]*

---

## SLIDE 10 — Future Improvements

*[60–90 seconds. Show self-awareness, not defensiveness. Go through the five quickly — don't dwell on any one.]*

"The analysis was built on one dataset covering seven months. There are five clear directions to extend it, all of which came directly out of the notebook.

**One — longer observation period.** Seven months gives us enough to build a model, but not enough to validate the loyalty definition across a full seasonal cycle. A full year or more of data would provide more loyal-class examples and let us test whether the 4.6% rate holds across different periods.

**Two — richer transaction data.** The dataset has five columns — transaction ID, user, shop, timestamp, price. No product categories, no demographics, no promotion history, no marketing channel data. Any of those would meaningfully improve signal. Shop categories alone would transform the unique-shops feature — visiting a grocery store and a pharmacy is very different from visiting the same merchant twice.

**Three — additional behavioural features.** Purchase regularity within the 60-day window is something we couldn't fully capture with the current features. A feature tracking whether a user buys in both weeks of each month — not just the aggregate count — would help identify customers who are building a genuine habit versus those who burst and fade.

**Four — model refinement.** Gradient Boosting with calibrated predicted probabilities is the natural next comparison. I'd expect a meaningful recall improvement. Calibrated probabilities also make the segment thresholds more reliable in production — a score of 0.70 should genuinely mean 70% probability, which requires explicit calibration that Random Forest doesn't provide natively.

**Five — business-driven segmentation.** The 33% and 66% cutoffs are heuristic starting points. The right way to set them is via decision theory: at what probability does the expected ROI of a marketing action become positive? That requires campaign cost and LTV data — a business and finance conversation, not a modelling one."

*[Advance slide.]*

---

## SLIDE 11 — Conclusion

*[This is your close. Speak slowly. Don't rush. Eye contact. Let the final line land before you say 'thank you'.]*

"I want to close with the three things this analysis delivers — because I think it's easy to get lost in the technical detail and lose sight of what was actually built.

**DEFINE.**

The Brand Loyalist Problem — 75% of users buying from a single merchant — is not a curiosity in the data. It is a fundamental challenge for building a meaningful loyalty programme on a BNPL platform. A loyalty definition that ignores it rewards the wrong users.

The five-component AND/OR definition addresses it directly. Four hard requirements that every loyal customer must clear simultaneously. A platform loyalty check with two routes — breadth through multiple merchants, or depth through sustained monthly engagement. 4.6% loyalty rate. 6,273 users. Every single one has earned the label.

**PREDICT.**

A new user's trajectory can be identified at Month 2 — before the habits are fixed, before the pattern is set. A Random Forest model trained on five features engineered from the first 60 days of transaction history achieves a ROC-AUC of 0.912. The model correctly ranks a loyal user above a non-loyal one more than 91% of the time. No data leakage. Scores are available at the earliest point that marketing can act on them.

**ACT.**

Three customer segments. Each with a different marketing implication.

High likelihood users — reinforce what's already happening. Medium likelihood users — nudge them toward the specific behaviour gap, which is usually platform breadth. Low likelihood users — deprioritise and redirect budget to where it moves outcomes.

The 83.9% observed loyalty rate in the High segment is not a model metric. It is a real-world observation. More than four in five users the model was confident about actually became loyal. The segmentation works.

The validation path is defined. A randomised A/B experiment — split new users by model score, send differentiated campaigns, measure loyalty rate and revenue per user at six months. The framework is ready. The model is ready. The next step is a controlled test in production.

Thank you — I'm very happy to take questions."

*[Pause for 2 seconds after 'thank you.' Don't fill the silence. Let it land.]*

---

## APPENDIX A — Production Pipeline

*[Use this section if asked: 'How would this work in production?' / 'What's the deployment plan?' / 'How do you monitor model drift?']*

"The pipeline has six stages — three in each row on the slide.

**Row 1: getting data into a score.**

Data Ingestion: transactions land in BigQuery or S3. On the first of each month, an EventBridge trigger fires the pipeline.

Feature Pipeline: for every user who crossed their 60-day mark since the last run, we compute the five features from days 0–60. This is a straightforward dbt or Apache Beam job — one row per user, five columns out.

Model Scoring: the serialised RandomForest object calls predict_proba() on that feature table. Each user gets a probability between 0 and 1.

**Row 2: turning a score into an action.**

Segmentation: the probability maps to a tier label — High above 0.66, Medium between 0.33 and 0.66, Low below 0.33. Those labels get written to the CRM or marketing automation platform. Marketing runs the appropriate campaign for each tier immediately.

Monitoring: two checks run monthly. A KS-test on each feature distribution against the training baseline — if a feature shifts significantly, something has changed in the user population or the data pipeline. And lagged ground truth tracking — at six months post-scoring, we check what fraction of High-scored users actually became loyal. If that drops materially below 83.9%, the model is drifting.

A/B Testing: new cohorts get randomly split into control and treatment. We measure loyalty rate and revenue per user at six months to establish causal impact — not just correlation. Campaign ROI feeds back into the threshold calibration."

---

## Extended Q&A Preparation

### If asked about the 4.6% loyalty rate being too low:

"The 4.6% is not a target I set — it's an outcome of the definition I built. The question is whether the definition is calibrated correctly for the business objective.

The assumption I made throughout this analysis is that loyal customers will receive some form of monetary benefit — cashback, exclusive merchant access, early features. When real money is attached to a label, the cost of a false positive is measurable. A user who qualifies on a technicality — Christmas shopping in December, a two-month spike before churning — but doesn't represent genuine sustained loyalty, consumes reward budget without generating the long-term LTV that justifies it.

A loyalty programme covering 30% of users isn't really a loyalty programme — it's a blanket discount scheme. The unit economics of the two are very different.

If the business determines that the rewards are low-cost enough to justify broader reach, or if LTV data shows that users qualifying at lower thresholds still generate meaningful incremental revenue, then the thresholds should be relaxed — but that's a decision that should be made with LTV data in hand, not by looking at a percentage and deciding it looks too small."

---

### If asked why recall is low and what you'd do:

"Recall of 0.45 means we're catching 45% of future loyal users at the default 0.5 threshold. That is the primary area for improvement, and I want to be clear about why it happens and what the concrete fixes are.

The root cause is the training set size. 1,108 loyal users out of 34,420 total. class_weight='balanced' helps by increasing the penalty for missing a loyal user during training, but there's a ceiling on what any model can learn from this few positive examples.

Three fixes in priority order. First, lower the classification threshold. Instead of predicting 'loyal' when the probability exceeds 0.5, lower it to 0.3 or 0.25. More users get the loyal label, including more actual loyal users — but also more false positives. The right threshold is determined by the cost ratio: how much does it cost to send a campaign to a false positive versus how much value is lost by missing a true loyal user?

Second, switch to Gradient Boosting. XGBoost and LightGBM tend to achieve higher recall on imbalanced datasets because of their sequential error correction mechanism. I'd expect a meaningful improvement here — possibly closing the gap from 0.45 to 0.55 or higher.

Third, grow the training set. As time passes and more users complete their loyalty journey, the positive training set grows naturally. A model retrained in 6 months with twice as many loyal examples will perform better than the current one."

---

### If asked which assumption worries you most:

"The assumption I'm least comfortable with is that the training population — users who joined between June and September 2020 — is representative of users who will join in the future.

We trained on users from a specific 4-month window in 2020. If Spendy's user acquisition strategy changes significantly — targeting a different demographic, partnering with different merchant categories, running different promotions — the feature distributions of new users may look quite different from the training population. The model was trained to score users who look like 2020 users. If 2026 users look different, the model's predictions will be less reliable.

This is exactly what the monitoring layer addresses. By running KS tests on feature distributions every month and tracking observed loyalty rates per segment, we can detect drift before it significantly degrades the model's performance. But it's the assumption I'd want to actively monitor rather than take for granted."

---

### If asked what you'd do differently with more time:

"Three things, in order of impact.

First — compare Gradient Boosting models. XGBoost and LightGBM are the natural next comparison, and I'd expect a meaningful improvement in recall specifically. That experiment probably takes a day to set up and run properly.

Second — bring in LTV data to back-solve the loyalty thresholds. Right now the thresholds are set by looking at data distributions and finding where meaningful breaks exist. That's reasonable for a first pass. But the right way to set thresholds in production is to calculate them from business value: what is the expected LTV of a loyal user over 12 months, what is the cost of a loyalty reward, and at what threshold combination does the programme have positive expected value? That calculation requires actual LTV data, which this dataset doesn't have.

Third — design and run an A/B experiment to validate the segmentation strategy causally. The 83.9% observed loyalty rate in the High segment is compelling, but it's observational — we don't know how much of that is attributable to the model's accuracy versus the users simply being predisposed to loyalty regardless of any campaign. A randomised controlled trial is the only way to establish causality. Split new users into control and treatment at random, deliver segment-differentiated campaigns to treatment, and measure loyalty rate and revenue per user at 6 months. That gives you the causal impact — not just the correlation."

---

*Script prepared for Spendy Data Scientist Final Round — 2026*
*Edit freely — this is deliberately over-written so you can cut to your natural pace.*
