# TakeMeter — Planning Document

---

### Community

I chose **r/investing** (reddit.com/r/investing), a large general-purpose investing discussion community with hundreds of thousands of active participants.The discourse is highly varied in quality and directional stance, which makes it a strong fit for a sentiment classification task. These classifications are important for this community because they allow people to quickly gauge what the overall forum sentiment is like, which can help with research, and deciding what plan to carry out next.

---

### Label Taxonomy

**Label 1: `bullish`**
The post expresses net optimism about a stock, sector, asset class, or the market overall — predicting gains, recommending buying or holding, or dismissing concerns about a downside risk.

*Example 1:*
> "Inflation is cooling faster than the Fed expected and earnings season has been surprisingly strong. I've been adding to my S&P 500 position every dip and I think we're headed for new all-time highs by Q3."

*Example 2:*
> "Everyone is sleeping on small-cap value right now. The valuation discount to large caps is at a 20-year extreme. When rates stabilize this is going to rip."

---

**Label 2: `bearish`**
The post expresses net pessimism about a stock, sector, asset class, or the market overall — predicting losses, recommending selling or avoiding, or warning of significant downside risk.

*Example 1:*
> "The SpaceX IPO is going to be a disaster for retail investors. The valuation being floated makes zero sense given their actual revenue and the regulatory environment around Starlink. Don't touch it."

*Example 2:*
> "Commercial real estate is quietly becoming a systemic problem. Regional banks are sitting on massive unrealized losses. This isn't priced in yet and it's going to matter."

---

**Label 3: `neutral`**
The post takes no clear directional stance. It asks a question, recaps news without an opinion, presents genuinely balanced pros and cons, or describes a portfolio situation without advocating for a view.

*Example 1:*
> "Can someone explain how rising yields affect bond prices? I keep seeing this mentioned but I'm not sure I understand the mechanism."

*Example 2:*
> "The Fed held rates steady today as expected. Powell's statement acknowledged both persistent inflation and signs of labor market softening. Markets closed roughly flat."

---

### Hard Edge Case and Decision Rule

**Edge case 1: mixed-stance posts**

> "I'm rotating out of tech entirely — valuations are still stretched and I think we see another leg down. Moving into energy and commodities, which I think are the only sectors with real upside from here."

This post is bearish on tech but bullish on energy/commodities. It cannot cleanly be both.

**Decision rule:** Label by the dominant stance (the direction the post spends more words arguing, or the conclusion it ends on). If a post opens with a bearish argument and closes with a bullish recommendation, label it `bullish`. If it's genuinely 50/50 with no dominant direction, label it `neutral`.

In the example above, the post's primary action is a rotation into energy/commodities with a bullish sentiment on that trade. Despite the bearish framing on tech, the dominant stance is constructive --> label `bullish`.

**Edge case 2: sarcasm**

> "Oh great, another rate hike. Totally fine. Everything is completely under control. 🙂"

The literal words sound calm, but the intent is clearly bearish. **Decision rule:** Label by the intended sentiment, not the literal words. Sarcasm markers (emojis, phrases like "totally fine" or "nothing to see here," explicit irony) flip the label to the opposite of the surface reading.

---

## Full Specification
 
### Data Collection Plan
 
**Source:** Public posts and top-level comments from r/investing, collected manually by browsing the Hot, Top (past month), and New feeds.
 
**Volume target:** 70–75 examples per label (210–225 total), aiming for a roughly balanced distribution. A target of ~33% per label keeps the model from defaulting to a majority class.
 
**Collection strategy:**

| Feed | Posts to visit | Examples per post | Target examples |
|---|---|---|---|
| Hot | ~27 | 1 post + 1 comments = 2 | ~80 |
| Top (past month) | ~23 | 1 post + 1 comments = 2 | ~70 |
| New | ~20 | 1 post + 1 comments = 2 | ~60 |

- Skip any post under 15 words.
- Collect posts and comments together, since both appear in real community feeds and the model should generalize across both formats.
- Prioritize variety over recency: include posts about different asset classes (equities, bonds, commodities, real estate), time horizons (day trading reactions vs. long-term theses), and post lengths (short reactions and long-form arguments).
- Save raw text and label in a CSV with a `notes` column for difficult cases.

**Imbalance contingency:** After labeling the first 100 examples, check the distribution. If any label is below 25 examples (< 25%), actively seek examples of that label specifically — filter by post content or search for relevant keywords. Neutral posts tend to be the hardest to find in adequate volume on an opinion-heavy forum, so I may need to deliberately target question posts and news recap threads to reach the target count.
 
