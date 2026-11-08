# T6: Has this been tried? Novelty check, the graveyard of cheap-MMO-scaling efforts, and what big companies already do

Researcher notes for Miles (Sumo Brothers, Godot, Hetzner). Written 2026-11-08. Claim ids are `T6-xx` and match the claim index at the end.

Labels used:
- **VERIFIED**: I read it in the primary source (link and exact quote given).
- **SNIPPET**: seen only in a search-engine excerpt because the page was blocked.
- **UNVERIFIED**: secondary source only, or from memory.
- **GUESS**: my own judgement.

An abstract fetched through the Semantic Scholar or Crossref API is the paper's own text, so I label it VERIFIED (abstract). Machine-written "TLDR" summaries from Semantic Scholar are labelled UNVERIFIED.

**Limits on this search (read first):**
- The shared WebSearch budget for this run ran out after my sixth search. Every later web lookup used only allowed hosts: GitHub (repos, raw files and the search API), arXiv, Crossref, Semantic Scholar and www.microsoft.com (Microsoft Research PDFs).
- The OpenAlex free daily budget for this IP was used up (HTTP 429), so I could not use OpenAlex at all.
- The Hacker News search API (hn.algolia.com), Reddit and the GDC Vault were all blocked.
- Hacker News, Reddit and GDC coverage is therefore thin. Section 9 lists the blocked hosts.

---

## 0. Bottom line in plain language

1. **The exact idea has not been published.** I searched arXiv, Semantic Scholar, GitHub repositories and GitHub code, and ran a web search. I found no public paper, repository or blog post that applies LLM-guided program search (FunSearch, AlphaEvolve, OpenEvolve, ShinkaEvolve and similar tools) to game-server problems: interest management, netcode, load balancing, session packing or server meshing. In that narrow sense the idea is new. (T6-02, T6-03, T6-14)
2. **The broader idea is 20 years old.** Using search or learning to optimise game servers has been tried many times:
   - genetic algorithms, ant-colony search and GRASP for splitting virtual worlds across servers (2003–2009);
   - genetic algorithms for placing game servers (2021, 2024);
   - deep reinforcement learning for game-server load balancing (2020–2021);
   - neural-network load prediction for renting servers (2008–2011);
   - learned message throttling for MMO servers (2026);
   - reinforcement-learning matchmaking deployed in live games (2021–2024).

   Apart from matchmaking at large studios, I found no sign that any of these were adopted in production. (T6-04 to T6-11)
3. **The graveyard mostly did not die from weak algorithms.** The causes were:
   - money and corporate decisions (Sun's Project Darkstar after the Oracle takeover; Improbable's SpatialOS);
   - integration cost and lock-in (SpatialOS for Unreal was "an Unreal Engine fork");
   - security and home-internet limits for peer-to-peer designs;
   - evaluations on made-up workloads that did not match real games.

   The clearest technical failure was architectural. Darkstar's own final README says adding server nodes "may reduce performance, not improve it". Better tuning of a heuristic would not have fixed that. (T6-20 to T6-45)
4. **The best primary source on why research was ignored** is a 2011 Cambridge thesis by John Miller (Microsoft Research). He names three reasons:
   - "Applicability of evaluation": researchers tested on their own made-up workloads.
   - "Deployability": algorithms "often failing catastrophically when deployed".
   - "Security".

   Any AI-found algorithm must pass the same three tests. (T6-39)
5. **Big platforms already own the decision points, and their shipped logic is simple.**
   - Roblox scores servers with a hand-weighted sum (Friends 15, Latency 3, and so on).
   - PlayFab sizes its standby pool with three threshold rules (×1.5, ×3, ×4).
   - GameLift FleetIQ ranks placements by a fixed order: latency, then cost, then destination, then location.
   - Agones defaults to "Packed".
   - Unity fills each snapshot packet from a priority queue.

   Simple rules mean there is room for improvement. But the companies can do this search themselves: Google already deployed an AlphaEvolve heuristic in its own Borg scheduler, Roblox has an ML matchmaking team, and authors who appear to be from NetEase report deploying RL matchmaking in two games. (T6-50 to T6-59)
6. **Verdict.** "AI-driven algorithm search for MMO servers" is under-explored in public, but the prize is smaller than it sounds. A result gets adopted only if it:
   - is plain, readable code;
   - drops into an existing plug-in point without forking an engine;
   - is tested on real player traces with a held-out test set;
   - leaves server authority and security unchanged;
   - reports savings in money and in player-perceived quality against a tuned baseline.

   An open benchmark built from real traces may be worth more to the community than any single evolved heuristic. (GUESS, T6-70 to T6-73)

---

## 1. What I searched (search log)

Raw results are saved under `/tmp/claude-0/-home-user-OogaPL/32372848-6606-541b-85af-54d683c4e916/scratchpad/dl/t6-*`, listed in section 10.

### 1.1 arXiv: 44 searches (arxiv.org/search, all fields, newest first)

