# A. Prior-art leads: fact-check (as of 2026-11-07)

## How this was checked (read first)

- **Network limits in this session.** Only `github.com`, `raw.githubusercontent.com`, `arxiv.org`, `api.crossref.org` and package registries could be reached directly. These were all blocked: dev.epicgames.com, unrealengine.com, gamesbeat.com, news.ycombinator.com, spacetimedb.com, developers.rune.ai, doc.photonengine.com, fab.com, improbable.io, easel.games, dl.acm.org, reddit, wikipedia, web.archive.org, steamcharts and others. The GitHub REST API (`gh api`) is also gated for repos not attached to this session.
- **What I used instead.**
  - **(a) Repos.** I made bare `git clone`s of public repos and read files and commit/tag dates straight from git. This is the best evidence in this report, and the dates are exact.
  - **(b) Search snippets.** I used the web-search tool's summaries of pages on the primary domain (dev.epicgames.com, unrealengine.com, doc.photonengine.com, easel.games). These are marked **"(search snippet)"**. The page was not fetched, so the wording may be paraphrased.
- **Status labels.**
  - **VERIFIED** means a primary source was read directly (repo, paper metadata).
  - **VERIFIED (snippet)** means the primary domain says it according to a search snippet, but I could not fetch the page.
  - **SECONDARY ONLY** means only news or aggregator sources were found.
- To upgrade the snippet-level items, allow `dev.epicgames.com`, `unrealengine.com`, `doc.photonengine.com`, `easel.games`, `spacetimedb.com` and `news.ycombinator.com` in the session network settings and re-fetch.

---

## 1. Epic Verse (Fortnite / UEFN / UE6)

### 1a. "Functions run as transactions that can roll back and re-run"
- **Status: VERIFIED for rollback, with a nuance. "Re-run" applies only to the planned distributed system.**
- **Rollback: verified.**
  - Links: Book of Verse source repo https://github.com/verselang/book (`docs/08_failure.md`, `docs/13_effects.md`, `docs/00_overview.md`).
    - Who: the `verselang` org. 84 of the repo's commits use `@epicgames.com` author emails, and the merger of the most recent PR uses `@northeastern.edu`.
    - Last commit seen: 2026-11-02. Licence CC0-1.0, per the GitHub page.
  - Quote from `08_failure.md`: "When you execute code in a failure context, changes to mutable variables are provisional—they only become permanent if the entire context succeeds."
  - Quote from `00_overview.md`: "Methods marked with `<transacts>` provide automatic rollback on failure."
  - Effects table in `13_effects.md`:
    - `<transacts>` = `{reads, writes, allocates}`, described as "*Exclusive; default*".
    - `<computes>` = "*Exclusive; Pure computation*".
    - `<decides>` = "*Cannot combine with `<suspends>`*".
    - `no_rollback` = "*To be deprecated; Transactions disallowed*".
  - Also in `13_effects.md`: "The `<transacts>` is the default for functions."
  - Official docs, also covered by search snippet:
    - https://dev.epicgames.com/documentation/en-us/uefn/failure-in-verse: "all functions called in the failure context must have the effect specifier <transacts>".
    - https://dev.epicgames.com/documentation/fortnite/speculative-execution: "If the expression fails, the effects of the expression are rolled back, as though the expression never happened."
- **"Re-run": only for the planned distributed system.**
  - Link: Epic's "The road to Unreal Engine 6", https://www.unrealengine.com/news/the-road-to-ue-6 (search snippet).
  - According to the snippet, the runtime pauses a transaction, moves the needed object to another server and reruns the transaction. Epic calls this early prototype work.
- **Important caveat for tick-lang.** According to the same post (snippet), today "Verse runs on a single thread" and calls C++ "built via a custom LLVM compiler that automatically transactionalizes the C++ code".
  - That compiler is AutoRTFM. The Epic tech blog post is https://www.unrealengine.com/en-US/tech-blog/bringing-verse-transactional-memory-semantics-to-c (search snippet).
  - The snippet says it shipped on some Fortnite servers from v28.10, and that the first version handles single-threaded transactions only.

### 1b. "Most data immutable"
- **Status: VERIFIED.**
- Link: `verselang/book/docs/05_mutability.md`.
- Quote: "Values in Verse are immutable by default: once created, a value never changes. Mutation is available but opt-in and visible. You declare a variable with `var`, change it with `set`, and the effect system records both in the signature of any function that does so."

