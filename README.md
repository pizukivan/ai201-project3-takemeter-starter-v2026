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

| Measure | Baseline | Trained | Difference |
|---|---|---|---|
| Overall accuracy |  |  |  |
| Macro F1 |  |  |  |
| F1 — `label_one` |  |  |  |
| F1 — `label_two` |  |  |  |

**What I predicted before I looked:**
<!-- Milestone 1 asks you to write this BEFORE seeing the trained numbers. A
     prediction made afterwards isn't one. -->

**What the gap actually means:**
<!-- If the baseline matched your trained model, your fine-tuning added
     nothing — and that is a real finding, not a failure. Say it plainly. -->



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

| true \ predicted |  |  |  |
|---|---|---|---|
| **** |  |  |  |
| **** |  |  |  |
| **** |  |  |  |

**My biggest off-diagonal number, and what it means:**
<!-- Not "the model made mistakes" — WHICH boundary it didn't learn, and which
     direction. "7 real analysis posts were called hot_take and only 3 went the
     other way" is a direction, not just an error rate. -->



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

**Agreement rate:** ___ / 30 = ___%

<!-- Nobody grades this number. A 60% who argues every disagreement from the
     stated rules beats a 95% who wrote "staff was right" nine times. Several
     of the 30 were chosen because they're genuinely ambiguous — you should be
     winning some of these. -->

**Disagreements**

<!-- Three lines each: the post, both labels, and who you think is right and
     why — grounded in the staff definitions you were both applying.

     Then sort each into one of three piles:
       (a) the rule covered it and I applied it loosely → a consistency problem
       (b) the rule genuinely doesn't say               → a gap in the definitions
       (c) the rule is ambiguous here and my reading is defensible → argue it.
           This is a legitimate win.

     Pile (a) is the one that matters most for your diagnosis: if you applied a
     written rule two different ways on 30 posts, that is direct evidence about
     what you did across your own 200. -->

**1.**
> *The post:*
>
> *Staff said / I said:*
>
> *My call, and why:*
>
> *Which pile:*

**2.**
> *The post:*
>
> *Staff said / I said:*
>
> *My call, and why:*
>
> *Which pile:*

**What the pattern in my disagreements tells me:**



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
