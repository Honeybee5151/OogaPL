# T4: Where the simulation runs and how much CPU it needs (candidate problem family B)

Researched 2026-11-08 for Miles (Sumo Brothers, Godot, one Hetzner Linux box). The question is whether AI-driven algorithm search (LLM-guided program search: FunSearch, AlphaEvolve, OpenEvolve and similar) is feasible here, and where it should be aimed. Nothing is being built yet.

Labels: **VERIFIED** = read in the primary source (link + exact quote). **VERIFIED_FIG** = read off a chart in the primary source (approximate). **SNIPPET** = only seen in a search-engine excerpt because the host was blocked. **UNVERIFIED** = secondary source only, or memory. **WRONG** = contradicted by the primary source. **GUESS** = my own estimate (arithmetic shown).

Quotes from PDFs were extracted with `pdftotext -layout`. Where a word was split across a line break ("low-est"), I removed the hyphen. Nothing else in the quotes was changed.

Local copies of everything I downloaded are under `/tmp/claude-0/-home-user-OogaPL/32372848-6606-541b-85af-54d683c4e916/scratchpad/dl/t4-*` (one folder per source, listed in §9).

Related notes by other agents (not redone here): T1 (AI-search track record, FunSearch details) is in `/tmp/claude-0/-home-user-OogaPL/32372848-6606-541b-85af-54d683c4e916/scratchpad/ai-search/T1-ai-search.md`. T2 (server costs, Hetzner prices, CPU per player) is in `/tmp/claude-0/-home-user-OogaPL/32372848-6606-541b-85af-54d683c4e916/scratchpad/ai-search/T2-costs.md`.

---

## 0. Bottom line in plain language

1. **Splitting a big shared world across servers or cores (dynamic partitioning) is still not solved in industry.** Each real system takes a simplified route:
   - Minecraft's Folia splits the world only where players are already far apart. A crowd in one spot still runs on one thread.
   - Star Citizen has run a *static* server mesh since Dec 2024. In 2026 it began starting and stopping servers by load, but only for instances. Splitting territories was still "Tentative" in October 2026.
   - SpatialOS used fixed grids, and Improbable later froze its developer kits.
   - EVE Online pins each solar system to one node, reshuffles nodes once a day, adds dedicated nodes by hand for announced battles, and slows game time (TiDi) when a node overloads.
2. **Academics have studied this since about 2002.** There are many heuristics: microcells, kd-trees, region-growing, locality-aware shedding and heat diffusion. Since about 2019 there have also been some reinforcement-learning attempts.
   - All of these were evaluated in small simulations, mostly with synthetic player movement.
   - A 2012 survey chapter says there is not even an agreed way to measure "load".
   - I found **no** LLM program-search attempt on this problem.
   - The obstacle is building a believable evaluator, not running the search.
3. **Packing game sessions onto as few machines as possible (bin packing over time, plus autoscaling) is the sub-problem best suited to AI search.**
   - It has a cheap, deterministic evaluator: replay a log of matches.
   - The evolved code can be a small scoring function, kept safe by checks outside it, as in the Google Borg result.
   - Good public baselines exist: Agones "Packed", GameLift target tracking, First Fit, and 2026 algorithms that use known departure times.
   - **But the headroom is small.** On real Azure VM data, plain First Fit is within about 13% of a lower bound on server-time. Knowing departure times gets within about 3%. Google's AI-evolved Borg heuristic recovered 0.7% of fleet compute, and its ML lifetime predictor (LAVA) cut stranded resources by about 2–3%.
4. **For Sumo Brothers today, this whole family is worth roughly €0.**
   - Matches are small and separate, so there is no shared world to split.
   - With one dedicated Hetzner box billed monthly, there is nothing to pack.
   - It only starts to matter when Miles runs several hourly-billed servers and his peak and off-peak loads differ a lot. Even then, a simple rule ("add a server when free slots fall below N, remove it when empty") will capture most of the saving.
   - Also, Hetzner's dedicated-vCPU cloud plans showed "currently unavailable" on 2026-11-08.
5. **Hot spots inside the simulation** (NPC level-of-detail, spatial indexing, pathfinding, saving to the database) are ordinary performance engineering. Profiling usually beats search here.
   - EVE profiled its fleet fights and found physics was under 10% of the physics-server tick.
   - The real cost was missile bookkeeping, which humans then fixed.

---

## 1. Dynamic world partitioning and load balancing (across servers or cores)

### 1.1 The problem in one paragraph
A shared world (an MMO zone, a Minecraft world, a star system) is split into pieces that run on different cores or machines. Players crowd into "hot spots": towns, sieges, world events. The system must decide when to split a busy piece, when to merge quiet pieces, and where to move entities. Moving an entity costs CPU and latency. Interactions across a boundary need extra messages, or are forbidden outright.

- Carlini's thesis, VERIFIED (arXiv [1510.08940](https://arxiv.org/abs/1510.08940), §2.2.5; local `dl/t4-arx-1510.08940/1510.08940.txt`):
  - "These areas, called hotspots, are the principal reason of the load unbalancing in virtual environment."
  - "Currently, there is no an uniform view on how to measure the load on a MMOG." **[T4-21, T4-22]**

### 1.2 Industry state of the art

