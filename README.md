# TakeMeter

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first** — setup, the
> notebook, the baseline, and what to do when something breaks.
>
> Once `python test.py` passes:
>
> ```bash
> head -5 data/practice_labels.csv     # the shape your labels.csv needs
> ```
>
> Then open `takemeter.ipynb` **in this folder** — in VS Code, or with
> `jupyter notebook` if you prefer. Pick the kernel: the `.venv` inside this
> project. Run section 1, which reports the hardware you'll be training on.
> Everything else waits until you have data.
>
> Nothing to upload, nothing to connect, no accounts and no keys. The notebook
> runs on your machine and writes next to your code.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     HOW TO USE THIS FILE

     Unit 5 asks for the first five sections. Unit 6 adds the five below them.

     Everything is pasted as TEXT. No screenshots, no images.

     ⚠️ The confusion matrix especially. The notebook prints one as a markdown
     table, ready to copy. A screenshot of a matrix earns nothing. Paste the
     table.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 5 — THE BUILD ═══════════════════════ -->

## What This Does

TakeMeter sorts Hacker News comments by how much support they offer for what they say. It labels each comment `analysis` (backed by something specific and checkable), `opinion` (a claim with no verifiable support), or `reaction` (a short response with no developed claim, such as a quip or a bare question). I picked Hacker News because it is public, has a huge volume of text, and ranges from detailed technical arguments to one-word replies.

---

## Label Taxonomy

### `analysis`

**Definition:** makes a point backed by something specific and checkable (a number, a named product or event, a source, or a described mechanism) that is what supports the claim.

**Example 1:**
> When I was in ads, Q4 was always at least half of our yearly revenue. Q1 usually came in at less than 1/4 of Q4.

**Example 2:**
> There's been some prior art in this space I feel that should be mentioned from k8s SIG batch in the Kueue project https://kueue.sigs.k8s.io/docs/concepts/topology_aware_scheduling/

### `opinion`

**Definition:** a confident claim or judgment with no verifiable support offered, including first-hand experience with unnamed or unspecific details.

**Example 1:**
> Text search is, but normal people dont want text search. Normies want to concierge fetch them perfect answer on first try. LLMs are really good at convincing you they just did that.

**Example 2:**
> I'm not gonna buy a car that doesn't have a dealer network. Being able to go in person and complain when shit hits the fan is a requirement.

### `reaction`

**Definition:** a short response with no developed claim: a quip, a jab, a one-word answer, agreement, or a bare question.

**Example 1:**
> What is your point?

**Example 2:**
> Spoken like a script kiddie

### The hardest boundary

**Which two labels:** `analysis` and `opinion`.

**The decision rule I used every time:**
A post is `analysis` only if the specific, checkable thing is what supports its claim, meaning a reader could verify the claim itself. Naming a product, company, or technology is not enough. A story or claim with nothing verifiable in it is `opinion`, including first-hand experience with unnamed or unspecific details. A question with no claim is `reaction`, and a jab that also states a real claim is `opinion`.

I tightened this rule after stress-testing it (see How I Used AI). My first version said "names a specific checkable thing", which let "Switched to Postgres and it's been fine" count as analysis.

---

## The Dataset

**Where the posts came from:** public Hacker News comments from the HN Algolia API, pulled on 29 October 2026 (the newest comments plus keyword searches for startup, privacy, python, science and money). At most three comments per thread, usernames removed, whole comments kept. Comments over about 800 characters are mostly missing because they could not be pulled cleanly, so the set leans toward shorter comments, and the keyword searches skew some topics. Claude (an AI assistant) pulled and cleaned the comments for me.

**How I labelled them:** I labelled 20 rows cold (no tools, marked `cold` in the note column). Claude pre-labelled the other 186 rows in four batches from my written definitions and decision rule (marked `prelabeled`). I went through each batch and did not change any of its labels. Claude also flagged the rows it was least sure about.

**Counts per label** (as of unit 5, before the unit 6 relabelling; the changed counts are in The Improvement):

