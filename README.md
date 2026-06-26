# TakeMeter: r/leagueoflegends Discourse Classifier

## Project Overview

TakeMeter is a text classifier designed to label Reddit posts and comments from **r/leagueoflegends** by discourse type. The goal is not to decide whether a post is “good” or “bad” in a vague sense, but to separate different kinds of community speech: reasoned analysis, unsupported hot takes, immediate reactions, and intentionally unserious meme/shitpost content.

I chose **r/leagueoflegends** because it is a large, active, text-heavy community with a wide range of discourse styles. Users discuss champion balance, patch notes, esports, ranked experiences, lore, Riot decisions, and community drama. This makes it a good classification task because posts often look similar on the surface while serving very different functions: one post may be a serious balance argument, another may be a frustrated rant, another may be a quick emotional reaction, and another may be a joke using the same League vocabulary.

---

## Label Taxonomy

I used four final labels: `analysis`, `hot_take`, `reaction`, and `meme_shitpost`.

### `analysis`

A post should be labeled `analysis` if it makes a structured argument about gameplay, balance, esports, lore, or strategy using specific evidence such as patch changes, champion interactions, matchup details, statistics, draft context, or in-game reasoning.

Examples:

1. “The reason Jinx became stronger this patch is not just the direct buff. With early dragon fights lasting longer and enchanters being picked more often, she gets more time to scale and reset in teamfights.”

2. “T1’s draft lost because they had no reliable way to start fights. Their only hard engage was on Rell, and once she fell behind, the poke comp could never force objectives on their own terms.”

### `hot_take`

A post should be labeled `hot_take` if it makes a bold, broad, or controversial claim with little support, usually stated confidently as if it is obvious or final.

Examples:

1. “This is the worst Worlds meta of all time and anyone pretending otherwise is coping.”

2. “ADC players are the most entitled players in the game and the role has been broken for years.”

### `reaction`

A post should be labeled `reaction` if it is an immediate emotional response to a specific match, play, patch note, announcement, personal experience, or community event, with little attempt to make a broader argument.

Examples:

1. “That Faker Azir shuffle was insane. I actually screamed.”

2. “No way they buffed Yuumi again. I cannot believe this is real.”

### `meme_shitpost`

A post should be labeled `meme_shitpost` if its main purpose is humor, irony, exaggeration, parody, or absurdity rather than sincere analysis, complaint, or argument.

Examples:

1. “My jungler ganked my lane once and I immediately knew I was in loser’s queue because Riot would never let me experience happiness.”

2. “BREAKING: Yuumi players have discovered the keyboard. Analysts are calling this the biggest mechanical development since Faker’s debut.”

---

## Data Collection and Labeling Process

I collected public posts and comments from **r/leagueoflegends**, including discussion threads about champion balance, patch changes, esports, ranked experiences, Riot decisions, lore/media discussion, and general community reactions. I intentionally sampled across different thread types so the dataset would not only represent one kind of League discourse.

During labeling, I assigned each item one final label based on the post’s main function. I also used a temporary `skip` label for items that were not really community takes, such as plain news links, bot/mod text, survey posts, art-only posts, recruiting posts, or simple help requests. These `skip` rows were removed before model training.

Final collected label distribution before removing `skip`:

| Label           | Count |
| --------------- | ----: |
| `analysis`      |   182 |
| `hot_take`      |   109 |
| `reaction`      |   101 |
| `meme_shitpost` |    40 |
| `skip`          |   111 |

Final training/evaluation dataset after removing `skip`:

| Label           |   Count |
| --------------- | ------: |
| `analysis`      |     182 |
| `hot_take`      |     109 |
| `reaction`      |     101 |
| `meme_shitpost` |      40 |
| **Total**       | **432** |

The dataset is somewhat imbalanced, especially because `meme_shitpost` is much smaller than the other classes. This imbalance became important during evaluation because the models had a harder time learning the minority class.

---

## Difficult Labeling Examples

Some examples were difficult because League comments often combine sincere arguments, frustration, humor, and immediate reactions in the same post. I used the rule that the final label should reflect the post’s main function, not just its topic or tone.

