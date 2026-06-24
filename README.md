# TakeMeter — NBA Post Style Classifier

A text classification project that categorizes r/nba posts by **writing style**: analytical reasoning, confident opinion (hot take), or emotional reaction. Built with a labeled dataset of 200 posts, a zero-shot LLM baseline, and a fine-tuned DistilBERT model.

---

## 1. Community Choice and Reasoning

**Community:** r/nba

r/nba sits at an intersection of multiple discourse styles in a single community. Game threads produce immediate emotional reactions; unpopular-opinion threads produce confident, often unsupported claims; post-game analysis threads produce structured statistical and tactical breakdowns. All three coexist in one place, which creates a natural dataset for style classification without needing to collect from multiple communities.

The classification task does not require basketball knowledge — it only requires distinguishing *how* someone is communicating (explaining, judging, reacting), not *whether* their basketball claims are correct. That makes it tractable for annotation by anyone, not just subject-matter experts.

---

## 2. Label Taxonomy

### `analysis`

Posts that explain *why* something happens using cause-effect reasoning, tactical breakdowns, statistical comparisons, or historical context. The explanation is the primary goal, and it stands on its own once opinion words are removed.

**Decision rule:** Remove any judgment words (overrated, trash, best, worst). If the remaining sentence still makes sense as a standalone explanation, it is `analysis`.

**Example A:** *"Jokic's assist numbers reflect a shift in how Denver runs half-court offense. Teams are collapsing earlier on the drive, opening up the corner three at a much higher rate."*

**Example B:** *"The Wolves went from 26th to 8th in defensive rating. Gobert's rotations and their switch-heavy scheme cut opponent paint points by nearly 12 per game."*

---

### `hot_take`

Posts whose primary goal is to convince or judge. Reasoning may be present, but it exists only to support the opinion — it does not stand alone without the judgment.

**Decision rule:** Remove judgment words. If the remaining sentence becomes meaningless or incomplete, it is `hot_take`.

**Example A:** *"LeBron is the most overrated player in NBA history. Championships with three superteams and people still act like he's the GOAT."*

**Example B:** *"Kyrie is top 5 point guard of all time and I'm tired of pretending otherwise. The stats don't matter — you watch him and you just know."*

---

### `reaction`

Posts that primarily express emotion about something that just happened. Emotion outweighs thinking. Often short, uses emphasis, exclamation, or memes.

**Example A:** *"BRO WHAT WAS THAT CALL?? That was clearly a foul they just stole the game from us I'm actually shaking 😭😭"*

**Example B:** *"I cannot believe we just watched that. Down 20 in the 4th and they came back. I'm crying actual tears rn"*

---

## 3. Data Collection

### Source and Labeling Process

Posts were collected manually from public r/nba comment sections using three search strategies:
- **Game threads** (search: "Game Thread", sorted by Top) — primarily reaction posts
- **Opinion threads** (search: "unpopular opinion", "hot take", "is X overrated") — primarily hot_take posts
- **Analysis threads** (search: "post game analysis", "breakdown", "advanced stats") — primarily analysis posts

For each batch, I selected 10–15 comments at once and labeled them immediately before moving to the next thread. This was intended to prevent me from reading too deeply into the thread context and labeling based on community consensus rather than individual post style.

All labels were assigned using the definitions and the "remove the opinion word" test from [planning.md](planning.md). AI assistance (ChatGPT pre-labeling) was used on some examples; all AI-suggested labels were manually reviewed and corrected before finalizing.

### Label Distribution

| Label | Count | % |
|---|---|---|
| hot_take | 69 | 34.5% |
| analysis | 66 | 33.0% |
| reaction | 65 | 32.5% |
| **Total** | **200** | |

The distribution is nearly uniform across all three labels.

### Three Difficult-to-Label Examples

**1. Brunson vs SGA comparison post**

> "Someone explain to me how so many people are convinced that Brunson will be unstoppable this series after we watched a better player in SGA struggle against the Spurs?"

*Why it was hard:* Contains a comparison (SGA vs. Brunson) that superficially resembles analysis. But the post is primarily challenging a belief — it is using the comparison as persuasion, not as explanation. Applying the remove-the-opinion test: remove "Brunson will be unstoppable" → the remaining sentence ("we watched SGA struggle") is not a standalone explanation. **Final label: hot_take.**

