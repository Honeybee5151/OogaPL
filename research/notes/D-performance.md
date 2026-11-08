# D. Performance evidence for tick-lang's patterns

Compiled 2026-11-07/08. Each number below was read from the primary source: the PDF text or a rendered figure, the official blog, or the benchmark repo.

**Status labels**
- **VERIFIED**: I read the number in the primary source.
- **VERIFIED (fig.)**: I read it off a chart in the primary source. Treat it as ±1 point.
- **UNVERIFIED**: I could not fetch the primary source. The number comes from a search snippet or a secondary page, and the reason is given.
- **WRONG / MISLEADING**: the claim as usually stated does not match the source.

**Local copies.** Downloaded PDFs are in `/tmp/claude-0/-home-user-OogaPL/32372848-6606-541b-85af-54d683c4e916/scratchpad/research/pdfs/`. Extracted text is in `.../research/txt/`, rendered figures in `.../research/img/`, web pages in `.../research/html/`, and GitHub files in `.../research/gh/`.

**Network note.** This session's environment only allowed arXiv and GitHub at first. I added 25 exact hosts to the user's personal "Standard" environment through approval prompts: Google, Meta, USENIX, ACM, SRE, AWS, Rust, Chromium, ScyllaDB, university mirrors, EVE Online and others. Sources still unreachable:
- the original Berger et al. 2002 PDF (TLS failure upstream at umass.edu; the tufts.edu mirror resets the connection);
- the Accelerometer paper PDF (the Facebook CDN is blocked and dl.acm.org serves a bot challenge). I used the authors' ASPLOS talk slides instead.
- the Amazon Builders' Library page was fetched, but it contains no measured numbers.

---

## Summary verdict per pattern

| # | Pattern | Strength of evidence that it *pays off* | One-line reason |
|---|---|---|---|
| 1 | Deterministic fixed tick, I/O at edges | **Thin** (on performance) | No primary source measures the cost or benefit of determinism. The one vendor claim of "no noticeable impact" (Box2D) is UNVERIFIED. Rollback re-simulation cost is structural (N re-sim frames), and GGPO's docs give no numbers. |
| 2 | Share-nothing zones, one core each | **Moderate, with real counter-evidence** | Thread-per-core cut update tail latency by up to 71% vs Memcached (Enberg 2019), but the same paper says it is weaker when work does not spread across cores. Shinjuku shows partitioned queues (d-FCFS) lose badly under high-dispersion or heavy-tailed work. EVE Online's single-node hot spots needed "time dilation" down to about 5–10% speed. |
| 3 | SoA + auto-parallel rules | **Moderate but strongly workload-dependent** | An AoS→SoA study found up to 25% gain on O(N²) kernels, but up to 75% *degradation* on linear kernels once conversion is counted, and no gain on ARM. In EVE, physics simulation is only "5–10% of the load". |
| 4 | Per-tick/per-request arenas | **Strong that regions help; moderate on size of gain** | Gay & Aiken: regions up to 16% faster. Berger et al. 2002: regions up to 44%. The 2026 re-evaluation finds modern allocators have cut this to "at most 15%", while regions resist fragmentation (naive allocation slows down up to 2×). At fleet scale, allocation is roughly 6–9% of Google cycles (Kanev Fig. 4). |
| 5 | Bounded queues + load shedding | **Moderate (mostly qualitative)** | Google SRE and AWS give design guidance and worked examples, not measurements. SEDA has one measured experiment, and it rejected 98% of requests to keep 90th-percentile latency at 11.8 s against a 5 s target. |
| 6 | Versioned zero-copy validated messages | **Strong for speed; validation cost is the catch** | The validated zero-copy read in rkyv is about 7.8× faster than prost decode on the `log` dataset. But validation is about 45× the cost of an unvalidated read there, and FlatBuffers validation on strings costs about 1.4 ms vs 32 µs unvalidated. |
| 7 | Escape hatch | n/a | Not a performance claim. |
| — | Compile to C/Rust → LLVM | **Moderate** | Koka→C is within 10% of C++ `std::map` on rbtree, faster than OCaml, Haskell, Swift and Java on allocation-heavy benchmarks. No primary measurement of the compile-time or debug-info costs. |
| — | Routine PGO/AutoFDO/BOLT from live profiles | **Strong** | AutoFDO: 10.5% geomean, 85% of instrumented FDO's gain. BOLT: 2–15% across Meta services. Propeller: 1.1–8% over PGO+ThinLTO. Chrome PGO: up to 10% page loads. rustc PGO: 10–16% build-time reduction. |

---

## A. Datacenter tax (Kanev et al., ISCA 2015) and Meta follow-ups

Primary source: Kanev et al., "Profiling a Warehouse-Scale Computer", ISCA 2015 — https://www.comp.nus.edu.sg/~hebs/course/cs6284/papers/Kanev-isca15.pdf

