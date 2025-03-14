---
title: "Thoughts on Functional Programming"
date: 2025-03-14T21:33:12+01:00
author: "Paul Hübner"
type: "post"
image: ""
tags: ["haskell", "fp"]
---

One of the earliest frustrations I encountered in programming was that I had to re-do stuff I had already done before, maybe 6-12 months after first starting.
Often writing similar code that was just about different enough to make copy/paste or reuse tricky is not fun, especially when the novelty factor is gone.
Not to mention all the Java boilerplate...

<!--more-->

I like declarative programming because it allows me to focus on what I want to do, and not how I want to do it.
It's one of the reasons why I grew so fond of `java.util.stream`.
Instead of having to write the same big for-loop, I could now get the data I wanted super concisely.
Using Scala was extremely pleasant, enough to convince myself to learn Haskell one morning.
Before I knew it, I found myself becoming a functional programming enthusiast.

I wrote Haskell professionally for a while, and I became familiar with some advanced concepts, the most important one being **algebraic effects**.
Attempting to explain these would be futile, so I will instead refer to the following excellent resources:
* [This talk](https://www.youtube.com/watch?v=y5jZnMImbMY) walks the viewer from monads to algebraic effects in an engaging and educational way, and you can take something away regardless of your background. It's probably the best talk I've ever seen.
* Dan Abranov's [blog post](https://overreacted.io/algebraic-effects-for-the-rest-of-us/) uses a very practical approach.
* James Hobson's [blog post](https://www.hobson.space/posts/algebraic-effects/) for the very lazy TL;DR.
* While not strictly about effects, this [tutorial](https://serokell.io/blog/introduction-to-free-monads) on how to use/make `Free` monads.
* The [Effectful](https://hackage.haskell.org/package/effectful) package that allows you to implement (fast) algebraic effects in vanilla Haskell.

**With effects, I felt ecstatic about programming again.**
I could compose building blocks arbitrarily, using a beautiful and concise syntax, without taking a terrible performance penalty.
It was now almost trivial to compose error handling with readers and state.

Unfortunately, it's too good to be true.
When you have an effect stack of 10+ effects (most of them being `State a`, and neccessary to distinguish as having one big record would not work for composition), you're going to get a headache tracking state.
And yes, this in a domain where FP is easily the best tool for the job.

I came to realize **my biggest unsolveable problem is state**; immutable or not, eventually, it becomes too much.
At some point programing boils down to just state manipulation.
And it's an absolute *drag*. I call it **state fatigue**.
And while my Haskell code was so abstract it was almost an imperative embedded DSL, it was also massively painful to debug.
Imperative languages have much better tooling to debug (and also much better tooling in general, in my opinion).
I think that was the day I realized I had gone full circle.

A more practical industry-related remark from me about working with FP: it's exhausting.
When I'm typing/tabbing `public class ABC implements XYZ`, my brain is idle, and I can relax.
When I write Haskell, not so much.
I tend to spend about 80% of my time thinking, and then 20% of the time writing.
I've never been as exhaused after 8 hours (of actually thinking 8 hours) as when I was working with Haskell full-time. 

**So, what's the takeaway?** I don't know, to be honest
I like FP. I really enjoy working with Haskell + Effectful, I don't think that will stop.
Elixir is also fantastic, it has great tooling, cuts a little bit of purity for convenience, and makes me so happy every time I use it.
But all roads lead to state. *The sooner I accept that no language or paradigm will make state manipulation as bearable, let alone fun, as it was when I was 13, the better.*
I genuinely understand the urge to become a farmer now.
