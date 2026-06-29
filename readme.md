# TakeMeter — Classifying Discourse Quality on r/soccer
**AI201 · Project 3**

A fine-tuned text classifier that sorts r/soccer comments into three discourse types: evidence-based **analysis**, unsupported **hot_take**, and emotional **reaction**. Compares a fine-tuned DistilBERT against a zero-shot Llama-3.3-70B baseline.

---

## Community

**r/soccer.** I chose r/soccer because the discourse is high-volume, plain-English, and varies enormously in quality, exactly what makes a quality classifier interesting. The data was collected during the 2026 World Cup, when the subreddit is at peak activity: match-clip threads, post-match debate, transfer news, and manager post-mortems all running at once. Critically, the quality distinction here is about the **form** of a comment — does it reason with evidence, just assert, or just emote ,which means the labels can be applied consistently without deep tactical expertise.

## Label Taxonomy

**`analysis`** — supports a claim with specific, checkable evidence: a stat, a tactical observation tied to what happened, a historical/contextual comparison, or concrete reasoning. Strip the opinion and the evidence still stands.
- *"Nine years ago, in February 2017, Canada were ranked 117th in the world behind Zimbabwe, Latvia and El Salvador. Now they're in the last 16 of the World Cup."*
- *"MLS has a unique structure called 'Designated Players'. Each team has a $6.4M budget to fill out 20 roster slots, then three DP slots that don't count against the cap — so you get 20 players around $320k and 2-3 stars."*

