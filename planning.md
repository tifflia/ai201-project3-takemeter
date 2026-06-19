# TakeMeter — planning.md

## Community

The community I'm choosing is the subreddit for fans of Avatar: The Last Airbender, The Legend of Korra, the comics, the upcoming Avatar Studios animated movies and other projects, novels, games, and all other Avatar content ([r/TheLastAirbender](https://www.reddit.com/r/TheLastAirbender/)). This community is a good fit for a classification task as fans here are known to have strong opinions in topics ranging from ATLA vs. Korra discourse, character analysis, and lore discussion. The Avatar universe has enough narrative complexity that it makes for rich and plentiful analysis from the fandom.

---

## Labels

### `analysis`
The post constructs an argument grounded in specific textual evidence. A skeptical reader could evaluate the claim using what's provided. Doesn't have to be correct, just has to reason rather than assert.

**Key signal:** Episode references, dailogue quotes, character arc structure, or worldbuilding details used to support a claim.

**Examples:**
- "Aang's airbending style is deliberately passive until Book 3 — the show uses offense/defense ratio as a visual marker of his emotional state"
- "Zuko's Blue Spirit arc recontextualizes his entire season 1 motivation: he wasn't hunting the Avatar for honor, he was hunting for his father's approval, and the show signals this before he consciously knows it"

### `hot_take`
A confident, often controversial opinion stated without meaningful supporting evidence. The claim might be true, but the post asserts rather than argues.

**Key signal:** First-person opinion framing ("I think," "unpopular opinion"), no episode-level evidence, asserting of conclusion without logical chain

**Examples:**
- "Katara is a better protagonist than Aang and the show would've been stronger with her as the lead"
- "Toph is overrated — she's comic relief who happens to be powerful"

### `bait`
Structure to provoke rather than discuss. The rhetorical framing is the point as it invites outrage or pile-on responses rather than genuine engagement with an idea.

**Key signal:** Absolutist language ("objectively," "anyone who disagrees"), ATLA-vs-Korra divide setup, strawmanned fandom positions, hyperbole that can't be defused without losing the entire claim.

**Examples:**
- "Korra ruined the franchise and anyone defending it is just coping"
- "ATLA fans only hate Korra because they can't handle a female protagonist — it's embarrassing"

---

## Hard Edge Cases

There will likely be many posts that ambiguously fall between `hot_take` (genuine opinion) and `bait` (hyperbole designed for outrage). To handle these cases, remove the provocative framing/phrasing and decide whether there are still any coherent, discussable points. If there are, label it `hot_take` and if not, label it `bait`.

If a post makes a falsifiable causal or structural claim about how the narrative works, label it `analysis`. Even if it doesn't cite specific scenes, an argument still exists and a reader could verify or refute such posts by rewatching. A `hot_take` would assert a conclusion without a logical mechanism.

---

## Data Collection Plan

### Sources

Posts will be collected manually by browsing r/TheLastAirbender across three queues:

- **Hot and Top (all time)** — surfaces high-engagement posts likely to include bait and hot takes
- **New** — captures a more representative slice of everyday discourse, including analysis posts that didn't go viral
- **Search by keyword** — targeted browsing for underrepresented labels (e.g., searching "theory," "breakdown," "essay" to surface analysis; "unpopular opinion," "controversial take" for hot takes)

Post titles, body text, and top-level comments are all fair game, since bait often lives in the title while analysis often lives in the comments.

### Target counts

| Label | Target | Rationale |
|-------|--------|-----------|
| `analysis` | 70 | Naturally rarer; requires targeted search to surface enough examples |
| `hot_take` | 70 | Common but easily confused with bait at the boundary |
| `bait` | 60 | Frequent but risks over-representing ATLA-vs-Korra framing |
| **Total** | **200** | — |

A roughly balanced dataset avoids the model collapsing to majority-class prediction. Exact balance isn't required, but no label should fall below 50 examples.

### If a label is underrepresented after 200 examples

The most likely shortfall is `analysis` — genuinely argued posts are rarer than opinionated ones. If `analysis` is below 50 after an initial collection pass:

