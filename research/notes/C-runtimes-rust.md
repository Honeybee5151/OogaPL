# C. Runtime and engine prior art, and the Rust "library instead of a language" question

Research date: 2026-11-07. Scope: prior art for tick-lang's seven patterns: (1) deterministic fixed-tick `state + inputs -> new state`, (2) share-nothing zones, one core each, talking only by messages, (3) struct-of-arrays entity storage with rules parallelised automatically from declared data access, (4) per-tick arena memory, (5) bounded queues with deliberate load shedding, (6) versioned, zero-copy, validated network messages, (7) an `unsafe`-like escape hatch. Hot reload is covered too, and so is the central question: **could a Rust library or macro DSL deliver most of this?**

## 0. Method, labels and access limits

- **VERIFIED**: I read the primary source in this session (the project's own docs or source on GitHub, crates.io API metadata, arXiv, or the vendor's own page). The link points to that source.
- **SNIPPET**: the primary source is known and linked, but its host was blocked from this sandbox, so I saw the text only as a search-engine excerpt. Treat these as likely correct, but not read in full.
- **UNVERIFIED**: background knowledge or secondary write-ups only. Check before quoting.
- "Last release" dates come from the crates.io API (`/api/v1/crates/<name>`), which was fetched on 2026-11-07. GitHub activity dates come from GitHub pages fetched the same day.
- **Network limits in this session:** these hosts did not resolve and **must be allowed in the session's network settings** before the SNIPPET items can be upgraded: tokio.rs, docs.rs, scylladb.com, seastar.io, bevy.org, flecs.dev, gafferongames.com, gamedeveloper.com, factorio.com, docs.unity3d.com, gdcvault.com, without.boats, digitalcommons.calpoly.edu. I worked around tokio.rs by reading the same posts from the `tokio-rs/website` GitHub repo, and around docs.rs by reading crate source on GitHub.
- Numbers appear only with their workload, baseline and source. Where none was found, the text says so.

---

## 1. Thread-per-core / share-nothing runtimes

### 1.1 Seastar (C++), the framework under ScyllaDB

