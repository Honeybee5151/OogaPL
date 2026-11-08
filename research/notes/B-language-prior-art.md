# B. Language prior art for tick-lang

Research date: 2026-11-07. Scope: languages and systems whose ideas map onto tick-lang. tick-lang's design: a deterministic fixed tick (`state + inputs -> new state`); share-nothing zones/actors that talk only by messages; struct-of-arrays (SoA) entity storage; rules auto-parallelised from declared data access; per-tick arena memory; bounded queues; versioned, zero-copy, validated network messages; an `unsafe` escape hatch; compiles to Rust.

## How to read this report (sources and labels)

- **VERIFIED (link)**: I fetched the linked page in this session and saw the claim there. Most of these are GitHub repo pages, raw source/docs files on GitHub, arXiv papers, or microsoft.com.
- **SEARCH-SUMMARY (link)**: the claim comes from a web-search result summary of that link. The session's network policy blocked the page, so I could not open it. Treat these as leads, not facts.
- **UNVERIFIED**: from background knowledge. I did not confirm it here.

**Network caveat.** The session's egress policy blocked most official sites: ponylang.io, gleam.run, elixir-lang.org, roc-lang.org, unison-lang.org, halide-lang.org, people.csail.mit.edu, commit.csail.mit.edu, dl.acm.org, link.springer.com, ziglang.org, codeberg.org, futhark-lang.org, modular.com, vale.dev, inko-lang.org, odin-lang.org, wikipedia, lunatic.solutions, serokell.io and mrphilgames.com. Where possible I used the website's *source* on GitHub instead. For example, Pony's "Why not Pony" page came from `ponylang/ponylang-website`, Odin's overview from `odin-lang/odin-lang.org`, and Zig's release notes from `ziglang/www.ziglang.org`. The GitHub REST API was also blocked, so all star counts and dates were read from rendered github.com pages.

**Dates on GitHub release pages.** The fetched release pages often show "DD Mon" with no year. GitHub leaves the year off for dates in the current year, but the page-to-text conversion can also drop it. So for each project I also give a commit date *with year* from that repo's commits page.

---

## Summary table

| Language | Relevance to tick-lang | Key borrowable idea | Alive? (evidence date) |
|---|---|---|---|
| Pony | High | Reference capabilities (`iso`/`val`/`tag`) make zero-copy message passing race-free; per-actor GC | Yes: ponyc 0.78.1, commits Nov 7, 2026 |
| Encore | Medium | Active objects + capabilities; "Bestow/Atomic" for cross-actor atomicity | **Deprecated** (notice committed Feb 23, 2026) |
| Lunatic | Medium | Wasm-sandboxed processes, each with its own heap, preemption and per-process permissions | **Dormant**: last commit Mar 29, 2025 |
| Gleam | Medium | Typed actor messages (`Subject(T)`) | Yes: v1.19.2; commits Nov 7, 2026 |
| Elixir | Medium | Set-theoretic gradual types added to an existing dynamic language with no annotations | Yes: v1.20.5; commits Nov 7, 2026 |
| Koka | High | Perceus RC + reuse = "functional but in-place" (FBIP); effect handlers | Yes, research: v3.2.9 (2026-09-17) |
| Roc | **Very high** | Platform/application split: pure app, platform owns all I/O and allocation | Yes, pre-0.1: commits Nov 7, 2026 |
| Unison | Medium | Content-addressed definitions; hash-keyed caching of deterministic tests | Yes: release/1.5.0; commits Nov 6, 2026 |
| Halide | **Very high** | Separate the algorithm (what) from the schedule (how) | Yes: v22.1.0; commits Nov 5, 2026 |
| Futhark | High | Data-parallel SOACs; compile to a *library* that a host program calls | Yes: 0.28.2; commits Nov 7, 2026 |
| Bend / HVM2 | Low-Medium (cautionary) | Automatic parallelism from program structure | Active, but Bend has pivoted ("Bend 2"); commits Nov 7, 2026 |
| Jai | Medium (historical) | SoA as a type-level switch | Closed beta; not public (search only) |
| Zig | High | Explicit allocators, `ArenaAllocator`, `MultiArrayList` (SoA), comptime | Yes, pre-1.0; moved to Codeberg (Nov 2025) |
| Odin | High | `#soa` arrays/slices/dynamic arrays; `context.temp_allocator` "freed once per cycle/frame" | Yes: dev-2026-11; commits Nov 7, 2026 |
| Verona | **Very high** | Regions with per-region memory strategy (arena!), cowns + `when` (BoC) multi-resource atomic behaviours | Yes, research: commits Nov 6, 2026 |
| Inko | High | Share-nothing processes; only `uni T` (unique) or value types are sendable | Yes: v0.21.1 (Jul 20, 2026) |
| Hylo | Medium | Mutable value semantics (no shared mutable aliasing) | Old repo superseded by `hylo-new` |
| Vale | Low-Medium (cautionary) | Generational references, region borrow checking | **Archived** (README, Sep 23, 2026 commits) |
| Lobster | Medium | Game-focused; compile-time RC via ownership analysis; inline by-value structs | Yes: commits Nov 6, 2026 |
| Bevy ECS (adjacent, not a language) | **Very high** | Scheduler infers parallelism from systems' declared data access | Yes (README fetched) |
| Austral / Cone / Mojo / Carbon | Low | See "Marginal" section | Mixed |

---

## 1. Actor and share-nothing languages

