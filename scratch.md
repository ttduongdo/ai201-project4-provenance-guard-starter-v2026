## Signals and Scoring

### Signal one — the model signal

**What it measures:** How predictable the text is to a local language model (its perplexity), squashed onto a 0–1 score where higher means more likely AI.

**Why that might differ between human and AI writing:** A language model writes by picking likely next words, so its own output is easy for another model to predict. Human writing wanders more, with odd word choices and specific details a model doesn't expect.

**What it can't see:** Who wrote the text. It rewards common words in short, ordinary sentences, so plainly written human work (a second-language writer, a younger writer, someone writing simply on purpose) scores as machine-like, while ornate AI text full of rare words can score as human. It is also built for prose, so very short poems give it little to work with and the score is noisy.

### Signal two — the style signal

**What it measures:** The shape of the text, using three measures, each flipped so higher means more likely AI, and averaged equally:
- spread of line or sentence length (standard deviation divided by mean; poems are measured by line, and a `/` beside a space counts as a line break)
- vocabulary variety (the share of unique words, averaged over 25-word windows so length doesn't dominate)
- punctuation variety (how many of 10 kinds of punctuation appear, capped at 6)

**Why that might differ between human and AI writing:** Generated text tends toward even, mid-length lines and a narrower, safer vocabulary with flat punctuation, while human writing is lumpier: a three-word line next to a long one, and more varied marks.

**What it can't see:** Meaning. It only measures shape, so anything that changes the shape without changing the substance (adding typos, breaking up lines, mixing in a quotation, throwing in varied punctuation) moves it, and that is cheap to do on purpose. It also flags real writers whose style is regular on purpose, such as formal verse and plain minimalist poems.

### The combining rule

**The rule:** A weighted average of the two scores. If the two signals disagree and the average leans AI (above 0.5), the average is pulled back toward 0.5 by the size of the disagreement, so a high-confidence AI result needs both signals to point the same way. A human-leaning average is left alone. Confidence is how far the final score is from 0.5.

**The numbers:** Weights 0.5 (model) and 0.5 (style), set in `config.py`. When the average is above 0.5: score = 0.5 + (average − 0.5) × (1 − |model − style|). Confidence = |score − 0.5| × 2.

**Where it lives:** `scoring.py::combine_signals`

**A case where my two signals split, and what my rule does with it:** For a short human poem ("The bottom of my shoes / are clean / From walking in the rain"), the model signal scored 0.039 (very human) while the style signal scored 0.458 (near the middle, since three even short lines look regular). The two disagree by 0.42. The average is 0.248, below 0.5, so my rule leaves it alone and the combined score stays human-leaning at about 0.25. Had it gone the other way, with the model signal high and the style signal low, the rule would have pulled the score back toward 0.5 instead of accusing.