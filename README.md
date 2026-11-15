# Provenance Guard

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first.**
>
> Once `python test.py` passes, in one terminal:
>
> ```bash
> python app.py
> ```
>
> and in another:
>
> ```bash
> curl -X POST http://127.0.0.1:5000/ping \
>   -H "Content-Type: application/json" \
>   -d '{"message": "hello"}'
> ```
>
> That's a web service. Something sent it text and it sent something back.
> Everything this unit is filling in the middle.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     Unit 7 asks for the first five sections. Unit 8 adds the five below.

     Everything is pasted as TEXT. A pasted curl response gets full credit;
     a picture of your terminal gets none.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 7 — THE BUILD ═══════════════════════ -->

## What This Does

<!-- The path a submission takes, from arriving to the label a reader sees.
     Six or seven steps — this is the trace you wrote in Milestone 1. -->

1. **Arrives at `POST /submit`:** Assigns the text a submission ID and timestamp.
2. **Validation:** Rejects empty/too short submission and missing `creator_id`, no score produced (status 400).
3. **`detector.py`:** A local model measures how predictable the text is and produces an AI-likeness score.
4. **`stylometry.py`:** Measures text style (line-length variation, repeated openings, stock-word rate) and adds a second score.
5. **Combining, `scoring.py`:** Merges the two scores into one 0–1 score using the weights in `config.py`.
6. **Label, `scoring.py`:** Compares the score to the human and AI thresholds in `config.py` and returns a guess ("human", "unsure" or "ai") + the plain-English sentence the reader sees.
7. **Log and respond:** Saves both signals + score + guess + label to the `content_id` and returns as `/submit` response.

**`POST /appeal`:** If a writer disagrees with the result, they send a POST request with the `content_id` and their reasoning. The service finds the saved record, marks it as `under_review`, and returns its status. 

**`GET /content/<id>`:** At any point, `GET /content/<id>` returns the current status of one item.

### Endpoints

**POST /submit**
- In: `text` (string, required), `creator_id` (string, required)
- Out: `content_id`, `guess` ("human" | "unsure" | "ai"), `confidence` (0–1), `label` (the sentence a reader sees), `model_score`, `style_score`
- Errors: 400 if `text` is empty or too short; 503 if the detector can't load

**POST /appeal**
- In: `content_id`, `reasoning`
- Out: `content_id`, `status` ("under_review")
- Errors: 404 if `content_id` doesn't exist

**GET /content/<id>** (stretch)
- Out: `content_id`, `status` ("decided" or "under_review"), `guess`, `label`
- Errors: 404 if `content_id` doesn't exist

**GET /log?limit=N**
- Out: the last N saved entries

### Stretch Features

1. **Third signal: stock-word rate** Counts how often words from a list of stock poetic imagery ("whispers", "tapestry", "echoes", etc.) appear per 100 words. (add to combining rule and decide how to weigh 3 signals)
2. **`GET /content/<id>`.** Returns the current status of one submission (scored or appealed), along with its guess and label.


---

## Signals and Scoring

<!-- For each signal: what property it measures, why that might differ between
     human and AI writing, and WHAT IT CAN'T SEE.

     The blind spot is not optional. If you can't name it you don't understand
     the signal yet — and unit 8's attack set is built out of blind spots. -->

### Signal one — the model signal

**What it measures:** How predictable the text is to a local language model (0-1 score)

**Why that might differ between human and AI writing:** A language model writes by picking likely next words, so its own output can be easy for another model to predict. Human writing wanders more with odd word choices and specific details a model doesn't expect.

**What it can't see:** Plainly written human work (a second-language writer, a younger writer, someone writing simply on purpose) may score as AI-like, while ornate AI text full of rare words can score as human.

### Signal two — the style signal

