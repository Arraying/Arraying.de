---
title: "Help, My Java Object Vanished (and the GC is Not at Fault)"
date: 2025-10-18T12:00:00+02:00
author: "Paul Hübner"
type: "post"
image: ""
tags: ["java", "jvm", "compiler", "valhalla"]
---

Today I'm going to talk about a recent journey as a HotSpot Java Virtual Machine developer working on the OpenJDK project. While running tests for a new feature, I realized my Java objects and classes were arbitrarily disappearing! What followed was probably the most interesting debugging and fixing experience of my life (so far), which I wanted to share with the world.

<!--more-->

This post is targeted towards a wider (computer science) audience. Knowledge about Java or the JVM is **not expected**, but a slight curiosity towards low-level programming is encouraged. My intentions with this essay are to:
1. introduce Project Valhalla and value objects;
2. give insights into the inner workings of HotSpot (& hopefully motivate folks to contribute);
3. pragmatically demonstrate how JVM flags can be used to help you, the developer;
4. teach some lessons I learned in debugging;
5. document processes for my future self/colleagues; and
6. give myself an opportunity to yap about an achievement (and draw bad ASCII art).

I've written *a lot*. **I've included summaries** for each of the sections, sans takeaways. They're designed to be coherent, so use them to your advantage and *read what you find interesting*, although I truly believe there is value in reading the entire post.

## Introduction