- **What it is:** a C++ framework for server applications. It runs **one thread per core**, each with a cooperative task scheduler. Memory is partitioned per core, and cores communicate only by explicit message passing (`smp::submit_to`). It also includes an optional userspace TCP stack. VERIFIED, from the [Seastar tutorial](https://github.com/scylladb/seastar/blob/master/doc/tutorial.md):
  > "Seastar programs use the share-nothing programming model, i.e., the available memory is divided between the cores, each core works on data in its own part of memory, and communication between cores happens via explicit message passing."
  >
  > "Unscalable programming practices (such as taking locks) can devastate performance on many cores."
- **Sharded services:** `seastar::sharded<T>` creates one instance of `T` per core and supplies start, invoke-on and stop machinery. This is the direct analogue of tick-lang's "zone owned by one core". VERIFIED, same tutorial, "sharded services" section.
- **Bounded pipes:** `seastar::pipe<T>` "has a fixed-size buffer ... the producer fiber blocks when it writes to a full pipe". VERIFIED, same tutorial.
- **Escape valve:** the tutorial notes that other threads *can* see each other's memory but are "strongly discouraged" from touching it. The share-nothing rule is a **convention enforced by discipline, not by C++**. VERIFIED, same tutorial. This is one gap a language, or Rust's `Send`/`!Send`, could close.
- **Who and status:** ScyllaDB. The README lists users: ScyllaDB, Redpanda and Ceph Crimson (VERIFIED, [README](https://github.com/scylladb/seastar)). Very active: the newest commit on `master` is dated 2026-11-05 (VERIFIED, [commits page](https://github.com/scylladb/seastar/commits/master)). Licence: Apache-2.0.

### 1.2 Measured evidence for thread-per-core

| Claim | Workload / baseline | Who measured | Status |
|---|---|---|---|
| ScyllaDB 4.4 showed 2x–5x the throughput of Cassandra 4.0 on the same 3-node cluster. A table shows max throughput of 80k/s for ScyllaDB 4.4.3, 40k/s for Cassandra 4.0 and 30k/s for Cassandra 3.11. With a P99 < 10 ms constraint, 3x–8x. Cassandra 4.0 P99 is "80% to 2,200% greater". | 3 × AWS i3.4xlarge, cassandra-stress-style load. The same report admits that 40-node Cassandra beats 4-node ScyllaDB on latency at low, under-utilised throughput. | **ScyllaDB (vendor)**, Aug 2021 | SNIPPET: [scylladb.com blog](https://www.scylladb.com/2021/08/24/scylladb-vs-cassandra-performance-results/), [white paper PDF](https://Scylladb.com/wp-content/uploads/wp-apache-cassandra-4-performance-benchmark-3.pdf) |
| Discord's message store: Cassandra p99 for historical reads was 40–125 ms; ScyllaDB p99 was a steady 15 ms. Writes: 5–70 ms vs a steady 5 ms. The cluster went from 177 Cassandra nodes to 72 ScyllaDB nodes. | Production migration by a user (Discord), 2023. **Confounded:** Discord also added Rust "data services" with request coalescing at the same time. | Discord (user, not vendor), retold by secondary sources | SNIPPET / secondary only: [Discord blog](https://discord.com/blog/how-discord-stores-trillions-of-messages), [ScyllaDB repost](https://resources.scylladb.com/blog/how-discord-stores-trillions-of-messages) |
| Thread-per-core KV store cut tail latency "by up to 71%" vs baseline Memcached. | Authors' own KV store with application-level partitioning and inter-thread messaging, against Memcached's multi-threaded shared-state design, on commodity Linux. The authors say the gains are limited by request steering and OS interfaces. | Enberg, Rao, Tarkoma (Univ. Helsinki), ACM/IEEE ANCS 2019 (academic, independent of vendors) | SNIPPET: the abstract was quoted in search results ([paper mirror](https://penberghtbprolorg-s.evpn.library.nenu.edu.cn/papers/tpc-ancs19.pdf)). The Datadog Glommio post cites the same 71% figure (VERIFIED below). |
| Independent ScyllaDB vs Cassandra comparison: ScyllaDB better on MVs, secondary indexes, LWT, CDC, caching | Benchmark-pipeline paper. Its abstract has no headline numbers. | Tyshchenko, Kouatly, Khan, *Applied Computer Systems* 2025 (independent) | SNIPPET: [DOAJ](https://doaj.org/article/1cec490eef0f460bad4ce044eb99e67e) |

**Reading of the evidence:** the strongest numbers come from the vendor. The one independent academic data point (ANCS'19) supports a *tail-latency* benefit. None of these isolate "thread-per-core" from "C++ vs JVM", "no GC" or "better I/O path". ScyllaDB's 2020 comparison was explicitly framed as C++ vs Java GC (SNIPPET, [ScyllaDB 2020](https://www.scylladb.com/2020/10/06/c-scylla-in-battle-royale-against-javas-zgc-shenandoah-g1-cassandra/)). The report should not cite ScyllaDB numbers as proof of the *architecture* alone.

### 1.3 Glommio (Rust, Datadog)

- **What it is:** a "Cooperative Thread-per-Core crate for Rust & Linux based on `io_uring`" with no helper threads (VERIFIED, [README](https://github.com/DataDog/glommio)). Datadog's introduction by Glauber Costa (a former ScyllaDB engineer), dated 2020-11-02, gives the rationale: "The biggest advantage of this model is that locks are never necessary". It also adds latency-classed task queues with proportional shares (VERIFIED, [Datadog blog](https://www.datadoghq.com/blog/engineering/introducing-glommio/)). The blog contains **no Glommio benchmark**. Its only number is the 71% cited from ANCS'19.
- **Alive status:** last crates.io release **0.9.0 on 2024-03-25**. Last commit on `master` was **2025-04-21** (VERIFIED, crates.io and the [commits page](https://github.com/DataDog/glommio/commits/master)). Not archived, but effectively dormant. A secondary source says the maintainer has left (UNVERIFIED).

### 1.4 Monoio (Rust, ByteDance)

- **What it is:** "A thread-per-core Rust runtime with io_uring/epoll/kqueue". Tasks need not be `Send`/`Sync` because "the data does not escape the thread on await points, unlike on work-stealing runtimes such as Tokio" (VERIFIED, [README](https://github.com/bytedance/monoio)). **This is the key Rust insight: the type system can encode "owned by one core" as `!Send`.**
- **Measured (vendor, 2021):** an echo-style network benchmark on ByteDance's production network, with client and server on separate machines. Server: Xeon Gold 5118 with 10 GbE X710, Linux 5.15, nightly-2021-11-26. Baselines: Tokio and Glommio. Results as written: "under 4 cores, the peak performance is about twice that of Tokio; under 16 cores, it is close to 3 times". With 1 core and few connections, Monoio has *higher* latency than Tokio (io_uring vs epoll). With 100-byte messages, Tokio shows "very little performance improvement or even degradation" with more cores. VERIFIED that these are the vendor's claims ([docs/en/benchmark.md](https://github.com/bytedance/monoio/blob/master/docs/en/benchmark.md)). Not independently replicated, and the numbers are in charts only.
- **Alive status:** last crates.io release **0.2.4 on 2024-08-20**. The repo is still committed to (newest commit **2026-10-15**, VERIFIED, [commits page](https://github.com/bytedance/monoio/commits/master)).

### 1.5 Tokio: work-stealing, and why it is not thread-per-core

- **Design:** a per-processor run queue with work stealing, a throttled "searching" state and batched sibling notification. The Tokio team's own argument against the sharded (Seastar) model, VERIFIED, from ["Making the Tokio scheduler 10x faster", 2019-10-13](https://github.com/tokio-rs/website/blob/master/content/blog/2019-10-scheduler.md), the source of tokio.rs/blog/2019-10-scheduler:
  > "This is the strategy used by Seastar. Because synchronization is almost entirely avoided, this strategy can be very fast. However, it's not a silver bullet. Unless the workload is entirely uniform, some processors will become idle while other processors are under load ... Most 'real world' workloads are not uniform. Because of this, general purpose schedulers tend to avoid this model."
- **Measured (Tokio's own, 2019):** new vs old scheduler microbenchmarks: `chained_spawn` went from 2,019,796 ns/iter to 168,854 ns/iter, and `ping_pong` from 1,279,948 to 562,659. Hyper "hello world" under `wrk -t1 -c50 -d10` went from 114.61k to 153.17k req/s, "an increase of 34%". Workload: Tokio's benchmarks. Baseline: the Tokio 0.1 scheduler. VERIFIED, same post.
- **Thread-per-core inside Tokio:** `LocalRuntime` ("capable of driving tasks which are not `Send + Sync`"; "cannot be moved between threads"; internally `current_thread`) was added in 1.41.0 (2024-10-22) and **stabilised in 1.51.0 (2026-04-03)** (VERIFIED, [CHANGELOG](https://github.com/tokio-rs/tokio/blob/master/tokio/CHANGELOG.md) and [local_runtime source](https://github.com/tokio-rs/tokio/blob/master/tokio/src/runtime/local_runtime/runtime.rs)). One `LocalRuntime` per pinned core is now a stable way to build share-nothing zones on Tokio.
- **Bounded channels / backpressure:** Tokio's tutorial (VERIFIED, [channels.md](https://github.com/tokio-rs/website/blob/master/content/tokio/tutorial/channels.md)):
  > "Unbounded queues will eventually fill up all available memory and cause the system to fail in unpredictable ways. Tokio takes care to avoid implicit queuing."

  The `tokio::sync::mpsc` docs say: "If the bounded channel is at capacity, the send is rejected and the task will be notified when additional capacity is available. In other words, the channel provides backpressure." (VERIFIED, [mpsc/mod.rs](https://github.com/tokio-rs/tokio/blob/master/tokio/src/sync/mpsc/mod.rs)). Tokio still ships `unbounded_channel`, so boundedness is opt-in.
- **Alive status:** tokio **1.54.1, 2026-10-21** (VERIFIED, crates.io).

**What tick-lang should borrow from section 1:**
- Seastar's `sharded<T>` shape: one owner per core and explicit cross-core calls.
- Monoio's use of `!Send` to make zone ownership a type error rather than a convention.
- Tokio's warning that sharding underuses cores when load is uneven. Zones therefore need a migration or rebalancing story (moving a zone between cores at a tick boundary), or tick-lang has to accept the trade explicitly.
- Tokio's "no implicit queuing" rule, made mandatory: no unbounded constructor at all.

---

## 2. ECS storage and scheduling from declared access

### 2.1 Bevy ECS

- **Parallelism from declared access (VERIFIED, [bevy_ecs README](https://github.com/bevyengine/bevy/blob/main/crates/bevy_ecs/README.md)):** "Bevy ECS can use function parameter types to determine what data needs to be sent to the system. It also uses this 'data access' information to determine what Systems can run in parallel". The multi-threaded executor "run[s] as many of them in parallel as possible".
- **Ambiguity detection (VERIFIED, [schedule.rs](https://github.com/bevyengine/bevy/blob/main/crates/bevy_ecs/src/schedule/schedule.rs)):** `ScheduleBuildSettings::ambiguity_detection: LogLevel` can be `Ignore`, `Warn` or `Error`. It **defaults to `LogLevel::Ignore`**. With `Error`, building the schedule fails on any unordered pair of conflicting systems. The setting is per schedule, and "child schedules ... do not inherit this modification" (VERIFIED, [nondeterministic_system_order example](https://github.com/bevyengine/bevy/blob/main/examples/ecs/nondeterministic_system_order.rs)). There is also a `debug`-feature `shuffle_seed`, which randomises system order within constraints to expose missing ordering edges. Its docs warn that the multi-threaded executor "can also produce orderings that are **not possible** in single-threaded execution".
- **Determinism caveats:**
  - The Bevy example says: "Unless the order is explicitly specified, their relative order is nondeterministic" (VERIFIED, same example).
  - The bevy_ggrs pitfalls guide says: "Bevy does not guarantee a stable iteration order for queries" (VERIFIED, [bevy_ggrs docs/pitfalls.md](https://github.com/gschup/bevy_ggrs/blob/main/docs/pitfalls.md)).
  - Bevy maintainers stated they "don't make a promise about execution order when there is system ordering ambiguity", and that this includes the single-threaded executor (SNIPPET, [PR #2504](https://github.com/bevyengine/bevy/pull/2504)). Tracking work on Bevy's internal ambiguities: [#4377](https://github.com/bevyengine/bevy/issues/4377). Some cross-crate ambiguities "cannot be resolved, since neither crate knows that the other exists" (SNIPPET, [PR #6341](https://github.com/bevyengine/bevy/pull/6341)).
- **Hot reload:** Bevy 0.17 shipped "initial integration of Dioxus subsecond for limited hotreloading" (SNIPPET, secondary: [AlternativeTo](https://alternativeto.net/news/2025/10/bevy-0-17-adds-raytraced-lighting-improved-observers-and-events-headless-ui-widgets-and-more)).
- **Alive status:** bevy / bevy_ecs **0.20.1, 2026-10-28**. Releases are roughly quarterly: 0.17.0 on 2025-09-30, 0.18.0 on 2026-01-13, 0.19.0 on 2026-06-19 and 0.20.0 on 2026-10-08 (VERIFIED, crates.io). Still pre-1.0, with breaking changes every release.

### 2.2 flecs (Sander Mertens, C/C++, with Rust bindings)

- **Storage (VERIFIED, [FAQ](https://github.com/SanderMertens/flecs/blob/master/docs/FAQ.md)):** "all entities with the same components are stored together in an archetype. This provides efficient CPU cache utilization and allows for things like vectorization". "System functions are called for each matching archetype ... it provides systems with direct access to component arrays." It cites Mertens's "Building an ECS #2: archetypes and vectorization" on Medium (not fetched; host blocked).
- **Pipeline / threading (VERIFIED, [Systems.md](https://github.com/SanderMertens/flecs/blob/master/docs/Systems.md)):**
  - The pipeline analyses which components systems read and write in order to **insert sync points automatically**.
  - During `progress()` the world is read-only, and structural changes are queued per thread ("staging"). That makes "multiple threads to iterate the same world without taking locks" safe.
  - Multithreaded systems are **data-parallel within a system**: "divides the number of matched entities across the threads ... the same entity is always processed by the same thread, until the next sync point" (e.g. 1000 entities over 4 threads gives 0..249, 250..499 and so on). This differs from Bevy's inter-system parallelism.
  - `immediate` systems are single-threaded escapes from read-only mode.
- **Relevance:**
  - The *deterministic slicing* of entities to threads, together with queued structural changes merged at sync points, is a good template for a deterministic tick.
  - Command-queue merge order must be defined. For example, merge by thread index, which is fixed by the slicing.
- **Alive status:** flecs **v4.2.0, released "04 Nov"** and marked Latest on the [releases page](https://github.com/SanderMertens/flecs/releases/tag/v4.2.0) fetched 2026-11-07. The page shows no year; it is almost certainly 2026, given the preceding v4.1.6 on 29 Jun. The Rust binding `flecs_ecs` (by Indra-db, not Mertens) was last released as 0.2.2 on 2025-11-17 (VERIFIED, crates.io).

### 2.3 Unity DOTS: job safety system, Burst, Entities, Netcode for Entities

The Unity docs host was blocked, so most items below are SNIPPET.
- **Job safety system:** "tracks what is reading and writing to any NativeContainer". If two scheduled jobs write the same `NativeArray`, it "throws an exception when the offending job is scheduled". `[ReadOnly]` lets jobs run concurrently. The checks are **runtime, Editor/Play Mode only**. "There is no protection against accessing static data from within a job", which "circumvents all safety systems". SNIPPET: [Unity manual, JobSystemNativeContainer](https://docs.unity3d.com/es/current/Manual/JobSystemNativeContainer.html), 2019.4-era text.
  - **Lesson:** the precedent most like tick-lang's "declared access" relies on **runtime checks in development builds** and admits a hole for statics. Rust's borrow checker plus Bevy's typed params give *compile-time* soundness for ECS data. Both, however, miss side channels.
- **Burst / HPC#:** Burst compiles a C# *subset*, "High Performance C#". It rejects managed-object methods, `catch`, writes to static fields (except SharedStatic), and foreach over generic `IEnumerable<T>` (SNIPPET, [Burst HPC# overview](https://docs.unity3d.com/Manual/burst/csharp-hpc-overview.html)).
  - **Lesson:** Unity concluded that a *library* on C# was not enough. It built a **compiler-enforced language subset**. This is the strongest industrial precedent for tick-lang's "restricted language" thesis.
- **Burst `FloatMode.Deterministic`:** "deterministic implementations of math functions" and some FP optimisations disabled. Subnormals flush to zero on all platforms (6.7 docs). It is 64-bit only, NaN bit patterns may vary, and determinism breaks if non-deterministic-mode function pointers are passed in or hardware intrinsics are used (SNIPPET, [Burst float precision and determinism](https://docs.unity3d.com/Manual/burst/float-precision-determinism.html)).
- **Netcode for Entities:** server-authoritative ghost replication with client prediction. On new snapshots, predicted ghosts reset `PredictionStartTick` to the snapshot tick. The server "will run 0 or more ticks in a frame ... much like the fixed update logic of physics" (SNIPPET, [prediction manual 1.3](https://docs.unity.cn/Packages/com.unity.netcode%401.3/manual/prediction.html)). It is **not lockstep**. It needs determinism only enough for prediction to converge, and corrects from server snapshots.
- **Measured DOTS vs GameObject numbers:** **none found with a clear workload and baseline**. Unity's own claim of "several to tens of times faster ... highly scene-dependent" is quoted only second-hand. UNVERIFIED; do not cite.

### 2.4 Measured SoA / ECS vs object-per-entity

| Result | Workload | Baseline | Source / status |
|---|---|---|---|
| Average FPS, uncapped: OOP 36.13, AoS-ECS 61.51, archetype SoA 65.81, SoA parallel 109.54. Capped at 60: OOP 25.49, AoS 47.71, SoA 48.76, SoA-PAR 58.54 | Tower-defence simulation: max 20,000 entities and 15,000 enemies, enemy spawn every 0.05 s, turret fire every 0.02 s. i7-12650H with 32 GB DDR5; rendering on an RTX 4060. Implemented in **Scala** on LibGDX/LWJGL. One representative run, "minimal variance". | Object-per-entity OOP ("each object ... encapsulates its data and methods") | VERIFIED: Tasnim & Zhao, "The Essence of Entity Component System", [arXiv:2606.14919](https://arxiv.org/abs/2606.14919) (12 Jun 2026; SAC '26 per search) |
| ecs_bench_suite: Rust ECS vs Rust ECS (simple_insert with 10k entities, simple_iter, fragmented_iter, schedule, heavy_compute, add_remove, serialize) | Results as violin charts only; no OOP baseline | — | VERIFIED: [repo](https://github.com/rust-gamedev/ecs_bench_suite), **archived 2022-11-27** |
| Mike Acton, "Data-Oriented Design and C++", CppCon 2014 | Keynote. Principle: "the transformation of data is the only purpose of any program". **No benchmark with a stated baseline was found.** | — | SNIPPET: [isocpp.org listing](https://isocpp.org/blog/2015/01/cppcon-2014-data-oriented-design-and-c-mike-acton) |

**Reading of the evidence:** in the one measured study, most of the gain comes from *leaving object-per-entity* (36 to 61 FPS). The step from AoS to SoA layout adds little single-threaded (61 to 66). **Parallelism contributes the largest step** (66 to 110). For tick-lang, "auto-parallelise from declared access" is probably the bigger lever, and the SoA layout itself the smaller one. The study is single-workload, in Scala/JVM, and includes rendering, so it should be weighted lightly.

### 2.5 Other Rust ECS crates

- **hecs** (Ralith): a "library, not a framework". There is no system scheduler, and queries are plain loops. Last release **0.11.2, 2026-10-04** (VERIFIED, [README](https://github.com/Ralith/hecs) and crates.io). A good base if tick-lang wants its *own* scheduler.
- **legion** (Amethyst): last release **0.4.0, 2021-02-25** (VERIFIED, crates.io). The GitHub repo page fetched 2026-11-07 shows **no archived banner** (VERIFIED, [repo](https://github.com/amethyst/legion)). It is not archived, but it is dormant (no release in over 5 years), so do not build on it.

**What tick-lang should borrow from section 2:**
- Bevy's "access inferred from the function signature".
- Ambiguity-as-error. Bevy makes it opt-in; tick-lang should make it **mandatory**.
- flecs's automatic sync-point insertion, read-only world with per-thread command queues, and deterministic entity-to-thread slicing.
- Burst's lesson that a compiler-enforced subset is what makes these guarantees real.

---

## 3. Deterministic lockstep, rollback and floating point

### 3.1 GGPO (Tony Cannon)

- **Open-sourcing:** Cannon announced on 2019-10-09 that GGPO was open source under the MIT licence (SNIPPET, [Wikipedia](https://en.wikipedia.org/wiki/GGPO) and [Game Developer](https://gamedeveloper.com/game-platforms/devs-can-now-use-the-rollback-networking-sdk-ggpo-for-free-in-their-own-projects)). The README states "GGPO is available under The MIT License" (VERIFIED, [repo](https://github.com/pond3r/ggpo)).
- **Requirements on game code** (VERIFIED, [DeveloperGuide.md](https://github.com/pond3r/ggpo/blob/master/doc/DeveloperGuide.md)):
  - "The game simulation must be fully deterministic ... advancing the game state by exactly 1 frame must result in identical game states for all players."
  - Callbacks: `save_game_state` (returns buffer, length, **checksum** and frame), `load_game_state`, `advance_frame` and `free_buffer`.
  - "your game state should be determined solely by the inputs".
  - The RNG seed must match and the RNG state must be part of the game state.
  - Avoid wall-clock time, or pass one player's time as an input.
  - **SyncTest**: "execute a 1 frame rollback for every frame ... raises an error if they differ". "By running synctest on developer systems continuously ... you can identify desync causing bugs immediately".
- **Alive status:** last commit on `master` was **2019-11-07** (VERIFIED, [commits page](https://github.com/pond3r/ggpo/commits/master)). It is dormant and Windows-only to build, per the README.

### 3.2 GGRS, bevy_ggrs and backroll (Rust)

- **GGRS** (Georg Schuppe, `gschup`): a "reimagination of the GGPO network SDK written in 100% safe Rust". Callbacks are replaced by a returned list of requests: `GgrsRequest::{SaveGameState, LoadGameState, AdvanceFrame}`, and "Handling them is mandatory". It offers `DesyncDetection` "by comparing checksums between peers" plus a `DesyncDetected` event, and `SyncTestSession` (VERIFIED, [README](https://github.com/gschup/ggrs) and [src/lib.rs](https://github.com/gschup/ggrs/blob/main/src/lib.rs)). The README says GGRS "is fairly stable". Last release **0.13.0, 2026-06-26** (VERIFIED, crates.io).
- **bevy_ggrs:** runs a dedicated `GgrsSchedule` and snapshots "only the components and resources you register". Last release **0.22.0, 2026-06-26** (VERIFIED). Its [pitfalls guide](https://github.com/gschup/bevy_ggrs/blob/main/docs/pitfalls.md) (VERIFIED) is effectively **a list of guarantees a library cannot enforce**:
  - unstable query iteration order ("Always sort queries that mutate state");
  - Bevy `Messages<T>` are not snapshotted;
  - `Local<T>` is not snapshotted;
  - reading `ButtonInput` directly instead of `PlayerInputs`;
  - **unregistered components and resources "causing silent desyncs"**;
  - `Entity` IDs change across rollback (needs `MapEntities`);
  - `GlobalTransform` is updated outside the rollback schedule;
  - change detection fires on every restore.
- **backroll** (HouraiTeahouse): pure-Rust GGPO, async, "still in an early beta stage". Last release **0.6.0, 2022-11-13** (VERIFIED, [README](https://github.com/HouraiTeahouse/backroll-rs) and crates.io). Dormant.

### 3.3 Overwatch (Tim Ford, GDC 2017)

- "'Overwatch' Gameplay Architecture and Netcode": Overwatch uses an ECS architecture, and the talk covers "how this networked simulation works, and how it leverages determinism to achieve responsiveness and precision" (SNIPPET, [GDC Vault](https://gdcvault.com/play/1024001/-Overwatch-Gameplay-Architecture-and)).
- Secondary claim: only three systems (movement, weapon, state script) handle gameplay netcode, in a server-authoritative design with heavy prediction (UNVERIFIED, [Edgegap write-up](https://edgegap.com/blog/game-backend-deep-dive-overwatch-2016-netcode-architecture-rollback)). Commonly retold details (stateless systems, "singleton components", component and system counts) are UNVERIFIED here; check them against the talk video.

### 3.4 Age of Empires: "1500 Archers on a 28.8" (Terrano & Bettner, 2001)

- Commands are tagged to run **two "communications turns" later**: commands issued on turn 1000 execute on turn 1002. "Turns were typically 200 msec in length." The comm-turn length adapts: it rises quickly with latency and settles slowly. Design target: 8 players on a "16MB Pentium 90 with a 28.8 modem" (SNIPPET, [Game Developer / Gamasutra, 2001-03-22](https://www.gamedeveloper.com/programming/1500-archers-on-a-28-8-network-programming-in-age-of-empires-and-beyond)).
- The authors are usually given as Mark Terrano and Paul Bettner. My search excerpt named Terrano in the text but did not show the byline; one slide deck mirror credits a different academic group, which reflects who presented the slides, not who wrote the article. Treat the byline as UNVERIFIED until the page is read.

### 3.5 Floating-point determinism

- **Glenn Fiedler, "Floating Point Determinism":** cross-machine determinism is possible "provided you use an executable built with the same compiler, run on machines with the same architecture, and perform some platform-specific tricks". AMD and Intel differed on transcendental functions. MotoGP replays could not be shared between Xbox and PC. Fiedler considers the question "by no means settled" (SNIPPET, [gafferongames.com](https://gafferongames.com/post/floating_point_determinism/)).
- **Rust today (VERIFIED):**
  - RFC 3514 (float semantics, accepted; [text](https://github.com/rust-lang/rfcs/blob/master/text/3514-float-semantics.md)) specifies that primitive float operations "generally produce results that exactly match IEEE 754-2008". The non-determinism is limited to **NaN bit patterns**: "if you never use `to_bits`, `copysign`, `is_sign_negative`, or `is_sign_positive` on a NaN ... your code will not be able to observe any non-determinism".
  - Rust does **not** fuse `a*b+c` into FMA on its own; the RFC lists FMA contraction as a forbidden transformation under strict IEEE.
  - Known exceptions: 32-bit x86 using x87 (i586/i686) and old MIPS.
  - **Transcendentals are explicitly non-deterministic.** The std docs for `powi`, `powf`, `exp` and others say: "The precision of this function is non-deterministic. This means it varies by platform, Rust version, and can even differ within the same execution from one invocation to the next" ([library/std/src/num/f32.rs](https://github.com/rust-lang/rust/blob/main/library/std/src/num/f32.rs)).
  - **Consequence:** Rust `+ - * / sqrt` are cross-platform deterministic on mainstream 64-bit targets. `sin`, `exp`, `powf` and so on must be replaced by a pinned software implementation, such as the `libm` crate or fixed-point. A library can *provide* those, but cannot *stop* a system calling `f32::sin`.
- **rapier `enhanced-determinism`:** the feature maps to `simba/libm_force` and `parry3d/enhanced-determinism` (VERIFIED, [rapier3d Cargo.toml](https://github.com/dimforge/rapier/blob/master/crates/rapier3d/Cargo.toml)). Since v0.35.0-beta.0 (2026-08-02) it "can now be combined with `parallel`, with results bitwise identical for any thread-pool size, and costs no measurable performance anymore". Cross-platform determinism "rests on glam's shared scalar core plus `libm`-pinned transcendentals". It is still incompatible with `simd8` and needs matching numeric features. v0.35.0 (2026-08-08) canonicalised signed zeros in contact impulses, "removing a source of cross-platform divergence" (VERIFIED, [CHANGELOG](https://github.com/dimforge/rapier/blob/master/CHANGELOG.md)). Last release rapier3d **0.36.1, 2026-10-10** (crates.io). This is an existence proof that **deterministic, parallel float physics is achievable in Rust as a library**, given discipline over math functions.
- **Photon Quantum:** the `FP` struct "completely replaces all usages of floats and doubles to ensure cross-platform determinism". It is Q48.16 (48 integral bits, 16 fractional), with FPVector2/3, FPMatrix, FPQuaternion and RNGSession (SNIPPET, [Quantum fixed-point manual](https://doc.photonengine.com/en-us/quantum/current/manual/quantum-ecs/fixed-point)). The session config has `ChecksumCrossPlatformDeterminism` (SNIPPET, API docs).
- **Factorio:**
  - Multiplayer uses deterministic lockstep, sending only inputs. Desync reports are produced for debugging (SNIPPET, [FFF #188](https://factorio.com/blog/post/fff-188); deep dive [FFF #76](https://factorio.com/blog/post/fff-76)).
  - "Multithreading is hard, deterministic multithreading even more so" (SNIPPET, [FFF #415](https://direct.factorio.com/blog/post/fff-415)).
  - A third-party page cites FFF #446 (Sept 2026) saying Wube had never seen an ARM64 vs x86 desync (UNVERIFIED).

**What tick-lang should borrow from section 3:**
- GGPO's three-function contract (save, load, advance) and checksums, generated *by the compiler* for all state, not just "registered" state.
- SyncTest-style "re-simulate every tick and compare" as a built-in test mode.
- AoE's scheduling of inputs for tick N+k.
- Deterministic math made the *only* math available (Quantum's FP, rapier's libm pinning). Sorted or stable iteration by construction.

---

## 4. Rust building blocks mapped to tick-lang's seven patterns

All dates are the newest crates.io release (VERIFIED, crates.io API, 2026-11-07).

| tick-lang pattern | Existing Rust pieces (last release) | Covers | Gap a library leaves |
|---|---|---|---|
| 1. Deterministic fixed tick `state+inputs→state` | bevy_ecs 0.20.1 (2026-10-28) fixed schedules; ggrs 0.13.0 (2026-06-26); bevy_ggrs 0.22.0 (2026-06-26); rapier `enhanced-determinism`; lightyear 0.31.0 (2026-10-21) has a `crates/deterministic` replication crate ([README](https://github.com/cBournhonesque/lightyear)) | Tick loop, rollback, checksums, deterministic physics | Purity of systems (no clock, OS RNG, I/O, `HashMap` order, statics, `Local`), complete snapshots, and stable iteration order are all **convention** (see the bevy_ggrs pitfalls). |
| 2. Share-nothing zones per core + messages | monoio 0.2.4 (2024-08-20); glommio 0.9.0 (2024-03-25, dormant); tokio 1.54.1 `LocalRuntime` (stable since 1.51.0); core pinning crates | One runtime per core; `!Send` tasks make cross-core sharing a **type error** | Nothing stops `static`, `Arc<Mutex<_>>` or `thread_local!` side channels. Zone rebalancing is not provided. |
| 3. SoA storage + auto-parallel from declared access | bevy_ecs (typed params plus ambiguity detection); hecs 0.11.2 (2026-10-04) for storage only; flecs_ecs 0.2.2 (2025-11-17) bindings; legion 0.4.0 (2021, dormant) | Archetype SoA, parallel scheduling, conflict detection (off by default) | Access is declared through types, so it is sound for ECS data but blind to interior mutability inside resources. Inter-system order among non-conflicting systems is unspecified. Ambiguity detection is opt-in. |
| 4. Per-tick arena | bumpalo 3.20.3 (2026-05-22) | "phase-oriented allocations"; mass reset; **lifetimes `'bump` statically prevent arena references escaping a `reset()`**, which needs `&mut Bump` ([README](https://github.com/fitzgen/bumpalo)) | `Drop` is not run for arena objects (unless `bumpalo::boxed::Box`). Code can still use the global allocator; making the arena the *only* allocator inside a tick is not enforceable. |
| 5. Bounded queues + load shedding | tokio::sync::mpsc (bounded = backpressure); crossbeam-channel 0.5.17 (2026-09-05; `try_send` → `TrySendError::Full`); flume 0.12.0 (2025-12-08; bounded/unbounded/rendezvous) | Bounded MPSC/MPMC with non-blocking full detection, which is enough to build shed policies | Unbounded constructors exist in the same crates. No crate ships *named shedding policies* (drop-oldest, priority, per-sender quota) as a first-class, auditable declaration. |
| 6. Versioned, zero-copy, validated messages | rkyv 0.8.18 + bytecheck 0.8.3 (2026-08-05); flatbuffers 25.12.19 (2025-12-19); capnp 0.27.3 (2026-10-27); serde 1.0.229 (2026-07-18) | rkyv: derive-based, zero-copy, validation "upfront". FlatBuffers/Cap'n Proto: schema evolution and validation, but with an external schema language. | **No single Rust option is derive-from-Rust + zero-copy + validated + schema-evolving.** rkyv's own matrix lists "Schema evolution: no" for rkyv and "yes" for Cap'n Proto/FlatBuffers ([feature-comparison.md](https://github.com/rkyv/rkyv/blob/main/book_src/feature-comparison.md)). Compatibility checking between message versions has to be added (e.g. a build script diffing a stored schema). |
| 7. `unsafe`-like escape hatch | Rust `unsafe`; `#[allow(clippy::...)]`; flecs-style `immediate` systems; Bevy exclusive systems | Memory-unsafety escape is native to Rust | Rust's `unsafe` covers *memory* safety only. tick-lang's hatch covers *determinism/effects* (call the clock, do I/O, touch another zone). Rust has no effect system, so a library can only approximate this with capability tokens or lints that users can bypass. |
| Networking around all of this | lightyear 0.31.0 (server-authoritative Bevy, prediction/interpolation, WebTransport); renet 2.0.1 (2026-10-30; reliable/unreliable channels, netcode auth); naia 0.25.0 (2026-05-12; compile-time `Protocol` of replicated types, Tribes 2 model, authority delegation) | Transport, replication, channels | Not lockstep-first (except lightyear's deterministic crate); message versioning is not first-class. |

**Measured serialization numbers relevant to pattern 6** (VERIFIED, [rust_serialization_benchmark README](https://github.com/djkoloski/rust_serialization_benchmark), last updated 2026-11-05):
- **Workload:** the `mk48` data set, "composed of mk48.io game updates", which makes it directly relevant.
- **Hardware:** 4-vCPU KVM guest on an Intel Xeon 6975P-C, rustc 1.101.0-nightly.
- **Caveat:** maintained by volunteers including rkyv's author, djkoloski.

| Crate | Serialize | Deserialize | Zero-copy "Read" (unvalidated) | Validated access | Size (bytes) |
|---|---|---|---|---|---|
| rkyv 0.8.18 | 738.46 µs | 1.9026 ms unvalidated / 2.4200 ms validated | 185.97 ns | 435.94 µs (validated upfront) | 2,075,936 |
| flatbuffers 25.12.19 | 3.4793 ms | n/a | 1.9049 µs | 3.7758 ms (validated upfront) | 2,325,620 |
| capnp 0.27.3 | 1.9253 ms | n/a | — | 43.102 ns access / 312.56 ns read (validated on demand) | 2,724,288 |
| bitcode 0.6.9 (not zero-copy) | 623.92 µs | 2.1009 ms | — | — | 971,318 |
| serde_json (baseline) | 13.097 ms | 24.012 ms | — | — | 9,390,461 |

**Takeaway:** on game-update data, upfront validation of an rkyv archive (about 0.44 ms) costs roughly a quarter of a full deserialize (about 1.9 ms). Cap'n Proto's on-demand validation is cheaper still for partial reads. "Validated zero-copy" is affordable at tick rates. The missing piece is *versioning*, not speed.

---

## 5. Hot reload in Rust

- **hot-lib-reloader** (Robert Krahn): reloads a `dylib` exposing `#[unsafe(no_mangle)]` functions such as `step(&mut State)`. Limitations: "Types of structs and enums that are used in both the executable and library cannot be freely changed. If the layout of types differs you run into undefined behavior". Reloadable functions "cannot be generic". It suggests serializing state across reloads (VERIFIED, [README](https://github.com/rksm/hot-lib-reloader-rs)). Last release **0.8.2, 2025-08-11**.
- **Subsecond** (Dioxus Labs): runtime hot-*patching* via `subsecond::call` and `subsecond::current(fn)`. It rewinds the stack to the "cleanest" hot entry point (VERIFIED, [README](https://github.com/DioxusLabs/dioxus/tree/main/packages/subsecond)). Limitations (VERIFIED, [lib.rs docs](https://github.com/DioxusLabs/dioxus/blob/main/packages/subsecond/subsecond/src/lib.rs)):
  - It "only patches the 'tip' crate". This is attributed to rustc's non-deterministic build graph and to generics cascading.
  - Globals are partly supported: new globals' destructors never run, and changed static initialisers are not observed.
  - "Subsecond currently does not support hot-reloading of structs ... If layout or alignment change ... the program will crash". Frameworks must "re-instance", i.e. throw away or rebuild state.
  - `dx serve --hotpatch` is required while it is experimental.

  Dioxus 0.7 made it the headline feature (SNIPPET). The crate's first stable release was **0.7.0 on 2025-10-31**; it is now at 0.7.10 (2026-07-30) with **0.8.0-alpha.3 (2026-10-16)** (VERIFIED, crates.io). Bevy 0.17 has initial integration (SNIPPET).
- **Implication for tick-lang:** both Rust tools reload *code* and leave *state layout changes* to the framework. tick-lang owns the state schema (SoA tables) and the tick boundary, a natural quiescent point. It can do what neither tool does: **migrate state between schema versions at a tick boundary**, by reusing the message-versioning machinery from pattern 6. A Rust library built around an ECS with reflection (Bevy) could approximate this, but would have to run the migration itself; the tools above do not.

---

## 6. Rust's limits as a host for a "tick-lang as a library"

1. **Proc macros see tokens, not types or callees.** "Procedural macros operate over *token streams* instead of AST nodes", and they are unhygienic (VERIFIED, [Rust Reference, procedural macros](https://github.com/rust-lang/reference/blob/master/src/procedural-macros.md)).
   - A `#[system]` macro can reject `Instant::now()` written literally in the body. It cannot know that a helper `foo()` it calls reads the clock, uses a `HashMap` iteration order or touches a `static`. It also cannot resolve aliases (`use std::time::Instant as T`).
   - *(Analysis based on the reference text, not a measured claim.)*
2. **Clippy `disallowed-methods` / `disallowed-types`.** Configured in `clippy.toml` as fully-qualified paths with an optional `reason` and `replacement` (VERIFIED, [lint_configuration.md](https://github.com/rust-lang/rust-clippy/blob/master/book/src/lint_configuration.md)). Limitations (analysis):
   - It is configured per crate or workspace, not per "system" scope, so banning `Instant::now` also bans it in the networking layer.
   - It is opt-in per CI and silenced by `#[allow]`.
   - It checks the crate being linted, not the bodies of dependency code it calls.
3. **Dylint** (Trail of Bits): "runs lints from user-specified, dynamic libraries", so custom type-aware lints are possible. An example would be "functions registered as tick systems may not transitively call items in a deny-list" (VERIFIED, [README](https://github.com/trailofbits/dylint)). Last release dylint_linting **6.3.2, 2026-10-30**. This is the strongest *library-side* enforcement available. It is still advisory, depends on nightly-compiler internals, and is bypassable.
4. **No effect system.** Rust's `unsafe` marks memory-safety obligations only. There is no way to mark a function "deterministic" or "performs I/O" and have the compiler propagate that. Capability tokens (pass `&Clock` only to systems allowed to read time) are a known library idiom (analysis), but globals bypass them.
5. **What Rust *does* enforce well, at library level:**
   - Exclusive and aliasing access to ECS data (borrow checker plus Bevy's scheduler).
   - Thread confinement (`!Send` in monoio and `LocalRuntime`).
   - Arena lifetimes (bumpalo).
   - Exhaustive matching of message enums.
   - Basic-float IEEE determinism on 64-bit targets (RFC 3514).
6. **Compile-time cost.** I found no measured source this session isolating proc-macro or derive cost for an ECS-heavy game. That proc-macro-heavy crates (serde derives, Bevy) lengthen build times is UNVERIFIED here. If tick-lang generates Rust, it inherits rustc compile times either way. It could emit simpler, macro-free Rust, which may *reduce* them; that is untested.

---

## 7. Synthesis for the report

### How much of tick-lang a Rust library could deliver today

Most of the **runtime machinery** already exists as maintained crates, and could be assembled into a "tick framework":

| Piece | Crates |
|---|---|
| Fixed-tick schedule plus rollback, checksums and sync-test | bevy_ecs + ggrs/bevy_ggrs |
| Deterministic parallel physics | rapier `enhanced-determinism` |
| Thread-per-core zones | Tokio `LocalRuntime` or monoio, one per pinned core |
| Bounded channels | crossbeam/flume/tokio |
| Per-tick arenas | bumpalo |
| Zero-copy validated messages | rkyv + bytecheck |
| Replication | lightyear/naia/renet |
| Code hot-patching | subsecond |

A reasonable estimate (my judgement, not measured) is that **roughly 60–75% of the *mechanism* value** (the "it runs, fast, in parallel, with rollback" part) is available as a library plus a `#[system]`/`#[message]` macro layer and a Dylint lint pack.

### What a library cannot enforce

These are the guarantees that justify a language:

1. **Determinism of rule bodies.** No ambient clock, OS RNG, I/O, `HashMap`/`HashSet` iteration order, statics, thread-locals or interior mutability hidden in resources. Lints catch literal cases only, and do not see dependency bodies.
2. **Complete state capture.** bevy_ggrs snapshots only *registered* components and resources. Anything else "caus[es] silent desyncs". A language that owns all state can make save, load and checksum total by construction.
3. **Stable iteration and merge order.** Bevy query order and the order of ambiguous systems are unspecified. Ambiguity detection defaults to `Ignore`.
4. **Deterministic math only.** Rust's transcendentals are documented as non-deterministic. A library can offer `libm` or fixed-point types, but cannot remove `f32::sin` from the user's reach.
5. **Share-nothing without side channels.** `!Send` stops accidental sharing of *tasks*. `static`, `Arc<Mutex<_>>` and globals in dependencies remain available.
6. **Bounded-only queues and declared shedding.** Unbounded constructors sit in the same crates.
7. **Schema-evolution rules for messages and state.** rkyv has none; FlatBuffers and Cap'n Proto need an external IDL. A compatibility check across versions is not built in anywhere.
8. **An *effects* escape hatch.** Rust's `unsafe` is about memory, not determinism. A "this rule may read the clock" marker cannot be compiler-checked in Rust.
9. **Hot reload with state-layout migration.** Subsecond and hot-lib-reloader explicitly do not migrate struct layouts.

**Industrial precedent:** Unity went down this path and built Burst/HPC#, a *compiler-enforced* subset, on top of a runtime job-safety checker. The checker is Editor-only and does not protect statics. That is evidence that libraries plus runtime checks were not judged sufficient.

**Counterpoint for the report:** a "Rust library plus Dylint lint pack plus SyncTest in CI" stack catches most determinism bugs *empirically*, as GGPO's own advice says ("use SyncTest. A lot."). tick-lang has to argue that *static* guarantees are worth a new toolchain, rather than claim the runtime is otherwise impossible.

### Key things tick-lang should borrow

- Seastar `sharded<T>`, and Monoio/`LocalRuntime` `!Send` zone ownership.
- Tokio's "no implicit queuing".
- Bevy's signature-derived access, with ambiguity as a hard error.
- flecs's automatic sync-point insertion, staged per-thread command queues and deterministic entity slicing.
- GGPO's save/load/advance plus checksum plus SyncTest, generated for *all* state.
- AoE's scheduling of inputs for turn N+2.
- Quantum's FP-only math, or rapier's libm-pinned floats.
- rkyv's validation cost profile, combined with FlatBuffers/Cap'n Proto-style evolution rules.
- Subsecond's "rewind to the cleanest hot entry point", with the tick boundary as that point.
