# TakeMeter — Planning Document

---

### Community

I chose **r/investing** (reddit.com/r/investing), a large general-purpose investing forum with hundreds of thousands of active participants. What makes it a good fit for sentiment classification is that the discourse is all over the place: some posts are optimistic buy recommendations, others are doom-and-gloom predictions, and plenty are just people asking questions or recapping news. That variety is exactly what makes the classification problem interesting, and getting it right would actually be useful to people in the community who want a quick read on the overall mood before diving in.

---

### Label Taxonomy

**Label 1: `bullish`**
The post expresses net optimism about a stock, sector, asset class, or the market overall, predicting gains, recommending buying or holding, or brushing off concerns about downside risk.

*Example 1:*
> "Inflation is cooling faster than the Fed expected and earnings season has been surprisingly strong. I've been adding to my S&P 500 position every dip and I think we're headed for new all-time highs by Q3."

*Example 2:*
> "Everyone is sleeping on small-cap value right now. The valuation discount to large caps is at a 20-year extreme. When rates stabilize this is going to rip."

---

**Label 2: `bearish`**
The post expresses net pessimism about a stock, sector, asset class, or the market overall, predicting losses, recommending selling or avoiding, or warning about significant downside risk.

*Example 1:*
> "The SpaceX IPO is going to be a disaster for retail investors. The valuation being floated makes zero sense given their actual revenue and the regulatory environment around Starlink. Don't touch it."

*Example 2:*
> "Commercial real estate is quietly becoming a systemic problem. Regional banks are sitting on massive unrealized losses. This isn't priced in yet and it's going to matter."

---

**Label 3: `neutral`**
The post takes no clear directional stance. It asks a question, recaps news without an opinion, lays out balanced pros and cons, or describes a portfolio situation without pushing a view.

*Example 1:*
> "Can someone explain how rising yields affect bond prices? I keep seeing this mentioned but I'm not sure I understand the mechanism."

*Example 2:*
> "The Fed held rates steady today as expected. Powell's statement acknowledged both persistent inflation and signs of labor market softening. Markets closed roughly flat."

---

### Hard Edge Cases and Decision Rules

**Edge case 1: mixed-stance posts**

> "I'm rotating out of tech entirely — valuations are still stretched and I think we see another leg down. Moving into energy and commodities, which I think are the only sectors with real upside from here."

This post is bearish on tech but bullish on energy/commodities, which means it can't cleanly fit both labels.

