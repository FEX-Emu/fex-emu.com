---
layout: post
title: The scourge of x86 emulation
author: FEX-Emu Maintainers
date: 2026-09-17
---

<link rel="stylesheet" href="{{ site.baseurl }}/assets/css/scourge.css">

Welcome to the first feature article on our site. We're going to cover an ongoing problem with x86 emulation that affects every application that we
emulate. This comes down to a single over-arching term that has wide-reaching ramifications; Emulating the [x86 Total Store Ordering memory model
(x86-TSO)](https://en.wikipedia.org/wiki/Processor_consistency#Similarity_to_SPARC_V8_TSO,_IBM-370,_and_x86-TSO_memory_models).

The problems with emulating this memory model on the [weak ordering memory model](https://en.wikipedia.org/wiki/Consistency_model#Weak_ordering)
that ARM defines is multi-faceted and covers multiple issues. We're going to go over all the problems that we can encounter and the ways we solve
(or in some cases can't solve) in this article. Get yourself a snack and a warm drink to enjoy, this is going to be a long one.

* Generate table of contents
{:toc}

# <ins>What exactly is x86-TSO?</ins>
Before diving in to how we work around the x86 memory model problem, we need to first discuss exactly what it is. A memory model is a set of rules
for how memory accesses in a system behave in relation to each other. The rules will dictate how loads and stores interact in a single-threaded or a
multi-threaded environment. There's a handful of popular memory memory models implemented in various forms of hardware, but the two we care about
today is ARM's relaxed (or weak) consistency model, and the x86 variant of Total-Store-Ordering consistency model. These two models are basically the
two extremes of the spectrum; where ARM is the most relaxed, allowing significant hardware optimizations; and x86 is the most strict, enforcing a very
strong coherency model that doesn't allow a lot of room for optimization. One thing to be careful about when discussing memory models is the
difference between consistency and atomicity. While these are related, they are not the same nor guaranteed in <span data-toggle="tooltip" title="We will come back to this"><span class="tooltip_text">all cases</span></span>.

The best way to explain how the differences in memory models work is to start with how x86 handles this. With TSO being very strict in how it
operates, the programmer can assume that when a memory store occurs, that this will be coherently visible to all other processors in the system.
This additionally means that when a memory load occurs, all stores before it "logically" will have been completed, or at least visible. This matches
programmer expectations, you write to memory, it becomes visible as at the point of writing, as this is intuitive to think about when programming. The
stores are effectively ordering the visibility of the loads, thus the name of the model. There's a bit of nuance with how this operates but isn't
strictly necessary to understand.

The weak memory model that ARM has is a bit less intuitive about how it operates. By default the regular memory loads and stores that ARM uses aren't
strictly coherent across processors in your system, allowing the CPU to operate more efficiently most of the time. When a store instruction
executes, that piece of memory (the cacheline) isn't immediately visible to other processors in the system. Saving on precious power and efficiency
because it's expensive in hardware to invalidate other core's cachelines, or allow them to <span data-toggle="tooltip" title="The act of reading another processors cachelines"><span class="tooltip_text">snoop</span></span>
another processor's caches. Relatedly if a processor is loading data from memory that another processor has written to, it's not guaranteed that this
load will even see this updated memory. This sounds like it would cause some significant problems in a multi-threaded application right? Older
versions of ARM (ARMv7 and older) used a memory barrier instruction to ensure ordering, which had significant performance implications.

To get around this limitation of consistency, ARM also introduced load-acquire, and store-release memory instructions. In C++ parlance this maps to
**std::atomic's** *memory_order_acquire* and *memory_order_release* definitions respectively. In ARM's terminology, these instructions also aren't
***technically*** considered to be atomic operations, but programmers conflate the two. FEX has used the terms atomic-load and atomic-store to mean
the same thing! The distinction _usually_ doesn't matter, but when discussing these topics it may be better to be pedantic about it.

The primary use case for these instructions is to force memory ordering between these class of instructions. ARM calls this the "Release Consistency
sequentially consistent (RCsc)" model. Without getting too far in to the weeds about how this model operates, the basic gist is that the load-acquire
instructions must be observed sequentially without reordering, and the store-release instructions must as well while fulfilling
"barrier-ordered-before" semantics. Removing the costly memory barrier instruction required in older ARM architecture versions.

# <ins>The humble beginnings of ARMv8.0-a</ins>
This is the premise of where we start in ARMv8.0-a when we're emulating the x86-TSO memory model. We make all x86 memory loads turn in to ARM's
**load-acquire** instructions, and x86 memory stores turn in to **store-release** instructions. This gives FEX effectively the same memory semantics
as x86, although we are actually being *more* strict than what is necessary. This is because we had no middle-ground which exactly matches behaviour.
As one might think, it is *exceedingly* costly to emulate TSO wth this instructions and we have microbenchmarks that can show this.
As ARM CPUs weren't designed to have these relatively rare acquire/release instructions suddenly become the vast majority of instructions executed.

First let's start with something easy and use a microbenchmark that is fairly nice to the hardware. No tricky edge-cases, just accessing memory in
the common case. This gives us some baseline numbers for what the best-case situation should be.

<div id="memory_access_times" style="min-width: 250px; height: 400px; margin: 0 auto"></div>

Let's break down this graph as it tells us a few interesting stories. The Load and Store columns of each machine is representing our baseline
performance number that our hardware should be attempting to achieve. These aren't trying to max out the memory bandwidth of each system, but do the
same amount of work for each type of operation.  If we turn our attention to the acquire-load results, we can see that out of the
five CPUs tests, three of them have their performance hindered quite a bit by using acquire-loads! Additionally we can see that the AmpereOne CPU has
release-store instructions that are strikingly low compared to the other results, and the M1 Acquire/LRCPC load instructions are quite a bit lower than the baseline as well.

The AmpereOne results in particular showcase how bad this legacy path can get. These instructions were never designed to be
used this way. Using acquire-release semantics for every load for x86 emulation actually imposes some really strict limitations on ARM CPUs in that
the load instructions can no longer be ordered around each other at all. So when you have millions of them in flight per second, the performance isn't
really expected to be good. But because these are the only instructions we had with ARMv8.0-a, it's what we had to use. While Cortex-X4 and
Cortex-X925 have amazing performance for these, you can see how the Oryon-3 has deprioritized their importance.

<h2 class="no_toc">Where do we go from here?</h2>

Let's take a closer look at the LRCPC-load instructions, which is mandatory since ARMv8.3. This extension adds a bunch of new load instructions to the ARM ISA and adds a new memory model on top of
ARM's *RCsc* model from before. This new "Release Consistency processor consistent (RCpc)" memory model is what we've been wanting! This extension is
designed around the requirements that x86 emulation requires, and is expected to get utilized heavily on hardware that implements it. As you can see from the
graph, almost all of the platforms have their LRCPC-loads matching their regular loads in <span data-toggle="tooltip" title="Except for Apple. we'll come back to that"><span class="tooltip_text">performance.</span></span>

With this new extension that is mandated by newer ARM versions, we basically get <span data-toggle="tooltip" title="Not quite. We'll come back to that"><span class="tooltip_text">**solved**</span></span>
memory performance. At least according to this microbenchmark that seems to be the case. Once FEX detects this extension we stop using Acquire-Load instructions
entirely and switch over to LRCPC-Load instead. But what's going on with that Apple M1 result..?

<div id="memory_access_times_apple" style="min-width: 250px; height: 300px; margin: 0 auto"></div>

This is where we need to commend Apple's path towards solving this problem. With their Apple Silicon processors they directly added support for the
x86-TSO memory model. When the CPU feature is toggled, their ***regular*** load/store ARM instructions change behaviour to match what x86 requires. They went
this route knowing that they will need a high performance solution for their hardware when switching to the ARM ecosystem exclusively.
This is why on their hardware the LRCPC-load instructions are actually aliases of their acquire-load instructions, because their x86
emulator doesn't even use these instructions! Because they implement the x86-memory model, they just use regular load/store instructions, which can be seen in our
microbench results as indiscernable performance overhead. To be fair to the other platforms, this thread-wide TSO mode toggle does have some
performance impact, we just don't see it here. When FEX detects this CPU feature from [Asahi Linux](https://asahilinux.org/) we will also enable this
and get the "free" performance improvement. A potential concern is that when jumping between x86 emulation and ARM code, that the ARM code
will pay unnecessary overhead due to all its accesses being TSO now. While this is a reasonable concern, the amount of ARM native code executing under
emulation approaches 0%. As a developer, you don't care about 1% of memory accesses becoming 10% slower, you care about 99% of accesses becoming 15% of
the "ideal" (As shown in AmpereOne results).

As a note, we think a TSO mode is the best path forward for ensuring high performance x86 emulation on the platform. Because this ensures that every memory
access instruction behaves how we want or expect. This is shown with the official **FEAT_LRCPC** extension actually having three versions that
apply bandages to the implementation each time.

* **FEAT_LRCPC** - Adds basic GPR TSO load instructions
* **FEAT_LRCPC2** - Adds small offset immediate to TSO load instructions
* **FEAT_LRCPC3** - Adds basic vector and stack-based TSO load & store instructions

Even with these three extensions, there is edge-case behaviour that can't be emulated as nicely as if we had a TSO hardware toggle.
We are expecting there to be additional extensions versions as time goes on, trying to fix some of the additional problems we'll discuss
later in the article.

# <ins>I thought accessing memory was the easy bit?</ins>
In the previous section, we were being nice to the ARM hardware and playing along with the underlying hardware's alignment requirements to get a
baseline for what the performance should look like. When emulating x86 although, we run face first in to a glaring problem right from the start. Your
favourite x86 applications don't care about alignment! They'll access memory however they please, crossing cacheline granularities, doing atomics that
aren't aligned. You think of the alignment problems, these games are doing it. This problem is so bad that we have a term associated with it, called <span data-toggle="tooltip" title="We'll come back to that"><span class="tooltip_text">**split-locks.**</span></span>
These are such a big deal that even the Linux kernel will capture when these occur and slow down games when they do it! Causing many gamers to tinker
with kernel options to avoid the slowdown!

But we aren't going to talk about full on split-locks yet, let's get started with just load-store instructions in an environment that doesn't care
about alignment. x86 makes certain guarantees to the programmer; if you do a load-store and it is inside of a <span data-toggle="tooltip"
title="Effectively 64-bytes mandated"><span class="tooltip_text">cacheline</span></span> then that load-store will be both <span data-toggle="tooltip"
title="The data won't tear"><span class="tooltip_text">atomic</span></span> and still match the coherency model as described before. However, to be a
little bit nice to the hardware developers, if the load-store *does* cross a cacheline, the data isn't atomic and other threads can and will see it
tear. So the programmer needs to be careful as a basic load-store is not a split-lock.

The problem with emulating these basic accesses with load-acquire/store-release is that ARMv8.0 requires what is known as **natural alignment**.
This means that for whatever size of data being accessed, the offset in memory must match the size. So for an 8-byte access, it must be at offsets; 0,
8, 16, 24, etc. This works well for native ARM applications, but what happens when we don't obey natural alignment requirements? For ARM, this means
the instruction with raise an <span data-toggle="tooltip" title="Or a SIGBUS for Linux people"><span class="tooltip_text">**alignment fault.**</span></span>
The hardware validates that the alignment requirements are fulfilled and if they are not then the CPU will fault. This usually results in a crash but
FEX does special handling.

Inside of FEX's JIT mechanism we keep track of memory load-store instructions that are emulating the
x86 load-stores. When we know that a load-store can cause an alignment fault we have what is known as a **patchpoint** in the code. For load-store
instructions, this shows up as a **NOP** instruction either before or after the load-store. When a alignment fault occurs as one of these patchpoints, FEX
will capture the fault, patch the code from a load-acquire/store-release instruction to a **basic** equivalent load-store, and wraps the instruction
in a data memory barrier. Then it continues executing!

<p align="middle">
<img style='object-fit:contain; height:100%; width:49%' src="{{site.baseurl}}/images/posts/scourge/before_patch.png">
<img style='object-fit:contain; height:100%; width:49%' src="{{site.baseurl}}/images/posts/scourge/after_patch.png">
<p style="text-align:center;font-weight:bold;">Before then after patching</p>
</p>

That entire discussion from before about how ARMv8.0-a added these new fancy load-acquire, store-release instructions? We immediately fall
back to the classic memory barrier instruction instead when alignment behaviour doesn't match. Our previous chart didn't show this bad case, so let's
bring in some fresh data.

<div id="unaligned_memory_access_times" style="min-width: 250px; height: 600px; margin: 0 auto"></div>

Oh, that's a lot of data to sift through. While again good to see how far away the hardware is from the "optimal" path while emulating TSO, it's not what we care about here.
It is interesting to note that this microbench doesn't showcase much of a difference between aligned and unaligned for regular load/stores so we just
calculated an average between the two.
We'll be removing the x86 CPU and the regular load-store data from the ARM columns, as these aren't the common FEX paths. This way we'll have a more
targeted view about how badly unaligned memory accesses hurt under emulation.

<div id="unaligned_memory_access_times_reduced" style="min-width: 250px; height: 600px; margin: 0 auto"></div>

Now that we have a much more reasonable graph of data, let's walk from left to right on this and discuss what is going on.

<h2 class="no_toc">AmpereOne</h2>
This one is pretty interesting, both the aligned and unaligned load instructions are roughly equivalent and fall within noise. This means that even
though the unaligned loads are getting hit with a <span data-toggle="tooltip" title="DMB"><span class="tooltip_text">data memory barrier</span></span> penalty,
the CPU just handles it. This might be the case that the benchmark is bottlenecked by other things, considering how much lower the performance is
compared to other platforms.

Meanwhile the store side is not looking to be in a good shape even without unaligned. It nearly isn't visible on the chart! When hitting
unaligned stores we're looking at ~8.5% of a performance hit, but because we are already starting so low it is hard to notice. This is also in stark
contrast to regular store instructions getting ~28GB/s in this bench.

The only conclusion we can come to here is that Ampere is optimizing for some server class workload and doesn't really match consumer hardware
behaviour. It's an interesting datapoint, but our users aren't typically running games on this class of hardware.

<h2 class="no_toc">Cortex-X4</h2>
This is a highly popular CPU core that is living inside the [Qualcomm Snapdragon 8 Gen 3](https://en.wikipedia.org/wiki/Steam_Frame). We only tested
this one core from the SoC to not overwhelm the chart with data. Quite a large number of handhelds ship with this so it's an interesting
target. This CPU actually does **surprisingly** well considering it's the only cellphone SoC on this list. Overall this core kind of falls in line
with what we would expect from it and the graph trends follow with the next-generation Cortex in that chart.

The main topics for this CPU are that its aligned loads and stores are reasonably powerful, getting around 11.5GB/s and
6.7GB/s respectively. What's interesting is the performance falloff when it needs to deal with unaligned loadstores, hitting the **DMB** instructions
penalizes the core roughly evenly between loads and stores at around 50% in this benchmark.

This seems to imply that the CPU can keep a decent number of LRCPC-release loadstores in flight so the **DMB** instructions hurt more when they are
encountered, but it isn't causing world-ending performance. Just that a 50% performance hit due to alignment isn't an amazing result.

<h2 class="no_toc">Cortex-X925</h2>
Following up the X4, let's stop by the [DGX Spark](https://www.nvidia.com/en-us/products/workstations/dgx-spark/) and its X925 cores. Not only is this
a newer CPU core from ARM, it's running on a system with dramatically more memory bandwidth. 273GB/s in the platform versus the previous 76.8GB/s. This
means that we get fairly similar results to the X4 even, just the graph scales a little higher. Interestingly enough, the performance penalty for
unaligned accesses roughly match the X4 even. Although it looks like the stores can recover a little faster, likely due to the faster memory helping
out. No surprises here, just consistently matching performance across the generations.

<h2 class="no_toc">Oryon-3</h2>
This CPU core design is hot off the presses from Qualcomm. Linux support is still in the process of coming up but it already has a strong showing.
The most interesting result from this actually comes from the fact that aligned LRCPC-load instructions are matching the
performance of regular loads! That means in the case of a well-behaved application we can typically expect full performance. This continues onward to
the release-store instructions being quite capable, although it doesn't quite match regular stores with only 68% of the bandwidth. Not a bad showing
in the slightest.

This CPU also can't escape from the penalty of unaligned LRCPC-release loadstores. The load side is roughly matching the ~70% performance penalty of
the Cortex-X925, likely because the Snapdragon X2 Elite also has tons of bandwidth. But the store side actually gets off a little worse at ~43% of the
performance. Even with these performance hits of unaligned accesses, this platform is actually faster than the aligned accesses from the Cortex
offerings.

One of the weird things about this platform is that it was advertised to have "Fully coherent 96KB 6-way L1 cache with 64B coherency granules." Which
to our reading implied that unaligned accesses should have dramatically less of a performance impact. Interesting... keep that in mind.

<h2 class="no_toc">Apple M1</h2>
This is the big one we need to talk about. This is the one that was a game changer, it was the "Apple moment." It showed everyone that ARM was
not only feasible, it could be faster. These numbers on this chart are amazing and it's the result of Apple sticking the TSO memory
model directly in to their hardware. Instead of using LRCPC-release accesses for this one, we just enabled their TSO feature and the aligned
versions basically match the unaligned version. Maybe a 5% performance hit on the stores? Compared to every other device on that chart, it's
effectively nothing. This primarily comes down to unaligned accesses no longer requiring **DMB** instructions to be backpatched in to the code, as the
hardware just handles it directly.

For us, this is what it means to take x86 emulation seriously on ARM and it really shows that Apple cared that their customers would have a good
experience running software both natively and emulated. They saw the problem and just <span data-toggle="tooltip" title="Not entirely. More on that later"><span class="tooltip_text">"solved"</span></span> it, making it go away.
That said, when the TSO mode _is_ enabled, you do get a performance hit. Comparing to the previous graph it's only getting 76% of the regular
store performance, and the load performance basically matches; that's much more tolerable to bear when everything is so much faster.

<h2 class="no_toc">Wrapping up unaligned LRCPC/release accesses</h2>
Wrapping up this section, we need to talk about one of the performance improvements that all of these vendors actually support. This is an
extension that ARM whipped up called **FEAT_LSE2** which all of these tested platforms implement. We previously talked about how acquire/LRCPC/release
memory accesses require natural alignment in order to not incur the wrath of the CPU raising alignment faults. ARM actually thought about
this problem and implemented this extension which helps x86 emulation (and probably other workloads). This extension loosens the alignment
requirements of not only acquire/LRCPC/release load store instructions, it _also_ loosens the requirement for read-modify-write atomics!

That sounds all well and good, but here's the kick to the teeth: that means it only provides marginal performance gains for x86 emulation. This
extension only loosens the alignment requirements to allow unaligned memory accesses inside of a 16-byte granule. Any access that crosses that 16-byte
granule still receives an alignment fault. x86 applications don't really care about the alignment of their memory accesses, so we get
unaligned accesses across the entire cacheline. It's only read-modify-write atomics that **try** to avoid crossing a cacheline on x86!

So thanks for the attempt, it's nice to see, but it doesn't really move the needle. Since we're already talking about it, let's dive in to those RMW
atomics shall we?

# <ins>Oh no, what are these atomic instructions?</ins>
Like most modern instruction sets, x86 supports atomic memory operations. These are instructions that execute an ALU operation on data in memory
atomically, allowing no intermediate state to be visible. In x86 terms this operates on memory that is both atomic and coherent, while ARM lets you
choose to be only atomic *or* both atomic and coherent. We touched on this briefly before but there is actually a difference between operating on data
atomically, and coherency of that data. What difference does it make?

For all of the previous x86 memory model discussion we have been talking about the coherency implications of loads and stores being visible to other
processors in the system. What we entirely glossed over is the atomicity requirements of these memory accesses. In the world of x86 a load or store
_usually_ completes atomically even when unaligned. This means that if you're storing 8-bytes of data, and another thread is loading those 8-bytes in
a race condition it will never suddenly see a mix of the data from before the store and after the store. In ARM these atomicity guarantees are
**significantly** weaker, meaning if you do an unaligned store instruction the specification of the ISA has zero guarantees about reading a **tear** in
the data. Thankfully for **naturally** aligned load-store instructions, ARM has a specification called **"single-copy atomicity"** which guarantees
you don't get a tear for these accesses. Also good news; that **FEAT_LSE2** extension from before? It actually extends the
single-copy atomicity guarantees to _any_ unaligned access inside of a 16-byte granule! The downside is that x86 has single-copy atomicity
guarantees across a full cacheline, so once again the extension still didn't solve anything completely, just reduced the number of occurences.

Enough about the differences in atomicity and coherency. Where's the actual atomic instructions? What do they do? Starting in ARMv8.1-a, our ISA
has gained instructions that mostly matches x86 atomic instructions in behaviour. Let's just give the full list to show how they map directly in our
JIT.

| **x86** | **ARMv8.1-a** |
| - | - |
| LOCK DEC | ldaddal |
| LOCK INC | ldaddal |
| LOCK NEG | [???](https://github.com/FEX-Emu/FEX/blob/e02953dc174531551219712df20355dcf4afc089/unittests/InstructionCountCI/FlagM/Atomics.json#L1368-L1381) |
| LOCK NOT | ldeoral |
| LOCK ADC | ldaddal |
| LOCK ADD | ldaddal |
| LOCK AND | ldclral |
| LOCK OR | ldsetal |
| LOCK SBB | ldaddal |
| LOCK SUB | ldaddal |
| LOCK XADD | ldaddal |
| LOCK XOR | ldeoral |
| LOCK BTC | ldclralb |
| LOCK BTR | ldeoralb |
| LOCK BTS | ldsetalb |
| XCHG | swpal |
| LOCK CMPXCHG | casal |
| CMPXCHG8B | caspal |
| CMPXCHG16B | caspal |

Well would you look at that, we have a full list of the 19 atomic RMW operations and they basically map directly to some ARM instructions. Ignore the questionable
one as it's not used in real workloads and we would get far too in to the weeds talking about it. We have a pretty clear 1:1 mapping between the
architectures, job's done right? That's the funny thing about x86 emulation, just because we have these instructions doesn't mean we get to wire them
up without problems. We spent all this time talking about how unaligned accesses can really hurt performance of regular loads and stores, this same
problem also applies to RMW atomics!

<div id="unaligned_lock_add_latency" style="min-width: 250px; height: 400px; margin: 0 auto"></div>

With this graph, we are looking at a single atomic instruction with its memory address landing somewhere within a cacheline. If we included all of the
data for all 19 atomic operations then this data would be even more overwhelming than it already is. All these atomic operations behave _roughly_
equivalent so it would be redundant and wouldn't matter for what we're discussing here anyway. This is also the first graph in this post that is
actually using logarithmic scaling, so when reading it make sure to understand that the performance difference from the fastest to slowest result is
on the scale of around 1000x.

Starting with the x86 Zen processor on this graph; these are the results that our emulation should be striving to achieve. As we can see, if the access is
fully contained within a cacheline then the latency of the instruction is the same at 1.44ns. This can be explained by x86 having "atomic cachelines"
or "coherent cachelines", where as long as an unaligned atomic operation stays within a cacheline then it roughly costs the same. This is a really
powerful feature of x86 that has been supported for decades at this point so games end up relying on this heavily without even realizing it. The
stand-out result for x86 is the final result that is crossing a 64-byte granule and taking ~660ns! That's an amazingly slow result at ~458x slower
compared to the other results because this is finally the hardware using **split-locks**.

We need to take a moment here to shout out an article that [Chips and Cheese](https://chipsandcheese.com/p/investigating-split-locks-on-x86){:target='_blank'} wrote
while we were preparing to write our article. They do a great deep dive in to why these **split-locks** are so dramatically slower and is worth the
read if you're unaware of how they work. Specifically we need to mention that x86 split-locks maintain the atomicity and coherency requirements of
x86-TSO and will ***never*** tear the data even when crossing a cacheline. This is kind of nuts and we'll explain this more later.

Now for our ARM processors, let's start with the natural alignment latency numbers. As we can see, all of our platforms perform fairly well but even
the latest cores don't get anywhere near x86. Even our fastest ARM platform is ~3x the latency compared to x86; This directly impacts performance of
games but usually isn't the direct bottleneck so it's hard to measure exactly how much. Continuing onward to the next data point, we can actually
combine the results for 16-byte granule and 64-byte granule crossing with most of our ARM platforms. Due to how the ARM specification defines how
unaligned atomics work, both of these results are roughly equivalent and FEX treats them the same as the x86 **split-lock** problem.

We keep bringing up this split-lock problem but how exactly does FEX emulate them and what makes it so slow? "I thought Apple M1 added x86-TSO support
in the hardware, why is it still slow?" If you recall how we brought up before that **FEAT_LSE2** introduced support for unaligned memory accesses within
a 16-byte granule; these split-lock operations end up hitting the same alignment problems as before but are dramatically slower. FEX
can't backpatch any of these instructions to just do a **DMB** operation, so we cause an **alignment-fault** every time one gets executed. This means that
we do a kernel -> userspace signal handler -> kernel -> original code dance. _every—single—time_ one of this split-lock operations execute.
Jumping between kernel-space and userspace is slow on every platform and when you're executing thousands of these per second it adds up very quickly.
This is why the emulation of these feature is so terribly slow on ARM.

One ARM platform today actually partially resolved this problem although. The Oryon-3 CPU cores introduced what they advertised as "coherent
cachelines" and we can see this in our microbenchmark results here. Just like with x86, if the atomic memory access in anywhere inside of the 64-byte
cacheline, the performance matches the natural alignment version! This is a tremendous improvement that means the CPU is on par with x86 in
feature support until the point it tries to cross a cacheline. We need to applaud Qualcomm on implementing this feature, as it resolves a major
performance and correctness problem around split-locks for x86 emulation. The hardware still doesn't support 64-byte **split-locks** so we still fall
down the FEX emulated path in that instance although.

Continuing on to the Apple result; even though they added x86-TSO memory accesses to their hardware for some reason they neglected to implement full
cacheline unaligned atomics like Oryon did. It seems like they should have expected this edge case to surface and implement it but that's just speculation.
This is why you can see the cross 16-byte granule behaving the same as other platforms even with the TSO hardware toggle enabled.

You might have also noticed another little data quirk in the graph. We have an asterisk on the Cortex-X4 result in this benchmark and the performance
of the unaligned atomics are dramatically faster than significantly newer CPUs. It is somehow managing to have only
~209ns latency, while the X925 is latency is 1060ns; that's a 5x perf improvement! How can this possibly be the case? This is actually some fun
"special sauce" that is shipping on the platform we're testing on, which is of course the **[Valve Steam
Frame](https://store.steampowered.com/sale/steamframe)**. Because Valve cares about the performance of their existing gaming catalogue, they are shipping a
[kernel patch](https://github.com/bylaws/linux/commit/7ae989a43ae7e3cb8007ac21c28dacc24c9d8320) that one of the FEX developers whipped up. This allows
the Linux kernel itself to handle the unaligned atomic without that slow dance with FEX and userspace, allowing it to be dramatically faster. If other
platforms want to ship this patch in the kernel then we recommend picking it up as and FEX will automatically start using it.

Speaking of kernel intervention, we need to talk about how split-lock emulation is not actually quite correct under FEX due to limitations in the
hardware. In order to implement this mandatory feature of x86 correctly, any time we do a 16-byte or 64-byte split-lock, the only way to
handle it is to have the kernel implement the feature. Right now FEX implements this as a "best-effort" attempt that can actually tear the data in
some cases. You'll recall that before we said split-locks on x86 will never tear right? Not even the Oryon-3 with its "coherent cachelines" have resolved
this problem yet.

# <ins>What do you mean split-lock is mandatory?</ins>
Implementing split-lock emulation with today's ARM hardware in a performant matter is actually really difficult to do. A naive implementation is to
use a global mutex and whenever a split-lock occurs we will ensure to acquire the mutex before doing the operation. This means that any
_participating_ split-lock operation will funnel through this mutex. This is correct except for the issue that any aligned atomic operation
isn't a split-lock and won't participate. Due to the split-lock emulation code needed to be implemented as two 64-bit compare-exchange
operations with each half straddling the granularity boundary, we can get a tear with a non-participating atomic still. A trivial example is one
thread constantly modifying an atomic in the middle of the cacheline, and then another thread modifying _only_ the integer on one half. This might sound
like a contrived example initially, but there are lock-less [linked-list](https://en.wikipedia.org/wiki/Linked_list) implementations that behave exactly like this!
Depending on which half the aligned thread is modifying, either the first or second CAS in the split-lock code will fail. If the first CAS fails, then
that's safe and the code can retry, if the *second* CAS fails that means the data has torn and we can do nothing but hope it doesn't corrupt data and
crash. This will entirely depend on the algorithm that the guest application is using so we don't control it.

An alternative approach that is completely untenable is to have the kernel track all processes and threads that are sharing memory with each other,
then when a thread needs to emulate a split-lock the kernel can halt ***every*** process that is sharing memory with that process, do the split-lock
in isolation, and then restart the world. The performance implications of this approach aren't viable. Applications and games can end up doing thousands or more
split-locks per second and halting the world will have an intractable performance hit that is dramatically worse than even x86 native.

If we want to ensure correctness in the emulation of split-locks FEX needs to have hardware support in some form to support these. Although we're not
saying that all atomic operations should now support split-locks like x86, that would also not be viable. The good news is that ARM actually has an
extension for this that does exactly what we want. ARM has an extension call **[Transactional Memory
Extension](https://en.wikipedia.org/wiki/Transactional_memory)** that could solve our problem. This extension allows our code to do some number of
operations inside of a transactional region, then commit that work atomically; if the commit operation fails, then we can simply retry. The downside
of this extension? ARM has officially deprecated the extension and no one ever shipped it. This is likely for the best as the x86 version of the
extension has had an abundance of problems that caused it to be disabled on many platforms.

So we need something else to emulate split-locks correctly. For a solution that we believe works for both FEX needs and ARM vendor needs, we have come
up with the idea that a 128-bit **CASP** instruction can be given the ability to have each half of the CASP perfectly straddle
the atomic granule boundary, 64-bits on the lower half, and 64-bits on the upper half. Then _only_ in that case does the instruction not raise an
alignment-fault and tries to do the CAS operation. This works because x86 only has up to 64-bit unaligned atomic operations, so both halves of the
operation can always be fully enclosed by our single operation.

But you may be asking yourself, "how is this any better than the hardware just supporting split-locks?" That's a good thought and we need to be
careful with the how exactly we describe this operation. For x86 their atomic operations must _always_ succeed without tear. For our emulated
approach, we can have this ARM **CASP** instruction fail safely and then we can try again. This is one of the benefits of CAS is that
the operation can fail for _any_ reason and it must be tried again. The instruction then also returns the data that it loaded from memory in that time
so the program has the latest up to date memory. This is an important distinction since that means FEX can retry the **CAS** operations infinite times
until it inevitably succeeds! This is a benefit of ARM LL/SC architecture that basically allows this to work. A tricky thing is that the hardware does
need to guarantee forward progress at *some* point but it already has support for that for other reasons so it's completely viable! The only newly
added failure mode to the *CAS* instruction is purely if one of the two cachelines got acquired by another core before it could do the full operation.
Even if the hardware still requires up to a couple thousand cycles to guarantee forward progress, that basically matches x86 behaviour.

We think this would be the best way forward for x86 emulation of split-locks on ARM platforms, but we're not hardware architects so all we can do is
complain and hope someone solves it for us. We'll leave the split-lock discussion there for now so we can move on to another interesting problem.

# <ins>Wait, uncached memory needs to work?</ins>

Before we get in to this topic we need to talk about the term "uncached" because it can mean a couple of things depending on your view of the
world. For the purposes of this article, we are using Vulkan terminology because we care about games primarily. In Vulkan terms we have
*VK_MEMORY_HOST_CACHED_BIT* which means that the host CPU caches this memory. The lack of this bit is what we care about here, and what we refer to as
"uncached." As for what this means to the memory subsystem, it gets a little more complicated than you would think. In particular when the memory is
living on a GPU, potentially over PCIe, when the memory is "uncached" it will also typically (but not always!) also gain the flag
*VK_MEMORY_HOST_COHERENT*. This means that because of the uncacheable property of the memory, the CPU and GPU always have a coherent world memory view
with each other.

For the CPU this typically means the memory can be mapped up to three ways. When asking for "cached" memory, this typically has a memory type of
[Write-back](https://en.wikipedia.org/wiki/Cache_%28computing%29#WRITE-BACK) which is also what regular memory mapping types are. "uncached" mapping
can be either [Write-Combine](https://en.wikipedia.org/wiki/Uncacheable_speculative_write_combining) or "Strong Uncacheable". The "Strong Uncacheable"
implementation is basically non-existant for userspace applications so we can ignore that for today's discussion. This limits us to effectively **WB**
(cached) and **WC** (uncached) memory types. Cached is what games typically use for staging buffers, and then uncached is what we use when passing data directly
to the GPU.

This is code-ified in many game engines that if you don't expose support for uncached buffer types then some don't work. This comes down to a
behaviour detail around the differences of [UMA](https://en.wikipedia.org/wiki/Unified_memory_architecture) systems like APUs and PCIe GPUs. UMA
systems will typically expose the ability to allocate memory that is cached, coherent, and GPU visible. Where PCIe GPUs can't guarantee that behaviour
so game developers need to either use a staging buffer and an async copy of the data over to the GPU, or use "uncached" memory to very carefully
shuffle the data over to the GPU through PCIe. Because of how ubiquitous PCIe is with PC gaming, some engines won't even do UMA specific code
paths and will do the uncached approach regardless!

With that little introduction out of the way for what uncached means for us. Let's bring up a benchmark for how fast cached memory is on some UMA
Snapdragon systems. This will let us get a baseline for how the performance should be regularly.

<div id="cached_memory_bench" style="min-width: 250px; height: 400px; margin: 0 auto"></div>

For both the **Steam Frame** and **Snapdragon X2 Elite** these are some really good results. As we would expect, the Oryon-3 platform has more memory
bandwidth so it is able to scale higher in the chart, but both are hitting dozens of gigabytes per second in their results. This graph sets a good
baseline for what "normal" **write-back** memory can achieve. Let's now show uncached results to see the performance differences.

<div id="system_uncached_access" style="min-width: 250px; height: 400px; margin: 0 auto"></div>

There's some strange things happening here so we had to use logarithmic again on this graph. Let's talk about the good first that has shown up.
Due to uncached memory buffers being write-combine, we can see that the regular stores for our ARM platforms match the cached benchmark
results. This comes down to write-combine memory using what is coined as [write combine buffers](https://en.wikipedia.org/wiki/Write_combine_buffer)
that actually _very_ temporarily keep around a cacheline of data so that write-combine can burst a cacheline of memory at a time. Interestingly enough
it looks like the Zen 4's WCB can't quite keep up with cached, but considering this is expected to be going over a PCIe bus it's probably fine.

Now let's get in to the really ugly results that we have here. Starting off with the easier to explain is the load bandwidth from write-combined
memory is abysmal on all platforms tested. If we're using Zen as our baseline for performance, then our regular load instructions are ARM are winning,
but the LRCPC loads are worse. What's going on here? This is a quirk of how write-combined memory operates, because it is uncached our load
instructions are required to go out to system memory for every single access to maintain semantics. Then when we add LRCPC-loads on top of that, it
just compounds the problem even further. But the worst case out of all of this is just how badly the store performance is, compared
to the performance that Zen gets on the stores, this is basically a showstopper. Up to **816x worse** bandwidth! We had games like [Hollow
Knight: Silksong](https://store.steampowered.com/app/1030300/Hollow_Knight_Silksong/) and [Subnautica
2](https://store.steampowered.com/app/1962700/Subnautica_2/) run at less than 1FPS because of this performance cliff.

As we were saying above, when there are PCIe GPUs in the mix then games will need to use uncached memory to pass data to the GPU. When emulating x86
games on platforms with a dedicated PCIe GPU then we are in an unwinnable situation and we are guaranteed to run dramatically slower. Remember how ARM
has added the family of **FEAT_LRCPC1/2/3** extensions from before to improve x86 memory model emulation? This is what happens when we hit an
edge-case that isn't supported. All of these extensions add new instructions to handle loading memory using x86-TSO memory model semantics but none of
them solve storing to write-combine memory with x86-TSO semantics. All the way from ARMv8.0-a our store instructions use the regular `store-release`
instructions regardless of the backing memory type. The only way for FEX to work around this problem is to selectively disable TSO-emulation when it
becomes an issue, so x86 emulation platforms with PCIe GPUs will always be a worse experience than UMA. At least until we get another **FEAT_LRCPC4**
or similar to resolve the issue.

For users on UMA systems then rejoice, there's a workaround for gaming that we use to improve performance. Because we know when a platform supports
cache-coherent CPU and GPU combinations, we can have the video driver ***always*** use cached buffers and never encounter this problem.
NVIDIA already does this on their Tegra platforms, Snapdragon has been supporting this since at least Adreno 600 class GPUs, and there are many
Mali platforms where this is also the case. We have a [Adreno Turnip](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/41323) patch that
ensures when FEX is running, we never hit uncached memory for platforms that support it. A funny thing is that since Asahi users have a hardware TSO
bit, they just naturally don't encounter this problem in the wild, but getting a PCIe GPU on to that platform is a different story altogether. There's
also a fun quirk where Radeon GPUs on ARM platforms hide all write-combine memory to instead be write-back but we'll talk about that another time.

# <ins>Looking towards a brighter future</ins>

After that marathon of an article we hope you have a better understanding of some of the challenges that emulating the x86-TSO memory model brings.
Where we started with ARMv8.0 as a minimum spec and where the hardware has provided dramatic improvements over the years in nothing short of
astounding. While not all of the edge-cases are yet resolved at the architecture level, it looks like there is a genuine commitment across the
ecosystem for trying to improve the worst cases. We have various vendors solving some parts of the problem and moving the needle forward for better
compatibility. Maybe in another decade as we look back at this time we'll laugh about the problems we were encountering now, while enjoying some quality
x86 games that will never see a port to ARM hardware. Keeping the legacy of the PC gaming ecosystem alive, regardless of where we might end up playing
it.

---

<script src="https://ajax.googleapis.com/ajax/libs/jquery/1.8.2/jquery.min.js">
</script>
<script src="https://code.highcharts.com/highcharts.js">
</script>
<script src="https://code.highcharts.com/modules/exporting.js">
</script>

<script type="text/javascript">
Highcharts.chart('memory_access_times', {
    chart: {
        type: 'column'
    },
    title: {
        text: 'Memory access bandwidth'
    },
    subtitle: {
        text: 'Higher is better (Zen truncated)'
    },
    xAxis: {
        categories: ['AmpereOne', 'Cortex-X4', 'Oryon-3', 'Cortex-X925', 'Apple M1', 'Zen 4'],
        crosshair: true,
    },
    yAxis: {
        min: 0,
        max: 50000,
        title: {
            text: 'GB/s'
        },
        labels: {
            formatter: function () {
               return this.value / 1000 + 'GB/s';
           }
        }
    },
    plotOptions: {
        column: {
            dataLabels: {
                enabled: true
            }
        }
    },
    tooltip: {
        valueSuffix: ' MB/s'
    },
    plotOptions: {
        column: {
            pointPadding: 0.2,
            borderWidth: 0
        }
    },
    series: [
        {
            name: 'Load',
            data: [22006.65, 21335.06, 34584.97, 28180.54, 22999.53, 'N/A']
        },
        {
            name: 'Store',
            data: [33116.88, 21386.46, 35442.95, 29667.51, 22516.16, 'N/A']
        },
        {
            name: 'Acquire-Load',
            data: [3460.26, 18137.21, 5049.66, 29538.66, 13768.21, 'N/A']
        },
        {
            name: 'LRCPC-Load',
            data: [22902.43, 23122.65, 35438.60, 29665.82, 14042.76, 38637.99]
        },
        {
            name: 'Release-Store',
            data: [855.89, 21360.26, 35452.88, 29658.30, 22405.70, 76278.19]
        }
    ]
});

Highcharts.chart('memory_access_times_apple', {
    chart: {
        type: 'column'
    },
    title: {
        text: 'Apple M1 memory access bandwidth'
    },
    subtitle: {
        text: 'Higher is better'
    },
    xAxis: {
        categories: ['Load', 'Store', 'Acquire-Load', 'LRCPC-Load', 'Release-Store'],
        crosshair: true,
    },
    yAxis: {
        min: 0,
        title: {
            text: 'GB/s'
        },
        labels: {
            formatter: function () {
               return this.value / 1000 + 'GB/s';
           }
        }
    },
    tooltip: {
        valueSuffix: ' MB/s'
    },
    plotOptions: {
        column: {
            pointPadding: 0.2,
            borderWidth: 0
        }
    },
    series: [
        {
            name: 'TSO mode disabled',
            data: [22999.53, 22516.16, 13768.21, 14042.76, 22405.70]
        },
        {
            name: 'TSO mode enabled',
            data: [23071.27, 22511.66, 13754.98, 14009.63, 22411.91]
        }
    ]
});

Highcharts.chart('unaligned_memory_access_times', {
    chart: {
        type: 'column'
    },
    title: {
        text: 'Memory access bandwidth (x86-bench)'
    },
    subtitle: {
        text: 'Higher is better'
    },
    xAxis: {
        categories: ['AmpereOne', 'Cortex-X4', 'Oryon-3', 'Cortex-X925', 'Apple M1', 'Zen 4'],
        crosshair: true,
    },
    yAxis: {
        min: 0,
        max: 60000,
        title: {
            text: 'GB/s'
        },
        labels: {
            formatter: function () {
               return this.value / 1000 + 'GB/s';
           }
        }
    },
    plotOptions: {
        column: {
            dataLabels: {
                enabled: true
            }
        }
    },
    tooltip: {
        valueSuffix: ' MB/s'
    },
    plotOptions: {
        column: {
            pointPadding: 0.2,
            borderWidth: 0
        }
    },
    series: [
        {
            name: 'LRCPC/TSO-acquire - Aligned',
            data: [5488.64, 11540.48, 29911.04, 14182.4, 22656, 35952.64]
        },
        {
            name: 'LRCPC/TSO-acquire - Unaligned',
            data: [5616.64, 5785.6, 21432.32, 6819.84, 22650.88, 35921.92]
        },
        {
            name: 'Release/TSO-stores - Aligned',
            data: [307.2, 6717.44, 29675.52, 5985.28, 33315.84, 36423.68]
        },
        {
            name: 'Release/TSO-stores - Unaligned',
            data: [281.6, 3312.64, 12712.96, 3993.6, 31662.08, 36633.6]
        },
        {
            name: 'Loads - Aligned/Unaligned',
            data: [5396.48, 45224.96, 29957.12, 26721.28, 22661.12, 'N/A']
        },
        {
            name: 'Stores - Aligned/Unaligned',
            data: [27944.96, 43596.8, 43320.32, 54804.48, 43811.84, 'N/A']
        },

    ]
});

Highcharts.chart('unaligned_memory_access_times_reduced', {
    chart: {
        type: 'column'
    },
    title: {
        text: 'Memory access bandwidth (x86-bench)'
    },
    subtitle: {
        text: 'Higher is better (M1 in TSO mode)'
    },
    xAxis: {
        categories: ['AmpereOne', 'Cortex-X4', 'Oryon-3', 'Cortex-X925', 'Apple M1'],
        crosshair: true,
    },
    yAxis: {
        min: 0,
        max: 40000,
        title: {
            text: 'GB/s'
        },
        labels: {
            formatter: function () {
               return this.value / 1000 + 'GB/s';
           }
        }
    },
    plotOptions: {
        column: {
            dataLabels: {
                enabled: true
            }
        }
    },
    tooltip: {
        valueSuffix: ' MB/s'
    },
    plotOptions: {
        column: {
            pointPadding: 0.2,
            borderWidth: 0
        }
    },
    series: [
        {
            name: 'LRCPC/TSO-acquire - Aligned',
            data: [5488.64, 11540.48, 29911.04, 14182.4, 22656]
        },
        {
            name: 'LRCPC/TSO-acquire - Unaligned',
            data: [5616.64, 5785.6, 21432.32, 6819.84, 22650.88]
        },
        {
            name: 'Release/TSO-stores - Aligned',
            data: [307.2, 6717.44, 29675.52, 5985.28, 33315.84]
        },
        {
            name: 'Release/TSO-stores - Unaligned',
            data: [281.6, 3312.64, 12712.96, 3993.6, 31662.08]
        }
    ]
});

Highcharts.chart('unaligned_lock_add_latency', {
    chart: {
        type: 'column'
    },
    title: {
        text: '64-bit atomic add latency by alignment within a cacheline (x86-bench)'
    },
    subtitle: {
        text: 'Lower is better (logarithmic)'
    },
    xAxis: {
        categories: ['AmpereOne', 'Cortex-X4\*', 'Oryon-3', 'Cortex-X925', 'Apple M1', 'Zen 4'],
        crosshair: true,
    },
    yAxis: {
        type: 'logarithmic',
        title: {
            text: 'ns'
        }
    },
    plotOptions: {
        column: {
            dataLabels: {
                enabled: true
            }
        }
    },
    tooltip: {
        valueSuffix: ' ns'
    },
    plotOptions: {
        column: {
            pointPadding: 0.2,
            borderWidth: 0
        }
    },
    series: [
        {
            name: 'Natural alignment',
            data: [6.59, 5.61, 11.89, 4.27, 9.06, 1.44]
        },
        {
            name: 'Unaligned crossing 16-byte granule',
            data: [1463.18, 209.35, 11.71, 1045.64, 1483.78, 1.44]
        },
        {
            name: 'Unaligned crossing 64-byte granule',
            data: [1464.95, 209.54, 1631.85, 1060.01, 1786.47, 659.19]
        }
    ]
});

Highcharts.chart('cached_memory_bench', {
    chart: {
        type: 'column'
    },
    title: {
        text: 'system memory "cached" memory access speed (x86-bench)'
    },
    subtitle: {
        text: 'Higher is better'
    },
    xAxis: {
        categories: ['Cortex-X4', 'Oryon-3', 'Zen 4'],
        crosshair: true,
    },
    yAxis: {
        title: {
            text: 'GB/s'
        },
        labels: {
            formatter: function () {
               return this.value / 1000 + 'GB/s';
           }
        }
    },
    plotOptions: {
        column: {
            dataLabels: {
                enabled: true
            }
        }
    },
    tooltip: {
        valueSuffix: ' MB/s'
    },
    plotOptions: {
        column: {
            pointPadding: 0.2,
            borderWidth: 0
        }
    },
    series: [
        {
            name: 'Load',
            data: [23021.84, 26600.32, 0]
        },
        {
            name: 'Store',
            data: [42165.71, 56761.70, 0]
        },
        {
            name: 'LRCPC/TSO-Load',
            data: [22974.25, 26283.45, 38701.65]
        },
        {
            name: 'LRCPC/TSO-Store',
            data: [36111.70, 30596.56, 76897.60]
        }
    ]
});

Highcharts.chart('system_uncached_access', {
    chart: {
        type: 'column'
    },
    title: {
        text: 'system memory write-combine memory access speed (x86-bench)'
    },
    subtitle: {
        text: 'Higher is better (logarithmic)'
    },
    xAxis: {
        categories: ['Cortex-X4', 'Oryon-3', 'Zen 4'],
        crosshair: true,
    },
    yAxis: {
        type: 'logarithmic',
        title: {
            text: 'GB/s'
        },
        labels: {
            formatter: function () {
               return this.value / 1000 + 'GB/s';
           }
        }
    },
    plotOptions: {
        column: {
            dataLabels: {
                enabled: true
            }
        }
    },
    tooltip: {
        valueSuffix: ' MB/s'
    },
    plotOptions: {
        column: {
            pointPadding: 0.2,
            borderWidth: 0
        }
    },
    series: [
        {
            name: 'Load',
            data: [725.92, 485.41, 0]
        },
        {
            name: 'Store',
            data: [41595.52, 56894.52, 0]
        },
        {
            name: 'LRCPC/TSO-Load',
            data: [44.47, 25.88, 75.16]
        },
        {
            name: 'LRCPC/TSO-Store',
            data: [33.34, 87.12, 27212.37]
        }
    ]
});

</script>
