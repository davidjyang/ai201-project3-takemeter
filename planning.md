# TakeMeter Planning Document

## 1. Community

For this project, I chose **r/leagueoflegends**, a large Reddit community focused on League of Legends gameplay, champion balance, esports, patch changes, ranked experiences, and general community discussion. This community is a strong fit for a text classification task because the discourse is active, text-heavy, and varied: posts range from detailed gameplay analysis to emotional reactions, unsupported balance claims, and intentionally unserious memes or shitposts.

This variation makes the classification problem interesting because League of Legends discussion often depends on whether a post is actually making a reasoned argument or simply expressing frustration, hype, or humor. A regular user in the community would likely recognize the difference between a useful gameplay explanation, a dramatic hot take, an immediate reaction to a match, and a joke post, even though those categories can sometimes overlap.

## 2. Labels

I will use four labels: `analysis`, `hot_take`, `reaction`, and `meme_shitpost`.

### `analysis`

A post should be labeled `analysis` if it makes a structured argument about gameplay, balance, esports, or strategy using specific evidence such as patch changes, champion interactions, matchup details, statistics, draft context, or in-game reasoning.

Example posts:

1. “The reason Jinx became stronger this patch is not just the direct buff. With early dragon fights lasting longer and enchanters being picked more often, she gets more time to scale and reset in teamfights.”

2. “T1’s draft lost because they had no reliable way to start fights. Their only hard engage was on Rell, and once she fell behind, the poke comp could never force objectives on their own terms.”

### `hot_take`

A post should be labeled `hot_take` if it makes a bold, broad, or controversial claim with little support, usually stated confidently as if it is obvious or final.

Example posts:

1. “This is the worst Worlds meta of all time and anyone pretending otherwise is coping.”

2. “ADC players are the most entitled players in the game and the role has been broken for years.”

### `reaction`

A post should be labeled `reaction` if it is an immediate emotional response to a specific match, play, patch note, announcement, or community event, with little attempt to make a broader argument.

Example posts:

1. “That Faker Azir shuffle was insane. I actually screamed.”

2. “No way they buffed Yuumi again. I cannot believe this is real.”

### `meme_shitpost`

A post should be labeled `meme_shitpost` if its main purpose is humor, irony, exaggeration, parody, or absurdity rather than sincere analysis, complaint, or argument.

Example posts:

1. “My jungler ganked my lane once and I immediately knew I was in loser’s queue because Riot would never let me experience happiness.”

2. “BREAKING: Yuumi players have discovered the keyboard. Analysts are calling this the biggest mechanical development since Faker’s debut.”

## 3. Hard Edge Cases

The hardest edge cases will be posts that mix a sincere claim with emotional or humorous framing. For example, a post like “Riot clearly has no idea how to balance Zeri. Every time she is playable, she takes over pro play, but when they nerf her, she becomes useless in solo queue” could be read as `hot_take`, `analysis`, or even `reaction` depending on the surrounding context.

My annotation rule will be to label the post based on its main function, not just its tone. If the post provides specific, verifiable evidence or reasoning that would still support the claim without the emotional wording, I will label it `analysis`. If it makes a broad claim without enough support, I will label it `hot_take`. If it is responding emotionally to a specific event, I will label it `reaction`. If the main purpose is comedy, exaggeration, parody, or a community in-joke, I will label it `meme_shitpost`, even if it contains an underlying complaint.

Another difficult boundary will be between `hot_take` and `meme_shitpost`, because League players often express opinions through exaggeration. For example, “Riot buffed Yone again because apparently the balance team lost to one in silver promos” contains a balance complaint, but the phrasing is mainly comedic. I will label this as `meme_shitpost` unless the surrounding post develops a sincere argument about balance.

## 4. Data Collection Plan

I will collect public posts and comments from **r/leagueoflegends**. I will sample from multiple types of threads so the dataset is not dominated by one style of discussion. Useful sources include post-match discussion threads, patch note threads, champion balance discussions, esports match threads, ranked complaint threads, and high-engagement general discussion posts.

My initial target is **200 total annotated examples**, with a goal of approximately **50 examples per label**:

* `analysis`: 50 examples
* `hot_take`: 50 examples
* `reaction`: 50 examples
* `meme_shitpost`: 50 examples

During collection, I will avoid taking too many examples from a single thread because that could make the model learn thread-specific vocabulary instead of general discourse patterns. I will also remove or skip posts that are too short to classify meaningfully, such as single-word comments or comments that depend entirely on an image, video, or unavailable context.

If one label is underrepresented after 200 examples, I will collect additional examples targeted toward that label. For example, if `analysis` is underrepresented, I will sample more from detailed patch discussion, esports analysis, champion mains discussions, or long-form gameplay threads. If `meme_shitpost` is underrepresented, I will sample from highly upvoted joke posts, offseason threads, or comments using obvious parody and community memes. If I still cannot collect at least 35 examples for a label, I will reconsider whether that label is common enough to include or whether it should be merged with another category.

## 5. Evaluation Metrics

Accuracy alone is not enough for this task because the labels may not be perfectly balanced, and some mistakes are more important than others. For example, a model could get decent accuracy by overpredicting common categories while failing to identify `analysis`, which is probably the most useful category for a community tool.