**Decision rule:** Label by the dominant stance — whichever direction the post spends more words arguing, or wherever it lands at the end. If a post opens with a bearish argument but closes with a bullish recommendation, label it bullis`. If it's genuinely split with no dominant direction, label it neutral.

In this case, the post's main action is a rotation into energy/commodities with a constructive view on that trade. The tech bearishness is setup, not the conclusion. Label: bullish.

**Edge case 2: sarcasm**

> "Oh great, another rate hike. Totally fine. Everything is completely under control. 🙂"

The literal words sound calm, but the intent is clearly bearish. **Decision rule:** Label by the intended sentiment, not the literal words. Sarcasm markers (emojis, phrases like "totally fine" or "nothing to see here,") flip the label to the opposite of the surface reading.

---

## Full Specification
 
### Data Collection Plan
 
**Source:** Public posts and top-level comments from r/investing, collected manually by browsing the Hot, Top (past month), and New feeds.
 
**Volume target:** 70–75 examples per label (210–225 total), aiming for a roughly balanced distribution. Keeping it near 33% per label matters — otherwise the model just learns to predict the majority class.
 
**Collection strategy:**

| Feed | Posts to visit | Examples per post | Target examples |
|---|---|---|---|
| Hot | ~27 | 1 post + 1 comment = 2 | ~54 |
| Top (past month) | ~23 | 1 post + 1 comment = 2 | ~46 |
| New | ~20 | 1 post + 1 comment = 2 | ~40 |
 
Near the end of collection, targeted keyword searches ("market crash," "overvalued," "recession") helped fill label gaps and push the total to 200.

A few ground rules I followed throughout:
- Skip any post under 15 words.
- Collect posts and comments together — both show up in real feeds and the model should handle both.
- Prioritize variety over recency: different asset classes (equities, bonds, commodities, real estate), different time horizons (day-trading reactions vs. long-term theses), and different post lengths (short takes and long-form arguments).
- Save raw text and label in a CSV with a `notes` column for difficult cases.

**Imbalance contingency:** After labeling the first 100 examples, I checked the distribution. If any label was below 25 examples (< 25%), I actively sought more of that type. Neutral posts were the hardest to find in adequate volume — an opinion-heavy forum isn't naturally full of question posts — so I deliberately targeted question threads and news recap posts to reach the target count.
 
---
 
### Evaluation Metrics
 
**Why accuracy alone isn't enough:** With three balanced classes, a model that always predicts the majority class hits ~33% accuracy — technically above random, but completely useless. Worse, a model that nails bullish and bearish but completely fails on neutral can still post "acceptable" overall accuracy. That would look fine on paper while being broken for a third of its inputs.
 
**Metrics I'll use:**
 
- **Per-class F1 score** (primary): F1 is the harmonic mean of precision and recall for each label. It catches both failure modes: a model that barely predicts a label at all (high precision, low recall) and one that over-predicts it (low precision, high recall). I'll report F1 for all three classes separately.
- **Macro-averaged F1** (summary): The unweighted average of per-class F1. Treats each class equally regardless of size, which is right here — all three labels matter equally.
- **Confusion matrix**: A 3×3 matrix showing where the errors pile up. The most useful signal is directionality — does the model confuse `bearish` with `neutral` more than `bearish` with `bullish`? That would tell me whether it's failing at stance detection or at intensity.
- **Overall accuracy**: Reported for both models for easy comparison, but always read alongside F1.

**Why these metrics fit this task:** Sentiment classification on a balanced, three-class dataset is best evaluated with macro F1 because there's no cost asymmetry between label types — getting a bullish post wrong is just as bad as getting a neutral one wrong. If the dataset ends up slightly imbalanced, macro F1 still penalizes ignoring minority classes.
 
---
 
### Definition of Success
 
A classifier is genuinely useful for a community sentiment tool if it clears all three of these thresholds on the held-out test set:
 
1. **Overall accuracy ≥ 75%** — meaningfully above the 33% random baseline
2. **Per-class F1 ≥ 0.65 for all three labels** — no label can be effectively ignored
3. **Fine-tuned model outperforms zero-shot baseline on macro F1** — if fine-tuning on 200 labeled examples doesn't beat a general-purpose LLM with no task-specific training, the labeling effort didn't produce a learnable signal

These thresholds are achievable on a 200-example dataset while still being high enough to mean something. A model hitting 75% accuracy with balanced per-class F1 is right three out of four times across all sentiment types — good enough for an aggregate sentiment dashboard, though not something I'd use for any high-stakes individual decision.
 
If the model falls short: check whether the underperforming label has too few training examples, whether the annotation was inconsistent for that class, or whether the label boundary itself is too ambiguous to learn from text alone.
 
---
 
### AI Tool Plan
 
**1. Label stress-testing (before annotation)**
 
Before annotating anything, I'll give Claude my three label definitions and both edge case rules and ask it to generate 10 posts that sit at the boundary between two labels — specifically targeting the neutral/bearish edge (where a factual-sounding post might actually be making a bearish argument) and the bullish/neutral edge (where optimism might be hedged enough to read as balanced). If Claude generates posts I can't cleanly classify with my existing rules, that's a signal to tighten the definitions *before* I commit to 200 annotations.
 
**2. Annotation assistance (during data collection)**
 
I'll use Claude to pre-label batches of 20–30 posts at a time by providing it the label definitions and raw post text. For every batch, I'll review and correct each pre-assigned label independently — no pre-label gets finalized without me reading the post myself. Pre-labeled examples will be flagged in a `prelabeled` column in the CSV so the annotation assistance is fully disclosed.
 
**3. Failure analysis (after fine-tuning)**
 
After running the fine-tuned model on the test set, I'll paste all misclassified examples into Claude and ask it to identify common patterns: Are wrong predictions concentrated in short posts? Posts with financial jargon but no directional language? A specific label pair? I'll verify each suggested pattern by re-reading the misclassified examples myself and include in the evaluation report both the patterns that held up and any Claude suggestions I had to discard after review.
 
---
 
## Milestone 3: Hard Annotation Decisions
  
**Hard case 1: Long analytical post with a bearish conclusion buried in data**
 
> *"The $4 trillion liquidity drain — In March Nasdaq changed how companies enter the Nasdaq 100... SpaceX listed with a 4% free float, meaning almost no supply, yet an estimated 22 to 27 billion in forced passive buying hit immediately... When the rotation reverses, there may not be enough liquidity left in the drained markets to catch the fall."*
 
This post reads like a neutral financial analysis for most of its length — it cites specific numbers, raises a question at the end, and has the register of data journalism. The factual framing and question-like ending both pull toward `neutral`. But reading it through, the entire structure builds toward a warning: capital is being drained from gold, crypto, and EM markets, and when the rotation reverses, the markets won't be liquid enough to absorb the outflow. The conclusion is explicitly bearish ("there may not be enough liquidity left to catch the fall").
 
**Decision:** Labeled `bearish`. The analytical register is a delivery vehicle for a directional argument, not genuine balance. This is the "factual-sounding but actually bearish" pattern the edge case rules were written to handle.
 
---
 
**Hard case 2: Question post that reveals a bullish conclusion**
 
> *"Should a young person invest in stocks or real estate (rentals)?... I'm 17, soon to be 18... the more I learn about the broader financial systems, the more I start to believe that real estate could be a better investment... it's my personal belief that real estate (with a tenant) could be a better place to store my capital if I want to maximize long-term wealth creation... What do you all think? Am I missing something, or is it truly just an avenue to wealth creation that is often ignored?"*
 
The question framing at the opening and closing pull toward `neutral`. But the body of this post isn't actually open — the author has already formed a bullish opinion on real estate, cites the leverage argument, and is mostly seeking validation rather than genuine input.
 
**Decision:** Labeled `bullish`. The question framing is a social convention, not evidence of a genuinely neutral stance. Posts that ask questions rhetorically while arguing a position belong in the label that matches the position being argued.
 
---
 
**Hard case 3: Venting post with an implied bearish market view**
 
> *"The worst part about everything he is doing is that it is like watching a slow moving train crash... Lets just rip off the bandaid and crash the stock market like its never been done before. Lets just embargo the whole world and isolate the states. Why wait and draw out the process? Get it over with so we can rebuild and put this behind us."*
 
This is frustration-venting about political policy, not a structured investment argument. It mentions a market crash but the framing is fatalistic ("get it over with so we can rebuild") rather than a prediction or recommendation. I went back and forth between `neutral` (no actionable investment stance, just venting) and `bearish` (explicitly anticipates a severe market crash).
 
**Decision:** Labeled `bearish`. Even though the post is emotional rather than analytical, it explicitly predicts and essentially advocates for a severe market crash. The sentiment toward market outcomes is unambiguously negative. Venting posts that express a clear directional market view get labeled by that view — not automatically `neutral` just because they're not analytical.
 
---

## Baseline Results:
Baseline accuracy: 0.833  (evaluated on 30/30 parseable responses)

Per-class metrics (baseline):
              precision    recall  f1-score   support

     neutral       0.79      0.92      0.85        12
     bearish       0.88      0.78      0.82         9
     bullish       0.88      0.78      0.82         9

    accuracy                           0.83        30
   macro avg       0.85      0.82      0.83        30
weighted avg       0.84      0.83      0.83        30

The baseline stumbled most on neutral labels. My guess is that bearish and bullish language can appear inside a neutral comment — as context or comparison — without the post itself taking a directional stance, and the model had trouble separating the two.

---
 
## Stretch Features