| Example                                                                                                                                                                                                                                                                                | Possible Labels              | Final Label     | Decision                                                                                                                                                                               |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------- | --------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| “Riot has no idea what to do with Zeri. She is either pick/ban in pro or completely useless in solo queue, and somehow every rework makes the problem worse.”                                                                                                                          | `analysis` / `hot_take`      | `hot_take`      | I labeled this as `hot_take` because it points toward a real balance issue but does not provide enough specific patch history, statistics, or gameplay reasoning to count as analysis. |
| “That Baron throw was disgusting. They had no vision in river, burned Maokai ult early, and still walked in one at a time like solo queue players.”                                                                                                                                    | `reaction` / `analysis`      | `analysis`      | I labeled this as `analysis` because, even though the tone is emotional, the post gives specific in-game reasons for why the play failed.                                              |
| “I find it hilarious how they’ve broke Ziggs’ legs, shattered his jaw, amputated his left arm, and put a blindfold on him so he can’t possibly deal damage, but Brand’s still walking around pressing W+E on the wave and dealing 6k damage because you happened to be on the screen.” | `hot_take` / `meme_shitpost` | `meme_shitpost` | I labeled this as `meme_shitpost` because the complaint about balance is present, but the extreme cartoonish exaggeration makes humor the dominant function.                           |
| “It’s pretty clear by now that the top quest doesn’t do anything for toplaners... Meanwhile, the ADC quest is doing too much for botlaners...”                                                                                                                                         | `analysis` / `hot_take`      | `hot_take`      | I labeled this as `hot_take` because it makes a confident balance claim with some reasoning, but the evidence is mostly broad and assertive rather than specific or verifiable.        |

These examples show that the hardest boundary was often not topic-based. The hard part was deciding whether a post was actually reasoning through a claim or just presenting a confident opinion, emotional response, or joke.

---

## Baseline Model

### Baseline Description

My final baseline model was **TF-IDF + Logistic Regression**. I originally attempted to use a zero-shot LLM baseline through Groq, but I ran into free-tier API limits while trying to classify the test set. Because of that, I replaced the API-based baseline with a local baseline that was small, free, repeatable, and fast.

The final baseline used:

* `TfidfVectorizer`
* unigram and bigram features with `ngram_range=(1, 2)`
* `LogisticRegression`
* `class_weight="balanced"` to reduce the effect of label imbalance

Because this was a traditional machine learning baseline, it did **not** use a prompt. Results were collected by fitting the TF-IDF + Logistic Regression pipeline on the training split and predicting labels for the same held-out test set used to evaluate the fine-tuned model.

### Original LLM Prompt Attempt

Before switching to TF-IDF + Logistic Regression, I attempted a zero-shot LLM baseline with a prompt like this:

```text
You are classifying Reddit posts and comments from r/leagueoflegends.
Assign each post to exactly one of the following categories.

analysis: The post makes a structured argument using specific evidence or reasoning.
hot_take: The post makes a bold, broad, or controversial claim with little support.
reaction: The post is an immediate emotional response to a specific event.
meme_shitpost: The post is primarily humorous, ironic, exaggerated, absurd, or parodic.

Respond with ONLY the label name.
```

I did not use the Groq results in the final comparison because the API run was interrupted by token limits. The reported baseline below is the local TF-IDF + Logistic Regression model.

---

## Fine-Tuning Approach

The fine-tuned model used **`distilbert-base-uncased`** as the base model. I fine-tuned it as a four-class sequence classifier using the labels `analysis`, `hot_take`, `reaction`, and `meme_shitpost`.

The dataset was split into training, validation, and test sets, with the final test set containing 65 examples. The model was evaluated on the same held-out test set as the baseline.

One important hyperparameter decision was to evaluate the model using **macro F1** rather than accuracy alone. Because the dataset is imbalanced, especially for `meme_shitpost`, accuracy can hide poor performance on smaller classes. Macro F1 weights each class equally, so it better reflects whether the classifier is learning all four discourse categories instead of only the most common one.

I also kept the model lightweight by using DistilBERT rather than a larger transformer. This matched the project goal of building a small fine-tuned classifier rather than relying on a large hosted LLM.

---

## Evaluation Report

### Overall Results