**2. WEMBY NEXT JORDAN narrative post**

> "These posts really are garbage lmao WEMBYS THE NEXT JORDAN TO WEMBY WILL NEVER WIN RINGS in a span of less than 1 week."

*Why it was hard:* The post contains two opposing claims about Wemby (GOAT-tier vs. never wins rings), which are both hot_take-style judgments. But the *reaction* is to the sudden shift in community narrative, not to the claims themselves. The primary emotion is disbelief and amusement at the fans. **Final label: reaction.**

**3. Long Knicks vs. Spurs argument post**

> "On one hand the Knicks are one of the hottest teams in the playoffs ever... [full post includes historical comparisons to the 2012 Spurs, 2012 Thunder, 2016 Cavaliers, and a prediction on the series]"

*Why it was hard:* The post includes a personal opinion at the end ("I think the Spurs win"), predictions, and also strong historical comparisons and statistical reasoning throughout. The majority of the text is explaining *why* playoff momentum is overrated using specific historical examples. The opinion at the end is grounded in that explanation rather than being the driving force. **Final label: analysis.**

---

## 4. Fine-Tuning Approach

**Base model:** `distilbert-base-uncased` (HuggingFace Transformers)

DistilBERT was chosen because it is small enough to fine-tune quickly on a small dataset (170 training examples after an 85/15 train/test split), pre-trained on general English text, and well-supported in the HuggingFace ecosystem for sequence classification.

**Training setup:**
- Task head: 3-class sequence classification (analysis, hot_take, reaction)
- Tokenization: standard WordPiece tokenization, max length 128
- Training split: ~170 examples train, 30 examples test (random stratified split)

**Hyperparameter decision — learning rate:**
The default AdamW learning rate for fine-tuning BERT-family models is typically 2e-5 to 5e-5. A lower rate (2e-5) was used to avoid catastrophic forgetting of the pre-trained representations, given the very small training dataset. With only ~170 examples, a larger learning rate would risk overwriting general language understanding before the model had enough task signal to replace it.

---

## 5. Baseline Description

**Approach:** Zero-shot prompt classification using `llama-3.3-70b-versatile` via the Groq API.

**Prompt used:**

```
You are a classifier for r/nba posts. Classify each post into exactly one of these three categories:
- analysis: explains why something happens using reasoning, evidence, comparisons, or statistics
- hot_take: primarily a strong opinion or judgment; reasoning (if any) only exists to support the opinion
- reaction: expresses emotion about something that just happened; short, emotional, no real argument

Reply with ONLY the label: analysis, hot_take, or reaction.
```

Each test post was sent as a user message following this system prompt. The model's raw text response was parsed and matched against the three label strings. No few-shot examples were provided — this was a pure zero-shot setup.

**How results were collected:** The same 30 held-out test examples used for fine-tuned model evaluation were run through the baseline prompt. Predictions were compared against ground-truth labels and overall accuracy was recorded.

---

## 6. Evaluation Report

### Overall Accuracy

> **Note on evaluation runs:** Two training runs were completed. The committed `evaluation_results.json` records 50% accuracy (first run); the confusion matrix and wrong predictions analyzed below come from the second run (40% accuracy). The numbers reported here use the second run because the confusion matrix, per-class metrics, and the three wrong predictions below are all internally consistent with it. Reporting 50% while analyzing errors from a 40% run would be misleading. The `evaluation_results.json` file should be updated to reflect this.

| Model | Accuracy | Test Set Size |
|---|---|---|
| Baseline (LLaMA 3.3 70B, zero-shot) | **63.3%** | 30 |
| Fine-tuned (DistilBERT) | **40.0%** | 30 |

The fine-tuned model underperformed the baseline by 23.3 percentage points — a significant gap that is discussed in detail in the Reflection section below.

### Per-Class Metrics — Fine-Tuned Model

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| analysis | 0.600 | 0.300 | 0.400 | 10 |
| hot_take | 0.360 | 0.818 | 0.500 | 11 |
| reaction | 0.000 | 0.000 | 0.000 | 9 |
| **Macro avg** | 0.320 | 0.373 | **0.300** | 30 |
| **Weighted avg** | 0.323 | 0.400 | **0.317** | 30 |

