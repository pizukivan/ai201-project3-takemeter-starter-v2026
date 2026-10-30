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

**Counts per label:**

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

**Split sizes:** train 144 / validation 31 / test 31 (stratified 70/15/15). Test per label: `opinion` 15, `analysis` 9, `reaction` 7. `reaction` has fewer than about 8 in the test split, so its scores will be jumpy from seed to seed.

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

Claude proposed my starting label definitions, pulled the comments from the HN API, and pre-labelled 186 of the 206 rows (marked `prelabeled`). I labelled the other 20 cold. I reviewed each batch and did not change any of the pre-labels.

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

<!-- Five criteria across three seeds. The notebook's section 6 prints the
     spread table; the Target and Verdict columns are yours. -->

| Criterion | Target | Seed 42 | Seed 7 | Seed 2024 | Verdict |
|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |
| 2.  |  |  |  |  |  |
| 3.  |  |  |  |  |  |
| 4.  |  |  |  |  |  |
| 5.  |  |  |  |  |  |

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

<!-- MET or MISSED against LAST UNIT's target. The target has to hold across
     all three seeds, not turn up sometimes. -->

| # | Criterion | Target | Verdict | How I decided |
|---|---|---|---|---|
| 1 |  |  |  |  |
| 2 |  |  |  |  |
| 3 |  |  |  |  |
| 4 |  |  |  |  |
| 5 |  |  |  |  |

**Diagnoses**

<!-- For each miss: the cause, and how you know. The four common causes are:
     too few examples for a label, a boundary you applied inconsistently, a
     genuinely hard label pair, and a task the model can't reach from this
     much data.

     ⚠️ Use your agreement report as evidence. It is the only instrument you
     have that can tell a LABELLING problem from a MODEL problem, and this
     section is graded on whether you used it that way. -->



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

**What I changed:**

**Which diagnosis pointed at it:**

### Run Log — After

| Criterion | Target | Seed 42 | Seed 7 | Seed 2024 | Verdict |
|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |
| 2.  |  |  |  |  |  |
| 3.  |  |  |  |  |  |
| 4.  |  |  |  |  |  |
| 5.  |  |  |  |  |  |

**Did it help, and how do I know:**

<!-- If it didn't, say so. Relabelling that didn't help is a genuinely
     interesting result and earns full credit. -->



---

## What's Still Broken

<!-- For each criterion still missed: what you'd do, and why you stopped. -->



**The gap between what I meant my labels to capture and what the model
learned:**
<!-- Two sentences. Your confusion matrix is the evidence. -->



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