| Model                                 | Accuracy | Macro F1 | Weighted F1 |
| ------------------------------------- | -------: | -------: | ----------: |
| TF-IDF + Logistic Regression Baseline |   0.5077 |   0.4466 |      0.4743 |
| Fine-tuned DistilBERT                 |   0.4462 |   0.3800 |      0.4600 |

The fine-tuned model performed worse than the baseline by about **0.0615 accuracy points**. This was a regression rather than an improvement. The baseline was not strong overall, but it was more reliable than the fine-tuned model on this small dataset.

---

## Baseline Per-Class Metrics

| Label            | Precision | Recall | F1-score | Support |
| ---------------- | --------: | -----: | -------: | ------: |
| `analysis`       |      0.52 |   0.82 |     0.64 |      28 |
| `hot_take`       |      0.30 |   0.19 |     0.23 |      16 |
| `reaction`       |      0.56 |   0.33 |     0.42 |      15 |
| `meme_shitpost`  |      1.00 |   0.33 |     0.50 |       6 |
| **Accuracy**     |           |        | **0.51** |      65 |
| **Macro avg**    |      0.59 |   0.42 |     0.45 |      65 |
| **Weighted avg** |      0.52 |   0.51 |     0.47 |      65 |

The baseline strongly favored `analysis`, predicting it 44 times out of 65 test examples. It handled many long explanation-style posts correctly, but it struggled with `hot_take` and often treated broad unsupported claims as analysis.

---

## Fine-Tuned Model Per-Class Metrics

| Label            | Precision | Recall | F1-score | Support |
| ---------------- | --------: | -----: | -------: | ------: |
| `analysis`       |      0.68 |   0.54 |     0.60 |      28 |
| `hot_take`       |      0.47 |   0.50 |     0.48 |      16 |
| `reaction`       |      0.28 |   0.33 |     0.30 |      15 |
| `meme_shitpost`  |      0.12 |   0.17 |     0.14 |       6 |
| **Accuracy**     |           |        | **0.45** |      65 |
| **Macro avg**    |      0.39 |   0.38 |     0.38 |      65 |
| **Weighted avg** |      0.49 |   0.45 |     0.46 |      65 |

The fine-tuned model improved recall for `hot_take` compared with the baseline, but it became worse overall. It especially struggled with `reaction` and `meme_shitpost`, which require recognizing tone, humor, and context rather than only topic vocabulary.

---

## Fine-Tuned Model Confusion Matrix

Rows are true labels; columns are predicted labels.

| True Label \ Predicted Label | `analysis` | `hot_take` | `reaction` | `meme_shitpost` |
| ---------------------------- | ---------: | ---------: | ---------: | --------------: |
| `analysis`                   |         15 |          3 |          7 |               3 |
| `hot_take`                   |          5 |          8 |          3 |               0 |
| `reaction`                   |          1 |          5 |          5 |               4 |
| `meme_shitpost`              |          1 |          1 |          3 |               1 |

The most important pattern is that the fine-tuned model spread predictions across all four labels, but it did not learn the boundaries cleanly. It confused `analysis` with `reaction` seven times and confused `reaction` with `hot_take` or `meme_shitpost` frequently. It also performed very poorly on `meme_shitpost`, correctly identifying only 1 out of 6 examples.

---

## Error Analysis: Specific Wrong Predictions

### 1. Hot take predicted as analysis

**Post:**

> “Nah you’re over complicating it, there aren’t any western sololaners besides caps that can carry consistently. You need to have a good adc to play though and just hope your sololaners can stay relatively even and skirmish well.”

**True label:** `hot_take`
**Predicted label:** `analysis`
**Confidence:** 0.40

This error shows the model confusing `hot_take` with `analysis`. The post uses esports vocabulary and gives a strategic-sounding explanation about team roles, ADCs, solo laners, and skirmishing. However, the claim is broad and unsupported: it does not provide match examples, statistics, or specific evidence. This boundary is difficult because many League hot takes are written in the language of analysis. To fix this, the training data would need more examples of confident but weakly supported esports claims labeled as `hot_take`.

### 2. Meme/shitpost predicted as reaction

**Post:**

> “I find it hilarious how they've broke Zigg's legs, shattered his jaw, amputated his left arm, and put a blindfold on him so he can't possibly deal damage, but Brand's still walking around pressing W+E on the wave and dealing 6k damage because you happened to be on the screen.”

