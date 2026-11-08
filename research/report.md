# tick-lang — Phase 1 research report

Date: 2026-11-08. Author: Claude (research for Miles). Status: **finished, waiting for Miles's go-ahead before Phase 2.**

## How to read this

Every claim carries one of these labels:

| Label | Meaning |
|---|---|
| **VERIFIED** | I (or a research agent) read it in the primary source: the paper, the official docs, the project's own repo, or the vendor's own blog. The link goes there. |
| **VERIFIED (fig.)** | Read off a chart in the primary source. Treat it as approximate. |
| **SNIPPET** | The primary page is known and linked, but this session could not open it (blocked or bot-protected). The claim comes from a search engine's excerpt of that page. Probably right, but not read in full. |
| **UNVERIFIED** | Only secondary sources or memory. Do not rely on it. |
| **WRONG** | The claim as given to me does not match the source. |
| **GUESS** | My own estimate or judgement. Not a measurement. |

Detailed evidence (exact quotes, dates and file paths) is in `research/notes/A–D`. This file is the summary.

**Access note.** The session's network was locked down at first. Miles approved several batches of extra hosts during the work. Epic's sites (`unrealengine.com`, `dev.epicgames.com`) still answer automated requests with HTTP 403. So the Epic and UE6 *roadmap* items are SNIPPET-level, but the Verse *language* items are VERIFIED from Epic's open-source Book of Verse on GitHub. Unity's manual (`docs.unity3d.com/Manual`) also returned 403.

---

## The short version

1. **Nobody has built this.** I found no language or system that combines all four of: determinism guaranteed by the compiler, automatic parallelism, built-in snapshot/rollback/state hash, and compiling to Rust or C for a standalone server not tied to an engine. The pieces exist separately (section 2.3).
2. **Most of the *machinery* already exists as Rust libraries.** These include Bevy ECS, GGRS rollback, rapier deterministic physics, bumpalo arenas, rkyv zero-copy messages, bounded channels and thread-per-core runtimes. What a library **cannot** do is *guarantee* the rules: no hidden clock, complete snapshots, no side channels between zones, deterministic maths only, bounded queues only, and safe message versioning. That guarantee is the real product (section 4.5).
3. **The speed case is weaker than the reliability case.** Against *expert* Rust, a compiler that emits Rust has no known way to be faster, except by routinely doing things experts often skip, such as profile-guided builds (10–15% is the typical documented gain). Against *average* code, the evidence for big wins is real but thin and mostly from other domains (section 3).
4. **Recommendation: GO, with a change of emphasis.** Pitch it as "a server that cannot desync, cannot leak and cannot fall over", with speed as a bonus. Build it as a small language that compiles to a Rust crate, sitting on a Rust runtime that is useful on its own. Prove it with one small, falsifiable prototype (section 5).

---

## 1. The earlier session's leads, re-checked

