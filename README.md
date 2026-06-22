# TakeMeter — Reddit Investing Sentiment Classifier

**AI201 · Project 3**

A fine-tuned text classifier that labels posts and comments from r/investing as bullish, bearish, or neutral. The project compares a fine-tuned DistilBERT model against a zero-shot LLM baseline to see how much value 200 labeled examples actually adds.

---

## Community Choice

**Community:** r/investing (reddit.com/r/investing)

r/investing is a large, general-purpose investing forum with hundreds of thousands of active participants discussing equities, bonds, real estate, crypto, macroeconomics, and personal portfolio strategy. It turned out to be a good fit for a sentiment classification task for a few reasons:

1. Posts routinely take explicit directional stances — "this is going to crash," "add on every dip" — which makes labeling the clear cases pretty straightforward.
2. The community is large enough that all three sentiment types (bullish, bearish, neutral) show up in natural proportion, so I didn't have to force the data collection.
3. The distinction between these three categories is actually meaningful to community members. Knowing whether the current forum mood is bullish or bearish is useful context for someone doing research or deciding what to do next.

---

## Label Taxonomy

### `bullish`
The post expresses net optimism about a stock, sector, asset class, or the market overall — predicting gains, recommending buying or holding, or dismissing concerns about downside risk.

**Example 1:**
> "Inflation is cooling faster than the Fed expected and earnings season has been surprisingly strong. I've been adding to my S&P 500 position every dip and I think we're headed for new all-time highs by Q3."

**Example 2:**
> "Everyone is sleeping on small-cap value right now. The valuation discount to large caps is at a 20-year extreme. When rates stabilize this is going to rip."

---

### `bearish`
The post expresses net pessimism about a stock, sector, asset class, or the market overall — predicting losses, recommending selling or avoiding, or warning about significant downside risk.

**Example 1:**
> "The SpaceX IPO is going to be a disaster for retail investors. The valuation being floated makes zero sense given their actual revenue and the regulatory environment around Starlink. Don't touch it."

**Example 2:**
> "Commercial real estate is quietly becoming a systemic problem. Regional banks are sitting on massive unrealized losses. This isn't priced in yet and it's going to matter."

---

### `neutral`
The post takes no clear directional stance — it asks a question, recaps news without an opinion, lays out genuinely balanced pros and cons, or describes a portfolio situation without pushing a view.

**Example 1:**
> "Can someone explain how rising yields affect bond prices? I keep seeing this mentioned but I'm not sure I understand the mechanism."

**Example 2:**
> "The Fed held rates steady today as expected. Powell's statement acknowledged both persistent inflation and signs of labor market softening. Markets closed roughly flat."

---

## Data Collection

**Source:** Public posts and top-level comments from r/investing, collected manually by browsing the Hot, Top (past month), and New feeds. When the label distribution started skewing, I used targeted keyword searches ("market crash," "overvalued," "recession," "buy the dip") to fill gaps.

**Labeling process:** Every example was labeled by me using the definitions above. I used Claude to pre-label batches of 20–30 posts, but reviewed and corrected every single pre-assigned label before finalizing it. About 30% of the pre-labels ended up getting changed — mostly posts Claude called `bearish` that I relabeled `neutral` when the bearish framing wasn't the dominant conclusion, and vice versa. Pre-labeled examples are flagged in the `notes` column of the CSV.

**Label distribution (200 examples):**

| Label   | Count | % of total |
|---------|-------|------------|
| neutral | 77    | 38.5%      |
| bearish | 64    | 32.0%      |
| bullish | 59    | 29.5%      |

The distribution is reasonably balanced. Neutral skews a bit high because r/investing naturally produces a lot of question posts and news recap threads that don't take directional positions.

---

### Difficult-to-Label Examples

**1. Mixed-stance rotation post**
> "I'm rotating out of tech entirely — valuations are still stretched and I think we see another leg down. Moving into energy and commodities, which I think are the only sectors with real upside from here."

This is bearish on tech but explicitly bullish on energy/commodities. Neither label captures it on its own.
**Decision:** Label by the dominant or concluding stance. This post ends with a constructive recommendation (buy energy/commodities), so the dominant stance is bullish. Labeled `bullish`.