**Minecraft: Folia, regionised multithreading (PaperMC). VERIFIED**
- Sources: [github.com/PaperMC/Folia](https://github.com/PaperMC/Folia) README, last commit 2026-10-29, local `dl/t4-folia/repo/README.md`. Plus the docs source in [github.com/PaperMC/docs](https://github.com/PaperMC/docs): `src/content/docs/folia/admin/reference/region-logic.md` and `overview.md`, local `dl/t4-paperdocs/repo/...`.
- How it splits:
  - "Folia groups nearby loaded chunks to form an "independent region."" "The tick loops are executed on a thread pool in parallel." **[T4-01]**
  - Regions are kept by *proximity*, not by load: "Maintenance of regions is done by merging nearby regions together, marking which regions are eligible to be ticked, and finally by splitting any regions into smaller independent regions." Rule 4 of the overview: "Adjacent regions must eventually merge to form a single region." **[T4-02]**
  - The regionizer works on "region sections" of N×N chunks (N a power of two). It has tuning knobs: "recalculation count", "max dead section percent", "empty section creation radius" and "merge radius".
- Who benefits: "Server types that naturally spread players out, like skyblock or SMP, will benefit the most from Folia." Hardware: "Ideally, at least 16 _cores_ (not threads)." Tested on a server with "~330 players peak". **[T4-03]**
  - GUESS: a crowd in one place forms one region, which is ticked by one thread. Folia does not split by load.
- Scheduling of regions onto threads is "similar to EDF". But "the scheduling algorithm is neither NUMA aware nor CPU core aware". **[T4-04]**
- Cost to developers: "So, have your expectations for compatibility at 0." (for plugins). **[T4-05]**

**Minecraft: MultiPaper, one world across several servers. VERIFIED**
- Source: [github.com/MultiPaper/MultiPaper](https://github.com/MultiPaper/MultiPaper) README, last commit 2025-05-22, local `dl/t4-multipaper/repo/README.md`.
- "MultiPaper is in public beta." It is a "1.20.1 Purpur fork".
- The master "Decides who gets to tick the chunk (first in first served basis)". The built-in proxy "automatically selects the multipaper server with the lowest load".
- "Instead of having 1 server handling 100 players, you can have 10 servers handling 10 players each." **[T4-38]**
- GUESS: the ownership rule (first come, first served) is naive. Each server also keeps cached copies of other servers' chunks in sync, which costs extra CPU. This is a testbed where a smarter ownership policy could be tried, but it adds capacity, not savings.

**Star Citizen: server meshing. SNIPPET (all CIG hosts blocked)**
- Static server meshing went live with the Alpha 4.0 preview in Dec 2024. Patch-note line: "Server Meshing: 5:5:500 Player Shards". Wiki: "server meshing went fully live in December 2024 with the preview of alpha 4.0 with 500 players per shard" ([starcitizen.tools Alpha 4.0.0](https://starcitizen.tools/Update%3AStar_Citizen_Alpha_4.0.0)). **[T4-06]**
- CIG's Server Meshing Q&A ([comm-link 18397](https://api.star-citizen.wiki/comm-links/18397)): "with the static mesh, we will have the first version of a true mesh but, as the name 'static' indicates, the ability to scale this mesh is very limited". **[T4-07]**
- A forum post quoting CIG says that under static meshing the "worst case is if all the players decide to spread themselves out between all the locations assigned to a single server node", so the player cap per shard is set by the worst case ([forums.frontier.co.uk](https://forums.frontier.co.uk/goto/post?id=9600048)). **UNVERIFIED** (secondary, seen in an excerpt). **[T4-10]**
- Letter from the Chairman, 2026-08-27 ([starcitizen.tools mirror](https://starcitizen.tools/Comm-Link:Letter_from_the_Chairman_-_2026-08-27)):
  - The team calls the current step "Quasi Dynamic Server Meshing (QDSM) because we are not yet subdividing territories in the persistent universe, but it is the first time servers are spun up and down based on load".
  - "Instancing in 4.10 is the first use case of dynamic server meshing in Star Citizen." **[T4-08]**
- Roadmap Roundup, 7 Oct 2026 ([RSI Spectrum](https://robertsspaceindustries.com/spectrum/community/SC/forum/3/thread/roadmap-roundup-october-7-2026)): "DGS Planet Subdivision" splits planets into smaller territories so one dynamic game server does not handle a whole body. It is "Tentative" for Alpha 4.11, with "Nyx 1" first. **[T4-09]**
  - I could not confirm that it had shipped by 2026-11-08.
  - One search summary claimed a 4 Nov 2026 roundup calls it "the first dynamic implementation", "limited to Nyx I for now". A second search found no such edition. Treat it as unconfirmed.
- A Feb 2026 report says the "quasi-dynamic" work of spinning servers up and down "without crashing the host server is ongoing" ([massivelyop.com, 2026-02-06](https://massivelyop.com/2026/02/06/star-citizen-cto-outlines-progress-on-server-meshing-and-crafting-with-alpha-4-7-on-track-for-march/)).

**SpatialOS (Improbable). VERIFIED for the code, SNIPPET for the business history**
- Unreal GDK ([github.com/spatialos/UnrealGDK](https://github.com/spatialos/UnrealGDK); last commit 2020-09-03, "Merging final GDK for Unreal 0.11.0 release"; local `dl/t4-spatialgdk/repo`). `GridBasedLBStrategy.h`: "A load balancing strategy that divides the world into a grid. Divides the load between Rows * Cols number of workers". It has a fixed `InterestBorder`. **[T4-11, T4-14]**
- Unity GDK ([github.com/spatialos/gdk-for-unity](https://github.com/spatialos/gdk-for-unity); last commit 2021-10-07; local `dl/t4-spunity/repo`):
  - Launch configs use static layouts: `"rectangle_grid": {"cols": 2, "rows": 1}` and `"points_of_interest": {"num_workers": 1, "points": [...]}`. **[T4-12]**
  - README: "This is the repository for the last version of the GDK plugin that is compatible with the SpatialOS legacy platform. The GDK is frozen for users who have not yet migrated their project to SpatialOS running on zeuz." It asks users to make contact "before 30th December 2021". **[T4-13]**
- Business history, SNIPPET:
  - Several SpatialOS games shut down. For Lazarus, a report says "Operating the game would be so costly due to fees for software licenses, server costs, and the effort of constant updates" ([mein-mmo.de](https://mein-mmo.de/en/another-mmo-using-the-wonder-technology-spatialos-is-being-shut-down-what-is-going-on,379672)).
  - Improbable sold The Multiplayer Group to Keywords and the Midwinter studio to Behaviour; Nightingale stopped using SpatialOS (search excerpts from [en.wikipedia.org/wiki/Improbable_(company)](https://en.wikipedia.org/wiki/Improbable_(company)) and [wccftech](https://wccftech.com/nightingale-inflexion-games-bought-tencent-spatialos-not-being-used/)).
  - I found nothing on SpatialOS-on-zeuz after 2021. **[T4-15]**

**EVE Online: node assignment and Time Dilation (TiDi). SNIPPET (eveonline.com and forums blocked)**
- **TiDi** slows the game clock on an overloaded node, in fine steps, down to a configured floor of 10%.
  - CCP's 2011 post ([Introducing Time Dilation](https://www.eveonline.com/news/view/introducing-time-dilation-tidi)): "there's no reason we can't run at 98% time if we're just slightly overloaded".
  - EVE University wiki ([wiki.eveuniversity.org/Time_Dilation](https://wiki.eveuniversity.org/Time_Dilation)): the system is designed so it goes "no slower than 10% of normal time".
  - CCP also acknowledged "we won't be able to handle indefinitely-sized fights". **[T4-16]**
- **Node assignment.** Each solar system runs on one SOL node. A CCP dev blog says the cluster has "approximately 250 server nodes", and that startup involves "allocating which solar-systems will run on which node" ([Behind the Scenes of a long EVE Online downtime](https://www.eveonline.com/news/view/behind-the-scenes-of-a-long-eve-online-downtime)). **[T4-18]**
  - A 2021 forum post on a node death: the system "auto-allocates those services based on recent resource use" ([forums.eveonline.com/t/node-death-2021-04-27/308039](https://forums.eveonline.com/t/node-death-2021-04-27/308039)).
- **Manual reinforcement for big fights.** CCP's March 2023 post "Regarding the Fleet Fight in X47L-Q" ([forum](https://forums.eveonline.com/t/regarding-the-fleet-fight-in-x47l-q/398872)):
  - Node reinforcements happen after downtime.
  - "In this instance the automated distribution placed one staging system with X47L-Q by chance and this caused some problems." **[T4-17]**
- **Where the CPU went.** CCP Masterplan's 2010 blog "Fixing Lag: Drakes of Destiny (Part 1)" ([eveonline.com](https://www.eveonline.com/news/view/fixing-lag-drakes-of-destiny-part-1-1)): during missile spam, "less than 10% of the overall time in Destiny is spent in Evolve" (Evolve is the physics step). "Each missile launch adds a ball to space" and each explosion removes one. **[T4-19]**
  - Engadget's summary of Part 2 says caching and refactoring cut the missile-spam load "from 95% to under 60%" ([engadget 2010-12-14](https://www.engadget.com/2010/12/14/eve-devblogs-discuss-developments-in-the-war-on-lag-and-rmt/)). **[T4-20]**
  - This matches the earlier-research note that physics was "5-10% of the load". The primary blog's wording is "less than 10%" of Destiny time in one profiled scenario.

**Other systems, briefly. SNIPPET**
- **Hadean Aether Engine.** EVE "Aether Wars" demo at GDC 2019: "14,274 connected clients" in one source; PC Gamer reports "nearly 4,000 human players" and ">10,000 ships". The load is spread across servers, but no partitioning details are public ([pcgamer](https://www.pcgamer.com/eve-online-gdc-tech-demo-pitted-more-than-10000-ships-against-each-other)). **[T4-34]**
- **Unreal Engine.** Epic's "MultiServer Replication Plugin" is described by Epic as "experimental" ([forums.unrealengine.com KB](https://forums.unrealengine.com/t/knowledge-base-experimental-an-introduction-to-the-multiserver-replication-plugin/2648232)). **[T4-35]**
- **Instancing instead of splitting.**
  - Guild Wars 2's megaserver creates map copies as needed. It places each player in the copy with the highest affinity score: party, guild, language, home world ([ArenaNet 2014](https://www.guildwars2.com/en/news/introducing-the-megaserver-system/)).
  - WoW Classic "layering" runs parallel copies of a realm at launch and merges them later ([wowhead](https://www.wowhead.com/news=294589/blizzard-clears-up-layering-tech-misconceptions-in-classic-wow)). **[T4-36]**
- **Roblox picks a server for each joining player with a hand-weighted linear score. VERIFIED** ([github.com/Roblox/creator-docs](https://github.com/Roblox/creator-docs) `content/en-us/matchmaking/scoring.md`; local `dl/t4-roblox/repo`). **[T4-37]**
  - "Servers are scored using a weighted sum formula." "The joining player is matched to the server with the highest server score."
  - Default weights: Friends 15, Latency 3, Text Chat 3, Occupancy 2, Play History 2, Language 2, Age 1, Voice Chat 1, Device Type 0.
  - Occupancy signal: "occupancySignalScore = # players in server / serverCapacity". It favours fuller servers, which is a packing incentive.
  - This is exactly the "priority function" shape that FunSearch and AlphaEvolve evolve. But its true objective (player retention) can only be measured by live A/B tests, not offline (GUESS).
- Cross-reference to T2 (SNIPPET there): Albion Online uses about 1 km² clusters, each on one thread. New World uses a static set of EC2 hosts per world. OSRS runs one process per world, up to 2,000 players, with a 0.6 s tick.

### 1.3 Academic work, 2002 to 2026
The metadata below is VERIFIED via the Crossref or Semantic Scholar APIs. Most full texts were behind blocked hosts, so results are SNIPPET unless marked.

| Year | Paper | Idea | Result |
|---|---|---|---|
| 2002 | Lui & Chan, "An efficient partitioning algorithm for distributed virtual environment systems", IEEE TPDS, doi:10.1109/71.993202 (234 citations, S2) | partitioning as an optimisation over load and communication | n/a (abstract not available) |
| 2003 | Lee & Lee, VRST, doi:10.1145/1008653.1008681, "...highly-skewed user distribution" | dynamic load distribution under skewed crowds | n/a |
| 2003 | Duong & Zhou, ICON, doi:10.1109/icon.2003.1266179 | dynamic load sharing | n/a |
| 2005 | Chen, Wu, DeLap, Knutsson, Lu, Amza, "Locality aware dynamic load management for massively multiplayer games", PPoPP, doi:10.1145/1065944.1065982 | shed load to *neighbouring* servers only | SNIPPET: cuts mean response time "by as much as a factor of 6" against a global algorithm and "a factor of 8" against static partitioning, in "trace-driven simulations of massive player flocking" ([pdf](https://www.eecg.utoronto.ca/~amza/papers/f87-chen.pdf)) **[T4-25]** |
| 2005 | Balan et al., "Matrix: Adaptive Middleware for Distributed Multiplayer Games", Middleware, doi:10.1007/11587552_20 | adaptive region splitting middleware | n/a |
| 2005 | De Vleeschauwer et al., "Dynamic microcell assignment for MMOG", NetGames, doi:10.1145/1103599.1103611 | small fixed "microcells" reassigned to servers; ILP plus heuristics | SNIPPET: a later journal version reports that a constrained ILP gives the best assignments, and a random player-following heuristic is faster **[T4-29]** |
| 2006 | Lu, Parkin, Morgan, "Load balancing for MMOG", NetGames, doi:10.1145/1230040.1230064 | | n/a |
| 2008/2011 | Nae, Iosup, Prodan et al. (SC08; IEEE TPDS 22(3), doi:10.1109/TPDS.2010.82) | renting data-centre capacity on demand, with a neural-network load predictor | SNIPPET: operators "over-provision a large static infrastructure capable of sustaining the game peak load"; dynamic provisioning is "much more efficient than its static alternative" **[T4-28]** |
| 2009 | Bezerra & Geyer, Multimedia Tools & Applications, doi:10.1007/s11042-009-0302-z | region growing (ProGReGA), later kd-trees | SNIPPET: the kd-tree method shifts split coordinates when a server is overloaded |
| 2009 | Lim & Lee, PRESENCE, doi:10.1162/pres.18.1.16 | task-based load distribution | n/a |
| 2010 | Deng & Lau, VRST, doi:10.1145/1889863.1889910 | heat-diffusion load balancing | n/a |
| 2018 | Farlow & Trahan, FDG, doi:10.1145/3235765.3235769 | periodic load balancing heuristics | VERIFIED (abstract via S2): "we propose a novel periodic approach to this problem of load balancing along with a collection of heuristics ... finding that these new heuristics provide more system balance than those existing methods." **[T4-27]** |
| 2020 | Bamutange et al., "Zoneless Load Balancing for MMOG", ELECOM, doi:10.1109/elecom49001.2020.9296989 | | n/a |

Data on how players really move. VERIFIED: Liang et al., "Avatar Mobility in Networked Virtual Environments", arXiv [0807.2328](https://arxiv.org/abs/0807.2328) (2008); local `dl/t4-arx-0807.2328/`. **[T4-23, T4-24]**
- "We collected mobility traces of 84,208 avatars spanning 22 regions over two months in Second Life".
- "the spatial distribution of avatars is far from uniform ... This observation supports the need for more sophisticated partitioning scheme, beyond grid-based partitioning."
- On earlier work: "previous research mainly based their evaluations either on simulations with a simple model of avatar behavior (such as random way-point mobility model), or on collected traces from small-scale games and NVEs."

Other trace sets (VERIFIED metadata via Crossref):
- "World of warcraft avatar history dataset", MMSys 2011, doi:10.1145/1943552.1943569. It records zone-level presence, not positions.
- "Avatar movement in World of Warcraft battlegrounds", NetGames 2009, doi:10.1109/netgames.2009.5446226.

### 1.4 Machine-learning and reinforcement-learning attempts
- **Kim & Kim 2020**, "A load balancing scheme for gaming server applying reinforcement learning in IoT", *Computer Science and Information Systems*, doi:10.2298/csis190917026k. VERIFIED (abstract via Crossref): "we proposed an agent that applies a deep reinforced learning method to distribute loads for gaming servers ... we have compared performances of our proposed scheme and a previus research, ProGreGA, widely used scheme through simulation." The search excerpt adds a "15*15 grid world" with "225 cells" (SNIPPET). **[T4-30]**
- **PPO load balancing.** A JIPS paper is described as combining "the dynamic allocation programming method, a type of greedy algorithm, and proximal policy optimization", using Unity ML-Agents and eight servers (SNIPPET, [jips-k.org](https://jips-k.org/full-text/view?doi=10.3745%2FJIPS.03.0158)). A related paper is "A Study on Load Distribution of Gaming Server Using Proximal Policy Optimization", J. Korea Game Society 2019, doi:10.7583/jkgs.2019.19.3.5 (VERIFIED metadata). **[T4-31]**
- **AFLL (Kang & Kim, Sogang University)**, arXiv [2601.10998](https://arxiv.org/abs/2601.10998), Jan 2026. VERIFIED; local `dl/t4-arx-2601.10998/`. **[T4-32]**
  - Headline: "Experimental results with 1,000 concurrent users show that AFLL achieves 48.3% reduction in CPU time (6,143ms → 3,177ms) ... compared to learning disabled."
  - How: it "dynamically adjusts transmission rates using gradient descent". In other words, it blocks or throttles message types.
  - Setup: "Simulation clients were implemented with a probability model to reproduce actual player behavior". "Experiments were conducted on Intel Core i7-12700K (12 cores), 32GB RAM, Windows 11 Pro environment".
  - GUESS: the baseline is the same prototype with learning switched off. It is not a tuned interest-management scheme, so the 48% mostly measures how much sending less saves.
- **Why Google chose evolved code over deep RL for a placement problem.** AlphaEvolve paper (arXiv [2506.13131](https://arxiv.org/abs/2506.13131), §3.3.1; local copy from T1 at `dl/t1-alphaevolve/ae.txt`), VERIFIED: "AlphaEvolve was chosen over a deep reinforcement learning approach because its code solution not only leads to better performance, but also offers clear advantages in interpretability, debuggability, predictability, and ease of deployment". **[T4-33]**
- **LLM program search (FunSearch, AlphaEvolve, OpenEvolve) on world partitioning, game-session placement or MMO load balancing:** none found. I searched arXiv and the web (GUESS, absence of evidence; consistent with T1 §7). The closest is the vendor quote from Kuro Games in T1. **[T4-71]**

### 1.5 Open problems (my synthesis, GUESS unless cited)
1. **What "load" means.** There is no agreed load metric (Carlini, VERIFIED). CPU per tick depends on interactions: AFLL argues load grows with n·(n−1) state transmissions in a crowd (VERIFIED, AFLL §2.1).
2. **Boundaries.** Folia forbids cross-region access. SpatialOS used a fixed interest border. Star Citizen took years to reach dynamic splitting. Both the cost of a boundary and the cost of a migration depend on the engine.
3. **Predicting hot spots.** EVE still reinforces nodes by hand after downtime. Its automatic distribution once put a staging system on the battle's node "by chance".
4. **Static worst-case provisioning wastes capacity** (Star Citizen, UNVERIFIED). Dynamic schemes add overhead. Whether they save *money*, rather than raising capacity per world, is not shown publicly anywhere I could reach.
5. **Graceful degradation.** TiDi trades time for CPU. Nobody has published an optimal policy for when to slow time versus when to split.
6. **Evaluation.** Most papers use synthetic movement. Real traces exist (Second Life, WoW), but they carry no per-entity CPU cost.

---

## 2. Packing game sessions or zones onto machines (bin packing over time, autoscaling)

### 2.1 The problem in one paragraph
Matches or zones (items) arrive and later leave. Each uses CPU, RAM and bandwidth (a vector). Each must go on a server (bin) that has room, usually without moving it later. You pay for each server for as long as it is on, rounded to the billing unit. You also want spare capacity so players do not wait while a new server boots.

Theory calls this **MinUsageTime Dynamic (Vector) Bin Packing**. Murhekar, Arbour, Mai, Rao (Adobe), arXiv [2304.08648](https://arxiv.org/abs/2304.08648), VERIFIED: "Several cloud-based applications, such as cloud gaming, rent servers to execute jobs which arrive in an online fashion." Their bounds on how far an algorithm can be from optimal grow with µ, "the ratio of the max/min item durations". For example: "We prove that the CR of Move To Front is at most (2µ + 1)d + 1". **[T4-51]**

### 2.2 What production systems do
**Agones (Google, open source). VERIFIED** ([github.com/googleforgames/agones](https://github.com/googleforgames/agones) docs; local `dl/t4-agones/repo/site/content/en/docs/`; latest tag v1.62.0, commit "Release-1.62.0 (#4816)" dated 2026-11-04)
- The default "Packed" strategy: "It attempts to _pack_ as much as possible into the smallest set of nodes, to make scaling infrastructure down as easy as possible." **[T4-40]**
- Its allocation is a fixed rule: "allocation will prioritise allocating `GameServers` to nodes that are running on Nodes that already have allocated `GameServers` running on them."
- On scale-down it removes Ready servers "from Nodes with the _least_ number of `Ready` and `Allocated` `GameServers` on them".
- Admitted headroom: "The default Kubernetes scheduler doesn't do a perfect job of packing, but it's a good enough job for what we need - at least at this stage." **[T4-41]**
- Autoscaling: "Fleet autoscaling is the only type of autoscaling that exists in Agones." **[T4-42]**
  - Policies: Buffer, Counter, List, Webhook, Schedule/Chain, and an alpha Wasm policy (quickstart dated 2025-12-09): "the logic computing the number of target replicas is executed by a WebAssembly module that you provide."
  - GUESS: Webhook and Wasm give a ready plug-in point for an evolved policy.
- Several matches per process: "sometimes it can be a more economical use of resources to run multiple concurrent game sessions from within a single `GameServer` instance." **[T4-43]**

**AWS GameLift (now "Amazon GameLift Servers"). VERIFIED from the developer guide archived on GitHub** (repo [awsdocs/amazon-gamelift-developer-guide](https://github.com/awsdocs/amazon-gamelift-developer-guide). Content was deleted at archive time, so I read commit 7e08295 of 2023-06-15; local `dl/t4-gamelift/repo`. docs.aws.amazon.com was blocked, so the current docs were not checked.)
- Target-based autoscaling "adjusts capacity levels based on the fleet metric `PercentAvailableGameSessions`".
- The trade-off, in AWS's words: "With a large buffer, you minimize wait time, but you also pay for extra resources that you may not use." And "It can take minutes to start up new instances and server processes." **[T4-44]**
- FleetIQ queue placement is a fixed priority order: latency, then cost, then destination, then location.
- "Spot Instance fleets are considered not viable when the interruption rate for a particular instance type is too high."
- "Spot Instances can be interrupted with two minutes' notification when AWS needs the capacity back." **[T4-45]**
- Marketing number: AWS's 2018 launch said savings "of up to 90%" with FleetIQ plus Spot ([aws.amazon.com news](https://aws.amazon.com/about-aws/whats-new/2018/02/amazon-gamelift-fleetiq-and-spot-instances-reduce-costs-by-up-to-90-percent)). SNIPPET; a vendor claim. **[T4-46]**

**Market changes in 2026. SNIPPET**
- **Hathora:** "announced on March 4, 2026 that it would end its game server hosting business, and the service shut down on May 5, 2026". It was acquired by Fireworks AI. Stormgate lost its online modes as a result ([techspot](https://www.techspot.com/news/111969-stormgate-servers-go-dark-following-ai-focused-hosting.html); [supercraft](https://crux.supercraft.host/blog/hathora-shut-down-where-to-go-after-may-2026/)). **[T4-47]**
- **Unity Multiplay Game Server Hosting:** deprecated as of April 1, 2026, and licensed to Rocket Science Group ([status.unity.com notice 362941](https://status.unity.com/info_notices/362941)). **[T4-48]**
- **Vendor savings claims, with no public method:**
  - Edgegap: "reduce your total cost of operations by up to 70%" ([edgegap.com](https://edgegap.com/)).
  - AccelByte: hybrid bare-metal plus cloud gives "up to 40% savings on compute" ([accelbyte.io](https://accelbyte.io/blog/hybrid-approach-to-multiplayer-hosting)).
  - I found **no independent measurement** of game-server fleet utilisation. **[T4-49]**

### 2.3 Research results that bound the headroom
- **Lee & Tang (NTU), "Evaluation of Dynamic Vector Bin Packing for Virtual Machine Placement"**, arXiv [2602.14704](https://arxiv.org/abs/2602.14704), Feb 2026. Local `dl/t4-arx-2602.14704/` (pages rendered as `p-05.png` and `p-08.png`). **[T4-52, T4-53]**
  - Setup, VERIFIED: real Microsoft Azure traces (4,551,058 VM requests). There are "five resource dimensions (d = 5): core, memory, HDD, SSD and NIC"; HDD is dropped where it plays no role, leaving 4. The paper evaluates "each of the 28 instances". The score is total server usage time divided by a lower bound.
  - Without knowing when items leave, VERIFIED: "The First Fit algorithm produces the lowest mean performance ratio among all the algorithms tested". Best Fit "performs close to First Fit".
  - Fig. 3, VERIFIED_FIG: mean ratios are about **1.13** (First Fit), 1.15 (Best Fit), 1.3 (Round Robin Next Fit), 1.34 (MRU) and about 2.4 (Next Fit).
  - When departure times are known, VERIFIED: "Prioritized NRT performs very well and better than all the other algorithms". Fig. 8, VERIFIED_FIG: mean ratio about **1.03**.
  - Robustness, VERIFIED: "categorizing items by their departure times outperforms that by their durations in the clairvoyant setting, but the former is less robust against prediction errors".
  - Synthetic versus real data, VERIFIED: "earlier studies [16], [17] showed that MRU is empirically the best in their evaluations with synthetic data, but this is not the case in our evaluation with real-world data". This is a warning against tuning on synthetic traces.
  - GUESS arithmetic: the lower bound is at most the optimum, so any algorithm can beat First Fit by at most 1 − 1/1.13 ≈ **11.5%** of server-time on this data. It can beat departure-aware Prioritized NRT by at most 1 − 1/1.03 ≈ **2.9%**.
  - GUESS: game matches have a much narrower duration range than VMs, which have log-normal lifetimes. The bounds scale with µ, so First Fit is probably even closer to optimal for matches.
- **Google LAVA** (Ling et al.), arXiv [2412.09840](https://arxiv.org/abs/2412.09840) v2, 2025. VERIFIED; local `dl/t4-arx-2412.09840/`. In production: "decreasing stranded compute and memory resources by ∼3% and ∼2% respectively. It increases empty hosts by 2.3-9.2 pp in production". It uses ML lifetime distributions with re-prediction. **[T4-54]**
- **AlphaEvolve on Borg**, VERIFIED (arXiv 2506.13131 §3.3.1):
  - It is "a vector bin-packing problem with two variables". The heuristic only re-ranks machines Borg already judged feasible, so decisions are "effectively correct by construction".
  - It "continuously recovers on average 0.7% of Google's fleet-wide compute resources". **[T4-56]**
- **HELIOS** (Ding & Wang), arXiv [2609.09164](https://arxiv.org/abs/2609.09164), Sep 2026. LLM-evolved multi-cloud orchestration policies. VERIFIED; local `dl/t4-arx-2609.09164/`. **[T4-59]**
  - Abstract: "the evolved policy reduces penalized operating cost by 45% versus a single-cloud best-fit baseline and by 9-19% versus calibrated multi-cloud heuristics."
  - Caveat 1, from the paper itself: most of the 45% is not from evolution. "Spot capacity is the largest single factor (−spot: +140% cost)". "Evolution contributes +14% cost / +24% J when removed."
  - Caveat 2: against the calibrated greedy heuristic, "the win is SLO". In Table II (PlanetLab held-out), raw cost is **$723 ± 26 for HELIOS against $665 ± 49 for Greedy-MultiCloud**, so HELIOS costs more in dollars.
  - Caveat 3: the search was tiny: "5 generations and 19 candidate programs (38 simulated episodes)".
  - Caveat 4: guardrails mattered. Without them the policy family degraded "to 97-98% premium-tier downtime under a 10x spot-hazard stress test".
- **Theory for equal-length jobs** (close to fixed-length matches): Masoori, Narayanan, Pankratov, "Renting Servers in the Cloud: The Case of Equal Duration Jobs", arXiv [2108.12486](https://arxiv.org/abs/2108.12486) v4 (2024). VERIFIED: "We prove a tight bound of 2 on the competitive ratio of NextFit." **[T4-55]**

### 2.4 How directly FunSearch applies (T1 covers FunSearch in depth)
- What FunSearch's evaluator does, VERIFIED from the code in [google-deepmind/funsearch](https://github.com/google-deepmind/funsearch) `bin_packing/bin_packing.ipynb` (T1's clone at `dl/t1-funsearch/repo`). It is **online 1-D bin packing where items never leave**:
  - `for item in items: ... priorities = priority(item, bins[valid_bin_indices]) ... bins[best_bin] -= item`.
  - The score is the number of bins used. **[T4-57]**
- What carries over: the *shape*. A `priority(item, feasible_bins)` function maps one-to-one onto "which existing server gets this match", as in Agones Packed, Roblox scoring and the Borg trick.
- What does not carry over: departures, vectors (CPU, RAM, bandwidth), billing in server-hours, boot delays, player wait time and regions. The FunSearch evaluator would have to be rewritten, for example as the Lee & Tang setup plus boot delay and a wait-time constraint.
- Generalisation risk, VERIFIED from T1's copy of Sim, Renau, Hart, arXiv [2501.11411](https://arxiv.org/abs/2501.11411):
  - "The hand-designed heuristic BF ranks 1st".
  - LLM heuristics "do not generalise to distributions that differ from their training examples". **[T4-58]**

### 2.5 What this means in euros for Miles (Hetzner)
- **Billing, VERIFIED** ([docs.hetzner.com/cloud/billing/faq](https://docs.hetzner.com/cloud/billing/faq/); raw HTML `dl/t4-hetzner/faq.html`): "Servers have both a monthly price cap and a price per hour." "We always round up the hourly usage of a server." "We will bill you for each cloud server until you choose to delete them." **[T4-60]**
- **Price and availability, VERIFIED on 2026-11-08** ([hetzner.com/cloud/general-purpose](https://www.hetzner.com/cloud/general-purpose/); raw HTML `dl/t4-hetzner/gp.html`):
  - CCX13 (2 dedicated vCPU, 8 GB) costs "€ 0.0697 per hour" with a cap of "€ 43.49 per month" (excl. VAT).
  - All six CCX plans show "This product is currently unavailable. Please check back later." **[T4-61]**
- **GUESS arithmetic** **[T4-62]**:
  - The cap is reached after 43.49 / 0.0697 ≈ 624 h, about 26 days.
  - A server used only for a 6-hour evening peak runs 6 × 30 = 180 h, costing 180 × 0.0697 ≈ **€12.55 against €43.49** if kept on. That saves about €31, or 71%, *per peak server per month*.
  - That saving needs (a) a real peak/off-peak gap, (b) more than one server, and (c) cloud capacity actually being available.
  - Dedicated (AX) servers are monthly, with setup fees (see T2 §1.1), so packing or autoscaling them saves nothing until a whole box can be cancelled.
- **GUESS:** for Sumo Brothers on one box, the € value of better packing is zero. Even with about 5 hourly servers, a tuned "buffer plus First Fit" rule should capture most of the ~71% peak saving. An evolved heuristic would add at most a few % of that.

---

## 3. Simulation hot spots that might suit AI search (brief, lower priority)

- **NPC level of detail and time-slicing.** VERIFIED, Spigot `entity-activation-range` (docs source in [PaperMC/docs](https://github.com/PaperMC/docs) `src/config/paper/spigot.yml`; local `dl/t4-paperdocs/repo/src/config/paper/spigot.yml`). **[T4-63]**
  - "The distance in blocks from a player at which entities tick normally. Outside this range, entities tick less frequently."
  - Defaults are round, hand-picked numbers: animals 32, monsters 32, raiders 64, water mobs 16, villagers 32. There are wake-up timers such as "How long to wake an inactive animal up for, in ticks." (100).
  - This is about 25 knobs. GUESS: Bayesian optimisation of the knobs probably gets most of the gain. Program search only helps if the *rule* itself changes, for example by using predicted player paths.
  - EVE's TiDi (§1.2) is time-slicing for the whole node under overload.
- **Spatial indexing and collision broad-phase.** VERIFIED, Rapier's changelog ([raw.githubusercontent.com/dimforge/rapier/master/CHANGELOG.md](https://raw.githubusercontent.com/dimforge/rapier/master/CHANGELOG.md); local `dl/t4-rapier/CHANGELOG.md`). Humans still find big structural wins here. **[T4-64]**
  - v0.27.0-beta.0 (11 July 2025): "Replace the hierarchical SAP broad-phase by a broad-phase based on parry’s new BVH structure."
  - v0.35.0-beta.0 (2 Aug 2026): "The broad phase was reworked for large, mostly-static worlds: scenes with very large static collider counts now pay near-zero per-step broad-phase cost."
  - But physics was under 10% of the EVE Destiny tick in the profiled fight (SNIPPET, §1.2).
- **Crowd pathfinding.** The best evidence that LLM-written heuristics can beat hand-made ones in search problems is from classical planning, not games. Corrêa, Pereira, Seipp, arXiv [2503.18809](https://arxiv.org/abs/2503.18809) (NeurIPS 2025), VERIFIED: "The resulting LLM-generated heuristics solve many more unseen test tasks than state-of-the-art domain-independent heuristics for classical planning." **[T4-65]**
  - GUESS: game grids and navmeshes already have strong *domain-specific* heuristics (octile distance, JPS, landmarks, flow fields), so the headroom is smaller. I did not source this.
- **Saving to the database (persistence).** VERIFIED: Vaz Salles et al., "An Evaluation of Checkpoint Recovery for Massively Multiplayer Online Games", PVLDB 2009, doi:10.14778/1687627.1687769 (abstract via Semantic Scholar). **[T4-66]**
  - "This architecture limits scalability, forcing MMO developers to either invest in high-end hardware or to over-partition their virtual worlds."
  - "we recommend MMO developers to adopt a copy-on-update scheme with a double-backup disk organization to checkpoint game state."
  - T2 found a SNIPPET that New World's DynamoDB "handles about 800,000 writes every 30 seconds". At that scale, write cost can rival compute.
- **The EVE lesson.** CCP profiled first, found that missile bookkeeping, not physics, was the cost, and fixed it by hand: "from 95% to under 60%" (SNIPPET, §1.2). For an indie server, a profiler plus the conventional work from earlier research (ECS layout, PGO) comes before any search.

---

## 4. Evidence against (stated plainly)
1. **Irrelevant to the current game.** Sumo Brothers has small, separate matches and one box. Dynamic world partitioning does not apply. Packing has nothing to pack (GUESS, from the context given).
2. **Small proven gains in packing.** These are 0.7% (Borg), ~2–3% (LAVA, and FunSearch on its training distribution), at most ~11.5% over First Fit even with perfect knowledge (Azure data, GUESS bound), and 9.6% on a self-defined penalised cost that is *higher* in raw dollars (HELIOS). All of these are at fleet scale.
3. **Weak or synthetic evaluations in partitioning research.**
   - Movement models are synthetic (Liang et al.).
   - There is no agreed load metric (Carlini).
   - RL work is in small simulations against older heuristics (Kim & Kim 2020; JIPS).
   - AFLL's 48% compares against "learning disabled".
4. **Industry avoids clever dynamic partitioning.** Static grids (SpatialOS), proximity-only regions (Folia), static meshing (Star Citizen 2024–2026), manual node reinforcement plus TiDi (EVE), and instancing (GW2, WoW, Roblox) are the norm. Dynamic schemes take years and restrict gameplay or plugins.
5. **Synthetic-to-real gap in packing too.** "MRU is empirically the best ... with synthetic data, but this is not the case ... with real-world data" (Lee & Tang). LLM-evolved bin-packing heuristics overfit (Sim et al.; Herrmann & Pallez in T1).
6. **Thin hosting market and capacity limits.** Two hosting platforms closed in 2026 (Hathora, Multiplay). Hetzner's dedicated-vCPU cloud plans show "currently unavailable", so autoscaling may not get capacity when it asks.

---

## 5. Verdict for this problem family
- **Feasible as an experiment:** yes, for **session placement plus autoscaling** (candidate C2).
  - The evaluator is a cheap, deterministic replay of match logs.
  - The search space is a small scoring function with feasibility enforced outside it, the Borg pattern.
  - Strong, simple baselines exist.
  - **Expected gain is single-digit percent of server-hours**, and it is worth money only with several hourly-billed servers. A grid-searched two-parameter rule may match it.
- **Most "MMO-relevant" but evaluator-limited:** dynamic world partitioning (C1).
  - The real obstacles are a believable CPU and interaction cost model and realistic crowd traces, not the search.
  - A credible plan would first build an open benchmark: real movement traces plus a CPU model calibrated on a real server, for example Folia or MultiPaper with Yardstick bots. Only then evolve split, merge and assign policies against the strongest classic baselines.
  - Nobody appears to have done LLM program search here, but decades of hand-designed heuristics and some RL show the problem itself is not untried.
- **Hot spots** (C3–C6) are ordinary code optimisation and parameter tuning. Profile first.

---

## 6. Candidates (also in the structured result)
See the structured output. In short:
- C1: dynamic partitioning policy, scored on held-out traces with a calibrated CPU model.
- C2: session placement plus autoscaling, scored in € under Hetzner billing with a wait-time constraint.
- C3: NPC LOD and time-slicing at fixed fidelity.
- C4: spatial index and broad-phase, checked for exact set equality.
- C5: pathfinding heuristics.
- C6: persistence write policy at a fixed data-loss window.

Yardstick ([github.com/atlarge-research/yardstick](https://github.com/atlarge-research/yardstick); last commit 2026-09-30; local `dl/t4-yardstick/repo`) is a usable harness for C1 and C3 on Minecraft-like servers. VERIFIED: "Yardstick is a benchmark for Modifiable Virtual Environments (MVEs) such as Minecraft. It deploys a game server, runs a workload of emulated players against it, collects performance metrics". It warns when "workload nodes were resource constrained". **[T4-39]**

---

## 7. Open questions
- Did Star Citizen's "DGS Planet Subdivision" ship in Alpha 4.11 or later, and with what player and server counts?
- Is physics still under 10% of an EVE fleet-fight tick in 2026, and what dominates now?
- Is there any public trace with per-entity positions *and* per-tick server CPU from a real MMO?
- Sumo Brothers numbers: match length distribution, CPU per match, peak-to-off-peak ratio, number of servers. These decide whether C2 is worth anything.
- Will Hetzner Cloud supply burst capacity in 2026, given the "currently unavailable" plans?
- The exact results of Nae et al. 2011, De Vleeschauwer 2005, Bezerra & Geyer 2009 and Lui & Chan 2002 (full texts were blocked).
- Does SpatialOS (on zeuz) still operate in 2026?

---

## 8. Blocked or limited hosts (claims from them are SNIPPET or UNVERIFIED)
- Did not resolve (proxy policy): robertsspaceindustries.com, starcitizen.tools, api.star-citizen.wiki, www.starshipdealers.com, www.eveonline.com, forums.eveonline.com, wiki.eveuniversity.org, www.engadget.com, www.pcgamer.com, www.eecg.utoronto.ca, www.inf.usi.ch, jips-k.org, elib.mi.sanu.ac.rs, edgegap.com, docs.edgegap.com, gameye.com, accelbyte.io, crux.supercraft.host, www.techspot.com, mein-mmo.de, theimpound.com, docs.unity.com, status.unity.com, aws.amazon.com, dev.epicgames.com, forums.unrealengine.com, www.guildwars2.com, wiki.guildwars2.com.
- Throttled: api.openalex.org (daily budget exhausted), api.semanticscholar.org and api.crossref.org (intermittent HTTP 429; many abstracts elided by publishers).

## 9. Files downloaded (all under `/tmp/claude-0/-home-user-OogaPL/32372848-6606-541b-85af-54d683c4e916/scratchpad/dl/`)
- `t4-folia/repo` (PaperMC/Folia)
- `t4-paperdocs/repo` (PaperMC/docs, sparse)
- `t4-multipaper/repo`
- `t4-agones/repo` (sparse docs; tag v1.62.0 fetched)
- `t4-gamelift/repo` (archived GameLift guide, commit 7e08295)
- `t4-spatialgdk/repo`
- `t4-spunity/repo`
- `t4-roblox/repo` (creator-docs, sparse)
- `t4-yardstick/repo`
- `t4-rapier/CHANGELOG.md`
- `t4-hetzner/{gp.html,faq.html}`
- arXiv PDFs and text: `t4-arx-2601.10998`, `t4-arx-2304.08648`, `t4-arx-2602.14704` (plus rendered figure pages), `t4-arx-2412.09840`, `t4-arx-2108.12486`, `t4-arx-2609.09164`, `t4-arx-0807.2328`, `t4-arx-1510.08940`, `t4-arx-2503.18809`
- API responses: `t4-s2/*.json` (Semantic Scholar), `t4-crossref/*.json`, `t4-arxivsearch/*.html`
- Reused read-only from T1 downloads: `t1-alphaevolve/ae.txt`, `t1-funsearch/repo`, `t1-binpack-critique/*.txt`

---

## 10. Claim index (ids match the structured result)

| id | status | claim (short) | source |
|---|---|---|---|
| T4-01 | VERIFIED | Folia: nearby chunks form independent regions ticked in parallel on a thread pool | github.com/PaperMC/Folia README |
| T4-02 | VERIFIED | Folia regions merge and split by proximity, not by load | PaperMC/docs region-logic.md, overview.md |
| T4-03 | VERIFIED | Folia helps only when players spread out; ≥16 cores; tested at ~330 players | Folia README |
| T4-04 | VERIFIED | Folia's region scheduler is EDF-like and not NUMA- or core-aware | PaperMC/docs overview.md |
| T4-05 | VERIFIED | Folia plugin compatibility: "expectations ... at 0" | Folia README |
| T4-06 | SNIPPET | Star Citizen static meshing live since Dec 2024, 500 players per shard | starcitizen.tools Alpha 4.0.0 |
| T4-07 | SNIPPET | CIG: static mesh's "ability to scale ... is very limited" | api.star-citizen.wiki comm-link 18397 |
| T4-08 | SNIPPET | Aug 2026: QDSM spins servers up and down by load; territories not yet subdivided; 4.10 instancing is the first use | Letter from the Chairman 2026-08-27 |
| T4-09 | SNIPPET | DGS Planet Subdivision "Tentative" for 4.11 with Nyx I first; not confirmed shipped by 2026-11-08 | Roadmap Roundup 2026-10-07 |
| T4-10 | UNVERIFIED | Static meshing caps players per shard by the worst case | forum post quoting CIG |
| T4-11 | VERIFIED | SpatialOS Unreal GDK load balancing is a static Rows×Cols grid with an interest border | spatialos/UnrealGDK GridBasedLBStrategy.h |
| T4-12 | VERIFIED | SpatialOS launch configs use static rectangle_grid and points_of_interest layouts | spatialos/gdk-for-unity *_launch.json |
| T4-13 | VERIFIED | SpatialOS GDK for Unity frozen for the legacy platform; migration to zeuz by Dec 2021 | gdk-for-unity README |
| T4-14 | VERIFIED | Unreal GDK's last release was 0.11.0 (repo's last commit 2020-09-03) | UnrealGDK git log |
| T4-15 | SNIPPET | SpatialOS games closed; Lazarus cited licence and server costs; Improbable sold game units | mein-mmo.de; Wikipedia |
| T4-16 | SNIPPET | EVE TiDi: fine-grained slow-down to a 10% floor | eveonline.com TiDi post; EVE Uni wiki |
| T4-17 | SNIPPET | EVE: reinforcement after downtime; auto-distribution put a staging system on the battle's node "by chance" | forums.eveonline.com X47L-Q |
| T4-18 | SNIPPET | EVE: ~250 nodes; systems assigned to nodes at startup; reallocation by recent resource use | eveonline.com downtime blog; forum |
| T4-19 | SNIPPET | EVE: physics (Evolve) under 10% of Destiny time in missile spam | eveonline.com "Drakes of Destiny" |
| T4-20 | SNIPPET | EVE: missile fix cut load from 95% to under 60% | engadget 2010 |
| T4-21 | VERIFIED | No agreed way to measure MMOG load | arXiv 1510.08940 |
| T4-22 | VERIFIED | Hotspots are the main cause of load imbalance | arXiv 1510.08940 |
| T4-23 | VERIFIED | Second Life traces (84,208 avatars): non-uniform crowds, so beyond-grid partitioning is needed | arXiv 0807.2328 |
| T4-24 | VERIFIED | Earlier evaluations used simple mobility models | arXiv 0807.2328 |
| T4-25 | SNIPPET | Chen et al. 2005: up to 6× (vs global) and 8× (vs static) lower response time in simulation | eecg.utoronto.ca PDF |
| T4-26 | VERIFIED | 2002–2020 academic MMO load-balancing literature (metadata) | Crossref / Semantic Scholar |
| T4-27 | VERIFIED | Farlow & Trahan 2018: periodic heuristics give more balance than prior methods | S2 abstract, doi:10.1145/3235765.3235769 |
| T4-28 | SNIPPET | Nae et al.: static over-provisioning for peak; dynamic provisioning more efficient | ICPE/SC08 abstracts |
| T4-29 | SNIPPET | De Vleeschauwer: microcells, ILP and a player-following heuristic | imec repository |
| T4-30 | VERIFIED | Kim & Kim 2020: deep-RL load balancer compared with ProGReGA in simulation | Crossref abstract, doi:10.2298/csis190917026k |
| T4-31 | SNIPPET | PPO load balancing (Unity ML-Agents, 8 servers) | jips-k.org |
| T4-32 | VERIFIED | AFLL: −48.3% CPU vs learning off, by throttling messages; simulated clients | arXiv 2601.10998 |
| T4-33 | VERIFIED | Google chose evolved code over deep RL for Borg placement | arXiv 2506.13131 |
| T4-34 | SNIPPET | Hadean/EVE Aether Wars player counts conflict; no partitioning details public | pcgamer |
| T4-35 | SNIPPET | Unreal MultiServer Replication plugin is experimental | forums.unrealengine.com |
| T4-36 | SNIPPET | GW2 megaserver and WoW layering handle crowding by instancing | guildwars2.com; wowhead |
| T4-37 | VERIFIED | Roblox server placement is a weighted sum (Friends 15 ... Occupancy 2 ...) | Roblox/creator-docs scoring.md |
| T4-38 | VERIFIED | MultiPaper: first-come chunk ownership, lowest-load proxy, public beta | MultiPaper README |
| T4-39 | VERIFIED | Yardstick: maintained MVE benchmark with emulated players | atlarge-research/yardstick |
| T4-40 | VERIFIED | Agones Packed packs into the fewest nodes (default) | agones docs |
| T4-41 | VERIFIED | Agones: the default scheduler "doesn't do a perfect job of packing" | agones docs |
| T4-42 | VERIFIED | Agones autoscalers: Buffer, Counter/List, Webhook, Wasm (alpha), Schedule/Chain; v1.62.0 on 2026-11-04 | agones docs, git tag |
| T4-43 | VERIFIED | Agones: several sessions per process can be "more economical" | agones docs |
| T4-44 | VERIFIED | GameLift target tracking on PercentAvailableGameSessions; buffer-versus-wait trade-off | awsdocs GameLift guide (2023) |
| T4-45 | VERIFIED | FleetIQ: fixed priority order; avoids Spot types with high interruption; 2-minute notice | awsdocs GameLift guide (2023) |
| T4-46 | SNIPPET | AWS 2018: "up to 90%" savings with FleetIQ + Spot | aws.amazon.com |
| T4-47 | SNIPPET | Hathora ended game hosting (announced 2026-03-04, off 2026-05-05) | techspot; supercraft |
| T4-48 | SNIPPET | Unity Multiplay hosting deprecated 2026-04-01 | status.unity.com |
| T4-49 | SNIPPET | Vendor savings claims (Edgegap 70%, AccelByte 40%) with no method | edgegap.com; accelbyte.io |
| T4-50 | GUESS | No independent measurement of game-server fleet utilisation found | n/a |
| T4-51 | VERIFIED | MinUsageTime DVBP models cloud gaming; bounds grow with the duration ratio µ | arXiv 2304.08648 |
| T4-52 | VERIFIED_FIG | Azure: First Fit ≈1.13× lower bound; departure-aware Prioritized NRT ≈1.03× | arXiv 2602.14704 Figs. 3 and 8 |
| T4-53 | VERIFIED | Departure-time packing is fragile to prediction errors; synthetic-data rankings did not hold on real data | arXiv 2602.14704 |
| T4-54 | VERIFIED | LAVA in production: ~3%/~2% less stranding, 2.3–9.2 pp more empty hosts | arXiv 2412.09840 |
| T4-55 | VERIFIED | Equal-duration jobs: NextFit competitive ratio exactly 2 | arXiv 2108.12486 |
| T4-56 | VERIFIED | AlphaEvolve Borg heuristic recovers 0.7% of fleet compute; correct by construction | arXiv 2506.13131 |
| T4-57 | VERIFIED | FunSearch's bin-packing evaluator is 1-D and online, and items never leave | google-deepmind/funsearch notebook |
| T4-58 | VERIFIED | Best Fit ranks 1st across 6,064 instances; LLM heuristics don't generalise | arXiv 2501.11411 |
| T4-59 | VERIFIED | HELIOS: −9.6% penalised cost vs calibrated greedy; gain mostly from Spot; tiny search; guardrails needed | arXiv 2609.09164 |
| T4-60 | VERIFIED | Hetzner Cloud: hourly price with monthly cap; hours rounded up; billed until deleted | docs.hetzner.com |
| T4-61 | VERIFIED | CCX13 € 0.0697/h, € 43.49/month; all CCX plans "currently unavailable" on 2026-11-08 | hetzner.com/cloud/general-purpose |
| T4-62 | GUESS | Peak-only server (6 h/day) ≈ €12.55 vs €43.49/month (−71%) | arithmetic from T4-61 |
| T4-63 | VERIFIED | Spigot entity-activation-range: hand-set NPC LOD knobs | PaperMC/docs spigot.yml |
| T4-64 | VERIFIED | Rapier swapped SAP for BVH (2025) and reworked it for static worlds (2026) | rapier CHANGELOG |
| T4-65 | VERIFIED | LLM-generated heuristic code beats domain-independent planning heuristics | arXiv 2503.18809 |
| T4-66 | VERIFIED | MMO checkpointing: copy-on-update with double backup recommended | PVLDB 2009 abstract |
| T4-67 | GUESS | For Sumo Brothers on one box: partitioning irrelevant, packing worth ≈ €0 | context + T4-60/61 |
| T4-68 | GUESS | Packing headroom ≤11.5% over First Fit, ≤2.9% over Prioritized NRT (Azure); likely smaller for matches | arithmetic from T4-52 |
| T4-69 | GUESS | C2 is the most AI-search-ready; C1 is limited by its evaluator | synthesis |
| T4-70 | GUESS | Industry norm is static partitioning + instancing + TiDi; dynamic schemes take years | synthesis of T4-01..20, T4-36 |
| T4-71 | GUESS | No LLM program-search application to game-server partitioning or placement found | absence of evidence (arXiv/web search) |