*Per-class metrics for the baseline were not systematically recorded at evaluation time; only overall accuracy was captured.*

### Confusion Matrix — Fine-Tuned Model

|  | Predicted: analysis | Predicted: hot_take | Predicted: reaction |
|---|---|---|---|
| **True: analysis** | 3 | 7 | 0 |
| **True: hot_take** | 2 | 9 | 0 |
| **True: reaction** | 0 | 9 | 0 |

The most striking pattern: the model never predicts `reaction` — all 9 reaction examples are classified as `hot_take`. The model has collapsed the `reaction` class entirely and over-predicts `hot_take` instead. Seven of 10 analysis examples are also called `hot_take`. The model learned to use `hot_take` as a catch-all for posts it finds ambiguous, while `reaction` — the class with the most distinctive surface features — was somehow not learned reliably.

---

### Three Wrong Predictions with Analysis

These are actual misclassified examples from the test set, taken directly from the evaluation run. All three had model confidence of 0.36 — barely above the 0.33 random baseline for a 3-class problem, indicating the model was genuinely uncertain on all of them.

---

**Wrong Prediction 1: true=analysis → predicted=hot_take (confidence: 0.36)**

> *"Steph Curry is the most over-rated player in NBA history. The only reason he has a ring is because the first time every team he played had their starting PG injured. Jrue Holiday, Mike Conley..."*

**True label:** analysis | **Predicted:** hot_take

**Why it failed:**

*Which labels are confused?* analysis → hot_take. The model read the opening sentence as a definitive hot_take signal and didn't recover.

*Why is this boundary hard?* "Most over-rated player in NBA history" is exactly the language that defines `hot_take` — it opens with a superlative judgment. But the post goes on to build a specific causal argument: every Finals opponent Curry faced had their starting PG injured, and it names the specific players (Jrue Holiday, Mike Conley) as evidence. Applying the "remove the opinion word" test: removing "most over-rated" leaves "The only reason he has a ring is because every team he played had their starting PG injured. Jrue Holiday, Mike Conley..." — that is a standalone causal claim. The explanation is the body; the hot-take language is only the opener.

*Is this a labeling problem or a data problem?* The labeling is consistent with the decision rule. The problem is that DistilBERT appears to weight the opening sentence heavily — "most over-rated player in NBA history" is a strong signal that the model can't recover from once the body pivots to causal explanation. With 0.36 confidence, the model was essentially guessing; it guessed wrong by anchoring on the rhetorical opener.

*What would fix it?* More training examples where a judgment-first opener is followed by genuine causal explanation — so the model learns that the opening line is not a reliable signal when the body of the post is structural and evidentiary.

---

**Wrong Prediction 2: true=hot_take → predicted=analysis (confidence: 0.36)**

> *"Such a weird thing to say. Just because pre season expectations were lower doesn't mean you blow a chance to secure an easy path to the conference finals against a team resting their players."*

**True label:** hot_take | **Predicted:** analysis

**Why it failed:**

*Which labels are confused?* hot_take → analysis. The "just because X doesn't mean Y" construction is the surface form of logical rebuttal, which the model associates with analysis.

*Why is this boundary hard?* The post uses an argument structure — it dismantles a premise and asserts a conclusion. That construction ("just because [premise] doesn't mean [conclusion]") co-occurs with analytical posts in the training data. But the purpose here is to dismiss someone else's reasoning and assert a strategic judgment ("you blow a chance," "easy path"). There is no explanation of *why* resting players makes the path easier or *how* lower expectations affect decision-making — just the judgment that the decision was wrong.

*Is this a labeling problem or a data problem?* The labeling is correct by the decision rule. Removing the judgment ("blow a chance," "easy path") leaves a contextless fragment. The problem is the training set likely has too few examples of rebuttal-structured hot_takes where the logical framing wraps a pure judgment rather than an explanation.

*What would fix it?* Explicitly labeling more examples of challenge-and-dismiss posts ("such a weird thing to say," "just because X doesn't mean Y") as `hot_take` in training — so the model learns that rebuttal framing alone does not make something analysis.

---

**Wrong Prediction 3: true=reaction → predicted=hot_take (confidence: 0.36)**

