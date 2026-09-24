---
title: "One Question, Not Three Hundred"
authors: [benfrank241]
slug: "2026/09/24/listwise-reranking-typesafe-jev"
date: 2026-09-24T16:00
tags: [hindsight, agent-memory, recall, reranking, retrieval, typesafe, benchmarks, deep-dive]
description: "Rerankers score one candidate at a time, and that single property dictates their whole design. Hindsight 0.10.1 asks one question with every candidate as an option instead. The numbers, the design that failed, and the limits."
image: /img/blog/listwise-reranking.png
hide_table_of_contents: true
---

![Listwise reranking: every candidate becomes an option in a single question, and the answer is the ranking](/img/blog/listwise-reranking.png)

A reranker is the last thing standing between retrieval and your agent's context window. Retrieval hands it a few hundred plausible memories; it decides which ones the model actually sees.

Every reranker we shipped before this one worked the same way: score one candidate against the query, then the next, then the next. As we [wrote in August](https://hindsight.vectorize.io/blog/2026/08/28/cross-encoder-reranking-agent-memory), you pay one model forward pass per candidate, and that single property dictates the entire design of the stage. It is why reranking runs last. It is why there is a candidate cap at all.

Hindsight 0.10.1 adds a provider that does not work that way. It asks one question, with every candidate as an option, and the answer is the ranking.

<!-- truncate -->

## TL;DR

- **One request, however many candidates.** The whole pool goes out as options in a single question rather than one call each.
- **It is more accurate and much faster.** Against the local default at 30 candidates: recall@1 0.800 to 0.950, and 0.12 to 0.027 seconds per query.
- **The gap widens with the pool.** At 240 candidates, recall@1 0.583 to 0.783.
- **Cutting the irrelevant tail is a second, optional question**, and it is off by default for a good reason.
- **It is a third-party hosted API**, so it is not for everyone. The local reranker is still the default and still free.

## Why scoring one at a time is the problem

A cross-encoder reads the query and one document together and returns a relevance score. Do that for every candidate and you get a ranking. It works, and it is the standard approach.

The cost is linear in candidates, which forces two compromises. Reranking has to run last, on a set already narrowed by cheaper retrieval, because you cannot afford it earlier. And the set has to be capped: `HINDSIGHT_API_RERANKER_MAX_CANDIDATES` defaults to 300, and everything past that is discarded by fusion rank before the reranker ever sees it.

There is a subtler problem. Scoring candidates independently asks the model to pin each one to an absolute scale, and it has to re-derive that scale on every call. Nothing in the call tells it whether a 0.6 here means the same thing as a 0.6 there. Calibration becomes the caller's problem, which is why the reranking stage carries normalisation logic at all.

## One question, every candidate an option

The `typesafe` provider replaces the scoring loop with a single typed question. Set `HINDSIGHT_API_RERANKER_PROVIDER=typesafe` and recall's reranking runs on TypeSafe's Jev model, `jev-latest` by default.

The question is a **Choice**: a question type that returns a probability for every option, summing to one. Hand it the whole candidate pool as options and the answer is the ranking, in one request. The design note in the code explains why that is better than scoring each candidate alone:

> A Choice answers with a probability for every option, summing to 1, so handing it the whole pool returns the ranking in a single call. That beats scoring each candidate on its own: judged together the model only has to say which candidate beats which, instead of pinning every candidate to an absolute scale it must re-derive each time.

Relative judgement is simply an easier task than absolute judgement. The model never has to decide what 0.6 means. It only has to decide what beats what.

One consequence worth internalising: **the scores that come back are positions, not confidences.** The top candidate scores 1.0 and each subsequent one is 1/n lower. A 0.7 means "the best of these," not "relevant," and scores from two different pools are not comparable. That matters if you were planning to set an absolute score floor, because for this provider that floor is filtering on rank position rather than on relevance.

## The numbers

Measured on LoCoMo, with gold defined as the dataset's own evidence turns, against the current default reranker on the same tasks.

**30 candidates per query:**

| | recall@1 | recall@5 | NDCG@10 | s/query |
|---|---|---|---|---|
| `local` MiniLM (current default) | 0.800 | 0.876 | 0.850 | 0.12 |
| **typesafe, ranking only (the default)** | **0.950** | **0.966** | **0.957** | **0.027** |

**240 candidates per query, the pool size a production recall actually reranks, 60 questions:**

| | recall@1 | recall@5 | NDCG@10 | s/query |
|---|---|---|---|---|
| `local` MiniLM | 0.583 | 0.719 | 0.682 | 0.41 |
| **typesafe, ranking only** | **0.783** | **0.903** | **0.856** | **0.063** |

Fifteen points of recall@1 at 30 candidates, twenty at 240, in a fraction of the wall time. The gap widens with the pool because the pairwise approach is doing 240 sequential judgements where the listwise one is doing a single comparison.

Worth separating from those figures: a second measurement compared the two *question shapes* on the same model, rather than comparing models. On a 200-question LoCoMo set, the listwise shape scored recall@1 0.94 against 0.87 for one call per candidate, at a thirtieth of the calls. So the gain is not only "a better model." The shape of the question is doing real work.

## Above 250 candidates

A Choice has an option ceiling, and the implementation stays clear of it at 250. Larger pools are ranked in rounds: chunk the candidates, rank each chunk concurrently, take the top twelve of each as finalists, and rank the finalists against each other in one more call.

The reason you cannot simply concatenate the rounds is that each round's probabilities are normalised within its own call. A 0.4 in round one and a 0.4 in round two are shares of different pools.

Candidates that win no round keep their round order behind the finalists. The code is candid about what that means: they are the ones the cut would discard anyway. At the default cap of 300 you are getting two rounds plus a final, and positions past the top two dozen are ordered within their chunk rather than globally. If precise ordering deep in the tail matters to you, that is a real limitation.

## The design that did not work

This is the part I find most interesting, because it is a negative result with numbers attached.

If you have a Choice and you want to cut irrelevant candidates, the obvious move is to add a "none of these" option. If the model picks it, nothing is relevant. It is the first thing anyone would try.

It fails badly. Choice options are unrivalled alternatives rather than points on a scale, so "none of these" does not compete with the candidates on relevance, it simply wins outright whenever the query is hard. **Thirty-five of two hundred questions came back completely empty.**

So the cut is a different question type: a **Score**, whose levels are ordered, which is what a cut point actually needs. It is shown the ranked shortlist and asked how far down the list genuine relevance extends. The levels are plain language — only the first, the first two, the first three, the first five, the first ten, all of them — and the model picks one. There is no threshold to tune, by design.

There is deliberately no "nothing is relevant" level, and the reasoning is the same failure in a smaller form. Recall runs on a pool that retrieval already judged plausible, and one weak memory the caller can dismiss beats silence. Adding that level cost 7% of queries returning nothing and dropped gold retention from 0.81 to 0.65.

Both of those numbers are in the source, not in a benchmark suite. They are the record of two designs being tried and rejected, which is usually the part that never gets written down.

## Why the cut is off by default

Set `HINDSIGHT_API_RERANKER_TYPESAFE_PRUNE_CANDIDATES=true` and the second question runs. Everything past the chosen depth is dropped rather than carried forward.

It works, and the precision numbers are dramatic. On the 30-candidate run it keeps 1.6 candidates out of 30 and lifts the precision of what survives from 0.051 to 0.850, a factor of seventeen. **It also cuts 19% of the gold evidence**, and on a real bank it took 300 candidates down to 3.

That is a very different answer to hand an agent, and it is why the flag stays off. Precision that high is exactly what you want when the consumer is an LLM prompt and every irrelevant memory is wasted context. It is not what you want when a human is going to read the list and decide.

One limit that is not documented anywhere else, and which you should know before turning it on: **the cut only ever sees the top twelve candidates, so with pruning enabled recall returns at most twelve results regardless of pool size.** The shortlist is capped at twelve and the chosen depth is clamped to it. That is consistent with "300 candidates down to 3," but if you were expecting a large pool to yield a large relevant set, it will not.

The counterweight is that at least one candidate always survives. No query comes back empty.

## The honest limits

Four things to weigh before switching a production bank over.

**It is a third-party hosted API.** Jev runs at `api.typesafe.ai` and needs `HINDSIGHT_API_RERANKER_TYPESAFE_API_KEY`. The base URL is configurable, but Hindsight does not ship the model, a container or a chart for it. If your reason for self-hosting Hindsight is that memory contents must not leave your infrastructure, this provider sends candidate text to someone else's, and that is disqualifying regardless of the benchmark.

**It fails closed.** A single configured reranker that keeps erroring propagates the failure after its retry budget. To degrade gracefully you configure a fallback chain with the indexed `HINDSIGHT_API_RERANKER_<n>_*` settings and put `rrf` last, so a bad day falls back to fusion order instead of failing the recall. This is the same advice as in the August post and it matters more with a network dependency in the path.

**It is server-level.** A bank cannot pick its own reranker. `enable_reranking` lets a bank turn reranking off, but provider selection is a server setting, so a multi-tenant deployment runs one reranker for everyone.

**It does nothing about duplicates.** Ranking is not deduplication. Two near-identical memories both occupy ranked slots, and with pruning on they consume two of your twelve. Deduplication happens elsewhere, in consolidation and in `prefer_observations`, and [we wrote about that yesterday](https://hindsight.vectorize.io/blog/2026/09/23/bring-facts-not-beliefs).

## If you cannot use it

The default has not changed. `local` runs `cross-encoder/ms-marco-MiniLM-L-6-v2` in-process, costs nothing, sends nothing anywhere, and is the 0.800 row in the table above. That is a perfectly respectable reranker.

Beyond it: `tei` for a self-hosted Hugging Face inference endpoint, with Helm templates in the repo; `flashrank` for something small and local; `jina-mlx` on Apple Silicon, though note its licence is non-commercial; and `rrf`, which skips neural reranking entirely and keeps the fusion order.

None of these prune. The cut is specific to this provider, because it is the only one whose output is a real decision rather than an ordering.

## Where it sits in recall

For context, the stage this replaces is one step in a longer pipeline. Retrieval runs its arms in parallel, reciprocal rank fusion merges them, the merged pool is capped at 300 by fusion rank, and only then are payloads fetched for the survivors. That ordering is deliberate: the wide arms move ids and scores, and the expensive hydration happens only for what got that far.

The reranker reads the hydrated text, which is why it comes after. Downstream, the combined scoring stage multiplies the reranker's output by recency, temporal proximity and evidence strength, then the token budget cuts by rank.

That last detail is the argument for the cut. The token budget always cuts by rank, which means when nothing is relevant it keeps whatever happens to be left. Deciding that the tail is irrelevant is a better answer than letting a budget decide it by arithmetic.

## FAQ

**Do I have to change anything to get this?**
No. The default reranker is unchanged. This is opt-in via `HINDSIGHT_API_RERANKER_PROVIDER=typesafe`.

**Does it cost more?**
It replaces many small calls with one, and the listwise comparison measured roughly a thirtieth of the calls for the same task. Pricing is TypeSafe's, not ours, so check theirs against your recall volume.

**Should I turn on pruning?**
Turn it on when the consumer is an LLM prompt and every irrelevant memory is wasted context, and only after trying it on your own data. Remember the two costs: 19% of gold evidence, and a hard ceiling of twelve results.

**Can different banks use different rerankers?**
No. Provider selection is server-level. A bank can only disable reranking entirely.

**What happens if TypeSafe is down?**
Without a fallback chain, the recall fails after its retries. Configure indexed fallback members with `rrf` last and it degrades to fusion order instead.

## Learn more

- [Cross-Encoder Reranking: The Last Stage of Agent Memory Recall](https://hindsight.vectorize.io/blog/2026/08/28/cross-encoder-reranking-agent-memory) on the stage itself, normalization and failing open
- [Knowledge Graphs vs Vector Search](https://hindsight.vectorize.io/blog/2026/08/24/knowledge-graphs-vs-vector-search) on what the retrieval arms feeding this actually do
- [Bring the Facts, Not the Beliefs](https://hindsight.vectorize.io/blog/2026/09/23/bring-facts-not-beliefs) on the deduplication that reranking does not do
- [What's new in Hindsight 0.10.1](https://hindsight.vectorize.io/blog/2026/09/21/version-0-10-1) for the rest of the release