**2. Sarcastic bearish post**
> "Oh great, another rate hike. Totally fine. Everything is completely under control. 🙂"

The words are calm, but the sarcasm markers (ironic phrasing, emoji) make the bearish frustration obvious.
**Decision:** Label by intended sentiment, not literal words. Sarcasm flips the surface reading. Labeled `bearish`.

**3. Analysis with an implied bearish conclusion**
> "May CPI inflation will break above 4% and the ECB will hike on Thursday. For those wondering if markets have more room to fall..."

Reads like neutral data analysis, but the framing ("for those wondering if markets have more room to fall") makes the bearish implication clear.
**Decision:** When the framing and conclusion point in a direction even through factual language, label by the implied stance. Labeled `bearish`. The fine-tuned model predicted `neutral` on this one — a failure case I dig into below.

---

## Fine-Tuning Approach

**Base model:** `distilbert-base-uncased` (HuggingFace)

DistilBERT is a 66M-parameter distilled version of BERT. I chose it because it trains in under 15 minutes on a free Colab T4 GPU and handles short-to-medium text classification well.

**Training setup:**
- Train/validation/test split: 70% / 15% / 15% (140 / 30 / 30 examples), stratified by label
- Epochs: 3
- Learning rate: 2e-5
- Batch size: 16 (train), 32 (eval)
- Weight decay: 0.01
- Warmup steps: 50
- Best model selected by validation accuracy

**Why I kept 3 epochs:** I didn't push to 5 or more because the validation accuracy was already declining across epochs (0.43 → 0.40 → 0.37). That pattern meant the model wasn't converging on a useful signal — not underfitting. More epochs would have just made generalization worse.

---

## Baseline

**Model:** `llama-3.3-70b-versatile` via Groq API (zero-shot, no task-specific training)

**Prompt approach:** The system prompt gave the model all three label definitions with one example post per label, then told it to output only the label name. The prompt was written to match the label definitions in `planning.md` exactly — the same grounding a human annotator would have.

**Baseline accuracy:** 83.3% on the 30-example test set (30/30 parseable responses)

---

## Evaluation Report

### Overall Results

| Model                          | Accuracy | Macro F1 |
|--------------------------------|----------|----------|
| Zero-shot baseline (Groq LLaMA) | 0.833    | 0.830    |
| Fine-tuned DistilBERT           | 0.367    | 0.230    |

Fine-tuning produced a **regression of 0.467 accuracy points** relative to the baseline. The fine-tuned model doesn't meet any of the success thresholds from `planning.md` (accuracy ≥ 0.75, per-class F1 ≥ 0.65, outperforms baseline).

---

### Per-Class Metrics

**Zero-shot baseline (Groq LLaMA-3.3-70b):**

| Label   | Precision | Recall | F1   | Support |
|---------|-----------|--------|------|---------|
| neutral | 0.79      | 0.92   | 0.85 | 12      |
| bearish | 0.88      | 0.78   | 0.82 | 9       |
| bullish | 0.88      | 0.78   | 0.82 | 9       |
| **macro avg** | **0.85** | **0.82** | **0.83** | 30 |

**Fine-tuned DistilBERT:**

| Label   | Precision | Recall | F1   | Support |
|---------|-----------|--------|------|---------|
| neutral | 0.37      | 0.83   | 0.51 | 12      |
| bearish | 0.33      | 0.11   | 0.17 | 9       |
| bullish | 0.00      | 0.00   | 0.00 | 9       |
| **macro avg** | **0.23** | **0.31** | **0.23** | 30 |

The fine-tuned model essentially **never predicts `bullish`** (F1 = 0.00) and is nearly as bad on `bearish` (F1 = 0.17). The only class it handles at all is `neutral`, and even that is inflated — it's calling a lot of non-neutral posts neutral (precision 0.37).

---

### Confusion Matrix (Fine-Tuned Model, Test Set)

