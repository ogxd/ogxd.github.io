---
title: "When to care about performance"
date: 2024-02-03T08:11:55+02:00
draft: true
summary: 
tags: 
- dotnet
- performance
- optimization
- debt
---

Trauma of the past

Moore's law

Culture of productivity

Many languages and frameworks have been designed to make developers more productive. This is a good thing, but it has a side effect: it makes developers less aware of the performance implications of their code.

Feels like a waste of time. Frustration

This article goes contre courant. The point is not to say
In fact, I wrote this article in VS code, and github copilot kept suggesting exactly the kind of banalities I am willing to fight in this article.

Leave technical debt of code optimization for another article.

"Premature optimization is the root of all evil" - Donald Knuth

Problems
- When should I care about performance? What is "premature optimization"?
- 

What is performance? CPU? Latency? Memory usage? 
Performance is 
Big O notation space & time complexity

Code optimization, trade-off and entropy


Context
What is the frequency? How often my function is going to be invoked?

---

TODO

# TL;DR; for lazy readers

When planning a development, think about how often (roughly) the resulting code will be invoked.
- **If the code isn't going to be invoked often**, then you don't necessarily need to care about performance. Keep it simple, and focus on readability, maintainability and testability. 
- **If the code is going to be invoked often**, then you should care about performance (the more often the more you should). Plan some extra time for benchmarking, optimization passes and profiling in the real world.

# Time is money

The performance of the code developers write is roughly proportional to the time they spend on it. Experience may modulate this relationship, but it's still a good rule of thumb.

<img style="width:256px" src="perf-vs-dev-time.png" />

Unless you're only programming for fun, it's more than likely that you are writing code for a few bucks, and most likely you're paid based on time. This means it is expected for you to be productive, thus limiting the time you can spend coding a given feature.

<img style="width:256px" src="perf-vs-dev-time-bounded.png" />

TODO

# What's the context?

TODO

## Frequency

TODO

## Volatility (a.k.a. "code churn")

Optimization does not necessarily means increased complexity: often, simplicity is the key to performance. Still, we can't deny that once the simplicity card has been played, the next optimization levers can imply more complex data structures, exotic algorithms, or unsafe code to name a few. This can make a piece of code more complex to grasp, maintain and evolve.  

For this reason, it's important to consider the volatility of the code. If the code is going to be modified often, then you should be more cautious about the trade-offs you make. For instance, a new business feature is likely to receive a lot of changes, so you may want not to go too deep while optimizing. On the other hand, a change on the logging stack is less likely to change (given its company-internal nature and the fact that logging is a well-defined practice), thus is more appropriate to use more complex optimization techniques in this context. 

## Some examples

### Example 1: Write some internal tooling for your company

TODO

### Example 2: Do a background work for a desktop software

TODO

### Example 3: Apply business rules against incoming requests for a web service

TODO

### Example 4: Change some internals of the Linux kernel networking stack

TODO

# So what then?

TODO