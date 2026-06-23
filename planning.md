# Parlay — A FIFA 2026 World Cup Discourse Classifier

*AI201 Project 3: Show What You Know (TakeMeter)*

Parlay is a fine-tuned text classifier that sorts World Cup prediction comments by the *kind* of claim they make. The long-term goal is a prediction tool that aggregates what fans on Reddit say about who will win the 2026 FIFA World Cup — but raw fan comments are mostly noise. Before any aggregation can happen, Parlay has to tell a reasoned tactical breakdown apart from a flag-waving one-liner. This classifier is that first filtering layer.

---

## 1. Community

**Community chosen:** Football (soccer) prediction discourse on Reddit, drawn from two threads asking who will win the 2026 World Cup: one on `r/worldcup` (collected the day of annotation, mid-tournament) and one on `r/AskTheWorld` (collected ~2 weeks earlier, pre/early tournament).

**Why it fits a classification task:** "Who will win?" threads attract an unusually wide spread of comment quality, which is exactly what a discourse classifier needs. In the same thread you find detailed bracket-path breakdowns ("Argentina has the easiest path to the quarters if they get past Uruguay in the R32..."), bold narrative calls ("It's the Netherlands' turn"), bare one-word votes ("Germany"), and people who refuse to engage at all ("no one's winning this, Infantino's wallet is"). That range means the four labels actually have something to separate. It also matters to the project's real purpose: Parlay should weight thoughtful, evidence-based consensus more heavily than sheer volume of hype, so distinguishing reasoning from vibes is the whole point.

---

## 2. Labels

Four mutually exclusive labels. The decision boundary for each can be stated in one sentence, and the distinctions reflect how fans themselves talk about good vs. bad takes.

### `analytical_prediction`
A prediction backed by verifiable football logic — squad depth, player form, tactical matchups, bracket path, head-to-head history, or coaching — where the strength of the claim is proportional to the evidence given.
- *Example:* "France or Spain are most likely. I'd put Argentina top three but they haven't been tested and their midfield/defense looks weak so far."
- *Example:* "Argentina has the easiest path to the quarters if they get past the brutal R32 vs Uruguay; after that Colombia/Canada/Belgium are the only obstacles."

### `hot_take`
A confident, bold, or contrarian prediction that rests on narrative, gut feeling, generalization, or a single fact stretched into an absolute claim, rather than proportional evidence. Includes "team X always bottles/flops" tropes and rigging claims that still name a winner.
- *Example:* "Its Argentina. Nobody is stopping Messi again."
- *Example:* "We all know Daddy Infantino is gonna hand it to Argentina."

### `casual_vote`
A minimal-effort post that names a country, states a rooting preference, or drops a meme/flag with no attempt to justify the choice. Bare picks and "I support X" both live here.
- *Example:* "France no doubt."
- *Example:* "I think Spain or France will win. I'll be supporting Spain, obviously."

### `meta_skepticism`
The post declines to predict — "too early to tell," "it's a toss-up" — or redirects to critiquing the format, hosting, FIFA, or corruption instead of naming a winner.
- *Example:* "I haven't seen enough to be decisive yet. I have about five teams I could believe could lift it."
- *Example:* "No one's winning this World Cup. Infantino's wallet is winning."

---

## 3. Hard edge cases

These are the genuinely ambiguous post types, with the rule I commit to before annotating.

**Edge case 1 — analytical_prediction vs. hot_take (the "single fact, absolute claim").**
A post cites one real football factor but uses it to justify a guarantee far bigger than the evidence supports, e.g. "Argentina has the easiest path to the semis... but they have the refs and FIFA on their side." **Rule:** if the scope of the claim stays proportional to the evidence, it's `analytical_prediction`; if one fact is stretched into an absolute or wrapped in a conspiracy/narrative frame, it's `hot_take`.

**Edge case 2 — analytical_prediction vs. meta_skepticism (reasoning that refuses to land).**
A post reasons in football terms but concludes that *no one* can be picked yet, e.g. "It's too early — it could go to literally anyone between these seven teams." **Rule:** if the post ends without committing to a target team, it's `meta_skepticism`, even if the language sounds analytical, because it gives Parlay no team to weight.

**Edge case 3 — casual_vote vs. meta_skepticism (the bare "I support X" with a refusal).**
"I will not make any predictions. I support Spain with all my heart." **Rule:** a clear rooting preference, even alongside a refusal to predict, is `casual_vote` (it expresses a positive lean toward a team). Reserve `meta_skepticism` for posts that name no team at all and reject the premise of predicting.

**Threading note (real-data cleanup).** The scraped data splits some comments into reply fragments and includes automod bot text, pure banter ("Lmao", "wow"), deleted comments, and off-topic tangents (cricket jokes, ticket-price logistics). These carry no prediction or preference, so they are flagged `exclude=yes` in the full dataset and kept out of the 222 labeled examples. This keeps the labeled set honest instead of forcing junk into a catch-all.

---

## 4. Data collection plan

