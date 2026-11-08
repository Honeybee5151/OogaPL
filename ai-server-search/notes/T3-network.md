# T3: Network and replication efficiency for MMOs (candidate problem family A)

Research notes for Miles (Sumo Brothers, Godot, one Hetzner Linux box). Written 2026-11-08.
Question: is it feasible to use AI-driven algorithm discovery (LLM-guided program search such as AlphaEvolve or FunSearch, or open-source versions like OpenEvolve and ShinkaEvolve) on MMO networking? Covers interest management, update priority under a bandwidth budget, delta compression and quantisation, and bounded inconsistency. If it is feasible, where exactly should it be aimed?

Every claim carries a label:
- **VERIFIED**: read in the primary source; an exact quote is given.
- **VERIFIED_FIG**: read off a figure or table image in the primary source (approximate).
- **SNIPPET**: seen only in a search-engine excerpt because the host is blocked in this session.
- **UNVERIFIED**: from a secondary source only, or from memory.
- **WRONG**: a common claim that the primary source contradicts.
- **GUESS**: my own estimate or judgement, with the arithmetic shown.

Downloads live under `/tmp/claude-0/-home-user-OogaPL/32372848-6606-541b-85af-54d683c4e916/scratchpad/dl/t3-*`, one folder per source. Repositories were cloned shallow or sparse, and PDFs were converted with `pdftotext -layout`.

---

## 0. Bottom line in plain language

1. **This is one of the best-shaped areas for AI program search, and nobody seems to have tried it.** The pieces to evolve are small functions that real engines already let you plug in:
   - an "importance" or priority function (Unity `BatchScaleImportanceFunction`, Unreal Iris `UNetObjectPrioritizer`, Torque `getUpdatePriority`),
   - a visibility filter (Godot `add_visibility_filter`),
   - a consistency policy (the Dyconits `DyconitPolicy` interface, four methods),
   - an entropy coder for snapshots.

   Each can be scored automatically, quickly and deterministically by replaying recorded game traffic. I found no published attempt to apply LLM-driven evolution to game netcode, interest management or snapshot compression (GUESS, absence of evidence). The closest work is LLM-designed adaptive-bitrate and congestion-control algorithms (NADA, PolicySmith, ReForge), which shows the method transfers to network heuristics scored by trace replay.
2. **Humans have already done this search by hand, and the improvements were real but shrinking.**
   - In 2015, Glenn Fiedler posted a public compression challenge: 901 physics cubes at 60 Hz.
   - His hand-tuned bit-packer: **531.95 bytes per packet (255.3 kbit/s), 1.48 % of raw** (VERIFIED).
   - Fabian Giesen then hill-climbed an arithmetic coder over three days, writing his score into most commit messages: 3435 → 572 → 445 bytes per frame (VERIFIED, from his git log). The last five steps gained only 0.4–1.5 % each.
   - A third person ran a million-try random search over coder parameters (VERIFIED).

   This is exactly the loop that AlphaEvolve automates. The documented curve suggests further gains over the best human result would be modest (GUESS: 5–15 %).
3. **The textbook techniques cut bandwidth by large factors. The remaining headroom over good versions of them is smaller.** Measured savings against stated baselines:
   - interest management that accounts for obstacles: **up to 6× fewer update messages** (VERIFIED);
   - Donnybrook attention-based "interest sets": a 1,000-player game drops from **12 Mb/s to 670 kb/s per player** (VERIFIED);
   - Dyconits bounded inconsistency: **up to 85 % less bandwidth and 40 % more players** (VERIFIED);
   - Fiedler's quantisation plus delta coding: **17.37 Mbit/s down to about 255 kbit/s** (VERIFIED);
   - Oodle Network's trained packet compressor: **more than 28 % on top** (SNIPPET).

   AI search would be competing against these well-engineered baselines, not against naive code.
4. **For Miles, saving bytes saves almost no money.** Hetzner gives dedicated root servers "unlimited traffic" on a 1 Gbit uplink, includes 20 TB/month with EU cloud servers, and charges €1/TB beyond that (VERIFIED).
   - 20 TB/month equals about 62 Mbit/s sustained, enough for roughly 1,500 players online 24/7 at 40 kbit/s each (GUESS, arithmetic in §6).
   - The same traffic on AWS at a widely quoted $0.09/GB (UNVERIFIED) would cost about 90× more per byte.
   - The commercial cost driver is server **CPU** for serialisation and relevancy, not bytes. Riot says dedicating one full core per VALORANT match "would make it prohibitively expensive" (SNIPPET). Epic says the per-actor, per-connection relevancy loop limited Fortnite's server CPU (SNIPPET). Unity says serialisation cost "scale[s] linearly with the number of ghosts" (VERIFIED).
5. **Godot, Miles's engine, ships with none of the standard techniques.**
   - Its `MultiplayerSynchronizer` sends configured properties to all peers by default.
   - It offers per-peer visibility filters and send intervals, but no bandwidth budget, no priority accumulator and no quantisation.
   - Its variant "compression" covers only bools and integers ("Any other case is not yet compressed"). All VERIFIED from Godot source.

   So the **first** win for Miles is plain engineering with known techniques (quantise, delta against acknowledged state, priority accumulator, visibility filter). AI search makes sense only **after** that, on one narrow function.
6. **The main risks are the scoring metric and overfitting.**
   - Donnybrook could validate its bandwidth trade-off only with a **human user study** of 88 people: "How fun was this game?" (VERIFIED).
   - The Dyconits authors' own tutorial leaves open "without creating inconsistencies large enough to be noticed by players?" (VERIFIED).
   - Trained compressors must be retrained for each game patch (SNIPPET, Epic Oodle docs).
   - The ReForge authors warn that a designed network heuristic "fails on the world that arrives after" (VERIFIED).
7. **Verdict for this family:** technically feasible and genuinely novel, but low economic value for Miles at his scale. It is worth doing as a small, well-scoped experiment (§8, candidates A2 and A3) once the basics are in his netcode. It is not a route to large server-cost savings on Hetzner.

---

## 1. Interest management (area of interest, AoI)

**What it is:** the server decides, per client, which entities that client hears about, so it does not broadcast every change to everyone. Common forms:
- distance or radius,
- grids and "towers" (a coarse grid used to look up neighbours),
- quadtrees,
- visibility through precomputed potentially-visible sets (PVS) or ray casts,
- attention-based sets,
- the aura/nimbus model of Benford and Fahlén (1993) (UNVERIFIED, from memory; not read here).