So what was I doing? I was [changing the `markWord` to conform to the format specified by JEP450](https://bugs.openjdk.org/browse/JDK-8367073) for [Project Valhalla](https://openjdk.org/projects/valhalla/). It's likely these terms won't mean anything to you, so let me break it down.

### Java 101

Java is a multi-purpose programming language. It is compiled to platform-independent Java bytecode. This bytecode is then executed by a Java Virtual Machine, which performs, among other things, automatic garbage collection. The reference implementation of the JVM is called HotSpot. It features just-in-time compilation, which compiles (some) frequently executed Java methods into native code for performance gains.

### Object Headers

Java objects reside in the heap. Each object has metadata associated with it, which is stored in the **object header**. If there's anything I want you to take away from this, it's that there are some bits in there that are very interesting.

Traditionally, the object header consists of the **mark word** (called `markWord` in source code) and **class word** (contains the class pointer, which is totally irrelevant to this blog post), totaling 96-128 bits on 64-bit architectures. I'll go more in-depth about the mark word soon.

In order to reduce the size, JDK 24 introduces [JEP 450: **Compact Object Headers**](https://openjdk.org/jeps/450)[^1]. Essentially, they realized that it was possible to squeeze the class pointer into the mark word, making the class word redundant. Consequently, with Compact Object Headers, object headers total 64 bits on 64-bit architectures.

[^1]: Compact Object Headers was made a production feature in JDK 25 via [JEP519](https://openjdk.org/jeps/519), though it remains disabled by default at the time of writing.

So, what sort of information is in the mark word? JEP 450 does a good job at [explaining the exact layout](https://openjdk.org/jeps/450), but for the sake of this blogpost only the eleven least-significant bits are relevant. They're laid out on 64-bit systems as follows[^2]:

[^2]: The actual layout of the `markWord` follows the Compact Object Headers format regardless of whether they are enabled. If the feature is disabled, the class pointer will just be delegated to the class word. You can [find the layout in the source code](https://github.com/openjdk/jdk/blob/5251405ce9ab1cbd84b798a538cb3865ea4675e9/src/hotspot/share/oops/markWord.hpp).

```
    7   3  0
 VVVVAAAASTT
          ^^----- tag bits
         ^------- self-forwarding bit (GC)
     ^^^^-------- age bits (GC)
 ^^^^------------ Valhalla-reserved bits
```

Let's *briefly* go through what these mean:
* `TT` represents the two tag bits, and they are used for locking. I'm not going to go into monitoring in depth. Briefly, `01` indicates the object is not locked, whereas `00` and `10` indicate lightweight[^3] and monitor[^3] locking, respectively.
* `S` represents the self-forwarding bit. It is used by some garbage collectors, but the details are irrelevant.
* `AAAA` represents the age bits. Generational garbage collectors use these bits to track object tenuring.
* `VVVV` represents four distinct Valhalla-reserved bits (and also called `unused_gap_bits` in the implementation). More on them later.

[^3]: More detail on the locking modes can be found in the experimental [Compact Object Headers JEP](https://openjdk.org/jeps/450).

### Project Valhalla

[Project Valhalla](https://openjdk.org/projects/valhalla/) is also dubbed "Java's Epic Refactor" with many feature sets in development. The relevant one here is [JEP 401: Value Classes and Objects](https://openjdk.org/jeps/401). In short, value objects are distinguished by their fields, which allows for optimizations such as heap flattening[^4] and scalarization[^5]. Note that "regular classes/objects" are called **identity classes/objects**.

[^4]: Brian Goetz illustrates this nicely in his [JVMLS Valhalla update](https://www.youtube.com/watch?v=IF9l8fYfSnI) talk.

[^5]: Instead of passing references to value objects, their fields can end up as local fields or in registers for better performance.

Valhalla is a fork of the JDK. Although we frequently pull in upstream changes, Valhalla naturally lags behind mainline development. When Compact Object Headers was merged into Valhalla, a slightly different layout of the eleven least-significant bits was chosen, such that the merge would be *less invasive*. The chosen format was as follows (contrasted with the JEP 450 layout):
```
    7   3  0
 VVVVAAAASTT  <- JEP 450
 VVVAAAAVSTT  <- Valhalla
        ^------- we care about this bit, value object bit
```

As illustrated, the least-significant Valhalla bit resided below the age bits. **I'd like to draw attention to the `V` in position 3**[^6]. This is the bit that, when set, indicates whether the object is a value object. **It's also the central concept in this post**. Feel free to disregard bits 8-10.

[^6]: Internally, this is called the "inline type bit," since value classes can be seen as inline types. However, this terminology is confusing, so I'm going to call it the value object bit in this post.

Why do we need this bit? HotSpot does different things for value objects and identity objects in many places (for example, the optimizations I mentioned earlier). We need a quick and easy way to check if something is a value object, hence a bit in the object header.

### My Change

My change is pretty straightforward: updating the eleven least-significant bits from `VVVAAAAVSTT` to `VVVVAAAASTT` (as JEP 450 dictates). You can think of this as moving the value object bit up (or age bits down).

### Summary

The mark word is part of the object header and contains metadata for Java objects. Project Valhalla introduces value objects. These are objects distinguished solely by the values of their fields. In Valhalla, one of the header metadata bits indicates whether the object is a value object. In Valhalla, this was at bit index 3, and needed to be moved to bit index 7 to comply with Compact Object Headers (JEP 450).

## Widespread Failure

The change was relatively straightforward. Unfortunately, it caused 75 test failures on several architectures and platforms. These test failures were problematic because they were:
1. **Widespread**. Many different components (mostly *outside* the VM) were affected.
2. **Intermittent**. They would sometimes pass, and sometimes fail. As I would later find out, this also made them difficult to reproduce.
3. **Non-obvious**. In the VM, we have a plethora of assertions. Most of the time something goes wrong, an assertion fails or the VM crashes with e.g. a segmentation fault. Here, it seems like (almost all) the issues manifested at the application level.

When it comes to debugging, that's a really bad scenario. Considering the HotSpot codebase is ~550 KLoC[^7], not to mention extremely complex, we really want reproducible crashes in a subcomponent.

[^7]: Estimate via `find src/hotspot -type f | xargs wc -l`. Much of this is probably copyright headers.

Throughout my debugging journey, three distinct symptoms were relevant. Together, they describe many of the failing test cases.

### Symptom A: Failed Unit Tests

In HotSpot, we have unit tests (written with the GoogleTest framework). Four of the [`markWord` tests](https://github.com/openjdk/valhalla/blob/a687d94e6afcb5326f558f196dcefe2dbbc72b10/test/hotspot/gtest/oops/test_markWord.cpp) failed:
```
[  FAILED  ] 4 tests, listed below:
[  FAILED  ] markWord.inline_type_prototype_vm
[  FAILED  ] markWord.null_free_flat_array_prototype_vm
[  FAILED  ] markWord.nullable_flat_array_prototype_vm
[  FAILED  ] markWord.null_free_array_prototype_vm
```

### Symptom B: `java.lang.AssertionError`

There were multiple tests, such as [`java/util/jar/JarFile/mrjar/MultiReleaseJarProperties.java`](https://github.com/openjdk/valhalla/blob/a687d94e6afcb5326f558f196dcefe2dbbc72b10/test/jdk/java/util/jar/JarFile/mrjar/MultiReleaseJarProperties.java) or [`java/util/logging/modules/GetResourceBundleTest.java`](https://github.com/openjdk/valhalla/blob/a687d94e6afcb5326f558f196dcefe2dbbc72b10/test/jdk/java/util/logging/modules/GetResourceBundleTest.java) which failed in the **TestNG harness** with a Java `AssertionError`[^8].

```
java.lang.AssertionError: l should not be null
	at org.testng.ClassMethodMap.removeAndCheckIfLast(ClassMethodMap.java:55)
	at org.testng.internal.TestMethodWorker.invokeAfterClassMethods(TestMethodWorker.java:193)
    [...]
	at com.sun.javatest.regtest.agent.MainActionHelper$AgentVMRunnable.run(MainActionHelper.java:335)
	at java.base/java.lang.Thread.run(Thread.java:1474)
```
[^8]: These are different than the assertion errors in HotSpot and reside in the application space.

What's very curious about this failure is if you check [the source code](https://github.com/testng-team/testng/blob/d50b2ad2d6809d52131a07071fe229b1b901e08c/testng-core/src/main/java/org/testng/ClassMethodMap.java#L46), they actually perform a null check on the variable `l`, so it shouldn't just disappear!
```java
public boolean removeAndCheckIfLast(ITestNGMethod m, Object instance) {
  Collection<ITestNGMethod> l = classMap.get(instance);
  if (l == null) {
    throw new IllegalStateException(
        "Could not find any methods associated with test class instance " + instance);
  }
  l.remove(m);
  // It's the last method of this class if all the methods remaining in the list belong to a
  // different class
  for (ITestNGMethod tm : l) { // <- AssertionError thrown here
    if (tm.getEnabled() && tm.getTestClass().equals(m.getTestClass())) {
      return false;
    }
  }
  return true;
}
```

### Symptom C: `java.lang.NoClassDefFoundError`

A few tests within `sun/security/krb5` such as [`sun/security/krb5/etype/UnsupportedKeyType.java`](https://github.com/openjdk/valhalla/blob/a687d94e6afcb5326f558f196dcefe2dbbc72b10/test/jdk/sun/security/krb5/etype/UnsupportedKeyType.java) failed with a `NoClassDefFoundError`. The actual class that could not be found differed between tests and even between test runs of the same test. Classic intermittent shenanigans.

```
java.lang.NoClassDefFoundError: sun/security/krb5/internal/Krb5
	at java.base/jdk.internal.loader.NativeLibraries.load(Native Method)
	[...]
	at java.base/java.lang.Thread.run(Thread.java:1474)
```

### Summary

I implemented the `markWord` changes and got many widespread, intermittent, and non-obvious test failures. Crudely summarized, "things are null when they shouldn't be."

## Act I: Semantic Difficulties

The lowest hanging fruit here are the failed unit tests. Not only are they written in C++, they also expose the functions they are testing. The failure was straightforward:
```cpp
EXPECT_TRUE(mark.decode_pointer() == nullptr);
```
Where `mark` is the `markWord` of a value object. Taking a look at the source code:
```cpp
// Recover address of oop from encoded form used in mark
inline void* decode_pointer() const {
  return (EnableValhalla && _value < static_prototype_value_max) ? nullptr :
    (void*) (clear_lock_bits().value());
}
```
It became clear to me that there are some Valhalla-specific shenanigans going on (by the `EnableValhalla`). But what's a `static_prototype_value_max`?

### Static Prototypes

Looking in the `markWord.hpp`, there were a few constants related to `static_prototype`:
```cpp
  static const uintptr_t static_prototype_mask    = ...;
  static const uintptr_t static_prototype_mask_in_place = static_prototype_mask << lock_bits;
  static const uintptr_t static_prototype_value_max = (1 << age_shift) - 1;
```
It turns out, `static_prototype_mask` and `static_prototype_mask_in_place` are unused. The `static_prototype_max_value` in question is just all bits up to the age bits set to 1. Recall that the pre-JEP450 Valhalla bits are `AAAAVSTT`. This would generate a mask of `1111`. Since I moved the position of the age bits, this mask changed. Maybe that's why the test fails?

Why do we do this? The documentation says the following about static prototypes:
```cpp
//  Static types bits are recorded in the "klass->prototype_header()", displaced
//  mark should simply use the prototype header as "slow path", rather chasing
//  monitor or stack lock races.
```

What are static type bits? What's a static type? I have no clue. My colleagues said it was part of an older model of Valhalla and no longer a thing. The verdict: **delete the conditional and remove all references to static types**. I did that, and while the (now slightly altered) unit tests passed, all my application-level failures still persisted.

### Emulation

One of the things I've been taught in education and practice is that in general one should focus on understanding. While that's certainly true, I don't think anyone understands HotSpot, and knowledge is very compartmentalized. Small changes in one place could go against assumptions in another and cause bizarre failures.

To save myself a reasoning-induced headache, I decided to emulate the old behaviour in `decode_pointer()` by translating the current `markWord` into the legacy format and then keeping the static type check. Maybe there was some counterpart or invariant somewhere in the VM that magically made it work. If the emulated version worked, it would give me some more clues about where to poke around further.

The process of emulating wasn't very straightforard:
1. Changing the `markWord` required recompilation of many, many files, leading to ~10 minute builds for every change. Very annoying when trying to rapidly iterate.
2. To run tests (incl. unit tests), the infrastructure needs to build the JDK. To build the JDK, one needs to compile classes via `javac`, which in turn uses HotSpot. If one makes catastrophic changes in HotSpot, one will crash while building the JDK, meaning **one can't run the C++ GoogleTest unit tests**.

Specifically with regards to (2), there was an assertion plaguing me:
```cpp
markWord m = markWord::encode_pointer_as_mark(p);
assert(m.decode_pointer() == p, "encoding must be reversible");
```
It's not important to know what this does, just that encoding a mark word must be [invertible](https://en.wikipedia.org/wiki/Inverse_function). So, whatever I was doing in `decode_pointer` broke this property.

Since much of the `markWord.{hpp, cpp}` was just pure C++, I ended up copying the files into its own C++ project and iterating there. This gave me quicker turnaround times, and I could test for invertibility without breaking the JDK build. It ain't pretty, but it worked.

Unfortunately, even with emulation, I still saw a plethora of test failures. The dreaded `java.lang.AssertionError` and `java.lang.NoClassDefFoundError` errors persisted. I stripped the emulation, removed the static type checking code again, updated the unit tests, and cut my losses.

### Summary

When debugging the failed unit test, I realized it tested for semantics that no longer existed. Even when emulating the old behaviour, my issues still persisted. While I had succeeded in removing dead code and updating unit tests, this didn't fix the plethora of broken Java application-level tests.

## Act II: Reduce, Reproduce & Recycle

> ℹ️ I've linearized the order of events in this Act for readability and understandability. In reality, I bounced between the failing test cases and investigations quite a lot. I'm also repeating some principles that may seem obvious, but I believe there is value in restating them.

Failures in application code can be notoriously difficult to debug. What's particularly hard is when you don't know where the issue(s) originated from. If a HTTP server errors, it's probably in that route's controller or in the middleware stack. That's not to say that it's straightforward to understand or fix (especially getting to the root of cascading microservice failures!), but it provides a starting point.

HotSpot has many moving parts, so in many cases this adds more dimensions to the problem. *Temporally*, failures can manifest much later after the introduction of the bug. For example, only after the 10th garbage collection cycle. The *locality* is also non-obvious sometimes. Take a miscompilation[^9] for example, it could change the state of a variable and break an invariant a completely unrelated component of the virtual machine. Assertions *can* stop bugs from spreading too far, but can't ensure locality.

[^9]: A miscompilation happens when a compiler emits wrong code. Ideally, this is garbage and we crash upon execution, but in unfortunate cases it can introduce extremely subtle bugs into programs.

To stay sane, it's important to **reduce the size of the programs** and to ensure that **bugs are reproducible**. Unfortunately, neither of these can be taken for granted.

### Action Plan

With that in mind, it was time for an action plan. These are questions I try to answer in order to get a better understanding of what is going on, a prerequisite to bugfixing.
- **When are we failing?** Can we increase and decrease our chance of failure?
- **Where are we failing?** This has several sub-questions:
    - With which *component/feature* (e.g. garbage collection, Compact Object Headers) does the bug manifest?
    - Is there a place in HotSpot source code that is suspicious?
    - Where does the bug manifest in the application code?

### C2

On top of a bytecode interpreter, HotSpot has **two just-in-time compilers, C1 and C2**. There is no "the JIT compiler". C1 compiles fast but is limited in terms of optimizations, C2 produces extremely optimized machine code but takes comparatively long to run. In the past, client VMs (i.e. those for end users) would ship C1, and server VMs came with the C2 compiler. Nowadays, both C1 and C2 compile your Java code in a process called [tiered compilation](https://www.baeldung.com/jvm-tiered-compilation)[^10]. Note that **not all methods will be JIT compiled**. HotSpot uses extensive profiling to decide which methods are worth compiling with which compiler.

[^10]: This is one of the best developer-oriented overviews of tiered compilation I've seen. Big kudos. Even though there are some things I would nitpick (e.g. performance is rarely, if ever, monotonically increasing), I still find myself coming back to this for its succint high-level explanation.

There are six garbage collectors in the source code, though those that get shipped depends on your JDK vendor. My personal configuration include Serial & Parallel[^11], G1[^12], and Z[^13].

[^11]: I really like the overview [this blogpost](https://abiasforaction.net/understanding-jvm-garbage-collection-part-6-serial-and-parallel-collector/) by Akhil Mehra gives.*

[^12]: [Oracle's explanation](https://docs.oracle.com/en/java/javase/25/gctuning/garbage-first-g1-garbage-collector1.html) of G1 is a fantastic source of truth, but quite information-dense and I found it challenging the first few reads. In that sense [Akhil Mehra's G1 post](https://abiasforaction.net/understanding-jvm-garbage-collection-part-9-garbage-first-g1-garbage-collector-gc/) is a strong alternative.

[^13]: ZGC is the coolest GC in my opinion, but quite advanced. [This master thesis](https://urn.kb.se/resolve?urn=urn:nbn:se:uu:diva-539617) explains the surface level concepts quite well.

The table below shows how I generally target components. Initial trial-and-error found that the tests mentioned above would usually fail within five runs. So, I repeated each experiment 20 times.

| Component to target | JVM flags | Description |
| --- | --- | --- |
| Interpreter | `-Xint` | Forces all code to be run in the interpreter. |
| C1 only, trivial methods | `-Xcomp -XX:TieredStopAtLevel=1` | Forces JIT compilation of *everything*, and stops after C1 compiling trivial methods.
| C1 only, fully compiled | `-Xcomp -XX:TieredStopAtLevel=3`|  Same as above, but with more profiling.
| C2 only | `-Xcomp -XX:-TieredCompilation` | Compilation without tiers forces C2. |
| Serial GC | `-XX:+UseSerialGC` | Tells HotSpot to use Serial. |
| Parallel GC | `-XX:+UseParallelGC` | Tells HotSpot to use Parallel. |
| G1 GC | `-XX:+UseG1GC` | Tells HotSpot to use G1, though this is the default in most cases. |
| Z GC | `-XX:+UseZGC` | Tells HotSpot to use Z. |

I didn't actually re-run all the possible compiler and GC combinations possible (16 total). Instead, **I assumed that there was no interaction between compilers and GC** and that they could be tested independently. Why? I had played around with heap sizes and it seemed to have no effect on failure frequency. Generally, **smaller heaps mean more frequent garbage collection** and a higher likelihood of manifesting. Just to be clear, this was a gamble overall, as **no interaction is not always guaranteed** and if was wrong I'd end up wasting a lot of time and resources.

So, what did I actually do? *Did my gamble pay off?*
1. I picked a failing test and ran it interpreter-only, saw that it passed after 20 iterations. So it's probably a compiler issue.
2. Since I know the interpreter works, I skipped `-Xcomp` and just added `-XX:TieredStopAtLevel=1` for trivial C1 compilation. This meant that my code would get interpreted until HotSpot decided some trivial methods would get compiled. This passed 20 iterations too.
3. At this point I could have tried full C1 compilation, but I had a hunch it's probably C2. So I ran with `-XX:-TieredCompilation` and behold, the bug manifested!
4. As a sanity check, I ran with with C2 compilation on Serial, Parallel, G1 and Z. This was to ensure that all GCs see test failures, which they did.

I would say it paid off. I ran a few other test cases, they also confirmed that C2 compilation was the culprit. Using `-XX:-TieredCompilation` saw the bug manifest roughly 1/2 of the time. My colleagues suggested seeing if enabling Compact Object Headers with `-XX:+UseCompactObjectHeaders` makes a difference. And that was spot on, since I wasn't able to reproduce with it enabled. To answer the question of **when we fail**: when we use C2 without Compact Object Headers.

So what does this tell us? Application logic failures with no VM crashes (via assertion failure or otherwise) indicates **it's possibly (but not necessarily) at least one C2 miscompilation** when disabling Compact Object Headers. That's pretty bad luck, since C2 is extremely complex and miscompilations are really hard to track down on top of that.

### The `jtreg` Saga (Reproducing is Hard)

While I knew that the bug manifested when using C2 without Compact Object Headers, I still had no idea where the miscompilation happened. While there are many `if (UseCompactObjectHeaders)` in the HotSpot/C2 code, there are so many implicit interactions within the runtime one would need a lot of luck to find the bug that way.

As a VM engineer, there are many tools in our toolbox to help with this uncertainty. For example, [GDB](https://en.wikipedia.org/wiki/GNU_Debugger)[^14] allows stepping through the code and inspecting state. [`rr`](https://rr-project.org) is great, because it captures program behavior once, and then one can deterministically step through (usually forwards, but also backwards). That's invaluable with intermittent failures.

[^14]: I'm on macOS so I use [LLDB](https://lldb.llvm.org) instead, but it's more or less the same thing.

These tools take the `java` invocation as arguments. In OpenJDK, tests are run with the home-grown [`jtreg`](https://openjdk.org/jtreg/) framework, and invoked through `make test`, which in turn creates a `jtreg` Java process. When a test fails, it gives a "re-run command" which allows the develper to run it via `java` directly. This command is highly complex, since the `jtreg` subprocess usually spawns a few of its own subprocesses and supports different [execution modes](https://openjdk.org/projects/code-tools/jtreg/intro.html). Furthermore, two of my three symptoms were TestNG harnesses, which added another level of complexity.

Unfortunately, **the bug never manifested via re-runs**. I spent countless days trying, fiddling with the flags, to no avail. Without being able to observe failures directly (only via `make test`), debugging's difficulty is locked to hard mode. It seemed like I hit a dead end.

### Minimizing

With no clear debugging entrypoint, my only hope was to minimize the test cases. That way, I would be able to use unorthodox methods like `printf` statements or adding `ShouldNotReachHere()` assertions. I picked [`java/util/jar/JarFile/mrjar/MultiReleaseJarProperties.java`](https://github.com/openjdk/valhalla/blob/a687d94e6afcb5326f558f196dcefe2dbbc72b10/test/jdk/java/util/jar/JarFile/mrjar/MultiReleaseJarProperties.java) as a starting point.

I usually minimize in two steps:
1. Remove half of the method calls. I add them back one by one if the test no longer fails/the bug no longer manifests.
2. Once no more method calls can be removed, I inline the existing method calls, cleaning up/constant folding in the process. Go back to step 1 until nothing can be inlined further.

There are tools that do minimization programatically. [`creduce`](https://github.com/csmith-project/creduce) is a good example and also works for Java. I reduced `MultiReleaseJarProperties.java` manually since it gave me a lot of exposure to Java libraries I hadn't used before. Plus, it's good practice.

After many, many iterations, I couldn't inline any further since I hit the Java standard library. Unfortunately, [the minimized test](https://gist.github.com/Arraying/f3b68f0c4c35f493acd7826840492116) was still more complex than I had hoped it would be. Therefore, **another approach was necessary**...

### Summary

By trying a variety of HotSpot flags, I was able to narrow down the issue to a problem related to the C2 JIT-compiler with Compact Object Headers disabled. I managed to minimize a test case substantially, but could not narrow it down as much as I wanted to. I suspected that it is a miscompilation bug, which makes it very hard to track down.


## Act III: Compilation Mining

When HotSpot JIT-compiles programs, there is a *lot* of machine code produced, not only for the application itself, but also for the Java launcher. It's just not possible (or, extremely time intensive) to individually investigate every single compilation.

In my case, I couldn't minimize any further. So, I had to rely on automated techniques in order to try to localize where the bug was.

### What's Compiled?

In order to find out which methods get compiled, one can use `-XX:+PrintCompilation`. This prints out a long list of compiled methods, for example[^15]:
```
154    1     n       jdk.internal.misc.Unsafe::getReferenceVolatile (native)
206    2     n       java.lang.Object::hashCode (native)
207    3     n       java.lang.invoke.MethodHandle::linkToStatic(LLLLLLL)L (native)   (static)
219    4             java.lang.Enum::ordinal (5 bytes)
227    5     n       java.lang.System::arraycopy (native)   (static)
...
```


[^15]: Note that compilation levels are *not* shown in this case, as I am only using the C2 compiler with `-XX:-TieredCompilation`.

We're just interested in the method name in this case. For more information on what the different numbers and symbols mean, [I recommend Stephen Fox' blogpost](https://foxstephen.net/quick-ref-print-compilation).

In my case, there were ~400 compilations. Again, this would be insanely tedious to manually crawl. I needed to automate.

### Finding the Culprit(s)

There are two ways one can try to automatically find the method(s) causing issues. Of course, they assume that sufficiently many iterations are performed for the bug to manifest.
1. Go through every single method, and only compile that method. Every method that sees the bug manifest should be investigated further.
2. Go through every single method, and exclude this method from compilation. Only investigate methods whose exclusion *did not* cease the bug from manifesting.

I decided to go down the **first route**. The second route is *probably better* if there are interactions between methods that cause the issues. However, it takes longer to get to a result, so I decided to take a bet on the first method.

So how does one actually control compilations? By using `-XX:CompileCommand`[^16]. With this, I can specify if I *only* want to compile a specific method, or if I want to *exclude* another method. For example, `-XX:CompileCommand=compileonly,java.lang.Enum::ordinal` ensures that **if HotSpot decides to compile** `Enum::ordinal`, this is the only compiled method. Similarly, `-XX:CompileCommand=exclude,java.lang.Enum::ordinal` ensures that HotSpot will **never compile it**. Since we are only compiling with C2 (`-XX:-TieredCompilation` flag), these `CompileCommand`s control C2's behavior.

[^16]: A decently old but still useful blogpost on the CompileCommand option is that of [Jean-Philippe Bempel](https://jpbempel.github.io/2016/03/16/compilecommand-jvm-option.html).

Having gathered all compiled methods from `-XX:+PrintCompilation`, after preprocessing[^17], I wrote a little script to automate the search process. The gist of the script is as follows:
```java
// main(String[] args), while iterating through all methods:
String flag = "-XX:CompileCommand=compileonly," + method;
if (!run(flag)) {
    System.err.println("Failure for " + method);
}
// run(String otherFlags):
ProcessBuilder pb = new ProcessBuilder(
    "bash",
    "-c",
    "cd ~/valhalla && make test TEST=\"" + TEST + "\" JTREG=\"REPEAT_COUNT=5;VM_OPTIONS=-XX:-TieredCompilation -XX:+PrintCompilation " + otherFlags + "\""
);
```
[^17]: I stripped everything before the method name so I could easily `String#split` the line and take the first string. I also removed the `<init>` and `(native)` methods as these cannot be targetted by `-XX:CompileCommand` directly.

Delightfully, it produced a result:
```
Failure for java.util.concurrent.ConcurrentHashMap::get
```

So, something related to `ConcurrentHashMap::get` was causing the problem! At this point, it's **still unclear if there is a miscompilation** or the bug simply manifests in relation to this compiled method. But at least, I now knew a little bit more about where to look.

### Inlining

Remember when I said `-XX:CompileCommand=compileonly` ensures that only a specific method gets compiled? In reality, it also **includes inlined methods**[^18] if C2 decides they're worth inlining. The `-XX:+PrintInlining` flag will show what's being inlined. Warning: it's often so verbose that `make test` will truncate the output.

[^18]: [Inlining](https://en.wikipedia.org/wiki/Inline_expansion) is an optimization method that replaces function calls with their body. This saves function call overhead, but more importantly, transforms code in such a way that it may be eligible for other optimizations.

Taking a look at `-XX:CompileCommand=compileonly,java.util.concurrent.ConcurrentHashMap::get -XX:+PrintInlining`, it shows the following:
```
@ 1   java.lang.Object::hashCode (0 bytes)   (intrinsic, virtual)
@ 4   java.util.concurrent.ConcurrentHashMap::spread (10 bytes)   inline (hot)
@ 34   java.util.concurrent.ConcurrentHashMap::tabAt (21 bytes)   inline (hot)
  @ 14   jdk.internal.misc.Unsafe::getReferenceAcquire (7 bytes)   (intrinsic)
@ 67   java.util.Objects::equals (51 bytes)   force inline by annotation
  @ 0   jdk.internal.misc.PreviewFeatures::isEnabled (4 bytes)   inline (hot)
  @ 39   java.lang.Object::equals (11 bytes)   failed to inline: virtual call
@ 137   java.util.Objects::equals (51 bytes)   force inline by annotation
  @ 0   jdk.internal.misc.PreviewFeatures::isEnabled (4 bytes)   inline (hot)
  @ 39   java.lang.Object::equals (11 bytes)   failed to inline: virtual call
```

Now, I could disable each method one by one and see if any of them caused the bug to cease. For example disabling `ConcurrentHashMap::spread`: `-XX:CompileCommand=dontinline,java.util.concurrent.ConcurrentHashMap::spread`. The only exceptions are intrinsic methods, who remain unaffected by the flag and will be inlined.

When I disabled every inlined method and the error still persisted, it meant that there were two options. First, the bug could be in `ConcurrentHashMap::get` itself. Or, the bug is in an intrinsic. Taking a look at [the source code for `ConcurrentHashMap::get`](https://github.com/openjdk/valhalla/blob/a687d94e6afcb5326f558f196dcefe2dbbc72b10/src/java.base/share/classes/java/util/concurrent/ConcurrentHashMap.java#L950), it seemed unlikely that it would be there. The code uses very common programming constructs that would have most likely caused failures in other tests too.

My hunch was that the bug was in the `Object::hashCode` intrinsic, simply because Valhalla does things with `hashCode`. And indeed, turning off the intrinsic by [returning `false`](https://github.com/openjdk/valhalla/blob/a687d94e6afcb5326f558f196dcefe2dbbc72b10/src/hotspot/share/opto/library_call.cpp#L5300C36-L5300C44) in the function fixed the issue.

It seemed the issue was in the `Object::hashCode` intrinsic. However, at this point it was still not possible to definitely pinpoint that it was a miscompilation, but signs did point towards one. In any case, I was now at the point where the compilation was sufficiently small to bring out the big guns.

### Summary

By individually compiling methods that were C2-compiled during a run where the bug manifested, I found that `ConcurrentHashMap::get` was the suspect. Upon inspecting the inlining, the C2 intrinsic for `Object::hashCode` was determined to be the culprit.

## Act IV: The Bug

The intrinsic source code is quite long. My OOP professor would have said it "has a high cyclomatic complexity." I added a few print statements to each branch to figure out which path we were taking. Recall that I could not reproduce this bug via `gdb`, hence print statements were my only available tool.

While doing that, I noticed the [`if (!UseObjectMonitorTable)`](https://github.com/openjdk/valhalla/blob/a687d94e6afcb5326f558f196dcefe2dbbc72b10/src/hotspot/share/opto/library_call.cpp#L5365) conditional. Suddenly, I had a theory! The bug did not manifest with Compact Object Headers. Could it be that using Compact Object Headers sets this flag? I could easily verify this:
* `java -XX:+UseCompactObjectHeaders -XX:+PrintFlagsFinal -version | grep UseObjectMonitorTable` showed it was enabled.
* `java -XX:-UseCompactObjectHeaders -XX:+PrintFlagsFinal -version | grep UseObjectMonitorTable` showed it was disabled.

So, Compact Object Headers sets this flag and does *not* take the branch. **It's likely an issue related to something in that branch**. The branch adds a bunch of extra instructions, but nothing immediately stands out.

### C2 Compilation Graph

I took a look at the difference in generated machine code. [Configuring](https://openjdk.org/groups/build/doc/building.html) the HotSpot source code using the `--enable-hsdis-bundling --with-hsdis=llvm` flags[^19] enables machine code inspection. That let me run with the `-XX:CompileCommand=print,java.util.concurrent.ConcurrentHashMap::get` which prints the disassembled machine code of the whole method, including inlined methods and intrinsics. As I already saw from the source code, I saw extra instructions with Compact Object Headers / `UseObjectMonitorTable` disabled.

[^19]: This requires a LLVM toolchain, but it's also possible to achieve using `gcc`/`binutils`.

Instead, I had to think at a higher level. C2, like many compilers, optimizes on an intermediate representation (IR). Programs get transformed into an IR that is not Java bytecode, but not yet machine code. C2's IR is a big graph, called Sea of Nodes[^20]. It performs optimizations by doing transformation and analysis passes over the IR. Using the [Ideal Graph Visualizer](https://github.com/openjdk/jdk/tree/master/src/utils/IdealGraphVisualizer) I was able to take a diff between the graph of `ConcurrentHashMap::get` with `UseObjectMonitorTable` enbled and disabled, for the first and last optimization passes. For context, this method produced about 400 nodes and 1000+ edges, which is why I don't have a screenshot. The **diff yielded no interesting changes** unfortunately. While some nodes had differences, these could be attributed to different addresses or branch probabilities.

[^20]: A good academic introduction is in [Tianxing Wu's master thesis](https://urn.kb.se/resolve?urn=urn:nbn:se:kth:diva-368250).

**Was this not a miscompilation after all?** Was the actual issue deeper in the runtime and the bug just mainfested in this compilation unit? While I learned which feature flag was responsible, it still didn't explain what was going on.

### The Bitmask

I needed to understand what `UseObjectMonitorTable` actually does. Crudely put, an object monitor is used for locking and sychronization. The [wiki page](https://wiki.openjdk.org/display/HotSpot/Synchronization+Using+The+ObjectMonitorTable) based on the [pull request](https://github.com/openjdk/jdk/pull/20067) was a helpful starting point. To understand my bug only the first line of the PR is relevant (inflating is a state of the `markWord`):
> When inflating a monitor the ObjectMonitor* is written directly over the markWord and any overwritten data is displaced into a displaced markWord.

I asked my GC colleagues to see if they had an idea, and we collaborated on the last step of finding the bug. We determined how to attach to the process since the bug would only manifest through `make test`. It turns out, the following combination worked quite well:
* Running the test with `-XX:+ShowMessageBoxOnError` and `-XX:AbortVMOnException=java.lang.AssertionError` set.
* Waiting for the test to hang, most likely indicating a failure. `ShowMessageBoxOnError` expects input via standard input, but since the `jtreg` infrastructure wraps everything, input cannot be given. Hence, it hangs until the test times out.
* Using [`jps`](https://docs.oracle.com/en/java/javase/25/docs/specs/man/jps.html) to get a list of Java processes.
* Connecting the `lldb` debugger via `lldb -p <PID>` and check the threads using `thread backtrace all` to see which of the processes had the message box showing. This would be the failed process.

In the end though, attaching didn't help much. It did indicate that some threads were waiting, with a lightweight locking mode (what this means is irrelevant), which made us look at the [intrinsic](https://github.com/openjdk/valhalla/blob/a687d94e6afcb5326f558f196dcefe2dbbc72b10/src/hotspot/share/opto/library_call.cpp#L5365) again in detail. Specifically, the `LM_LIGHTWEIGHT`. Note that the misaligned comment is a faithful reproduction of the source code.
```cpp
  // Test the header to see if it is safe to read w.r.t. locking.
// This also serves as guard against inline types
  Node *lock_mask      = _gvn.MakeConX(markWord::inline_type_mask_in_place);
  Node *lmasked_header = _gvn.transform(new AndXNode(header, lock_mask));
  if (LockingMode == LM_LIGHTWEIGHT) {
    Node *monitor_val   = _gvn.MakeConX(markWord::monitor_value);
    Node *chk_monitor   = _gvn.transform(new CmpXNode(lmasked_header, monitor_val));
    Node *test_monitor  = _gvn.transform(new BoolNode(chk_monitor, BoolTest::eq));

    generate_slow_guard(test_monitor, slow_region);
  } else { /* ... */ }
```

Conceptually, this code *generates machine code* that does the following:
1. It defines a constant represented by the bitmask `markWord::inline_type_mask_in_place`. This bitmask is the logical OR of the value object bit and the two lock bits.
2. It performs a logical AND on the object header (where the `markWord` resides) and the locking mask to create a masked header.
3. It defines another constant representing the monitor value (`0b10`).
4. It compares the masked header with the monitor value. If it's equal, it takes a slow path needed for locking (the details of which are irrelevant).

**The bug is in the first step**. The following diagram represents the old and new bitmasks for the eleven least-significant bits:
```
    7   3  0
 VVVAAAAVSTT  <- Before my change
        ^-^^---- 1011 bitmask
 VVVVAAAASTT  <- After my change
    ^-----^^---- 10000011 bitmask
```

When an object monitor is inflating, the `markWord` is replaced by the object monitor's native address (sans least-significant lock bits). Let's denote this native address as `P..PP`. The `markWord` then becomes:
```
    7   3  0
 PPPPPPPPPTT  <- Before my change
        ^-^^---- 1011 bitmask
 PPPPPPPPPTT  <- After my change
    ^-----^^---- 10000011 bitmask
```

So we are doing a bitmask on a pointer instead of metadata! If we got unlucky and bit number 3 or 7 (depending on before/after my change) were set to 1, the mask would include an insignificant bit. In that case, comparing to `0b10` (the monitor value) would yield that it is not equal, and the slow path would not be taken, even though it should have been! We should just check the lock bits instead. **It's a miscompilation after all!**

**Setting the mask to `markWord::lock_mask_in_place`** seemed to address Symptom B (`java.lang.AssertionError`) and Symptom C (`java.lang.NoClassDefFoundError`). Recall that Symptom A (failed unit tests) had already been fixed. Finally, running it through the usual [tiered testing](https://openjdk.org/groups/build/doc/testing.html) showed no regressions compared to before my changes.

### Summary

An incorrect bitmask was checking native pointer bits instead of metadata bits when `UseObjectMonitorTable` was disabled. If a bit in the pointer was set to 1, the required slow path is not taken. Compact Object Headers was setting the `UseObjectMonitorTable` flag to `true`, which is why the issue did not manifest here.

## Act V: Post Mortem

One big question is why did this not affect the virtual machine before my change? This can be explained by the fact that 64-bit native memory pointers, obtained via `malloc`, are 16-byte[^21] aligned. They will have four trailing zeroes, of which *the latter two may be overriden by the locking metadata*. Considering the diagram:
```
    7   3  0
 PPPPPPP00??  <- Before my change
        ^-^^---- 1011 bitmask
 PPPPPPP00??  <- After my change
    ^-----^^---- 10000011 bitmask
```

[^21]: After publishing this post, I was [made aware](https://old.reddit.com/r/java/comments/1pa5xiu/help_my_java_object_vanished_and_the_gc_is_not_at/nrizt3h/) of the fact that [this is not a valid assumption for all platforms](https://github.com/openjdk/jdk/pull/28235).

It shows that the bit in position 3 was always zero. Therefore, the value object bit in the bit mask would AND with a zero, which produces a zero. That's why the bug only manifested after moving the value object bit.

I have no idea why this caused the symptoms that it did. A miscompilation means the program can behave arbitrarily, and that it did. While I acknowledge this might be an unsatisfying ending, it's simply not the best use of my time to chase the bug to its symptoms. I'm more than happy to guide anyone who is interested in doing so though.

### Summary

The bug did not manifest before I changed the value object bit location because we got lucky with native memory alignment. The bit that the faulty bitmask was checking was always zero. I have no idea why objects turned null or why classes could not be found.

## Closing Thoughts & Takeaways

The takeaways I would like to highlight are as follows:
- **Have a strong methodology and challenge assumptions.** With a large codebase like HotSpot, reasoning is the required to (sanely) make progress in debugging. Derive a methodology appropriate for the bug you are facing.
- **Ask the right questions at the right time.** Know what/when to ask and whom to ask. There's no shame in asking for help, and it's often a learning experience for both parties. If I spend a day without progress, I usually take that as an indicator to seek external help and/or input.
- **Use your tools!** This may be ironic since I wasn't able to utilize `lldb` successfully, but debugging tools are incredibly useful. At least get familiar with the basics, they're not as hard and scary as they seem.

I think I can attribute my success in this instance to asking the right questions to the right people. I'm not great at debugging, but a lot of people around me are and were glad to help. Thank you to all my colleagues for helping me out when I was stuck, and answering my many questions. I'm extremely grateful to be able to learn from you.

---

### Edited 2025-12-27

- Pointed out that `malloc` isn't guaranteed 16-byte aligned on all platforms.
- Various language fixes.
- Updated an `AAAAVSLL` to `AAAAVSTT` to be consistent with prior notation.
