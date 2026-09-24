---
title: "Jev Doesn't Write. It Decides."
authors: [benfrank241]
slug: "2026/09/24/jev-typed-decisions-reranking"
date: 2026-09-24T16:00
tags: [hindsight, agent-memory, recall, reranking, jev, typesafe, retrieval, deep-dive]
description: "Jev is a model that produces no text at all, only typed decisions with calibrated probabilities. Hindsight 0.10.1 uses it to rerank memory, in one request instead of three hundred."
image: /img/blog/jev-reranking.png
hide_table_of_contents: true
---

![Jev: typed decisions instead of text, used to rerank agent memory in a single request](/img/blog/jev-reranking.png)

A reranker does not need a model that can write. It needs a decision: out of these three hundred memories, which ones actually answer the question?

Yet the tools available have always been one of two bad fits. A small cross-encoder is fast and cheap but scores each candidate in isolation, pinning it to an absolute scale it re-derives every call. A large language model judges better but writes prose you have to parse, at a latency and price that makes three hundred calls absurd.

[Jev](https://typesafe.ai) is neither. It is a model that produces no text at all.

Hindsight 0.10.1 adds it as a reranker provider, and the interesting part is not that we adopted a faster model. It is that a model built to return typed decisions let us ask a question nobody could afford to ask before.

<!-- truncate -->

## TL;DR

- **Jev is a "System One" model**: it returns typed decisions with calibrated probabilities instead of text, in roughly 70 to 500 milliseconds.
- **We use it differently than the obvious way.** The natural mapping is one yes/no question per candidate, which is still pairwise. We ask a single question with every candidate as an option.
- **One request, however many candidates.** Against the local default at 30 candidates: recall@1 0.800 to 0.950, and 0.12 to 0.027 seconds per query.
- **The design that failed is the interesting one**, and it comes down to picking the right question primitive.
- **It is a third-party hosted API**, so it is opt-in. The local reranker is still the default and still free.

## What a System One model is

The name is a reference to Kahneman: System One is the fast, automatic, pattern-matching kind of thinking, as opposed to the slow deliberate kind. TypeSafe's argument is that most decisions software needs from a model are System One decisions, and that we have been paying for System Two to make them.

The concrete differences from an LLM:

**It returns no text.** You send a passage and a typed question; you get back a typed answer. Not JSON-formatted text that you hope parses, but a value constrained by a schema. There is no string to validate, because there is no string.

**It is trained for calibration rather than preference.** TypeSafe calls this Reinforcement Learning for Calibrated Decisions, as against the RLHF that shapes conversational models. The claim is epistemic honesty: when it says 0.8, it is right about that kind of answer roughly 80% of the time. Worth reading carefully, because calibration holds **in aggregate**, not on any individual answer, which is a meaningfully weaker guarantee than it first sounds.

**It is fast and very cheap.** TypeSafe quotes end-to-end response times of 70 to 500 milliseconds, and input at $0.042 per million tokens with output free. That last detail is not a rounding error in the pricing page, it is a consequence of the design: there is no output to charge for.

Those are TypeSafe's figures, not ours. Ours are further down, and they are about recall quality rather than the model in isolation.

## Why that shape fits reranking

Reranking is the purest possible System One task. There is no reasoning to show, no prose to generate, no tool to call. There is a pile of candidates and a question about which ones matter, and the only thing you want back is an ordering.

Everything an LLM gives you beyond that ordering is waste you are paying for and then throwing away. Everything a cross-encoder gives you is an uncalibrated number you have to normalise yourself before it means anything.

So the fit is obvious. What was not obvious was how to ask.

## Three primitives, and the one everybody reaches for

Jev exposes a small vocabulary of question types, and the whole design of our integration is a question of which one to use.

- **Choice** picks among unordered options and returns a probability for every one of them, summing to one. Up to 255 options.
- **Score** rates against ordered levels, up to ten of them, returning a probability-weighted value.
- **Noul** returns the probability that a yes/no proposition holds.

The obvious mapping for reranking is Noul: ask "is this candidate relevant to this query?" for each pair, then sort by the returned probability. It is clean, it is what the primitive appears designed for, and at Jev's prices it is affordable in a way that per-candidate LLM scoring never was. It is also, as far as we can tell, the pattern TypeSafe's own reranking material suggests.

We did not use it. Asking one question per pair is still pairwise — three hundred round trips for three hundred candidates, and three hundred independent judgements that never see each other. Jev makes that cheap, but cheap pairwise is still pairwise.

## One Choice, every candidate an option

Instead we hand the entire pool to a single Choice. Each candidate becomes an option; the probabilities that come back are the ranking. One request, however many candidates.

The reasoning, from the design note in the code:

> A Choice answers with a probability for every option, summing to 1, so handing it the whole pool returns the ranking in a single call. That beats scoring each candidate on its own: judged together the model only has to say which candidate beats which, instead of pinning every candidate to an absolute scale it must re-derive each time.

Relative judgement is an easier task than absolute judgement. The model never has to decide what 0.6 means in the abstract. It only has to decide what beats what, with all the evidence in front of it at once.

One consequence to internalise: **the scores that come back are positions, not confidences.** The top candidate scores 1.0 and each next one is 1/n lower. A 0.7 means "the best of these," not "relevant," and scores from two different pools are not comparable. If you were planning to set an absolute score floor for this provider, that floor filters on rank position, not on relevance.

## The numbers

Measured on LoCoMo, gold defined as the dataset's own evidence turns, against the reranker Hindsight ships by default.

**30 candidates per query:**

| | recall@1 | recall@5 | NDCG@10 | s/query |
|---|---|---|---|---|
| `local` MiniLM (current default) | 0.800 | 0.876 | 0.850 | 0.12 |
| **Jev, ranking only (the default)** | **0.950** | **0.966** | **0.957** | **0.027** |

**240 candidates per query, the pool size a production recall actually reranks, 60 questions:**

| | recall@1 | recall@5 | NDCG@10 | s/query |
|---|---|---|---|---|
| `local` MiniLM | 0.583 | 0.719 | 0.682 | 0.41 |
| **Jev, ranking only** | **0.783** | **0.903** | **0.856** | **0.063** |

Fifteen points of recall@1 at 30 candidates, twenty at 240, in a fraction of the wall time. The gap widens with the pool, because the pairwise approach is making 240 sequential judgements where the listwise one makes a single comparison.

A separate measurement isolates the question shape from the model. On a 200-question LoCoMo set, the listwise shape scored recall@1 0.94 against 0.87 for one call per candidate **using the same model both times**, at a thirtieth of the calls. So this is not only "Jev is better than MiniLM." How you ask it is doing real work.

## The design that did not work

Here is the part I find most instructive, because it is a negative result with numbers, and because it is entirely about choosing the right primitive.

Ranking is only half of what you want. The other half is knowing where relevance stops. And if you have a Choice, the obvious way to express that is to add a "none of these" option. If the model picks it, nothing is relevant.

It fails badly. Choice options are unrivalled alternatives, not points on a scale, so "none of these" does not compete with the candidates on relevance. It simply wins outright whenever the query is hard. **Thirty-five of two hundred questions came back completely empty.**

So the cut is a different primitive: a Score, whose levels are *ordered*, which is exactly what a cut point needs. It is shown the ranked shortlist and asked how far down genuine relevance extends. The levels are plain language — only the first, the first two, the first three, the first five, the first ten, all of them — and the model picks one. There is no threshold to tune, by design.

There is deliberately no "nothing is relevant" level either, for the same reason in miniature. Recall runs on a pool retrieval already judged plausible, and one weak memory the caller can dismiss beats silence. Adding that level cost 7% of queries returning nothing and dropped gold retention from 0.81 to 0.65.

Both of those numbers live in source comments rather than a benchmark suite. They are the record of two designs being tried and rejected, which is usually the part nobody writes down.

## The cut, and why it is off by default

Turn pruning on and the second question runs, dropping everything past the chosen depth instead of carrying it forward.

The precision numbers are dramatic. On the 30-candidate run it keeps 1.6 candidates out of 30 and lifts the precision of what survives from 0.051 to 0.850, a factor of seventeen. **It also cuts 19% of the gold evidence**, and on a real bank it took 300 candidates down to 3.

That is a very different answer to hand an agent, which is why the flag stays off. Precision that high is what you want when the consumer is an LLM prompt and every irrelevant memory is wasted context. It is not what you want when a person is going to read the list.

One limit documented nowhere else: **the cut only ever sees the top twelve candidates, so with pruning on, recall returns at most twelve results regardless of pool size.** The shortlist is capped at twelve and the chosen depth is clamped to it. Consistent with "300 down to 3," but if you expected a large pool to yield a large relevant set, it will not. The counterweight is that at least one candidate always survives, so no query comes back empty.

## The limits worth knowing

**Jev's Choice ceiling is 255 options**, and we stop at 250 to stay clear of the edge. Larger pools are ranked in rounds: chunk the candidates, rank each chunk, take the top twelve of each as finalists, rank the finalists. You cannot simply concatenate rounds, because probabilities are normalised within a call. Candidates that win no round keep their round order behind the finalists, so at the default cap of 300, ordering deep in the tail is within-chunk rather than global.

**Jev's context is 32,000 tokens for state plus questions.** Our provider does not truncate candidate text before sending it, so a pool of unusually long memories is something to size up rather than assume.

**It is a third-party hosted API.** Jev runs at TypeSafe's endpoint and needs an API key. If your reason for self-hosting Hindsight is that memory contents must not leave your infrastructure, this sends candidate text to someone else's, and that is disqualifying no matter how good the benchmark.

**It fails closed.** A configured reranker that keeps erroring propagates the failure after its retry budget. Configure a fallback chain with `rrf` last so a bad day degrades to fusion order rather than failing the recall. That matters more with a network dependency in the path.

**It is server-level**, so a bank cannot pick its own reranker, only turn reranking off. And **ranking is not deduplication**: two near-identical memories both occupy slots, and with pruning on they eat two of your twelve. That work happens in consolidation and `prefer_observations`, which we [wrote about yesterday](https://hindsight.vectorize.io/blog/2026/09/23/bring-facts-not-beliefs).

## If you cannot use it

The default has not changed. `local` runs a MiniLM cross-encoder in-process, costs nothing, sends nothing anywhere, and is the 0.800 row above. That is a perfectly respectable reranker.

Beyond it: `tei` for a self-hosted Hugging Face inference endpoint, with Helm templates in the repo; `flashrank` for something small and local; `jina-mlx` on Apple Silicon, though its licence is non-commercial; and `rrf`, which skips neural reranking and keeps the fusion order.

None of them prune, because none of them produce a decision. They produce an ordering, and the lowest score in an ordering still means "least bad of these" rather than "not relevant." That distinction is the whole reason the cut exists here and nowhere else.

## FAQ

**Do I have to change anything?**
No. Jev is opt-in via the reranker provider setting. The default is unchanged.

**Is Jev a Vectorize model?**
No. TypeSafe is a separate company and Jev is their model. Hindsight supports it the way it supports Cohere, TEI and the rest: as one provider among several.

**Does it cost more than the local reranker?**
The local one is free, so yes in absolute terms. But the listwise shape uses roughly a thirtieth of the calls of per-candidate scoring, and Jev charges for input only. Check TypeSafe's pricing against your recall volume.

**Should I turn on pruning?**
Only when the consumer is an LLM prompt and every irrelevant memory is wasted context, and only after trying it on your own data. Remember the two costs: 19% of gold evidence, and a ceiling of twelve results.

**What happens if TypeSafe is down?**
Without a fallback chain, the recall fails after its retries. Configure indexed fallback members with `rrf` last and it degrades to fusion order instead.

## Learn more

- [Cross-Encoder Reranking: The Last Stage of Agent Memory Recall](https://hindsight.vectorize.io/blog/2026/08/28/cross-encoder-reranking-agent-memory) on the stage itself, normalization and failing open
- [Knowledge Graphs vs Vector Search](https://hindsight.vectorize.io/blog/2026/08/24/knowledge-graphs-vs-vector-search) on the retrieval arms that feed it
- [Bring the Facts, Not the Beliefs](https://hindsight.vectorize.io/blog/2026/09/23/bring-facts-not-beliefs) on the deduplication reranking does not do
- [What's new in Hindsight 0.10.1](https://hindsight.vectorize.io/blog/2026/09/21/version-0-10-1) for the rest of the release