| Label | Count | Share |
|---|---|---|
| `opinion` | 102 | 49.5% |
| `analysis` | 56 | 27.2% |
| `reaction` | 48 | 23.3% |
| **Total** | 206 | 100% |

**Three hard cases**

**1.**
> *The post:* "Japan has had high-speed trains since the 1960's and have had zero passenger fatalities. They must be doing something right."
>
> *Could have been:* `analysis` or `opinion`.
>
> *I chose, because:* `opinion` (a cold label). The first sentence is a checkable fact, but the post only uses it as a springboard for a vague conclusion. Under my rule the fact has to be what supports the claim, and "doing something right" is not a claim the fact can verify.

**2.**
> *The post:* "'One Swift file' except it's two Swift files"
>
> *Could have been:* `reaction` or `analysis`.
>
> *I chose, because:* `reaction`. It is a checkable correction, but it is delivered as a quip and builds no point. This is one of the cases where my rule does not clearly settle it.

**3.**
> *The post:* "Google doesn't pay for transit. Their private glass was the whole reason the acquisition worked."
>
> *Could have been:* `opinion` or `analysis`.
>
> *I chose, because:* `opinion`. It names specific things (transit, private fiber, an acquisition), but nothing in the post supports the claim or lets a reader verify it.

All three involve a checkable fact that is not backing a real argument. I expect that to be where the model struggles too.

---

## The Training Run

**Base model:** `distilbert-base-uncased`

**Settings:** 3 epochs, learning rate 2e-5, batch size 16, seed 42, 128 tokens per post. Trained on an Apple Silicon GPU (mps), torch 2.14.1.

**Anything I changed from the defaults, and why:** nothing. I used the defaults.

**Split sizes** (unit 5 run, before the unit 6 relabelling)**:** train 144 / validation 31 / test 31 (stratified 70/15/15). Test per label: `opinion` 15, `analysis` 9, `reaction` 7. `reaction` has fewer than about 8 in the test split, so its scores will be jumpy from seed to seed.

---

## How I Used AI

**Moment 1: stress-testing my labels**

- *What I asked for:* eight new posts that sat on the boundaries of my three labels, which I classified on my own.
- *What came back:* in round one I matched Claude's labels on 5 of 8; I disagreed on three posts (a bare question that names ARM, an anecdote about "three companies and microservices", and "Rust 1.80 fixed that exact bug"). That showed "names a specific checkable thing" was too loose, because it did not separate naming a product from backing a claim.
- *What I changed:* I adopted a tighter rule (the checkable thing must be what supports the claim) and added worked examples. Round two matched on 7 of 8, and I kept the rule as written, including treating a verifiable claim with no cited source as `analysis`.

**Moment 2: writing my acceptance criteria**

- *What I asked for:* I asked Claude to write the criteria and numbers for me.
- *What came back:* it declined to pick the numbers, since they had to be mine. It gave me facts about my data instead: always guessing `opinion` scores about 0.50, and the test split would be about 31 posts. I picked the numbers from options. Claude pointed out that my 0.70 per-label F1 target implied accuracy of about 0.70, so my 0.55 accuracy target could never fail.
- *What I changed:* I raised the accuracy target to 0.70. I also chose the reason for each number myself from the options Claude gave.

**Pre-labelling disclosure:**

Claude proposed my starting label definitions, pulled the comments from the HN API, and pre-labelled 186 of the 206 rows (marked `prelabeled`). I labelled the other 20 cold. I reviewed each batch and did not change any of the pre-labels. In unit 6, a separate Claude instance that never saw my labels acted as the second labeler for criterion 4, and another Claude pass re-applied my `analysis` rule to 143 rows for my improvement (24 changed, marked `prelabeled, relabeled`). Claude also drafted the README sections and verdicts from my numbers and choices, which I then read and edited. I labelled the 30 staff posts myself, with no help.

<!-- ═══════════════════════ UNIT 6 — THE TEST ═══════════════════════

     Don't fill these in during unit 5.
     ═══════════════════════════════════════════════════════════════════ -->