- **Pure absence (0 results):** `"server meshing"`, `SpatialOS`, `"game session" placement`, `autoscaling "game server"`, `"area of interest" multiplayer`, `"multiplayer" "LLM-guided"`, `FunSearch game`, `"evolution of heuristics" game`, `"game server" evolutionary`, `"game server" "genetic"`, `"massively multiplayer" "load balancing"`, `"online game" "load balancing"`, `"LLM" heuristic "cloud gaming"`.
- **No relevant results:** `netcode` (3 hits, none about game networking); `"rollback netcode"` (1 hit, a multi-agent research framework); `"interest management"` (4 hits; only a 2015 position paper and a 2008 Second Life trace study are about games); `Agones` (only words like "agonist").
- **`"game server"` in all fields: 15 results in total.** Only one is about MMO server performance: AFLL (arXiv 2601.10998, January 2026). **VERIFIED [T6-01]** ([search](https://arxiv.org/search/?query=%22game+server%22&searchtype=all)). arXiv is not the main venue for game networking, which mostly appears in ACM/IEEE NetGames, MMSys and similar, so I also searched Semantic Scholar.
- **`OpenEvolve`: 27 results** (2025–2026), none about games or game servers.
- **`AlphaEvolve game`: 4 results**, all about game theory or multi-agent learning (for example arXiv 2602.16928, "Discovering Multiagent Learning Algorithms with Large Language Models"), not game servers.

### 1.2 Semantic Scholar: 36 bulk boolean queries, plus two batch lookups for abstracts

Queries are saved in `dl/t6-s2/bNN/query.txt`. Highlights:
- `("large language model" | LLM | LLMs) + (MMO | MMORPG | "massively multiplayer") + (server | servers | infrastructure | latency | bandwidth)`: **0 results**.
- `("large language model" | LLM | LLMs) + ("load balancing" | "bin packing" | scheduling) + (evolutionary | evolution | "program search") + (game | games)`: **0 results**.
- `(FunSearch | AlphaEvolve | OpenEvolve | ShinkaEvolve | "evolution of heuristics" | ReEvo) + (game | multiplayer | games)`: 7 results, all about game playing or game theory (for example "Code-Space Response Oracles", arXiv 2603.10098).
- `("large language model" | LLM | LLMs) + ("game server" | "multiplayer server" | netcode | "MMO server" | "server meshing" | "game servers" | "multiplayer game")`: 8 results. None optimise servers; they include a Roblox chat-moderation study and LLM game agents.
- `"server meshing"`: 14 results. None are about games; the only "server-meshed" paper is a 2026 robot-teleoperation paper.
- The many older hits for metaheuristics, RL and ML on game servers are in section 2.2.

### 1.3 GitHub: 44 repository searches and 18 code searches (api.github.com)

- **Repository searches with 0 results:** `openevolve game`, `funsearch game`, `shinkaevolve game`, `openevolve netcode`, `evolve netcode llm`, `llm evolutionary matchmaking`, `openevolve load balancing`, `mmo server llm optimization`, `game server reinforcement learning load balancing`, `area of interest reinforcement learning`, `netcode machine learning`.
- **Repository searches with only unrelated hits:** `alphaevolve game` (1 hit, VLM inference); `openevolve "game server" in:readme` (3 hits, all link lists).
- **Code searches for OpenEvolve's marker comment `"EVOLVE-BLOCK-START"`** combined with `netcode`, `"game server"`, `"area of interest"`, `snapshot players` or `bandwidth entities`: **0 results each**. **VERIFIED [T6-03]** (saved in `dl/t6-gh-code/`).
  - Caveat: GitHub code search only indexes default branches and part of all repositories, so this is weak evidence.
- `"funsearch.evolve" server` found only bin-packing and TSP demos.

### 1.4 Web search (6 queries before the shared budget ran out)

- Extended search `AlphaEvolve OR OpenEvolve OR FunSearch "game server" OR netcode OR "multiplayer" heuristic discovered`. The search tool's summary said: "I found no evidence of any AlphaEvolve, OpenEvolve, or FunSearch heuristic being discovered for a game server, netcode, or multiplayer system." **SNIPPET [T6-14]**.
- `LLM evolutionary search game networking interest management replication priority evolved heuristic`: nothing on game networking.
- `site:news.ycombinator.com AlphaEvolve game servers MMO`: the site filter was not applied and no Hacker News thread came back. I could not search Hacker News properly, because hn.algolia.com is blocked and the budget ran out.
- Three searches on company documentation (Roblox, GameLift FleetIQ, Unreal Iris) are used in section 4.
- **Not searched because of the budget:** Reddit, GDC Vault, Improbable, Hadean and Star Citizen 2026 news, and the Darkstar cancellation news.

---

## 2. Novelty check: what exists

### 2.1 LLM-guided program search on game-server problems: nothing public found

- **[T6-02] GUESS** (absence of evidence across all the searches in section 1): no paper, repository or blog applies FunSearch, AlphaEvolve, OpenEvolve or ShinkaEvolve-style LLM program search to interest management, netcode or replication priority, game-server load balancing or world partitioning, session or server packing, or server meshing.
- This agrees with the sibling notes T1 §7, T3 §7 and T4 §1.4, which searched independently.
- **The one game-company mention I know of** comes from sibling T1, not from me. T1 found a Kuro Games quote on Google Cloud's AlphaEvolve blog: "We applied it to a complex backend optimization challenge and saw substantial performance gains in specific server-side workloads." It gives no numbers. **UNVERIFIED [T6-12]**: T1 labels it VERIFIED, but I could not re-open the page because cloud.google.com is blocked. If true, at least one studio is already using commercial AlphaEvolve on server code in private. The public record would then understate how much this is being tried.

### 2.2 Earlier search and learning methods on the same problems (they exist, mostly unadopted)

| Year | Work | Method | Problem | What the source says | Label |
|---|---|---|---|---|---|
| 2003–2009 | Morillo, Orduña, Fernández, Duato (Univ. Valencia): "An ACS-based partitioning method for distributed virtual environment systems" (IPDPS 2003, [10.1109/IPDPS.2003.1213283](https://doi.org/10.1109/IPDPS.2003.1213283)); "A comparison study of evolutive algorithms for solving the partitioning problem in distributed virtual environment systems" (Parallel Computing 2004, [10.1016/j.parco.2003.12.006](https://doi.org/10.1016/j.parco.2003.12.006)); "A genetic approach for adding QoS to distributed virtual environments" (Computer Communications 2007, [10.1016/j.comcom.2006.08.015](https://doi.org/10.1016/j.comcom.2006.08.015)); "M-GRASP: A GRASP With Memory for Latency-Aware Partitioning Methods in DVE Systems" (IEEE TSMC-A 2009, [10.1109/TSMCA.2009.2025024](https://doi.org/10.1109/TSMCA.2009.2025024)) | ant colony, genetic algorithms, GRASP | assigning avatars to servers in a virtual world | Titles and venues confirmed (Crossref/S2). Semantic Scholar's machine TLDR of the 2007 paper: GA "can provide significantly better solutions than other heuristic methods while requiring shorter execution times" | **VERIFIED** metadata [T6-04]; result claim **UNVERIFIED** |
| 2021 | Zhao, Zheng, Liu, "Server Allocation for Massively Multiplayer Online Cloud Games Using Evolutionary Optimization", ACM TOMM, [10.1145/3433027](https://doi.org/10.1145/3433027) | genetic algorithm | renting cloud servers for MMO players at minimum cost | "The problem is formulated into minimizing an objective function involving the cost of server rental, the cost of data transfer and the network latency during the gaming time." "Extensive experiments based on the player behavior in “World of Warcraft” are conducted" "the method gives a lower cost and a shorter network latency in most of the time." | **VERIFIED** abstract [T6-05] |
| 2021 / 2024 | Benamer, Boussetta, Ben Hadj-Alouane, "A Genetic Algorithm for the Placement of Latency-Sensitive Multiplayer Game Servers in the Fog", GLOBECOM 2021, [10.1109/GLOBECOM46510.2021.9685952](https://doi.org/10.1109/GLOBECOM46510.2021.9685952) (journal version in Cluster Computing 2024) | ILP plus genetic algorithm | game-server placement | "An efficient and timely solution is provided, using a novel multi-penalty genetic algorithm (GA) heuristic." | **VERIFIED** abstract [T6-06] |
| 2020 / 2021 | Kim & Kim, "A load balancing scheme for gaming server applying reinforcement learning in IoT", ComSIS 2020, [10.2298/csis190917026k](https://doi.org/10.2298/csis190917026k); Kim, PPO version, JIPS 2021, [10.3745/JIPS.03.0158](https://doi.org/10.3745/JIPS.03.0158) | deep RL, PPO | game-server load balancing | "we proposed an agent that applies a deep reinforced learning method to distribute loads for gaming servers." | **VERIFIED** abstract [T6-07] |
| 2008–2011 | Nae, Iosup, Prodan et al., "Neural Network-Based Load Prediction for Highly Dynamic Distributed Online Games" (Euro-Par 2008, [10.1007/978-3-540-85451-7_22](https://doi.org/10.1007/978-3-540-85451-7_22)); "Dynamic Resource Provisioning in Massively Multiplayer Online Games" (IEEE TPDS 2011, [10.1109/TPDS.2010.82](https://doi.org/10.1109/TPDS.2010.82); 134 citations in S2) | neural-network forecasting | how many servers to rent, and when | (titles and metadata only; T4 has excerpts) | **VERIFIED** metadata [T6-11] |
| 2026 | Kang & Kim, "AFLL: Real-time Load Stabilization for MMO Game Servers Based on Circular Causality Learning", [arXiv 2601.10998](https://arxiv.org/abs/2601.10998) | learned weights (backpropagation) for throttling message types | MMO server overload | "traditional approaches either uniformly throttle all message types regardless of importance (damaging gameplay) or apply fixed heuristic rules that fail to adapt to dynamic workloads." "Through controlled experiments with 1,000 concurrent players, AFLL reduced average CPU time by 48.3% (13.2ms to 6.8ms)" | **VERIFIED** abstract (abs page fetched 2026-11-08) [T6-10]. The numbers differ from the earlier version T4 quoted (6,143 ms → 3,177 ms), so the paper has been revised. |
| 2014 | Millar, Hodson, Lamont, Peterson, "Multi-objective optimization of dead-reckoning error thresholds for virtual environments", CTS 2014, [10.1109/CTS.2014.6867626](https://doi.org/10.1109/CTS.2014.6867626) | multi-objective optimisation | dead-reckoning thresholds (netcode) | title only | **VERIFIED** metadata |
| 2007–2024 | Learned dead reckoning (McCoy et al. 2007; Duarte et al. 2020; Walker et al. 2024) | neural networks | netcode bandwidth | covered in T3 §7 | see T3 |

Plain reading: people have used search and learning on these exact problems before. What has not been tried in public is having an LLM write and evolve the heuristic code itself (GUESS).

### 2.3 Matchmaking: where large companies already deploy learned optimisers

| Work | Who | What the abstract says | Label |
|---|---|---|---|
| EOMM, WWW 2017, [arXiv 1702.06820](https://arxiv.org/abs/1702.06820) | Electronic Arts with UCLA authors (EA named in the abstract) | "matchmaking based on fairness is not optimal for engagement" "Our simulation on real data from a popular game made by Electronic Arts, Inc. (EA) supports our theoretical results" | **VERIFIED** abstract [T6-09] |
| Kolen et al., AAAI 2018, [10.1609/aaai.v32i1.11388](https://doi.org/10.1609/aaai.v32i1.11388) | EA | "These solutions, such as dynamic difficulty adjustment, in-game content and activity recommendations, matchmaking, and game balancing, have had major impact on engagement, revenue, and development resources within EA." | **VERIFIED** abstract [T6-09] |
| GloMatch, KDD 2021, [10.1145/3447548.3467074](https://doi.org/10.1145/3447548.3467074) | Deng, Li, Wang, Hu, Wu, Gong. I believe this is NetEase Fuxi AI Lab (**UNVERIFIED**, affiliation from memory) | "we solve it with the help of an effective policy-based deep reinforcement learning algorithm. Quantitative experiments on simulation and online game environments demonstrate the effectiveness" | **VERIFIED** abstract [T6-08] |
| EnMatch, AAAI 2024, [10.1609/aaai.v38i8.28760](https://doi.org/10.1609/aaai.v38i8.28760) | same group | "we formulate the problem into a reinforcement learning framework, in which a neural combinatorial optimization problem is built and solved." "...based on several real-world datasets and online deployments on two games." | **VERIFIED** abstract [T6-08] |
| MGMatch, KDD 2024, [10.1145/3637528.3671553](https://doi.org/10.1145/3637528.3671553) | same group | "Evaluation results on real industry datasets show that our method can deliver near-optimal solutions within 100ms." | **VERIFIED** abstract [T6-08] |

Plain reading: matchmaking is the one server-side decision where big studios already apply serious learned optimisation and report it. For a small developer, matchmaking competes with these in-house teams (GUESS).

### 2.4 Related but different: LLM program search on networking problems outside games

Sibling T3 §7 lists LLM program search on adjacent networking problems: NADA (video bitrate adaptation), PolicySmith (caching and congestion control), ReForge (bitrate rules) and Glia (LLM inference clusters). The method works on neighbouring systems problems; no one has pointed it at game servers yet (see T3; not re-verified here).

### 2.5 What the absence means (GUESS [T6-70])

- **Absence of public work is real, but weaker than it looks:**
  - Game studios rarely publish infrastructure work. EA and NetEase do, but Roblox, Epic, Riot and others mostly do not.
  - The AlphaEvolve service is sold privately.
  - arXiv has only 15 papers that mention "game server" at all.
- **The field is small, and the old work did not get used.** That is a warning about demand, not only an opportunity.

---

## 3. The graveyard: who tried to make big worlds cheap, and why they stalled

### 3.1 Sun Microsystems' Project Darkstar, then the RedDwarf Server fork

**What it promised.** From the RedDwarf distribution README in the final source tree ([github.com/dworkin/reddwarf](https://github.com/dworkin/reddwarf), file `sgs-server-dist/src/main/etc/README`; repository last pushed 2014-02-18): "RedDwarf lets developers write scalable, reliable, persistent and fault-tolerant game server applications using simple single-threaded, event-driven code." **VERIFIED**

**What it delivered.**
- From the same README: "this distribution contains a complete single-system version of the RedDwarf Server."
- Its "KNOWN ISSUES" section: "Multi-node operation is currently not scalable. Depending on the application, adding additional application nodes to a multi-node cluster may reduce performance, not improve it. We expect this behavior to improve in future releases." **VERIFIED [T6-20]** (local copy: `dl/t6-reddwarf-files/sgs-server-dist_src_main_etc_README`).

**They did try algorithmic load balancing, late.**
- CHANGELOG, version 0.10.0: "Rebranded as RedDwarf (ticket #2)" and "Added affinity graph builders and group finders, for load balancing multi-node clusters (required for issue #54)".
- The `affinity` package uses community detection. The package docs say an affinity group "is a set of identities which are affiliated with each other and should be co-located on a node". The group finder implements "the label propagation algorithm described in "Near linear time algorithm to detect community structures in large-scale networks" by Raghavan, Albert and Kumara (2007)". The files are headed "Copyright 2007-2010 Sun Microsystems, Inc."
- **VERIFIED [T6-21]** ([affinity package](https://github.com/dworkin/reddwarf/tree/master/sgs-server/src/main/java/com/sun/sgs/impl/service/nodemap/affinity); local `dl/t6-reddwarf-files/`).
- So an "AI-ish" heuristic for deciding who shares a server already existed in 2010. It did not rescue a design whose shared transactional data store made extra nodes a net loss. This is GUESS: the README states the symptom, not the root cause.

**What it was.**
- A 2009 thesis abstract: "Project Darkstar is a Sun Labs research project, which has developed an open-source middleware for MMOGs." **VERIFIED [T6-22]** (Rashid 2009, abstract via [Semantic Scholar](https://www.semanticscholar.org/paper/a05acfe9608d4c7cbbb5641ae6d6a3aabb6537bc)).
- Its lead, Jim Waldo, summarised the lesson in CACM (August 2008): "Online games and virtual worlds have familiar scaling requirements, but don't be fooled: Everything you know is wrong." **VERIFIED [T6-23]** (abstract via Crossref, [10.1145/1378704.1378716](https://doi.org/10.1145/1378704.1378716)).

**Why it ended.**
- A copy of a wiki article stored on GitHub says: "Upon Oracle's acquisition of Sun, support for Project Darkstar was stopped... the community rebranded the latest codebase of Project Darkstar's repositories and released it as RedDwarf Server" ([mihaibivol/Xapian-Testing, cswiki/RedDwarf_Server](https://github.com/mihaibivol/Xapian-Testing/blob/master/cswiki/RedDwarf_Server)). **UNVERIFIED [T6-24]** (secondary). It is consistent with the 0.10.0 "Rebranded as RedDwarf" changelog entry (VERIFIED). I could not reach Wikipedia, the Wayback Machine or news sites.

**Verdict: both causes.** The scale-out architecture did not deliver (technical), and funding ended with the acquisition (business). A cleverer placement heuristic was tried and was not the bottleneck (GUESS).

### 3.2 Improbable's SpatialOS

**Integration cost.** From the Unreal GDK README ([github.com/spatialos/UnrealGDK](https://github.com/spatialos/UnrealGDK), `master`, last commit 2022-01-24):
- "The SpatialOS Game Development Kit (GDK) for Unreal is an Unreal Engine fork and plugin with associated projects."
- "In order to transform Unreal from a single-server engine to a distributed model, we made a number of small changes to Unreal Engine code."
- **VERIFIED [T6-25]**

**End of the platform.** From the same README:
- "This is the repository for the last version of the GDK plugin that is compatible with the SpatialOS legacy platform. The GDK is frozen for users who have not yet migrated their project to SpatialOS running on zeuz."
- Users with projects were asked to contact Improbable "before the 30th December 2021".
- The repositories are archived (GitHub search API: `spatialos/UnrealGDK` and `spatialos/gdk-for-unity`, `archived: true`).
- SpatialOS Online Services README ([github.com/spatialos/online-services](https://github.com/spatialos/online-services)): "Please note that this code has been deprecated and is no longer being supported in any way by Improbable."
- **VERIFIED [T6-25, T6-27]**

**The shipped load balancing was simple.**
- The final public GDK release (branch `0.11.0`, commit 884b065, 2020-09-03, "Merging final GDK for Unreal 0.11.0 release") ships `GridBasedLBStrategy.h`: "A load balancing strategy that divides the world into a grid. Divides the load between Rows * Cols number of workers, each handling a square of the world"; "Intended Usage: Create a data-only blueprint subclass and change the Cols, Rows, WorldWidth, WorldHeight." **VERIFIED [T6-26]** ([file](https://github.com/spatialos/UnrealGDK/blob/0.11.0/SpatialGDK/Source/SpatialGDK/Public/LoadBalancing/GridBasedLBStrategy.h)).
- T4 found the same static layouts in the Unity GDK.

**What happened to games built on it.**
- The fan project Worlds Adrift Reborn ([github.com/WAReborn/WorldsAdriftReborn](https://github.com/WAReborn/WorldsAdriftReborn)) "attempts to recreate the original MMO experience by replacing the official infrastructure with custom, self-hosted substitutes". One component is "A custom replacement for the SpatialOS SDK written in C++".
- It warns that "a final update before shutdown removed most of the game content."
- **VERIFIED [T6-28]**: the game shut down, and keeping it alive meant rebuilding the proprietary SDK.

**Cost complaints.** Sibling T4 found search excerpts (SNIPPET there) saying that for the SpatialOS game Lazarus, "Operating the game would be so costly due to fees for software licenses, server costs, and the effort of constant updates" (mein-mmo.de). T4 also found that Improbable sold its studios. **UNVERIFIED [T6-29]** (I could not re-check).

**Verdict: mainly business and operations.** The causes were platform fees and costs, lock-in, a forked engine, and games that did not succeed. The public load balancer was a static grid, so these failures were not caused by a weak load-balancing algorithm (GUESS).

### 3.3 Hadean (Aether Engine)

- **Public activity stops early.** Hadean's public GitHub repositories stop around 2019: `hadeaninc/aether-demo-client` last pushed 2018-04-18, `hadeaninc/libzinc` 2019-01-27, `hadeaninc/rehosted-artifacts` 2023-03-30 (GitHub search API). **VERIFIED [T6-30]** (metadata).
- **The pitch.** The README says: "Aether Engine is a Distributed Simulation Engine designed to effortlessly create virtual worlds of unprecedented scale and complexity." **VERIFIED [T6-30]**
- **Later direction.** From memory, Hadean shifted toward defence and enterprise simulation. **UNVERIFIED [T6-31]**; www.hadean.com is blocked and the web budget had run out. T4 has a SNIPPET on the 2019 EVE "Aether Wars" demo (about 14,000 clients in one source).
- **Verdict: unclear from primary sources.** There is no sign of an algorithmic failure. The visible pattern is that a scale technology found no lasting game market (GUESS).

### 3.4 Star Citizen server meshing (status in 2026)

All Cloud Imperium (CIG) hosts are blocked. I read the community wiki "Unofficial Road to Dynamic Server Meshing" ([github.com/un0btanium/sc-server-meshing](https://github.com/un0btanium/sc-server-meshing), commit 6514c1e, 2026-08-29). It links its claims to CIG patch notes, monthly reports and letters. **UNVERIFIED [T6-32, T6-33]**: a secondary community source.

- **Static Server Meshing.** Its source list cites the Alpha 4.0 patch notes ("Alpha 4.0 - Static Server Meshing"). T4 adds a SNIPPET: "Server Meshing: 5:5:500 Player Shards", December 2024.
- **Quasi-Dynamic Server Meshing.** "First introduced in Alpha 4.10 with Siege of Orison being reworked as a 4 player squad instanced experience." This puts instances on separate game servers. The cited source is "Alpha 4.10 - Server Instancing, Quasi-Dynamic Server Meshing" (patch notes) and the Monthly Report of July 2026.
- **Entity Authority Load Balancer:**
  - "not a lot is known about this feature and in which form it will be implemented (e.g. utilizing a clustering algorithm)."
  - It quotes CIG: "Progress was also made on a smarter way to divide the game world into regions that can each be handled by their own server, including support for grid-based spatial mapping that makes it easier to pinpoint exactly which servers should be responsible for specific areas of space."
- **Dynamic Server Meshing V1** is still pending. The wiki states the motivation: "With Static Server Meshing, the load can't be optimally balanced yet. Some game servers might be underutilized, others might be overloaded. This is also not efficient in terms of server cost".
- T4 has SNIPPETs from CIG's 27 August 2026 letter ("Quasi Dynamic Server Meshing (QDSM) because we are not yet subdividing territories") and from the 7 October 2026 roadmap ("DGS Planet Subdivision", "Tentative" for 4.11).

**Verdict.** After more than a decade, dynamic load balancing is still not shipped. The visible blockers are engineering: entity authority hand-over, persistence, the replication layer and crash recovery. A missing heuristic is not the visible blocker (GUESS). Once CIG does split territories dynamically, the territory-assignment heuristic is the kind of piece an AI search could tune. CIG is a private company with a large team, though (GUESS).

### 3.5 Peer-to-peer MMO research: Colyseus (NSDI 2006), Donnybrook (SIGCOMM 2008) and the post-mortem

**Donnybrook** (Bharambe, Douceur, Lorch, Moscibroda, Pang, Seshan, Zhuang). I read the full PDF ([microsoft.com copy](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/donnybrook.pdf); local `dl/t6-donnybrook/donnybrook.txt`). **VERIFIED**
- **Motivation was cost** [T6-34]:
  - "Peer management reduces play cost, allows small publishers to enter the market, and reduces the investment risk in games of unknown popularity."
  - "the key limitation to scale in peer-managed games is the players’ upload capacity."
- **Hand-tuned weights** [T6-13]: "With only several hours of tuning on our part, we found that weighting interaction recency 1.5 times more than proximity and aim yielded good playability independent of game scale in Quake III". This is a hand-tuned attention function that an AI search could tune. Its quality was judged by a human user study, not a formula.
- **Its critique of Colyseus** [T6-35]: "despite Colyseus’s scalable object discovery, it will still require a player to send updates every frame to every other player who is near his objects. Hence, bandwidth requirements remain quadratic in the number of players in the battle."
- **What was left undone** [T6-36]:
  - "We are currently working on an Internet deployment of a large-scale game using Donnybrook, to learn more about the benefits and limitations of our approach."
  - "we may discover how players try to gain unfair advantage from Donnybrook’s new techniques and what is necessary to defend against such cheating."
  - On home routers: Some “strict” NATs do not support STUN-like approaches, so "We exclude players using such NATs".
- I found no record of that Internet deployment. **GUESS [T6-37]** (absence of evidence; web budget exhausted).

**Colyseus itself.**
- I could not read the NSDI 2006 paper (usenix.org and cs.cmu.edu are blocked).
- Semantic Scholar's machine summary of the CMU tech-report version says "per-node bandwidth requirements for Colyseus are an order of magnitude lower than traditional client-server or broadcast architectures". **UNVERIFIED**
- The Donnybrook critique above (by overlapping authors) is the VERIFIED limitation.

**The post-mortem by an MMO-trace researcher.** Miller and Crowcroft, "The Near-Term Feasibility of P2P MMOG’s", NetGames 2010 ([PDF on microsoft.com](https://www.microsoft.com/en-us/research/wp-content/uploads/2010/11/NetGames-P2PMMOG.pdf)). **VERIFIED [T6-38]**
- "Ten years of significant research on peer-to-peer distributed virtual environments (P2P DVE’s) has yielded many new techniques, yet few have been adopted by deployed systems."
- "We found pure P2P pub-sub solutions would occasionally saturate residential broadband connections, and would result in average latency more than 10x greater than client-server solutions."

**Miller's thesis** (Cambridge UCAM-CL-TR-809, 2011; [PDF on microsoft.com](https://www.microsoft.com/en-us/research/wp-content/uploads/2011/10/UCAM-CL-TR-809.pdf)). **VERIFIED [T6-39]**
- "Deployed DVEs remain expensive centralized implementations despite significant research outlining ways to distribute DVE workloads."
- "This dissertation shows previous DVE research evaluations are inconsistent with deployed DVE needs."
- "I found many promising approaches, none of which have been adopted by broadly distributed DVEs." He gives three possible reasons:
  1. "Applicability of evaluation. Most of the research proposals include evaluation against researcher-proposed workloads, without any clear link to broadly adopted real-world systems."
  2. "Deployability. Algorithms which work well in numerical simulations and in well-provisioned computer labs perform differently in the real world, often failing catastrophically when deployed."
  3. "Security. Many of the decentralization proposals make the simplifying assumption that security will be handled separately." Also: "as online game providers have found, security is paramount for DVE adoption and longevity."

**The survey.** Yahyavi and Kemme, ACM Computing Surveys 2013 ([10.1145/2522968.2522977](https://doi.org/10.1145/2522968.2522977); abstract via Crossref). **VERIFIED [T6-40]**
- "Distributing a game among peers makes maintaining control over the game more complex."
- "Peer-to-peer architectures also tend to be vulnerable to churn and cheating."

**Verdict.** P2P failed on deployability (home upload bandwidth and latency), security and cheating, and evaluation realism. Within those limits the algorithms worked (VERIFIED from the sources above). The cost motive was the same as Miles's, but the industry chose central servers.

### 3.6 Kiwano, Manycraft and OneSim (Orange Labs / LIP6, 2013–2016)

**Papers** (VERIFIED metadata via Crossref):
- "Kiwano: A scalable distributed infrastructure for virtual worlds" (HPCS 2013, [10.1109/HPCSim.2013.6641489](https://doi.org/10.1109/HPCSim.2013.6641489))
- "Manycraft: Scaling minecraft to millions" (NetGames 2013, [10.1109/NETGAMES.2013.6820617](https://doi.org/10.1109/NETGAMES.2013.6820617))
- "Kiwano: Scaling virtual worlds" (WSC 2016, [10.1109/WSC.2016.7822230](https://doi.org/10.1109/WSC.2016.7822230))

**OneSim** (MMVE 2014, [10.1145/2577387.2577394](https://doi.org/10.1145/2577387.2577394)): "a distributed system to allow an unlimited number of users to be together in one sim". **VERIFIED [T6-42]** (abstract).

**Diaconu's thesis** ("Scalability for virtual worlds", 2015; abstract via [Semantic Scholar](https://www.semanticscholar.org/paper/1b3ff7e1a4450ad9f367ce3d45439f50a01912ad); the PDF on theses.hal.science is blocked). **VERIFIED [T6-41]**
- "Mais leur déploiement à large échelle reste difficile et il est en conséquence souvent évité." (Large-scale deployment "remains difficult and is consequently often avoided".)
- "Dans Kiwano nous utilisons la triangulation de Delaunay pour fournir à chaque avatar un nombre constant de voisins en moyenne, indépendamment de leur densité ou distribution géographique." (Delaunay triangulation gives each avatar a roughly constant number of neighbours.)
- Evaluation was by simulation: "évalués en simulant des dizaines de milliers d'avatars connectés à la même instance de Kiwano" ("evaluated by simulating tens of thousands of avatars").

I found no production adoption. **GUESS [T6-43]** (absence of evidence).

### 3.7 Others seen in passing

- **channeld / ChanneldUE.** An open-source distributed Unreal dedicated-server system, still active ([github.com/channeldorg/channeld-ue-plugin](https://github.com/channeldorg/channeld-ue-plugin), pushed 2025-12-26). Its README lists:
  - "Increase the maximum capacity of a single UE dedicated server to 100-200 players without modifying the engine code."
  - "Cloud-based dynamic load balancing can greatly save server costs (under development)."
  - **VERIFIED [T6-44]**: an open system where the dynamic load balancer is still unwritten. It is a possible test bed.
- **Second Life traces** (Liang et al., [arXiv 0807.2328](https://arxiv.org/abs/0807.2328)): earlier research "mainly based their evaluations either on simulations with a simple model of avatar behavior (such as random way-point mobility model), or on collected traces from small-scale games and NVEs." **VERIFIED [T6-45]**. This is the same evaluation-realism problem Miller describes.
- **Minecraft's Folia and MultiPaper** (alive, in public beta): see T4 §1.2.

### 3.8 Summary: algorithm or business?

| Effort | What it promised | What happened (primary evidence) | Main cause | Label |
|---|---|---|---|---|
| Project Darkstar / RedDwarf | single-threaded code, automatic scale-out | "Multi-node operation is currently not scalable"; support stopped after Oracle bought Sun; community fork | **both**: architecture plus funding | VERIFIED / UNVERIFIED |
| SpatialOS | many servers per world as a paid platform | Unreal fork; legacy platform frozen (2021); services deprecated; games shut down; static grid load balancing | **business and operations** (cost, lock-in) | VERIFIED |
| Hadean | huge simulated worlds | public game-facing repos stop in 2019 | unclear; probably market | VERIFIED metadata / GUESS |
| Star Citizen | dynamic meshing | static (2024) and quasi-dynamic (2026) only | **engineering complexity** | UNVERIFIED |
| Colyseus, Donnybrook, P2P research | server-free scale | "few have been adopted"; residential links saturate; cheating | **deployability, security, evaluation** | VERIFIED |
| Kiwano | unlimited avatars | simulation only | **demand and deployment** | VERIFIED / GUESS |
| GA/ACS/GRASP partitioning (2003–09) | better world partitions | papers only | **evaluation realism, demand** | GUESS |

**[T6-71] GUESS.** None of these died because the heuristic was 10% worse than it could have been. A better heuristic found by AI search would not have saved any of them. It can only help where the system already works and a scoring or priority function is the remaining knob.

---

## 4. What large companies already do in-house (the competition)

### 4.1 Roblox

**Default matchmaking** (creator docs on GitHub, [scoring.md](https://github.com/Roblox/creator-docs/blob/main/content/en-us/matchmaking/scoring.md) and [index.md](https://github.com/Roblox/creator-docs/blob/main/content/en-us/matchmaking/index.md), fetched 2026-11-08). **VERIFIED [T6-50]**
- "When a player tries to join your game, the matchmaking service finds all eligible servers that the player can join, scores these servers using signals, and matches the player to the server with the highest score."
- "Servers are scored using a weighted sum formula."
- Default weights: Friends 15, Latency 3, Text Chat 3, Occupancy 2, Play History 2, Language 2, Age 1, Voice Chat 1, Device Type 0.
- "In the default Roblox configuration, the Friends signal's weight is greater than the sum of the weights of every other signal."

**In-house ML.**
- Roblox job adverts describe an ML matchmaking team building "candidate-server retrieval, ranking, and assignment algorithms" under latency, capacity and reliability limits, plus online and continual learning ([freehire.me listing](https://freehire.me/jobs/senior-machine-learning-infrastructure-engineer-matchmaking-roblox-hxvznq7l), [zapply.jobs listing](https://zapply.jobs/jobs/4b6aa267-bc02-42b4-a8d1-61a481a4b978/)). **SNIPPET [T6-51]**
- A 2023 Roblox newsroom post says matchmaking involves "orchestrating Roblox engine instances across our fleet of edge data centers" ([about.roblox.com](https://about.roblox.com/newsroom/2023/10/inside-tech-solving-matchmaking-roblox), blocked). **SNIPPET [T6-52]**
- Plain reading: Roblox's creator-facing scorer is a hand-weighted linear function, and Roblox staffs a team to learn better ones. An outsider cannot plug code into Roblox's matcher; creators can only set weights or write custom matchmaking (GUESS).

### 4.2 Unreal Engine Iris (Epic)

dev.epicgames.com is blocked. From the search excerpt of the Iris prioritization docs ([iris-prioritization-in-unreal-engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/iris-prioritization-in-unreal-engine)):
- Prioritizers are pluggable: a developer implements `UNetObjectPrioritizer` and registers it under `NetObjectPrioritizerDefinitions`.
- A sphere prioritizer is built in. One example "adds the value of OwnerPriorityBoost to the priority that is normally calculated by the sphere prioritizer for the owning connection".
- Priority accumulates across ticks until the object is sent.
- **SNIPPET [T6-53]**
- T4 also has a SNIPPET that Epic's multi-server replication plugin is "experimental".
- Plain reading: Epic ships hand-designed prioritizers and a plug-in interface. An evolved prioritizer could be delivered as a plug-in without forking the engine (GUESS).

### 4.3 Unity Netcode for Entities

From the package docs (mirror of the published package `com.unity.netcode@7.2.0`, released 2026-10-29; [optimize-ghosts.md](https://github.com/needle-mirror/com.unity.netcode/blob/master/Documentation~/optimization/optimize-ghosts.md)). **VERIFIED [T6-54]**
- "The server operates with a fixed bandwidth target and sends a single snapshot packet of customizable size on every network tick."
- "It fills this packet with the ghosts with the highest importance, determined by a priority queue of ghost chunks (rebuilt each tick)."
- "You can also supply your own method to scale importance on a per-chunk, per-connection basis, using `GhostImportance.BatchScaleImportanceFunction`."
- "The built-in form of importance scaling in Unity Netcode is distance-based".
- Plain reading: Unity documents an official slot for a custom priority function. That is the ideal drop-in point for an AI-found heuristic (GUESS).

### 4.4 AWS GameLift (now "Amazon GameLift Servers") and FleetIQ

From the developer guide archived on GitHub ([awsdocs/amazon-gamelift-developer-guide](https://github.com/awsdocs/amazon-gamelift-developer-guide), commit 7e08295, 2023-06-15; files `doc_source/gamelift-howitworks.md`, `queues-design.md`, `spot-tasks.md`). **VERIFIED [T6-55]**
- "Queues use an algorithm, called FleetIQ, to look for the best possible placement for each game session request. The FleetIQ algorithm prioritizes the search for available game servers based on low player latency, hosting costs, geographical locations, or other fleet characteristics."
- Default order: "**Latency** – Lowest average latency for all players in the request", then "**Cost** – Lowest hosting cost, if latency is equal in multiple locations", then destination, then location.
- On Spot instances: queues enable "the use of FleetIQ to avoid resources with a high chance of interruption".

The current FleetIQ guide (docs.aws.amazon.com, blocked) is described in a search excerpt: FleetIQ uses a prediction algorithm based on each instance type's recent availability and interruption rate ([gsg-howitworks-logic](https://docs.aws.amazon.com/gameliftservers/latest/fleetiqguide/gsg-howitworks-logic.html)). **SNIPPET [T6-56]**

Plain reading: AWS already owns the Spot-risk prediction and the placement ranking. Its users can only reorder the criteria (GUESS).

### 4.5 Google: Agones, and AlphaEvolve inside Google

**Agones** ([scheduling-and-autoscaling.md](https://github.com/googleforgames/agones/blob/main/site/content/en/docs/Advanced/scheduling-and-autoscaling.md)). **VERIFIED [T6-57]**
- "The default scheduling strategy (`Packed`) is designed to work with the Kubernetes autoscaler out of the box."
- "Under the "Packed" strategy, allocation will prioritise allocating `GameServers` to nodes that are running on Nodes that already have allocated `GameServers` running on them."

**AlphaEvolve on Google's own cluster scheduler, Borg** ([arXiv 2506.13131](https://arxiv.org/abs/2506.13131) v1, §3.3.1; local `dl/t6-alphaevolve/ae.txt`). **VERIFIED [T6-59]**
- "Because this heuristic only influences the ranking of machines already determined by Borg to be available and capable of running each pending job, the resulting scheduling decisions are effectively correct by construction."
- "We use a simulator of our data centers to provide feedback to AlphaEvolve based on historical snapshots of workloads and capacity across Google’s fleet."
- "We measure the performance of AlphaEvolve’s heuristic function on an unseen test dataset of recent workloads and capacity to ensure generalization."
- The heuristic "continuously recovers on average 0.7% of Google’s fleet-wide compute resources, which would otherwise be stranded."
- "AlphaEvolve was chosen over a deep reinforcement learning approach because its code solution not only leads to better performance, but also offers clear advantages in interpretability, debuggability, predictability, and ease of deployment—essential qualities for a mission-critical system."

Plain reading: the method's owner already applies it to its own placement problem. Anyone running game servers at Google scale can do the same in-house.

### 4.6 Microsoft PlayFab Multiplayer Servers

From the PlayFab docs on GitHub ([dynamic-standby.md](https://github.com/MicrosoftDocs/playfab-docs/blob/docs/playfab-docs/multiplayer/servers/dynamic-standby.md), ms.date 03/22/2021). **VERIFIED [T6-58]**
- "If Dynamic Standby is enabled, an auto scaling heuristic will trigger and adjust the target standby value used by the platform to compensate for the rate of active server allocations."
- The three rules:
  - "IF (Active Servers > 1X Target Standby) AND ((Actual Standby / Target Standby Floor) < 0.50) THEN Target Standby = 1.5 * Target Standby"
  - the same rule with 0.25 → 3.0 ×
  - the same rule with 0.005 → 4.0 ×
- Plain reading: a big vendor's warm-pool policy is three hand-set threshold rules. That is plenty of room for search, but the gain is limited by how much idle standby capacity costs (see T2 and T4).

### 4.7 What this means for competition (GUESS [T6-72])

- **Where an outsider could contribute:**
  - Open-source engines and orchestrators where heuristics are visible and pluggable: Unity's importance function, Unreal Iris prioritizers, Agones strategies, channeld's unwritten load balancer, Folia and MultiPaper, Godot.
  - Small and medium studios that run their own servers, as Miles does, and cannot hire an ML team.
- **Where an outsider cannot compete:**
  - Closed platform internals: Roblox's matcher, FleetIQ, Fortnite or Star Citizen back ends.
  - The big companies either already apply learned optimisation (EA, NetEase, Roblox) or own the search tool (Google).

---

## 5. Is "AI-driven algorithm search for MMO servers" genuinely under-explored?

**Yes, in the narrow sense; no, in the broad sense; and the reasons matter.** (GUESS [T6-70], built on the VERIFIED items above)

- **The narrow sense is unexplored.** I found no public work evolving game-server heuristic code with LLMs. The nearest are LLM program search on non-game networking (T3) and cluster scheduling (AlphaEvolve Borg).
- **The broad sense is old.** Metaheuristics, RL and neural forecasting have been applied to the same problems since 2003. Apart from matchmaking, they produced papers rather than products.
- **Why the space is quiet (GUESS):**
  - Game-server infrastructure is a small, secretive field (15 arXiv papers mention "game server").
  - The key heuristics sit inside closed platforms.
  - Realistic evaluation data is scarce (Miller; Liang; see T5).
  - For most games, server cost is not the factor that decides whether the game succeeds (see T2).
  - The famous failures were caused by business, integration and security problems, which a better heuristic cannot fix.
- **"Under-explored" is therefore partly a warning.** The space is quiet partly because the payoff has been small, not only because nobody thought of it.

---

## 6. What would make a result get adopted rather than ignored

An evidence-based checklist: each item is tied to a primary source.

1. **Plug into an existing slot; do not fork an engine.**
   - Evidence: SpatialOS was "an Unreal Engine fork", and its games died with the platform (T6-25, T6-28).
   - Slots that already exist: Unity's `BatchScaleImportanceFunction` (T6-54), Iris prioritizers (T6-53), Agones allocation strategy and priorities (T6-57), and channeld's missing load balancer (T6-44).
2. **Make it safe by construction.** Let the heuristic only re-rank options the system already considers valid, as with AlphaEvolve in Borg: "effectively correct by construction" (T6-59).
3. **Ship readable code, not a neural network.** AlphaEvolve beat deep RL on "interpretability, debuggability, predictability, and ease of deployment" (T6-59).
4. **Use real traces and a held-out test set.**
   - Miller's first reason for non-adoption: "evaluation against researcher-proposed workloads" (T6-39).
   - Liang: "random way-point" models are inadequate (T6-45).
   - Google tested on "an unseen test dataset of recent workloads" (T6-59).
5. **Test in deployment-like conditions.** Miller's second reason: algorithms "often failing catastrophically when deployed" (T6-39). Run the result on real WAN latency and loss, not only in a simulator.
6. **Do not weaken server authority or security.** Miller's third reason, plus the P2P history (T6-38 to T6-40). A heuristic that saves bandwidth by trusting clients will not be adopted.
7. **Measure what players notice, not only bytes or CPU.** Donnybrook needed a human user study and hand-tuned weights (T6-13). A good proxy metric, or a small user study, is part of the deliverable.
8. **Beat a tuned baseline and report money.** Comparing against "learning disabled" (AFLL, T6-10) or a naive baseline will not convince engineers. Report euros per month at a stated player count, for example on Hetzner (see T2 for prices). (GUESS)
9. **Publish an open benchmark and keep it maintained.** The graveyard shows that companies and lone projects disappear (Darkstar, SpatialOS, Hadean). A benchmark plus an MIT-licensed reference heuristic survives its authors. Fiedler's 2015 compression challenge (see T3 §3.3) is an example of a game-networking benchmark that drew outside entries. (GUESS)

---

## 7. Candidate problems this topic points to (also in the structured result)

These overlap with T3 and T4. I list two whose adoption path is clearest from the evidence above.

**C1. Warm-standby sizing for session-based game servers.**
- **Baselines:** PlayFab's three-rule Dynamic Standby (T6-58); a fixed buffer; GameLift-style target tracking (T4).
- **Metric:** total server-hours plus λ × player-seconds spent waiting for a server, or allocation failures, on held-out demand traces.
- **Interface:** a plain function, `target_standby = f(recent allocations, active count, time of day)`.
- **Risk:** small absolute savings for small fleets, and Hetzner billing granularity limits the gains (see T2).

**C2. Co-location (affinity) heuristic for multi-server worlds.**
- **Baselines:** label propagation as in Darkstar (T6-21); a static grid as in SpatialOS (T6-26); Folia proximity regions and MultiPaper first-come-first-served (T4).
- **Metric:** cross-server interactions per tick plus a penalty for peak load imbalance and for migrations per minute, on Second Life and WoW traces.
- **Interface:** a scoring function for region or entity assignment, for example inside channeld's unwritten load balancer.
- **Risk:** few games need multi-server worlds; Sumo Brothers almost certainly does not (GUESS).

---

## 8. Open questions

- Are there AlphaEvolve-on-Google-Cloud case studies from game studios beyond Kuro Games, and what did Kuro optimise? (cloud.google.com is blocked.)
- What does Roblox's internal ML matchmaker optimise, and how is it evaluated? (Roblox sites are blocked; only job-advert snippets were seen.)
- Did Donnybrook's planned Internet deployment ever happen? (No record found.)
- What is the 2026 status of Improbable (M²/Morpheus) and Hadean in games? (Their sites are blocked and the web budget was exhausted.)
- Did Star Citizen's "DGS Planet Subdivision" ship in Alpha 4.11 before 2026-11-08? (CIG hosts are blocked; see T4.)
- I could not read the primary text of Colyseus (NSDI 2006) or of the Valencia GA/ACS partitioning papers, so their quantitative results are not verified here.

---

## 9. Blocked or limited hosts (claims from them are SNIPPET or UNVERIFIED)

- **Proxy 403 or DNS failure:** docs.aws.amazon.com, docs.amazonaws.cn, about.roblox.com, dev.epicgames.com, informatica.uv.es, inria.hal.science (and theses.hal.science), dl.acm.org, www.usenix.org, www.cs.cmu.edu, starcitizen.tools, robertsspaceindustries.com, packages.unity.com, www.hadean.com, ims.improbable.io, web.archive.org, en.wikipedia.org, learn.microsoft.com, export.arxiv.org, hn.algolia.com, www.reddit.com.
- **Rate-limited:**
  - api.openalex.org: daily free budget exhausted, HTTP 429 "Insufficient budget".
  - api.semanticscholar.org: intermittent 429; most queries eventually succeeded.
- **WebSearch:** the shared per-turn budget was exhausted after 6 of my queries, so there were no further web searches.
- **Worked:** github.com (search API, raw files, git clone), arxiv.org, api.crossref.org, api.semanticscholar.org, www.microsoft.com (Microsoft Research PDFs and its WordPress API).

---

## 10. Files downloaded (all under `/tmp/claude-0/-home-user-OogaPL/32372848-6606-541b-85af-54d683c4e916/scratchpad/dl/`)

- **Search results:**
  - `t6-arxs/`, `t6-arxs2/`: arXiv search pages (44 queries)
  - `t6-s2/b01..b36/`: Semantic Scholar bulk queries; `t6-s2-batch1/`, `t6-s2-batch2/`: abstracts
  - `t6-gh-search*/`, `t6-gh-code*/`: GitHub repository and code searches
  - `t6-crossref/`, `t6-crossref-darkstar/`: Crossref metadata and abstracts
- **Papers:**
  - `t6-donnybrook/donnybrook.{pdf,txt}`: Donnybrook (SIGCOMM 2008)
  - `t6-msr-p2p/p2p.{pdf,txt}`: Miller and Crowcroft, NetGames 2010
  - `t6-msr-tr809/tr809.{pdf,txt}`: Miller thesis, UCAM-CL-TR-809
  - `t6-alphaevolve/ae.{pdf,txt}`: AlphaEvolve arXiv 2506.13131v1
  - `t6-arx-liang/`: Liang et al., arXiv 0807.2328
  - `t6-arx-abs/`: abstract pages for arXiv 2601.10998, 2007.15373, 2512.02358, 1505.01864
- **Graveyard sources:**
  - `t6-reddwarf-files/`: Darkstar/RedDwarf README, CHANGELOG, affinity package docs
  - `t6-readmes/`, `t6-readmes2/`: SpatialOS, Hadean, channeld, Worlds Adrift Reborn and other READMEs
  - `t6-spatialgdk/repo/`: SpatialOS Unreal GDK (branch 0.11.0, blob-less clone)
  - `t6-scmesh/repo/`: community Star Citizen server-meshing wiki (commit 6514c1e)
- **In-house documentation:**
  - `t6-unity-netcode/repo/Documentation~/`: Unity Netcode 7.2.0 docs (needle-mirror)
  - `t6-gamelift/repo/`: archived GameLift developer guide (read at commit 7e08295)
  - `t6-roblox/`: Roblox creator-docs matchmaking pages
  - `t6-agones/sched.md`: Agones scheduling docs
  - `t6-playfab/dynamic-standby.md`: PlayFab docs

---

## 11. Claim index (ids match the structured result)

| id | status | claim (short) | source |
|---|---|---|---|
| T6-01 | VERIFIED | arXiv has only 15 results for "game server" in any field; one is about MMO server load (AFLL) | arxiv.org/search |
| T6-02 | GUESS | No public LLM-program-search work on game-server problems | sections 1–2 (absence) |
| T6-03 | VERIFIED | GitHub code search: "EVOLVE-BLOCK-START" with netcode / "game server" / "area of interest" gives 0 results | api.github.com search |
| T6-04 | VERIFIED (metadata) | GA, ant-colony and GRASP partitioning of virtual worlds, 2003–2009 (Morillo et al.) | Crossref / S2 |
| T6-05 | VERIFIED | GA for MMO cloud server allocation (TOMM 2021) | Crossref abstract |
| T6-06 | VERIFIED | GA for fog game-server placement (GLOBECOM 2021) | S2 abstract |
| T6-07 | VERIFIED | Deep RL game-server load balancing (2020) | S2 abstract |
| T6-08 | VERIFIED | RL / neural combinatorial-optimisation matchmaking with "online deployments on two games" (EnMatch); GloMatch; MGMatch | S2 abstracts |
| T6-09 | VERIFIED | EA: EOMM, and EA's AI platform including matchmaking | arXiv / S2 abstracts |
| T6-10 | VERIFIED | AFLL learned throttling: −48.3% CPU in controlled tests with 1,000 players | arXiv abs |
| T6-11 | VERIFIED (metadata) | Neural load prediction for MMO provisioning (2008–2011) | Crossref / S2 |
| T6-12 | UNVERIFIED | Kuro Games used AlphaEvolve on server workloads (from T1) | T1 notes |
| T6-13 | VERIFIED | Donnybrook attention weights were hand-tuned in "several hours" | Donnybrook PDF |
| T6-14 | SNIPPET | Web search found no AlphaEvolve/OpenEvolve/FunSearch result for game servers | WebSearch |
| T6-20 | VERIFIED | RedDwarf: "Multi-node operation is currently not scalable ... may reduce performance" | GitHub README |
| T6-21 | VERIFIED | RedDwarf 0.10.0 added label-propagation affinity groups for load balancing | GitHub CHANGELOG and package docs |
| T6-22 | VERIFIED | Darkstar was "a Sun Labs research project" | thesis abstract (S2) |
| T6-23 | VERIFIED | Waldo, CACM 2008: "Everything you know is wrong" | Crossref abstract |
| T6-24 | UNVERIFIED | Support for Darkstar stopped after Oracle bought Sun; community fork became RedDwarf | wiki copy on GitHub |
| T6-25 | VERIFIED | SpatialOS Unreal GDK was an engine fork; legacy platform frozen; migrate to zeuz before 30 Dec 2021 | GitHub README |
| T6-26 | VERIFIED | Final public GDK (0.11.0, Sept 2020) load-balanced with a static Rows × Cols grid | GitHub source |
| T6-27 | VERIFIED | SpatialOS Online Services deprecated, "no longer being supported in any way" | GitHub README |
| T6-28 | VERIFIED | Worlds Adrift shut down; fans re-implemented the SpatialOS SDK to revive it | GitHub README |
| T6-29 | UNVERIFIED | SpatialOS game Lazarus shut down partly over license and server costs | T4 snippet |
| T6-30 | VERIFIED | Hadean's public Aether repos stop around 2018–2019 | GitHub metadata and README |
| T6-31 | UNVERIFIED | Hadean pivoted to defence and enterprise simulation | memory |
| T6-32 | UNVERIFIED | Star Citizen: static meshing in 4.0; quasi-dynamic instancing in 4.10; dynamic load balancer unknown | community wiki |
| T6-33 | UNVERIFIED | CIG quote on a "smarter way to divide the game world into regions" | community wiki |
| T6-34 | VERIFIED | Donnybrook's goal was cheaper hosting for small publishers | Donnybrook PDF |
| T6-35 | VERIFIED | Colyseus is still quadratic in battles (per Donnybrook) | Donnybrook PDF |
| T6-36 | VERIFIED | Donnybrook left cheating and Internet deployment as future work; excluded strict-NAT players | Donnybrook PDF |
| T6-37 | GUESS | No record that Donnybrook's Internet deployment happened | absence |
| T6-38 | VERIFIED | P2P techniques: "few have been adopted"; pure P2P latency more than 10× client-server | Miller and Crowcroft 2010 |
| T6-39 | VERIFIED | Miller thesis: research not adopted because of evaluation realism, deployability, security | UCAM-CL-TR-809 |
| T6-40 | VERIFIED | P2P MMO survey: vulnerable to churn and cheating; harder to keep control | Crossref abstract |
| T6-41 | VERIFIED | Kiwano: deployment "souvent évité" (often avoided); evaluated by simulating tens of thousands of avatars | thesis abstract (S2) |
| T6-42 | VERIFIED | OneSim: "unlimited number of users ... in one sim" | S2 abstract |
| T6-43 | GUESS | No production adoption of Kiwano found | absence |
| T6-44 | VERIFIED | channeld: dynamic load balancing "(under development)" | GitHub README |
| T6-45 | VERIFIED | Earlier virtual-world research used random-waypoint or small traces | arXiv 0807.2328 |
| T6-50 | VERIFIED | Roblox: weighted-sum server scoring, Friends 15, others ≤ 3 | GitHub creator-docs |
| T6-51 | SNIPPET | Roblox has an ML matchmaking team (retrieval, ranking, assignment) | job adverts |
| T6-52 | SNIPPET | Roblox matchmaking orchestrates engine instances across edge data centres | about.roblox.com |
| T6-53 | SNIPPET | Unreal Iris: pluggable prioritizers, sphere prioritizer | dev.epicgames.com |
| T6-54 | VERIFIED | Unity Netcode 7.2: fixed bandwidth target, priority queue, custom importance function | package docs mirror |
| T6-55 | VERIFIED | GameLift FleetIQ: ordered criteria (latency, cost, ...); avoids high-interruption Spot | archived awsdocs |
| T6-56 | SNIPPET | Current FleetIQ predicts Spot viability from recent availability and interruption rate | docs.aws.amazon.com |
| T6-57 | VERIFIED | Agones defaults to "Packed" allocation | GitHub docs |
| T6-58 | VERIFIED | PlayFab Dynamic Standby uses three threshold rules (×1.5, ×3, ×4) | GitHub docs |
| T6-59 | VERIFIED | AlphaEvolve in Borg: correct by construction; simulator; unseen test set; 0.7%; chosen over RL | arXiv 2506.13131 |
| T6-70 | GUESS | Under-explored in the narrow sense, not the broad sense; absence is partly a demand signal | synthesis |
| T6-71 | GUESS | The graveyard did not die from weak heuristics | synthesis |
| T6-72 | GUESS | Big companies own the closed decision points; outsiders can contribute via open plug-in points | synthesis |
| T6-73 | GUESS | Adoption checklist (section 6) | synthesis |