| Lead | Verdict | Evidence |
|---|---|---|
| **Verse: functions run as transactions that roll back** | **VERIFIED** | Epic's open-source *Book of Verse* (repo `verselang/book`, Epic authors, CC0): "Methods marked with `<transacts>` provide automatic rollback on failure." `<transacts>` is the default effect. [`docs/00_overview.md`](https://github.com/verselang/book), [`docs/13_effects.md`](https://github.com/verselang/book) |
| Verse: "… and re-run" | **PARTLY**: re-running is part of the *planned* distributed system, not today's language | SNIPPET: Epic's ["The road to Unreal Engine 6"](https://www.unrealengine.com/news/the-road-to-ue-6) describes pausing a transaction, moving data to another server and re-running it, as early prototype work. |
| Verse: most data immutable | **VERIFIED** | "Values in Verse are immutable by default." [`docs/05_mutability.md`](https://github.com/verselang/book) |
| Verse: execution is deterministic | **OVERSTATED** | The book promises determinism only for pure `<computes>` functions ([`13_effects.md`](https://github.com/verselang/book)). The Verse Calculus paper's "deterministic" means a property of logic programming, not lockstep-safe simulation ([DOI 10.1145/3607845](https://doi.org/10.1145/3607845), VERIFIED via Crossref abstract). I found no Epic claim of bit-identical cross-machine replay. |
| Verse: Epic plans to spread one program over many servers; not shipped | **SNIPPET** | Same UE6 post: "working toward a full distributed software transactional memory system"; today "Verse runs on a single thread". |
| UE6 early access late 2027 | **SNIPPET** | Same post: "targeting a Unreal Engine 6 Early Access release at the end of 2027". |
| Verse usable outside Fortnite/UEFN | **NO (as of now)** | VERIFIED: the Book's `DEVELOPMENT.md` requires "Access to fortniteMain repository"; only the docs are open. SNIPPET: Tim Sweeney promised an open-source compiler "possibly as soon as 2025" ([X, June 2024](https://x.com/TimSweeneyEpic/status/1800184322508587449)), and I found no sign it shipped. |
| Lead repo `UnrealVerseGuru/VerseProgrammingLanguage` | **WRONG as a source**: unofficial, stale | Its README says "not endorsed by Epic Games"; last commit 2023-04-07. VERIFIED ([repo](https://github.com/UnrealVerseGuru/VerseProgrammingLanguage)) |
| **Easel**: language for multiplayer games with built-in rollback | **VERIFIED** | [Easel docs](https://easel.games/docs/learn/multiplayer/rollback-netcode): "anything written in the Easel programming language is always deterministic"; "Easel incrementally snapshots the execution state of all behaviors". It is a hosted 2D browser platform; no public source or licence found. The lead HN link ([44000867](https://news.ycombinator.com/item?id=44000867)) is a comment in the "Show HN: Easel" thread (16 May 2025), where the maker (`BSTRhino`) explains why they built a custom language. |
| **SpacetimeDB**: database that is the game server; BitCraft; BSL | **VERIFIED, with detail** | Licence is BSL 1.1, changing to AGPLv3 (with linking exception) on 2031-11-05; some crates are Apache-2.0 ([`LICENSE.txt`](https://github.com/clockworklabs/SpacetimeDB)). Modules can be Rust, C#, TypeScript or C++. "Each database runs its transactions one at a time on a single thread, by design." The full BitCraft backend runs on it; BitCraft's server code is public (Apache-2.0, [BitCraftPublic](https://github.com/clockworklabs/BitCraftPublic)). |
| SpacetimeDB performance | **VERIFIED as a vendor claim** | [Their benchmark post](https://spacetimedb.com/blog/benchmarking): 303,920 ± 4,712 TPS (TypeScript) and 265,541 ± 940 TPS (Rust) on a bank-transfer workload, run by the vendor; the post itself acknowledges criticism. Not a game-tick workload. |
| **Rune**: deterministic JS on clients and server | **VERIFIED** | "your game's `logic.js` file will run on every client and on Rune's servers" ([docs source](https://github.com/rune/rune)). Determinism is enforced by an **ESLint plugin**, not a compiler. MIT licence; active (last commit 2026-08-27). |
| **Mun** quiet since 2024–25 | **PARTLY WRONG** | No release since v0.5.0 (2023-12-28), but maintenance commits resumed Aug–Oct 2026 after a 15-month gap. Essentially one maintainer. VERIFIED ([repo](https://github.com/mun-lang/mun)) |
| **Photon TrueSync** | **Discontinued** | SNIPPET: "development of Photon TrueSync & Thunder is ceased" ([Photon docs](https://doc.photonengine.com/truesync/current/reference/known-issues)). |
| **Rollback-Core for UE5** | **Exists, early, commercial** | SNIPPET: GregOrigin plugin, v0.3.0, "under heavy development" ([Fab listing](https://www.fab.com/listings/03aa010f-2413-4a98-91b1-4fa67d0aedd4)). The promised free GitHub version was not found. |
| *(new)* **Photon Quantum** | **VERIFIED: the closest commercial match** | "the FP struct (Fixed Point) completely replaces all usages of floats and doubles to ensure cross-platform determinism … Q48.16"; "Casts from float and double are marked as [Obsolete] errors and do not compile" ([fixed-point docs](https://doc.photonengine.com/en-us/quantum/current/manual/quantum-ecs/fixed-point)). Game state must be declared in its `.qtn` DSL, which generates serialization for "snapshots, game saves, killcam replays" and checksumming ([DSL docs](https://doc.photonengine.com/quantum/v3/manual/quantum-ecs/dsl)). It is Unity/C# only. |
| Domain languages won through constraints (Erlang, P4, eBPF) | **Not re-checked in depth** | These are well known but were not re-sourced in this round. UNVERIFIED as cited claims. |
| "No language combines all four properties" | **Holds** after a deliberate search | See section 2.3. |

---

## 2. Wider prior art

### 2.1 Languages: what to borrow, what went wrong

"Alive" dates come from the project's own repo or releases page, read 2026-11-07 (VERIFIED unless noted).

| Project | Who | Alive? | Borrow | What went wrong / warning |
|---|---|---|---|---|
| **Roc** | Richard Feldman + foundation | Yes, pre-0.1 | **Platform/app split.** The app is pure, and the platform (host code) does all I/O and memory allocation. "Roc's standard library does not include any effectful functions; they all come from the platform." VERIFIED ([tutorial](https://github.com/roc-lang/roc/blob/main/docs/mini-tutorial-new-compiler.md)) | Many years with no 0.1. Rewrote its compiler from Rust to Zig, partly because "Rust's compile times are slow" ([Feldman gist](https://gist.github.com/rtfeldman/77fb430ee57b42f5f2ca973a3992532f), VERIFIED). |
| **Halide** | MIT/Adobe origin, community | Yes (v22.1.0) | **Separate what to compute from how (schedule).** PLDI'13: "performance up to 5× faster than hand-tuned C, intrinsics, and CUDA implementations optimized by experts over weeks or months" from "simple Halide programs written in a few hours". VERIFIED ([paper PDF](https://people.csail.mit.edu/jrk/halide-pldi13.pdf)) | Hand-writing schedules is hard: "Manually exploring these options is time-consuming" ([arXiv 2012.07145](https://arxiv.org/abs/2012.07145), VERIFIED), hence a decade of autoscheduler work. **Ship a good automatic default.** |
| **Verona** | Microsoft Research + Uppsala + Imperial | Yes, research | **Regions.** Per-request regions use "arena allocation … fast deallocation of the entire region" in the paper's server example. **`when (a, b)`** grabs several owners atomically, a model for cross-zone trades. VERIFIED ([arXiv 2309.02983](https://arxiv.org/abs/2309.02983), [BoC paper](https://www.microsoft.com/en-us/research/publication/when-concurrency-matters-behaviour-oriented-concurrency)) | Experimental, restarted once, no production users (UNVERIFIED). |
| **Pony** | Sylvan Clebsch, community | Yes (0.78.1) | Reference capabilities make zero-copy messages race-free. | Its own "Why not Pony" page lists "Lack of high-quality 3rd party libraries", "Limited native tooling", "We don't have an IDE". Mailboxes are unbounded, so overload from outside "needs intervention by the programmer". VERIFIED ([why-not-pony](https://raw.githubusercontent.com/ponylang/ponylang-website/main/docs/discover/why-not-pony.md), [backpressure.pony](https://raw.githubusercontent.com/ponylang/ponyc/main/packages/backpressure/backpressure.pony)) |
| **Inko** | Yorick Peterse (UNVERIFIED) | Yes (v0.21.1) | Pony's idea cut down to one qualifier: only unique `uni T` or value types can be sent between processes. VERIFIED ([docs](https://raw.githubusercontent.com/inko-lang/inko/main/docs/source/getting-started/concurrency.md)) | Small community. |
| **Koka** | Daan Leijen (MSR) | Yes, "research language" | **Perceus / FBIP**: a pure `state → new state` compiles to in-place updates when the old state is not shared. Compiles to C. VERIFIED ([Perceus](https://www.microsoft.com/en-us/research/publication/perceus-garbage-free-reference-counting-with-reuse/)) | Not production-ready by its own README. |
| **Hylo** | (UNVERIFIED) | Restarted (new compiler) | Mutable value semantics: no shared mutable references at all. | Implementation restarted. |
| **Futhark** | DIKU, Copenhagen | Yes (0.28.2) | **Compile to a library the host calls**, not to an executable. VERIFIED ([usage docs](https://raw.githubusercontent.com/diku-dk/futhark/master/docs/usage.rst)) | No irregular (ragged) arrays, and game entities are irregular. |
| **Bend / HVM2** | Higher Order Co. (Victor Taelin) | Yes, pivoted to "Bend 2" | Automatic parallelism from program structure. | **Cautionary.** HVM2's own paper: single-threaded it is "about 5x slower than GHC, and this number can grow to 100x". VERIFIED ([HVM2.pdf](https://github.com/HigherOrderCO/HVM2/blob/main/paper/HVM2.pdf)). **Scaling is not speed: benchmark against fast single-threaded code.** |
| **Odin** | Ginger Bill (UNVERIFIED) | Yes | `#soa` arrays with normal access syntax; `context.temp_allocator` "freed once per cycle/frame" is a per-tick arena. VERIFIED ([overview](https://raw.githubusercontent.com/odin-lang/odin-lang.org/master/content/docs/overview.md)) | Nothing stops a pointer escaping the frame arena. tick-lang should make that a compile error. |
| **Zig** | Zig Software Foundation | Yes, moved to Codeberg | Explicit allocators, `ArenaAllocator`, `MultiArrayList` (SoA as a library). VERIFIED (std source on GitHub) | Pre-1.0, heavy churn. |
| **Jai** | Jonathan Blow | Closed beta (SNIPPET) | SoA as a type-level switch (early demos). | Reportedly dropped the SOA keyword for metaprogramming (UNVERIFIED, community sources). A decade without a public release. |
| **Gleam** | Louis Pilfold (UNVERIFIED), sponsors | Yes (v1.19.2) | Typed actor messages (`Subject(T)`). | Some Erlang/OTP features are "not possible to represent in a type safe way". Design types *with* the actor model, not after. VERIFIED ([gleam_otp](https://raw.githubusercontent.com/gleam-lang/otp/main/README.md)) |
| **Elixir** | José Valim | Yes (v1.20.5) | Types added gradually, as warnings, without breaking code. VERIFIED ([CHANGELOG](https://raw.githubusercontent.com/elixir-lang/elixir/v1.20/CHANGELOG.md)) | Took years; still warning-only. |
| **Lunatic** | Bernard Kolobara (SNIPPET) | **Dormant** since Mar 2025 | Wasm processes, each with its own heap and permissions. | A startup-backed runtime that stopped without an announcement. |
| **Unison** | Unison Computing | Yes | Code identified by hash, so message schemas could be identified by hash too. | Custom toolchain (code in a database). |
| **Encore** | Uppsala | **Deprecated** (Feb 2026) | Cross-actor atomicity ("bestow/atomic"). | Research-grant lifecycle; it has a published post-mortem. VERIFIED ([README](https://github.com/parapluu/encore)) |
| **Vale** | Evan Ovadia (UNVERIFIED) | **Archived** (Sep 2026) | Region borrow checking. | A single-maintainer project that ended. VERIFIED ([README](https://github.com/ValeLang/Vale)) |
| **Lingua Franca** *(new find)* | lf-lang org (UC Berkeley origin: UNVERIFIED) | Yes (commits Nov 2026) | **Closest academic match.** "Deterministic interaction", "executes transparently on multiple cores without compromising determinacy", compiles to C. VERIFIED ([repo](https://github.com/lf-lang/lingua-franca)) | No rollback or netcode. Its Rust target is marked unstable ([reactor-rs](https://github.com/lf-lang/reactor-rs)). |

### 2.2 Runtimes, ECS and rollback

| Project | Who / alive | Borrow | Warning |
|---|---|---|---|
| **Seastar** (C++, under ScyllaDB) | ScyllaDB; very active | `sharded<T>`: one instance per core; cores talk only by messages; fixed-size `pipe<T>`. VERIFIED ([tutorial](https://github.com/scylladb/seastar/blob/master/doc/tutorial.md)) | Share-nothing is "strongly discouraged" sharing, i.e. **a convention, not enforced by C++**. |
| **Tokio** | Tokio team; 1.54.1 (2026-10-21) | Bounded channels give backpressure; `LocalRuntime` (stable since 1.51.0) gives one runtime per core with non-`Send` tasks. VERIFIED ([CHANGELOG](https://github.com/tokio-rs/tokio/blob/master/tokio/CHANGELOG.md)) | Tokio's own argument against Seastar's model: "Unless the workload is entirely uniform, some processors will become idle while other processors are under load." VERIFIED ([2019 scheduler post](https://github.com/tokio-rs/website/blob/master/content/blog/2019-10-scheduler.md)) |
| **Monoio / Glommio** (Rust thread-per-core) | ByteDance (active) / Datadog (dormant since 2025) | `!Send` tasks make "owned by one core" a *type error*. VERIFIED ([monoio](https://github.com/bytedance/monoio)) | — |
| **Bevy ECS** | Bevy org; 0.20.1 (2026-10-28) | Runs systems in parallel from their declared data access. VERIFIED ([README](https://github.com/bevyengine/bevy/blob/main/crates/bevy_ecs/README.md)) | Conflict ("ambiguity") detection **defaults to `Ignore`**. Order of unordered systems is "nondeterministic". VERIFIED ([schedule.rs](https://github.com/bevyengine/bevy/blob/main/crates/bevy_ecs/src/schedule/schedule.rs)) |
| **flecs** | Sander Mertens; v4.2.0 | Inserts sync points automatically from read/write analysis; splits entities across threads by fixed ranges; queues structural changes per thread. VERIFIED ([Systems.md](https://github.com/SanderMertens/flecs/blob/master/docs/Systems.md)) | — |
| **Unity DOTS / Burst** | Unity | **The strongest industrial precedent for this project.** Burst compiles "a high performance subset of C# called High Performance C# (HPC#)". VERIFIED ([Burst docs](https://docs.unity3d.com/Packages/com.unity.burst@1.8/manual/csharp-hpc-overview.html)). In other words, Unity concluded a library wasn't enough and built a restricted language. | The job-safety checks are runtime, Editor-only, and "There is no protection against accessing static data from within a job". SNIPPET ([manual](https://docs.unity3d.com/es/current/Manual/JobSystemNativeContainer.html)) |
| **GGPO** | Tony Cannon; MIT; last commit 2019 | Contract: save / load / advance-one-frame, plus checksum and **SyncTest** (roll back one frame every frame and compare). VERIFIED ([DeveloperGuide](https://github.com/pond3r/ggpo/blob/master/doc/DeveloperGuide.md)) | All on the programmer: "The game simulation must be fully deterministic". |
| **GGRS / bevy_ggrs** | gschup; 0.13.0 / 0.22.0 (2026-06-26) | GGPO in safe Rust, with desync detection by checksum. | Its own **pitfalls guide** is a list of what a library cannot enforce: unstable query order, unregistered components "causing silent desyncs", `Local<T>` not saved, and more. VERIFIED ([pitfalls.md](https://github.com/gschup/bevy_ggrs/blob/main/docs/pitfalls.md)) |
| **Age of Empires lockstep** | Paul Bettner (byline), Gamasutra 2001 | "commands issued during turn 1000 would be scheduled for execution during turn 1002"; "Turns were typically 200 msec". VERIFIED ([article](https://www.gamedeveloper.com/programming/1500-archers-on-a-28-8-network-programming-in-age-of-empires-and-beyond)) | — |
| **Floating point** | Glenn Fiedler | — | Cross-machine float determinism is possible only "with the same compiler, run on machines with the same architecture, and perform some platform-specific tricks"; MotoGP replays could not move between Xbox and PC. VERIFIED ([article](https://gafferongames.com/post/floating_point_determinism/)). Rust's `+ - * / sqrt` are IEEE-exact on mainstream 64-bit targets (VERIFIED, [RFC 3514](https://github.com/rust-lang/rfcs/blob/master/text/3514-float-semantics.md)), but `sin`, `exp` and `powf` are documented as "non-deterministic" (VERIFIED, [std f32.rs](https://github.com/rust-lang/rust/blob/main/library/std/src/num/f32.rs)). |
| **rapier** `enhanced-determinism` | Dimforge; 0.36.1 | Bitwise-identical parallel physics "for any thread-pool size" since 0.35. VERIFIED ([CHANGELOG](https://github.com/dimforge/rapier/blob/master/CHANGELOG.md)). This proves deterministic parallel float physics is doable in Rust. | Needs discipline over which maths functions are called. |
| **EVE Online** | CCP | One star system = one node; overload is handled by **slowing game time** ("time dilation"), "down to 5% of real time or something". VERIFIED ([dev blog 2011](https://www.eveonline.com/news/view/introducing-time-dilation-tidi)) | Hot spots are the real enemy of "one zone, one core". |
| **SpatialOS** (Improbable) | Archived | — | **Cautionary tale.** It was a proprietary cloud runtime; the Unreal GDK needed a private engine fork; it was frozen in 2021. VERIFIED ([gdk-for-unity README](https://github.com/spatialos/gdk-for-unity)). Worlds Adrift could not be self-hosted after shutdown (UNVERIFIED, press). This supports "plain Rust output, no platform lock-in". |
| **Hathora** | — | — | Shut down May 2026 after an acquisition (UNVERIFIED, secondary only). Another hosted-platform risk. |

### 2.3 Has anyone combined it all?

The four properties: **P1** determinism guaranteed by the compiler, **P2** automatic parallelism, **P3** built-in snapshot/rollback/hash, **P4** compiles to Rust/C for a standalone server not tied to an engine.

| System | P1 | P2 | P3 | P4 |
|---|---|---|---|---|
| Lingua Franca | ✅ | ✅ | ❌ | ✅ C (Rust unstable) |
| Photon Quantum | partly (fixed-point types + DSL) | ❌ | ✅ | ❌ Unity |
| Easel | ✅ (claimed) | ❌ | ✅ | ❌ hosted |
| Verse | partly (pure functions only) | ❌ today | transactions only | ❌ UEFN |
| SpacetimeDB | ❌ | ❌ (one thread per DB) | transactions only | partly |
| Bevy + GGRS | ❌ (convention) | ✅ | ✅ (registered state only) | ✅ library |
| Rune | lint only | ❌ | ✅ | ❌ hosted |

**Nothing found has all four** (VERIFIED as "not found" after targeted searches of GitHub, HN, Reddit and arXiv; absence can never be fully proven). The gap is real but narrow. Lingua Franca shows the compiler side is doable; Quantum shows the game side is wanted.

---

## 3. Performance evidence for each pattern

Strength ratings are my judgement of the evidence (GUESS), but every number in the table is sourced.

| # | Pattern | Best measured gain (workload, baseline, source) | Strength |
|---|---|---|---|
| 1 | **The tick** (determinism, I/O at edges) | **No performance measurement found.** Its value is reliability: replays, rollback, tests. Cost of rollback = re-simulating k frames; GGPO gives "around 3 frames" as an example but no CPU numbers. VERIFIED ([GGPO guide](https://github.com/pond3r/ggpo/blob/master/doc/DeveloperGuide.md)). | **Thin** (on speed). Strong on reliability (Factorio, AoE, fighting games all ship it). |
| 2 | **Share-nothing zones** | Up to **71%** lower update tail latency, a thread-per-core KV store vs Memcached (Enberg et al., ANCS 2019, VERIFIED, [PDF](https://penberg.org/papers/tpc-ancs19.pdf)). The same paper admits "a weakness of the thread-per-core model for workloads that do not distribute work to all cores". **Counter-evidence:** Shinjuku shows partitioned queues failing on uneven work (up to 6.6× throughput for its alternative; VERIFIED, [NSDI'19](https://www.usenix.org/system/files/nsdi19-kaffes.pdf)); EVE's one-hot-node fights (above). ScyllaDB's "2x–5x over Cassandra" is a vendor number that also mixes in C++ vs Java (VERIFIED as vendor claim). | **Moderate, with real counter-evidence.** |
| 3 | **SoA entities + auto-parallel rules** | Tower-defence sim, 20k entities: OOP 36 FPS → SoA 66 → **parallel SoA 110** (Scala; one run; includes rendering). VERIFIED ([arXiv 2606.14919](https://arxiv.org/abs/2606.14919)). Most of the gain came from leaving objects and from parallelism, not from SoA layout itself. A compiler AoS→SoA study: up to **25% faster** on heavy kernels, up to **75% slower** on light loops once conversion is counted, no gain on ARM. VERIFIED ([arXiv 2502.16517](https://arxiv.org/abs/2502.16517)). EVE says physics is only "5-10% of the load" in big fights. | **Moderate, depends on the workload.** "SoA is always faster" is **WRONG**. |
| 4 | **Per-tick arenas** | Regions up to **44%** faster than a good allocator in 2002 (Berger et al.), but **at most 15%** against modern allocators in a 2026 re-run, where naive allocation "slows down by up to 2× under heap fragmentation, while region allocators are unaffected". VERIFIED ([arXiv 2605.17119](https://arxiv.org/abs/2605.17119)). Allocation is roughly 6–9% of Google fleet cycles (VERIFIED (fig.), Kanev 2015 Fig. 4). nginx/APR pools: **no measurement found**. | **Strong that it helps; moderate on how much** (0–15% speed, plus no fragmentation, plus no leaks). |
| 5 | **Bounded queues** | Mostly design guidance, not measurement: Google SRE ([cascading failures](https://sre.google/sre-book/addressing-cascading-failures/)) and [AWS](https://builder.aws.com/content/3Eun1EEyX6p2e3VYNyRLSJzLuMV/using-load-shedding-to-avoid-overload) (VERIFIED, qualitative). SEDA's one measured shedding run rejected 98% of requests and still missed its 5 s target (11.8 s at the 90th percentile). VERIFIED ([SOSP'01](https://www.sosp.org/2001/papers/welsh.pdf)). | **Moderate** consensus, **thin** measurement. The value is "falls over gracefully", not speed. |
| 6 | **Zero-copy validated messages** | On mk48.io game-update data: validating an rkyv archive takes ≈0.44 ms vs ≈1.9 ms to fully deserialize. On string-heavy logs, a validated rkyv read is ≈7.8× faster than protobuf (prost) decoding, but validation costs ≈45× an unvalidated read. On float arrays, validation is nearly free. VERIFIED ([rust_serialization_benchmark](https://github.com/djkoloski/rust_serialization_benchmark), 2026-11-05; maintained by rkyv's author). | **Strong** for game-like data (mostly numbers). |
| 7 | **Escape hatch** | Not a performance pattern. | n/a |
| — | **Compile to Rust/C** | Koka compiled to C runs "within 10%" of C++ `std::map` on its red-black-tree benchmark. VERIFIED ([Perceus TR](https://www.microsoft.com/en-us/research/uploads/prod/2020/11/perceus-tr-v4.pdf)). **No data found** on the costs: compile time, debugging generated code, fitting the borrow checker. | **Moderate.** |
| — | **Routine profile-guided rebuilds** | AutoFDO (sampled live profiles): **10.5%** mean on Google apps, 84.84% of instrumented FDO's gain; FDO "commonly in the 10-15% range" (VERIFIED, [CGO'16](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45290.pdf)). BOLT: **5.4%** average, 8.0% max (HHVM) over HFSort+LTO (VERIFIED, [CGO'19](https://arxiv.org/abs/1807.06735)); "2 percent to 15 percent" across services is Meta's **blog**, not the paper (VERIFIED, [blog](https://engineering.fb.com/2018/06/19/data-infrastructure/accelerate-large-scale-applications-with-bolt/)). Propeller: 1.1–8% beyond PGO+ThinLTO, and "BOLT optimized binaries failed at startup for all but one" of Google's apps (VERIFIED, [ASPLOS'23](https://snehasish.github.io/assets/pdf/shen-asplos23.pdf)). rustc's own PGO build: 10–16% faster builds (VERIFIED, [Rust blog](https://blog.rust-lang.org/inside-rust/2020/11/11/exploring-pgo-for-the-rust-compiler/)). **Catch:** rustc officially supports only *instrumented* PGO, not profiles sampled from a live server (VERIFIED, [rustc book](https://doc.rust-lang.org/rustc/profile-guided-optimization.html)). | **Strong** (10% is a fair typical figure), but "from live profiles" needs extra tooling on the Rust path. |

### 3.1 The "datacenter tax" claim, corrected

- Kanev et al. (ISCA 2015): the abstract says "nearly 30%", but the body measures **22–27%** of Google fleet cycles. Only two parts are stated in the text: memmove/memcpy **4–5%** of all cycles, and compression about **one quarter** of the tax. Other splits (allocation ≈6–9%, protobuf ≈4–5%, RPC ≈2–3%, hashing ≈1.5%) are **readings from Figure 4**, not stated numbers. VERIFIED / VERIFIED (fig.) ([PDF](https://www.comp.nus.edu.sg/~hebs/course/cs6284/papers/Kanev-isca15.pdf))
- "Over 50% at Meta": the 2023 retrospective says "multiple key services in Meta datacenters were shown to spend 50+% of their compute cycles in 'tax' code", citing Accelerometer (ASPLOS 2020). VERIFIED ([retrospective](https://sites.coecis.cornell.edu/isca50retrospective/files/2023/06/kanev_2015_profiling.pdf)). Accelerometer's category ("orchestration") is broader than Kanev's, so the two numbers are not directly comparable.
- **None of this profiles a game server.** It is RPC microservices. It is a good reason to care about allocation and serialization, not proof of what tick-lang would save.

### 3.2 The earlier session's guesses, checked

| Guess | Verdict |
|---|---|
| "2x or more against average server code" | **Plausible but UNVERIFIED for game servers.** The one ECS study shows ~3× (OOP → parallel SoA), on one Scala workload with rendering. The ScyllaDB-style numbers are vendor-run and confounded. Phase 3's benchmark must measure this. |
| "5–20% against expert hand-tuned Rust or C++" | **Probably too optimistic** (GUESS). Generated Rust goes through the same LLVM as expert Rust. tick-lang can only win where the expert *didn't bother*: routine PGO (~10%, sourced above), layout tuning, perfect parallel scheduling. An expert who does all of that should tie or win. **Expect roughly a tie against experts; any loss must be reported.** |
| "Most of the value is reliability, plus expert design by default" | **Supported.** Every strong source above is about reliability (desyncs, overload, leaks) or about making expert patterns the default. |

---

## 4. The honest case against

### 4.1 Where tick-lang would lose

- **Hot zones.** If every player piles into one ring, one core does all the work while the others sit idle. Tokio's team (VERIFIED) and EVE (VERIFIED) both hit exactly this. tick-lang needs zone splitting, or EVE-style slowdown, or it must accept the limit openly.
- **Irregular or "do-everything" game logic.** Inventories, scripts and AI with deep branching do not fit SoA or data-parallel rules well (Futhark's ragged-array limit and the AoS→SoA study's losses, VERIFIED above).
- **Anything that needs the existing ecosystem** in the tick: physics engines, pathfinding and AI libraries. Each one is either rewritten, or wrapped as "trusted deterministic", which weakens the guarantee.
- **Small games.** For a 4-player sumo bout, the CPU work is tiny. Speed gains will not show; only the reliability gains will.
- **Floats.** Guaranteeing cross-machine determinism means banning `sin`/`exp` from the standard library (VERIFIED: documented as non-deterministic) and supplying our own, or using fixed point as Quantum does. That is real work, and fixed point is awkward to write.

### 4.2 What Verse and SpacetimeDB already cover

- **Verse** covers transactions and rollback, immutability by default, and effect types, with client prediction (`<predicts>`) coming. But it is **locked to Fortnite/UEFN**, its compiler is not public, it runs **single-threaded today**, and its multi-server plan is early prototype work (SNIPPET). It is no use to a Godot developer on a Hetzner box. **It is the biggest threat** if Epic ships its open-source compiler as promised.
- **SpacetimeDB** covers persistence, transactions and a real MMO in production (BitCraft). But it is one thread per database "by design", it has no fixed tick, no determinism guarantee and no rollback netcode, and it uses the BSL licence. It solves the *MMO-backend* problem, not the *deterministic-simulation* problem. They could be complementary: tick-lang for the fight, a database for accounts.
- **Photon Quantum** is the closest in spirit (deterministic, rollback, checksums, a state DSL), but it is Unity/C#, not parallel, and not standalone.

### 4.3 The escape-hatch problem

- If people need the hatch often, the guarantees mean nothing. Unity's job system is the example: its safety checks have a documented hole for statics (SNIPPET).
- Rust's `unsafe` works because it is **rare and local**. tick-lang's hatch must be the same, *and* it has to cover effects, not just memory: "this code reads the clock", "this calls a Rust crate I promise is deterministic".
- My proposed rule (GUESS, for Phase 2):
  - Hatch code may **never** run inside a tick rule unless it is marked `trusted deterministic`, and SyncTest-style checks then run on it in every test build.
  - Everything else (network, disk, clock, logs) lives in the platform at the edges, as in Roc.
- The risk that remains: a "trusted" function that isn't deterministic. **The language can shrink this risk but cannot remove it.**

### 4.4 Ecosystem cost

| Need | Cost | Mitigation |
|---|---|---|
| **Editor support** | A language server (errors, go-to-definition) is a project of its own. Pony's own docs list "no IDE" as a reason not to use it (VERIFIED). | Start with error messages that point to tick-lang lines; add a basic LSP later. Keep the syntax small. |
| **Debugger** | You debug generated Rust. I found no data on how painful that is. As far as I know, Rust has no `#line` directive to map generated code back to the source (UNVERIFIED). | Make the generated Rust readable. Determinism helps a lot here: any bug can be **replayed exactly** from the input log, which most servers can't do. |
| **Libraries** | Almost none at the start. | Rust crates through the platform (the edges) for I/O; a small built-in maths/physics library for the tick. |
| **Compile times** | Rust compile times are slow, and Roc left Rust partly for this (VERIFIED). | Emit simple, macro-free Rust; measure it in Phase 3. |
| **People** | Projects with one maintainer die: Vale, Lunatic, Encore (VERIFIED). | Keep the scope tiny, keep the Rust runtime useful on its own, and write docs as we go. |

### 4.5 A Rust library or macro DSL instead? A serious comparison

| | **A. Rust library + lints** (Bevy/GGRS-style crate, Dylint custom lints, SyncTest in CI) | **B. Closed DSL inside a Rust proc macro** (`tick! { … }`: the macro parses the whole sub-language) | **C. Small language → generated Rust crate** (this proposal) |
|---|---|---|---|
| No hidden clock, RNG or I/O in rules | ❌ Lints catch literal calls only; they can't see inside helpers or dependencies, and `#[allow]` turns them off (VERIFIED, [clippy config](https://github.com/rust-lang/rust-clippy/blob/master/book/src/lint_configuration.md), [Rust reference](https://github.com/rust-lang/reference/blob/master/src/procedural-macros.md)) | ✅ inside the block (the macro owns the syntax) | ✅ |
| Complete snapshots (no "forgot to register" desyncs) | ❌ bevy_ggrs: unregistered state causes "silent desyncs" (VERIFIED) | ✅ | ✅ |
| Deterministic maths only | ❌ can't stop `f32::sin` | ✅ | ✅ |
| No side channels between zones | ❌ `static`, `Arc<Mutex>` and thread-locals all work | ✅ inside the block | ✅ |
| Fixed order for conflicting rules | ⚠️ Bevy can error on conflicts, but it is opt-in | ✅ | ✅ |
| Bounded-only queues | ❌ unbounded constructors are in the same crates | ✅ | ✅ |
| Message versioning checks | ❌ none built in (rkyv: "Schema evolution: no", VERIFIED) | ✅ | ✅ |
| Cargo, rust-analyzer, crates.io | ✅ free | ⚠️ works, but IDE help inside big macros is weaker (UNVERIFIED) | ❌ own tooling; output is a normal crate |
| Error messages | ✅ rustc's | ⚠️ the macro has to produce them | ✅ ours, as good as we make them |
| Syntax freedom | — | ❌ must be Rust tokens | ✅ |
| Effort to first useful version | **Low** (weeks; GUESS) | **Medium** | **Medium-high** |
| Share of the *mechanism* value | ~60–75% (GUESS by the research agent) | ~same as C | 100% |
| Share of the *guarantee* value | **Low**: it catches bugs in tests, not at compile time | **High** | **High** |

**Reading:**
- **A** is cheap and gives most of the *speed and machinery*, but almost none of the *guarantees*. Since the guarantees are the main value, A alone doesn't test the bet.
- **B and C give the same guarantees.** The only difference is packaging: B rides on Cargo for free but is stuck with Rust's token syntax and weaker macro tooling; C gets its own syntax and readable output files, but needs its own tooling.
- **Both B and C need the same runtime underneath**, and that runtime is basically A.
- Unity faced the same choice and built a restricted language (Burst/HPC#, VERIFIED). That is the strongest outside evidence that libraries were not enough.

---

## 5. Recommendation

### GO, with a change of emphasis

1. **Sell reliability first, speed second.** The evidence strongly supports "can't desync, can't leak, can't fall over under load, replay any bug". It supports speed gains only weakly. Claim speed only after Phase 3 measures it.
2. **Architecture: Roc-style split.**
   - The **game rules** are tick-lang: pure, deterministic, compiled to a Rust crate.
   - The **platform** is a hand-written Rust runtime: network, clock, threads, arenas, zones, snapshots.
   - The runtime is useful even if the language fails, so it is the fallback.
3. **Compiler front-end: standalone (C) for the prototype**, because generated Rust you can read is easier to debug. Keep the option of shipping it later as a `tick!{}` macro (B) for free Cargo integration. That is a packaging decision we can make later.
4. **Drop or defer:** automatic profile-guided rebuilds (rustc doesn't support live sampled profiles yet), cross-machine float determinism beyond x86-64/ARM64, and multi-machine zones. All of these are out of scope for v1.
5. **Add two things the research says are needed:**
   - a **hot-zone story**, either zone splitting or EVE-style slowdown;
   - **cross-zone transfers as a built-in atomic step**, modelled on Verona's `when`, so a player is never in two zones or in none.

### The smallest prototype that proves or kills the bet

**Workload:** N circles in a ring pushing each other (simple physics), 60 ticks/s, at 1k, 10k and 100k entities, with 1 zone and with 4 zones that hand players across.

**Three versions:**
- **(a) tick-lang** → generated Rust;
- **(b) idiomatic Rust** (Vec of structs, plain loops);
- **(c) expert Rust** (SoA, rayon or per-core zones, arenas).

**It passes if all of these hold** (thresholds are my proposal, GUESS; Miles decides):
1. **Determinism:** the same inputs give the same state hash over 5 runs, and across x86-64 and ARM64 if an ARM machine is available. A Hetzner ARM box is one option.
2. **The guarantee test:** a "desync zoo" of about 10 small bugs taken from the bevy_ggrs pitfalls list and GGPO's guide (clock reads, unsaved state, HashMap order, unbounded queue, cross-zone sharing, `sin`). tick-lang must **reject every one at compile time**. The same bugs written against the Rust library (A) are expected to compile, which is the point.
3. **Speed:** (a) must be **no slower than (b)** at 10k and 100k entities, and within **10%** of (c). Report p50/p99 tick time, ticks/s, memory and lines of code, with the spread over at least 5 runs on one machine.
4. **Size:** (a) should be clearly shorter than (c). If it isn't, the "expert design by default" claim fails.

**It fails (stop or pivot to the library) if:**
- the desync zoo can be blocked just as well by a ~500-line macro plus lint pack (GUESS at the size);
- or (a) is slower than idiomatic Rust;
- or writing the ring demo in tick-lang feels worse than writing it in Rust.

**Real-world check, after the toy works:** port the bout sim from Miles's Sumo Brothers repo (read-only). Optionally, run the generated Rust inside Godot through godot-rust ("Rust bindings for Godot 4", MPL-2.0, VERIFIED, [gdext](https://github.com/godot-rust/gdext)), so client and server can run the exact same rules. That would be a strong selling point for a Godot developer, but it is a GUESS that it works smoothly.

---

## Appendix: open items

- Epic's UE6 roadmap pages: re-read once the site can be fetched (403 to automated requests).
- Unity job-system "statics" hole: page returned 403; only a snippet was seen.
- Box2D's claim that cross-platform determinism has "no noticeable performance impact": snippet only (`box2d.org` not allowed).
- Berger et al. 2002 original PDF and the Accelerometer paper: numbers taken from the authors' own later paper and slides.
- Erlang/P4/eBPF "won through constraints": not re-sourced this round.
- No primary source measures the CPU cost of determinism (fixed point, no FMA) in a game. Phase 3 should measure it.