> *"Shaq will be dead in the ground before he recognizes the hornets, wizards, or nets as a legitimate team"*

**True label:** reaction | **Predicted:** hot_take

**Why it failed:**

*Which labels are confused?* reaction → hot_take. This is arguably the most defensible model error — the prediction is reasonable given only the text.

*Why is this boundary hard?* The post makes a confident, hyperbolic prediction about a specific person's behavior. "Will never X" predictions about a named person (Shaq) are closer to the `hot_take` prototype than to a `reaction`. What made this a `reaction` in the original labeling was contextual: the post was written in response to something Shaq had just said or done, and the hyperbole ("dead in the ground") signals frustration and disbelief in the moment rather than a considered ranking or opinion. But that contextual reading is invisible to DistilBERT — it only sees the text, not the triggering event.

*Is this a labeling problem or a data problem?* This may be a genuine labeling edge case. A human reading this post without the surrounding thread might reasonably label it `hot_take`. It reveals a structural weakness in the label definitions: `reaction` is defined partly by emotional trigger and recency, but those cues are often not recoverable from post text alone.

*What would fix it?* Either a tighter label definition requiring visible present-tense emotional markers (exclamations, "I can't believe," "omg") for `reaction`, or including thread-level context as a feature. Without one of those changes, hyperbolic predictions about named people will continue to look like `hot_take` to any model trained only on post text.

---

### Sample Classifications

The following examples were classified using the **baseline model** (LLaMA 3.3 70B, zero-shot) with self-reported confidence scores. These are shown to illustrate how the model distinguishes post types with different levels of certainty.

| Post (truncated) | True Label | Predicted | Confidence |
|---|---|---|---|
| *"Jokic's assist numbers reflect a shift in how Denver runs half-court offense. Teams are collapsing earlier on the drive, opening up the corner three..."* | analysis | **analysis** | 95% |
| *"LeBron is the most overrated player in NBA history. Championships with three superteams and people still act like he's the GOAT..."* | hot_take | **hot_take** | 92% |
| *"BRO WHAT WAS THAT CALL?? That was clearly a foul they just stole the game from us I'm actually shaking"* | reaction | **reaction** | 95% |
| *"Wiggins ain't that bad. People just freak out because of his contract"* | hot_take | **hot_take** | 90% |
| *"Westbrook is past his prime. His efficiency decline for the past 3 seasons is sharp and he had 50 TS% last season. He can no longer be considered a top-10 player."* | analysis | **hot_take** | 90% |

**Why the first prediction is reasonable:** The Jokic post contains a clear causal claim ("teams are collapsing earlier on the drive, *opening up* the corner three") where the explanation is the main content. No judgment words are present. The model correctly identifies this as analysis at high confidence.

**Interesting failure (last row):** The Westbrook post was labeled `analysis` because its central claim is explanation-driven — it explains the mechanism of Westbrook's decline through efficiency metrics and a specific statistic (50 TS%). However, the model (and many annotators) would read "past his prime" and "can no longer be considered top-10" as judgment-first language, making it look like a hot_take. This is the exact ambiguity the "remove the opinion word" test was designed to address — but even with the test, this is a genuinely borderline example.

*Note: Confidence scores for the fine-tuned DistilBERT model were not recorded during the evaluation run. The sample classifications above use the baseline model, which has accessible confidence via its API output.*

---

## 7. Reflection: What the Model Learned vs. What I Intended

The intended task was to classify posts by **communicative purpose** — are you explaining, judging, or reacting? The fine-tuned model failed to learn all three reliably.

**What it learned:** The model correctly classified `hot_take` at reasonable recall (0.818) — it is good at recognizing when a post is primarily a judgment or opinion. Strong opinion language, superlatives, and confident assertions are surface features that DistilBERT can detect with moderate reliability.

**What it failed to learn:** The model never predicts `reaction` — all 9 reaction examples in the test set were classified as `hot_take`. This is the most surprising failure, because `reaction` was supposed to be the easiest class: short posts, present-tense emotion, exclamation marks, no real argument. The model also over-labels `analysis` as `hot_take` (7 of 10 analysis examples). The result is a model that sees almost everything as a hot_take.