**`hot_take`** — a bold, confident opinion asserted without real supporting evidence. Might be right, but it argues by assertion (or uses a decorative stat that isn't genuine reasoning).
- *"MLS football is so weird. A random team will sign a superstar and that will be their only notable player."*
- *"Zlatan not glazing himself for once... Messi is truly the GOAT."*

**`reaction`** — an immediate emotional response to a moment or result. Expresses a feeling, not an argument.
- *"WE'VE WON A WORLD CUP KNOCKOUT MATCH COME ONNNNNNNN."*
- *"Ronaldo asking for a rheumatologist live on air."*

## Data Collection & Labeling

- **Source:** public comments scraped from r/soccer World Cup 2026 threads (match clips, post-match discussion, transfer news, manager post-mortems). Public posts only.
- **Volume:** 555 raw comments parsed and cleaned (UI noise, bots, deleted comments, link-only and one-word comments removed), curated down to **206 labeled examples** with balanced classes.
- **Labeling process:** an LLM (Claude) produced a first-pass label per comment using the definitions above; I then reviewed every label against the original comment and corrected disagreements. Borderline cases were flagged in a `notes` column. *(See AI Usage.)*

**Label distribution (206 total):**

| Label | Count | Share |
|---|---|---|
| reaction | 81 | 39.3% |
| analysis | 63 | 30.6% |
| hot_take | 62 | 30.1% |

No class exceeds 70%; each clears 20%.

**Three genuinely difficult examples:**

1. *"Deserved, South Africa were offering nothing"* — between `analysis` and `hot_take`. It gestures at a reason ("offering nothing"), but the evidence is vague and not checkable. **Decision: `hot_take`** — the justification is decorative, not genuine reasoning.
2. *"That's why the last tournament he won was in 1998"* (re: Bielsa) — between `analysis` and `hot_take`. It uses a real date, but as a dismissive jab, not to build an argument (another commenter immediately corrected it as cherry-picked). **Decision: `hot_take`** — a real fact used decoratively is still a hot take.
3. *"What an absolute shitshow of a team"* — between `reaction` and `hot_take`. It's an evaluative judgment, but makes no specific claim and is pure venting. **Decision: `reaction`** — emotional outburst with no argument.

## Fine-Tuning Approach

- **Base model:** `distilbert-base-uncased` with a 3-class classification head.
- **Split:** 70 / 15 / 15 train / validation / test, stratified by label (144 / 31 / 31).
- **Training:** 3 epochs, learning rate 2e-5, batch size 16, weight decay 0.01, 50 warmup steps; best model selected on validation accuracy.
- **Key hyperparameter decision:** kept the default 3 epochs / 2e-5. On only 144 training examples, more epochs overfit fast and a higher learning rate destabilized validation accuracy; 3 epochs at 2e-5 was the most stable setting and the recommended starting point for a dataset this size.

## Baseline

Zero-shot classification with Groq `llama-3.3-70b-versatile` on the same locked test set, temperature 0. The prompt gave the three label definitions plus one example each and instructed the model to output only the label name. Unparseable responses were excluded from baseline metrics.

---

## Evaluation Report

### Overall accuracy
| Model | Accuracy |
|---|---|
| Zero-shot baseline (Llama-3.3-70B) | [0.806] |
| Fine-tuned DistilBERT | 0.645 |

### Per-class metrics — fine-tuned
| Label | Precision | Recall | F1 |
|---|---|---|---|
| analysis | 0.556 | 1.000 | 0.714 |
| hot_take | 0.000 | 0.000 | 0.000 |
| reaction | 0.833 | 0.833 | 0.833 |

Macro-F1 ≈ 0.52.

### Per-class metrics — baseline
| Label | Precision | Recall | F1 |
|---|---|---|---|
| analysis | [1.00] | [0.80] | [0.89] |
| hot_take | [0.83] | [0.56] | [0.67] |
| reaction | [0.71] | [1.00] | [0.83] |

### Confusion matrix — fine-tuned (test set)
Rows = true, columns = predicted.

| true \ pred | analysis | hot_take | reaction |
|---|---|---|---|
| **analysis** | 10 | 0 | 0 |
| **hot_take** | 7 | 0 | 2 |
| **reaction** | 1 | 1 | 10 |

The story is in row 2: of 9 true hot_takes, **zero** were predicted as hot_take — 7 went to `analysis`, 2 to `reaction`. The model predicted "hot_take" only once across the entire test set, and that prediction was wrong. `analysis` absorbs the hot_takes (precision drops to 0.556 while recall is a perfect 1.000); `reaction` stays clean.

### Three wrong predictions (with analysis)
All three are true `hot_take` comments the model read as `analysis` — the single failure mode that defines this model.

1. **Text:** "It's actually crazy he never played for a big European team, crazy shot stopper in his prime"
   **True:** hot_take · **Predicted:** analysis
   **Why:** It names a player and makes a career claim, so it carries the surface features of analysis — but "crazy shot stopper in his prime" is pure assertion with zero checkable evidence. The model keys on the football-vocabulary and treats it as reasoning.

2. **Text:** "The saddest thing is that Marcelo Bielsa is a very talented coach with many good ideas. But he is basically too inflexible, kind of a dictator, in managing the dressing room."
   **True:** hot_take · **Predicted:** analysis
   **Why:** Measured tone, named subject, no caps or profanity. It reads like analysis structurally, but every claim ("talented," "inflexible," "dictator") is unsupported characterization. The model has learned "calm + named coach = analysis."

3. **Text:** "Canada won the game but Mbokazi was definitely Man of the Match"
   **True:** hot_take · **Predicted:** analysis
   **Why:** A confident evaluative claim ("definitely Man of the Match") with no supporting reasoning. The named player plus declarative structure looks like analysis to the model, which never learned that a confident claim without evidence is the whole point of a hot take.



### Sample classifications

| Comment                                                                                                                                                                                                       | True     | Predicted | Confidence |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | --------- | ---------- |
| "Anyone who thinks Fox is better than Telemundo is not a true fan of the sport."                                                                                                                              | hot_take | analysis  | 0.35       |
| "Too bad more players don’t have the confidence to go for the full volley like this. It’s a lost art."                                                                                                        | hot_take | analysis  | 0.34       |
| "I guess I haven’t been paying enough attention to the conditions they have been going through. Just the pure look of defeat on his face reveals so much. They deserve better."                               | reaction | hot_take  | 0.34       |
| "It's actually crazy he never played for a big European team, crazy shot stopper in his prime"                                                                                                                | hot_take | analysis  | 0.35       |
| "Eidur Gudjohnsen was one of my favorite player from the 04-06 Chelsea. He was a smart player and I'd argue is very underrated."                                                                              | hot_take | analysis  | 0.38       |
| "Canada won the game but Mbokazi was definitely Man of the Match"                                                                                                                                             | hot_take | analysis  | 0.36       |
| "The saddest thing is that Marcelo Bielsa is a very talented coach with many good ideas. But he is basically too inflexible, kind of a dictator, in managing the dressing room."                              | hot_take | analysis  | 0.37       |
| "MLS football is so weird. A random team will sign a superstar and that will be their only notable player."                                                                                                   | hot_take | reaction  | 0.35       |
| "It was a boring game, but it’s probably the greatest win in Canadian soccer history. edit: Canadian men's soccer history."                                                                                   | reaction | analysis  | 0.36       |
| "Ehh... there's been a lot of controversy for them the past two years. You should look into it, the KFA has been corrupt for a long time and Korea has been going through something similar to Italy..."      | hot_take | analysis  | 0.34       |
| "You have a world cup only once every 4 years. How the fuck are you so spoiled that you can't coexist with some people for 2-3 weeks? Not a year, a couple of weeks. Shame on everyone for this toxic sit..." | hot_take | reaction  | 0.35       |

### Reflection — what the model learned vs. what I intended
I intended the model to learn the *function* of a comment: reasoning vs. asserting vs. emoting. What it actually learned is closer to **surface signals**. `reaction` has its own clear markers — caps, exclamation, profanity, short emotive bursts — so it scores well (F1 0.83). But `analysis` and `hot_take` share the same vocabulary (player names, confident claims, the occasional stat), and the only real difference is whether the evidence is genuine or decorative — which DistilBERT can't see. With no surface signal of its own, `hot_take` collapsed entirely into `analysis`: the model decided that any calm, on-topic comment with names and claims is analysis. The gap between intent and result is sharpest exactly where I predicted the boundary would be hard, and 144 training examples (≈43 hot_takes) was almost certainly too few to teach a distinction that subtle.

**What would fix it:** more hot_take training examples, especially ones that mimic analysis structure (named players + a decorative stat), so the model is forced to learn that unsupported confidence — not the absence of football vocabulary — is what defines a hot take.

### Definition of success — did it hit the bar?
No, and honestly so. My planning.md bar was macro-F1 ≥ 0.70 with no class F1 below ~0.60. Reaction (0.83) and analysis (0.71) clear it; hot_take (0.00) fails completely. The model is **not** deployable as a three-way classifier, but it reveals a specific, fixable problem rather than a vague one — which is the more useful outcome for understanding the task.

### Spec reflection
- **One way the spec helped:** the "no class above 70%" rule forced me to curate rather than dump every comment. World Cup threads are reaction-heavy, and an uncurated set would have been ~70%+ reaction and trained a pure majority-class predictor.
- **One way I diverged:** the spec suggests ~200 manually collected examples; I scraped and cleaned 555, then curated down to 206, because match threads are noisy and I needed the larger pool to find enough genuine `analysis`.

### AI Usage
1. **Label design + stress-testing:** I had Claude help define the three labels and the analysis/hot_take decision rule, then pressure-test boundary cases before I committed to annotating.
2. **Pre-labeling:** Claude produced a first-pass label for all 206 comments. I reviewed every one against the original text and corrected the ones I disagreed with (the `notes` column flags the borderline cases I scrutinized hardest). The final labels are my calls.