**True label:** `meme_shitpost`
**Predicted label:** `reaction`
**Confidence:** 0.39

This error shows the model missing exaggerated humor. The post is about a real balance complaint, but the cartoonish violence and absurd exaggeration make it primarily a joke. The model likely focused on the complaint structure and champion names instead of recognizing that the rhetorical style is comedic. This may be partly a data problem because `meme_shitpost` had only 40 examples in the full labeled dataset and only 6 examples in the test set. More diverse meme/shitpost examples, especially ones that include balance vocabulary, would likely help.

### 3. Reaction predicted as analysis

**Post:**

> “It’s late game in soloq and most players are full build. Throughout the game, windows keeps asking my PC asked me 3-4 times to to sign into outlook and I keep cancelling it and alt-tabbing back... I literally threw because for some reason it was imperative that I sign into outlook... Literally windows diff. What’s your most rage inducing moment in soloq?”

**True label:** `reaction`
**Predicted label:** `analysis`
**Confidence:** 0.60

This error shows that long posts with many concrete details can look like `analysis` to the model even when they are really personal reactions. The post contains a detailed sequence of events, but the main purpose is to vent about a frustrating solo queue experience and invite others to share similar stories. This is a boundary problem between “detailed explanation” and “structured argument.” To fix it, I would add more long personal-experience posts labeled as `reaction` so the model learns that length and detail do not automatically mean analysis.

### 4. Analysis predicted as reaction

**Post:**

> “Brand has kind of always been busted in this mode. I get it though, as a main I think most ap augments work quite well on him. They gotta nerf his base aram stats further or something.”

**True label:** `analysis`
**Predicted label:** `reaction`
**Confidence:** 0.45

This error shows that the model sometimes treats informal tone as emotional reaction even when the post contains a basic balance argument. The post gives a reason for Brand’s strength in the mode and proposes a balance adjustment. It is not deeply evidenced, but it still makes a causal claim about AP augments and ARAM stats. This example suggests the label boundary may also be somewhat subjective: some annotators might reasonably see this as a hot take or reaction because the reasoning is brief. A tighter definition of “minimum evidence required for analysis” would help make labels more consistent.

---

## Sample Classifications

These are example posts run through the fine-tuned model.

| Post                                                                                                                                                                  | True Label      | Fine-Tuned Prediction |                            Confidence | Correct? |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- | --------------------- | ------------------------------------: | -------- |
| “Nah you’re over complicating it, there aren’t any western sololaners besides caps that can carry consistently...”                                                    | `hot_take`      | `analysis`            |                                  0.40 | No       |
| “I find it hilarious how they've broke Zigg's legs, shattered his jaw... but Brand's still walking around pressing W+E...”                                            | `meme_shitpost` | `reaction`            |                                  0.39 | No       |
| “It’s late game in soloq and most players are full build... windows asks me to SIGN INTO OUTLOOK AGAIN...”                                                            | `reaction`      | `analysis`            |                                  0.60 | No       |
| “	My understanding is that there are champions who are typically balanced towards pro play because they involve a high degree of communications between teammates which makes them worse in soloQ. The biggest example that comes to mind is Kalista and Renata.” | `analysis`      | `analysis`            | .69 | Yes      |

The correctly predicted `analysis` example is reasonable because the post defines what ability augments are and then explains a specific problem with them. Even though it has some opinionated wording, it is doing more than reacting emotionally: it gives context and makes a concrete claim about how the system works.

---

## What the Model Captured vs. What I Intended

The intended goal was to classify posts by **discourse function**: whether the post was reasoning, asserting, reacting, or joking. The models only partially learned that. Both models were better at detecting surface features like League vocabulary, post length, and whether the post contained champion or item names than they were at identifying the deeper purpose of the post.

The TF-IDF baseline mostly learned that long, topic-specific comments often correspond to `analysis`. This worked for many genuine analysis posts, but it also caused the baseline to overpredict `analysis` for hot takes and reactions that used detailed League terminology. The fine-tuned DistilBERT model was less collapsed into `analysis`, but it still did not reliably understand sarcasm, exaggeration, or the difference between a detailed personal story and a structured argument.

