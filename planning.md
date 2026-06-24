# Project 3 — TakeMeter: Text Classification

---

## 1. Community Selection

**Community:** r/nba

I chose r/nba because it contains a wide mix of discussion styles all in one place. Some posts are very data-driven and analytical, some are strong opinions about players or teams, and others are immediate emotional reactions after games. The community is also very active, which means there is enough variety in posts to build a meaningful dataset for training a model.

I am not a long-time active poster in this community, but that is fine — this project does not require understanding basketball. The task only requires identifying the **style of writing** in each post, not whether the content is correct.

The community is interesting because different types of discussion happen in the same place. Distinguishing between analysis, opinions, and reactions can help understand how fans communicate and respond to events.

**Why the discourse is varied enough:**

More substantive posts usually:
- include statistics (like shooting percentages, win rates, or advanced metrics)
- compare players or teams over time
- explain why something happened instead of just stating an opinion
- use structured reasoning or multiple supporting points

Less substantive posts usually:
- rely on strong opinions without evidence (e.g., "Player X is overrated")
- are purely emotional reactions to a game outcome
- make claims without explaining or supporting them
- focus on excitement, frustration, or memes rather than analysis

The key difference is whether the post is trying to **explain something using evidence and reasoning**, or simply **express an opinion or emotion**.

---

## 2. What This Project Measures

This project does NOT measure whether statements about basketball are correct.

Instead, it measures **how people communicate**. The goal is to classify posts based on writing style:
- whether they explain something
- whether they give a strong opinion
- whether they react emotionally

---

## 3. Label Taxonomy

### Label 1: `analysis`

#### Original definition
Posts that try to explain something using reasoning, evidence, reasoning, comparisons, or statistics.

**How to recognize it:**
- explains "why" something happened
- uses comparisons, statistics, or structured reasoning
- sounds logical or informative
“analysis requires explanation of mechanisms, not just supporting evidence or stats used to justify a claim”

#### Revised definition
> **Why it changed:** The original definition depended on tone and vibes ("uses reasoning"), which overlap constantly in NBA discourse. A comment like *"He's overrated because he can't defend switches"* contains a because-clause (sounds like reasoning) but is primarily a judgment. Definitions based on style alone could not distinguish the two reliably. The revised definition is based on **structure**: does the explanation stand on its own once opinion words are removed?

A comment is `analysis` ONLY IF it explains why something happens, breaks down cause/effect, compares mechanisms, or uses stats or tactical logic — **AND** if you remove opinion words, the sentence still makes sense as a standalone explanation.

**How to recognize it:**
- explains causes or effects
- tactical breakdown or statistical reasoning
- comparisons of mechanisms
- explanation-first: the reasoning stands on its own without judgment words

**Clarifying examples:**
- *"The offense struggles because spacing collapses when the center switches out."*
- *"They lost due to turnover rate and poor transition defense."*
- *"His efficiency drops when defended by stronger wings."*

**Example A (Reddit):** *"Jokic's assist numbers this season aren't just impressive — they reflect a fundamental shift in how Denver runs their half-court offense. When you compare his hockey assists to last year, you can see teams are collapsing earlier on the drive, which is opening up the kick-out corner three at a much higher rate."* 
**Example B (Reddit):** *"People keep sleeping on the Wolves' defensive rating improvement. They went from 26th last season to 8th this year. That's not luck — Gobert's rotations and their switch-heavy scheme have cut opponent paint points by nearly 12 per game."* 

---

### Label 2: `hot_take`

#### Original definition
Posts that strongly state an opinion without supporting it (without providing structured reasoning).

**How to recognize it:**
- strong judgment words (good/bad/best/worst/overrated)
- confident, assertive statement
- does not explain reasoning clearly
- a fact may be present, but it is used to support an opinion, not to explain something

#### Revised definition
> **Why it changed:** The original definition ("strong opinion without structured reasoning") fails on comments that include a because-clause. It's unclear whether the reasoning makes it analysis or the opinion makes it a hot take. The revised definition resolves this: a `hot_take` is opinion-first, meaning the reasoning only exists to support the judgment and is not independently meaningful.

A comment is `hot_take` IF its primary goal is to convince or judge, AND if you remove the opinion words, the sentence becomes meaningless or incomplete. Reasoning may be present, but it exists only to support the judgment — not as a standalone explanation.

**How to recognize it:**
- judgment is the primary goal
- includes ranking, criticism, or predictions (overrated, trash, GOAT, will never, better/worse than)
- reasoning is optional or decorative — it does not stand alone without the opinion
- opinion-first: removing judgment words leaves nothing meaningful