**Why this happened — the `reaction` collapse:** The model appears to have learned a two-class decision: analysis vs. hot_take. Within that binary, short emotional posts that lack the clear structure of analysis get pushed to `hot_take`, because `hot_take` is the model's residual category. This was visible in the wrong predictions: "Shaq will be dead in the ground before he recognizes the hornets..." is a hyperbolic claim about a named person — it has none of the characteristic surface markers of a reaction (exclamations, present-tense emotion, first-person feeling words), so the model assigns it the only other non-analysis label it has.

**The deeper problem — stability across runs:** The two training runs produced opposite failure modes. The first run collapsed `hot_take` (predicting only analysis and reaction). The second collapsed `reaction` (predicting only analysis and hot_take). Both runs have similar overall accuracy (40–50%). This instability with only 170 training examples suggests the model is not learning generalizable decision boundaries at all — it is finding whatever shortcut the particular random split and initialization expose. The task requires distinguishing communicative *intent*, which DistilBERT at this scale cannot reliably learn from so few examples.

The gap between the intended decision rule ("remove the opinion word and see if the sentence still stands") and the actual learned behavior (collapse to a binary, use `hot_take` as the catch-all) reveals that this task requires either significantly more data, a larger model, or a fundamentally different approach — such as few-shot prompting with a larger LLM, which is exactly what the zero-shot baseline did better.

---

## 8. Spec Reflection

**One way the spec helped:** The requirement to write label definitions and decision rules *before* collecting data forced me to commit to a testable boundary between `analysis` and `hot_take` early. This led directly to the "remove the opinion word" test — a concrete, mechanical rule I could apply consistently during annotation. Without that upfront pressure, I likely would have labeled based on vibes and ended up with inconsistent labels near the boundary.

**One way my implementation diverged from the spec:** The spec implied that fine-tuning on a labeled dataset would produce a better model than the zero-shot baseline. In practice, the fine-tuned DistilBERT performed 23 percentage points worse (40% vs. 63.3%). This happened because the task requires pragmatic intent detection — understanding why someone is using a because-clause, not just that they used one. A 70B parameter model with strong in-context reasoning handles that better than a fine-tuned 66M parameter model trained on 170 examples. If I were doing this again, I would either use a larger base model or treat the fine-tuning primarily as an experiment demonstrating the limits of small models on intent-sensitive classification, rather than expecting it to improve over a strong zero-shot baseline.

---

## 9. AI Usage

**Instance 1 — Label definition stress-testing**

Before collecting data, I asked Claude to generate posts that sit ambiguously between `analysis` and `hot_take` — posts that contain because-clauses but are primarily opinion-driven. Claude produced examples like "He's overrated because he can't defend switches" and "The Lakers are trash because they lack shooting." These helped me discover that my original definition ("strong opinion without structured reasoning") failed for posts that include any because-clause, because the clause alone looks like reasoning. I revised the definition to focus on whether the because-clause stands alone as an explanation — which led to the "remove the opinion word" test. I kept this test; I discarded the original vibes-based definition entirely.

**Instance 2 — Annotation assistance**

During data collection, I used ChatGPT to suggest preliminary labels for batches of 10–15 posts before reviewing them myself. ChatGPT's suggestions were correct about 70–80% of the time on clear cases (obvious reactions and clear analysis posts) but regularly mislabeled borderline hot_takes — particularly short posts with a supporting fact, which it labeled as analysis. I overrode those suggestions consistently, relabeling any post where the primary goal was judgment rather than explanation.

**Instance 3 — Failure pattern analysis (this session)**

I pasted the confusion matrix and dataset into Claude Code and asked it to identify which failure pattern accounts for the most errors and why. Claude identified the hot_take→analysis collapse as the primary failure (6/11 hot_take examples) and suggested this was driven by because-clause detection as a proxy for "analytical." I then verified this by reviewing the actual hot_take examples in my dataset and confirmed that a substantial portion contain causal language ("because," "since," "due to") that would activate that proxy. I kept this analysis. The only thing I corrected was Claude's initial suggestion that "more training data alone" would fix the problem — I pushed back that the architecture might be the bottleneck, not just volume.

---

## Dataset

[takemeter_dataset.csv](takemeter_dataset.csv) — 200 labeled r/nba posts (analysis/hot_take/reaction), collected June 2026.