|                  | Predicted: neutral | Predicted: bearish | Predicted: bullish |
|------------------|--------------------|--------------------|---------------------|
| **True: neutral** | 7                  | 2                  | 3                   |
| **True: bearish** | 2                  | 4                  | 3                   |
| **True: bullish** | 4                  | 3                  | 2                   |

The pattern here is hard to miss: the model predicts `neutral` or `bearish` for almost everything, and `bullish` for almost nothing. Of 9 true bullish examples, only 2 were correctly identified. It's essentially acting like a binary classifier that oscillates between neutral and bearish, with bullish nearly absent from its output.

---

### Wrong Predictions — In-Depth Analysis

**19 of 30 test examples were misclassified.**

#### Failure 1: Bearish posts predicted as neutral
> *"This week: May CPI inflation will break above 4% and ECB will hike on Thursday — for those wondering if markets have more room to fall..."*
> **True: bearish | Predicted: neutral (confidence: 0.37)**

This post is written in a factual, data-heavy register that leads into a directional implication. The fine-tuned model apparently learned to associate that register with `neutral`, keying on surface language rather than the concluding stance ("markets have more room to fall"). LLaMA got it right because it can reason about implied meaning. DistilBERT, trained on 140 examples, seems to have picked up a noisy rule: "factual-sounding = neutral."

#### Failure 2: Bullish posts predicted as neutral
> *"My 5 golden rules for finding stocks — 1. Only buy stocks in an uptrend..."*
> **True: bullish | Predicted: neutral (confidence: 0.37)**

This is a constructive investing advice post — clearly bullish in orientation. The model predicted neutral, which suggests it never developed a reliable `bullish` signal at all. Strategy and advice posts, even when clearly optimistic, got collapsed into neutral. That tracks with the confusion matrix showing bullish recall near zero.

#### Failure 3: Bullish post predicted as neutral due to negative opening
> *"I don't disagree it's a garbage stock but trying to short the richest man in the world who has a cult following... We're still in a bull market..."*
> **True: bullish | Predicted: neutral (confidence: 0.36)**

This post opens with a concession ("garbage stock") before arguing against shorting and affirming the bull market — the dominant stance is bullish. The model appears to have gotten tripped up by the negative language in the opening ("garbage stock," "short"), associating those words with `bearish` or pulling the prediction toward neutral. It's responding to individual words rather than the overall argument arc.

---

### Sample Classifications (Fine-Tuned Model)

| Post (truncated)                                                                 | True Label | Predicted | Confidence |
|----------------------------------------------------------------------------------|------------|-----------|------------|
| "S&P500 Index Committee to NOT Change Rules Regarding Megacaps..."               | neutral    | bearish   | 0.36       |
| "Increased AI bear sentiment and rotation into defensives is a contrarian bull..." | bearish    | neutral   | 0.37       |
| "Great opportunity to SHORT SpaceX now that the S&P refused to break its rules..." | bearish   | neutral   | 0.36       |
| "The gamification of investing will cause the next market crash to be far worse..." | bearish   | neutral   | 0.37       |
| "4 month update on Reddit's favourite stock picks..."                             | bullish    | neutral   | 0.36       |

**On the one correct neutral prediction:** The S&P 500 index committee post is a clean factual recap with no directional opinion — exactly what the model appears to have learned `neutral` means. That prediction is reasonable.

The near-uniform confidence (~0.36–0.37) across all predictions is a bad sign. A well-calibrated model should be more confident on clearer examples. These scores suggest the model didn't learn a meaningfully differentiated signal for any of the three classes.

---

### What the Model Learned vs. What Was Intended

**What was intended:** A three-way sentiment classifier that distinguishes bullish optimism, bearish pessimism, and neutral non-directional posts by understanding the overall argument structure and concluding stance.

**What the model actually learned:** A weak binary distinguisher that defaults heavily to `neutral`, occasionally predicts `bearish` when it sees obvious negative vocabulary, and almost never predicts `bullish`. The learned decision boundary is roughly: "negative financial language → bearish; everything else → neutral."

The model never learned `bullish` at all. This is probably because:

