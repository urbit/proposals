+++
up = 5
title = "Asynchronous Multiprocessing"
created = 2018-7-28
authors = "~rovnys-ricfer Ted Blackman <ted@tlon.io>"
editor = "~hidrel-fabtel Morgan Sutherland <morgan@tlon.io>"
status = rfc
++

# Summary

An argument for asynchronous multiprocessing in Urbit.

# Abstract

Urbit is single-threaded; it processes a single event at a time. This means that any long-running computation will block the event loop. A single-threaded Urbit cannot handle an HTTP request and a keystroke at the same time. It also keeps the majority of a typical computer’s processing capacity out of reach of a single Urbit, given the multiplicity of CPU and GPU cores on even small machines like cell phones. Various approaches to parallelism can be taken in Urbit for different applications. I believe asynchronous multiprocessing is the ideal approach for personal Urbit use, and I will provide a model that preserves event log determinism. First, let’s look at other parallelism strategies and what place they should have in the Urbiverse.

# Distributed Computation

The most obvious way to accomplish parallelism in Urbit is to run multiple Urbits and have them talk to each other using network messages. This is potentially appropriate for batch analysis workloads -- Urbit already ships with many of the features of Hadoop, and its networking and computation models could form a very solid foundation for distributed data processing.

It’s easy to picture a swarm of moons all running the same data-processing app as their parent, who is responsible for orchestration. The parent sends a datum to each child for processing, and the children report when they’ve completed the operation. The results of the computations can live on the moons to form an ad-hoc distributed database for this computation. Jetted matrix math is basically the only missing feature here, and that’s something that could be written right now against current Arvo and Vere (if someone wants to do this, I suggest jetting a library that allows for a computation to be built up before execution, based on my experiences with the limitations of NumPy in this regard; TensorFlow is a likely candidate).

The normal downsides of distributed computation apply, however: you now need to perform fleet management, deal with packet loss, and, importantly, serialize and deserialize every datum. This last task is important in Urbit, since as an OS, many of its common tasks involve compiling Hoon, and most Hoon programs contain a copy of the Hoon standard library, which is relatively large (what Ford calls the `%reef`). Taken as a whole, for common user-facing tasks, distributed computation is too heavyweight to be practical.

# Multithreaded Map-Reduce

Another relatively simple approach to parallelism is to implement a parallel version of `+turn`, which is Hoon’s function to map a function over a list. A `+parallel-turn` function could be added that has the same Hoon implementation as normal `+turn`, but whose jet uses a threadpool to parallelize the computation across threads. A parallel version of `+roll` (Hoon’s reduce) could similarly be added. The runtime would be responsible for spawning and reaping the threadpool as needed, managing memory, and marshalling the inputs and outputs of these functions.

Current Vere can almost support this. Its “road” memory model allows for multiple “sibling” roads, which could be used to allow multiple threads to allocate nouns safely within their own memory regions. However, this would require splitting the address space (the “loom”) into sections. With our current 32-bit loom, in practice this would cause a lot of out-of-memory errors. A 64-bit loom should have plenty of (virtual) address space to spare, so this would be fine.

Thread-based parallelism will be useful for things like complex Ford builds and potentially Clay queries, and since the threads run in the same process, there’s no serialization overhead. The big problem with it is just that it doesn’t help Urbit handle multiple events simultaneously.

You could theoretically get some improvement by manually breaking up every potentially long-running computation into many small stages, and then all of those are dumped into one global call to `+parallel-turn` that churns through a little bit of each computation during each event. This monster, though, in addition to being ugly, also suffers from two major failure modes.

The first failure mode is the problem with any cooperative multitasking model (e.g. coroutines), which is the tragedy of the commons: if any one computation decides to eat up CPU time, nothing can stop it. The Arvo kernel needs to prevent any userspace code from expropriating CPU resources, which is difficult in this model.