**What it measures:** The shape of the text using 3 measures weighted equally:
- spread of line or sentence length (standard deviation divided by mean; poems are measured by line, and a `/` beside a space counts as a line break)
- vocabulary variety (the share of unique words, averaged over 25-word windows so length doesn't dominate)
- punctuation variety (how many of 10 kinds of punctuation appear, capped at 6)

**Why that might differ between human and AI writing:** Generated text tends toward even, mid-length lines and a narrower, safer vocabulary with flat punctuation, while human writing is lumpier: a three-word line next to a long one, and more varied marks.

**What it can't see:** Meaning, because it only measures shape. It also flags real writers whose style is regular on purpose, such as formal verse and plain minimalist poems.

### The combining rule

<!-- Write the rule in words, then give the numbers. What happens when the two
     signals disagree? -->

**The rule:** An equally weighted average of the two scores. If 2 signals disagree and the average is above 0.5 (more likely AI), the average is pulled back toward 0.5 by the score difference. A human-leaning average is preserved. Confidence is how far the final score is from 0.5.

**The numbers:** Weights 0.5 (model) and 0.5 (style), set in `config.py`. When the average is above 0.5: Score = 0.5 + (average − 0.5) × (1 − |model − style|);Confidence = |score − 0.5| × 2

**Where it lives:** `scoring.py::combine_signals`

<!-- ⚠️ The grader checks your code against that line. If your rule lives
     somewhere else, say where. -->

**A case where my two signals split, and what my rule does with it:** For a short human poem ("The bottom of my shoes / are clean / From walking in the rain"), the model signal scored 0.039 (very human) while the style signal scored 0.458 (near the middle, since three even short lines look regular). The two disagree by 0.42. The average is 0.248, below 0.5, so my rule leaves it alone and the combined score stays human-leaning at about 0.25. Had it gone the other way, with the model signal high and the style signal low, the rule would have pulled the score back toward 0.5 instead of accusing.


---

## Label Variants

<!-- The exact text of all three labels, as a reader would see them, and the
     score range each covers.

     Write them for someone who has never heard the word "threshold". -->

| Label | Score range | The exact text a reader sees |
|---|---|---|
| high-confidence human |  |  |
| unsure |  |  |
| high-confidence AI |  |  |

**Why I worded the "unsure" one this way:**
<!-- It's the hardest of the three. It has to admit uncertainty without
     sounding like an accusation, because the person reading it may well have
     written every word themselves. -->



---

## Sample Run

<!-- One submission and one appeal, pasted as text: the request, the response,
     and the log entries. -->

**A submission**

<!-- The response needs content_id, guess, confidence, label, AND both
     signal scores - model_score and style_score. Unit 8's tools read those
     two off the response, so leaving them out costs you next unit's
     diagnostics. -->

```bash
$ curl -X POST http://127.0.0.1:5000/submit \
-H "Content-Type: application/json" \
-d '{"text": "The bottom of my shoes/ are clean/ From walking in the rain", "creator_id": "Jack Kerouac"}'
```

```json
{
  "confidence": 0.5,
  "content_id": "c34e2388-996d-4c5b-a327-faf0b4d84ee7",
  "guess": "unsure",
  "label": "placeholder",
  "model_score": 0.039,
  "style_score": 0.0
}
```

**An appeal**

```bash
$ curl -X POST http://127.0.0.1:5000/appeal \
  -H "Content-Type: application/json" \
  -d '{"content_id": "...", "reasoning": "..."}'
```

```json

```

**What the log shows afterwards**

```bash
$ curl "http://127.0.0.1:5000/log?limit=3"
```

```json
{
  "count": 3,
  "entries": [
    {
      "creator_id": "robbie",
      "event": "rejected",
      "reason": "missing_or_empty_text",
      "status": "rejected",
      "timestamp": "2026-11-14T07:45:05+00:00"
    },
    {
      "creator_id": "robbie",
      "event": "rejected",
      "reason": "missing_or_empty_text",
      "status": "rejected",
      "timestamp": "2026-11-14T07:45:15+00:00"
    },
    {
      "creator_id": "robbie",
      "event": "rejected",
      "reason": "text_too_short",
      "status": "rejected",
      "timestamp": "2026-11-14T07:45:26+00:00"
    }
  ]
}
```

---

## How I Used AI

**Moment 1**

- *What I asked for:*
- *What came back:*
- *What I changed:*

**Moment 2**

- *What I asked for:*
- *What came back:*
- *What I changed:*

<!-- ═══════════════════════ UNIT 8 — THE TEST ═══════════════════════ -->

---

## Rate Limiting

**My limits:** ___ per minute, ___ per day

**Why those numbers and not others:**
<!-- Think about two people: a real writer submitting their own work a few
     times an hour, and a script sending a thousand variations to map your
     thresholds. Your numbers have to be liveable for the first and hostile to
     the second. -->

**What a caller counts as:** <!-- per address, or per creator_id? They fail
differently — one script can look like a thousand callers, and one household
can look like one. -->

**The run of status codes when I pushed past it:**

<!-- python run_attacks.py --flood 15 -->

```

```

---

## Attack Run — Before

<!-- Every attack with its outcome and a held-or-broke mark.
     `run_attacks.py --label before` produces the table. -->



**Ten audit log entries from the run**

<!-- Choose entries that show the interesting failures, not the ten easiest. -->

```json

```

**The single worst outcome** — the one I'd least want to explain to a writer
whose work got caught by it:



---

## Run Log and Verdicts

<!-- Your five criteria across three trials. `run_eval.py --label before`. -->

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |
| 2.  |  |  |  |  |  |
| 3.  |  |  |  |  |  |
| 4.  |  |  |  |  |  |
| 5.  |  |  |  |  |  |

**Real output from one trial**, pasted as text, naming the file and function
that produced it:

```json

```

**Diagnoses**

<!-- For each miss AND each broken attack: the stage and the mechanism.

     Your service has five stages: the route, signal one, signal two, the
     combining rule, and the label mapping.

     Not a diagnosis:  "The evasion attacks got through."
     A diagnosis:      "Four evasion inputs got through. All four had typos
                        added to AI-generated text. My second signal measures
                        word variety, and typos raise word variety — so it read
                        them as more human. My combining rule weights that
                        signal at 0.6, so it dragged the whole score down."

     Look for a pattern. Five failures on one signal is one problem, not five. -->



---

## The Improvement

**What I changed:**

**Which diagnosis pointed at it:**

### Attack Run — After

<!-- Same format. `run_attacks.py --label after` -->



**Did it help, and how do I know:**

<!-- Widening the unsure band usually fixes one problem and creates another.
     Reporting that trade honestly is worth full credit. -->



---

## What's Still Broken

<!-- For each attack still getting through and each criterion still missed:
     what you'd do, and why you stopped. -->



**The trade I'd make if this ran for real:**

<!-- Two sentences. Fewer wrong accusations means more AI text getting through.
     Say which way you'd go and WHO PAYS FOR IT. -->



<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 7

       [ ] criteria.md has five numbered criteria, each naming a target
       [ ] Each has a reason underneath
       [ ] POST /submit returns content_id, guess, confidence, label,
           model_score and style_score
       [ ] Two signals that measure DIFFERENT things, each with a named blind spot
       [ ] Three label variants, all reachable, written out in full
       [ ] POST /appeal changes status and writes to the log
       [ ] Signals and Scoring names scoring.py::combine_signals
       [ ] Sample Run: a submission AND an appeal, as pasted text
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN

     SUBMISSION CHECKLIST — unit 8

       [ ] Rate limiting on /submit, with your numbers justified
       [ ] The pasted run of status codes showing 429
       [ ] Audit log with all nine fields, recording rejections too
       [ ] The attack set run IN FULL, every input with an outcome
       [ ] Ten audit log entries pasted as text
       [ ] Five criteria across three trials
       [ ] A diagnosis for every miss and every broken attack — stage AND mechanism
       [ ] One improvement, with Attack Run — After
       [ ] What's Still Broken, including the production trade
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
