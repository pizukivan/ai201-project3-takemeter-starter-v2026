# Acceptance criteria — TakeMeter

Five criteria that say what "working" means for this classifier, written in
unit 5 **before** anything was trained.

**All five are yours this time.** None are given. You've had two projects of
practice.

An acceptance criterion names a number. *"The model is accurate"* is an
opinion. *"Every label has an F1 of at least 0.60 on the held-out set"* is a
criterion.

Under each, write a sentence or two on **why that number**. A reason that says
something about your data or your taxonomy earns credit — *"I picked 0.60 F1
for `reaction` because it's my smallest label and I only have about 50
examples of it"*. A reason that could be attached to any project does not.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

---

## Pick numbers you can defend

Not numbers that sound impressive. Three labels means a coin-flip guesser gets
about 33%, so a target of 0.40 is barely a target. Your number should sit
somewhere you'd honestly call useful.

**Cover at least three of these five areas.** They're here as prompts, not as a
form to fill in — a criterion that fits none of them is fine if it names a
number.

| Area | A question it could answer |
|---|---|
| Overall accuracy | How often does it need to be right to be worth using? |
| Per-label performance | Is one label allowed to be much worse than the others? |
| Balance | How lopsided can your label counts get before it's a problem? |
| Consistency | If someone else labelled the same posts, how often should you agree? |
| Confidence | Should a confident prediction be right more often than an unsure one? |

Two things worth knowing before you pick numbers, because both will affect
whether you hit them:

- **Your smallest label will have the jumpiest score.** If a label has 50
  examples, about 8 land in the test split. An F1 computed on 8 examples moves
  a lot between seeds. A target for that label should be looser than one for
  your biggest label, and saying so is a good reason.
- **Unit 6 tests across three seeds, and the target has to hold across all
  three.** A target of 0.65 against results of 0.71, 0.62, 0.68 is a **miss**.
  Pick with that in mind — it is stricter than it first sounds.

---

## 1.

**Overall accuracy:** accuracy on the held-out test split is at least 0.70.

**Why this target:**

I raised accuracy to 0.70 to match my per-label F1 target, since every label at 0.70 F1 already implies accuracy near 0.70. Always guessing `opinion` scores about 0.50 on my data (it is 49.5% of my rows), so 0.70 is clearly better than guessing.

---

## 2.

**Per-label performance:** every label (`analysis`, `opinion`, `reaction`) has an F1 of at least 0.70 on the held-out test split.

**Why this target:**

A label the model gets wrong half the time isn't useful, so every label should be reliable, even though the test split is small (about 31 posts, roughly 7 `reaction`) and my smallest labels will be noisy.

---

## 3.

**Balance:** no single label is more than 70% of the rows in `labels.csv`.

**Why this target:**

I used the project's own cap. My largest label (`opinion`) is about 49.5% of 206 rows, so I'm well under it.

---

## 4.

**Consistency:** when a second labeler independently labels my 20 `cold` rows using only my written definitions and decision rule, at least 14 of 20 labels (70%) match mine.

**Why this target:**

Labeling `opinion` vs `analysis` is a judgment call on short, ambiguous Hacker News comments, so I picked a moderate bar for a three-label task.

---

## 5.

**Confidence:** on the test split, the half of predictions with the highest model confidence is at least 20 percentage points more accurate than the half with the lowest confidence.

**Why this target:**

I picked this on purpose as an ambitious target. It may be hard to hit with about 31 test posts, and I'd learn from missing it.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 6 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath, like this:

         ## 2. Every label performs acceptably

         The model performs well on all labels.

         **Why this target:** ...

         > **Revised in unit 6:** Every label has an F1 of at least 0.60 on
         > the held-out set.
         >
         > **Why revised:** "performs well" gave me nothing to check. I
         > couldn't produce a verdict from it at all.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "Overall accuracy of at least 0.65" → "at least 0.55", because
            0.65 turned out to be optimistic for 200 examples.

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.
     ───────────────────────────────────────────────────────────────────────── -->
