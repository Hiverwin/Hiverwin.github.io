---
layout: blog_post
title: "Why Can LLM Cache Hit Rates Reach 90%?"
date: 2026-07-07
tags:
  - Note
  - LLM Systems
  - Inference
  - Caching
---

I recently read an article asking a very practical question: why do production LLM dashboards so often show cache hit rates around 90%?

At first glance, that number sounds surprisingly high. It almost looks like a vendor-specific trick or a dashboard number that should be treated with suspicion. After reading through the engineering details, the answer felt much less mysterious. A 90% hit rate is not magic. It is what happens when modern inference systems, prefix caching, and agent-style usage patterns line up extremely well.

This note is my attempt to restate the article in a cleaner way and keep the parts that matter most. What made it interesting to me was not only the systems explanation. It also clarified something about AI products: once products become more stateful and agent-like, caching stops being a backend optimization detail and starts becoming part of the product economics.

## 1. What is actually being cached?

The key object is the **KV cache**.

In a transformer, once a token has been processed, the model can store its intermediate **key** and **value** states. Later tokens can attend to those cached states without recomputing the whole prefix again. This matters because the expensive part of inference is usually the long prompt prefix, especially during prefill.

So the cache is not storing final text outputs. It is storing internal attention states for previously processed tokens.

That distinction is important. When people hear "cache," they often imagine something like memoizing the full answer. In LLM serving, the more useful optimization is caching the reusable internal computation. In other words, the system is not remembering what it said. It is remembering expensive work it no longer wants to redo.

## 2. Why prefix caching works so well

The next step is **prefix caching**.

The idea is simple:

- if two requests share the same prefix,
- the model should only compute that prefix once,
- and later requests should directly reuse the cached KV states.

This is especially common in real LLM traffic:

- the same assistant keeps sending the same long system prompt,
- multi-turn conversations repeatedly include all previous history,
- coding agents keep resending tool instructions, policies, and earlier context.

As a result, a large fraction of requests begin with highly repeated token sequences.

That repeated prefix is exactly what prefix caching is designed for. This is where the engineering idea becomes product-relevant: repetition in prompts is not always bad design. Sometimes it is the natural consequence of building a stable assistant with memory, tools, and persistent instructions.

## 3. Why exact matching matters

One of the most counterintuitive points is that prefix caching is extremely strict.

The cache only works when the prefix matches exactly, token by token. Once the first mismatch appears, reuse stops from that point onward.

The article points out that this comes from positional encoding. A token's key and value are determined not only by *what* the token is, but also by *where* it appears in the sequence. If the earlier prefix changes, later hidden states are no longer the same object.

So the rule is:

- identical prefix before the divergence point -> still reusable,
- anything after the divergence point -> must be recomputed.

That is why stable prompts matter so much in practice. Small edits near the front of the request can destroy a surprisingly large amount of cache reuse. From a product point of view, this is a quiet but important constraint. Teams often think about prompts as flexible text. The inference system treats them much more like infrastructure.

## 4. Why the number gets so high in agent workloads

This is the most interesting part.

The reason hit rates can stay near 90% is not only that the cache exists. It is that **agent-style interaction is almost the perfect workload for prefix caching**.

Agent calls usually have the following shape:

1. a stable system prompt,
2. a stable tool schema,
3. an ever-growing interaction history,
4. only a small amount of new content appended at the end.

That means each new request looks very similar to the previous one. The old prefix is still there, and the only thing that changes is the tail.

Once the conversation becomes long enough, the reusable portion dominates the total request length. In other words, the more the agent keeps appending instead of rewriting, the more the cache has a chance to pay off.

This is why long-running coding agents, assistant sessions, and structured multi-turn workflows often have such stable hit rates.

What I like about this explanation is that it connects a low-level metric with a very recognizable product pattern. A high cache hit rate often means the product has become deeply conversational, stateful, and tool-driven. It is a signature of a certain kind of UX, not just a signature of a certain kind of serving stack.

## 5. Why TTL does not hurt as much as it sounds

Another detail I found useful is the role of TTL.

Many commercial systems keep cached prefixes only for a short time window, often minutes or hours. At first that sounds limiting. But in active agent sessions, requests keep arriving frequently enough that the cache effectively keeps renewing itself.

So in practice:

- short TTL matters if the session goes idle,
- short TTL matters if routing breaks locality,
- short TTL matters if users disappear and come back much later,
- but it matters much less during continuous active use.

That explains why dashboards can still show strong hit rates even when the cache is not meant to be permanent storage. In active agent sessions, "short-lived" infrastructure can still behave like durable memory from the user's point of view.

## 6. Two major implementation styles

The article also compares two well-known serving designs.

### vLLM: block-based automatic prefix caching

vLLM splits the prefix into fixed-size blocks. Each block gets a hash derived from its parent block hash plus its own content. In effect, the whole prefix becomes a chain of hashed blocks.

This design makes exact prefix matching efficient without building an explicit tree structure. Blocks can be independently managed and evicted with LRU.

The main point here is not the hash formula itself. The important idea is that a long prefix is turned into reusable units that are easy to compare and recycle.

### SGLang: RadixAttention

SGLang uses a radix tree to represent shared token prefixes more explicitly. Requests that share a prefix naturally occupy the same path in the tree.

This feels conceptually elegant because prefix sharing becomes a first-class data structure rather than only a block lookup trick.

The two systems are different in implementation, but they are solving the same core problem:

- how to recognize repeated prefixes quickly,
- how to reuse cached computation safely,
- how to evict old cache entries without breaking throughput.

What I find interesting here is that the serving layer is starting to look more like a data structure problem than a pure model problem. Once enough requests share context, the system is effectively managing a live graph of reusable computation.

## 7. Why high hit rate does not mean "the model is cheaper in every sense"

A high cache hit rate is good news, but it should still be interpreted carefully.

What it really means is:

- a large share of incoming requests reuse old prefixes,
- expensive prefill work is being skipped,
- latency and input cost can drop significantly,
- the product workload has strong structural repetition.

What it does **not** automatically mean:

- that every request is cheap,
- that the model itself has become more efficient in all phases,
- that decoding cost disappears,
- or that the system is globally optimized in every dimension.

The cache mostly helps with repeated prompt prefixes. It does not eliminate the cost of generating new tokens, and it does not rescue badly designed prompting patterns.

This matters because cache hit rate is easy to over-romanticize. It is a strong systems metric, but it is not the same thing as product quality. A product can have excellent cache locality and still feel confusing, unreliable, or badly scoped.

## 8. When hit rates collapse

The article also gives the reverse view, which is useful.

Hit rates drop quickly when the workload breaks prefix locality. Typical examples include:

- switching models in the middle of a workflow,
- changing the system prompt or tool definitions,
- keeping conversations too short,
- routing requests across backends that do not share cache state,
- inserting or editing content in the middle rather than appending at the end.

This is a good reminder that cache hit rate is partly a systems metric and partly a workload-shape metric.

If the usage pattern stops looking like "stable prefix + appended tail," the cache loses its leverage.

## 9. The simplest mental model

If I reduce the whole article to one sentence, it would be this:

**LLM cache hit rates get close to 90% when requests keep reusing the same long prefix and only append small new tails.**

That is why agent workflows are such a natural match:

- they preserve history,
- they preserve tools,
- they preserve instructions,
- and they keep extending instead of rewriting.

From that angle, the 90% number stops looking mysterious. It becomes a consequence of how the workload is shaped.

## 10. What I am taking away

The part I found most valuable is that cache hit rate is not just an infra curiosity. It is also a product signal.

A high hit rate often means the system is being used in a stable, stateful, agent-like way. It suggests that prompt structure is consistent and that the serving stack is successfully exploiting that consistency.

So the question "why is the hit rate 90%?" is really two questions at once:

1. what kind of caching system is running underneath?
2. what kind of user workflow is producing such reusable prefixes?

That is what made the article worth reading for me. It turns a dashboard number into a much clearer systems story.

It also leaves me with a broader product thought. As AI products become more agentic, the boundary between product design and inference engineering keeps getting thinner. Choices that look like UX decisions, such as whether a workflow keeps stable instructions, whether a conversation appends or rewrites history, or whether tools are declared consistently, can have direct effects on cost and latency. That makes caching feel less like a hidden optimization and more like part of the product architecture itself.

In that sense, a 90% cache hit rate is not just a happy backend metric. It is often evidence that the system has found a stable way to carry context forward.