### Pony
**Summary.** Pony is an actor-model language with ahead-of-time compilation. Its type system uses *reference capabilities*: `iso` (isolated), `val` (immutable), `ref`, `box`, `trn` and `tag`. These make data races impossible to express. The repo describes itself as "an open-source, actor-model, capabilities-secure, high performance programming language". VERIFIED (https://github.com/ponylang/ponyc). Each actor has its own heap. Memory is collected by ORCA, a concurrent per-actor collector that co-designs the GC with the type system. Paper: Clebsch, Franco, Drossopoulou, Yang, Wrigstad, Vitek, OOPSLA 2017, DOI 10.1145/3133896. SEARCH-SUMMARY (https://www.doc.ic.ac.uk/~scd/ORCA_17.pdf).

**Borrow.**
- **Capabilities for zero-copy messages.** Sending an `iso` or `val` across an actor boundary is zero-copy and race-free, and the compiler checks it. For tick-lang, a zone-to-zone message is exactly this: either a unique buffer that is moved, or an immutable buffer that is shared. Inko copied the same idea (`uni T`, "the same as Pony's isolated values"). VERIFIED (https://raw.githubusercontent.com/inko-lang/inko/main/docs/source/getting-started/concurrency.md)
- **Runtime backpressure.** Pony's mailboxes are unbounded, so the runtime uses scheduling-based backpressure instead: "When an actor becomes overloaded, the Pony runtime will deprioritize scheduling the actors that are sending to it". Overload from outside the program (such as TCP) cannot be detected automatically: "There's no way for the Pony runtime to detect this, so intervention by the programmer is needed." VERIFIED (https://raw.githubusercontent.com/ponylang/ponyc/main/packages/backpressure/backpressure.pony). **Lesson:** tick-lang's *bounded* queues are the stricter, deterministic version of this. Pony shows that unbounded mailboxes need a patch later.

**Weakness / what held it back.**
- The project's own "Why not Pony?" page lists "Lack of API stability", "Lack of high-quality 3rd party libraries" and "Limited native tooling". It also says "We don't have an IDE" and "The pool of open source, ready to use Pony libraries is very small." VERIFIED (https://raw.githubusercontent.com/ponylang/ponylang-website/main/docs/discover/why-not-pony.md)
- It is still pre-1.0 and "semi-regularly introduces breaking changes". VERIFIED (https://github.com/ponylang/ponyc/blob/main/README.md)
- Its memory management is per-actor only. Verona's authors note: "Pony only supports memory management on a per-actor level, (using tracing GC). We could make each request an actor, but it would have to communicate asynchronously with all surrounding state." VERIFIED (https://arxiv.org/pdf/2309.02983). This matters for tick-lang: you want arenas *finer* than a zone, such as per tick or per message.
- Reference capabilities are widely seen as hard to learn. UNVERIFIED.

**Who.** Sylvan Clebsch created it, with Imperial College / Causality-era research. SEARCH-SUMMARY (ORCA paper authors). It is now community-maintained. Sean T. Allen (Wallaroo Labs) has been a long-time lead. SEARCH-SUMMARY (https://speakerdeck.com/seantallen/pony-how-i-learned-to-stop-worrying-and-embrace-an-unproven-technology). Wallaroo's stream processor "Wally" says "The primary API for Wally is written in Pony". VERIFIED (https://github.com/WallarooLabs/wally)

**Alive.** Yes. The 0.78.1 release is dated "07 Nov" on the releases page, and the commits page shows "Rotate release notes as part of the 0.78.1 release" on November 7, 2026. The repo has 6.2k stars. VERIFIED (https://github.com/ponylang/ponyc/releases, https://github.com/ponylang/ponyc/commits/main)

### Encore (Uppsala)
**Summary.** Encore is an active-object language from Uppsala's parallel-programming group (`parapluu`). It combines actors/active objects with capability types, and research grew out of it such as "Bestow and Atomic" (Castegren, Wallin, Wrigstad, 2018). That paper's abstract states the core tension: "If actor isolation is broken, so is the single-thread-of-control abstraction." It proposes separating isolation from synchronisation. VERIFIED (https://arxiv.org/abs/1807.10130)

**Borrow.** Actors often need to update several of them atomically, for example a trade between two players in different zones. That needs a construct beyond plain message passing. Encore's answer was bestow/atomic; Verona's later answer is BoC (see below). tick-lang needs a story for cross-zone transactions.

**Weakness / failure.** The README says: "The Encore project is officially deprecated. You can read our post-mortem here" (Springer, DOI 10.1007/978-3-031-51060-1_3). VERIFIED (https://github.com/parapluu/encore/blob/development/README.md). The post-mortem chapter is "Encore: Coda" by Elias Castegren and Tobias Wrigstad, in *Active Object Languages: Current Research Trends* (LNCS, 2024), pp. 59-91. SEARCH-SUMMARY (https://ouci.dntb.gov.ua/works/4NBYNoY4); Springer itself was blocked. Before the deprecation notice, the last commit was Apr 11, 2019. VERIFIED (https://github.com/parapluu/encore/commits/development). **Lesson:** a research language that serves as a vehicle for papers dies when the funding or PhD cohort moves on. The repo has 44 stars. VERIFIED (https://github.com/parapluu/encore)

**Who.** Uppsala University (Wrigstad, Castegren et al.). SEARCH-SUMMARY.

**Alive.** No. The deprecation notice was committed Feb 23, 2026. VERIFIED (https://github.com/parapluu/encore/commits/development)

### Lunatic
**Summary.** Lunatic is "an Erlang-inspired runtime for WebAssembly". VERIFIED (https://github.com/lunatic-solutions/lunatic). In its README's words: "Each process has its own stack, heap, and even syscalls"; "All processes running on Lunatic are preemptively scheduled and executed by a work stealing async executor"; and "Fine-grained process permissions" are listed as done. Hot reloading is listed as *not* done. VERIFIED (https://raw.githubusercontent.com/lunatic-solutions/lunatic/main/README.md)

**Borrow.** Wasm gives hard memory isolation per process plus capability-style permissions. This is a model for sandboxing untrusted *mods/scripts* inside a zone. Wasm fuel metering could give deterministic preemption (UNVERIFIED that Lunatic uses fuel).

**Weakness / failure.** It stalled. The five most recent commits are dated Mar 29, 2025; Mar 28, 2025 ("Fix build with rust 1.80"); Mar 18, 2024; Oct 18, 2023; and Oct 9, 2023. VERIFIED (https://github.com/lunatic-solutions/lunatic/commits/main). The latest release is v0.13.2; the page did not render its year. VERIFIED (https://github.com/lunatic-solutions/lunatic/releases). The README admits WASI compatibility: "We're not quite there yet." VERIFIED (repo page). **Lesson:** a startup-backed runtime with a small core team can stop without any announcement.

**Who.** Bernard Kolobara created it. SEARCH-SUMMARY (https://serokell.io/blog/lunatic-with-bernard-kolobara). The company is "Lunatic Solutions". Dealroom lists it as a startup, with funding details marked as inferred. SEARCH-SUMMARY (https://app.dealroom.co/companies/lunatic). I found no 2024-2026 company news.

**Alive.** Effectively dormant since early 2025. The repo has 4.9k stars and no archive banner. VERIFIED.

### Gleam
**Summary.** Gleam is a statically typed functional language for the BEAM, and also compiles to JavaScript (the JS target is UNVERIFIED here). The repo says "A friendly language for building type-safe, scalable systems!" and has 22.0k stars. VERIFIED (https://github.com/gleam-lang/gleam)

**Borrow.** Typed mailboxes. In `gleam_otp`, actors receive messages through a `Subject(Message)`, and request/reply carries a `Subject(Int)` reply channel. The library's stated goals are "Full type safety of actors and messages" and "Have equivalent performance to Erlang's OTP." VERIFIED (https://raw.githubusercontent.com/gleam-lang/otp/main/README.md). tick-lang's typed zone-message endpoints should look like this.

**Weakness.** Retrofitting types onto an untyped actor runtime leaves gaps: "Not all Erlang/OTP functionality is included in this library. Some is not possible to represent in a type safe way, so it is not included." VERIFIED (same). **Lesson:** design the message and supervision model *together with* the type system, not after it.

**Who.** "Gleam is not owned by a corporation, instead it is kindly supported by its sponsors." VERIFIED (https://raw.githubusercontent.com/gleam-lang/gleam/main/README.md). Louis Pilfold created it (UNVERIFIED in this session).

**Alive.** Yes. v1.19.2 is dated 12 Oct (VERIFIED, https://github.com/gleam-lang/gleam/releases), and commits are dated Nov 7, 2026 (VERIFIED, https://github.com/gleam-lang/gleam/commits/main).

### Elixir (and its set-theoretic type system)
**Summary.** Elixir is a dynamic BEAM language. It is adding a gradual type system with set-theoretic types and semantic subtyping, described in "The Design Principles of the Elixir Type System" (Castagna, Duboc, Valim; arXiv June 2023, rev. Dec 2023). VERIFIED (https://arxiv.org/abs/2306.06391). In v1.20, "Elixir's type system now understands all language constructs and can infer types for your function definitions ... to find verified bugs and dead code". It performs whole-body inference, inference of guards, occurrence typing across clauses, and typing of map keys. All of this works "without a need to introduce type signatures (yet)". VERIFIED (https://raw.githubusercontent.com/elixir-lang/elixir/v1.20/CHANGELOG.md)

**Borrow.**
- **Union/intersection/negation types with occurrence typing.** These fit well for validated network messages. A message `%{..., foo: not_set()}` versus `if_set(integer())` is close to what a versioned schema needs ("field added in v3, optional before").
- **Process.** The type system was rolled out in steps, as warnings, without breaking existing code. That is a model for adding stricter checks to tick-lang over time.

**Weakness.** It is gradual and warning-only, with no user-written signatures yet (CHANGELOG, above). It took years: the paper is from 2023, and full inference only landed in 1.20. Soundness at `dynamic()` boundaries is weaker than in a statically typed design (UNVERIFIED characterisation).

**Who.** José Valim. The CHANGELOG says: "The type system was made possible thanks to a partnership between CNRS and Remote. The development work is currently sponsored by Fresha and Tidewave." VERIFIED (CHANGELOG above)

**Alive.** Yes. v1.20.5 is dated 29 Oct (VERIFIED, https://github.com/elixir-lang/elixir/releases), and commits are dated Nov 7, 2026 (VERIFIED, https://github.com/elixir-lang/elixir/commits/main).

---

## 2. Effects, purity, memory without GC

### Koka
**Summary.** Koka is a research functional language with row-typed effects and effect handlers. It compiles to C with no tracing GC, using Perceus reference counting. The README says: "Koka v3 is a research language that is currently under development and not quite ready for production use", and "Effect handlers let you define advanced control abstractions". It calls Perceus "an advanced compilation method for reference counting" that "optimizes functional-style programs to use in-place updates when possible". The latest release is "v3.2.9, 2026-09-17". The repo has 4.1k stars. VERIFIED (https://github.com/koka-lang/koka)

**Borrow.**
- **Perceus / FBIP.** The paper is "Perceus: Garbage Free Reference Counting with Reuse" (Reinking, Xie, de Moura, Leijen), MSR-TR-2020-42, the extended version of PLDI'21. Its abstract says reuse analysis enables "writing in-place mutating algorithms in a purely functional way". VERIFIED (https://www.microsoft.com/en-us/research/publication/perceus-garbage-free-reference-counting-with-reuse/). This is the key trick for `state + inputs -> new state`. Write the tick as a pure function. When the old state's reference count is 1, which it always is in a tick loop, the compiler reuses the old state's memory in place. You get purity in the semantics and mutation in the machine code.
- **Effect rows.** These can type what a rule may do: `read Position`, `write Velocity`, `send ZoneMsg`. That is the declared data access that tick-lang's auto-paralleliser needs.

**Weakness.** The project calls itself not production-ready (above). The Perceus abstract makes only a qualitative claim, "competitive with other state-of-the-art memory collectors". The page showed no numbers. VERIFIED (MSR page). RC still costs atomic operations for shared data, and cycles leak (UNVERIFIED for Koka specifically).

**Who.** Daan Leijen (README signed "Daan Leijen"), with Microsoft Research publications. VERIFIED (repo, MSR page)

**Alive.** Yes. The latest dev-branch commit is dated Oct 27, 2026. VERIFIED (https://github.com/koka-lang/koka/commits/dev)

### Roc (very relevant)
**Summary.** Roc is a fast, friendly, pure functional language. It has 6.2k stars and 54,085 commits. VERIFIED (https://github.com/roc-lang/roc). The README opens with "Work in progress!" and says Roc "is not ready for a 0.1 release yet". VERIFIED (https://raw.githubusercontent.com/roc-lang/roc/main/README.md). Its defining idea is the platform/application split. "Roc has a first-class concept of platforms and applications." "Every Roc application specifies exactly one platform." "The selected platform provides the app's I/O primitives." "Roc's standard library does not include any effectful functions; they all come from the platform." Effectful functions end in `!`. VERIFIED (https://github.com/roc-lang/roc/blob/main/docs/mini-tutorial-new-compiler.md). The platform's *host* is written in another language, such as Zig. Host authors implement the I/O primitives and also memory allocation and deallocation. Platforms "can create domain-specific security guarantees" and decide how concurrent I/O is scheduled. SEARCH-SUMMARY (https://www.roc-lang.org/platforms)

**Borrow (directly).** tick-lang is in effect a Roc-style split. The **app** is the game rules: pure, deterministic, `state + inputs -> new state`. The **platform** is the Rust runtime: networking, tick clock, persistence, thread pool, arena allocator. Because the platform supplies the allocator, a per-tick arena becomes a platform policy, not a language feature. Because the app cannot do I/O, determinism and replay come for free.

**Weakness.**
- No 0.1 release after many years (README).
- The compiler was rewritten from Rust to Zig. Richard Feldman's gist gives these reasons: "Rust's compile times are slow. Zig's compile times are fast."; "Rust's memory safety guarantees aren't major selling points in this particular project"; "We tend to pass around allocators for memory management (like Zig does, and Rust does not)". It also notes that Roc's standard library "has been written in Zig for years". The earliest comments on the gist are dated February 4, 2025. VERIFIED (https://gist.github.com/rtfeldman/77fb430ee57b42f5f2ca973a3992532f)
- Secondary reports say the rewrite reached feature parity in 2026, with figures like "300K lines", "464,000 lines of Zig" and "487 days". SEARCH-SUMMARY only (https://weeklyrust.substack.com/p/why-roc-is-moving-away-from-rust, https://zeli.app/es/story/48933149); treat the numbers as unconfirmed.
- **Lesson for tick-lang ("compiles to Rust"):** compile time of the *generated* Rust and of the compiler itself is a real risk. Roc left Rust partly over compile times.

**Who.** Richard Feldman. Sponsors include the Roc Programming Language Foundation via Every.org, GitHub Sponsors, and corporate sponsors "Lambda Class, ohne-makler, and Decem". VERIFIED (README)

**Alive.** Yes, very active. Commits are dated November 7, 2026, and `build.zig` is at the repo root. VERIFIED (https://github.com/roc-lang/roc/commits/main, repo page)

### Unison
**Summary.** Unison is "a statically-typed functional language with type inference, an effect system, and advanced tooling ... based around a big idea of content-addressed code, in which function are identified by a hash of their implementation rather than by name, and code is stored as its AST in a database." The README lists "No builds", "Perfect caching of tests, only rerunning determinstic tests if dependencies changed", and "Semantically-aware version control". It can be used "in conjunction with Unison Cloud for building distributed systems." VERIFIED (https://raw.githubusercontent.com/unisonweb/unison/trunk/README.md). The repo has 6.8k stars. VERIFIED (https://github.com/unisonweb/unison)

**Borrow.**
1. **Hash message schemas and rule code.** A versioned network message type could be identified by the hash of its definition. Then a client and server agree on wire layout by hash, not by a hand-bumped version number.
2. **Hash-keyed deterministic replay tests.** A deterministic tick function is perfectly cacheable, so re-run a replay only when a rule it depends on changes.

**Weakness.** Storing code in a database instead of text files means a custom toolchain (UCM, its own version control). The distributed story is tied to a commercial cloud product. Both are UNVERIFIED characterisations. The README does confirm that the cloud product exists.

**Who.** Unison Computing, the company behind unison.cloud (the company name is UNVERIFIED on a fetched page; the repo org is `unisonweb`).

**Alive.** Yes. release/1.5.0 is marked Latest and dated 02 Oct (VERIFIED, https://github.com/unisonweb/unison/releases). Commits are dated November 6, 2026, including a merged "jit" PR. VERIFIED (https://github.com/unisonweb/unison/commits/trunk)

---

## 3. Data-parallel and scheduling languages

### Halide (very relevant)
**Summary.** Halide is a DSL embedded in C++ for image and array pipelines. It separates the *algorithm* (what each stage computes, as pure functions over coordinates) from the *schedule* (tiling, vectorisation, parallelism, fusion versus recomputation, storage). The README says: "Halide is a programming language designed to make it easier to write high-performance image and array processing code on modern machines." The repo has 6.6k stars. VERIFIED (https://github.com/halide/Halide)

**The PLDI 2013 paper.** "Halide: A Language and Compiler for Optimizing Parallelism, Locality, and Recomputation in Image Processing Pipelines" (Ragan-Kelley, Barnes, Adams, Paris, Durand, Amarasinghe), PLDI'13, pp. 519-530. SEARCH-SUMMARY (https://researchr.org/publication/Ragan-KelleyBAPDA13). I could **not** fetch the PDF (people.csail.mit.edu, commit.csail.mit.edu and dl.acm.org were all blocked). Its headline claims, as given by search summaries of the PDF:
- Autotuned Halide is "up to 5x faster than hand-tuned C, intrinsics, and CUDA implementations" written by experts. Those took "weeks or months" to optimise; the Halide versions were written "in a few hours". SEARCH-SUMMARY (https://people.csail.mit.edu/jrk/halide-pldi13.pdf)
- Lines of code. Per a summary of a Ragan-Kelley talk, the expert hand-optimised local Laplacian filter in Adobe Camera Raw/Lightroom was about 1,500 lines, took roughly three months, and ran about 10x faster than reference C. SEARCH-SUMMARY (https://halide-lang.org/assets/lectures/Halide_CVPR_intro.pdf). The search engine's OCR of the paper's results table was garbled, so **I report no per-benchmark LOC or speedup numbers from the table**. Check the PDF directly before quoting any.

**Verified follow-up evidence** (Anderson, Adams [Adobe], Ma, Li, Jin, Ragan-Kelley [MIT], "Efficient Automatic Scheduling of Imaging and Vision Pipelines for the GPU", OOPSLA; arXiv 2012.07145):
- "We build on Halide, a domain-specific compiler that decouples the algorithm – what to compute – from the the schedule – how to compute it. This separation makes it easier to explore different schedules for a given algorithm, but finding high-performance schedules remains a challenge."
- "Manually exploring these options is time-consuming, and it is difficult to predict ahead of time whether a change will help or hurt performance."
- The autoscheduler finds schedules "on average 1.7× faster than existing automatic solutions (up to 5×), and competitive with what the best human experts were able to achieve". It reports "average compile time speedups of 49× (up to 530×)".

VERIFIED (https://arxiv.org/abs/2012.07145, PDF text)

**Borrow.** Make tick-lang's *rules* the algorithm. Make *placement* a separate, optional schedule: which zone/thread, batch size, SoA versus AoSoA chunking, ordering. A correct default is derived automatically from declared reads/writes. Experts can override the schedule without touching the rules, and the schedule *cannot change results*. Halide's guarantee that a schedule never changes output is exactly the determinism guarantee tick-lang wants.

**Weakness.** By the project's own later account, hand-writing schedules is hard and slow, which is why a decade of autoscheduler research followed. Halide is also restricted to rectangular, stencil-like array pipelines. **Lesson:** ship a good automatic default schedule from day one; do not require users to write schedules.

**Who.** It began at MIT CSAIL and Adobe (paper authors above). It is now maintained by the `halide` GitHub org. Industrial users such as Google and Adobe were not stated on the fetched pages (UNVERIFIED).

**Alive.** Yes. v22.1.0 is dated 31 Oct (VERIFIED, https://github.com/halide/Halide/releases), and commits are dated November 5, 2026 (VERIFIED, https://github.com/halide/Halide/commits/main).

### Futhark
**Summary.** "Futhark is a purely functional data-parallel programming language in the ML family. It can be compiled to typically very efficient parallel code, running on either a CPU or GPU. The language is developed at DIKU at the University of Copenhagen ... It is quite stable and suitable for practical programming." VERIFIED (https://raw.githubusercontent.com/diku-dk/futhark/master/README.md)

**Borrow.**
- Second-order array combinators (`map`/`reduce`/`scan`), with a compiler that knows their parallel semantics.
- The **library deployment model**: "compiling a Futhark program to an executable is useful for testing, it is not suitable for production use. Instead, a Futhark program should be compiled into a reusable library in some target language, enabling integration into a larger program." VERIFIED (https://raw.githubusercontent.com/diku-dk/futhark/master/docs/usage.rst). This is the same shape as Roc's platform split, and as "tick-lang compiles to a Rust crate the server embeds".

**Weakness.** It deliberately excludes irregular data. The glossary says an irregular array, one "where the elements do not have the same size ... [is] not supported in Futhark." VERIFIED (https://raw.githubusercontent.com/diku-dk/futhark/master/docs/glossary.rst). GPU backends "usually sequentialise" irregular nested loops. SEARCH-SUMMARY (https://futhark.readthedocs.io/en/latest/glossary.html). Game entities are irregular (variable inventories, neighbour lists), so tick-lang needs ragged or segmented structures as a first-class concept.

**Who.** DIKU, University of Copenhagen, led by Troels Henriksen (the name is UNVERIFIED on a fetched page).

**Alive.** Yes. 0.28.2 is dated 27 Oct (VERIFIED, https://github.com/diku-dk/futhark/releases), and commits are dated November 7, 2026 (VERIFIED, https://github.com/diku-dk/futhark/commits/master).

### Bend / HVM (HigherOrderCO): cautionary
**Summary.** HVM2 is "a massively parallel Interaction Combinator evaluator". VERIFIED (https://github.com/HigherOrderCO/HVM2). Bend 1 (2024) was a high-level language that ran on HVM2 and promised automatic parallelism with no annotations.

**Benchmark claims (primary source).** HVM2 paper by Victor Taelin: "scaling from 400 million interactions per second (MIPS) (Apple M3 Max; single thread), to 5,200 MIPS (Apple M3 Max; 16 threads), to 74,000 MIPS (NVIDIA RTX 4090; 32,768 threads)". VERIFIED (https://github.com/HigherOrderCO/HVM2/blob/main/paper/HVM2.pdf, abstract)

**Critiques, including the paper's own.**
- Section 13.3 "Single Core Inefficiency": "In single-thread CPU evaluation, HVM2, is, baseline, still about 5x slower than GHC, and this number can grow to 100x on programs that involve loops and mutable arrays". Also: "a single-core C program that adds numbers from 0 to a few billions will easily outperform an HVM2 one that uses thousands of threads". VERIFIED (HVM2.pdf, pp. 23-24)
- The old HVM1 README admits HVM is "from 50% faster to 3x slower in single thread performance" compared to GHC, and "HVM is still to be considered a prototype." VERIFIED (https://github.com/HigherOrderCO/HVM)
- In issue #525 (May 29, 2024), `-O all` made the README `sum` example *slower*: `run-c` took 22.72s with `-O all` versus 3.36s by default, and `run-cu` 2.44s versus 0.40s. VERIFIED (https://github.com/higherorderco/bend/issues/525)
- In a May 2024 Reddit thread, users reported a single-threaded C++ build beating Bend's parallel sum on an RTX 4090. A co-founder replied that the earlier "4090 vs single core" claim came from an out-of-date pitch deck. SEARCH-SUMMARY (https://redlib.hbubli.cc/r/programming/comments/1cuh4pg/fireship_mindbending_new_programming_language_for/l4l4s3k)

**Pivot.** The Bend repo now describes "Bend 2: a fast language that blocks AI mistakes via proof", with targets "be as fast as C on the CPU, as fast as CUDA on the GPU". It says "Parallelism requires balanced calls" and "BEND IS YOUNG. EXPECT BUGS". It has 24.9k stars. VERIFIED (https://github.com/HigherOrderCO/Bend)

**Lesson.** *Scaling* is not *speed*. Automatic parallelism that starts from a 5-100x single-thread deficit loses to a plain sequential loop. tick-lang should parallelise *coarse* units (rules over SoA chunks, zones) on top of a fast sequential baseline. Benchmarks should always report single-thread versus the best sequential implementation.

**Who.** Higher Order Company (Victor Taelin). VERIFIED (paper author)

**Alive.** Active. Bend commits are dated November 7, 2026. VERIFIED (https://github.com/HigherOrderCO/Bend/commits/main)

---

## 4. Game-oriented and systems languages: SoA and allocators

### Jai (Jonathan Blow)
**Summary.** Jai is a game-oriented systems language with heavy compile-time metaprogramming. It is **not public**. As of early 2026 it is a closed beta; Wikipedia lists the latest preview as "0.2.026 (Beta), dated 21 February 2026". SEARCH-SUMMARY (https://en.wikipedia.org/wiki/Jai_(programming_language)). There is no official public-release timeline. One aggregator claims a first game ships Oct 8 with the engine open-sourced afterwards. SEARCH-SUMMARY, unconfirmed (https://byteiota.com/jai-programming-language-jonathan-blow-2026/)

**SoA history.** The fan-written JaiPrimer (from early demos, around 2015) shows `Entity :: struct SOA { ... }`, which switches layout "without breaking the supporting code". VERIFIED (https://raw.githubusercontent.com/BSVino/JaiPrimer/master/JaiPrimer.md). The same file warns: "Jon has said that he doesn't think the contents of this document describe Jai well". Community sources say the built-in SOA/AOS keyword **was removed** because "its functionality is covered by all metaprogramming stuff". SEARCH-SUMMARY (Reddit r/Jai, https://publish.obsidian.md/xybre/permalink/b3c9b40e-8690-4334-aaab-32f70f1b74d6). This is not confirmed by an official source.

**Borrow.** Keep layout (AoS/SoA/AoSoA) *orthogonal to access syntax*, so a layout change does not break rule code. Jai's experience suggests building this from general compile-time reflection, not a one-off keyword.

**Weakness.** Over a decade without a public release means no external ecosystem or validation. Everything known comes second-hand. This is a cautionary tale for tick-lang about shipping something usable early.

**Who.** Jonathan Blow / Thekla. **Alive:** in closed beta (SEARCH-SUMMARY only).

### Zig
**Summary.** Zig is a systems language built around `comptime` and explicit allocators. It moved off GitHub: the repo's About text is "Moved to Codeberg", and the last GitHub commit was "README: migrated to codeberg" on Nov 27, 2025. The repo has 43.3k stars. VERIFIED (https://github.com/ziglang/zig, https://github.com/ziglang/zig/commits/master)

**Borrow.**
- **`ArenaAllocator`**: "takes an existing allocator, wraps it, and provides an interface where you can allocate and then free it all together". VERIFIED (https://raw.githubusercontent.com/ziglang/zig/master/lib/std/heap/arena_allocator.zig). Allocators are passed explicitly, so a function's signature shows whether it allocates and from where. For tick-lang, a rule receives a `tick_arena` and the compiler proves nothing allocated in it escapes the tick.
- **`MultiArrayList`**: stores each field of a struct in a separate list, which "improves cache usage if only some fields or just tags are needed". Fields are accessed with `.items(.field)`. VERIFIED (https://raw.githubusercontent.com/ziglang/zig/master/lib/std/multi_array_list.zig). This is SoA as a comptime library, not a language feature, which is the same conclusion Jai reached.

**Weakness.** It is pre-1.0 with heavy churn. The 0.15.1 notes describe "the Writergate scandal ... rocks the boat with tidal waves of breaking API changes". Development is "funded via Zig Software Foundation, a 501(c)(3) non-profit". VERIFIED (https://raw.githubusercontent.com/ziglang/www.ziglang.org/main/src/download/0.15.1/release-notes.html). 0.16.0 reportedly shipped on Apr 14, 2026, with "I/O as an interface". SEARCH-SUMMARY (https://engineered.at/articles/zig-0-16-0-release-notes-juicy-main); the official page on ziglang.org/Codeberg was blocked.

**Who.** Andrew Kelley / Zig Software Foundation (ZSF VERIFIED; the name Kelley is UNVERIFIED here).

**Alive.** Yes. Development continues on Codeberg (not fetchable here); 0.16.0 dates are SEARCH-SUMMARY.

### Odin
**Summary.** Odin is a data-oriented systems language aimed at games and graphics. Its relevant features are first-class SoA types and implicit context allocators.

**Borrow.**
- **`#soa[N]T`, `#soa[]T` and `#soa[dynamic]T`**, which use "Same syntax as AOS and treat as if it was an array" (`v_soa[0].x`). SoA syntax (`v_soa.x[0]`) is also available, along with SoA pointers and `soa_zip`/`soa_unzip`.
- **Per-frame scratch allocator.** "`context.temp_allocator` is for temporary and short lived allocations, which are to be freed once per cycle/frame/etc." By default it is "a scratch allocator (a growing arena based allocator)", reset with `free_all(context.temp_allocator)`. This is exactly a per-tick arena.

VERIFIED (https://raw.githubusercontent.com/odin-lang/odin-lang.org/master/content/docs/overview.md)

**Weakness.**
- The per-frame arena is a *convention*. Nothing stops a temp pointer from escaping past `free_all`. tick-lang should make that a compile error, which is easy if tick-arena values have a region type.
- No 1.0 and no spec: "Odin is not currently self hosted nor will be until after version 1.0 when the main implementation of the Odin compiler adheres to a specification". The FAQ also prefers manual dependency vendoring over package managers. VERIFIED (https://raw.githubusercontent.com/odin-lang/odin-lang.org/master/content/docs/faq.md)

**Who.** Ginger Bill (UNVERIFIED here). Production use at JangaFX (UNVERIFIED).

**Alive.** Yes. Monthly tags dev-2026-11 / dev-2026-11a are dated 03 Nov (VERIFIED, https://github.com/odin-lang/Odin/releases), and commits are dated November 7, 2026 (VERIFIED, https://github.com/odin-lang/Odin/commits/master).

### Lobster
**Summary.** "Lobster is a statically typed programming language with a Python-esque syntax", with "compile-time memory management". The repo has 2.8k stars. VERIFIED (https://github.com/aardappel/lobster). Its memory doc says: "Lobster combines its original (runtime) reference counting with a ownership analysis algorithm, to get 'compile time reference counting'". It notes the language "has so far focussed on the areas of games and computer graphics". Inline by-value `struct`s make 2D/3D vectors free: "The struct disappears." VERIFIED (https://raw.githubusercontent.com/aardappel/lobster/master/docs/source/memory_management.md)

**Borrow.**
- Small vector/quaternion types must be by-value and inline. They are the fields of SoA columns.
- Ownership inference that "mostly does not require programmer intervention" is a good model for how much ownership annotation game programmers will tolerate.

**Weakness.** Reference counting "can't collect cycles", and the author relies on a "cycle report" at program exit. VERIFIED (same doc). It is effectively a one-person language with a small community (UNVERIFIED).

**Who.** Wouter van Oortmerssen (repo owner `aardappel`; the name is UNVERIFIED on a fetched page).

**Alive.** Yes. Commits are dated November 6, 2026. VERIFIED (https://github.com/aardappel/lobster/commits/master)

### Bevy ECS (adjacent prior art, not a language)
Included because it is the most direct existing implementation of "rules auto-parallelised from declared data access". "Bevy ECS can use function parameter types to determine what data needs to be sent to the system. It also uses this 'data access' information to determine what Systems can run in parallel with each other." "The built in 'parallel executor' considers dependencies between systems and (by default) run as many of them in parallel as possible." VERIFIED (https://raw.githubusercontent.com/bevyengine/bevy/main/crates/bevy_ecs/README.md)

**Lesson.** This already works in Rust as a library. tick-lang's value over Bevy has to come from things a library cannot do: compile-time proofs of determinism, arena non-escape, bounded queues and message versioning. Parallel scheduling alone is not enough. Bevy's need for explicit ordering ("define explicit dependencies between systems") also shows that tick-lang must define a *deterministic* order for conflicting rules, not leave it to the executor.

---

## 5. Ownership, regions, value semantics (2023-2026)

### Project Verona (Microsoft Research): very relevant
**Summary.** Verona is a "Research programming language for concurrent ownership". The README says "This repository contains an experimental Verona compiler and runtime toolchain". The repo has 3.7k stars. VERIFIED (https://github.com/microsoft/verona)

**Borrow 1: regions with a per-region memory strategy.** Objects are grouped into isolated regions, and a thread may mutate only its active region (its "window of mutability"). "Memory is managed locally for each region." In the paper's worked server example, "each request is wrapped in a region ... These regions use arena allocation: allocations in the region persist until the region itself is deallocated. This gives cheap bump-pointer allocation and fast deallocation of the entire region once the response has been computed." A region can switch away from arena allocation "at the small cost of changing one annotation at the creation site". VERIFIED (Arvidsson, Castegren, Clebsch, Drossopoulou, Noble, Parkinson, Wrigstad, OOPSLA 2023; https://arxiv.org/abs/2309.02983 and PDF text). For tick-lang this maps to: **per-tick arena = a region; zone state = a long-lived region; an outgoing message = an `iso` region transferred to another zone.**

**Borrow 2: Behaviour-Oriented Concurrency (BoC).** "Updating multiple actors as a single atomic operation is a challenging task". BoC creates atomic, ordered units of work with "exclusive access to a collection of independent resources". On Savina, BoC programs are more compact than actor versions with "comparable, if not better, performance". VERIFIED (Cheeseman, Parkinson, Clebsch, Kogias, Drossopoulou, Chisnall, Wrigstad, Liétar, OOPSLA2 Vol. 7, Oct 2023; https://www.microsoft.com/en-us/research/publication/when-concurrency-matters-behaviour-oriented-concurrency). In Verona, `when (a, b) { ... }` acquires cowns (concurrent owners) together. SEARCH-SUMMARY (https://2023.splashcon.org/details/splash-2023-oopsla/86/When-Concurrency-Matters-Behaviour-Oriented-Concurrency). For tick-lang, this solves the cross-zone trade/transfer problem without locks or two-phase messaging. The `happens-before` order derived from overlapping cown sets (CONCUR 2026 follow-up, SEARCH-SUMMARY) is a candidate for a deterministic cross-zone order.

**Weakness.** It is experimental and research-only, and the original code was archived to an `old_main` branch with a restart (repo README, VERIFIED). It has no production users (UNVERIFIED).

**Who.** Microsoft Research (Parkinson, Clebsch, Chisnall et al.) with Uppsala (Wrigstad) and Imperial (Drossopoulou). VERIFIED (author lists)

**Alive.** Yes. Commits are dated Nov 6, 2026 ("Carry Project Verona website into verona-bc history") and Oct 12, 2026 ("Keep moved cown regions alive during ownership transfer"). VERIFIED (https://github.com/microsoft/verona/commits/main)

### Inko
**Summary.** "Inko is a language for building concurrent software with confidence ... deterministic automatic memory management, move semantics, static typing, type-safe concurrency". It compiles via LLVM. VERIFIED (https://raw.githubusercontent.com/inko-lang/inko/main/README.md). Its processes "don't share memory, instead values are moved between processes". Only "sendable" values cross: value types, or unique `uni T` values, created by `recover`. "Mutable borrows are never sendable." Unique values "are the same as Pony's isolated values, just using a name we feel better". VERIFIED (https://raw.githubusercontent.com/inko-lang/inko/main/docs/source/getting-started/concurrency.md)

**Borrow.** Inko shows that Pony's `iso` idea can be simplified into ownership plus a single `uni` qualifier. That is a much smaller concept budget than Pony's six capabilities, and probably the right size for tick-lang messages.

**Weakness.** It has a small community (1.3k stars), and GitHub is only a "Read-only mirror of https://codeberg.org/inko-lang/inko". VERIFIED (https://github.com/inko-lang/inko)

**Who.** Yorick Peterse (UNVERIFIED here).

**Alive.** Yes. Tags v0.21.1 (Jul 20, 2026) and v0.20.0 (Apr 20, 2026). VERIFIED (https://github.com/inko-lang/inko/tags)

### Hylo (formerly Val)
**Summary.** "Hylo is a safe systems programming language leveraging mutable value semantics and generic programming." VERIFIED (https://github.com/hylo-lang/hylo-new)

**Borrow.** Mutable value semantics means no shared mutable references at all. Values are mutated in place through `inout`-style projections. `new_state = tick(state, inputs)` then reads as pure value semantics, but compiles to in-place mutation. This is the same conclusion as Koka's FBIP, reached through types instead of reference counting.

**Weakness.** The implementation was restarted. The old repo says "Outdated Implementation of Hylo" and "Most of our efforts now go towards the new compiler at https://github.com/hylo-lang/hylo-new". Its last commits (Oct 19, 2026) include "Outdated note in README before archiving". VERIFIED (https://github.com/hylo-lang/hylo-old, https://github.com/hylo-lang/hylo-old/commits/main). The new repo has 86 stars. VERIFIED (hylo-new page)

**Who.** Dave Abrahams and Dimi Racordon (UNVERIFIED here).

**Alive.** Yes, with a new compiler in progress.

### Vale: cautionary
**Summary and weakness.** Vale "uses the new generational references technique for memory safety" and planned "region borrow checking". The README now says "Vale is archived and no longer being worked on", naming a successor language "Valen". The repo has about 2.0k stars, and its last commits are three "Update README.md" commits on September 23, 2026. VERIFIED (https://github.com/ValeLang/Vale, https://github.com/ValeLang/Vale/commits/master)

**Lesson.** Promising region ideas died with a single-maintainer project before shipping. tick-lang should take the *idea* (region-scoped immutability for a phase of computation, such as "during a tick, last tick's state is a read-only region") and not depend on such a project.

**Who.** Evan Ovadia (UNVERIFIED).

---

## 6. Marginal or not relevant (briefly)

- **Austral**: "Systems language with linear types and capability-based security". "Linear capabilities enable fine-grained permissioned access to low-level facilities." VERIFIED (https://github.com/austral/austral). This is relevant only as a model for the `unsafe` escape hatch: make unsafe/FFI a *capability value* that must be passed in from the platform, not a keyword anyone can type. The last commit was Jul 28, 2025 (VERIFIED, https://github.com/austral/austral/commits/master), so it is slow or dormant.
- **Cone** (Jonathan Goodwin): pluggable memory-management strategies per reference type. This is conceptually close to Verona's per-region strategy (UNVERIFIED characterisation). The commits page showed merges dated November 7, 2026. VERIFIED (https://github.com/jondgoodwin/cone/commits/master). I did not assess it further.
- **Mojo** (Modular): a Python-family systems language for AI hardware. The repo README says the Mojo compiler source has been available "since August 2026". Lockfiles pin "Mojo 1.3.0.dev2026110705" (commit of Nov 7, 2026). VERIFIED (https://github.com/modular/modular, https://github.com/modular/modular/commits/main). Its relevance is low: it is aimed at kernels and accelerators, not deterministic simulation.
- **Carbon** (Google): "Carbon Language is currently an experimental project", "not ready for use", working toward 0.1. VERIFIED (https://github.com/carbon-language/carbon-lang). **Not relevant.** It is a C++ successor focused on interop, with no concurrency or determinism story.

---

## 7. Cross-cutting lessons for tick-lang

1. **Copy Roc's platform/app split as the core architecture.** Rules are a pure app. The Rust runtime is the platform, and it owns I/O, the clock, threads and the allocator. Futhark's "compile to a library, not an executable" says the same thing. Determinism and replay come from the type system, not from discipline.
2. **Pure semantics, in-place machine code.** Koka's Perceus/FBIP and Hylo's mutable value semantics both show that `state -> new state` can compile to in-place updates. Choose one mechanism. Unique ownership of the state per tick makes this trivial when compiling to Rust (`fn tick(state: State, ...) -> State` with moves).
3. **Halide-style algorithm/schedule separation, but auto-schedule by default.** Halide's own authors say hand-written schedules are "time-consuming" and the schedule space is "combinatorially large". A schedule must never change results.
4. **Declared data access is proven prior art (Bevy).** tick-lang's added value must be compile-time guarantees: determinism, a fixed order for conflicting writes, arena non-escape and bounded queues.
5. **Regions are the unifying memory idea (Verona).** Per-tick arena, per-zone state and per-message buffers are all regions with different lifetimes and strategies. Verona's paper uses arena regions per request in a server.
6. **Keep the message-ownership vocabulary small.** Pony's six reference capabilities hurt adoption, and its own docs cite a tiny ecosystem and no IDE. Inko reduced it to `uni T` plus value types. Gleam shows that typing must be designed in alongside the actor model, not retrofitted.
7. **Bounded by design.** Pony's unbounded mailboxes required a runtime backpressure system that still needs "intervention by the programmer" for external pressure. tick-lang's bounded queues with a declared overflow policy are the deterministic fix.
8. **Cross-zone atomicity needs a primitive.** Encore (bestow/atomic) and Verona (BoC `when` over several cowns) both found plain messages insufficient for multi-actor updates such as trades.
9. **SoA should be a layout choice, not a code rewrite.** Odin `#soa`, Zig `MultiArrayList` and Jai's (reportedly removed) `SOA` all keep AoS-style access syntax. Odin's `context.temp_allocator` "freed once per cycle/frame" is exactly the per-tick arena, but unchecked. tick-lang should make escape a compile error.
10. **Benchmarks: single-thread first.** HVM2's own paper admits "about 5x slower than GHC ... can grow to 100x". Scaling curves without a sequential baseline are a credibility trap.
11. **Project risk.** Encore (deprecated), Vale (archived), Lunatic (silent since Mar 2025) and Jai (closed for over a decade) show what kills languages: single maintainers, research-grant lifecycles, startups and no public release. Roc's Rust-to-Zig rewrite is a warning that the compile times of "compiles to Rust" matter.