**Source:** Comments scraped from the two prediction threads named in §1, parsed into a flat list. The raw scrape yielded 247 comment units.

**Volume and per-label counts (actual, after annotation):**

| Label | Count | Share |
|---|---|---|
| casual_vote | 118 | 53.2% |
| analytical_prediction | 48 | 21.6% |
| hot_take | 35 | 15.8% |
| meta_skepticism | 21 | 9.5% |
| **Total labeled** | **222** | **100%** |
| (excluded as non-classifiable) | 25 | — |

**Imbalance — what I'm doing about it (honest assessment).** `casual_vote` dominates at 53% and `meta_skepticism` is under the project's ~20%-per-class target at 9.5%. This is real: most prediction-thread comments genuinely are one-line votes. Rather than relabel honest data to hit a quota, I will (a) keep `takemeter_labeled.csv` as the honest ground truth, stratified 70/15/15 so every class appears in train/val/test; and (b) address the skew at *training* time via class weighting in the loss (or down-sampling `casual_vote` in the train split) so the model doesn't just learn to predict the majority class. If `meta_skepticism` recall stays poor after that, I will collect additional examples by pulling more "is it too early to tell" and boycott/corruption threads, since those are where this class concentrates. The test set is left untouched and stratified so reported metrics reflect the real distribution.

---

## 5. Evaluation metrics

**Primary:** macro-averaged F1 across all four classes, plus per-class precision/recall/F1, plus a full 4×4 confusion matrix.

**Why these and not accuracy alone:** with `casual_vote` at over half the data, a model that blindly predicts `casual_vote` would score ~53% accuracy while being useless for Parlay. Accuracy hides that. Macro-F1 weights every class equally, so the model is rewarded for getting the rare-but-important classes right. The per-class breakdown matters because Parlay's value lives in two specific cells: high **precision on `analytical_prediction`** (so the aggregator doesn't elevate hype into its "expert" signal) and decent **recall on `hot_take`** (so bold noise gets correctly filtered out rather than leaking into the analytical bucket). The confusion matrix tells me *which* pairs get confused — I specifically expect analytical↔hot_take and casual↔meta confusion, matching the edge cases above.

---

## 6. Definition of success

Concrete, checkable thresholds:

- **Deployment-useful bar:** macro-F1 ≥ 0.65 on the test set, **and** `analytical_prediction` precision ≥ 0.75 (Parlay must not let hot takes contaminate its analytical signal — precision on this class is the single most important number).
- **Fine-tuning must earn its keep:** the fine-tuned DistilBERT must beat the zero-shot Groq `llama-3.3-70b-versatile` baseline on macro-F1 on the same test set. If it doesn't, fine-tuning didn't help and I'll say so.
- **Edge-case behavior:** on the borderline analytical-vs-hot_take cases, the fine-tuned model should match my labels more often than the baseline does.

If macro-F1 lands below ~0.55, I'll treat the project as having surfaced a labeling or data-volume problem rather than a working classifier, and report it honestly in the evaluation write-up.

---

## 7. AI Tool Plan

This project has no implementation code to generate, so AI tools help in three specific places:

**Label stress-testing (before annotating).** I gave the four definitions and edge-case rules to an LLM and asked it to generate ~10 comments that deliberately straddle `analytical_prediction` and `hot_take`. The ones I couldn't classify in a few seconds drove the "proportional to the evidence" rule in §3. This happened before the 222 examples were labeled.

**Annotation assistance (disclosure).** All 222 examples were labeled by me directly, using the decision rules above; an LLM was used as a *second reader* to flag rows where its call disagreed with mine, and I reviewed every flagged row and made the final decision (e.g., reclassifying the "host countries / Italy sucks" comment to excluded because it names no winner, and aligning the two split fragments of the "Dutch looked good" comment to `casual_vote`). No labels were accepted from the model without my review. The `rationale` column records the reason for each call.

**Failure analysis (after evaluation).** Once the models run, I'll feed the list of misclassified test examples to an LLM and ask it to find systematic patterns — specifically whether the model leans on comment *length* or trigger words like "because," "think," or "path" rather than actual reasoning. I'll verify any claimed pattern by hand against the confusion matrix before writing it up, since the model can hallucinate a tidy story that the data doesn't support.

---

## Repository artifacts produced so far

- `takemeter_labeled.csv` — 222 labeled examples with `id, source_thread, comment_text, label, split, rationale`. This is the file uploaded to Colab for fine-tuning.
- `takemeter_dataset_full.csv` — all 247 scraped units including the 25 `exclude=yes` rows, kept for transparency about what was filtered and why.

## Next steps (Milestone 3+)

1. Upload `takemeter_labeled.csv` to the starter Colab notebook; map labels to integer IDs.
2. Fine-tune `distilbert-base-uncased` with class weighting to counter the `casual_vote` skew; tune learning rate and epochs against the validation split.
3. Run the zero-shot Groq `llama-3.3-70b-versatile` baseline on the identical test split.
4. Produce `evaluation_results.json` and `confusion_matrix.png`; write the evaluation report with ≥3 analyzed errors.