The second failure mode is that anything that it would be very difficult to implement a reasonable scheduling policy (see [Queueing Theory](https://en.wikipedia.org/wiki/Queueing_theory#Service_disciplines)). For example, a task consisting of a large number of small tasks will take much longer to run than if it were broken up into fewer, larger blocks.

# Software Transactional Memory

A more exotic approach to parallelism would take advantage of some of the unique properties of Urbit to implement a form of software transactional memory. The overall procedure would be that when event `e1` comes in, start processing it. Then when event `e2` is received, even though `e1` hasn’t completed yet, start running `e2` against the same subject against which we’re running `e1`. This is a form of speculative execution (always a great idea!), since if `e1` doesn’t cause mutation to any part of the subject accessed during the processing of `e2`, this second result will be valid. Otherwise, the second transaction will fail, and we’ll have to rerun `e2` using the result of `e1` as the subject.

Fortunately, most possible states of a running Arvo are close to fixed points of the Nock evaluation function: in general, asking Arvo to do something produces a new Arvo that’s very similar to the old one. This means it’s likely that most of an Arvo state won’t be modified from one event to the next (this fact suggests that more generally, assuming the identity function might be a good place to start for Nock optimization strategies). The two limiting cases here are that Arvo is entirely unchanged after running an event, in which case all subsequent computations are still valid, and that Arvo replaces itself with the Nock equivalent of a baked potato, in which case no subsequent computations are valid.

In order to determine whether the `e2` result is valid when run against `e1`’s original subject, we need to know which pieces of the subject it accesses. If we track access (either read or write) to every tree address (expressed as an absolute address with respect to the root of the Arvo subject). To accomplish this, every time we run a Nock 0 lookup, if the target was part of the initial Arvo subject, set its ‘accessed’ bit. This could be stored in a separate data structure in the interpreter, or it could be embedded in the noun’s struct. It might even be possible to simply set the ‘accessed’ bit for a memory address every time the interpreter looks at it at all, irrespective of whether it’s accessed through a nock 0. This would need to only apply to the nouns that were allocated before the event started running.

We also need to track which addresses get mutated during `e1`. If the intersection of addresses mutated during `e1` and accessed during `e2` is not the null set, then the transaction fails. With a “simple” implementation, we’d only know the `e2` was valid after `e1` had finished processing, in case `e1` mutates some tree address right at the end of its computation. Given that Nock computations sometimes restrict themselves to subtrees, it might be mathematically possible in some cases to obtain a transaction success on `e2` even before `e1` completes, but I think that would require implementing a Nock version of [escape analysis](https://en.wikipedia.org/wiki/Escape_analysis). It should be possible to obtain transaction failures for `e2` before `e1` completes, though, since as soon as that set intersection becomes non-null, the transaction is known to fail.

A more limited version of this that might be easier to implement in a performant manner would be to manually identify likely “commutation points” in the kernel’s tree. Good candidates for this would be the root of each vane’s tree. When a tree address is accessed or mutated, mark its whole vane as accessed. This way the set intersection is much smaller, and this might expose ways to optimize the necessary accounting.

This may be some juicy math, but I don’t think it would be the world’s easiest thing to implement -- especially in an efficient manner -- and it also uses events as transaction boundaries, which is fairly coarse. Maybe the runtime could analyze Nock and discover finer transaction boundaries on the fly (think automatically discovering that `+turn` can be parallelized), but I haven’t verified that this is even theoretically possible, much less practically feasible.

Another issue with this approach is that many events cause mutations to vane state in several vanes, which decreases the likelihood that the transaction would succeed. For example, an HTTP request often mutates state in Eyre, Ford, and Gall, and keystrokes usually mutate at least Dill and Gall. If access was tracked at the level of vanes, keystrokes and HTTP requests wouldn’t commute.

Finally, even if we did implement this, it would impose eccentric incentives on data structure design, since we’d try to make mutations less likely to clobber each other. We’d end up doing things like replacing treaps (which we use for maps and sets) with data structures that store keys at predetermined indexes in the tree to prevent tree rotation from destroying our precious commutativity.

# Asynchronous Multiprocessing

As far as I can tell, our best hope for practical Urbit parallelism is asynchronous multiprocessing. Here’s what I mean:

Let’s say Arvo receives an HTTP request for a Fora page (everyone’s favorite Urbit performance pathology). Arvo routes the request from Unix to Eyre, which produces a request that Arvo routes to Ford to perform the build. Ford hasn’t built any Fora pages before, and they all depend on a sidebar, so Ford determines that the first thing to do is build each individual markdown file of all the files in the `/fora` directory (in reality, this is more like step 5, but that’s immaterial for this example). Instead of running these builds synchronously, Ford produces a list of output moves that Arvo routes back to Vere. Each output move contains a Nock computation (subject and formula) to be run asynchronously.

Vere then kicks off some number of new processes (ideally using [fork](http://man7.org/linux/man-pages/man2/fork.2.html) to get the copy-on-write semantics on the Arvo kernel memory), each one with its own Nock interpreter that executes the requested computation. When a computation completes, its result is injected back into Arvo as a new event.

Execution parameters could also be added to the request to run the computation. The most important is a timeout parameter, which would terminate the computation after a timer expires and send a `%timed-out` response back to Arvo instead. Other parameters could possibly include a memory limit and a ‘[nice](https://en.wikipedia.org/wiki/Nice_(Unix))’ level.

This can be looked at as a way of manually specifying the “commutation points” in computations that were intended to be automatically discovered by the software transactional memory system.

The naive implementation of this kind of parallelism suffers from the problem that we’d be frequently dumping big blobs of serialized Nock into the event log, since many of these asynchronous computations will produce new programs, each of which contains a copy of the kernel, which is usually deduplicated in memory but not on disc. We can get around this problem without losing determinism by writing a special entry to the event log that says “the result of running effect #3 of event #307.” When replaying events, we can lazily compute this by rerunning the previous event and running Nock on the computation contained in the specified output effect.

If an asynchronous event produces a Nock failure, that is a deterministic result and its stack trace can be injected back in as a new event without issue. If the computation fails due to an interpreter failure (a ‘bail’), then it never happened. If the runtime doesn’t find out about the failure immediately, it’ll know once the timeout expires and kill the process dead if it’s stuck in a loop or in one of those bizarre half-dead states that Unix processes get into sometimes.

We probably want to limit this interface to Ford, for two reasons. First, it wraps all async computations in a layer of type safety, which feels good. Second, it establishes a shared system scheduler. If you want a computation to be run asynchronously, send it to Ford. Ford will take it right out of the event loop, freeing up the main process to do whatever comes next.

All userspace code should be run this way to prevent user code from overtaking the kernel. This brings Urbit from a nonpreemptive single-threaded OS to something more akin to a [real-time kernel](https://en.wikipedia.org/wiki/Real-time_operating_system), since timeouts could be applied to all user code. This also has the desirable property that Gall apps could now be generalized to use the full [Actor model](https://en.wikipedia.org/wiki/Actor_model).

Another nice property of this paradigm is that if we want to kick off, say, a long-running batch analysis job, that can be done its own process, and internally it can use whatever synchronous jets are available without having to worry about its own scheduling as much. This property might lower the performance requirements for read queries in Clay, since multiple queries could run simultaneously without danger of data races or blocking the event loop.

One issue with this is that when the result of the computation is injected back as an event, it could contain the `%reef` standard library, and we won’t know a priori whether that noun can be deduplicated with the current standard library. We might want to add some code to the interpreter (likely triggered by a Nock hint) that tries to unify pointers to the standard library when the new event comes in. This would probably compare cryptographic hashes of the two standard libraries to prevent malicious user code from violating Nock.