### 1c. "Execution deterministic"
- **Status: PARTLY RIGHT and easy to over-read.**
- Epic states determinism as a design goal ("as deterministic as possible", from the Verse Language Reference page dev.epicgames.com/documentation/uefn/verse-language-reference, search snippet). This is not a guarantee of bit-identical lockstep replay across machines.
- The book makes the narrower claim for pure functions.
  - `13_effects.md`: "The `<computes>` specifier marks pure functions ... These functions are deterministic: given the same inputs, they always produce the same outputs."
  - Structured concurrency (`sync`) is described as "maintaining deterministic behavior" (`14_concurrency.md`).
- In the Verse Calculus paper, "deterministic" means deterministic functional logic programming (confluent rewriting, no choice nondeterminism). It does not mean deterministic simulation in the netcode sense.
  - Crossref abstract: "we describe the Verse calculus, VC, a new core calculus for deterministic functional logic programming ... We also show that the rewrite system is confluent for well-behaved terms."
- No primary source found says Verse guarantees cross-platform deterministic replay (floats etc.).

### 1d. Relevant extra: Verse has client prediction in its effect system (unreleased)
- **Status: VERIFIED.**
- Link: `13_effects.md`.
- Quotes:
  - "The `<predicts>` effect is not yet released."
  - "By default, functions have the `dictates` effect, meaning they run authoritatively on the server. The `<predicts>` specifier allows functions to run predictively on clients for responsiveness, with the server later validating and potentially correcting the results."
- This is direct prior art for tick-lang: authority and prediction are tracked as an effect.
- Also: "Functions without `<writes>` can be safely executed in parallel without locks". This is a claim about compiler potential, not shipped auto-parallelism.

### 1e. "Epic plans to spread one program across many servers automatically (not shipped)"
- **Status: VERIFIED (snippet), unshipped.**
- Link: https://www.unrealengine.com/news/the-road-to-ue-6 (search snippet).
- Snippet wording: Epic is "working toward a full distributed software transactional memory system", with plans "to distribute existing single-threaded-style Verse game code across multiple servers automatically". Early prototypes are "promising" and the results are experimental.

### 1f. "Verse to underpin UE6; UE6 early access targeted late 2027"
- **Status: VERIFIED (snippet).**
- Links:
  - https://www.unrealengine.com/news/the-road-to-ue-6 (search snippet): "targeting a Unreal Engine 6 Early Access release at the end of 2027, with the full release of UE6 coming 12-18 months later".
  - https://www.unrealengine.com/news/state-of-unreal-2026-top-news-from-the-show (search snippet) repeats the end-2027 target.
- Supporting primary hint: `verselang/book/DEVELOPMENT.md` says "As of the UE6 reorganization, both VMs live in `Engine/Binaries/Win64/`".
- Secondary sources say Blueprints, Actors and the C++ gameplay framework will be removed or deprecated in UE6. Reports conflict, so this is not confirmed here.