---
 
### Evaluation Metrics
 
**Why accuracy alone is not enough:** With three balanced classes, a model that always predicts the majority class would achieve ~33% accuracy — a trivially low floor, but accuracy still doesn't tell you *which* classes are failing. A model that perfectly classifies bullish and bearish posts but always gets neutral wrong would show acceptable overall accuracy while being completely useless for one third of its inputs.
 
**Metrics I will use:**
 
- **Per-class F1 score** (primary metric): F1 is the harmonic mean of precision and recall for each label. It catches both the case where the model is too conservative (high precision, low recall — predicts a label rarely but correctly) and too aggressive (low precision, high recall — predicts the label constantly, picks up real cases but also many false positives). I will report F1 for all three classes separately.
- **Macro-averaged F1** (summary metric): The unweighted average of per-class F1. This treats each class equally regardless of size, which is appropriate here since all three labels matter equally to the use case.
- **Confusion matrix**: A 3×3 matrix showing where the model's errors concentrate. The most informative signal is directionality — e.g., does the model confuse `bearish` with `neutral` more than `bearish` with `bullish`? That pattern would reveal whether the model is failing at stance detection or at intensity distinction.
- **Overall accuracy**: Reported for both models for direct comparison, but interpreted alongside F1, not in isolation.
**Why these metrics fit this task:** Sentiment classification on a balanced dataset with three equally-weighted classes is best evaluated with macro F1, because there is no "cost asymmetry" between label types — misclassifying a bullish post as bearish is as bad as misclassifying a neutral post as bullish. If the dataset ends up slightly imbalanced, macro F1 still penalizes the model for ignoring minority classes.
 
---
 
### Definition of Success
 
A classifier is genuinely useful for a community sentiment tool if it meets all three of the following thresholds on the held-out test set:
 
1. **Overall accuracy ≥ 75%**: This is significantly above the 33% random baseline
2. **Per-class F1 ≥ 0.65 for all three labels**: No single label can be effectively ignored by the model. A label with F1 below 0.65 means the model is failing and unreliable.
3. **Fine-tuned model outperforms zero-shot baseline on macro F1** — if fine-tuning on 200 labeled examples does not beat a general-purpose LLM with no task-specific training, the labeling effort did not produce a learnable signal and the labels or data quality need revision.
These thresholds are set to be achievable on a 200-example dataset while being high enough to be meaningful. A model that hits 75% accuracy with balanced per-class F1 is making correct calls three out of four times across all sentiment types — sufficient for an aggregate sentiment dashboard, though not reliable enough for any high-stakes decision.
 
If the model falls short: investigate whether the underperforming label has too few training examples, whether the annotation was inconsistent for that class, or whether the label boundary itself is too ambiguous to learn from text alone.
 
---
 
### AI Tool Plan
 
**1. Label stress-testing (before annotation)**
 
Before annotating any examples, I will give Claude my three label definitions and both edge case rules and ask it to generate 10 posts that sit at the boundary between two labels, specifically targeting the neutral/bearish boundary (where a factual-sounding post might actually be making a bearish argument) and the bullish/neutral boundary (where optimism might be hedged enough to read as balanced). If Claude produces posts I cannot cleanly classify using my existing decision rules, that is a signal to tighten the definitions before committing to 200 annotations.
 
**2. Annotation assistance (during data collection)**
 
I will use Claude to pre-label batches of 20–30 posts at a time by providing it the label definitions and the raw post text. For each batch, I will review and correct every pre-assigned label independently. I will not accept any pre-label without reading the post myself. Pre-labeled examples will be flagged in a prelabeled column in the CSV (value: yes) so the annotation assistance is fully disclosed in the AI usage section of the README.
 
**3. Failure analysis (after fine-tuning)**
 
After running the fine-tuned model on the test set, I will paste all misclassified examples into Claude and ask it to identify common patterns: Are wrong predictions concentrated in short posts? Posts that use financial jargon without directional language? A specific label pair? I will then verify each suggested pattern by re-reading the misclassified examples myself, and include in the evaluation report both the patterns that held up and any Claude suggestions I had to discard after review.
 
---
 
## Milestone 3: Hard Annotation Decisions
 
*To be filled in during data collection with at least 3 genuine edge cases and what was decided.*
 
---
 
## Stretch Features
 
*Update this section before starting any stretch feature.*