The biggest gap between my label definitions and the learned model boundary was tone. I intended `meme_shitpost` and `reaction` to be separated by communicative purpose, but the model often treated jokes, venting, and emotional reactions as interchangeable. This suggests that the dataset was too small and imbalanced for the model to learn subtle rhetorical categories, especially for `meme_shitpost`.

---

## Definition of Success Revisited

In `planning.md`, I defined success as:

* overall accuracy of at least 75%
* macro F1 of at least 0.72
* per-label F1 of at least 0.65 for every label
* `analysis` precision of at least 0.75
* no label pair confused more than 25% of the time

The final model did **not** meet these goals. The fine-tuned model reached only 0.446 accuracy and 0.38 macro F1. Its `analysis` precision was 0.68, below the planned 0.75 threshold, and its `meme_shitpost` F1 was only 0.14. Based on my original definition, this model is not good enough for deployment in a real community tool.

However, the failure is still useful. It shows that the label taxonomy captures real distinctions, but the dataset and model setup were not strong enough to learn those distinctions reliably. A better version of this project would need more data, more balanced label counts, and more explicit examples of hard boundaries.

---

## Spec Reflection

One way the spec helped was by forcing me to define labels before training the model. The requirement to write clear label definitions, examples, and hard edge cases made the annotation process more consistent than it would have been if I had only used vague labels like “good” and “bad.” The hard-edge-case section was especially useful because many r/leagueoflegends posts mix analysis, frustration, and humor in the same comment.

One way the implementation diverged from the spec was the baseline. I originally planned to use a Groq-hosted LLM as a zero-shot baseline, but I ran into API token limits before completing the run. I replaced that baseline with a local TF-IDF + Logistic Regression model because it was free, lightweight, deterministic, and easy to rerun. This changed the comparison: instead of comparing fine-tuning against a large LLM, I compared it against a traditional text-classification baseline.

Another divergence was the label distribution. I initially hoped for roughly 50 examples per label, but the final dataset had many more `analysis` examples than `meme_shitpost` examples. I kept the `meme_shitpost` label anyway because it represents an important type of League discourse, but the imbalance likely hurt model performance.

---

## AI Usage

I used AI assistance during several parts of this project, but I reviewed and revised the outputs myself.

### 1. Label taxonomy and planning assistance

I asked ChatGPT to help design a label taxonomy for r/leagueoflegends based on the assignment requirements. It suggested labels including `analysis`, `hot_take`, `reaction`, and `meme_shitpost`, along with definitions and example posts. I revised the taxonomy by adding a `skip` category during actual annotation for posts that were not community takes, such as patch note reposts, bot text, or link-only posts. The final labels used for training excluded `skip`.

### 2. Annotation assistance

I used an LLM to pre-label a small batch of examples before reviewing them myself, as the instructions suggested. The AI-generated labels were treated as first-pass suggestions, not final labels. I manually reviewed the examples, changed labels where needed, and used my own decision rules for ambiguous cases. For disclosure and reproducibility, I tracked which examples were AI-assisted separately from my final human-reviewed label decisions.

### 3. Failure analysis assistance

After evaluation, I asked ChatGPT to look at wrong predictions and identify common error patterns, as the instructions for the project required. It pointed out that the baseline overpredicted `analysis`, struggled with sarcasm and exaggeration, and often confused `hot_take` with `analysis`. I verified these patterns myself by checking the confusion matrices and rereading specific misclassified examples before including them in this README.


---

## Conclusion

The final fine-tuned DistilBERT model did not outperform the TF-IDF + Logistic Regression baseline. The baseline achieved 0.5077 accuracy, while the fine-tuned model achieved 0.4462 accuracy. The model learned some useful patterns, especially around `analysis` and `hot_take`, but it struggled with the more subtle distinctions involving emotional reactions, sarcasm, and meme/shitpost content.

The main lesson from this project is that label design is only the first step. Even with clear definitions, a small and imbalanced dataset makes it hard for a model to learn rhetorical distinctions that depend on tone and context. If I continued this project, I would collect more examples, especially for `meme_shitpost`, add more hard boundary cases to the training data, and tighten the annotation rules for borderline `analysis` versus `hot_take` examples.