1. **59 training examples (41 after stratified split) is too few** for DistilBERT to converge on a three-way boundary from scratch, especially when all three classes discuss the same domain (markets, stocks, prices) with heavily overlapping vocabulary.
2. **Bullish and bearish posts use mirrored language** — "buy the dip" vs. "sell now," "going to rip" vs. "going to crash" — and a general-purpose language model pretrained on web text may not have enough financial context to learn those directional associations from 140 examples.
3. **Validation accuracy declined every epoch** (0.43 → 0.40 → 0.37), which means the model wasn't overfitting after convergence — it never found a learnable pattern in the first place.

The zero-shot LLaMA baseline succeeds where the fine-tuned model fails because LLaMA already has deep financial vocabulary from pretraining and can apply it immediately with a clear prompt. Fine-tuning a small general-purpose model on 200 domain-specific examples didn't come close to competing with that.

---

### Root Cause Hypothesis

The core problem is almost certainly a **model-data mismatch**: `distilbert-base-uncased` is pretrained on general web text with no financial domain knowledge. Financial sentiment depends on understanding domain-specific signals ("rate hike," "rug pull," "rotation into defensives") that are rare in general pretraining data. A finance-pretrained model like `ProsusAI/finbert` would be a better starting point and would likely show substantially better performance on the same 200-example dataset.

Dataset size is the other constraint. 200 examples (140 for training) is at the absolute minimum for fine-tuning a 66M parameter model. Doubling the dataset to 400+ examples would likely produce meaningful improvement even without switching models.

---

## Spec Reflection

**One way the spec helped:** Writing label definitions precise enough that "two people reading them would agree on most examples" was the most valuable constraint I gave myself. It forced the edge case rules — dominant stance for mixed posts, intended sentiment for sarcasm — to be written out explicitly before annotation began, which made the actual labeling faster and more consistent, and gave me clear criteria for the failure analyses.

**One way implementation diverged from the spec:** The spec assumed the fine-tuned model would achieve meaningful accuracy and the interesting question would be *where* it failed. In practice, the model failed so broadly (accuracy 0.367, bullish F1 = 0.00) that the evaluation became more of a root cause analysis of why fine-tuning failed entirely. That was still informative — it showed that model-data domain mismatch and dataset size matter more than label quality — but it shifted the focus from "which boundary is hard?" to "why didn't the model learn any boundary?"

---

## AI Usage

**1. Label stress-testing (before annotation)**
Before annotating anything, I gave Claude the three label definitions and both edge case rules and asked it to generate 10 boundary posts, targeting the neutral/bearish and bullish/neutral edges. Several of the generated posts — particularly those written in a factual analysis register with a buried directional implication — couldn't be cleanly labeled with my original definitions. That prompted me to add the explicit "dominant stance" rule and the "intended sentiment for sarcasm" rule to `planning.md` before I started annotating.

**2. Annotation pre-labeling (during data collection)**
Claude pre-labeled batches of 20–30 posts using the label definitions and raw post text. I reviewed and manually corrected every pre-assigned label before using it. About 30% of pre-labels got changed — most commonly, posts Claude called `bearish` were relabeled `neutral` when the bearish framing wasn't the dominant conclusion, and posts Claude called `neutral` were relabeled `bearish` when factual framing masked a clearly pessimistic stance.

**3. Failure pattern analysis (post fine-tuning)**
The 19 misclassified test examples were pasted into Claude and I asked it to identify common patterns. Claude flagged three: (a) factual-register posts getting misclassified as neutral regardless of stance, (b) posts with negative opening language getting mislabeled bearish even when bullish overall, and (c) confidence scores clustering near 0.36–0.37 across all predictions suggesting poor calibration. All three were verified by manually re-reading the misclassified examples. One suggested pattern — that short posts were disproportionately wrong — didn't hold up on review; post length didn't appear to predict misclassification rate in this test set.

---

## Repository Contents

```
├── README.md                          ← This file
├── planning.md                        ← Design decisions and spec
├── reddit_investing_sentiment_data.csv ← 200 labeled examples
├── evaluation_results.json            ← Accuracy metrics for both models
├── confusion_matrix_1.png             ← Confusion matrix for fine-tuned model
└── Copy_of_ai201_project3_takemeter_starter_clean.ipynb ← Colab notebook
```