| Claim | Number | Workload / baseline | Exact quote / location | Status |
|---|---|---|---|---|
| Headline datacenter tax | "nearly 30%" | Google fleet, >20k machines, 3-yr study | Abstract: *"This “datacenter tax” can comprise nearly 30% of cycles across jobs running in the fleet"* | VERIFIED |
| Tax in the body of the paper | **22–27%** | Fig. 4, 11 months of fleet profiles | §4: *"“tax cycles” consistently comprise 22-27% of all execution"*; Fig. 4 caption: *"22-27% of WSC cycles are spent in different components of “datacenter tax”."* | VERIFIED. Note that the abstract says "nearly 30%" while the body says 22–27%. Fig. 4 peaks around 28% in late Y1. |
| memcpy/memmove | **4–5%** of all cycles | fleet | *"Just the variants of these two library functions represent 4-5% of datacenter cycles."* | VERIFIED |
| Compression | **≈¼ of tax cycles**, about 5–6% of all cycles | fleet | *"Approximately one quarter of all tax cycles are spent compressing and decompressing data."* Fig. 4 band is about 5–6% | VERIFIED (text) / VERIFIED (fig.) |
| Memory allocation | **≈6–9%** of all cycles | fleet | Text gives no number (*"make up a substantial component of WSC computation (as seen by allocation in Figure 4)"*). Fig. 4 band runs ≈6→12.5 (Jan Y1) to ≈6→14.5 (Nov Y1). | VERIFIED (fig.) |
| Protobuf | **≈4–5.5%** | fleet | Fig. 4 band; text: *"Generated code is the majority of the protobuf portion in Figure 4."* | VERIFIED (fig.) |
| RPC | **≈2–3%** | fleet | Fig. 4 band; text: load balancing, encryption and failure detection are *"approximately a third of RPC tax cycles. The rest are taken up by simple data movement of the payloads."* | VERIFIED (fig.) |
| Hashing | **≈1.5%** | fleet | Fig. 4 band; text: *"a small, but consistent fraction of server cycles"* | VERIFIED (fig.) |
| Kernel (not counted as tax) | "almost a fifth" | fleet | *"WSC applications spend almost a fifth of their CPU cycles in the kernel (Figure 5)"* | VERIFIED |
| Method under-counts | — | — | Leaf-only attribution: *"This also guarantees that we always under-estimate the fraction of cycles spent in tax code."* | VERIFIED |

**Caveat.** Figure readings come from a 200-dpi render (`img/k5_tl.png`). The text itself never gives percentages for allocation, protobuf, RPC or hashing. Any write-up that quotes such numbers "from the text" is wrong. Quote them as approximate readings of Fig. 4.

**ISCA@50 retrospective (2023)** — https://sites.coecis.cornell.edu/isca50retrospective/files/2023/06/kanev_2015_profiling.pdf

- Exact wording: *"As just one example, multiple key services in Meta datacenters were shown to spend 50+% of their compute cycles in “tax” code [20]."* Reference [20] is *A. Sriraman and A. Dhanotia, "Accelerometer…", ASPLOS 2020.* **VERIFIED.**
- The retrospective also notes that Google's later work moved from "reduce time spent in datacenter tax" to "optimize tax routines holistically for application productivity" [6]. Reference [6] is the TCMalloc paper, Hunter et al. 2021. **VERIFIED.**