1. **Targeted keyword search:** Pull posts tagged "essay," "theory," "meta," "breakdown," or "analysis" from the subreddit's flair system or full-text search. These are self-identified and tend to meet the label criteria.
2. **Expand to adjacent communities:** r/ATLAFanTheories, r/ATLA, and r/legendofkorra have higher concentrations of analytical posts. These communities discuss the same source material, so examples transfer cleanly.
3. **Extend to comment threads:** Top comments on episode discussion threads often contain dense analysis that never gets its own post. Scrape comment bodies from weekly discussion threads.

If `bait` is overrepresented (likely if hot/top queues skew toward engagement), downsample randomly rather than discarding — keep a held-out pool in case balance shifts during annotation.

---

## Evaluation Metrics

**Per-class F1:** A model's precision and recall for one specific label. Accuracy is misleading when classes aren't perfectly balanced. A model that learns to predict a `hot_take` 80% of the time will look decent on total accuracy while being useless. F1 per class tells me whether the model is not just learning the majority label. F1 will be reported for `analysis`, `hot_take`, and `bait` separately.

**Macro F1:** Average of per-class F1s, unweighted by class size. This is the headline number because it treats a failure on `analysis` as equally costly as a failure on `hot_take`. Weighted F1 would let the model hide poor `analysis` performance behind majority-class wins.

**Confusion Matrix:** A table displaying exact numbers of predictions that show where and how the model makes mistakes. For this task, the interesting failure modes are directional: does the model confuse `bait` and `hot_take` (the structurally similar pair)? Does it over-predict `hot_take` because it's the middle-ground label? The matrix makes those patterns visible and informs where my taxonomy decision rules need tightening.

---

## Definition of Success

### Minimum bar (genuinely useful)

A macro-F1 of **0.70** across all three labels, with no individual class F1 below **0.60**. Below this, at least one label is being predicted near-randomly, which means the classifier isn't reliably distinguishing the concepts it was built to separate.

The floor on `bait` specifically matters most. A community tool that mislabels bait as hot_take at high rates isn't surfacing the signal the tool exists to provide. If `bait` F1 falls below 0.60, the classifier isn't deployment-ready regardless of the macro average.

### Good enough for deployment

- Macro-F1 ≥ **0.75**
- Per-class F1 ≥ **0.70** on all three labels
- Confusion matrix shows the dominant error is `bait` ↔ `hot_take` (acceptable: these are genuinely adjacent), not `analysis` ↔ `bait` (unacceptable: these are structurally distinct)

At this level the classifier is useful as a soft signal — flagging posts for human review, tracking label distribution over time, or surfacing trends — without being trusted for high-stakes automated decisions like removal.

--- 

## AI Tool Plan

### Label stress-testing

Before annotating, I'll give Claude my three label definitions and the `bait` vs. `hot_take` decision rule, then prompt it to generate 10 synthetic posts that sit at the boundary between two labels. I'll specifically ask for 5 at the `bait`/`hot_take` boundary and 5 at the `hot_take`/`analysis` boundary, since those are the two pairs most likely to cause annotator disagreement.

For each generated post, I'll attempt to apply my decision rule and assign a label. If I hesitate in classifying, that's a signal the definition is under-specified. Specifically, I'll look for:
- Posts where the decision rule produces a different answer depending on how generously I read the post
- Posts where the key signal for one label (e.g., an episode reference) co-occurs with the key signal for another (e.g., absolutist language)

Any case I can't resolve cleanly becomes a new entry in the Hard Edge Cases section with an explicit decision rule added before annotation begins. The goal is zero ambiguous cases going into labeling, not zero ambiguous posts in the wild.

### Failure analysis

After evaluation, I'll export the model's wrong predictions as a list of (post text, true label, predicted label) triples and give them to Claude with the following prompt: *"Here are posts my classifier got wrong. Identify patterns in the mistakes. Are certain topics, post lengths, rhetorical structures, or phrasings systematically misclassified? Group the errors and describe what they have in common."*

I'll specifically look for:
- **Topic bleed:** Are errors clustered around specific subjects (Korra, Zuko, a particular episode)? If so, the model may have learned topic instead of discourse structure.
- **Length artifacts:** Are short posts disproportionately misclassified? Short posts have less signal and may default to `hot_take` regardless of actual label.
- **Framing false positives:** Does the model misclassify `hot_take` posts as `bait` when they use strong language, even without the structural markers of bait (tribal setup, strawmanning)?

To verify Claude's pattern claims, I'll manually read every post in each proposed error cluster and confirm whether the pattern holds before including it in my evaluation writeup.