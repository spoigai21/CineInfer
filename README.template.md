# CineInfer

**A movie recommender: tell it who you are, and it suggests 10 movies you're likely to love.**

It learned from {{data.ratings}} real movie ratings (the public MovieLens dataset: {{data.users}}
people rating {{data.movies}} movies) and picks its suggestions from all {{data.movies}} movies.

> Demo video: *coming soon.*

This README is written for anyone, no machine-learning background needed. The technical details
are in the [appendix](#appendix-technical-details) at the end. Every number in this README is
filled in automatically from the project's result files, so none of them were typed by hand.

---

## How well does it work?

To test it fairly, each person's **most recent** ratings were hidden from the system while it was
built. At the end, it made 10 suggestions for each of {{test.users}} people, and we checked how
many matched movies they **actually went on to love** (rated 4 or 5 stars). This final test was
run **once**, so it couldn't be tuned to look good.

{{table:plain_results}}

**In plain words:**
- Out of every 100 people, about **{{hits.two_stage.per100}}** found at least one movie they'd go
  on to love in their top 10.
- That's **{{hits.ratio.two_stage.most_popular}}× more good suggestions** than just recommending
  popular movies, and **{{hits.ratio.two_stage.ease}}× more** than the best traditional method.
- "About 1 good movie in 10" may sound low, but the test is strict: the typical person loved only
  about **{{hits.median_relevant}}** movies in their hidden period, out of {{data.movies}}. Guessing
  at random would find almost none. And a "miss" isn't necessarily a bad suggestion: it only means
  they didn't happen to rate that movie in the test period.

---

## How it works, in plain words

**1. Learn from the past, test on the future.** Each person's ratings are sorted by date. The
older ones are used for learning, and the newest ones are hidden and only used for the final
test, the way a teacher keeps exam questions secret. Automated checks make sure no "future"
ratings ever leak into the learning part.

**2. Start with simple methods as a benchmark.** Before anything complex, several traditional
methods were built, from "recommend whatever is popular" to "people who liked X also liked Y".
The complex system only counts if it beats the best of these.

**3. A neural network quickly finds 200 candidates.** It places every movie on a kind of "map"
where movies liked by the same people sit close together. It then places you on that map based on
your 10 most recently liked movies, and grabs the 200 movies closest to you, out of
{{data.movies}}, in about a millisecond.

**4. A second model carefully picks the final 10.** It looks more closely at those 200, using extra
information: how popular and well-rated each movie is, whether its genres match your taste, and
a second opinion from the best traditional method. It re-orders them and keeps the top 10. This
"quick search, then careful choice" design is common in large recommendation apps.

**5. A fast web service.** You (or an app) ask for a person's suggestions and get them back in a
few thousandths of a second. New users with no history get a list of popular movies instead.

---

## What we learned

- **The two-step design pays off.** The careful second step improved results by
  {{gap.two_stage.two_tower.ndcg.rel}} over the neural network alone.
- **Much of the success comes from people rating in bursts.** MovieLens records when someone
  *rated* a movie, not when they watched it, and people often rate many movies in one sitting:
  {{data.burst_1h}} of people rated all of their hidden movies within one hour. So the system is
  often predicting "what will you rate next tonight". It still does better than the traditional
  methods for people who come back later, but by less.
- **How you split the data changes the score a lot.** Testing with randomly shuffled ratings
  instead of by date made the same method score **{{split.ratio}}×** higher, an overly rosy
  picture. That's why everything here is tested on each person's most recent ratings.
- **Simpler tools were faster for the data preparation.** Preparing all {{data.ratings}} ratings
  took **{{bench.25M.duckdb.with}} seconds** with DuckDB, {{bench.25M.spark.with}} seconds with Spark
  (a tool built for many computers), and {{bench.25M.pandas.with}} seconds with pandas.

---

## Predictions made before building the neural network

Before training any neural network, I wrote down {{predictions.total_word}} predictions and saved
them publicly on GitHub with a timestamp, so they couldn't be quietly changed afterwards.
{{predictions.refuted_Word}} turned out wrong, and they're all reported:

| # | Prediction (in plain words) | Result |
|---|---|---|
| 1 | The neural network beats "recommend popular movies" by at least 50% | {{pred.1.verdict}} |
| 2 | A well-tuned traditional method nearly matches the neural network | {{pred.2.verdict}}: the neural network was clearly better |
| 3 | The careful second step adds only a little | {{pred.3.verdict}}: it added {{gap.two_stage.two_tower.ndcg.rel}} |
| 4 | "Recommend popular movies" covers the fewest different movies | {{pred.4.verdict}} |
| 5 | pandas is faster than Spark at every data size | {{pred.5.verdict}}: Spark was faster at the largest size |
| 6 | Suggestions come back in under 25 ms, even in the slowest 1% of cases | {{pred.6.verdict}}: the slowest 1% took {{latency.p6.p99}} ms |
| 7 | Testing on shuffled data inflates scores by at least 50%, and three test setups score in a predicted order | {{pred.7.verdict}}: the inflation was real ({{split.ratio}}×), but the order was wrong |

The exact wording and evidence for each are in the [appendix](#predictions-full-detail).

---

## Try it yourself

You'll need a Mac or Linux computer, Python 3.11 and Java 17 (on a Mac, also run
`brew install libomp`), plus about 15 GB of free disk space. The movie data is downloaded
automatically; it isn't stored in this repository, because its license doesn't allow sharing it.

```bash
make install      # set up everything
make test         # run the automated checks (no download needed)
make data         # download the movie ratings (and check they're not corrupted)
make prep         # prepare the data (~2 minutes)
```

The models themselves aren't stored in the repository either, so they have to be trained before
the recommender can run. That takes several hours on a laptop; the full list of commands is in the
[appendix](#running-everything). Once they're built, `make export serve` starts the recommender,
and the demo page at `http://localhost:8000` lets you pick any user (or a random or brand-new one)
and see the movies they recently liked next to the 10 suggestions, with how long each step took.

---

## Limitations

- **Tested on past data only.** The system was never shown to real users, so we don't know how
  people would actually respond to its suggestions.
- **Rating isn't watching.** People rate movies in bursts, which makes some predictions easier
  than they would be in a real app (see "What we learned").
- **Every MovieLens user has at least 20 ratings,** so the "new user" path is built and tested but
  never met a genuinely new person. {{data.movies_cold}} movies were rated only in the hidden
  periods, so the system never saw them and could never suggest them.
- **Some information about other people's future ratings** is available to the system during
  learning (for example, how popular a movie *eventually* became). A follow-up experiment showed
  that removing this doesn't hurt (see the appendix).
- **It all ran on one laptop,** and the results are specific to movies and MovieLens.

---

## Glossary

| Term | Meaning |
|---|---|
| **Recommender** | a system that suggests items (movies, songs, products) a person might like |
| **Neural network** | a program with millions of adjustable numbers that it tunes by learning from examples |
| **Traditional methods** (EASE, ALS, item-kNN) | older, well-established recommendation techniques, mostly formula-based |
| **Ranker** | the second step that re-orders candidates to put the best ones on top (here, LightGBM, a model made of many small decision trees) |
| **Training / validation / test** | learning data / practice exam used to make decisions / final exam taken once |
| **NDCG@10** | a 0-to-1 score for a top-10 list: higher when movies the person loved appear, and higher still when they appear near the top |
| **ms (millisecond)** | one thousandth of a second |

---

## Appendix: technical details

### Results on the test set (scored once)

{{test.users}} users, each ranking against the full catalog with everything they'd already rated
masked. A movie counts as relevant if the user rated it ≥ 4 in their held-out test period.

{{table:headline}}

- **The two-stage system beats every baseline**: {{gap.two_stage.ease.ndcg.rel}} NDCG@10 over
  EASE ({{gap.two_stage.ease.ndcg.diff}}, 95% CI {{gap.two_stage.ease.ndcg.ci}}, paired bootstrap
  over users), and {{test.ratio.two_stage.most_popular}}× most-popular.
- **The ranker adds {{gap.two_stage.two_tower.ndcg.rel}}** over retrieval alone
  ({{gap.two_stage.two_tower.ndcg.diff}}, CI {{gap.two_stage.two_tower.ndcg.ci}}). It can only
  re-order what retrieval found: {{test.recall_ceiling_200}} of relevant test movies are in the
  200 candidates.
- **The two-tower model beats EASE by {{gap.two_tower.ease.ndcg.rel}}**
  ({{gap.two_tower.ease.ndcg.diff}}, CI {{gap.two_tower.ease.ndcg.ci}}) and recommends
  {{test.coverage_ratio.two_tower.ease}}× as much of the catalog ({{test.two_tower.coverage}} vs
  {{test.ease.coverage}} of the {{test.coverage_denominator}} movies with training data).
- Paired bootstrap on NDCG@10, {{gaps.chain}} (`results/test_gaps.csv`).

### Where the gains come from (session boundary)

{{data.burst_1h}} of users rated their whole test slice within one hour (median span
{{data.burst_median_min}} minutes). Test results by the gap between a user's last training rating
and their first test rating:

{{table:boundary}}

The two-tower model uses only a user's last 10 liked movies, so it's strongest within a session
({{boundary.two_tower.same_second}} vs EASE's {{boundary.ease.same_second}}) and **worse than EASE
when the test period starts more than an hour later** ({{boundary.two_tower.over_1h}} vs
{{boundary.ease.over_1h}}). Feeding EASE only the last 10 movies (the control row) doesn't
reproduce that. The ranker sees both scores, so the two-stage system beats EASE in every bucket,
including over 1 h ({{boundary.two_stage.over_1h}}).

### Ranker ablations

{{table:ablation}}

- **EASE's score is the ranker's most valuable feature.** Without it the ranker keeps only
  {{ablation.no_ease_share}} of its gain over retrieval.
- **Two time features are excluded** even though they help: they compare a user's last rating with
  the last time *anyone* rated a movie, which under a per-user split includes other users' ratings
  from after this user's cutoff.

### Removing the cross-user time leak (validation, after the fact)

A follow-up rebuilt the ranker's item features **point in time**: only ratings made strictly before
the user's last training rating, by anyone (`make pit-experiment`). The recomputed headline
reproduces Phase 6 exactly for every seed:

| Ranker features (validation, 3 seeds) | NDCG@10 | vs headline (95% CI) |
|---|---|---|
| Headline (full-train item stats) | {{pit.headline.ndcg}} | — |
| Point-in-time item stats | {{pit.pit.ndcg}} | {{pit.pit.diff}} {{pit.pit.ci}} |
| Point-in-time item stats + point-in-time recency | {{pit.pit_time.ndcg}} | {{pit.pit_time.diff}} {{pit.pit_time.ci}} |

The leak wasn't propping the headline up: point-in-time popularity is *more* informative. This was
designed after the test results were known, so it's reported on validation only.

### Predictions, full detail

{{predictions.confirmed}} of {{predictions.total}} confirmed, {{predictions.refuted}} refuted. Each
had a "refuted if" condition, was committed, tagged
([`predictions`](https://github.com/spoigai21/cine-infer/tree/predictions)) and pushed before any
neural model existed, and is settled by code from the result files.

{{table:predictions}}

- **#2, #3:** a well-tuned EASE didn't match the neural model, and the ranker added far more than
  expected; much of the two-tower's gain comes from predicting the rest of a rating session.
- **#5:** local-mode Spark's overhead dominates at 1M and 5M rows, but single-threaded pandas scales
  worse, and Spark wins at 25M. DuckDB and Polars beat both at every size.
- **#6:** p50 was fast, but p99 missed by {{latency.p6.p99_miss}} ms.
- **#7:** the random split inflates NDCG@10 more than predicted, but the global cutoff scored
  *above* the per-user split: its {{split.global.users}} test users are heavy raters still active
  after {{split.global_cutoff_date}}, a different population.

### How the split choice changes the numbers

{{table:splits}}

A random split reports {{split.ratio}}× (95% CI {{split.ratio_ci}}) the global cutoff's NDCG@10.
The per-user time split keeps {{split.user.users}} users evaluable (the global cutoff keeps
{{split.global.users}}) with no user's own future in their training data.

### Serving

`GET /recommend/{user_id}?k=10` retrieves 200 candidates by brute-force dot product over all
{{data.movies}} movies, builds 15 features, ranks with LightGBM and returns titles, scores and
per-stage timings. The server is NumPy + LightGBM only (no PyTorch) and loads in
{{latency.load_s}} s.

- **No training/serving skew:** on {{parity.users}} test users, the server's top-10 matches the
  batch pipeline's for {{parity.top10}}.
- **Latency ({{latency.p6.requests}} sequential HTTP requests, laptop CPU):** the run that settled
  prediction #6 measured p50 {{latency.p6.p50}} ms and p99 {{latency.p6.p99}} ms. Afterwards,
  single-threaded serving gave p50 {{latency.threads1.p50}} ms and p99
  {{latency.threads1.p99_range}} ms (default threading: p99 {{latency.default.p99_range}} ms). The
  popularity fallback takes {{latency.fallback.p50}} ms.

### Retraining with Airflow

An Airflow DAG runs prep → train → evaluate → **publish only if better**: validation NDCG@10 must
beat the live model by more than {{airflow.margin}} (just above measured run-to-run noise).
Publishing refits on train + val, rebuilds the ranker, exports and smoke-tests a bundle, then swaps
it in atomically. A deliberately worse {{airflow.epochs}}-epoch model ({{airflow.candidate}} vs
{{airflow.live}}) was rejected; in a sandbox, a better one ({{airflow.pub.candidate}} vs
{{airflow.pub.live}}) was published end to end. MovieLens is a fixed dataset, so this pipeline
retrains on the same data; a live deployment would add a step that pulls new ratings.

### Data-tool benchmark

The data-preparation pipeline in four tools with identical outputs ({{bench.runs}} runs, AC power,
load ≤ {{bench.max_load}}). Each cell is *with startup / without startup*, median of 3:

{{table:benchmark}}

![benchmark](results/benchmark.png)

### Tuning

All choices were made on {{data.val_users}} validation users; the test set was never used for any
decision.

{{table:tuning}}

### Running everything

```bash
make install        # .venv with everything
make data           # download + verify MovieLens 25M (not redistributable, never committed)
make prep           # Phase 1: splits + features (~2 min)
make baselines      # Phase 3: tune the baselines on validation (~1.5 h, mostly ALS)
make two-tower      # Phase 5: tune the retriever (~3 h on an Apple GPU)
make ranker         # Phase 6: two-stage system on validation (~45 min)
make final-test     # Phase 6b: refit everything, score test once (~1.5 h)
make export serve   # Phase 7: build the serving bundle, run the API + demo page on :8000
make airflow-install airflow-reject-demo   # Phase 8: a worse model is refused
make airflow-publish-demo   # Phase 8: the publish path, end to end, in a sandbox registry
make airflow-ui     # Airflow UI on :8080 (user admin; password in airflow_home/standalone_admin_password.txt)
make benchmark      # Phase 9 (plug in, idle machine)
make split-comparison test-analysis readme  # Phase 10
make pit-experiment # follow-up: point-in-time ranker features (validation only)
make test           # the test suite, on a synthetic fixture (also in CI)
```

Times are rough guides for one laptop. Run one pipeline at a time.

### Repository map

`src/` pipeline code · `tests/` pytest on a synthetic fixture · `results/` every number in this
README, as CSV · `dags/` Airflow · `cineinfer.md` the plan and predictions ·
`cineinfer-implementation.md` the build log, phase by phase. Data:
[MovieLens 25M](https://grouplens.org/datasets/movielens/25m/) (GroupLens; downloaded by
`make data`, not redistributed).