I will use the following evaluation metrics:

### Overall accuracy

Accuracy will show the basic percentage of posts classified correctly. This is useful as a general performance measure, but I will not rely on it by itself.

### Per-label precision

Per-label precision will show how trustworthy the model is when it assigns a specific label. This matters because if the model labels a post as `analysis`, I want that prediction to be reliable. Low precision for `analysis` would mean the model is incorrectly treating unsupported opinions or complaints as substantive arguments.

### Per-label recall

Per-label recall will show how many true examples of each category the model successfully catches. This matters because if the model misses many real `meme_shitpost` or `reaction` posts, it may not be useful for organizing community discourse.

### Macro F1 score

Macro F1 will average performance across all labels equally, rather than letting the largest class dominate the score. This is important because I want the classifier to work reasonably well for all four categories, not just the most common one.

### Confusion matrix

I will use a confusion matrix to inspect which labels the model mixes up most often. This will be especially important for edge cases such as `hot_take` versus `meme_shitpost`, `hot_take` versus `reaction`, and `analysis` versus `hot_take`.

## 6. Definition of Success

For this project, I would consider the classifier successful if it reaches the following objective benchmarks on a held-out test set:

* Overall accuracy of at least **75%**
* Macro F1 score of at least **0.72**
* Per-label F1 score of at least **0.65** for every label
* `analysis` precision of at least **0.75**
* No pair of labels confused with each other more than **25%** of the time in the confusion matrix

The most important success condition is that the model must reliably separate `analysis` from the other categories. In a real community tool, falsely labeling unsupported claims, reactions, or jokes as analysis would make the tool less useful because it would promote noisy content as if it were substantive. For deployment in a real community setting, I would accept the model as “good enough” only if it met the benchmarks above and its most common mistakes were understandable edge cases rather than obvious failures.

A stronger version of the model would reach at least **80% accuracy**, **0.78 macro F1**, and **0.80 precision for `analysis`**. That level of performance would make it genuinely useful as a lightweight sorting or moderation aid, especially if predictions were shown as suggestions rather than used as automatic final judgments.

### AI Tools Plan

## 7. AI Label Stress-Testing

To test whether my label definitions are clear enough, I generated boundary cases that could plausibly fit more than one label. These examples help clarify how I will handle ambiguous posts during annotation.

| Example Post                                                                                                                                                  | Possible Labels               | Final Label     | Explanation                                                                                                                                                      |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| “Riot has no idea what to do with Zeri. She is either pick/ban in pro or completely useless in solo queue, and somehow every rework makes the problem worse.” | `analysis` / `hot_take`       | `hot_take`      | The post points toward a real balance issue, but it does not provide specific patch history, statistics, or gameplay reasoning beyond a broad unsupported claim. |
| “That Baron throw was disgusting. They had no vision in river, burned Maokai ult early, and still walked in one at a time like solo queue players.”           | `reaction` / `analysis`       | `analysis`      | Even though the tone is emotional, the post gives specific in-game reasons for why the Baron setup failed.                                                       |
| “ADC is weak for exactly three minutes every season and then Riot gives them six items, three supports, and a handwritten apology.”                           | `hot_take` / `meme_shitpost`  | `meme_shitpost` | The main purpose is comedic exaggeration rather than a sincere balance argument.                                                                                 |
| “No way they buffed Yone again. This champion already has three screens of mobility and wins lane by missing half his abilities.”                             | `reaction` / `hot_take`       | `hot_take`      | The post reacts to a patch change, but its main function is to make a broad unsupported claim about Yone’s balance state.                                        |
| “Faker’s Azir shuffle was insane, but the real mistake was Gen.G stacking too tightly near mid without tracking flash cooldowns.”                             | `reaction` / `analysis`       | `analysis`      | The post begins as an emotional response but turns into a specific explanation of the play.                                                                      |
| “My jungler taxed one caster minion and my lane immediately entered a recession. Riot needs to investigate this economic terrorism.”                          | `complaint` / `meme_shitpost` | `meme_shitpost` | Although it is based on a frustrating in-game experience, the exaggerated absurd framing makes humor the main purpose.                                           |
| “This patch is awful. Every game is just whichever team picked the newest overtuned champion.”                                                                | `reaction` / `hot_take`       | `hot_take`      | The post expresses frustration, but it is mostly a broad claim about the patch without concrete examples or evidence.                                            |
| “People keep saying tanks are unkillable, but most of the clips are carries building no penetration and fighting after missing their main cooldowns.”         | `analysis` / `hot_take`       | `analysis`      | The post pushes back on a common opinion using specific gameplay reasoning about itemization and cooldown usage.                                                 |
| “NA is doomed. We lose one early skirmish and suddenly the whole team starts playing like the monitors turned off.”                                           | `reaction` / `meme_shitpost`  | `meme_shitpost` | The post responds to a likely esports moment, but the exaggerated phrasing makes it primarily a joke.                                                            |
| “Yuumi getting buffed after months of complaints is actually hilarious. I’m convinced Riot is trying to test the limits of human patience.”                   | `reaction` / `meme_shitpost`  | `reaction`      | The post uses humor, but it is mainly an immediate emotional response to a specific patch change rather than a full joke or parody.                              |