### 1g. "Is Verse open-sourced / usable outside UEFN?"
- **Status: No. The compiler and VM are not public as far as I can find. Only the docs (the Book of Verse) are open.**
- Evidence:
  - `verselang/book/DEVELOPMENT.md` lists as a prerequisite "Access to fortniteMain repository", and the test VMs are Windows executables from that internal tree (`VerseTestScriptCmdVM.exe`).
  - The README says only: "This is the open source documentation for the Verse programming language."
  - Tim Sweeney on X, June 2024 (https://x.com/TimSweeneyEpic/status/1800184322508587449, search snippet): "Permissively licensed open source Verse compiler, runtime, and draft specification are coming, possibly as soon as 2025."
  - No evidence that this happened by 2026-11.
  - In June 2026 Epic open-sourced "Lore", a Rust version-control system, under MIT. This comes from secondary sources only.

### 1h. Lead: https://github.com/UnrealVerseGuru/VerseProgrammingLanguage
- **Status: NOT OFFICIAL, stale.**
- The README says "This repo is not endorsed by Epic Games, Inc." and that the author is not an Epic employee.
- It contains links and notes only. The README also warns against confusing it with "verse-lang/verse" and "Idyllei/verse-pl".
- Git history: 45 commits, from 2023-03-15 to 2023-04-07 (last commit).
- **Related unofficial project:** https://github.com/rudi-cilibrasi/openverse, "an unofficial (not Epic Games related) open-source Verse runtime with Lean verification".
  - It is written from the Book of Verse and the VC paper.
  - It is brand new: first commit 2026-11-04, last commit 2026-11-07, 70 commits.

### 1i. Lead: https://gamesbeat.com/?p=326844
- **Status: SECONDARY, not fetched (host blocked).**
- The search result title is "Unreal Engine 6 will combine UE5 and UEFN into a unified engine | State of Unreal". Use the Epic primary sources above instead.

### 1j. Verse Calculus paper
- **Status: VERIFIED (metadata and abstract via Crossref, DOI 10.1145/3607845).**
- Title: "The Verse Calculus: A Core Calculus for Deterministic Functional Logic Programming".
- Published in PACMPL (ICFP issue), dated 2023-08-30, licence CC-BY 4.0.
- Authors: Augustsson, Breitner, Claessen, Jhala, Peyton Jones, Shivers, Steele Jr., Sweeney.
- The abstract quote is in 1c.
- The abstract says nothing about transactions, servers or games. It is about the semantics of the functional-logic core.

---

## 2. Easel
- **Claim:** a language for multiplayer games with built-in rollback.
- **Status: VERIFIED (snippet). Hosted and proprietary as far as I can tell. Alive in 2026.**
- **Links:**
  - Docs: https://easel.games/docs/learn/multiplayer/rollback-netcode (search snippet; the host is blocked).
  - Blog: https://easel.games/blog/2026-rollback-physics, "A Physics Engine with Incremental Rollback", discussed on HN as item 47981979 (search snippet).
- **Evidence (search snippets of easel.games):**
  - Easel describes itself as a high-level programming language, game engine and online platform for 2D browser games that makes games "multiplayer automatically".
  - Rollback page: "Easel incrementally snapshots the execution state of all behaviors, and restores them correctly during rollback". It can snapshot coroutines mid-execution.
  - On determinism, one snippet said the docs state that anything written in Easel "is always deterministic". A second search said that exact phrase is not on the rollback page, so treat the wording as unconfirmed. The concept itself (determinism is required for rollback) is stated.
  - 2026 blog: Easel built a custom physics engine on Parry (Dimforge) that snapshots and rolls back only the objects that change, because an off-the-shelf engine would have to snapshot the whole world. A "30–50x" figure appears only in a secondary summary and is UNVERIFIED.
- **Who:**
  - The PeerPush listing "Make Games with Easel" (2025-11-28) was posted by user `raysplaceinspace`. The same handle publishes Acolyte Fight on itch.io, so the maker is probably the Acolyte Fight developer. This is not confirmed by a primary page.
  - An r/rust thread (May 2025, via mirrors) has the developer saying Easel is written in Rust and hosted. Games run on Easel's servers, and inputs go peer-to-peer relayed via Cloudflare. Self-hosting is "planned".
- **Licence:** none found. No public source repo found. An HN commenter asked about open-sourcing, and no answer was found.
- **Lead HN 44000867:** could not fetch (HN blocked), and search gave nothing for that id.
- **Gap vs tick-lang:** it is a 2D browser engine plus platform, hosted, with no compile-to-Rust standalone server and no automatic parallelism. It is single-threaded per the HN snippet.

---

## 3. SpacetimeDB (Clockwork Labs)
- **Repo:** https://github.com/clockworklabs/SpacetimeDB. Last commit 2026-11-06. Latest tag `v2.15.0` dated 2026-11-06; the one before is `v2.14.1` (2026-11-02).

### 3a. Licence
- **Status: VERIFIED, with detail.**
- From `LICENSE.txt` (HEAD):
  - "Business Source License 1.1", Licensor "Clockwork Laboratories, Inc.", Licensed Work "SpacetimeDB 2.15.0".
  - Additional Use Grant: "no more than one SpacetimeDB instance in production and provided that you do not use the Licensed Work for a Database Service."
  - "Change Date: 2031-11-05".
  - "Change License: GNU Affero General Public License v3.0 with a linking exception".
  - Also: "Effective on the Change Date, or the fourth anniversary of the first publicly available distribution of a specific version ... whichever comes first".
- **Subdirectories differ.** Some crates symlink to `licenses/apache2.txt`, for example `crates/bindings` (the Rust module bindings) and `crates/sats`. Others symlink to `licenses/BSL.txt`, for example `crates/core`, `crates/bindings-typescript`, `crates/bindings-cpp` and `sdks/rust`.

### 3b. Languages
- **Status: VERIFIED. There are now FOUR module languages.**
- Quote from `docs/docs/00100-intro/00100-getting-started/00300-language-support.md`: "Modules can be written in four languages:"
  - "Rust ... compiled to WebAssembly"
  - "C# ... compiled to WebAssembly"
  - "TypeScript ... runs on V8"
  - "C++ ... compiled to WebAssembly"
- Client SDKs: Rust, C#, TypeScript and Unreal (C++/Blueprint).

### 3c. Threading and transactions
- **Status: VERIFIED.**
- Quote from `docs/docs/00100-intro/00100-getting-started/00550-misconceptions.md`: "**One transaction at a time.** Each database runs its transactions one at a time on a single thread, by design. ... Inter-database communication lets an app spread its work across many databases".
- Same file: "Transactions are ACID with serializable isolation."
- Quote from `transactions-atomicity.md`: "each reducer invocation is a transaction - either all of its changes succeed and are committed to the database, or all changes are rolled back as if the reducer never ran."
- A unit-test harness "lets tests control the clock, the random seed, and HTTP responses" (misconceptions.md). This is controlled determinism for tests, not a determinism guarantee.

### 3d. BitCraft
- **Status: VERIFIED as a vendor claim, and BitCraft is live.**
- README: "The entire backend of our MMORPG BitCraft Online runs as a single SpacetimeDB module". misconceptions.md: "BitCraft Online runs entirely on Spacetime in production."
- Server code is public at https://github.com/clockworklabs/BitCraftPublic under the Apache-2.0 LICENSE. Latest commit is "release oct-22" (2026-10-22).
- Player counts: SteamCharts (search snippet, about Sep 2026) shows roughly 672 average players over 30 days and a 1,035 peak. This is secondary and was not fetched.

### 3e. Performance numbers
- **Status: VERIFIED (snippet) as a vendor claim. The methodology is disputed.**
- "Let's talk benchmarks", https://spacetimedb.com/blog/benchmarking (search snippet, dated 2026-05-14).
  - The TypeScript module reached about 303,920 TPS and the Rust module about 265,541 TPS.
  - The workload is a "transfer" benchmark: balance transfers between accounts, counting completed transfers per second, rerun over five minutes.
- The repo's misconceptions.md confirms the benchmark exists and that it "reran the transfer benchmark over five minutes, measuring throughput and latency".
- Critiques: a dev.to analysis questions the durability settings (`withConfirmedReads`) and the SQLite baseline. The strn.cat "short technical review" was later archived by its author. The repo answers both.
- The repo's internal bench crate (`crates/bench/README.md`) says Criterion timings are "latencies to call a spacetime reducer without network delays".
- **Gap vs tick-lang:**
  - Not fixed-tick lockstep; it is reducer/event driven, with scheduled reducers.
  - No auto-parallelism inside a database (single thread by design).
  - No language-level determinism guarantee.
  - Transactions roll back, but there is no snapshot/rollback netcode.

---

## 4. Rune
- **Claim:** deterministic JS game logic runs on clients and server.
- **Status: VERIFIED.**
- **Link:** docs source in https://github.com/rune/rune (`docs/docs/how-it-works/server-side-logic.md`), published at https://developers.rune.ai/docs/how-it-works/server-side-logic.
- **Quotes:**
  - "your game's `logic.js` file will run on every client and on Rune's servers"
  - "The primary aim is to ensure that the code is deterministic, meaning that if you run the code multiple times with the same input it will produce the same result."
  - Banned or flagged: closure mutation, `async`/`await`, `try`/`catch`, `eval`, `this`, `Date`, `fetch` and regular expressions. "A notable exception to this list is `Math.random()` which Rune makes deterministic".
  - "only the action payloads are sent between clients and server, not the entire game state".
  - It is enforced by an ESLint plugin and CLI warnings, plus a whitelist of dependencies. This is linting, not a compiler guarantee.
- **Who:** Rune AI Inc. The LICENSE is MIT, "Copyright (c) 2024 Rune AI Inc.". The README claims "10 million installs" for the Rune app (vendor claim).
- **Alive?** Yes, the repo is active: last commit 2026-08-27, and a CLI version bump to 8.0.4 on 2026-08-18. The docs-folder blog/release notes stop at 2025-06-11. No shutdown found. A "Salesforce acquires Rune" item is about a different, defence-AI company named Rune.
- **Gap vs tick-lang:** JS with lint-level determinism, Rune-hosted only, no parallelism, no compile to Rust.

---

## 5. Mun
- **Claim:** quiet since 2024-25.
- **Status: PARTLY WRONG. There have been no releases since Dec 2023, but there was a maintenance burst in Aug–Oct 2026.**
- **Link:** https://github.com/mun-lang/mun
- **Evidence (git):**
  - Latest tag `v0.5.0` is dated 2023-12-28; `v0.4.0` is dated 2022-12-12.
  - Last commit 2026-10-02 (dependabot). Human commits on 2026-09-05/06 by Bas Zalmstra: "build: use Pixi for local development and CI", "chore: modernize dependencies", "fix: ignore filesystem events during runtime shutdown".
  - There is a gap from 2025-05-06 to 2026-08-17 with no commits.
  - Commit counts: 18 in 2024, 12 from 2025-01-01 to now.
- **README:** "Ahead of time compilation", "First class hot-reloading", "LLVM is used for compilation and optimization". Licence MIT OR Apache-2.0.
- **Who:** essentially one main maintainer (Bas Zalmstra); 7 of 12 commits since 2025 are his.

---

## 6. Photon TrueSync, Rollback Core for UE5, Photon Quantum

### 6a. Photon TrueSync
- **Status: DISCONTINUED. VERIFIED (snippet).**
- Link: https://doc.photonengine.com/truesync/current/reference/known-issues (search snippet).
- Wording: "development of Photon TrueSync & Thunder is ceased and we will not publish any further updates"; existing apps are unaffected; "we recommend to migrate apps that are still in development".

### 6b. "Rollback-Core for Unreal 5"
- **Status: EXISTS, early-stage and commercial. VERIFIED (snippet).**
- Links: https://gregorigin.itch.io/rollback-core; the Fab listing https://www.fab.com/listings/03aa010f-2413-4a98-91b1-4fa67d0aedd4; the UE forum thread https://forums.unrealengine.com/t/gregorigin-rollback-core-pro-accessible-netcode-framework-for-fighting-platform-games/2728262 (all search snippets).
- Description: "a deterministic GGPO-style rollback netcode framework built specifically for UE5". The Fab listing states version 0.3.0, "not yet production-ready ... under heavy development". The forum mentions a 0.2.0a update dated 2026-06-16.
- Who: "GregOrigin". The listing says a free open-source "Core" version is on GitHub, but https://github.com/GregOrigin shows 0 public repos, so I could not locate it.
- Gap: it is a UE5 plugin aimed at fighting and platform games. The user must keep their own logic deterministic.

### 6c. Photon Quantum
- **Status: VERIFIED (snippet), alive.**
- **Who:** Exit Games (Photon Engine).
- **Version evidence:** the API docs are titled "Quantum 3 3.0.14", and a "Quantum 3.1 preview" what's-new page exists. The Simple FPS sample lists 3.0.5 (2025-07-24) and 3.0.0 (2024-08-16). All from search snippets.
- **How it enforces determinism:**
  - **Fixed-point maths.**
    - https://doc.photonengine.com/en-us/quantum/current/manual/quantum-ecs/fixed-point (snippet): the FP type "replaces floats and doubles in simulation computations", format Q48.16, "Internally FP uses one long". Readable via `FP.RawValue`.
    - Quantum 2 docs (snippet): determinism rules out float/double and Unity vectors and physics; "the simulation needs to compute the same results on all clients, given they use the same input values".
  - **DSL `.qtn`.**
    - https://doc.photonengine.com/quantum/v3/manual/quantum-ecs/dsl (snippet): components and other runtime game-state types must be declared in Quantum's DSL.
    - Code generation produces serialization (snapshots, saves, killcam replays), checksumming and debug dumps.
    - The DSL is C#-like: `component`, struct, union, enum, bitset, list, dict, input, event, signal and global. Quantum 3.1 adds `#pragma module`.
  - **Predict/rollback.** Clients predict input locally, and a rollback system restores state and re-simulates mispredictions. Only inputs go over the network. A game-agnostic server handles input exchange and clock sync.
  - **State hashing.** API classes exist for `DeterministicTickChecksum` (a "64-bit checksum value" per tick), `ChecksumErrorFrameDump` and `DeterministicChecksumResult` (snippet of doc-api.photonengine.com).
- **Closest commercial match to tick-lang's runtime model.** But it is tied to Unity and C#, single-sim, and has no auto-parallel rules across zones. Determinism comes from the API discipline (FP types, the DSL) plus convention, not a compiler proof. I could not confirm whether Quantum 3 can run the simulation authoritatively on a standalone server; older forum posts mention a "custom quantum plugin" on Photon servers.

---

## 7. Search for a system with all four properties
The four properties: (P1) determinism guaranteed by language or compiler, (P2) automatic parallelism, (P3) built-in snapshot/rollback/state hashing, (P4) compiles to Rust or C for a standalone, engine-independent server.

**Result: none found.** Searches covered GitHub, general web, HN mirrors, Reddit mirrors and arXiv (extended mode). They found no language or system with all four. Closest matches by property:

| System | P1 lang-guaranteed determinism | P2 auto-parallel | P3 snapshot/rollback/hash | P4 Rust/C standalone | Evidence and status |
|---|---|---|---|---|---|
| **Lingua Franca** (lf-lang) | **Yes**: "ensuring deterministic interaction with explicit management of timing" | **Yes**: "executes transparently on multiple cores without compromising determinacy" | No (no rollback/netcode) | **Yes**: C, C++, Python, TS; a Rust generator exists in-tree (`core/src/main/kotlin/org/lflang/generator/rust`) but "The crate is unstable" (reactor-rs) | BSD-2. lingua-franca last commit 2026-11-04; reactor-rs last commit 2025-06-10. **Closest on P1+P2+P4; lacks P3 and the game domain.** |
| **Céu** (synchronous reactive) | Yes (synchronous model) | No | No | Yes: "Ceu compiles to C" | Last commit 2022-11-03. Represents the Esterel/Lustre family (the tick = synchronous instant idea). |
| **Verse** (Epic) | Partial (pure `<computes>` only; design goal "as deterministic as possible") | No (runtime is single-threaded per Epic, snippet; distributed STM planned) | Transactions/rollback yes; no snapshot or state-hash netcode; `<predicts>` unreleased | No (UEFN only; compiler not public) | See section 1 |
| **Photon Quantum** | Partial (FP types + DSL; convention) | No (single sim) | **Yes** (rollback, snapshots, per-tick checksums) | No (Unity/C#) | See 6c |
| **Easel** | Claimed yes (its own language) | No (single-threaded) | **Yes** (incremental snapshot incl. coroutines) | No (hosted, browser) | See section 2 |
| **Rune** | Lint-level (ESLint + CLI) | No | Yes (predict-rollback, internal) | No (JS, Rune-hosted) | See section 4 |
| **SpacetimeDB** | No (test harness only) | No ("one transaction at a time ... single thread") | Transaction rollback only | Partly (modules are Rust/C#/C++ to WASM or TS on V8; the server itself is Rust) | See section 3 |
| **Bevy + GGRS / bevy_ggrs** | No: "your game logic — must be deterministic!" | Bevy schedules systems in parallel, but the pitfalls doc warns "Bevy does not guarantee a stable iteration order for queries" | **Yes**: snapshots "only the components and resources you register"; `SyncTestSession` with `checksum_component_with_hash` | Yes (Rust, but a library not a language) | MIT/Apache. ggrs last commit 2026-10-19; bevy_ggrs last commit 2026-10-31 |
| **Lightyear** (Bevy) | No: "The simulation needs to be deterministic." | Bevy | Prediction/rollback, deterministic replication mode | Rust library | Last commit 2026-11-05 |
| **TimeSteward** (elidupree) | Partial (framework model) | "parallel computation" (marked with asterisk in README) | Yes (change past events, cheaply update present) | Rust library | Last commit 2019-03-19 (dead). Search snippet of the README. |
| **Hydro** (Berkeley) | Aims at "distributed safety"; correct by construction (not game determinism) | Dataflow (DFIR) "automatic vectorization and efficient scheduling" | No | Rust (staged framework) | Last commit 2026-11-07 |
| **Dojo** (Cairo, onchain) | Yes (provable / onchain execution) | No | No rollback netcode | No (Cairo / Starknet) | Last commit 2026-05-06 |
| **Unity Netcode for Entities** | No: docs say prediction errors come from "slightly different logic between clients and server, packet drops, and quantization errors" (snippet) | DOTS jobs | Rollback via `PredictedSimulationSystemGroup` re-simulating from the oldest applied tick (snippet) | No (Unity) | docs.unity3d.com (snippet) |
| **Roblox Parallel Luau** | No official determinism statement found | Actors run in parallel; "Scripts under the same Actor run one after another" (snippet) | No | No | luau-lang/luau last commit 2026-11-07 |
| **Hytale** | No | n/a | n/a | Java server plugins (`.jar`), Early Access; secondary sources only | — |
| **SnapNet** (High Horse) | No: "rollback ... without requiring strict determinism" (archived snapnet.dev, snippet) | n/a | Re-simulation rollback, server-authoritative | UE/Unity plugins | Commercial |
| **Colyseus** | No | No | No (delta-compressed state sync) | Node.js; "MIT licensed" | Last commit 2026-11-06 |
| **Nakama** (Heroic Labs) | No | No | No | Go server; runtime in "Lua, TypeScript/JavaScript, or native Go" | Last commit 2026-11-05 |
| **Rivet** | No | n/a | No | Has **pivoted** to general "Rivet Actors are long-lived processes with in-memory state" for "each agent, user, session, or tenant" | Last commit 2026-11-06 |
| **Hathora** | — | — | — | **Shut down.** Secondary sources: exit announced ~2026-03-04, service off 2026-05-05 after acquisition by Fireworks AI; Stormgate lost online modes. Several secondary sources also claim Unity Multiplay hosting closed 2026-03-31 (unverified). | Secondary only |
| **Hadean** | — | Distributed sim engine (2019 EVE: Aether Wars demo) | — | — | No 2025–26 info found; UNVERIFIED |
| **Darklang** | No | No | No | Backend language and infrastructure; "Dark-next isn't yet ready for production use"; Apache-2.0 | Last commit 2026-11-07 |
| Elixir/BEAM servers, Statecharts (SCXML), Ink | Not researched in depth. They cover share-nothing actors (BEAM), explicit state machines (statecharts) and narrative state (Ink). None of them provide P1 or P3 by themselves. | | | | Not verified |

**Takeaways:**
- The four properties exist separately:
  - P1+P2+P4: Lingua Franca and the synchronous-language family.
  - P3: Quantum, GGRS, Easel and Rune.
  - Transactions and rollback: Verse and SpacetimeDB.
- Nobody found combines them in one compiler-checked language for standalone servers.
- The nearest single-vendor direction is Epic's (Verse + distributed STM + `<predicts>`), but it is unreleased, single-threaded today and locked to UEFN.

### 7b. Improbable SpatialOS: cautionary tale
- **Primary (VERIFIED, git):**
  - https://github.com/spatialos/gdk-for-unity, archived, last commit 2021-10-07. README quotes:
    - "This is the repository for the last version of the GDK plugin that is compatible with the SpatialOS legacy platform. The GDK is frozen for users who have not yet migrated their project to SpatialOS running on zeuz."
    - "please contact us at migration@improbable.io before 30th December 2021."
    - Full docs now require contacting Improbable ("If you wish to get full access to the documentation please contact us").
  - https://github.com/spatialos/UnrealGDK, archived, last commit 2020-09-03 ("Merging final GDK for Unreal 0.11.0 release"). It needed a private Unreal Engine fork: "In order to transform Unreal from a single-server engine to a distributed model, we made a number of small changes to Unreal Engine code".
  - The `spatialos` GitHub org shows its GDK repos as "Public archive", with last updates in Jan 2022.
- **Secondary only** (Improbable's own blog is blocked):
  - Jan 2019 Unity ToS dispute. Unity said Improbable's account was incorrect, then revised its terms; Epic and Improbable set up a US$25M fund (Engadget, PCGamesN, Gamedeveloper).
  - Worlds Adrift (Bossa) closed in July 2019 as "no longer commercially viable". Bossa said SpatialOS was not to blame. One of three SpatialOS projects cancelled in quick succession (pcgamesinsider).
  - Improbable bought German hoster zeuz (Feb 2020) and moved SpatialOS onto it.
  - Improbable sold Midwinter (Scavengers) to Behaviour, then sold The Multiplayer Group to Keywords for £76.5M (CNBC, 2023-12-18).
- **Lessons for tick-lang:**
  - The runtime was proprietary and cloud-locked. Fans noted Worlds Adrift "only runs within its own cloud environment", so it could not be self-hosted after shutdown.
  - It depended on engine vendors' licences and forks.
  - Docs ended up behind sales.
  - These argue for tick-lang's "compile to plain Rust, standalone, not tied to an engine" stance.