---

## Baseline vs. Trained

<!-- Both models on the same posts. `python baseline.py --trained results.json`
     prints this table for you. -->

All scores are on the same 31 held-out posts (`test_split.csv`). The trained column is seed 42 from `results.json`. The baseline is `facebook/bart-large-mnli`, zero-shot, reading my label definitions (`baseline_results.json`).

| Measure | Baseline | Trained | Difference |
|---|---|---|---|
| Overall accuracy | 0.226 | 0.484 | +0.258 |
| Macro F1 | 0.126 | 0.217 | +0.091 |
| F1 — `analysis` | 0.378 | 0.000 | -0.378 |
| F1 — `opinion` | 0.000 | 0.652 | +0.652 |
| F1 — `reaction` | 0.000 | 0.000 | 0.000 |

**What my definitions were worth.** I also ran the baseline with bare label names instead of my definitions (`baseline_bare_names.json`):

| Measure | Definitions | Bare names | Difference |
|---|---|---|---|
| Overall accuracy | 0.226 | 0.452 | -0.226 |
| Macro F1 | 0.126 | 0.451 | -0.325 |
| F1 — `analysis` | 0.378 | 0.471 | -0.093 |
| F1 — `opinion` | 0.000 | 0.500 | -0.500 |
| F1 — `reaction` | 0.000 | 0.381 | -0.381 |

My definitions made the baseline worse, not better. The baseline with definitions called most posts `analysis` (recall 0.78 on `analysis`, 0 on the other two). My guess is that my long definitions make awkward hypotheses for this model, but I have not tested that. I report the definitions run as my baseline, and the bare-names run is shown above for comparison. Against the bare-names baseline, my trained model is the one that looks worse, which I note below.

**What I predicted before I looked:**

I predicted my trained model would beat the baseline only on `opinion`, where it scored an F1 of about 0.65, and that the baseline would do better on `analysis` and `reaction`. I wrote this before seeing the baseline's numbers, but after I had already seen my unit 5 results, where the trained model predicted `opinion` for every test post.


**What the gap actually means:**

Against the baseline with my definitions, the trained model is ahead on accuracy (0.484 vs 0.226) and macro F1 (0.217 vs 0.126), but all of that comes from `opinion`. The trained model predicted `opinion` for all 31 test posts, so its 0.484 accuracy is just the share of `opinion` posts in the test split (15 of 31), and its F1 is 0 for `analysis` and `reaction`. Against the bare-names baseline (0.452 accuracy, macro F1 0.451, F1 above 0.38 on every label) the trained model is ahead on accuracy by one post and far behind on macro F1. So fine-tuning on about 144 examples did not teach the model my boundaries, and a general model reading only the label names recognizes all three labels better. My prediction was right on the label level: the trained model wins only on `opinion`, and the baseline wins on `analysis`, but it did not win on `reaction` with definitions (both scored 0).




---

## Run Log — Before

Trained on an Apple Silicon GPU (mps), torch 2.14.1, `distilbert-base-uncased`, 3 epochs, lr 2e-5, batch 16. 31 test posts per seed (15 `opinion`, 9 `analysis`, 7 `reaction`). Source: `results_three_seeds_before.json`.

| Criterion | Target | Seed 42 | Seed 7 | Seed 2024 | Verdict |
|---|---|---|---|---|---|
| 1. Overall accuracy | ≥ 0.70 | 0.484 | 0.484 | 0.484 | MISSED |
| 2. Every label F1 (lowest label shown) | ≥ 0.70 | 0.000 (`analysis`, `reaction`; `opinion` 0.652) | 0.000 (same) | 0.000 (same) | MISSED |
| 3. Largest label's share of `labels.csv` | ≤ 70% | 49.5% | 49.5% | 49.5% | MET |
| 4. Second labeler matches my 20 cold rows | ≥ 14 of 20 | 13 of 20 | 13 of 20 | 13 of 20 | MISSED |
| 5. Most-confident third minus least-confident third, accuracy (revised from halves, see `criteria.md`) | ≥ 20 points | 50 points (0.7 vs 0.2) | 20 points (0.6 vs 0.4) | 60 points (0.8 vs 0.2) | MET |

