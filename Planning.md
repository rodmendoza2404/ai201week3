# TakeMeter — Planning
**AI201 · Project 3**
 
Design decisions and working notes behind the project. The README is the polished final report; this is the reasoning behind each call.
 
---
 
## 1. Community
 
**r/soccer.**
 
I chose r/soccer because the discourse is high-volume, plain-English, and varies enormously in quality — which is exactly what makes a quality classifier worth building. On any given day you get tactical breakdowns next to one-line hot takes next to pure match-thread emotion. I collected during the 2026 World Cup, when the sub is at peak activity (match clips, post-match debate, transfers, manager post-mortems), so reaching 200+ comments was easy. Crucially, the quality distinction here is about the **form** of a comment — does it reason with evidence, just assert, or just emote — so the labels can be applied consistently without deep tactical expertise.
 
---
 
## 2. Labels
 
Three mutually exclusive labels.
 
### `analysis`
The comment supports a claim with **specific, checkable evidence** — a stat, a tactical observation tied to what actually happened, a historical/contextual comparison, or concrete reasoning. If you removed the opinion, the evidence would still stand on its own.
- *"Nine years ago, in February 2017, Canada were ranked 117th in the world behind Zimbabwe, Latvia and El Salvador. Now they're in the last 16 of the World Cup."*
- *"MLS has a unique structure called 'Designated Players'. Each team has a $6.4M budget for 20 roster slots, then three DP slots that don't count against the cap."*
### `hot_take`
A bold, confident opinion asserted **without real supporting evidence**. It might be right, but it argues by assertion (or uses a decorative stat that isn't genuine reasoning).
- *"MLS football is so weird. A random team will sign a superstar and that will be their only notable player."*
- *"Zlatan not glazing himself for once... Messi is truly the GOAT."*
### `reaction`
An immediate **emotional response** to a moment or result — joy, anger, disbelief. The comment expresses a feeling, not an argument.
- *"WE'VE WON A WORLD CUP KNOCKOUT MATCH COME ONNNNNNNN."*
- *"Ronaldo asking for a rheumatologist live on air."*
---
 
## 3. Hard edge cases
 
**Anticipated hard case — a vent that contains a stat.**
> *"How do we lose to THEM when we had 70% possession and 20 shots?? Embarrassing."*
 
Possession and shots are stats, so it looks like `analysis`. But the dominant function is venting, and the numbers express outrage rather than build an argument.
**Decision rule:** if a comment mixes emotion and a stat, ask what the stat is *doing*. If it argues/explains a point → `analysis`. If it's there to express outrage and you couldn't reconstruct an argument from it → `reaction`.
Secondary boundary (`analysis` vs `hot_take`): evidence that's checkable and would stand without the opinion → `analysis`; evidence that's vague, cherry-picked, or decorative → `hot_take`.
 
**Three real cases I hit during annotation:**
 
1. *"Deserved, South Africa were offering nothing"* — between `analysis` and `hot_take`. It gestures at a reason ("offering nothing"), but the evidence is vague and not checkable. **Decided `hot_take`** — the justification is decorative, not genuine reasoning.
2. *"That's why the last tournament he won was in 1998"* (re: Bielsa) — between `analysis` and `hot_take`. It uses a real date, but as a dismissive jab, not to build an argument (another commenter immediately corrected it as cherry-picked). **Decided `hot_take`** — a real fact used decoratively is still a hot take.
3. *"What an absolute shitshow of a team"* — between `reaction` and `hot_take`. It's an evaluative judgment, but makes no specific claim and is pure venting. **Decided `reaction`** — emotional outburst with no argument.
---
 
## 4. Data collection plan
 
- **Source:** public comments from r/soccer World Cup 2026 threads (match clips, post-match discussion, transfer news, manager post-mortems). Public posts only — no auth-gated content.
- **Process:** scraped a raw pool, parsed out the Reddit UI noise (usernames, vote counts, bots, deleted comments, link-only and one-word comments), which left 555 clean candidate comments. Curated those down to **206 labeled examples** with balanced classes.
- **Balance handling:** World Cup threads skew heavily toward `reaction` and jokes, so an uncurated set would have been 70%+ reaction. `analysis` was the class most at risk of falling short, so I deliberately kept the discussion-thread comments (Bielsa post-mortem, MLS Designated Player explainers, the broadcast/ad-break debate) where evidence-based comments cluster. Final split: reaction 81 (39.3%), analysis 63 (30.6%), hot_take 62 (30.1%) — no class above 70%, each above 20%.
- **Pre-labeling:** used an LLM (Claude) to produce a first-pass label per comment from the definitions above, then reviewed and corrected every label myself. Borderline cases flagged in a `notes` column. Disclosed in the README AI-usage section.
---
 
## 5. Evaluation metrics
 
Accuracy alone isn't enough — the classes aren't perfectly balanced and aren't equally easy. I report:
 
- **Overall accuracy** — headline number, and the comparison point against the zero-shot baseline.
- **Macro-averaged F1** — averages F1 across the three classes equally, so a model that nails the two easy classes but fails the hard one can't hide behind a high accuracy number. (This turned out to matter a lot — see README.)
- **Per-class precision / recall / F1** — where the real story is. I expected the `analysis` vs `hot_take` boundary to be hardest, so I wanted to see whether `analysis` recall held while precision dropped (over-predicting analysis) or vice versa.
- **Confusion matrix** — to read the *direction* of the errors, which is more actionable than any single score.
---
 
## 6. Definition of success
 
The classifier would be genuinely useful if it could flag low-effort posts in a community tool without misfiring on people who actually made an argument. Concrete bar:
 
- **Macro-F1 ≥ 0.70**, with **no single class F1 below ~0.60**.
- **Fine-tuned beats the zero-shot baseline by a meaningful margin** (target ≥ +0.10 accuracy).
- The dominant residual error should be the expected `analysis` ↔ `hot_take` confusion, not `reaction` getting mixed up with the other two.
*(Outcome and honest assessment against this bar are in the README evaluation report.)*
 
---
 
## AI Tool Plan
 
This is a data/labeling project, so AI helps in three specific places:
 
**Label** Assist with label that I used the ones I couldn't classify cleanly to tighten the definitions before annotating.
 
**Annotation assistance.** Reviewed and corrected every label by reading the original comment. The `suggested`/`notes` tracking keeps the disclosure honest. Labels are my calls. Disclosed in README.
 
**Failure analysis.** After fine-tuning, fed the misclassified test examples to an LLM to surface patterns (which label pair is confused, short vs long, decorative stats), then verified each pattern by re-reading the examples before writing the evaluation.