### 1.1 Surveys and comparison studies
- **Liu and Theodoropoulos, "Interest management for distributed virtual environments: A survey", ACM Computing Surveys 46(4), 2014, DOI [10.1145/2535417](https://doi.org/10.1145/2535417).** VERIFIED (abstract via the Crossref API). Quote: "This article provides a comprehensive survey of the state of the art in the design of interest management algorithms and systems. The scope of the survey includes current and historical projects providing a taxonomy of the existing schemes". I did not read the full text (ACM, closed).
- **Boulanger, Kienzle and Verbrugge, "Comparing interest management algorithms for massively multiplayer games", NetGames 2006, DOI [10.1145/1230040.1230069](https://doi.org/10.1145/1230040.1230069).** Abstract read via [OpenAlex](https://api.openalex.org/works/doi:10.1145/1230040.1230069); the PDF hosts (sable.mcgill.ca, comp.nus.edu.sg, cs.mcgill.ca) are blocked.
  - VERIFIED: "interest management algorithms that take into account obstacles in the world reduce the number of update messages between players by up to a factor of 6".
  - VERIFIED: "some computationally inexpensive tile-based interest management algorithms can approximate ideal visibility-based interest management at very low cost".
  - VERIFIED, and important for building an evaluator: "measurements obtained with computer-controlled players performing random actions can approximate measurements of games played by real humans, provided that the starting positions of the random players are chosen adequately".
  - SNIPPET: the experiments ran on McGill's "Mammoth" research MMO, with eight algorithms compared. I could not see which algorithm is the baseline for the factor of 6.
- **Bharambe et al., "Donnybrook: Enabling Large-Scale, High-Speed, Peer-to-Peer Games", SIGCOMM 2008, DOI 10.1145/1402958.1403002.** VERIFIED from the full PDF, [microsoft.com copy](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/donnybrook.pdf); local copy `dl/t3-donnybrook/donnybrook.txt`.
  - Quadratic scaling. Quote: "Each player must receive updates about every other player whenever state changes, e.g., at ∼12n kb/s per player for n players in Quake III."
  - Why plain AoI fails in crowds. Quote: "no such limit occurs naturally, because population density in real games follows a power law [35]."
  - Interest sets. Quote: "Unlike an area of interest, which contains more players as player density increases, an interest set always remains small due to the limits of human attention."
  - Hand-tuned attention weights. Quote: "With only several hours of tuning on our part, we found that weighting interaction recency 1.5 times more than proximity and aim yielded good playability independent of game scale in Quake III". Also: "a different tuning of weights via play testing will likely be required for each game".
  - Numbers. Quote: "each player receives frequent updates, at a rate of 20 per second, from only five other players independent of game size. Guidance is sent to all players, but only once per second ... For a 1,000-player game, this reduces the requirement from 12 Mb/s to 670 Kb/s".
  - Players outside the interest set are simulated locally by "low-behavioral-fidelity doppelgängers" (bots driven by once-per-second guidance).
  - Interest sets change fast. Quote: "a player's interest set changes often, with average turnover of 68% per second according to our user study."
  - Validation needed humans. Quote: "On average, LoBW-IS is preferred by 4.5 points over LoBW, whereas the difference between LoBW-IS and HiBW is statistically insignificant." The question was "How fun was this game on a scale of 1 (not fun) to 10 (extremely fun)?", with "a total of 88 different participants". Stated limitation: "it has only two human players, and it has only 32 players total."
  - Workload data problem. Quote: "Existing fast-paced games rarely support more than 32 players, so it is not possible for us to obtain a real workload with hundreds of players. Instead we use the game workload generator developed by Bharambe et al. [4]".

### 1.2 What shipping engines do (current practice)
- **Quake III Arena (1999, GPL source):** the server culls entities with the precomputed potentially-visible set (PVS) of the client's map cluster. VERIFIED in [`code/server/sv_snapshot.c`](https://github.com/id-Software/Quake-III-Arena/blob/master/code/server/sv_snapshot.c): `clientpvs = CM_ClusterPVS (clientcluster);`, plus area bits and portal handling.
- **Unity Netcode for Entities 7.2.0** (docs mirror [needle-mirror/com.unity.netcode](https://github.com/needle-mirror/com.unity.netcode), `Documentation~/optimization/optimize-ghosts.md`, commit c4cabf4, 2026-10-29). VERIFIED:
  - "Ghost relevancy, also known as ghost filtering, is a server feature that allows you to define under what conditions a specific ghost entity is replicated on a client."
  - Modes "SetIsRelevant" and "SetIsIrrelevant".
  - "Use ghost relevancy to avoid replicating entities that the player can neither see nor interact with."
- **Unreal Engine Replication Graph** (dev.epicgames.com is blocked). SNIPPET from Epic's Chinese-language page, [replication-graph-in-unreal-engine](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/replication-graph-in-unreal-engine): Fortnite Battle Royale supported 100 players per match with about 50,000 replicated actors. The default approach, where each actor decides relevancy for each connection, "performs poorly at that scale and constrains server CPU" (search-tool paraphrase of the page).
- **Unreal Iris filtering.** SNIPPET, [iris-filtering-in-unreal-engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/iris-filtering-in-unreal-engine): "To save time and bandwidth, the filtering system filters which connections an actor or object can replicate". There are owner, connection, group and dynamic filters, including a grid filter.
- **Albion Online** (David Salz, Quo Vadis 2016 slides on slideshare, blocked). SNIPPET: a grid hash with 10×10 m cells, each player has a moving interest area, and "Players should only „see" (network-wise) what's on their screen".
- **Godot 4 (Miles's engine).** VERIFIED in [`modules/multiplayer/doc_classes/MultiplayerSynchronizer.xml`](https://github.com/godotengine/godot/blob/master/modules/multiplayer/doc_classes/MultiplayerSynchronizer.xml) (master, fetched 2026-11-08):
  - "By default, [MultiplayerSynchronizer] synchronizes configured properties to all peers."
  - `add_visibility_filter`: "Adds a peer visibility filter for this synchronizer. [param filter] should take a peer ID [int] and return a [bool]."

  So Godot gives a yes/no hook per synchronizer and peer, but no spatial index and no priority.

### 1.3 CPU cost of the spatial lookup (micro-benchmark)
- The open-source [bhhbazinga/AOI](https://github.com/bhhbazinga/AOI) repository compares crosslink (sorted-axis lists), quadtree and tower (coarse-grid) AoI. VERIFIED README: "We simulate N random moves of N units in a 1024*1024 map, each unit has 30 visible range. (1000<=N<=10000)", on an i7-7700K.
- VERIFIED_FIG from the repo's `benchmark.png`. At N = 10,000:

  | Model | Time for 10,000 moves |
  |---|---|
  | Crosslink | 2,772 ms |
  | QuadTree | 534 ms |
  | Tower | 499 ms |

  At N = 1,000 they take 18, 6 and 4 ms.
- GUESS: grids and towers are already near the practical optimum for uniform random movement, about 50 µs per move at N = 10,000 (499 ms / 10,000). This benchmark uses uniform random walks, not real crowds.

### 1.4 Recent research and AI attempts on interest management
- Recent academic work on interest management looks sparse. My searches for 2022–2026 IEEE Transactions on Games or arXiv papers on interest management found nothing (GUESS, absence of evidence; arXiv full-text search for "interest management" returned 4 unrelated hits).
- **"Combat state-aware interest management for online games: demo"**, Middleware Posters and Demos 2017, DOI 10.1145/3155016.3155019. UNVERIFIED (Semantic Scholar's machine-generated TLDR only): "dynamically adjust update rates based on the current game context". Rule-based, not learned.
- **I found no learned (machine-learning) or LLM-searched interest management** for game state (GUESS, absence of evidence). The nearest is DIA, below.
- **DIA (arXiv [2610.13698](https://arxiv.org/abs/2610.13698), 9 Oct 2026)** is adjacent: deep reinforcement learning that sets per-object bitrate for *cloud-gaming video*. VERIFIED: "DIA achieves at least a 20% higher normalized average Quality of Experience (QoE) than the State-Of-The-Art (SOTA) ABR algorithms in 5G scenarios." This is video, not entity replication.

---

## 2. Update priority and rate control under a bandwidth budget

### 2.1 The lineage: Tribes, then Torque, Halo, Fiedler, Unity and Unreal Iris
- **Tribes (Frohnmayer and Gift, "The TRIBES Engine Networking Model", GDC 1999).** SNIPPET (scribd and gamedevs.org blocked): "TRIBES must run well even over a 28.8 kbps modem, imposing a meager 2 Kbytes per second of bandwidth limitation on each client." The model has a "ghost manager" for scoping and per-object updates.
- **Torque3D (open-source descendant of the Tribes engine): the priority function is a hand-tuned weighted sum.** VERIFIED in [`Engine/source/T3D/gameBase/gameBase.cpp`](https://github.com/GarageGames/Torque3D/blob/master/Engine/source/T3D/gameBase/gameBase.cpp), function `GameBase::getUpdatePriority`.
  - Comment: "Calculate a priority used to decide if this object // will be updated on the client. All the weights // are calculated 0 -> 1 Then weighted together at the // end to produce a priority."
  - Weights: `static F32 sUpFov = 1.0;`, `sUpDistance = 0.4f;`, `sUpVelocity = 0.4f;`, `sUpSkips = 0.2f;`, `sUpInterest = 0.2f;`.
  - Terms: in-field-of-view, linear distance falloff, velocity across the view, `updateSkips * 0.5`, and a per-type "interest".
- **A long-lived bug in that hand-tuned function.** VERIFIED in the same file, in both GarageGames/Torque3D `master` and TorqueGameEngines/Torque3D `master`: `if (getTypeMask() & (PlayerObjectType || VehicleObjectType ))`.
  - The logical `||` evaluates to 1, and [`objectTypes.h`](https://github.com/GarageGames/Torque3D/blob/master/Engine/source/T3D/objectTypes.h) defines `StaticObjectType = BIT( 0 )`, `PlayerObjectType = BIT( 14 )`, `VehicleObjectType = BIT( 16 )`.
  - So the 0.75 "interest" bonus went to static objects instead of players and vehicles. The TorqueGameEngines `development` branch has it fixed: `(PlayerObjectType | VehicleObjectType )` (VERIFIED).
  - GUESS: the effect is small (interest carries a weight of 0.2), but it shows these heuristics shipped without any quantitative evaluation that would have caught a wrong term.
- **Halo: Reach (David Aldridge, "I Shot You First", GDC 2011).** UNVERIFIED (secondary summaries; GDC Vault and the summary sites are blocked).
  - Built on the Tribes model.
  - Bungie "used profiling in replays to cut usage to 20% of what they used for Halo 3".
  - Reported stats: "250kbits/s Minimum total upstream for the host of a solid 16 player game ... 45kbits/s Maximum bandwidth sent to one client" (ng.cs.ucl.ac.uk summary, seen only as a search excerpt).
- **Glenn Fiedler, "State Synchronization" (2015-01-05)**, source at [gafferongames/gafferongames `content/post/state_synchronization.md`](https://github.com/gafferongames/gafferongames/blob/master/content/post/state_synchronization.md); the site itself is blocked. VERIFIED:
  - The priority accumulator: "each frame we add the current priority for each object to its priority accumulator value then sort objects in order from largest to smallest priority accumulator value. The first n objects in this sorted list are the objects you should send that frame."
  - Budget: "typically you have some maximum bandwidth you want to support like 256kbit/sec".
  - Reset rule: "reset the priority accumulator to zero for objects that fit but leave the priority accumulator value alone for objects that didn't."
  - Hand-set priorities: "I calculate priority for the player cube as 1000000 ... interacting (red cubes) I give them a higher priority of 100 while at rest objects have priority of 1."
  - Adaptive budget: "The desired bandwidth can even be adjusted on the fly."
- **Unity Netcode for Entities 7.2.0** (`optimize-ghosts.md`, `limit-snapshot-size.md`). VERIFIED:
  - "The server operates with a fixed bandwidth target and sends a single snapshot packet of customizable size on every network tick. It fills this packet with the ghosts with the highest importance, determined by a priority queue of ghost chunks (rebuilt each tick)."
  - "Unity Netcode multiplies this base importance value by `ticksSinceLastSent`". This is an accumulator in another form.
  - A plug-in hook: "You can also supply your own method to scale importance on a per-chunk, per-connection basis, using [`GhostImportance.BatchScaleImportanceFunction`]". The built-in version is distance-based, using spatial tiles.
  - Also `MaxSendRate`, `NetworkStreamSnapshotTargetSize` ("a (soft) limit on per-connection bandwidth consumption") and `MinSendImportance`.
- **Unreal Iris prioritisation.** SNIPPET, [iris-prioritization-in-unreal-engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/iris-prioritization-in-unreal-engine):
  - "Not every replicated object or property is updated each frame due to limited bandwidth."
  - Priorities accumulate across network ticks until the object replicates (paraphrased from the Japanese page by the search tool).
  - Static and dynamic prioritisers exist (including `SphereNetObjectPrioritizer`), plus `NetObjectCountLimiter`.
  - You can write your own by implementing `UNetObjectPrioritizer`. This is a second production plug-in point.
- **Quake III rate control.** VERIFIED, `sv_snapshot.c` and `sv_client.c`:
  - Time between snapshots: `rateMsec = ( messageSize + HEADER_RATE_BYTES ) * 1000 / rate;` with `#define HEADER_RATE_BYTES 48 // include our header, IP header, and some overhead`.
  - Default client rate `cl->rate = 3000;` (bytes/s), clamped `if (cl->rate < 1000) ... else if (cl->rate > 90000)`.
  - GUESS: 3000 B/s = 24 kbit/s.
- **Godot.** VERIFIED (`MultiplayerSynchronizer.xml`): only `replication_interval` ("Time interval between synchronizations") and `delta_interval`. There is no per-client budget or priority.

### 2.2 Why this sub-problem has headroom (and why it is hard to measure)
- Every implementation above uses **hand-chosen constants**: Torque 1.0/0.4/0.4/0.2/0.2, Donnybrook "several hours of tuning", Fiedler 1,000,000/100/1. I found no published systematic optimisation of a replication-priority function against a quantitative error metric (GUESS, absence of evidence).
- Fiedler judged the result visually from videos. He also captured data offline because "I couldn't really tell looking at a noisy bandwidth meter if I was making any gains" (VERIFIED, `snapshot_compression.md`). Halo: Reach reportedly profiled replays (UNVERIFIED). Both are precedents for trace-replay evaluation.

---

## 3. Delta compression and quantisation of snapshots

### 3.1 Quake III (1999): deltas against the last acknowledged snapshot, plus a static trained Huffman table
VERIFIED in [`code/qcommon/msg.c`](https://github.com/id-Software/Quake-III-Arena/blob/master/code/qcommon/msg.c) and `sv_snapshot.c`:
- "try to use a previous frame as the source for delta compressing the snapshot".
- `MSG_WriteDeltaEntity`: "If force is not set, then nothing at all will be generated if the entity is identical".
- Per-field change bits, small-integer floats (`#define FLOAT_INT_BITS 13`), and a per-field bit width table `netField_t entityStateFields[]`.
- All bytes then go through an adaptive Huffman coder seeded from a fixed table of measured byte frequencies: `int msg_hData[256] = {` 250315, 41193, … and `Huff_addRef(&msgHuff.compressor, (byte)i);`.

So **a model fitted to recorded data has shipped in game netcode since 1999**.

### 3.2 Fiedler, "Snapshot Compression" (2015-01-04)
Source: [`content/post/snapshot_compression.md`](https://github.com/gafferongames/gafferongames/blob/master/content/post/snapshot_compression.md). Scene: 901 cubes, 60 snapshots per second. All VERIFIED.

| Step | Bandwidth or size | Quote |
|---|---|---|
| Uncompressed | 17.37 Mbit/s | "That's a _LOT_ of bandwidth: **17.37 megabits per-second!**" |
| Target | 256 kbit/s | "Our target bandwidth is __256 kilobits per-second__." |
| Raw cube state | 321 bits per cube | "This gives a total of 321 bits bits per-cube (or 40.125 bytes per-cube)." |
| Smallest-three quaternion (9 bits per component) | 128 → 29 bits per orientation | "2 + 9 + 9 + 9 = 29 bits per-orientation (down from 128 bits)" |
| Orientation quantisation alone | more than 5 Mbit/s saved | "This optimization reduces bandwidth by over 5 megabits per-second" |
| Velocity dropped; position at 2 mm precision | 80 bits per cube | "This reduces our cube state to 80 bits, or just 10 bytes per-cube." |
| Delta against acknowledged baseline | "the basic, big order of magnitude win" | "Cube n in snapshot 110 is the same as the baseline. One bit: Not changed!" |

- **Search done by hand on captured data:** "I wrote a short ruby script to find the best encoding with a greedy search."
- **Headroom beyond bit-packing:** "No matter how hard you tune your bit-packer a context aware arithmetic encoding can beat your result by more accurately modeling the probability of values that occur in your data set." Also: "[implementation] by Fabian Giesen beat my best bit-packed result by 25%."
- **An idea left unexploited:** "It's also possible to get a much better result for delta encoded orientations using the previous baseline orientation values to estimate angular velocity and predict future orientations".

### 3.3 The 2015 "networked physics data compression challenge": a ready-made benchmark
- **Fiedler's reference harness:** gist [gafferongames/bb7e593ba1b05da35ab6](https://gist.github.com/gafferongames/bb7e593ba1b05da35ab6), public domain. Its header cites "http://gafferongames.com/2015/03/14/the-networked-physics-data-compression-challenge/". It reads `delta_data.bin`, writes packets, then checks "all packets verify ok!". The recorded result at the end of the file (VERIFIED): "2837 input frames / writing 2831 packets / ... average bytes per-packet: 531.951960 / ... average kilobits per-second: 255.336941 / compression ratio: 1.48% of original size".
  - Consistency check (GUESS): raw size is 901 cubes × 10 values × 4 bytes = 36,040 bytes, and 531.95 / 36,040 = 1.476 %.
- **Fabian Giesen's entry:** [rygorous/gaffer_net](https://github.com/rygorous/gaffer_net), a context-modelling binary arithmetic coder. VERIFIED from `git log` (all commits 2015-03-15 to 2015-03-17):
  - "Initial 'dumb as a rock' version: 3435.47 bytes/frame"
  - "UExpGolomb: use BitTree for mag. 572b/frame in new dataset"
  - "Velocity estimator. 510b/frame"
  - "'Changing' context for orientation different. 488b/frame"
  - "New delta-quat coder for largest-axis switch. 478b/frame"
  - "Position delta magnitude context. 467b/frame"
  - "Orientation delta magnitude context. 460b/frame"
  - "SExpGolomb sign prediction; 455b/frame"
  - "More context for 'interacting' flag - 450b/frame"
  - "Even more flag craziness. 445b/frame"

  `main.cpp` reads `delta_data_realnew.bin`, and `static const int kFrameRate = 60; // just used to calc kbps`.
  - GUESS: 445 B × 60 × 8 = 213.6 kbit/s.
  - Steps shrank fast. 510 → 488 is 4.3 %. The last five (467 → 460 → 455 → 450 → 448 → 445) gained 1.5, 1.1, 1.1, 0.4 and 0.7 % (GUESS, arithmetic on the logged numbers).
- **Discrepancy (open):** 445 / 531.95 = 0.837, a 16 % reduction, while Fiedler wrote "25%". The files differ (`delta_data.bin` versus `delta_data_realnew.bin`; Giesen's log mentions a "new dataset" and "Gaffer says to skip first 6 frames for eval"), so the two numbers may not be directly comparable (GUESS).
- **Egon Elbre's entry:** [egonelbre/exp/physicscompress](https://github.com/egonelbre/exp/tree/master/physicscompress) already used **automated random search** over arithmetic-coder parameters. VERIFIED: `tries = flag.Int("tries", 1e6, "how many random tries should it make")`. The comment at the top of `main.go` records "#2831 265.434kbps ±127.341kbps".
- **Dataset availability:** the `.bin` files are not in any of these repositories (Giesen's `.gitignore` excludes `*.bin`), and gafferongames.com is blocked here. Whether the dataset can still be obtained is an open question.

### 3.4 Modern engines
- **Unity Netcode for Entities** (`manage-serialization-costs.md`, `compression.md`). VERIFIED:
  - "Replicated fields are delta-encoded against a predicted value (similar to a linear extrapolation) that's extrapolated from the last three baselines (the last three previously acknowledged values)."
  - "Netcode's quantization is optimized for [Huffman delta compression] to be executed on top of it".
  - "Sending and receiving ghost data involves expensive CPU read and write operations that scale linearly with the number of ghosts serialized in a packet."
  - Using a single baseline: "deserialization time is often reduced by around 50%."
  - "Unity Netcode defaults to a bandwidth-intensive snapshot send configuration to enable you to get up and running quickly."
- **Unreal Iris.** SNIPPET, [introduction-to-iris-in-unreal-engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/introduction-to-iris-in-unreal-engine): Iris works by "keeping a full copy of all replicated state data in a quantized form".
- **Oodle Network (RAD Game Tools, built into Unreal): a trained packet compressor.** SNIPPET (cbloomrants.blogspot.com, radgametools.com and dev.epicgames.com are blocked):
  - Charles Bloom, 2014-06-18, [oodle network test results](https://cbloomrants.blogspot.com/2014/06/06-18-14-oodle-network-test-results.html): "Oodle Network Compression provides > 28% reduction in all cases seen so far". Source-engine packets: "798.2 -> 519.6 average = 1.536:1 = 34.90% reduction".
  - Vendor page claims "over 6 to 1", which is "twice as much as zlib" (marketing; no method given).
  - Epic's [oodle-network docs](https://dev.epicgames.com/documentation/unreal-engine/oodle-network): you capture gameplay packets, train dictionaries, and "if your game changes patch to patch in appreciable ways, you need to retrain Oodle Network on new packet captures".
- **Godot.** VERIFIED in [`scene/main/multiplayer_api.cpp`](https://github.com/godotengine/godot/blob/master/scene/main/multiplayer_api.cpp): variant "compression" packs bools and integer widths, and "// Any other case is not yet compressed."
  - The transport layer offers generic compressors ([`ENetConnection.xml`](https://github.com/godotengine/godot/blob/master/modules/enet/doc_classes/ENetConnection.xml)): `COMPRESS_RANGE_CODER` ("ENet's built-in range encoding. Works well on small packets"), FastLZ, zlib and zstd.
  - Nothing is trained on game data, and there is no quantisation.
- **Header compression and multiplexing for MMORPG TCP flows.** Saldana et al., arXiv [2007.15373](https://arxiv.org/abs/2007.15373). VERIFIED: "The obtained bandwidth saving is about 60 percent." This applies only "when the packets of a number of players share the same link", for example between proxies and servers.

### 3.5 Learned compression in neighbouring areas
- **CurveCodec 2** (arXiv [2610.04211](https://arxiv.org/abs/2610.04211), v2 31 Oct 2026; skeletal *animation* compression, offline assets). VERIFIED:
  - "CurveCodec 2 needs 0.37x ACL's bytes at ACL's default precision of 0.01 cm under the worst-case contract and 0.22x at 0.1 cm under the mean contract". ACL is the animation-compression library used in production game engines.
  - Key lesson for snapshot coding: "a nearest-neighbour oracle over millions of training samples is no better than linear interpolation, and no learned in-betweener we tried paid for itself. What a network does learn is the distribution of the residuals the codec must send."
  - Also: "a small learned model whose integer inference is bit-exact across platforms".
  - GUESS: this matches Giesen's and Oodle's experience. Most of the gain from learning comes from a better probability model for residuals (the entropy-coding step), not from learned prediction.

---

## 4. Bounded inconsistency (Dyconits)

- **Donkervliet, Cuijpers and Iosup, "Dyconits: Scaling Minecraft-like Services through Dynamically Managed Inconsistency", ICDCS 2021, DOI [10.1109/ICDCS51616.2021.00021](https://doi.org/10.1109/ICDCS51616.2021.00021).** VERIFIED (abstract via the Semantic Scholar batch API; the PDF host atlarge-research.com is blocked).
  - "Minecraft-like games only scale using isolated instances that support at most a few hundred players in the same virtual world".
  - Existing techniques "either do not work for high-density areas ... or can introduce an unbounded amount of inconsistency that can lower the quality of experience."
  - "Dyconits allow game developers to partition offline the game-world and its objects into units, each with its own bounds. The Dyconits system controls, dynamically and policy-based, the creation of dyconits and the management of their bounds."
  - Result: "Our approach supports up to 40% more concurrent players and reduces network bandwidth by up to 85%, with only minor modifications to the game and without increasing game latency."
- **Library and pluggable policy.** VERIFIED in [atlarge-research/opencraft-dyconits](https://github.com/atlarge-research/opencraft-dyconits) (release 1.2.1, 2022-09-27), Kotlin:
  - README: "The policy must be provided by the library user and controls which state updates are forwarded to which subscribers, and controls the consistency bounds for each subscriber."
  - `DataClasses.kt`: `data class Bounds(val staleness: Int, val numerical: Int)` with "Staleness is measured in milliseconds."
  - `DyconitPolicy.kt` has exactly four methods: `globalUpdate()`, `update(sub)`, `weigh(message)` and `computeAffectedDyconit(any)`. **This is a small, well-defined target for program search.**
- **Built-in policies.** VERIFIED in [atlarge-research/opencraft](https://github.com/atlarge-research/opencraft) (HEAD 2026-11-01), `messaging/dyconits/policies/`: `ZeroBoundsPolicy`, `InfiniteBoundsPolicy`, `SingleDyconitPolicy` and `ChunkPolicy`.
  - `ChunkPolicy` subscribes each player to every chunk within view distance with `new Bounds(Integer.MAX_VALUE / 2, 2)`, which is no staleness limit and a numerical bound of 2.
  - It also re-subscribes only after the player moves (`previous.distanceSquared(location) < 256` means skip).
- **A ready-made experiment harness.** VERIFIED in the [opencraft-tutorial Dyconits exercise README](https://github.com/atlarge-research/opencraft-tutorial/blob/main/exercises/dyconits/README.md):
  - "you will evaluate the effect of bounded inconsistency on Opencraft's bandwidth usage. During this experiment, we will connect 50 players, and let these players move around on a flat plane."
  - "_Expected runtime: 5 minutes_".
  - "The chunk policy effectively implements Area of Interest (AoI)".
  - The open question it leaves: "Can we further reduce the network usage? Can we do so without creating inconsistencies large enough to be noticed by players?"
- **Related benchmark from the same group.** Yardstick, ICPE 2019, DOI 10.1145/3297663.3310307. VERIFIED abstract (Semantic Scholar): "Our findings indicate the scalability limits of these servers, that Minecraft-like services are poorly parallelized".

---

## 5. Measured numbers: bytes per player per second, and savings against a stated baseline

### 5.1 Per-player traffic

| Game or system | Number | Label and source |
|---|---|---|
| Tribes (1998), design budget | 2 KB/s per client (28.8 kbit/s modem) | SNIPPET, Frohnmayer and Gift via scribd |
| Quake III default client `rate` | 3000 B/s (24 kbit/s); allowed 1,000–90,000 B/s | VERIFIED code `sv_client.c` (kbit/s conversion is GUESS) |
| Quake III with no interest management | about 12n kbit/s per player for n players | VERIFIED, Donnybrook p.1 |
| Counter-Strike (busy server, 22 slots, 2002) | about 40 kbit/s per player; 800–900 kbit/s server total; 700–800 packets/s | SNIPPET, Feng et al., IMW 2002 (conferences.sigcomm.org blocked) |
| World of Warcraft, server to client | mean TCP payload 314 B at 6.05 packets/s → about 15.2 kbit/s payload (about 17.1 kbit/s with 40-byte TCP/IP headers) | VERIFIED table (Saldana et al. 2007.15373, Table II: "WoW 314 6.05"); kbit/s is GUESS: 314 × 6.05 × 8 = 15.2 kbit/s |
| World of Warcraft, client to server | 8.74 B at 9.51 packets/s | VERIFIED, same Table II |
| ShenZhou Online and Runes of Magic, server to client | 114 B at 8 packets/s; 99 B at 5.17 packets/s → about 7.3 and 4.1 kbit/s payload | VERIFIED table; kbit/s is GUESS |
| ShenZhou Online (MMORPG, 1,356 million packets) | "MMORPG and FPS games ... both generate small packets and require low bandwidths" | UNVERIFIED (Semantic Scholar machine TLDR of Chen et al., Computer Networks 2006) |
| Halo: Reach | 45 kbit/s maximum to one client; at least 250 kbit/s host upstream for 16 players | UNVERIFIED (secondary summary of GDC 2011) |
| Donnybrook, 1,000 players | 12 Mbit/s → 670 kbit/s per player | VERIFIED |
| Fiedler 901-cube physics demo, 60 Hz | 17.37 Mbit/s raw → 255.3 kbit/s (Fiedler) → about 213.6 kbit/s (Giesen, other dataset) | VERIFIED (Giesen's kbit/s is GUESS from 445 B × 60 × 8) |

GUESS summary: classic MMORPGs needed roughly 4–20 kbit/s downstream per player (WoW, ShenZhou and Runes of Magic rows), and FPS games roughly 24–45 kbit/s (Quake III default, Counter-Strike, Halo: Reach). Physics-heavy scenes with hundreds of moving objects reach hundreds of kbit/s even after heavy compression. I found no peer-reviewed per-player measurement for 2021–2026 titles (Fortnite, VALORANT, CS2); this is an open question.

### 5.2 Savings against a stated baseline

| Technique | Saving | Baseline | Label |
|---|---|---|---|
| Interest management that accounts for obstacles | up to 6× fewer update messages | other interest-management algorithms tested (exact baseline not visible) | VERIFIED abstract |
| Donnybrook interest sets plus doppelgängers | 12 Mbit/s → 670 kbit/s per player at 1,000 players | all-to-all Quake III updates | VERIFIED |
| Dyconits | up to −85 % bandwidth, +40 % players | unmodified Minecraft-like game (Opencraft) | VERIFIED abstract |
| Fiedler quantisation | orientation 128 → 29 bits; cube 321 → 80 bits | 32-bit floats | VERIFIED |
| Fiedler full pipeline (challenge) | 1.48 % of raw | 32-bit floats, 901 cubes | VERIFIED |
| Context-modelling arithmetic coding | 25 % (Fiedler's statement); 16 % by my comparison of the two logged numbers | Fiedler's best bit-packer | VERIFIED quote; 16 % is GUESS (datasets may differ) |
| Oodle Network trained dictionaries | more than 28 % (tiny packets 28.7 %, Source packets 34.9 %) | uncompressed game packets | SNIPPET |
| Unity single baseline | about 50 % less client deserialisation time | three-baseline predictive delta | VERIFIED docs |
| TCM header compression and multiplexing | about 60 % client to server | native TCP/IP headers on shared links | VERIFIED abstract |
| Halo: Reach | down to 20 % of Halo 3 bandwidth | Halo 3 | UNVERIFIED |
| VALORANT replication rewrite (properties to RPCs) | 4.54 ms → 0.45 ms of server frame time | UE4 property replication | UNVERIFIED (gigazine and techbang summaries of Riot's blog) |

---

## 6. Cost context: does saving bytes save Miles money?

- **Hetzner traffic policy.** VERIFIED, [docs.hetzner.com/robot/general/traffic](https://docs.hetzner.com/robot/general/traffic/), "Last change on 2024-05-21":
  - "All root servers have a dedicated 1 GBit uplink by default and with it unlimited traffic."
  - "We will charge € 1 ($1.20)/TB for overusage."
  - "CX Cloud Server 20 TB for EU"; "CPX Cloud Server 20 TB for EU"; "CCX Cloud Server 20-60 TB for EU locations".
  - "We calculate monthly traffic only using outgoing traffic."
- **AWS egress.** UNVERIFIED (third-party calculators; aws.amazon.com was not checked): $0.09/GB for the first 10 TB per month, with 100 GB/month free.
- **Arithmetic** (all GUESS; computed in Python):
  - 20 TB/month = 20e12 B / 2,592,000 s = 7.7 MB/s ≈ **61.7 Mbit/s sustained**.
  - Players online 24/7 that fit inside 20 TB: about 4,115 at 15 kbit/s, 1,543 at 40 kbit/s, 965 at 64 kbit/s, and 241 at 256 kbit/s.
  - One player-hour at 40 kbit/s = 18 MB. That costs about €0.000018 in Hetzner overage, or about $0.0016 on AWS at $0.09/GB.
  - A 10,000-concurrent-player game at 40 kbit/s needs 129.6 TB/month (400 Mbit/s). That is about **€110/month** of Hetzner cloud overage (or €0 on a 1 Gbit dedicated server), against about **$11,700/month** at a flat $0.09/GB on AWS.
- **CPU is the real cost driver in commercial titles.**
  - SNIPPET, Riot [VALORANT's 128-Tick Servers](https://technology.riotgames.com/news/valorants-128-tick-servers) (blocked): "Utilizing 1 full core per game would make it prohibitively expensive to host our game servers at scale." Also: "VALORANT's servers are optimized to fit 3 frames into ~7.8ms."
  - SNIPPET, Epic Replication Graph: per-actor, per-connection relevancy "constrains server CPU".
  - VERIFIED, Unity: serialisation CPU "scale[s] linearly with the number of ghosts serialized in a packet".
- **What this means** (GUESS): on Hetzner, the value of family A is in (a) server CPU spent on relevancy and serialisation, (b) quality for players on weak or mobile links, and (c) only marginally the bandwidth bill. A pure "bytes" metric is the wrong objective for Miles's costs. Use CPU per tick as a co-objective, or as the main objective.

---

## 7. Earlier ML and AI attempts on these problems

| Attempt | What it did | Result | Label |
|---|---|---|---|
| Quake III static Huffman table (1999) | byte frequencies measured from captured traffic seed the coder | shipped; no number given | VERIFIED code |
| Oodle Network (about 2013–present) | dictionaries and models trained on captured packets | more than 28 % reduction; must retrain per patch | SNIPPET |
| Fiedler 2015 | greedy search (Ruby script) for bit-packing ranges on a captured dataset | used in the shipped encoding | VERIFIED |
| Elbre 2015 | random search (1e6 tries) over arithmetic-coder parameters | 265.4 kbit/s on the challenge data | VERIFIED |
| McCoy, Ward, McLoone and Delaney, ACM TOMACS 2007, DOI [10.1145/1276927.1276929](https://doi.org/10.1145/1276927.1276929) ("neuro-reckoning") | a bank of neural nets predicts velocity changes; the sender transmits a predictive velocity vector when error exceeds a threshold | "Simulation results conducted using several human users in a highly interactive DIA indicate significant potential for improved scalability when compared to the use of IEEE DIS standard dead reckoning." (no figure in abstract) | VERIFIED abstract (Crossref) |
| Duarte, Pozo and Beltrani, Entertainment Computing 2020, DOI 10.1016/j.entcom.2019.100336 ("Smart Reckoning") | a classifier predicts when dead reckoning will fail, and a second model predicts the new direction; World of Warcraft traces; Weka | "results are presented for the accuracy of multiple algorithms" (accuracy only, no end-to-end bandwidth) | VERIFIED abstract (Semantic Scholar) |
| Walker, Gilhuly, Sadeghi, Delbosc and Smith, IEEE Transactions on Games 2024, DOI 10.1109/TG.2023.3237943 ("Predictive Dead Reckoning") | neural-net position predictor plus a path-tracking controller from robotics; racing cars | "replicating opponents with less data sent and up to 45% less error compared to the state-of-the-art" | VERIFIED abstract (Semantic Scholar) |
| DIA, arXiv 2610.13698 (2026) | deep RL, object-level bitrate for cloud-gaming video | "at least a 20% higher normalized average ... QoE" in 5G | VERIFIED abstract; adjacent problem |
| CurveCodec 2, arXiv 2610.04211 (2026) | learned entropy model for animation residuals | 0.37× ACL's bytes (worst-case contract) | VERIFIED abstract; adjacent problem |
| **LLM program search on network heuristics (not games)** | NADA (arXiv [2404.01617](https://arxiv.org/abs/2404.01617) v2): LLM-written ABR variants, filtered then evaluated | "produces novel ABR algorithms -- previously unknown to human developers -- that consistently outperform the original algorithm in diverse network environments" | VERIFIED abstract |
| | PolicySmith (arXiv [2510.08803](https://arxiv.org/abs/2510.08803)): web caching and congestion control | "discovers heuristics that outperform established baselines on standard open-source traces ... can generate safe policies that integrate directly into the Linux kernel" | VERIFIED abstract |
| | ReForge (arXiv [2608.15138](https://arxiv.org/abs/2608.15138)): an LLM edits a page of ABR rules; a replay over all past networks accepts or rejects each edit | "lift mean QoE from 1.23 to 1.74, past the best single policy at 1.66 and to 94% of an oracle"; motivation: designs fail "on the world that arrives after" | VERIFIED abstract |
| | Glia (arXiv [2510.27176](https://arxiv.org/abs/2510.27176)): multi-agent LLM designs routing, scheduling and autoscaling for LLM inference clusters | "perform at human-expert levels in significantly less time" | VERIFIED abstract |
| **LLM program search on game netcode, interest management or snapshot compression** | none found | — | GUESS (absence of evidence; several targeted searches) |

---

## 8. Candidate problems for AI search (one per sub-problem)

Common evaluator design (applies to all five):
- Replay recorded or bot-generated traces through a headless simulator.
- Seed everything so runs are deterministic.
- Keep held-out traces (other maps, densities and loss patterns) that the search never sees.
- Run the candidate code in a sandbox with a time limit, and never let it see the evaluator or the held-out data. The T1 notes document reward-hacking cases.
- Always report a CPU-time co-metric.

### A1. Interest-management (relevance) policy for crowded zones
- **Metric:** total entity updates (or bytes, using a fixed per-entity size) sent to all clients per second, over a trace, **subject to** relevance recall of at least 99.9 %. Recall means: for every (client, entity, tick) where the entity is within R metres **and** visible by ray-cast ground truth, **or** has interacted with the client in the last 3 s, the entity is in the client's set with state no older than 100 ms. The co-metric is server CPU µs per tick for N = 1,000–10,000 entities.
- **Evaluator:** load map occluders and per-tick positions. Call the candidate interest function each tick. Count updates. Compare with ground truth computed offline. Time the function. Include power-law crowd scenarios (Donnybrook: density "follows a power law").
- **Data:** position traces plus map geometry (from Miles's servers, or bots; Boulanger found random-action bots approximate humans "provided that the starting positions ... are chosen adequately").
- **Baselines:**
  - tower or grid AoI (499 ms for 10,000 moves of 10,000 units on an i7-7700K);
  - obstacle-aware or tile-based visibility (up to 6× fewer messages, and cheap tile methods approach the visibility ideal);
  - Donnybrook interest sets of size 5.
- **Headroom:** large against radius AoI, but **small against the best cheap tile or visibility methods** (Boulanger), and grids are already near-optimal for CPU (GUESS).
- **Prior AI attempts:** none found.
- **Risks:**
  - "Relevance" is a design choice (sound, abilities, anti-cheat), so a recall metric can be gamed or simply be wrong.
  - Map-specific overfitting.
  - It adds nothing for small-arena games like Sumo Brothers.
  - Bandwidth is almost free on Hetzner.

### A2. Replication priority (importance) function under a per-client bandwidth budget (**best fit**)
- **Metric:** the area under the error-versus-budget curve. Error is the mean and p99 position error (metres, or screen-space pixels) between each client's reconstructed state (using the game's interpolation or extrapolation) and server truth, weighted by on-screen relevance. Budgets: 32, 64 and 128 kbit/s per client. Loss: 0 %, 2 % and 5 %. The search changes only the priority function; the packing, budget and accumulator mechanics stay fixed.
- **Evaluator:** headless deterministic scene plus a seeded network emulator (latency, jitter, loss) plus a client reconstruction model. The candidate has the shape of Unity's `BatchScaleImportanceFunction`, Iris `UNetObjectPrioritizer` or Torque `getUpdatePriority`. Each run takes seconds.
- **Data:** recorded server state per tick (positions, velocities, events) from real matches, plus synthetic stress scenes.
- **Baselines:** Fiedler's priority accumulator; Unity importance × `ticksSinceLastSent` with distance tiles; Iris accumulating priorities with the sphere prioritiser; Torque's 5-term weighted sum.
- **Headroom:**
  - Every production version uses hand-set constants (Torque 1.0/0.4/0.4/0.2/0.2; Donnybrook "several hours of tuning"; Fiedler 1,000,000/100/1).
  - Torque shipped a logical-OR bug in this function that nobody's metrics caught.
  - Halo: Reach reportedly found big wins by profiling replays (UNVERIFIED).
  - GUESS for the size: 10–30 % lower error at equal bandwidth against a well-tuned accumulator; more against naive per-tick sending.
- **Prior AI attempts:** none for games. Analogues: NADA, ReForge and DIA for streaming video.
- **Risks:**
  - Error is not the same as perception (Donnybrook had to use a user study).
  - Interaction with the client smoothing model.
  - Overfitting to particular scenes.
  - It matters only when bandwidth-constrained, so it improves experience on weak links but barely changes Hetzner cost.

### A3. Snapshot delta plus entropy coding (lossless with respect to quantised state)
- **Metric:** mean bytes per snapshot (or kbit/s at 60 Hz) over a fixed dataset, **with a hard requirement that every snapshot decodes bit-exactly**. A secondary constraint is encode plus decode time of at most X µs per snapshot per client on one core. Score on held-out segments or datasets.
- **Evaluator:** the 2015 challenge harness (Fiedler's gist prints "average bytes per-packet" and verifies decoding) or Giesen's `main.cpp`. Each evaluation takes milliseconds. Add CPU timing and a held-out split.
- **Data:** the challenge `delta_data.bin` (2,837 frames × 901 cubes; **not found in any accessible repository**), or Miles's own recorded Godot snapshots.
- **Baselines:**
  - Fiedler 531.95 B per packet (255.3 kbit/s, 1.48 % of raw);
  - Giesen 445 B per frame (about 213.6 kbit/s; dataset `realnew`);
  - Elbre 265.4 kbit/s (random-search-tuned);
  - generic trained compressor (Oodle) more than 28 % on raw packets (SNIPPET).
- **Headroom:**
  - Giesen's curve flattened to 1.5 % per step or less.
  - Fiedler named an unexploited idea (predicting orientation from estimated angular velocity).
  - CurveCodec 2 shows learned residual models beat production codecs on animation data.
  - GUESS: 5–15 % beyond Giesen.
- **Prior AI attempts:** greedy and random parameter search (2015); trained models (Quake III 1999, Oodle). No LLM-evolution attempt found.
- **Risks:**
  - Overfits to one dataset or game patch (Epic: retrain per patch).
  - Arithmetic-coding CPU cost per client on the server.
  - Physics-cube data are not MMO data.
  - Saved bytes are worth about €0 on Hetzner.
  - The dataset may be unobtainable.

### A4. Bounded-inconsistency (Dyconit) policy for Minecraft-like worlds
- **Metric:** mean client-bound bytes per player-second (and the maximum number of players at a 50 ms tick-time target) under a fixed bot workload, **subject to** p99 staleness of at most 200 ms for blocks and entities within k chunks of the player, plus a numerical-error bound, plus no added latency.
- **Evaluator:** Opencraft plus the candidate `DyconitPolicy` (four methods). Run the opencraft-tutorial 50-bot experiment ("Expected runtime: 5 minutes") with several seeds. For speed, build an offline replay of logged publish events and subscriber positions so policies are scored in seconds, and confirm only the top candidates on the real system.
- **Data:** the public Opencraft and Yardstick harness; logged message streams; ideally human traces.
- **Baselines:** the Dyconits paper (−85 % bandwidth, +40 % players against the unmodified game); the public `ChunkPolicy` (`Bounds(Integer.MAX_VALUE / 2, 2)`), `ZeroBoundsPolicy` and `InfiniteBoundsPolicy`.
- **Headroom:** the policy space is barely explored (four simple policies in the repository), but the paper's 85 % probably captured most of the easy savings (GUESS).
- **Prior AI attempts:** none found.
- **Risks:**
  - Evaluation on the real system is slow and noisy.
  - The inconsistency metric has not been validated against what players notice ("without creating inconsistencies large enough to be noticed by players?").
  - The work is specific to Minecraft-like Java games.

### A5. Send-on-error extrapolation (dead-reckoning) policy
- **Metric:** updates (or bytes) per entity-second on held-out human movement traces, **subject to** p99 remote position error of at most ε (for example 0.1 m), plus a penalty for jerk and teleport events above a threshold as a stand-in for "believability".
- **Evaluator:** offline replay. The sender runs a copy of the receiver's predictor and sends whenever the error rule fires. The evolved code is predictor plus send rule plus receiver correction or smoothing.
- **Data:** per-game movement traces with collision events.
- **Baselines:**
  - IEEE DIS dead reckoning;
  - neuro-reckoning (McCoy 2007);
  - neural-net predictor plus path-tracking controller (Walker 2024: "up to 45% less error").
  - SNIPPET (Walker's Waterloo thesis summary): the path-tracking controller performed better than both.
- **Headroom:** three ML papers show better prediction, but none reports end-to-end bandwidth at equal perceived quality.
- **Prior AI attempts:** McCoy 2007, Duarte 2020, Walker 2024.
- **Risks:**
  - Game-specific.
  - Needs many traces.
  - In a physics sumo game, errors cluster at collisions.
  - In server-authoritative games with interpolation buffers, the gain may be small.

---

## 9. Verdict for this family (feasibility and where to aim)

**Technically feasible, and genuinely new. Economically minor for Miles.**

Why it suits AI search:
- Small, pluggable functions that real engines already expose.
- Cheap, deterministic trace-replay evaluators; the 2015 compression challenge and the Opencraft tutorial already exist.
- Documented human hill-climbing curves.
- No prior LLM-evolution attempt found.

Why the payoff is limited:
- The baselines are strong: textbook interest management, priority accumulators, quantisation and delta coding already remove 85–98 % of naive traffic.
- The remaining headroom is probably 5–30 % (GUESS).
- On Hetzner, bandwidth is effectively free; CPU, not bytes, is the cost.
- The metrics that can be computed automatically (position error, recall) are only stand-ins for what players notice.

Recommended order for Miles:
1. Implement the known techniques in his Godot netcode first: quantise, delta against acknowledged state, priority accumulator with a per-client budget, visibility filter. This is engineering, not discovery, and will likely give far larger gains than any AI search, because Godot ships none of them.
2. Record match traces from Sumo Brothers.
3. If he still wants to try AI search, aim first at **A2 (priority function, scored by error at a fixed budget, with a CPU co-metric)** or **A3 (entropy coder, scored in bytes with exact decoding)**. Keep held-out traces, and verify any winner with a short human playtest.

---

## 10. Open questions
- Can the 2015 Fiedler challenge datasets (`delta_data.bin`, `delta_data_realnew.bin`, `delta_data_round_two.bin`) still be obtained (gafferongames.com is blocked here)? Why does Giesen's 445 B per frame against Fiedler's 531.95 B per packet give 16 %, not the stated 25 %?
- What baseline is behind Boulanger's "factor of 6"? The thesis PDF (cs.mcgill.ca) is blocked.
- Are there peer-reviewed per-player bandwidth measurements for current titles (Fortnite, VALORANT, CS2, modern MMOs)?
- What is the actual per-tick CPU split (simulation versus relevancy versus serialisation) in a Godot server running Sumo Brothers? This decides whether family A matters at all for his costs.
- Is there a validated, automatically computable proxy for "players notice this inconsistency"? None was found; Donnybrook used a fun score from humans.
- Is the Halo: Reach claim ("20% of Halo 3 bandwidth via replay profiling") documented in the GDC slides?

## 11. Blocked hosts (used snippets or metadata instead)
www.sable.mcgill.ca, www.comp.nus.edu.sg, www.cs.mcgill.ca (Boulanger PDFs), export.arxiv.org (the arXiv API redirects there; the arxiv.org search page and PDFs work), gist.githubusercontent.com (gist raw; the gist HTML page worked), conferences.sigcomm.org (Counter-Strike paper), dev.epicgames.com, technology.riotgames.com, cbloomrants.blogspot.com, www.radgametools.com, atlarge-research.com, www.scribd.com, gafferongames.com, ng.cs.ucl.ac.uk, es.slideshare.net, thefengs.com, edgegap.com. The api.openalex.org daily budget was exhausted after one lookup, and api.semanticscholar.org was intermittently rate-limited (batch endpoint worked). www.microsoft.com **was** reachable (Donnybrook PDF).

## 12. Claims table

| ID | Status | Claim | Source | Quote (short) |
|---|---|---|---|---|
| T3-01 | VERIFIED | Obstacle-aware interest management cuts update messages up to 6× | https://api.openalex.org/works/doi:10.1145/1230040.1230069 | "reduce the number of update messages between players by up to a factor of 6" |
| T3-02 | VERIFIED | Cheap tile-based interest management approximates ideal visibility-based interest management | same | "can approximate ideal visibility-based interest management at very low cost" |
| T3-03 | VERIFIED | Random bots approximate human measurements if start positions are chosen well | same | "provided that the starting positions of the random players are chosen adequately" |
| T3-04 | VERIFIED | The 2014 ACM Computing Surveys paper is the standard interest-management survey | https://doi.org/10.1145/2535417 (Crossref abstract) | "a comprehensive survey of the state of the art in the design of interest management algorithms and systems" |
| T3-05 | VERIFIED | Quake III without interest management: about 12n kbit/s per player | https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/donnybrook.pdf | "at ∼12n kb/s per player for n players in Quake III" |
| T3-06 | VERIFIED | Interest sets stay small because human attention is limited | same | "an interest set always remains small due to the limits of human attention" |
| T3-07 | VERIFIED | Donnybrook: 1,000 players, 12 Mbit/s → 670 kbit/s | same | "reduces the requirement from 12 Mb/s to 670 Kb/s" |
| T3-08 | VERIFIED | Donnybrook attention weights were hand-tuned in hours and need retuning per game | same | "With only several hours of tuning on our part" |
| T3-09 | VERIFIED | Donnybrook's validation used a human fun-score study; the low-bandwidth interest-set version was statistically indistinguishable from high bandwidth | same | "the difference between LoBW-IS and HiBW is statistically insignificant" |
| T3-10 | VERIFIED | Player density in real games follows a power law, which breaks radius AoI | same | "population density in real games follows a power law" |
| T3-11 | VERIFIED | Interest sets turn over 68 % per second | same | "average turnover of 68% per second" |
| T3-12 | VERIFIED | Quake III culls entities by PVS cluster | https://github.com/id-Software/Quake-III-Arena/blob/master/code/server/sv_snapshot.c | "clientpvs = CM_ClusterPVS (clientcluster);" |
| T3-13 | VERIFIED | Quake III deltas against the last acknowledged frame | same | "try to use a previous frame as the source for delta compressing the snapshot" |
| T3-14 | VERIFIED | Quake III seeds its Huffman coder from a fixed table of measured byte frequencies | https://github.com/id-Software/Quake-III-Arena/blob/master/code/qcommon/msg.c | "int msg_hData[256] = {" |
| T3-15 | VERIFIED | Quake III default client rate is 3000 B/s, clamped to 1000–90000 | https://github.com/id-Software/Quake-III-Arena/blob/master/code/server/sv_client.c | "cl->rate = 3000;" |
| T3-16 | VERIFIED | Torque priority = hand-weighted sum (FOV 1.0, distance 0.4, velocity 0.4, skips 0.2, interest 0.2) | https://github.com/GarageGames/Torque3D/blob/master/Engine/source/T3D/gameBase/gameBase.cpp | "static F32 sUpFov       = 1.0;" |
| T3-17 | VERIFIED | Torque master has a `\|\|` bug that gives the interest bonus to static objects, not players or vehicles; fixed on the development branch | same, plus TorqueGameEngines development branch | "getTypeMask() & (PlayerObjectType \|\| VehicleObjectType )" |
| T3-18 | VERIFIED | Fiedler priority accumulator with a bandwidth budget | https://github.com/gafferongames/gafferongames/blob/master/content/post/state_synchronization.md | "reset the priority accumulator to zero for objects that fit" |
| T3-19 | VERIFIED | Unity: fixed bandwidth target, priority queue by importance × ticksSinceLastSent | https://github.com/needle-mirror/com.unity.netcode (Documentation~/optimization/optimize-ghosts.md) | "The server operates with a fixed bandwidth target" |
| T3-20 | VERIFIED | Unity exposes a pluggable importance-scaling function | same | "GhostImportance.BatchScaleImportanceFunction" |
| T3-21 | VERIFIED | Unity serialisation CPU scales linearly with ghosts | same repo, manage-serialization-costs.md | "scale linearly with the number of ghosts serialized in a packet" |
| T3-22 | VERIFIED | Unity predictive delta against an extrapolation from 3 baselines; single baseline cuts client deserialisation about 50 % | same | "deserialization time is often reduced by around 50%" |
| T3-23 | SNIPPET | Unreal Iris accumulates priorities; supports custom `UNetObjectPrioritizer` | https://dev.epicgames.com/documentation/en-us/unreal-engine/iris-prioritization-in-unreal-engine | "Not every replicated object or property is updated each frame due to limited bandwidth." |
| T3-24 | SNIPPET | Replication Graph motivated by Fortnite's 100 players and about 50,000 actors and the CPU limit | https://dev.epicgames.com/documentation/zh-cn/unreal-engine/replication-graph-in-unreal-engine | (search paraphrase) "constrains server CPU" |
| T3-25 | VERIFIED | Godot MultiplayerSynchronizer syncs to all peers by default; boolean visibility filter per peer | https://github.com/godotengine/godot/blob/master/modules/multiplayer/doc_classes/MultiplayerSynchronizer.xml | "synchronizes configured properties to all peers" |
| T3-26 | VERIFIED | Godot variant compression covers only bool and int | https://github.com/godotengine/godot/blob/master/scene/main/multiplayer_api.cpp | "Any other case is not yet compressed." |
| T3-27 | VERIFIED | Fiedler demo: 17.37 Mbit/s raw at 60 Hz; target 256 kbit/s | https://github.com/gafferongames/gafferongames/blob/master/content/post/snapshot_compression.md | "17.37 megabits per-second!" |
| T3-28 | VERIFIED | Quantisation takes the cube from 321 to 80 bits | same | "This reduces our cube state to 80 bits" |
| T3-29 | VERIFIED | Fiedler used greedy search on captured data to choose encodings | same | "I wrote a short ruby script to find the best encoding with a greedy search." |
| T3-30 | VERIFIED | Giesen's arithmetic coder beat Fiedler's bit-packer by 25 % (Fiedler's statement) | same | "beat my best bit-packed result by 25%" |
| T3-31 | VERIFIED | Fiedler challenge harness result: 531.95 B per packet, 255.3 kbit/s, 1.48 % of raw | https://gist.github.com/gafferongames/bb7e593ba1b05da35ab6 | "average kilobits per-second: 255.336941" |
| T3-32 | VERIFIED | Giesen's iterative log: 3435 → 572 → 445 B per frame over 3 days | https://github.com/rygorous/gaffer_net (git log) | "Even more flag craziness. 445b/frame" |
| T3-33 | GUESS | 445 B per frame at 60 Hz = 213.6 kbit/s; 16 % below Fiedler's 531.95 (datasets may differ) | arithmetic | 445 × 60 × 8 / 1000 |
| T3-34 | VERIFIED | Elbre used random search (1e6 tries) over coder parameters; 265.4 kbit/s | https://github.com/egonelbre/exp/tree/master/physicscompress | "how many random tries should it make" |
| T3-35 | SNIPPET | Oodle Network trained compression: more than 28 % reduction on real game packets; Source packets 34.9 % | https://cbloomrants.blogspot.com/2014/06/06-18-14-oodle-network-test-results.html | "> 28% reduction in all cases seen so far" |
| T3-36 | SNIPPET | Oodle dictionaries must be retrained when a patch changes traffic | https://dev.epicgames.com/documentation/unreal-engine/oodle-network | "retrain Oodle Network on new packet captures" |
| T3-37 | VERIFIED | Dyconits: −85 % bandwidth, +40 % players, no added latency | https://api.semanticscholar.org/graph/v1/paper/DOI:10.1109/ICDCS51616.2021.00021 | "reduces network bandwidth by up to 85%" |
| T3-38 | VERIFIED | Dyconits policy is a user-supplied interface; bounds are (staleness ms, numerical) | https://github.com/atlarge-research/opencraft-dyconits | "The policy must be provided by the library user" |
| T3-39 | VERIFIED | Opencraft tutorial: 50-bot, 5-minute bandwidth experiment; open question on noticeable inconsistency | https://github.com/atlarge-research/opencraft-tutorial/blob/main/exercises/dyconits/README.md | "without creating inconsistencies large enough to be noticed by players?" |
| T3-40 | VERIFIED | WoW server-to-client mean payload 314 B at 6.05 packets/s; client-to-server 8.74 B at 9.51 packets/s | https://arxiv.org/abs/2007.15373 (Table II) | "WoW 314 6.05 8.65 %" |
| T3-41 | GUESS | WoW downstream ≈ 15.2 kbit/s payload (≈17.1 kbit/s with 40-byte headers) | arithmetic | 314 × 6.05 × 8 / 1000 |
| T3-42 | SNIPPET | Counter-Strike about 40 kbit/s per player (busy 22-slot server, 2002) | http://conferences.sigcomm.org/imc/2002/imw2002-papers/168.pdf | "the bandwidth consumed per player is on average 40kbps" |
| T3-43 | SNIPPET | Tribes budget about 2 KB/s per client | https://www.scribd.com/document/278495122/Tribes-Networking-Model | "a meager 2 Kbytes per second of bandwidth limitation on each client" |
| T3-44 | UNVERIFIED | Halo: Reach: 45 kbit/s max per client; replay profiling cut bandwidth to 20 % of Halo 3 | https://ng.cs.ucl.ac.uk/?p=115 ; https://wolfire-blog.appspot.com/2011/03/GDC-Session-Summary-Halo-networking | "45kbits/s Maximum bandwidth sent to one client" |
| T3-45 | VERIFIED | Hetzner: unlimited traffic on dedicated (1 Gbit); 20 TB on EU cloud; €1/TB overage; outgoing only | https://docs.hetzner.com/robot/general/traffic/ | "We will charge € 1 ($1.20)/TB for overusage." |
| T3-46 | UNVERIFIED | AWS egress about $0.09/GB for the first 10 TB | https://cloudzero.com/blog/aws-egress-costs (secondary) | — |
| T3-47 | GUESS | 20 TB/month ≈ 61.7 Mbit/s ≈ 1,543 players at 40 kbit/s 24/7; a 10,000-player game ≈ €110/month on Hetzner cloud versus about $11.7k on AWS | arithmetic | — |
| T3-48 | SNIPPET | Riot: one core per VALORANT match "would make it prohibitively expensive"; 3 frames per 7.8 ms | https://technology.riotgames.com/news/valorants-128-tick-servers | "Utilizing 1 full core per game would make it prohibitively expensive" |
| T3-49 | UNVERIFIED | VALORANT replication cost cut from 4.54 ms to 0.45 ms by replacing property replication with RPCs | https://gigazine.net/news/20251009-valorant-server-sub-2ms (secondary) | — |
| T3-50 | VERIFIED | Neuro-reckoning (2007) improved on IEEE DIS dead reckoning in simulation | https://doi.org/10.1145/1276927.1276929 (Crossref abstract) | "significant potential for improved scalability when compared to the use of IEEE DIS standard dead reckoning" |
| T3-51 | VERIFIED | Smart Reckoning (2020) reported only prediction accuracy on WoW traces | https://doi.org/10.1016/j.entcom.2019.100336 (Semantic Scholar abstract) | "results are presented for the accuracy of multiple algorithms" |
| T3-52 | VERIFIED | Predictive Dead Reckoning (2024): up to 45 % less error with less data | https://doi.org/10.1109/TG.2023.3237943 (Semantic Scholar abstract) | "up to 45% less error compared to the state-of-the-art" |
| T3-53 | VERIFIED | CurveCodec 2: learned residual entropy model, 0.37× ACL's bytes; learned in-betweening did not pay | https://arxiv.org/abs/2610.04211 | "no learned in-betweener we tried paid for itself" |
| T3-54 | VERIFIED | NADA: LLM-designed ABR algorithms beat the original | https://arxiv.org/abs/2404.01617 | "consistently outperform the original algorithm in diverse network environments" |
| T3-55 | VERIFIED | PolicySmith: LLM-synthesised caching and congestion-control heuristics | https://arxiv.org/abs/2510.08803 | "discovers heuristics that outperform established baselines on standard open-source traces" |
| T3-56 | VERIFIED | ReForge: LLM-edited ABR rules, replay-verified; 1.23 → 1.74 QoE; warns designs fail on new conditions | https://arxiv.org/abs/2608.15138 | "fails on the world that arrives after" |
| T3-57 | VERIFIED | DIA: deep-RL interest-aware bitrate for cloud gaming, at least 20 % higher QoE in 5G | https://arxiv.org/abs/2610.13698 | "at least a 20% higher normalized average Quality of Experience" |
| T3-58 | VERIFIED_FIG | AoI micro-benchmark: 10,000 moves of 10,000 units: crosslink 2,772 ms, quadtree 534 ms, tower 499 ms | https://github.com/bhhbazinga/AOI (benchmark.png) | "We simulate N random moves of N units in a 1024*1024 map" |
| T3-59 | GUESS | No published LLM-evolution application to game netcode, interest management or snapshot compression exists | searches listed in §7 | — |
| T3-60 | VERIFIED | Header compression and multiplexing for MMORPG TCP traffic saves about 60 % on shared links | https://arxiv.org/abs/2007.15373 | "The obtained bandwidth saving is about 60 percent." |
| T3-61 | UNVERIFIED | Combat-state-aware interest management (2017 demo) adjusts update rates by game context | Semantic Scholar TLDR of DOI 10.1145/3155016.3155019 | "dynamically adjust update rates based on the current game context" |
| T3-62 | VERIFIED | Unity netcode defaults are deliberately bandwidth-heavy | needle-mirror com.unity.netcode, optimization/compression.md | "defaults to a bandwidth-intensive snapshot send configuration" |
| T3-63 | VERIFIED | Godot ENet offers only generic compressors (range coder, FastLZ, zlib, zstd) | https://github.com/godotengine/godot/blob/master/modules/enet/doc_classes/ENetConnection.xml | "ENet's built-in range encoding. Works well on small packets" |