Criteria 3 and 4 do not depend on the training seed, so the same value is repeated in each column. Criterion 4 was measured once (see Verdicts).

### Confusion matrix

<!-- ⚠️ TYPED AS A MARKDOWN TABLE. The notebook prints one ready to paste.
     An image of a matrix earns nothing. -->

Seed 42 (seeds 7 and 2024 gave the identical matrix). Rows are true labels, columns are what the model guessed. Trained on an Apple Silicon GPU (mps), torch 2.14.1.

| true \ predicted | analysis | opinion | reaction |
|---|---|---|---|
| **analysis** | 0 | 9 | 0 |
| **opinion** | 0 | 15 | 0 |
| **reaction** | 0 | 7 | 0 |

**My biggest off-diagonal number, and what it means:**

The biggest off-diagonal number is 9: all 9 real `analysis` posts were called `opinion` (and all 7 `reaction` posts too), and nothing went the other way. The model did not learn the `analysis`/`opinion` boundary or the `reaction`/`opinion` boundary at all; it labelled every post with my most common label.




---

## Verdicts and Diagnoses

| # | Criterion | Target | Verdict | How I decided |
|---|---|---|---|---|
| 1 | Overall accuracy | ≥ 0.70 | MISSED | 0.484 on all three seeds. That is exactly the share of `opinion` posts in the test split (15 of 31), because the model called every post `opinion`. |
| 2 | Every label F1 | ≥ 0.70 | MISSED | `analysis` and `reaction` scored 0.000 on every seed, and `opinion` scored 0.652, which is also under 0.70. |
| 3 | Balance | ≤ 70% | MET | The largest label (`opinion`) is 102 of 206 rows, 49.5%. |
| 4 | Consistency | ≥ 14 of 20 | MISSED | A second labeler with only my written definitions and rules matched 13 of 20 of my cold labels (65%). The second labeler was a separate Claude instance that never saw my labels. It is an AI, not a person, so this is evidence about how reproducible my rule is, not a human benchmark. |
| 5 | Confidence (revised) | ≥ 20 points, top third vs bottom third | MET | Gaps of 50, 20 and 60 points. Seed 7 is exactly 20, so it holds but only just. The result is weak: each third has 10 posts, and the model predicted one label for everything, so mean confidence when right (about 0.53) and when wrong (about 0.48) barely differs. |

**Diagnoses**

**Criteria 1 and 2 (same cause): the model collapsed onto my biggest label.** The confusion matrix shows all 9 `analysis` and all 7 `reaction` test posts called `opinion`, identically on three seeds. I think two causes are behind this and I cannot fully separate them.

1. *Too few examples for the two smaller labels.* `analysis` has 56 rows and `reaction` has 48 in total, so roughly 39 and 34 landed in training, and DistilBERT with 3 epochs never moved off the majority label. The evidence is that F1 is exactly 0 on those two labels and nonzero only on `opinion`, with no change across seeds.
2. *A boundary I applied inconsistently.* My agreement report shows 5 of my 9 disagreements with staff were on the `analysis` boundary, and I erred in both directions (calling data-backed posts `hot_take` or `reaction`, and general claims `analysis`). The second labeler matched only 13 of my 20 cold rows, with 4 of the 7 mismatches on posts where I said `analysis` and the second labeler said something lower, plus one the other way.

What points away from pure label noise: the zero-shot baseline reading only label names scored an F1 above 0.38 on all three labels, while my trained model scored 0 on two. So the labels carry some signal, and the model failed to learn it from 144 examples. I treat too few examples as the main cause and labeling inconsistency as a real contributor, and I don't claim to know the split between them. One caution: 186 of the 206 labels were pre-labelled by Claude and I did not correct them, so inconsistency in my training data is partly Claude's application of my rules, not only mine.