**Accelerometer (Sriraman & Dhanotia, ASPLOS 2020).** I could not fetch the paper PDF. These numbers come from the **authors' ASPLOS talk slides** at https://users.ece.cmu.edu/~asrirama/publication/asplos/accelerometer.pdf, read from rendered slides 12–14.
- Slide 13, "Orchestration overheads dominate across microservices". The application-logic share of cycles is roughly: Web ≈18%, Feed1 ≈60%, Feed2 ≈39%, Ads1 ≈59%, Ads2 ≈67%, Cache1 ≈31%, Cache2 ≈35%. Orchestration is therefore about 33–82%, and more than 50% for Web, Feed2, Cache1 and Cache2. **VERIFIED (fig., slides).**
- Slide 14 breaks out IO + IO pre/post + (de)compression + (de)serialization:
  - Web: 21+17+11+10 ≈ 59%
  - Cache1: 38+6+15+4 ≈ 63%
  - Cache2: 52+3+3+4 ≈ 62%
  - Google [Kanev'15] for comparison: 4+6+5 ≈ 15%

  **VERIFIED (slides).**
- Slide 12, "Memory" leaf functions (copy, allocation, free, compare): Web 37%, Feed1 8%, Feed2 20%, Ads1 28%, Ads2 28%, Cache1 26%, Cache2 19%; Google [Kanev'15] 13%. Kernel is 44% for Cache2. **VERIFIED (slides).** The commonly quoted "copying, allocating and freeing memory can consume 37% of cycles" is the Web service figure. That sentence reached me only via a search snippet of the abstract, so it is UNVERIFIED as abstract text.
- **Definition caveat:** Accelerometer's "orchestration" (I/O, compression, serialization, thread-pool management, logging…) is broader than Kanev's "tax". Comparing "50+% at Meta" with "22–27% at Google" mixes two category definitions.

**Relevance to tick-lang.** Allocation plus copying is about 10–14% of Google fleet cycles and up to 37% in one Meta service. Serialization and RPC are another 6–8% at Google and up to about 60% of cycles in Meta cache/web services when I/O is included. This is good motivation for patterns 4 (arenas) and 6 (zero-copy). But these are RPC microservices, not tick-based game servers. **No primary source profiles a game server's tax.** EVE's dev blog (section D) says its physics simulation was only 5–10% of server load in big fights.

---

## B. Profile-guided rebuilds (PGO / AutoFDO / BOLT / Propeller)

### BOLT — Panchenko et al., CGO 2019, https://arxiv.org/abs/1807.06735 (v2 PDF)

| Claim | Number | Workload / baseline | Quote | Status |
|---|---|---|---|---|
| Data-center apps | up to **8.0%** | Meta binaries; baseline is GCC + HFSort function reordering, plus LTO for HHVM | Abstract: *"BOLT achieves up to 8.0% performance speedups on top of profile-guided function reordering and LTO."* | VERIFIED |
| "5.4% average, up to 8% on HHVM" | **5.4% avg, 8.0% max (HHVM)** | 5 binaries: HHVM, TAO, Proxygen, 2× Multifeed | §6.1: *"BOLT's application resulted in a speedup, with an average of 5.4% and a maximum of 8.0% for HHVM."* Also: *"a comparison with FDO and AutoFDO was not possible."* | VERIFIED |
| Compilers | up to **20.4%** over FDO+LTO; up to **52.1%** without | GCC and Clang binaries | Abstract | VERIFIED |
| Clang details | 7.45% over GCC+PGO; 15.0% over Clang LTO+PGO | full build of Clang | §6.2.3 | VERIFIED |

**"2–15% across services"** comes from the Meta engineering blog, not the paper (https://engineering.fb.com/2018/06/19/data-infrastructure/accelerate-large-scale-applications-with-bolt/):
- *"BOLT optimizes placement of instructions in memory, thereby reducing CPU execution time by 2 percent to 15 percent."*
- *"Overall results range from 2 percent to 15 percent for different services, depending on the underlying CPU architecture."*
- *"the first BOLT-accelerated version of HHVM showed a 3 percent performance improvement … in many cases, it now accelerates HHVM by 8 percent."*

**VERIFIED** (blog). Note that the blog gives no per-service table.

### AutoFDO — Chen, Li, Moseley, CGO 2016, https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45290.pdf

- Abstract: *"The resulting performance has a geometric mean improvement of 10.5"* (sic). The published PDF truncates the sentence there; I confirmed this on the rendered page. The body says: *"The resulting binary is on average 10% faster than binaries optimized without AutoFDO."* **VERIFIED.**
- **85% of FDO's gains.** Intro: *"On benchmarks, AutoFDO achieves 85% of the gains of traditional FDO, but in practice many projects have switched to AutoFDO … and found performance to be equivalent"*. **VERIFIED.**
  - Table 1 (10 Google internal apps, speedup vs `-O2`): FDO mean **12.40%**, AutoFDO **10.52%**, ratio **84.84%**. Range: encoder 38% of FDO's gain; AI1 100%.
  - Table 2 (SPEC CPU2006 int): FDO mean 4.40%, AutoFDO 4.87%.
- FDO folklore: *"FDO … yielding improvements commonly in the 10-15% range and sometimes over 30%."* **VERIFIED.** This is the primary-source anchor for the "~10%" PGO folklore.
- Deployment: *"increased the number of FDO users at Google by 8X … Over half of CPU cycles used are now spent in some flavor of FDO-optimized binaries."* Profiling overhead: *"less than 1% overhead is observed"*; *"around 0.001% of observable cycles are profiled"*. **VERIFIED.**
- Staleness is handled by design (function name + line offset + discriminator). This is relevant for "routine rebuilds from live profiles".

### Propeller — Shen et al., ASPLOS 2023, https://snehasish.github.io/assets/pdf/shen-asplos23.pdf

- Abstract: *"An evaluation of internal warehouse-scale applications show Propeller improves performance by 1.1% to 8% beyond PGO and ThinLTO. Compiler tools such as Clang improve by 7% while MySQL improves by 1%. Compared to the state of the art binary optimizer, Propeller achieves comparable performance while lowering memory overheads by 30%-70% on large benchmarks."* **VERIFIED.**
- Practical caveat: *"BOLT optimized binaries failed at startup for all but one of the warehouse-scale applications. For Search, the BOLT optimized binary is 4% faster … Propeller … 3% faster."* **VERIFIED.** Robustness, not peak speed, decides which tool a "routine" pipeline can use.

### Other PGO numbers

| Source | Number | Workload / baseline | Quote | Status |
|---|---|---|---|---|
| Chromium blog 2020, https://blog.chromium.org/2020/08/chrome-just-got-faster-with-profile.html | "up to 10% faster page loads". Mac: responsiveness 3.9%, FCP 2.3%, Speedometer 2.0 7.7%. Windows: 7.3%, 3.5%, 11.4% | Chrome M85, Clang PGO vs non-PGO | *"pages loading up to 10% faster at the median"* | VERIFIED |
| Chromium blog 2016, https://blog.chromium.org/2016/10/making-chrome-on-windows-faster-with-pgo.html | "up to 15%". New-tab load 14.8%, first paint 5.9%, startup 16.8% | Chrome 53, MSVC PGO, Windows | *"make Chrome up to 15% faster on Windows"* | VERIFIED |
| Inside Rust blog 2020, https://blog.rust-lang.org/inside-rust/2020/11/11/exploring-pgo-for-the-rust-compiler/ | about 5% fewer instructions; **10–16%** wall-time reduction in build times | rustc + LLVM PGO vs non-PGO, rustc-perf suite | *"PGO seems to lead to a pretty solid 5% reduction of instruction counts"*; *"we get a 10-16% reduction of build times almost across the board for real world test cases"*; Firefox's PGO-built Clang gave *"up to 9% reduction of Firefox compile times"* | VERIFIED |
| Clang docs "up to 20%" | 20% | — | Only quoted in the Rust blog: *"Clang's own documentation even reports an up to 20% improvement"* | UNVERIFIED (secondary) |
| CPython `Doc/using/configure.rst` (GitHub main) | no number | — | *"Configuring Python using --enable-optimizations --with-lto (PGO + LTO) is recommended for best performance. The experimental --enable-bolt flag can also be used"* | VERIFIED (no number exists in the source) |

**Rust toolchain support.** The rustc book (https://doc.rust-lang.org/rustc/profile-guided-optimization.html) documents only **instrumented** PGO (`-Cprofile-generate` / `-Cprofile-use` plus `llvm-profdata`). It says an instrumented binary *"usually provides more accurate data and it is also what is supported by rustc"*. It does not document sample-based PGO from live `perf` profiles. **VERIFIED.**

`cargo-pgo` (https://github.com/Kobzol/cargo-pgo) wraps PGO and BOLT, but its README says *"BOLT and Docker support is currently experimental"*. **VERIFIED.**

**Implication.** "Routine AutoFDO from live profiles" is mature on the C/Clang path. On the Rust path it is instrumented PGO only, unless unstable flags are used. I have not verified whether rustc has an unstable sample-PGO flag.

**Strength: STRONG.** Several independent industrial sources give 2–15% gains, with sound methodology and named baselines.

---

## C. Arenas / regions

| Source | Number | Workload / baseline | Quote | Status |
|---|---|---|---|---|
| Gay & Aiken, PLDI 1998, "Memory Management with Explicit Regions", https://theory.stanford.edu/~aiken/publications/papers/pldi98a.pdf | unsafe regions up to **16%** faster; safe regions **5% slower to 9% faster**; memory **9% less to 19% more**; safety cost **negligible–17%**; one benchmark **24%** faster by grouping data | allocation-intensive C programs vs three malloc/free allocators and Boehm-Weiser GC | *"the safe region-based programs use from 9% less to 19% more memory … and are from 5% slower to 9% faster than the fastest allocator. Unsafe regions are never slower than other allocators and are up to 16% faster"* | VERIFIED |
| Berger, Zorn, McKinley, OOPSLA 2002 | regions up to **44%** faster; the other custom allocators are no better than Lea's allocator | 8 benchmarks vs dlmalloc | Restated in the 2026 follow-up: *"region allocators as the only custom allocation technique providing substantial improvements to execution time (up to 44%) compared to a state-of-the-art general-purpose allocator"* | VERIFIED via the same author's 2026 paper. Original PDF not fetchable. |
| Berger 2002: "six of eight … Lea as good or better"; memory retention up to **230%** | — | — | Search snippet of the abstract only | UNVERIFIED |
| **van Kempen & Berger, ISMM 2026**, "Reconsidering 'Reconsidering Custom Memory Allocation'", https://arxiv.org/abs/2605.17119 | region speedups now **at most 15%**. mimalloc-naive is slower than custom by 7.1% (197.parser), 14% (Blender geometry_nodes) and 15% (mudlle). Equal for 175.vpr and Clang. Per-class allocator over mimalloc only 2.3%. Under adversarial fragmentation, naive allocation slows **up to 2×** while regions are unaffected. | Modern HW, glibc / jemalloc 5.3.0 / mimalloc 2.2.5, Clang 21; adds Clang and Blender | *"reducing region allocator speedups from up to 44% in the original study to at most 15% in our evaluation … naïve allocation slows down by up to 2× under heap fragmentation, while region allocators are unaffected."* | VERIFIED |
| Protobuf C++ Arena guide (source: protocolbuffers.github.io repo, `content/reference/cpp/arenas.md`) | no numbers | — | *"Memory allocation and deallocation constitutes a significant fraction of CPU time spent in Protobuf code."* *"makes object allocation faster by reducing it to a simple pointer increment, and makes deallocation almost free … greater cache efficiency"*. Table: arena allocation and destruction *"Faster on average"*, but message moves are *"Sometimes a deep copy"*. Advises per-request granularity for servers. | VERIFIED (qualitative only) |
| TCMalloc/Temeraire, Hunter et al., OSDI 2021, https://www.usenix.org/system/files/osdi21-hunter.pdf | **+7.7%** RPS (mean CPU improvement) and **−2.4%** RAM over 8 apps. Fleet 1% experiment: **6%** fewer TLB-miss stalls, **26%** less fragmentation waste. Hugepage coverage 11.8→23% and 44.3→67.3%. | Google apps vs prior TCMalloc pageheap | Abstract. Also: *"Gains were not driven by reduction in the cost of malloc. Gains came from accelerating user code"*. In ads2, malloc cycles went 2.7%→3.5% while RPS rose 3.42%. | VERIFIED |
| mimalloc TR (Leijen et al. 2019), https://www.microsoft.com/en-us/research/uploads/prod/2019/06/mimalloc-tr-v1.pdf | **7%** and **14%** faster than tcmalloc and jemalloc on redis. Lean compiler: sharded free lists gave *"over 25%"* | redis, Lean | Abstract and §2 | VERIFIED |
| mimalloc README (GitHub, 2021-01-30 runs) | leanN **13%** faster than tcmalloc | 16-core AMD 5950x | *"there is a 13% speedup over tcmalloc"*. The README also warns the benchmarks *"do not measure the behaviour on very large and long-running server workloads"* | VERIFIED |
| nginx pools / APR pools measured gains | — | — | No primary measurement found | Not found |
| TCMalloc "temporal locality" (ASPLOS 2024 characterization paper) | — | — | Not on arXiv; ACM blocked | UNVERIFIED |

**Strength: STRONG that bulk-free regions are never worse and help most under fragmentation. MODERATE on the size of the gain.** Against a modern allocator (mimalloc/jemalloc), the speed gain is now 0–15%, not 44%. The TCMalloc paper suggests allocator *placement* (locality, hugepages) matters more than malloc cycles. That supports arenas for locality reasons. Tick-lang's per-tick arena is exactly the "lifetime known, bulk free" case where regions win.

---

## D. Thread-per-core / share-nothing zones

| Source | Number | Workload / baseline | Quote | Status |
|---|---|---|---|---|
| Enberg, Rao, Tarkoma, ANCS 2019, "The Impact of Thread-Per-Core Architecture on Application Tail Latency", https://penberg.org/papers/tpc-ancs19.pdf | update tail latency up to **71%** lower (modern HW) and **66%** (legacy). Read tail up to **20%** / **54%** lower with IRQ affinity configured. Default config: read up to **16%** / **9%**, update up to **67%** / **47%**. | Sphinx (custom partitioned thread-per-core KV) vs Memcached, 99th percentile | Abstract: *"reduces tail latency by up to 71% compared to baseline Memcached"*. Counter-point in the same paper: *"Sphinx's read and update tail latency are higher than Memcached's for a small number of concurrent connections. This shows a weakness of the thread-per-core model for workloads that do not distribute work to all cores."* Also: *"software-based request steering, such as message passing between threads used by Seastar and our prototype KV store, suffer from high overheads because of thread wake-ups"* | VERIFIED |
| ScyllaDB vendor benchmark (2021), https://www.scylladb.com/2021/08/24/scylladb-vs-cassandra-performance-results/ | **2x–5x** throughput; Cassandra 4.0 P99 **80%–2,200%** higher | ScyllaDB 4.4 vs Cassandra 4.0 / 3.11, 3× i3.4xlarge | *"ScyllaDB has 2x-5x better throughput than Cassandra 4.0"*; *"Cassandra 4.0 can achieve a better latency with 40 i3.4xlarge nodes than 4 i3.metal Scylla nodes"* when underutilized | VERIFIED as a **vendor** claim. It is not independent, and it confounds C++ vs JVM/GC with thread-per-core. |
| Seastar tutorial (GitHub `doc/tutorial.md`) | no numbers | — | Qualitative: *"Unscalable programming practices (such as taking locks) can devastate performance on many cores."* | VERIFIED (qualitative) |
| **Counter-evidence**: Shinjuku, Kaffes et al., NSDI 2019, https://www.usenix.org/system/files/nsdi19-kaffes.pdf | up to **6.6×** throughput, **88%** lower tail | RocksDB with point and range queries vs IX (partitioned d-FCFS) and ZygOS (work stealing) | *"d-FCFS is a poor option for heavy-tailed request distributions … Any long request blocks every short request assigned to the same queue in d-FCFS."* *"d-FCFS … suffers at moderate and high load as requests are not perfectly distributed across workers."* | VERIFIED |
| **Counter-evidence (games)**: EVE Online "Introducing Time Dilation", CCP Veritas, 2011-04-22, https://www.eveonline.com/news/view/introducing-time-dilation-tidi | game clock dilated *"down to 5% of real time or something"* at warp-in, then *"around 30%"*, then *"let's say 10%"*. Physics simulation is *"only … 5-10% of the load"* | single-node solar system with a large fleet fight | *"Going multi-threaded … increases how much load can be processed per second, but never as much as most people think."* | VERIFIED (dev blog; numbers are illustrative estimates, not a measured table) |

**Strength: MODERATE.** Partitioning clearly removes lock and cache-bouncing overheads, with up to 71% tail-latency cuts in one academic study. But both the academic evidence (Shinjuku, Enberg's own caveat) and the best-known game case (EVE: all players in one system means one hot node, handled by slowing time) show the weakness: **load imbalance, hot zones and cross-shard messaging cost**. Tick-lang needs a story for zone splitting or migration, or some degraded mode like TiDi.

---

## E. Struct-of-arrays / data-oriented design

| Source | Number | Workload / baseline | Quote | Status |
|---|---|---|---|---|
| Radtke & Weinzierl, "Annotation-guided AoS-to-SoA conversions and GPU offloading with data views in C++", arXiv 2502.16517 | up to **25%** faster (Intel) on quadratic kernels with enough particles per cell; **no speedup on ARM**; up to **75% slower** on linear kernels; **2–3×** after replacing branches with masking | SPH particle code; compiler-converted SoA vs AoS, *"All timings always comprise all conversion overhead"* | *"For the kernels with linear compute complexity, our compiler transformations introduce an overhead which can result in a performance degradation of up to 75% … For the quadratic kernels, we gain an moderate runtime improvement of up to 25% on the Intel platform … while the ARM chip shows no speedup."* *"If the data are continuous in memory right from the start, this performance gain evaporates."* | VERIFIED |
| Same authors, arXiv 2405.12507 (Clang annotation prototype) | — | SPH; AMD EPYC Genoa | *"Whether AoS or SoA is favourable depends on the execution context and algorithm step."* | VERIFIED (qualitative) |
| EVE Online TiDi blog | physics = **5–10%** of server load | real MMO fleet fight | see D | VERIFIED |
| Unity DOTS, Intel SoA guides, Bevy/ECS benchmark suites | — | — | Hosts blocked or repo not reachable; nothing fetched | UNVERIFIED / not fetched |

**Strength: MODERATE, workload-dependent.** The one rigorous primary study I could reach found SoA wins only for compute-heavy, vectorisable loops. It loses, up to 75%, when conversion or access overhead dominates in light linear loops, and gives no gain on one architecture. SoA also loses when a rule touches all fields of one entity (record-at-a-time access). That is a well-known effect, but I found no primary measurement of it here. Treat "SoA by default" as a tunable layout, not a guaranteed win. EVE's figure suggests entity simulation may not be the dominant server cost anyway.

---

## F. Bounded queues and load shedding

| Source | Content | Status |
|---|---|---|
| Google SRE book, "Addressing Cascading Failures", https://sre.google/sre-book/addressing-cascading-failures/ | Worked example, not a measurement: *"if the queue size is 10x the number of threads, the time to handle the request on a thread is 100 milliseconds. If the queue is full, then a request will take 1.1 seconds to handle"*. Guidance: *"it is usually better to have small queue lengths relative to the thread pool size (e.g., 50% or less)"*; *"Gmail often uses queueless servers"*. Example: 5% of requests that never complete leave the frontend able to serve only 19.6% of requests, an *"80.4% error rate"*. | VERIFIED (illustrative arithmetic, not measured) |
| Google SRE book, "Handling Overload", https://sre.google/sre-book/handling-overload/ | Rejection is not free: *"it's almost equally expensive to reject a request that requires a simple RAM lookup … as it is to accept and run that request"*; *"the backend can become overloaded even though the vast majority of its CPU is spent just rejecting requests!"* This motivates client-side adaptive throttling. A retry budget of *"10%"* limits growth. | VERIFIED (qualitative) |
| AWS Builders' Library, "Using load shedding to avoid overload" (now https://builder.aws.com/content/3Eun1EEyX6p2e3VYNyRLSJzLuMV/using-load-shedding-to-avoid-overload) | Goodput framing: *"Load shedding lets a server maintain its goodput … However, the act of shedding load isn't free, so eventually the server falls prey to Amdahl's law and goodput drops."* Warns: *"if a service is load shedding 60 percent of its traffic, the service's median latency might look pretty amazing even if its successful request latency is terrible"*. **No measured numbers or charts with data.** | VERIFIED (qualitative) |
| SEDA, Welsh, Culler, Brewer, SOSP 2001, https://www.sosp.org/2001/papers/welsh.pdf | Haboob (Java) vs Apache 1.3.14 and Flash with 1024 clients. Haboob response time mean 547 ms, max 3.8 s; Apache max *"over 93 seconds"*, Flash *"over 37 seconds"* (TCP backoff). Adaptive shedding with a 5 s target: *"90% of requests exhibiting a response time below 11.8 sec, and a maximum response time of only 22.1 sec. In this run, 98% of requests were rejected"*. The authors note the controller *"is unable to guarantee a response time below the target value"*. | VERIFIED |
| Little's law | Mathematical identity (L = λW). It needs no measurement. It is the reason bounded queues bound latency. | n/a |

**Strength: MODERATE.** The design consensus from Google, AWS and SEDA is strong. Published *measurements* are scarce and old (SEDA 2001), and the one measured shedding run shows the cost (98% rejection, latency target overshot). A game server shedding inputs is also different from a web service shedding requests. EVE chose time dilation rather than dropping work.

---

## G. Zero-copy serialization

**rust_serialization_benchmark** (djkoloski), README auto-generated 2026-11-05, rustc 1.101.0-nightly, Intel Xeon 6975P-C (4 vCPU) — https://github.com/djkoloski/rust_serialization_benchmark. The README warns: *"These benchmarks are a work in progress"*.

`log` dataset (HTTP request logs, *"small and contain many strings"*):

| Library | Full deserialize | Zero-copy read | Notes |
|---|---|---|---|
| serde_json 1.0.154 | 4.5987 ms | — | |
| prost 0.14.4 (protobuf) | 2.7265 ms decode | — | |
| postcard 1.1.3 | 1.9480 ms | — | |
| bitcode 0.6.9 | 1.4353 ms | — | |
| rkyv 0.8.18 | 1.4616 ms unvalidated / 1.8219 ms validated | **7.8456 µs unvalidated / 349.66 µs validated** | Access, validated: 340.35 µs |
| flatbuffers 25.12.19 | — | 32.220 µs unvalidated / **1.4387 ms validated** | Access: 1.1243 ns unvalidated vs 1.3955 ms validated |
| capnp 0.27.3 | — | 83.616 µs (validated on demand); 774.02 µs packed | Access 43.563 ns |

`mesh` dataset (one big array of triangles, mostly floats):

| Library | Result |
|---|---|
| rkyv | read 21.203 µs unvalidated / 19.555 µs validated; access validated 3.6742 ns. Validation is about free for plain-old-data. |
| flatbuffers | read 28.996 µs / 35.468 µs validated |
| prost | decode 9.5633 ms |
| serde_json | deserialize 68.771 ms |

(**VERIFIED**; the table values are copied from the README.)

Derived ratios, computed from the table above:
- `log`: a validated rkyv read (349.66 µs) is about **7.8×** faster than prost decode (2.7265 ms), about **13×** faster than serde_json, and about **4×** faster than bitcode.
- `log`: rkyv validation costs about **45×** an unvalidated read (349.66 vs 7.85 µs). For FlatBuffers it is about **45×** (1.4387 ms vs 32.2 µs).
- `mesh`: validation cost is negligible.

**The gain depends on data shape. Strings and many small objects make validation expensive.**

**FlatBuffers official benchmark** (`docs/source/benchmarks.md` in google/flatbuffers, master). The workload is *"about 10 objects containing an array, 4 strings, and a large variety of int/float scalar values … representative of game data"*, run on Windows 7, ×1 million iterations:
- Decode + traverse + dealloc: FlatBuffers **0.08 s** vs Protobuf LITE **302 s** vs RapidJSON **583 s** vs raw structs 0.02 s.
- Encode: 3.2 s vs 185 s vs 650 s.
- Wire size: 344 vs 228 bytes, so FlatBuffers is about 50% bigger than protobuf before compression.

**VERIFIED**, but this is an old, vendor-run benchmark, and its decode column does **not** include FlatBuffers' optional Verifier.

**Cap'n Proto** (`doc/index.md`): *"in benchmarks, Cap'n Proto is INFINITY TIMES faster than Protocol Buffers. This benchmark is, of course, unfair. It is only measuring the time to encode and decode a message in memory."* On safety: *"These accessors validate pointers before following them."* The project publishes no quantitative benchmark. **VERIFIED.** The "infinity" figure is a self-declared joke, not evidence.

**Strength: STRONG for read speed, moderate overall.** Validated zero-copy reads beat parse-then-use decoding by about 4–13× on string-heavy data and by orders of magnitude on POD arrays. But validation can cost about 45× an unvalidated read. Mandatory validation, which tick-lang requires, erases much of the "zero-copy" gain for string-heavy messages, and FlatBuffers wire size is larger than protobuf. Game-state deltas are mostly POD, which is the favourable case.

---

## H. Determinism costs

| Item | Evidence | Status |
|---|---|---|
| GGPO rollback (`doc/DeveloperGuide.md`, pond3r/ggpo) | Requirements: *"The game simulation must be fully deterministic"*; must *"load, save, and execute a single simulation frame without rendering"*; *"GGPO will call your advance frame callback many times during a rollback"*. Example: 4-frame transit with 1-frame delay gives *"a rollback is likely to be around 3 frames"*, i.e. roughly 3 extra simulation steps that frame. **No CPU-cost measurements.** | VERIFIED (qualitative) |
| Box2D v3 cross-platform determinism (box2d.org/posts/2024/08/determinism/) | Requires `-ffp-contract=off` (FMA off) and custom trig. Claims "no noticeable performance impact". | UNVERIFIED (box2d.org not allowed; search snippet only) |
| Cost of disabling FMA / fast-math, or of fixed-point | No primary measurement found | Not found |

**Strength: THIN.** There is no rigorous published measurement of determinism's CPU cost for game servers. The structural cost of rollback (re-simulating k frames, saving state each frame) is clear from GGPO's design but unmeasured. Tick-lang should plan to measure this itself.

---

## I. Compiling to C/Rust

| Source | Number | Workload / baseline | Quote | Status |
|---|---|---|---|---|
| Perceus TR (Reinking, Xie, de Moura, Leijen; PLDI 2021), https://www.microsoft.com/en-us/research/uploads/prod/2020/11/perceus-tr-v4.pdf | Koka (compiled **to C**, gcc 9.3) is *"within 10% of the performance"* of C++ `std::map` on rbtree | allocation-heavy benchmarks vs OCaml, Haskell, Swift, Java, C++ | *"the purely functional Koka implementation is within 10% of the performance of the in-place mutating algorithm in C++ (using std::map)"* | VERIFIED |
| Koka README benchmark chart (doc/bench-amd3600-nov-2020.png) | relative time (Koka = 1.00). rbtree: OCaml 1.26, Java 1.67, C++ **0.92**, Haskell 2.40, Swift 7.85. deriv: C++ 1.33, Java 0.86. nqueens: C++ **0.79**. cfold: Koka fastest (OCaml 5.05) | AMD 3600XT, Nov 2020 | README: *"evidence that the current Koka implementation of reference counting is viable and can be competitive and not as a direct comparison of absolute performance between languages"*. C++ often *"do[es] not reclaim memory at all"* in these benchmarks. | VERIFIED (fig.) |
| Nim, Chicken Scheme | — | — | Not fetched | Not found |
| Compile-time, debug-info-mapping, borrow-checker-as-target costs | — | — | No primary measurement found. Indirect: rustc PGO blog shows LLVM dominates opt builds (*"a full webrender-opt build will spend more than 80% of its time in LLVM"*). | VERIFIED (that quote) |

**Strength: MODERATE.** There is a good existence proof (Koka→C within 10% of C++, beating mature GC'd languages on allocation-heavy code). There is no primary evidence on the known costs (compile latency, source-level debugging through generated C/Rust, or fitting generated code to Rust's borrow checker). Those remain engineering risks, not measured facts.

---

## Claims to correct in the report

1. **"Nearly 30% datacenter tax"** is the abstract figure. The paper's own measured band is **22–27%**, and it under-counts by design. Per-component numbers other than memmove (4–5%) and compression (≈¼ of tax) are **figure readings only**.
2. **"Over 50% at Meta"**: the retrospective says *"multiple key services … 50+%"*, citing Accelerometer. Accelerometer measures **"orchestration"**, a broader category than Kanev's tax, and only some services exceed 50% (Web ≈82% orchestration; Feed1 and Ads2 ≈33–40%).
3. **BOLT "5.4% avg / 8% HHVM"** is the CGO paper (5 Meta binaries over HFSort). **"2–15%"** is the Meta blog, and the blog attributes the spread to CPU architecture.
4. **"AutoFDO ≈85% of FDO"** is correct (Table 1: 84.84%). The abstract's "10.5" is truncated in the published PDF.
5. **"Regions up to 44% faster"** dates from 2002. Against modern allocators it is now **at most 15%** (van Kempen & Berger, ISMM 2026).
6. **SoA "always faster"** is **wrong** as a blanket claim. One study measured up to 75% slower on linear kernels once conversion is counted.