**Clarifying examples:**
- *"He's overrated because he can't defend switches."* → remove "overrated" → "He can't defend switches." → incomplete floating fact → `hot_take`
- *"This team is trash because they always lose."* → remove "trash" → no longer captures intent → `hot_take`
- *"Wemby will never win a ring."* → judgment-only prediction → `hot_take`

**Example A (Reddit):** *"LeBron is the most overrated player in NBA history. Championships with three different superteams and people still act like he's the GOAT. Jordan never needed to join his rivals to win."* *(to be replaced with a real Reddit post)*

**Example B (Reddit):** *"Kyrie is a top 5 point guard of all time and I'm tired of pretending otherwise. His handles are unguardable when he's locked in. The stats don't matter — you watch him and you just know."* *(to be replaced with a real Reddit post)*

---

### Label 3: `reaction`
Posts that express emotion about something that just happened / primarily express emotion about a game, event, or result. Emotion outweighs thinking. *(Definition unchanged.)*

**How to recognize it:**
- anger, excitement, sadness, or surprise
- short messages
- emotional tone with little to no argument
- memes, hype, or frustration
- no explanation

**Example A:** *"BRO WHAT WAS THAT CALL?? That was clearly a foul they just stole the game from us I'm actually shaking 😭😭"* 

**Example B:** *"I cannot believe we just watched that. Down 20 in the 4th and they came back. This team never quits. I'm crying actual tears rn"*

---

## 4. Decision Rules for Ambiguous Cases

### The "Remove the Opinion Word" Test

For every borderline comment between `hot_take` and `analysis`, apply this two-step test:

**Step 1:** Identify and remove judgment words (overrated, trash, GOAT, not goat, will never, better than, worse than, best, worst, etc.)

**Step 2:** Check whether the remaining sentence still makes sense as a standalone explanation.
- If it **still makes sense** → label as **analysis**
- If it **loses meaning or becomes incomplete** → label as **hot_take**

### Side-by-side examples:

| Comment | Remove opinion | Result | Label |
|---|---|---|---|
| "The Lakers are poorly constructed because they lack shooting." | Remove "poorly constructed" → "They lack shooting." | Still meaningful | `analysis` |
| "The Lakers are trash because they lack shooting." | Remove "trash" → "They lack shooting." | No longer captures intent | `hot_take` |
| "They lose because of turnover rate and spacing issues." | No opinion words to remove | Standalone explanation | `analysis` |
| "They are the worst team in the league." | Remove "worst" → nothing meaningful left | No explanation existed | `hot_take` |

### Tiebreaker rules:
- If the post explains something using reasoning and no judgment words are primary → **analysis**
- If the post is a strong opinion and reasoning is decorative → **hot_take**
- If the post is mostly emotion → **reaction**
- A single fact or number used to support a judgment (not to explain something) → **hot_take**

### Worked example:

Post: *"Player X is overrated — their win rate is below .500."*

Apply the test: remove "overrated" → "their win rate is below .500." → this is a floating fact, not an explanation of why or how something works.

**Final decision: `hot_take`**

Reason: The stat exists only to support the judgment "overrated." Removing the opinion word leaves an incomplete thought, not a meaningful explanation.

---

## 5. Expected Challenges

- Some posts will mix emotion and opinion
- Some posts may include facts without explanation
- Some posts may be unclear at first

To handle this, I will follow the decision rules consistently and prioritize **writing style** over subject knowledge.

---

## 6. Data Collection Plan
Data will be collected from public r/nba discussion threads.

**Sources:**
- Game threads (search: "Game Thread", sorted by Top) → collect live reactions and emotional posts
- Opinion threads (search: "unpopular opinion", "hot take", "is X overrated") → collect strong opinion-based posts
- Analysis threads (search: "post game analysis", "breakdown", "advanced stats") → collect more structured posts

**Collection method — 10-Comment Sweep:**
1. Open a thread
2. Scroll until a dense comment section appears
3. Select 10–15 comments at once
4. Copy into CSV / Google Sheets and label immediately
5. Move to the next thread without deep reading

**Volume target:**
- 1 batch ≈ 10–15 comments
- 20 batches ≈ 200 labeled examples
- Estimated time: 60–90 minutes

**If a label is underrepresented:**
If after 200 examples one label has fewer than 50 examples, I will run an additional targeted sweep using search terms most likely to surface that label type, continuing until each label has at least 50 examples.