**Criterion 4: a labeling problem, measured directly.** 13 of 20 matches against a second labeler is 65%, short of my 70%. The 7 mismatches were rows 3, 4, 5, 9, 17, 19 and 20 of my cold set. In 4 of them I said `analysis` and the second labeler said a lower label, which fits my staff-set pattern of drifting on the `analysis` boundary. This is the clearest evidence in the project that my definition leaves room to apply it two different ways.

**Criterion 5: met, but I wouldn't lean on it.** A model that gives one answer for everything has no real notion of when it is right, so a 20-point gap across 10-post thirds is mostly noise.

---

## Agreement Report

<!-- Your rate against the staff set, and every disagreement adjudicated.

     Remember you labelled these 30 under the STAFF taxonomy in
     data/staff_taxonomy.md, not your own — so every argument below is made
     from those definitions and those decision rules. -->

**Agreement rate:** 21 / 30 = 70%

**Disagreements**

**1.**
> *The post:* I tracked sleep against session quality for eleven weeks. Under seven hours the next day's intervals were 6-8 seconds per mile slower, every time bar one. Small sample, one runner, but it's consistent enough that I've stopped scheduling hard days after late shifts.
>
> *Staff said / I said:* `analysis` / `hot_take`
>
> *My call, and why:* The post is one runner's data, and the writer says so themselves ("small sample, one runner"). The conclusion is a personal scheduling decision, so I read it as closer to a general claim than a fact doing the work. I still think that is a defensible reading of rule 1.
>
> *Which pile:* (c)

**2.**
> *The post:* Everyone blames the hill at 18 miles. I split the field into people who ran the first half under 1:45 and over, and the slowdown after 18 was almost identical for both groups. If the hill were the problem the fast starters would fall apart harder.
>
> *Staff said / I said:* `analysis` / `hot_take`
>
> *My call, and why:* Splitting the field by first-half pace is a specific test, but the post gives no numbers for the result ("almost identical"). The rules don't say whether an analysis counts as a checkable fact when the figures are not shown.
>
> *Which pile:* (b)

**3.**
> *The post:* Carbon plates aren't the whole story. My times dropped 2% in the shoes and another 2% the following year in the same pair, when the only thing that changed was going from 35 to 50 miles a week.
>
> *Staff said / I said:* `analysis` / `reaction`
>
> *My call, and why:* It is about the writer's own times and experience, which I read as reaction under rule 2. I now see it has specific figures (2%, 2%, 35 to 50 miles) and is not recent, but I still think my reading is defensible.
>
> *Which pile:* (c)

**4.**
> *The post:* Ran the same loop in 4C and in 24C within a fortnight, same effort by heart rate. 90 seconds slower in the heat over six miles. People keep saying heat costs you 'a bit' and it's more than that.
>
> *Staff said / I said:* `analysis` / `reaction`
>
> *My call, and why:* It reports what happened on the writer's own runs, which I read as reaction under rule 2. It does have a specific comparison (4C vs 24C, 90 seconds), but my reading is defensible.
>
> *Which pile:* (c)

**5.**
> *The post:* Three of us ran the same session on the same day, different shoes, within 10 seconds of each other over 5 miles. I'm not saying shoes do nothing, but whatever they do was smaller than the noise between three people.
>
> *Staff said / I said:* `analysis` / `reaction`
>
> *My call, and why:* It reports what happened to the writer and friends on one day, which I read as closer to reaction. The comparison is specific (within 10 seconds over 5 miles), but I think my reading is defensible.
>
> *Which pile:* (c)

**6.**
> *The post:* The club needs to stop pretending the Tuesday session is for everyone. It isn't and it never has been.
>
> *Staff said / I said:* `hot_take` / `reaction`
>
> *My call, and why:* It reads as frustration, a feeling, with no argument offered, which I took as reaction. Rule 2 does say a heated post making a general claim is still a hot_take, but I think my reading is defensible.
>
> *Which pile:* (c)

