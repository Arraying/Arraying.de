---
title: "GC-Agnostic Load Barriers"
date: 2025-08-27T18:00:00+01:00
author: "Paul Hübner"
type: "post"
image: ""
tags: ["java", "jvm", "gc"]
---

What are GC barriers and how do they relate to ahead-of-time compilation? This blog post explores the intersection of HotSpot's GCs and compilers through the lense of implementing GC-agnostic load barriers at the assembly level, and gives key insights on performance and overall takeaways.

<!--more-->

**This blog post was originally published on Oracle's Java blog.**
[Click here to be redirected there instead.](https://inside.java/2025/08/27/thesis-gc-agnostic-load-barriers/)

---

I recently completed my Master's degree in Software Engineering of Distributed Systems.
For my thesis, I collaborated with Oracle's HotSpot GC and compiler development teams in the Stockholm office.
With this post I would like to present my work and shed light onto the following topics:
* warmup process and how Project Leyden's looks into improving that;
* what loads are and why they sometimes require GC barriers;
* the Z Garbage Collector's actual barrier Assembly instructions;
* how to construct GC-agnostic barriers.

## Warmup

Like an airplane spooling up its engines to reach take-off speed, Java code on the HotSpot JVM undergoes a process called warmup before it starts executing as fast as possible.
During warmup, applications are interpreted and/or profiled, in order to determine how to just-in-time compile them into optimized machine code.
This is in contrast to ahead-of-time compilation (e.g., C++), where application code is optimized at compile time, bypassing the need for warmup at runtime.
Still, runtime-driven optimizations cannot be baked in, as they are in the HotSpot JVM.

Luckily, we can have our cake and eat it too.
[Project Leyden](https://openjdk.org/projects/leyden/) explores how to decrease warmup by doing ahead-of-time compilation.
A training run executes the application, allowing warmed-up machine code to be used immediately during production runs.
This bypasses the need to warm up in production, drastically increasing throughput.

## Problem Statement

A major drawback of the current ahead-of-time compiled code is that it requires the same GCs during training and in production.
This detriments runtime flexibility, as different GCs have [different advantages/disadvantages](https://docs.oracle.com/en/java/javase/24/gctuning/available-collectors.html).
The culprits for this requirement are GC barriers, which are snippets of bookkeeping instructions inserted around memory operations.
This bookkeeping is crucial to ensure that the GCs remain sound!
Importantly, *where*, *when*, and *how* to do this bookkeeping depends entirely on the memory operation and GC.
Conesquently, compiled code using one GC cannot be used with another GC.

In my work, I only focused on one memory operation: field accesses (e.g., `int foo = point.x;` represents an access of field `x`).
Field accesses are called **loads**, and I will use this terminology going forward.
My work centered on unifying the *when* and *how* to form GC-agnostic load barriers.
Since most things in computer science are trade-offs, my research question was: **how does code using GC-agnostic barriers perform compared to GC-specific barriers?**

## Implementation

In my thesis, I delimited myself to inspecting Serial, Parallel, G1 and Z Garbage Collectors on AArch64 (my laptop's architecture).
I only focused on the [C2 JIT-compiler](https://openjdk.org/groups/hotspot/docs/HotSpotGlossary.html) as this produces the long-living machine code and Project Leyden uses it.
In this section, I will only consider [uncompressed Oops](https://wiki.openjdk.org/display/HotSpot/CompressedOops), the only compression mode supported by all four of the above GCs.

### Answering *when*

There are several operations that could take place during loads, depending on the type of reference.
When an object's field is read, this is a strong reference, and the access classified as a **strong load**.
Java also has [weak references](https://docs.oracle.com/en/java/javase/24/docs/api/java.base/java/lang/ref/WeakReference.html), whose accesses are **weak loads**.
If an object is only referred to by weak references, it will be garbage collected.
This is useful for caches, and is for example used by [`WeakHashMap`](https://docs.oracle.com/en/java/javase/24/docs/api/java.base/java/util/WeakHashMap.html).

Presently, whenever a load occurs, strong or weak, a load barrier *may* be emitted *before* the field is dereferenced.
Under which condition this happens is shown in the table below.

| GC | Strong load emits barrier? | Weak load emits barrier? |
| --- | --- | --- |
| Serial | ❌ No | ❌ No |
| Parallel| ❌ No | ❌ No |
| G1 | ❌ No | ✅ Yes |
| Z | ✅ Yes | ✅ Yes |

The takeaway here is that for both strong and weak loads, at least one GC emits a barrier.
Therefore, a GC-agnostic load barrier must emit a load barrier at every strong and weak load.

### Answering *how*

> ℹ️ This section dives into AArch64 assembly. It is *not* required to understand why this assembly does what it does, only the instructions and their operands are relevant!

The high-level idea is to emit the same instructions for all GCs (for strong and weak loads, respectively), and then to "patch up" the operands to match GC-specific requirements during first-time execution.
HotSpot-compiled machine code sits in memory, with each instruction being 32 bits wide (on AArch64).
To patch an instruction at a given memory location, one can cast its value to a 32-bit unsigned integer and modify it with bitwise operations.
Patching is not novel to GC-agnostic barriers, as Z already employs this technique for its barriers.

GC agnostic barriers are best illustrated by example, I will use that of strong loads.
Since Z is the only GC that requires a load barrier for strong loads, its load barrier is used as the template for the agnostic load barrier.
Out of the box, it then works for Z.
Using instruction patching at runtime, Serial, Parallel, and G1 then modify the load barrier's instructions to not do anything.

Z's strong load barrier looks as follows.
In this case, the object reference is a [colored pointer](https://youtu.be/YyXjC68l8mw?feature=shared&t=816) stored in register `0`.

```
tbz  w0, Z_MASK, #0x8
b    Z_SLOW_PATH
lsr  x0, x0, #16
```
If some bit of the address - identified by `Z_MASK` - is zero, it jumps over the next `b` instruction, not executing it.
The next instruction branches to a "slow path", which is a GC terminology for necessary but slow bookkeeping instructions.
Lastly, the [colored pointer](https://youtu.be/YyXjC68l8mw?feature=shared&t=816) is stripped of its least-significant 16 bits, the metadata bits, retaining just the address.

Serial, Parallel, and G1 Garbage Collectors need to make sure that this barrier doesn't do anything.
They do not need to perform bookkeeping for strong loads.
This is done by patching the operands of the instructions in such a way that no actual work gets done.
The result is as follows:

```
tbz  wzr, Z_MASK, #0x8
b    Z_SLOW_PATH
lsr  x0, x0, #0
```

Instead of testing the address register `0`, the first instruction is changed to operate on the zero register `wzr`, whose value is always zero.
Therefore, it will always jump over the next `b` instruction, and Z's bookkeeping is never performed.
The address is still in register `0`, and since the other GCs do not have colored pointers, the amount shifted right by is patched to zero, which leaves the address unmodified.
In the end, no actual work gets done.

To summarize, **by patching operands, we've achieved GC-agnosticism for strong loads!**
The same ideas apply to the weak load barrier, omitted for brevity, where the instructions are a hybrid between those of G1 and Z.

## Performance Results

Emitting a bunch of extra instructions for GCs that do not need them has got to be detrimental to performance, right?
Unsurprisingly, yes.
When looking at execution time and P90 latencies, the *worst-case* regressions are shown in the table below:

| GC | Execution time regression | P90 latency regression |
| --- | --- | --- |
| Serial | 17% | 15% |
| Parallel | 19% | 17% |
| G1 | 17% | 20% |
| Z | 23% | 39% |

These results are **not necessarily bad!**
Many experiments only had single-digit percentage regressions.
Furthermore, interpreted code is generally an order of magnitude slower than compiled code, so in the end, there is still a net benefit.

## Conclusions and Future Work

GC-agnostic load barriers can be implemented through instruction patching.
A trade-off between the ability to choose which GC and runtime performance has to be made when using ahead-of-time compilation.
Nonetheless, GC-agnostic load barriers present a viable prototype.
To gain a broader understanding of the significant impact of GC-agnostic load barriers, the next step would be to integrate them with GC-agnostic store barriers, as investigated by my colleagues Arveen Emdad and Antón Seoane Ampudia.

Thank you to everyone at the Stockholm office for their guidance and support.
In particular, thank you to Erik Österlund, Roberto Castañeda Lozano, Tobias Wrigstad, and Daniel Lundén for their supervision and support, and Jesper Wilhelmsson for this tremendous opportunity to explore the inner workings of HotSpot's C2 compiler and GCs.
Thanks to Ana Maria Mihalceanu and Antón Seoane Ampudia for feedback on this post.

If this post piqued your interest, I encourage you to take a look at my [thesis report](https://urn.kb.se/resolve?urn=urn:nbn:se:kth:diva-368017) as it contains more details and results.