**Data quality rule:**
Skip threads where comments are repetitive memes, lack meaningful variation, or are overly shallow. The goal is diversity across labels, not just volume.

---


## 7. Evaluation Metrics
*(Question: Which metrics will you use to evaluate your model, and why are those the right ones for this specific task? Accuracy alone is not enough — explain what else you need and why.)*

To evaluate the model, I will not rely on accuracy alone because accuracy can hide important mistakes (for example, the model could perform well on one label but fail on others).

Instead, I will use the following metrics:

### 1. Accuracy
This measures the percentage of correct predictions overall.
It gives a general sense of performance but does not show where the model is failing.

### 2. Precision (per class)
Precision measures how often the model is correct when it predicts a specific label.

This is important because it helps detect when the model is overusing a label (for example, predicting “reaction” too often).

### 3. Recall (per class)
Recall measures how many real examples of a label the model successfully finds.

This is important because it shows whether the model is missing certain types of posts (for example, failing to detect “analysis” posts).

### 4. F1 Score (per class)
F1 score balances precision and recall.
This is useful because it gives a single score that reflects both false positives and false negatives.

### 5. Confusion Matrix
A confusion matrix will be used to visually inspect which labels the model confuses most often (e.g., hot_take vs reaction).

### Why these metrics are appropriate:
This task involves subjective language classification, where different labels can be similar in wording. Therefore, precision, recall, and F1 score are necessary to understand model behavior beyond simple accuracy.

## 8. Definition of Success
*(Question: What performance would make this classifier genuinely useful? What would you accept as "good enough" for deployment in a real community tool?)*

---

A successful model is one that consistently distinguishes between writing styles in a way that would be useful in a real community analysis tool.

### Minimum acceptable performance:
- Around 80% overall accuracy
- Balanced performance across all three classes (no label below ~70% F1 score)
- A confusion matrix that does not show extreme bias toward one label

### Good performance:
- 85–90% accuracy
- All labels have reasonably similar F1 scores
- Few consistent misclassification patterns

### Strong (deployment-level) performance:
- 90%+ accuracy
- Clear separation between:
  - analysis (reasoning-based posts)
  - hot_take (opinion-based posts)
  - reaction (emotion-based posts)
- Minimal confusion between hot_take and reaction

### What makes it “useful” in practice:
The classifier would be considered useful if it can reliably categorize posts in real time to help summarize discussion trends in a community (e.g., how much of a discussion is analytical vs emotional vs opinion-based).


## Hard Edge Cases

### 1. Brunson vs SGA comparison post
**Post:** Someone explain to me how so many people are convinced that Brunson will be unstoppable this series after we watched a better player in SGA struggle against the spurs?

**Why it was confusing:**
This post includes reasoning (comparison to SGA) but is also clearly argumentative and challenges an opinion. It sits between analysis and hot_take.

**Final label:** hot_take


### 2. "WEMBY NEXT JORDAN" discourse reaction
**Post:** These posts really are garbage lmao WEMBYS THE NEXT JORDAN TO WEMBY WILL NEVER WIN RINGS in a span of less than 1 week.

**Why it was confusing:**
This post is both emotional and opinionated. It reacts to shifting narratives rather than explaining basketball concepts, making it unclear whether it is reaction or hot_take.

**Final label:** reaction


### 3. Long Knicks vs Spurs argument post
**Post:** On one hand the Knicks are one of the hottest teams in the playoffs ever... (full post)

**Why it was confusing:**
This post mixes statistical reasoning, historical comparisons, and predictions. It blends analysis and hot_take elements throughout.

**Final label:** analysis

---

## 9. AI Tool Plan

*(Question: How will you use AI tools for label stress-testing, annotation assistance, and failure analysis?)*
Label Stress Testing
Before annotation, I will ask an AI tool to generate posts that sit between analysis and hot_take. If I cannot confidently label those examples, I will revise my label definitions before collecting data.
annotation assistance
I may use ChatGPT to suggest labels for a small batch of examples. However, every label will be manually reviewed before being added to the dataset. Any AI-assisted labels will be documented in my project notes.
(AI Usage Disclosure)
I used ChatGPT to suggest preliminary labels for some examples during dataset annotation. All AI-generated labels were manually reviewed and corrected when necessary before being added to the final dataset. Final labeling decisions were made by me using the definitions and decision rules described in planning.md.
Failure Analysis
After training, I will provide misclassified examples to an AI tool and ask it to identify common patterns. I will verify these patterns manually by reviewing the original examples and checking whether the labels or definitions should be improved.