**7.**
> *The post:* Road racing peaked in about 2015 and everything since has been marketing.
>
> *Staff said / I said:* `hot_take` / `analysis`
>
> *My call, and why:* Staff is right. "About 2015" is decoration on a conclusion the writer had already reached, with no fact carrying the argument. Rule 1 covered this and I applied it loosely.
>
> *Which pile:* (a)

**8.**
> *The post:* Every injury thread on this forum is the same injury and the same three replies.
>
> *Staff said / I said:* `hot_take` / `analysis`
>
> *My call, and why:* The rules don't say whether an observation about a forum's own threads counts as a checkable fact. "The same three replies" is a sweeping claim with no counted evidence, so staff's call is reasonable, but the rule doesn't settle it.
>
> *Which pile:* (b)

**9.**
> *The post:* Wind was so bad on the out section I was leaning at about 20 degrees. Coming back felt like cheating.
>
> *Staff said / I said:* `reaction` / `hot_take`
>
> *My call, and why:* Staff is right. It reports the writer's own run and how it felt, which is what rule 2 calls reaction. I applied it loosely.
>
> *Which pile:* (a)

**What the pattern in my disagreements tells me:**

Piles: (a) 2, (b) 2, (c) 5. Five of my nine disagreements (1 to 5) were staff `analysis` posts that I called `hot_take` or `reaction`. In each one the writer was reporting their own measurements or a comparison they ran, and I treated that as experience or opinion instead of as a checkable fact doing work. I defended that reading in four of the five. That is the opposite of what my own unit 5 rule says (a verifiable specific that supports the claim is `analysis`), so it is direct evidence that I applied the `analysis` boundary inconsistently. Disagreements 7 and 8 went the other way, where I called a general claim `analysis`, so I also let the boundary drift in the other direction.

---

## The Improvement

**What I changed:** I relabelled one boundary, `analysis` vs `opinion`, and nothing else (no change to the training settings, the `reaction` label, or my 20 cold rows). I wrote the rule down once more precisely and re-applied it to all 143 pre-labelled rows that were `analysis` or `opinion`. A separate Claude pass did the re-labelling blind to the old labels, in two halves, using this rule:

> A comment is `analysis` only if it contains a specific, checkable item (a number or measurement, including the writer's own, a date, version, named product, event or paper used as evidence, a source, link or quote, or a described mechanism with named parts) **and** that item supports the comment's main claim. A name, number or link that is only mentioned in passing, decorates a conclusion, or names the topic does not count. Everything else is `opinion`.

**24 of the 143 rows changed:** 22 went from `analysis` to `opinion` and 2 went from `opinion` to `analysis`. They are marked `prelabeled, relabeled` in the note column. Counts after: `opinion` 122 (59.2%), `analysis` 36 (17.5%), `reaction` 48 (23.3%), total 206.

Most of the 22 were short, bare statements ("It's a draft RFC", "That's for FDM. This is resin.", "your linux distro is built on perl and python") that I had inconsistently called `analysis` before because a specific word was present, even though nothing supported a claim. A few of the changes are arguable: row 1 (the SendCutSend comparison) and the Bushby press-coverage comment could reasonably stay `analysis`.

**What I expect before I run it:** this makes `analysis` smaller (from 56 to 36 rows) while making its definition tighter, so I think the model may still collapse onto `opinion`, or get worse on `analysis`, because it now has even fewer examples of the label it was already missing.

**Which diagnosis pointed at it:** my agreement report. 5 of my 9 staff disagreements were on the `analysis` boundary, in both directions, and a second labeler matched only 13 of my 20 cold rows. Together they pointed at a boundary applied inconsistently rather than at the training settings.

### Run Log — After

Same model and settings, same three seeds, trained on the relabelled `labels.csv`. Source: `results_three_seeds_after.json`. The test split is different from the "before" run (31 posts: 18 `opinion`, 6 `analysis`, 7 `reaction`, against 15 / 9 / 7 before), because the split is stratified on the labels I changed.

| Criterion | Target | Seed 42 | Seed 7 | Seed 2024 | Verdict |
|---|---|---|---|---|---|
| 1. Overall accuracy | ≥ 0.70 | 0.581 | 0.581 | 0.581 | MISSED |
| 2. Every label F1 (lowest label shown) | ≥ 0.70 | 0.000 (`analysis`, `reaction`; `opinion` 0.735) | 0.000 (same) | 0.000 (same) | MISSED |
| 3. Largest label's share of `labels.csv` | ≤ 70% | 59.2% | 59.2% | 59.2% | MET |
| 4. Second labeler matches my 20 cold rows | ≥ 14 of 20 | 13 of 20 | 13 of 20 | 13 of 20 | MISSED |
| 5. Most-confident third minus least-confident third (revised) | ≥ 20 points | 50 points (0.9 vs 0.4) | 60 points (0.9 vs 0.3) | 50 points (0.8 vs 0.3) | MET |

Criterion 4 did not change, and could not have: the relabelling left my 20 cold rows alone, so I did not re-measure it.

**Did it help, and how do I know:**

No. The model still predicted `opinion` for every test post on every seed. Accuracy went from 0.484 to 0.581, but that is not an improvement: the test split now has 18 `opinion` posts out of 31, and 18/31 is exactly 0.581, so the model scored the share of `opinion` posts it was always going to get. F1 for `analysis` and `reaction` is still 0.000, and macro F1 moved from 0.217 to 0.245 only because `opinion` F1 rose with its larger share. The before and after numbers are also on different test splits, so they are not a clean comparison. What I can say is that making the `analysis` boundary more consistent did not teach the model anything, and it left `analysis` with fewer examples (36 instead of 56), which probably made the data problem I diagnosed worse, as I predicted.




---

## What's Still Broken

Criteria 1, 2 and 4 are still missed.

- **Criterion 1 (accuracy ≥ 0.70) and criterion 2 (every label F1 ≥ 0.70):** the model predicts `opinion` for everything. I would add many more `analysis` and `reaction` examples (my 206 rows leave about 25 and 34 of them in training), then retrain, and consider weighting the smaller labels. I stopped after one change because the assignment asks for one, and my relabel was the change my diagnosis supported.
- **Criterion 4 (second labeler ≥ 14 of 20):** I would have a person, not Claude, label my cold rows from my written rule, then rewrite the `analysis` rule around the disagreements and reapply it. I did not re-measure it after relabelling.

**The gap between what I meant my labels to capture and what the model learned:**
I meant `analysis` to mark a post whose checkable specific actually supports its claim, and `reaction` to mark a short response with no claim. The confusion matrix shows the model learned none of that: in both runs, on all three seeds, every `analysis` and every `reaction` test post was labelled `opinion`.

<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 5

       [ ] criteria.md has five numbered criteria, each naming a NUMBER
       [ ] Each has a reason underneath tied to your data or taxonomy
       [ ] labels.csv: at least 150 rows, text/label/note, ONE file not split
       [ ] No label above 70%
       [ ] All five unit 5 sections have real content
       [ ] Label Taxonomy includes the decision rule for your hardest boundary
       [ ] The Dataset includes three hard cases
       [ ] results.json and test_split.csv committed (the notebook does this)
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN

     SUBMISSION CHECKLIST — unit 6

       [ ] Baseline vs. Trained table, with your prediction written beforehand
       [ ] Run Log — Before, five criteria across three seeds
       [ ] Confusion matrix TYPED AS A MARKDOWN TABLE
       [ ] A verdict on every criterion
       [ ] A diagnosis for every miss, using the agreement report as evidence
       [ ] Agreement Report with every disagreement adjudicated
       [ ] One improvement, with Run Log — After
       [ ] What's Still Broken
       [ ] results_three_seeds_before.json, results_three_seeds_after.json,
           baseline_results.json and
           agreement_results.json committed
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
