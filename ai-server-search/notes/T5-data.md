# T5: Can a trustworthy automatic evaluator be built? Data, benchmarks and testbeds

Research notes for Miles (Sumo Brothers, Godot, one Hetzner Linux box). Written 2026-11-08.

Question: AI-driven algorithm search (AlphaEvolve, FunSearch, OpenEvolve, ShinkaEvolve and similar) only works if a program can score each candidate quickly and faithfully. For MMO servers, is there public data, a benchmark, or an open-source server that such a scorer could be built on? For each problem type (interest management, load balancing / partitioning, session packing), could one person build a realistic evaluator in a few weeks, and what is missing?

Every claim carries a label:
- **VERIFIED**: read in the primary source; a short exact quote is given (in the claims table at the end).
- **VERIFIED_FIG**: read off a chart or table image in the primary source (approximate).
- **SNIPPET**: seen only in a search-engine excerpt because the host is blocked in this session (URL given).
- **UNVERIFIED**: secondary source only, or from memory.
- **WRONG**: a common claim that the primary source contradicts.
- **GUESS**: my own estimate, measurement or judgement (arithmetic or method shown).

Downloads are under `/tmp/claude-0/-home-user-OogaPL/32372848-6606-541b-85af-54d683c4e916/scratchpad/dl/t5-*`, one folder per source. Repository activity was measured with treeless clones limited to commits since 2025-11-08 (`git clone --bare --filter=tree:0 --shallow-since=2025-11-08`), so "commits in the last year" means commits on the default branch between 2025-11-08 and 2026-11-08. "Authors" means distinct author e-mail addresses in that window. PDFs were converted with `pdftotext -layout`.

Related notes from the same workflow, not repeated here: T1 (AI search methods, evaluator gaming, FunSearch bin-packing overfitting) at `.../scratchpad/ai-search/T1-ai-search.md`; T3 (network and replication, Dyconits, Boulanger, Donnybrook) at `.../scratchpad/ai-search/T3-network.md`.

---

## 0. Bottom line in plain language

1. **A fast, trustworthy scorer can be built, but mostly from data Miles records himself.** There is no public, current, MMO-sized dataset of fine-grained player positions that I could confirm is downloadable.
   - The famous MMO traces are old, small, coarse, or not public:
     - The World of Warcraft battleground traces (2009) cover 13 battles with about 35 players each and were never published as far as I can tell.
     - WoWAH (2011) is big (91,065 avatars over 3 years), but it only says which *zone* each avatar was in, sampled every 10 minutes.
     - The Second Life traces (2008) were "plan to share" or "publicly available" at the time; I could not find a working download.
   - The best public, human-made position data today comes from other genres:
     - **ESTA**: 1,558 professional Counter-Strike matches with positions at 2 Hz. It is on GitHub under CC BY-SA 4.0, but has only 10 players per match.
     - **awpy**: an open-source parser for new Counter-Strike 2 demos, plus a line-of-sight checker built from the map geometry.
     - **Space Station 14** (an open-source multiplayer game): its servers record full-state replays of every round, with up to about 80 players. The forum says these replays are public (SNIPPET, not checked).
2. **Every open benchmark uses scripted bots that move randomly, and the benchmark authors say so.**
   - Yardstick bots walk to random points in a box. Meterstick bots "move randomly in a 32-by-32 area". Veloren's bot client stands still. OpenSimulator's bot tool "randomly jump/walk around".
   - Meterstick and Servo state that "no player-behavior models" exist for Minecraft-like games.
   - Studies that compared real traces with generated movement found "significant differences". Miller's thesis concludes that the movement assumptions behind earlier work "do not match WoW's workload".
3. **The realistic testbeds that exist are good targets for small, swappable policies.**
   - **AzerothCore** (a World of Warcraft server emulator) has real servers with more than 1,000 players (SNIPPET). It has a module that runs "thousands of bots" that quest like players. It also contains a hand-made table of visibility constants whose comment invites people to "change existing ones".
   - **Minestom** (a Minecraft server library) exposes chunk-to-thread assignment as a one-method interface.
   - **Space Station 14** has about 4,000 lines of interest-management code, set by hand-chosen budgets.
   - **Veloren** (a Rust voxel RPG) has a small region-subscription system and peaked at 277 players.
4. **The main risks are known and well documented.**
   - Replaying recorded traces ignores the fact that players react to what the algorithm shows them (CausalSim).
   - Algorithms that won in simulators lost in a real randomized trial (Puffer).
   - Timing on cloud machines is noisy (Meterstick: the cloud raised the spread of results by up to 15 times).
   - When the bot machines are overloaded, the server looks healthy exactly when the test has stopped measuring anything (the Yardstick documentation).
   - Nobody has yet tied a server metric to what players actually feel (Meterstick says this is "not currently understood"). Donnybrook needed an 88-person user study.
   - T1 covers the AI search gaming the scorer.
5. **Verdict for each problem type (GUESS, details in section 5):**
   - **Interest management / update selection: yes**, a few weeks for one person, with an offline replay scorer that runs in seconds.
   - **Session / match packing: yes**, about 1–2 weeks of work, but the data has to come from Miles's own match logs. It also has the smallest payoff on one box.
   - **Load balancing / world partitioning:** a rough scorer is possible in a few weeks; a *trustworthy* one is not. No public trace has both MMO-scale player counts and fine positions, and the cost model (CPU against crowd density, migration cost) has to be measured on a real server. That takes calibration experiments that are slow and noisy.

---

## 1. Public traces of real player behaviour

### 1.1 Summary table

| Dataset | What it contains | Players | Sampling | Still downloadable? | Licence | Size | Status |
|---|---|---|---|---|---|---|---|
| WoW Arathi Basin battlegrounds (Miller and Crowcroft, NetGames 2009; Miller thesis 2011) | 3-D position + facing for every avatar near two observer clients, parsed from network traffic | 13 battles retained, average 35 unique participants per battle, 392 unique avatars, "more than sixty hours" of avatar movement | event-driven (as sent by the WoW server, only within about 250 yards) | No public release found in either paper or the thesis | none stated | not stated | VERIFIED (contents); GUESS (not public) |
| WoWAH, World of Warcraft Avatar History (Lee, Chen, Cheng, Lei, MMSys 2011) | per avatar: query time, avatar ID, guild, level, race, class, **zone** (no coordinates) | 91,065 avatars, 1,107 days (Jan 2006 – Jan 2009), one realm | every 10 minutes (SNIPPET) | original host does not resolve here; WPI mirror blocked here; Kaggle mirror (2008 subset) blocked here | citation requested (SNIPPET); otherwise unclear | 577,334,422-byte RAR (SNIPPET); about 3.5 GB as CSV (VERIFIED, parser comment) | VERIFIED (abstract, format); SNIPPET (mirror, interval) |
| Second Life, La and Michiardi (WOSN 2008, arXiv 0804.1928) | 2-D/3-D positions of all users on 3 lands | 2,656 / 3,347 / 1,568 unique users; 65 / 34 / 13 concurrent on average | every 10 s | the paper says "publicly available"; no working link found | none stated | not stated | VERIFIED (paper); availability unknown |
| Second Life, Liang, Tay, Neo, Ooi, Motani (arXiv 0807.2328) | identity, action, position, view direction | 84,208 avatars, 22 regions, two months, 62 million records | every 10 s | "plan to share"; no link found | none stated | not stated | VERIFIED (paper); availability unknown |
| Minecraft multiplayer | no public dataset found; HeapCraft collected 3,451 player-hours from 908 players on 43 servers | – | – | not found | – | – | SNIPPET (HeapCraft); VERIFIED that Meterstick and Servo say no behaviour models exist |
| MineRL (2019) | single-player human demonstrations, "over 60 million" state-action pairs | single player | per frame | yes (I did not check the host) | – | – | VERIFIED (abstract); not useful for multiplayer interest management (GUESS) |
| **ESTA** (Xenopoulos and Silva, arXiv 2209.09861) | per player per frame: x, y, z, velocity, view angles, health, weapon, "spotters" and more; plus kills, damages, grenades | 10 per match; 1,558 pro matches, 41,782 rounds, 7.9 M frames | **2 Hz** (64 ticks at 128 tick ≈ 0.50 s) | **yes, GitHub** `pnxenopoulos/esta` (README updated 2025-02-22) | **CC BY-SA 4.0** | 3.9 GB compressed; 25–75 MB per match decompressed | VERIFIED |
| Counter-Strike 2 demos via **awpy** (MIT) | full-tick parsing of CS2 demos + `VisibilityChecker` (line-of-sight from map collision meshes) | 10 per match | demo tick rate | parser on GitHub/PyPI; the demos themselves come from HLTV (not checked here) | MIT (parser) | – | VERIFIED (parser, visibility docs) |
| EgoCS-400K (arXiv 2606.18180, 2026) | player states, view directions, movements, inputs, events + rendered first-person video | >1,000 matches, 40,000 rounds, 13 maps, 10 viewpoints | – | host not checked | – | 10,000 h of video | VERIFIED (abstract) |
| **Space Station 14 Wizard's Den replays** | full server state per tick (the server's replay recorder runs the PVS with culling turned off) | up to `soft_max_players = 80` per server | every server tick | the forum says every round is recorded and downloadable (SNIPPET; the forum and CDN are blocked here) | the server privacy policy applies; licence of replay data unknown | up to 512 MB compressed per replay by default | VERIFIED (code, config); SNIPPET (public availability) |
| NCSOFT Lineage W trajectories (arXiv 2501.10461) | coordinates and timestamps | 778,656 training samples over 8 days; 26,136 trajectories over 7 days | **one minute** | no release statement found | – | – | VERIFIED (contents); GUESS (not public) |
| Game Trace Archive (Guo and Iosup, NetGames 2012) | 9 traces of games and "online meta-gaming networks" | ">8 million real players" | varies | old host does not resolve here; status unknown | – | – | SNIPPET |
| EVE Online ESI `system_jumps` | ship jumps per solar system in the last hour | whole game | hourly, live only | live API (not tested); no historical archive found | – | – | SNIPPET |

### 1.2 World of Warcraft battleground traces (Miller and Crowcroft)

- Primary sources (all reachable via microsoft.com):
  - NetGames 2009 paper: [johnmil-Netgames09.pdf](https://www.microsoft.com/en-us/research/wp-content/uploads/2009/11/johnmil-Netgames09.pdf).
  - Journal version "Group Movement in World of Warcraft Battlegrounds": [WowGroupMovement.pdf](https://www.microsoft.com/en-us/research/wp-content/uploads/2010/11/WowGroupMovement.pdf).
  - PhD thesis UCAM-CL-TR-809 (2011): [UCAM-CL-TR-809.pdf](https://www.microsoft.com/en-us/research/wp-content/uploads/2011/10/UCAM-CL-TR-809.pdf).
- **How it was captured (VERIFIED, thesis):** two observer characters stood on high ground and recorded the server's movement messages. "Updates are absolute positions, consisting of a client identifier, three-dimensional Cartesian position, facing information, and additional information I do not decode." The server only sends avatars "within avatar visual range – approximately 250 yards", so the trace is incomplete: movement was recorded for "73% of their participation time" (NetGames paper).
- **Size (VERIFIED):**
  - "I captured approximately 20 traces of Arathi Basin battles, and retained 13".
  - "The average battle had 35 unique participants".
  - "more than sixty hours of individual avatar movements, with 392 unique avatars in 456 avatar sessions".
  - Battles lasted 266 to 1,423 s (Table 1).
  - The thesis also captured questing, capital-city, dungeon, raid and PvP traffic (Table 7: 1,339 / 205 / 1,137 / 703 / 311 minutes).
- **Findings that matter for evaluators (VERIFIED):**
  - "a waypoint model is not sufficient to describe most avatar movement".
  - "5% of visited territory accounting for 30% of all time spent in a typical battleground".
  - Most travel between objectives is done alone, not in groups.
- **Availability:** neither paper nor the thesis gives a download location. The NetGames paper only says "We would also be interested in the results of others evaluating existing research with this real-world data." I found no public copy (GUESS: not public; asking the author is the only route).

### 1.3 WoWAH (World of Warcraft Avatar History)

- **Abstract (VERIFIED via OpenAlex, DOI [10.1145/1943552.1943569](https://doi.org/10.1145/1943552.1943569)):**
  - "comprises the records of 91,065 avatars. The data includes the avatars' game play times and a number of attributes, such as their race, profession, current level, and in-game locations, during a 1,107-day period between Jan. 2006 and Jan. 2009."
  - "It is available for free download at http://mmnet.iis.sinica.edu.tw/dl/wowah/."
- **Record format (VERIFIED, [myles-oneill/WoWAH-parser](https://github.com/myles-oneill/WoWAH-parser) comments):**
  - Fields: "dummy, query time, query sequence number, avatar ID, guild, level, race, class, zone, dummy, dummy".
  - Example row: `"0, 01/10/09 00:04:10, 5,78122,342, 80, Orc, Hunter, The Storm Peaks, HUNTER, 0"`.
  - So "location" means the zone name only; there are **no coordinates**.
  - The parser reads "about 130,000" small files into one CSV of about 3.5 GB.
- **Sampling and mirrors (SNIPPET):**
  - The [SIGMM user-traces page](https://sigmm.org/Resources/user-traces) describes "information of avatar in each ten-minute interval".
  - The [WPI MMSys dataset page](https://web.cs.wpi.edu/~claypool/mmsys-dataset/2011/wow/) lists `wowah.rar` at 577,334,422 bytes.
  - A [calmcode page](https://calmcode.io/datasets/wow-lite) reports "667,032 sessions".
  - The original host `mmnet.iis.sinica.edu.tw` does not resolve in this session; `web.cs.wpi.edu` and `www.kaggle.com` are blocked.
- **Use (GUESS):** good for session arrivals and durations, daily cycles, and zone-level load (which zones need how much capacity when). Useless for interest management, because there are no positions.

### 1.4 Second Life traces

- **La and Michiardi, arXiv [0804.1928](https://arxiv.org/abs/0804.1928) (VERIFIED):**
  - Their crawler sampled positions with "τ = 10 sec" over "24 hours traces".
  - The three lands: "Isle of View had 2656 unique visitors with an average of 65 concurrent users, Dance Island had 3347 unique users and 34 concurrent users in average and Apfel Land had 1568 users and 13 concurrent users in average."
  - The paper says "Our measurements are publicly available" but gives no URL. eurecom.fr is blocked here, and I found no CRAWDAD entry.
- **Liang et al., arXiv [0807.2328](https://arxiv.org/abs/0807.2328) (VERIFIED):**
  - "mobility traces of 84,208 avatars spanning 22 regions over two months ... giving a total of 62 million records".
  - Sampling: "We log information about all detected avatars at ten second intervals".
  - Sharing: traces "which we plan to share with the research community". No release found.
  - Caveat (VERIFIED): much activity is "camping" (sitting still to earn money), and "About 8% of our records have unknown avatar positions".
- Second Life movement is social-world movement, not game movement (GUESS). An open-source Second Life-compatible server exists (OpenSimulator, section 3), so these traces could in principle be replayed into it.

### 1.5 Minecraft

- No public multiplayer movement trace found. Two Opencraft-group papers state the gap directly (VERIFIED):
  - Meterstick: "no player-behavior models exists for MLGs".
  - Servo: "We use a randomized behavior because no MVE-specific player behavior models currently exist."
- HeapCraft's Epilog plugin logged movement and other events: "over 3451 player-hours of gameplay from 908 players over 43 different servers" (SNIPPET, search excerpt of the [AIIDE 2015 paper](https://ojs.aaai.org/index.php/AIIDE/article/view/12807)). No dataset download found.
- MineRL (arXiv [1907.13440](https://arxiv.org/abs/1907.13440)) has "over 60 million automatically annotated state-action pairs" of human Minecraft play (VERIFIED), but each episode is a single player.

### 1.6 Newer and adjacent datasets

- **ESTA** (VERIFIED, [GitHub](https://github.com/pnxenopoulos/esta), [arXiv 2209.09861](https://arxiv.org/abs/2209.09861)):
  - "contains frames ("game snapshots") parsed at 2 Hz".
  - 1,558 demos; 3.9 GB; CC BY-SA 4.0.
  - "ESTA data is only compatible with Awpy version 1.x".
  - I downloaded one match (`data/lan/0013db25-...json.xz`, 3.2 MB) and confirmed per-player `x, y, z, velocityX/Y/Z, viewX, viewY, spotters, ping` and more, at 64-tick spacing with `tickRate = 127`.
- **My measurement on that one match** (GUESS, own analysis; method: 128-unit grid cells, alive players only, warm-up rounds excluded; script output in this session):
  - de_overpass: 6,097 frames, 46,288 player-samples, 684 visited cells.
  - The busiest 5 % of visited cells hold **19.6 %** of player-time; the busiest 20 % hold **51.7 %**. Under uniform occupancy these would be about 5 % and 20 %.
  - On average 7.6 players are alive per frame.
  - The `spotters` field was non-empty in only 23 of 6,040 player-frames (rounds 4–6). So it is **not** a usable "who can see whom" ground truth.
- **awpy** (VERIFIED, [pnxenopoulos/awpy](https://github.com/pnxenopoulos/awpy), MIT, 10 commits in the last year, last 2026-10-27):
  - "Awpy parses Counter-Strike 2 demo files in Rust."
  - Its `VisibilityChecker` uses "map collision geometry to check whether a straight segment is blocked". Caveat: "It does not model smokes, moving doors, or bullet penetration."
  - Together these give a **line-of-sight ground truth** for scoring interest management on human CS2 traces.
- **EgoCS-400K** (VERIFIED, [arXiv 2606.18180](https://arxiv.org/abs/2606.18180)): "built from public professional CS and CS2 match demos", ">1,000 matches and 40,000 rounds, covering 13 maps and 10 player viewpoints per round".
- **DECOY** (VERIFIED, [arXiv 2509.06355](https://arxiv.org/abs/2509.06355)): a CS:GO simulator trained on tournament data; "replays generated from human data in DECOY closely match those observed in the original game". This is an example of a learned behaviour model validated against human replays.
- **Space Station 14 replays** (VERIFIED in code, SNIPPET for public access):
  - In RobustToolbox's `Robust.Server/Replays/ReplayRecordingManager.cs`, the recorder uses `new PvsSession(...) { DisableCulling = true }`, so recordings hold the full, unculled state.
  - `replay.server_recording_enabled` defaults to `true`; `replay.max_compressed_size` defaults to `1024L * 512` KB.
  - The Wizard's Den base config sets `soft_max_players = 80`.
  - The [forum announcement](https://forum.spacestation14.com/t/server-replays-are-now-available-on-wizards-den/5418) says every round "is automatically recorded and made available for viewing" (SNIPPET; host blocked).
  - **This is the largest human-driven, full-state, per-tick game trace source I found** (GUESS), but parsing needs the C# engine.
- **NCSOFT Lineage W** (VERIFIED, [arXiv 2501.10461](https://arxiv.org/abs/2501.10461)): "coordinates and timestamp logs sampled at one-minute intervals". Commercial MMOs log positions, but coarsely and privately (no release statement; GUESS not public).
- **Game Trace Archive** (SNIPPET, abstract excerpt): "9 traces of both games and OMGNs, covering more than 8 million real players". The host `gta.st.ewi.tudelft.nl` does not resolve here; Semantic Scholar shows the paper (DOI 10.1109/NetGames.2012.6404027, 52 citations) but no abstract.

---

## 2. Benchmarks, harnesses and bot load generators

| Tool | What it does | Bots / workload | Licence | Maintenance (last year) | Status |
|---|---|---|---|---|---|
| **Yardstick** ([atlarge-research/yardstick](https://github.com/atlarge-research/yardstick); ICPE 2019, DOI 10.1145/3297663.3310307) | deploys a Minecraft server, runs emulated players, collects tick time and system metrics; local, Ubicloud cloud, or DAS-6 cluster modes | mineflayer bots; `walkaround` = "walks, on foot, between random points inside a box"; `worldgen` = teleports | **no LICENSE file** at HEAD | 71 commits, 1 author e-mail, last 2026-09-30 | VERIFIED |
| **Meterstick** ([JerritEic/Meterstick](https://github.com/JerritEic/Meterstick); arXiv [2112.06963](https://arxiv.org/abs/2112.06963)) | measures performance *variability* (Instability Ratio, ISR) of Minecraft-like servers on AWS, Azure and DAS-5 | Yardstick bots: "25 players which move randomly in a 32-by-32 area"; plus farm, TNT and lag-machine worlds | GPL-3.0 | **no commits since 2022-02-17** | VERIFIED |
| **Opencraft** + **opencraft-tutorial** ([opencraft](https://github.com/atlarge-research/opencraft), [tutorial](https://github.com/atlarge-research/opencraft-tutorial)) | research Minecraft-like server with the Dyconits bounded-inconsistency library (see T3) | tutorial: "connect 50 players, and let these players move around on a flat plane"; "Expected runtime: 5 minutes" | MIT (both) | Opencraft `dev`: 40 commits, last 2026-11-01; tutorial: last commit 2023-11-08 | VERIFIED |
| **mineflayer** ([PrismarineJS/mineflayer](https://github.com/PrismarineJS/mineflayer)) | scriptable Minecraft bot client (used by Yardstick) | anything you script | MIT | 111 commits, 20 authors, last 2026-10-25 | VERIFIED |
| **Veloren bot client** (`client/src/bin/bot/` in [veloren](https://gitlab.com/veloren/veloren)) | registers, logs in and joins many bot accounts | each bot ticks with `comp::ControllerInputs::default()`, i.e. **no input: idle bots** | GPL-3.0 | (part of Veloren) | VERIFIED |
| **OpenSimulator pCampBot** (`OpenSim/Tools/pCampBot`) | stress-tests an OpenSimulator region | "clients that log in, randomly jump/walk around" | BSD-style | (part of OpenSimulator) | VERIFIED |
| **mod-playerbots** for AzerothCore ([mod-playerbots](https://github.com/mod-playerbots/mod-playerbots)) | server-side bots that play WoW 3.3.5 | "Random bots that wander through the world, complete quests, and otherwise behave like players"; claims "Excellent performance, even when running thousands of bots" | GPL-2.0 | 617 commits, 86 authors, last 2026-11-05 | VERIFIED (README; the performance line is the project's own claim) |
| Community AOI micro-benchmarks (e.g. [bhhbazinga/AOI](https://github.com/bhhbazinga/AOI)) | compare cross-link, quadtree and tower (grid) AOI structures | "We simulate N random moves of N units in a 1024*1024 map"; positions from `rand() % kMapWidth` | no licence file | last commit 2019-12-11 | VERIFIED |

Notes:
- **Yardstick 2019 (VERIFIED abstract, Semantic Scholar):** "Our findings indicate the scalability limits of these servers, that Minecraft-like services are poorly parallelized".
- **Yardstick 2026 has a built-in trustworthiness check (VERIFIED, README):** "If the machines running the bots run out of CPU or memory, the players send fewer actions and load fewer chunks, so the server does *less* work — and the run reports a healthy-looking server exactly when it has stopped measuring anything." It also isolates the system under test: "The game server always gets a machine to itself, so the emulated players never compete with the thing being measured for CPU."
- **Yardstick has no licence (VERIFIED: `git ls-files | grep -i licen` returns nothing; `pyproject.toml` has no licence field).** Strictly, that means it is not open source. Reuse beyond reading would need the authors' permission (GUESS, not legal advice).
- **Meterstick limitations (VERIFIED):** "It is not currently understood how ISR directly relates to player-perceived quality of experience". There is also "no publicly available analysis of environment-based workload prevalence".
- **No cross-game MMO benchmark suite** was found (GUESS, absence of evidence after the searches listed in section 8). Agones fleet tooling is covered in T4.

---

## 3. Open-source servers as testbeds

Activity: commits on the default branch between 2025-11-08 and 2026-11-08 (VERIFIED by my clones). Licences: read from each repository's licence files (VERIFIED).

| Server | Licence | Commits / authors in last year | Last commit | Real concurrency evidence | Swappable interest-management or load-balancing code |
|---|---|---|---|---|---|
| **AzerothCore** (WoW 3.3.5 emulator, C++) | GPL-2.0 | 3,652 / 162 | 2026-11-07 | ChromieCraft "new peak of over 1300 players online" in 2025 (SNIPPET, operator's own post); "at least 15,000 concurrent players" in adopters' stress tests (SNIPPET, vendor claim) | **Yes, small:** `DynamicVisibility.h` is a 7 × 5 table of hand-set (notify delay ms, AI notify delay ms, required move distance²) by player count band and map type, with the comment "feel free to add more intervals, change existing ones or move to conf file". Visibility range `Visibility.Distance.Continents = 100`; `MapUpdate.Threads = 1` by default. Bots: mod-playerbots. |
| TrinityCore (WoW emulator, C++) | GPL-2.0 | 1,633 / 84 | 2026-11-08 | not checked | similar grid and visibility design (UNVERIFIED) |
| **Space Station 14** + RobustToolbox (C#) | content MIT; engine MIT for code after 2019-03-13 (earlier code GPL-3) | 3,510 / 424 (content); 736 / 108 (engine) | 2026-11-06 | `soft_max_players = 80` on Wizard's Den (VERIFIED); about 3,500 concurrent on Steam (SNIPPET, tracker sites) | **Yes:** PVS system, about 4,035 lines in `Robust.Server/GameStates/`, with hand-set budgets `net.pvs_budget` 50, `net.pvs_enter_budget` 200, `net.pvs_exit_budget` 75 and `net.pvs_range` 25. The budget's own comment: "If ever entity spawning isn't hot garbage, this can be increased." Full-state replays for offline scoring. |
| **Veloren** (Rust) | GPL-3.0 | 934 / 57 (GitLab) | 2026-10-30 | record "277 players online at peak" at the 0.13 release party (2022, VERIFIED from the website source); no higher number in 2023–2026 posts | **Yes:** region subscription (`server/src/sys/subscription.rs` 283 lines; `common/src/region.rs` 325 lines; `REGION_SIZE = 1 << 9`) and entity sync (594 lines). Bot client exists but is idle. |
| **Minestom** (Java library) | Apache-2.0 | 281 / 47 | 2026-10-17 | not checked | **Yes:** `ThreadProvider<T>` is a single-method interface, `int findThread(T partition)`, plus a refresh type (NEVER/ALWAYS); the default `counter()` is round-robin. This is an evolvable load-balancing hook. |
| Paper (Minecraft server) | GPL-3.0 (inherited) | 530 / 140 | 2026-11-06 | Hypixel at "216,762 online players" stitches "thousands of (independent) MLG instances" (VERIFIED, quoted in Meterstick) | patches, not plug-ins (GUESS) |
| **Folia** (Paper fork, regionised threads) | GPL-3.0 (patches) | 66 / 5 | 2026-10-29 | not checked | regionizer lives inside a 1,090,131-byte patch (`0001-Region-Threading-Base.patch`), so swapping means maintaining a fork (GUESS); has a "Region profiler" patch for per-region timing (VERIFIED file name) |
| Pumpkin (Rust Minecraft server) | GPL-3.0 | 1,607 / 221 | 2026-11-08 | not checked | not checked |
| Valence (Rust Minecraft framework) | MIT | **5 / 3** | 2026-06-15 | – | dormant (GUESS from activity) |
| MultiPaper (one world across many servers) | GPL-3.0 | 0 | **2025-05-22** | – | "public beta", 1.20.1 Purpur fork (VERIFIED); stale |
| Cuberite (C++ Minecraft server) | LICENSE file present (type not checked) | 2 / 2 | 2025-12-16 | – | dormant (GUESS) |
| **Luanti** (formerly Minetest) | LGPL-2.1 | 849 / 194 | 2026-11-07 | not found (live server list only) | ranges are settings, not code: `active_object_send_range_blocks` 8, `active_block_range` 4, `max_block_send_distance` 13 |
| **Ryzom Core** | AGPL-3.0 | 1,671 / 14 (branch `core4`) | 2026-11-06 | not checked | separate position service `gpm_service` with `vision_delta_manager.cpp`, `cell.cpp`, `world_position_manager.cpp` (VERIFIED file names) |
| **OpenSimulator** (GitHub `opensim/opensim`) | BSD-style | 225 / 9 | 2026-11-07 | not checked | regions per simulator; Second Life-compatible, so Second Life traces could drive it (GUESS); bots via pCampBot |
| canary / forgottenserver (Tibia emulators) | GPL-2.0 | 316 / 40 and 40 / 12 | 2026-11-05 / 2026-09-29 | not checked | not checked |
| rAthena (Ragnarok Online emulator) | GPL-3.0 | 297 / 30 | 2026-10-06 | not checked | not checked |

Judgement (GUESS):
- **Best for "realistic players at scale":** AzerothCore. It has real 1,000+ player servers, thousands of questing bots, and a compact hand-tuned visibility table.
- **Best for "human traces with full state":** Space Station 14. It has replays of up to about 80 players and open interest-management code.
- **Best for clean plug-in experiments:** Minestom (`ThreadProvider`) and Veloren (region subscription). Both have small, isolated code, but no realistic bots.
- **Not useful as engines for Miles's own game:** none of these is Godot. They are research testbeds. The scorer that matters most for Miles is one built on traces from Sumo Brothers.

---

## 4. Pitfalls of simulation-based evaluation, and how earlier work validated

### 4.1 General networking

- **Floyd and Paxson, "Difficulties in simulating the Internet" (IEEE/ACM ToN 2001, DOI 10.1109/90.944338)** (SNIPPET): simulating the Internet is hard because of its "great heterogeneity and rapid change". They suggest "searching for invariants" and "judiciously exploring the simulation parameter space".
- **Kurkowski, Camp and Colagrosso, "MANET simulation studies: the incredibles" (2005)** (VERIFIED, Crossref abstract): "while the use of simulation has increased, the credibility of the simulation results has decreased".
- **Yoon, Liu and Noble, "Random waypoint considered harmful" (INFOCOM 2003)** (SNIPPET): "the average nodal speed consistently decreases over time"; some metrics "can drop by as much as 40%" over a 900-second simulation. Random movement models have hidden transients. Most game bots and AOI benchmarks above use exactly this kind of random waypoint or uniform random movement.
- **CausalSim (Alomar et al., arXiv [2201.01811](https://arxiv.org/abs/2201.01811))** (VERIFIED): "Current trace-driven simulators assume that the interventions being simulated (e.g., a new algorithm) would not affect the validity of the traces. However, real-world traces are often biased by the choices algorithms make during trace collection, and hence replaying traces under an intervention may lead to incorrect results."
  - For games (GUESS): replaying recorded movement through a new interest-management policy assumes players move the same way even though they now see different things.
- **Puffer (Yan et al., arXiv [1906.01113](https://arxiv.org/abs/1906.01113))** (VERIFIED): in a randomized trial with 56,000 users, "it is difficult for sophisticated or machine-learned control schemes to outperform a "simple" scheme (buffer-based control), notwithstanding good performance in network emulators or simulators." Their fix: learn "in situ on data from the real deployment environment". They also publish "an archive of traces and results each day", a model for open data.

### 4.2 Games and virtual worlds

- **Synthetic movement gives different answers (VERIFIED, Liang et al.):**
  - On FreeWar (400 players) and Orbius (28 players): "both observe significant differences in their results evaluated using the traces and using generated movements".
  - And: "Research, however, has shown that the evaluation results based on these simple models are significantly different from those based on actual traces."
- **Miller's thesis (VERIFIED):** "This dissertation shows previous DVE research evaluations are inconsistent with deployed DVE needs. Assumptions about avatar movement and proximity - fundamental scale factors - do not match WoW's workload". Also: frameworks "assume movement and arrival / departure properties of participants without any obvious experimental basis."
- **A nuance (see T3):** Boulanger et al. (NetGames 2006) found random-action bots can approximate humans if starting positions are chosen well. Real traces still differ in hotspots: 5 % of territory takes 30 % of time in WoW (VERIFIED, section 1.2), and 19.6 % of time sits in 5 % of cells in my CS:GO sample (GUESS).
- **No behaviour models for Minecraft-like games** (VERIFIED, Meterstick and Servo, section 1.5). Every Minecraft-like benchmark above uses random bots.
- **Measurement noise (VERIFIED, Meterstick):** "The choice of cloud causes a 1.39 up to 15.44 times increase in ISR IQR and a 1.09 up to 5.61 times increase tick time IQR." They used 50 iterations of the player workload to see the spread.
  - Hetzner's own docs (VERIFIED): shared-resource servers use "hypervisors ... to distribute usage rights of CPU resources among several virtual servers", while "Every dedicated resources instance has its own dedicated CPU resources".
  - So wall-clock timing on a shared vCPU is a poor fitness signal (GUESS). Count bytes and operations instead, or use dedicated cores and repeats.
- **Load generator saturation and co-location (VERIFIED, Yardstick README, section 2).**
- **No link from server metric to player experience (VERIFIED):**
  - Meterstick: ISR's relation to player experience "is not currently understood".
  - Donnybrook validated its bandwidth trade-off with "a total of 88 different participants". Its own stated limits: "it has only two human players, and it has only 32 players total."
- **Evaluator gaming and overfitting** by the AI search itself (the Sakana CUDA kernel exploits, the AlphaEvolve guide, FunSearch bin packing failing off-distribution) are covered with sources in T1. They apply fully here.

### 4.3 How earlier work validated (templates to copy)

| Work | Validation pattern | Status |
|---|---|---|
| Donnybrook (SIGCOMM 2008) | small human user study (88 people, 2 humans + bots per game, 32 players) → simulator driven by "the game workload generator ... based on the observed behavior of real Quake III players" → checked that simulated and user-study subscriber-set size distributions match → extrapolated to hundreds of players, because "it is not possible for us to obtain a real workload with hundreds of players" | VERIFIED |
| Puffer (NSDI 2020) | blinded randomized trial on real users; learned model trained in situ | VERIFIED |
| CausalSim (NSDI 2023) | small randomized trial → causal model removes trace bias → validated "with a real deployment" | VERIFIED |
| Miller (2009–2011) | real WoW traces as simulator workload instead of synthetic models | VERIFIED |
| Meterstick / Yardstick | real servers, real clouds, repeated iterations, but random bots | VERIFIED |
| AlphaEvolve for Borg (see T1) | historical-snapshot simulator → unseen recent workloads → fleet roll-out that "confirmed the simulator results" | see T1 |

---

## 5. Can one person build an evaluator with realistic data in a few weeks?

All estimates in this section are GUESS.

### 5.1 Interest management / update selection: yes (2–4 weeks)

- **Data, best first:**
  1. Miles's own Sumo Brothers server traces: per tick, every entity's position, velocity, state size and inputs. Example size: 8 players × 60 Hz × 32 B = 15,360 B/s, about 55 MB per hour uncompressed (GUESS arithmetic).
  2. CS2 demos parsed with awpy at full tick rate, plus `VisibilityChecker` for line-of-sight ground truth.
  3. ESTA (2 Hz, public, CC BY-SA).
  4. Space Station 14 replays, if they really are public (up to about 80 players, full state).
- **Evaluator:**
  - Offline replay: for each tick and each client, call the candidate `select(client, entities, budget)`; simulate what the client would display (last received state plus extrapolation); score bytes against error.
  - Deterministic, no wall-clock timing, about seconds per match.
  - Hold out maps and matches.
- **Plan:**
  - Week 1: trace format and recorder.
  - Week 2: replay harness, baselines (radius, grid, "send all") and metrics.
  - Week 3: held-out splits, anti-gaming (budget and recall enforced by the harness, time limits, sandbox).
  - Week 4: connect OpenEvolve or ShinkaEvolve; confirm the top few candidates in a headless Godot run with bots. Godot has `--headless` and `--fixed-fps <fps>`, which "disables real-time synchronization" (VERIFIED). Then a short human playtest.
- **Missing:**
  - Closed-loop effects (CausalSim).
  - An agreed perceptual error metric (Donnybrook needed humans).
  - MMO-scale public traces. Tiling several 10-player matches into one world can test CPU scaling but not real crowd interactions.

### 5.2 Load balancing / world partitioning: rough yes, trustworthy no (not in a few weeks)

- **Data:**
  - WoWAH gives zone-level presence every 10 minutes for 3 years (good for "which zones on which server"), if a mirror can be downloaded.
  - Fine-grained positions at MMO scale are not public.
  - Bots (playerbots, mineflayer) give load, but random movement misses hotspots and flash crowds.
- **Evaluator:** a simulator needs a **cost model**: CPU per entity, per interaction pair (it grows with crowd density), and per migration or hand-off. No public dataset gives this. It must be measured on the real server, for example with Folia's region profiler or Minestom per-thread tick times, using controlled bot crowds. Each such run takes minutes, and timings are noisy (Meterstick).
- **Effort:**
  - A toy simulator on WoWAH zones in about 1–2 weeks.
  - Calibration plus validation on a real multi-threaded server in another 3–6 weeks.
  - Even then, trust is limited by bot realism.
- **Missing:** a validated cost model, MMO-scale position traces, and migration-cost measurements. Relevance to Miles is also low: small matches on one box do not need world partitioning (GUESS; see T4).

### 5.3 Session / match packing: yes (1–2 weeks of work, but needs logged data)

- **Data:**
  - Miles's own match logs: start, end, player count, CPU and memory per match, measured on his Hetzner box.
  - WoWAH sessions can serve as a stress test for daily cycles.
  - No public match-level arrival and resource trace for an indie game was found.
- **Evaluator:** a discrete-event simulation of match arrivals onto servers with capacity limits and billing. It is fast (milliseconds per day simulated), deterministic, and easy to make tamper-proof.
- **Missing:**
  - Weeks of calendar time to collect logs.
  - Realistic bursts (launches, streamers).
  - Per-match resource profiles.
- **Payoff:** small at one box (see T1 for the Borg 0.7 % result and FunSearch heuristics failing off-distribution).

### 5.4 Throughput arithmetic: why the scorer must be offline (GUESS)

- **Live bot runs:**
  - Assume 1,000 candidate programs × 3 seeds × 5 minutes (the Opencraft tutorial run length) = 15,000 minutes = 250 hours, about 10.4 days on one machine.
  - Live runs also add noise.
- **Offline replay:** at 2 s per candidate-seed, the same search takes 1,000 × 3 × 2 s = 6,000 s, about 1.7 hours.
- **So use a cascade:**
  1. Offline replay for the search.
  2. Headless bot runs on dedicated cores for the top 10–20.
  3. A human playtest for the final 1–2.

---

## 6. Candidate evaluator-ready problems found in this topic

1. **Line-of-sight-aware interest set on human CS2 or ESTA traces.**
   - Metric: bytes per client-second at a fixed state size, subject to recall of at least 99.5 % of (client, entity, tick) triples that are within 2,000 units and line-of-sight visible (awpy), with staleness of at most 100 ms.
   - Public data; replay takes seconds.
   - Risk: only 10 players.
2. **Space Station 14 PVS budget and priority policy on Wizard's Den replays.**
   - Metric: bytes and entities sent per client-tick, plus pop-in and staleness events, at the same budgets.
   - Human data at about 80 players; heavy C# work; public availability unconfirmed.
3. **AzerothCore dynamic-visibility table with mod-playerbots load.**
   - Metric: p99 map-update ms at N bots, subject to a visibility-latency bound.
   - Slow and noisy. With only 105 numbers, plain Bayesian optimisation may be enough; it may not need LLM search (GUESS).
4. **Zone-to-server assignment on WoWAH.**
   - Metric: server-hours at a per-server player cap plus a migration penalty.
   - Offline and fast. Risk: a crude cost model and old data.
5. **Match packing from Miles's own logs.**
   - Metric: euro per month subject to zero rejected matches and a wait-time bound.
   - Easy and deterministic; tiny payoff at one box.

---

## 7. Open questions

- Are Wizard's Den replays still publicly downloadable, and under what terms? (Forum and CDN blocked here.)
- Can the Second Life traces (La and Michiardi; Liang et al.) still be obtained from the authors? Do they include enough of each region to replay into OpenSimulator?
- Is any WoWAH mirror still live (WPI, Academia Sinica, Kaggle)? What exactly is its licence?
- Would an AzerothCore server operator (for example ChromieCraft) share anonymised position logs? What would GDPR require?
- How well do mod-playerbots match human movement statistics (hotspots, group travel, session lengths)? No measurement found.
- Is there any study that links tick-time variability or update staleness to player-perceived quality for MMOs? Meterstick says this is future work.
- Did the Game Trace Archive data survive anywhere?

## 8. Searches that found nothing

I found no public MMO-scale fine-grained position dataset, no public Minecraft multiplayer movement trace, no maintained cross-game MMO benchmark suite, no open-source simulator for game-server fleet allocation, and no published LLM-driven search on game interest management or partitioning. All of these are GUESS, absence of evidence.

These searches came back empty for those items:
- Web searches for "dataset player trajectories multiplayer game positions", "Minecraft multiplayer player movement traces", "HeapCraft Epilog dataset", "open source simulator game server fleet allocation", and "realistic bot players load testing learned from traces".
- arXiv searches for "MMOG traces", "game server load testing" and "interest management multiplayer".

## 9. Blocked hosts in this session

web.cs.wpi.edu (WoWAH mirror), mmnet.iis.sinica.edu.tw (WoWAH original, DNS), homepage.iis.sinica.edu.tw, www.kaggle.com, qualinet.github.io, sigmm.org, typeset.io, www.cl.cam.ac.uk, eprints.whiterose.ac.uk, people.cs.rutgers.edu (HeapCraft), zenodo.org (Meterstick data), figshare.com, dataverse.harvard.edu, huggingface.co, data.4tu.nl, doi.org, dl.acm.org, ieeexplore.ieee.org, research.spec.org (Yardstick PDF), filelist.tudelft.nl, repository.tudelft.nl, gta.st.ewi.tudelft.nl (DNS), atlarge-research.com, www.eurecom.fr, sacheendra.github.io, export.arxiv.org, forum.spacestation14.com, wiki.spacestation14.com, docs.spacestation14.com, cdn.centcomm.spacestation14.com, central.spacestation14.io, opensimulator.org.

Also:
- The GitHub REST API (api.github.com) is not enabled for this session; git clone worked.
- The api.openalex.org daily budget was exhausted mid-session (shared IP).
- api.semanticscholar.org returned 429 errors intermittently.
- www.microsoft.com and gitlab.com were reachable.

---

## 10. Claims table

| ID | Status | Claim | Source | Exact quote |
|---|---|---|---|---|
| T5-01 | VERIFIED | WoW battleground traces: 13 battles retained, average 35 participants | https://www.microsoft.com/en-us/research/wp-content/uploads/2010/11/WowGroupMovement.pdf ; https://www.microsoft.com/en-us/research/wp-content/uploads/2009/11/johnmil-Netgames09.pdf | "We captured a few dozen traces of Arathi Basin battles, and retained 13" / "The average battle had 35 unique participants" |
| T5-02 | VERIFIED | Captured from server movement messages within about 250 yards; 73 % of participation time recorded | https://www.microsoft.com/en-us/research/wp-content/uploads/2011/10/UCAM-CL-TR-809.pdf | "Updates are absolute positions, consisting of a client identifier, three-dimensional Cartesian position, facing information" |
| T5-03 | VERIFIED | Trace size: 60+ h, 392 avatars, 456 sessions | thesis (above) | "more than sixty hours of individual avatar movements, with 392 unique avatars in 456 avatar sessions" |
| T5-04 | VERIFIED | Hotspots: 5 % of territory, 30 % of time; waypoint models insufficient | NetGames 2009 (above) | "5% of visited territory accounting for 30% of all time spent in a typical battleground" |
| T5-05 | GUESS | The WoW battleground traces are not publicly available (no release statement in paper, journal or thesis) | own reading of all three | – |
| T5-06 | VERIFIED | Miller: earlier DVE evaluations used movement assumptions that do not match WoW | thesis (above) | "This dissertation shows previous DVE research evaluations are inconsistent with deployed DVE needs." |
| T5-07 | VERIFIED | WoWAH: 91,065 avatars, 1,107 days, zone-level location; original download URL | https://api.openalex.org/works/doi:10.1145/1943552.1943569 | "comprises the records of 91,065 avatars" |
| T5-08 | VERIFIED | WoWAH records hold zone, not coordinates | https://github.com/myles-oneill/WoWAH-parser | "dummy, query time, query sequence number, avatar ID, guild, level, race, class, zone, dummy, dummy" |
| T5-09 | SNIPPET | WoWAH sampled in ten-minute intervals; RAR is 577,334,422 bytes on the WPI mirror | https://sigmm.org/Resources/user-traces ; https://web.cs.wpi.edu/~claypool/mmsys-dataset/2011/wow/ | "information of avatar in each ten-minute interval" |
| T5-10 | SNIPPET | WoWAH contains 667,032 sessions | https://calmcode.io/datasets/wow-lite | "91,065 avatars, and 667,032 sessions" |
| T5-11 | VERIFIED | Second Life (La and Michiardi): 10 s snapshots, 24 h, 65/34/13 average concurrent; "publicly available" (no URL) | https://arxiv.org/abs/0804.1928 | "Our measurements are publicly available" |
| T5-12 | VERIFIED | Second Life (Liang et al.): 84,208 avatars, 22 regions, 62 M records, 10 s; only a plan to share | https://arxiv.org/abs/0807.2328 | "which we plan to share with the research community" |
| T5-13 | VERIFIED | Real traces and generated movement gave significantly different results (FreeWar, Orbius) | https://arxiv.org/abs/0807.2328 | "both observe significant differences in their results evaluated using the traces and using generated movements" |
| T5-14 | VERIFIED | No player-behaviour models for Minecraft-like games (Meterstick) | https://arxiv.org/abs/2112.06963 | "no player-behavior models exists for MLGs" |
| T5-15 | VERIFIED | Servo also used random behaviour for lack of models | https://arxiv.org/abs/2305.00032 | "We use a randomized behavior because no MVE-specific player behavior models currently exist." |
| T5-16 | SNIPPET | HeapCraft: 3,451 player-hours, 908 players, 43 servers; no public dataset found | https://ojs.aaai.org/index.php/AIIDE/article/view/12807 | "over 3451 player-hours of gameplay from 908 players over 43 different servers" |
| T5-17 | VERIFIED | MineRL: 60 M single-player state-action pairs | https://arxiv.org/abs/1907.13440 | "over 60 million automatically annotated state-action pairs" |
| T5-18 | VERIFIED | ESTA: 1,558 demos, 2 Hz, 3.9 GB, CC BY-SA 4.0, on GitHub | https://github.com/pnxenopoulos/esta | "contains frames ("game snapshots") parsed at 2 Hz" |
| T5-19 | VERIFIED | ESTA needs the old awpy 1.x | https://github.com/pnxenopoulos/esta | "ESTA data is only compatible with Awpy version 1.x" |
| T5-20 | GUESS | One ESTA match: top 5 % of visited cells hold 19.6 % of player-time; `spotters` non-empty in 23/6,040 player-frames | own analysis of `data/lan/0013db25-4444-452b-980b-7702dc6fb810.json.xz` | – |
| T5-21 | VERIFIED | awpy parses CS2 demos; VisibilityChecker uses static collision geometry | https://github.com/pnxenopoulos/awpy | "It does not model smokes, moving doors, or bullet penetration." |
| T5-22 | VERIFIED | EgoCS-400K built from public pro CS/CS2 demos | https://arxiv.org/abs/2606.18180 | "built from public professional CS and CS2 match demos" |
| T5-23 | VERIFIED | DECOY: learned CS:GO simulator matches human replays | https://arxiv.org/abs/2509.06355 | "replays generated from human data in DECOY closely match those observed in the original game" |
| T5-24 | VERIFIED | NCSOFT Lineage W positions logged once a minute (private) | https://arxiv.org/abs/2501.10461 | "coordinates and timestamp logs sampled at one-minute intervals" |
| T5-25 | VERIFIED | SS14 server replays record full, unculled state | https://github.com/space-wizards/RobustToolbox (Robust.Server/Replays/ReplayRecordingManager.cs) | "{ DisableCulling = true }" |
| T5-26 | VERIFIED | Wizard's Den servers cap at about 80 players | https://github.com/space-wizards/space-station-14 (Resources/ConfigPresets/WizardsDen/wizardsDen.toml) | "soft_max_players = 80" |
| T5-27 | SNIPPET | Wizard's Den records every round and makes replays available | https://forum.spacestation14.com/t/server-replays-are-now-available-on-wizards-den/5418 | "automatically recorded and made available for viewing" |
| T5-28 | SNIPPET | Game Trace Archive: 9 traces, >8 M players; host does not resolve | https://ieeexplore.ieee.org/document/6404027 | "covering more than 8 million real players" |
| T5-29 | VERIFIED | Yardstick 2019: Minecraft-like servers are poorly parallelised | https://api.semanticscholar.org/graph/v1/paper/DOI:10.1145/3297663.3310307 | "Minecraft-like services are poorly parallelized" |
| T5-30 | VERIFIED | Yardstick repo is active (71 commits in the last year, last 2026-09-30), uses mineflayer random-box walkers, has no licence file | https://github.com/atlarge-research/yardstick | "walks, on foot, between random points inside a box" |
| T5-31 | VERIFIED | Yardstick: overloaded bot machines make the server look healthy | https://github.com/atlarge-research/yardstick (README) | "the run reports a healthy-looking server exactly when it has stopped measuring anything" |
| T5-32 | VERIFIED | Yardstick runs the server on its own machine | same | "The game server always gets a machine to itself" |
| T5-33 | VERIFIED | Meterstick player workload is 25 random walkers; repo last commit 2022-02-17; GPL-3 | https://arxiv.org/abs/2112.06963 ; https://github.com/JerritEic/Meterstick | "connect 25 players which move randomly in a 32-by-32 area" |
| T5-34 | VERIFIED | Cloud raises result spread | https://arxiv.org/abs/2112.06963 | "a 1.39 up to 15.44 times increase in ISR IQR" |
| T5-35 | VERIFIED | Meterstick: ISR not linked to player experience | same | "It is not currently understood how ISR directly relates to player-perceived quality of experience" |
| T5-36 | VERIFIED | Opencraft tutorial: 50 bots on a flat plane, 5-minute run; MIT; last commit 2023-11-08 | https://github.com/atlarge-research/opencraft-tutorial | "Expected runtime: 5 minutes" |
| T5-37 | VERIFIED | mineflayer is MIT and active (111 commits, 20 authors in the last year) | https://github.com/PrismarineJS/mineflayer | "The MIT License (MIT)" |
| T5-38 | VERIFIED | Veloren bot client sends no inputs (idle bots) | https://gitlab.com/veloren/veloren (client/src/bin/bot/main.rs) | "client.tick(comp::ControllerInputs::default()" |
| T5-39 | VERIFIED | OpenSimulator pCampBot bots walk and jump randomly | https://github.com/opensim/opensim (OpenSim/Tools/pCampBot/README.txt) | "randomly jump/walk around" |
| T5-40 | VERIFIED | mod-playerbots: questing bots; claims thousands of bots; needs a custom AzerothCore fork | https://github.com/mod-playerbots/mod-playerbots | "Excellent performance, even when running thousands of bots" |
| T5-41 | VERIFIED | Community AOI benchmark uses uniform random moves | https://github.com/bhhbazinga/AOI | "We simulate N random moves of N units in a 1024*1024 map" |
| T5-42 | VERIFIED | AzerothCore visibility constants are a hand-made table | https://github.com/azerothcore/azerothcore-wotlk (src/server/game/Misc/DynamicVisibility.h) | "feel free to add more intervals, change existing ones or move to conf file" |
| T5-43 | VERIFIED | AzerothCore defaults: continents visibility 100, one map-update thread; GPL-2; 3,652 commits in the last year | same repo, worldserver.conf.dist | "MapUpdate.Threads = 1" |
| T5-44 | SNIPPET | ChromieCraft (AzerothCore) peaked above 1,300 players in 2025 | https://www.chromiecraft.com/es/?p=35508 | "a new peak of over 1300 players online" |
| T5-45 | SNIPPET | AzerothCore claimed to handle 15,000 concurrent players in stress tests (vendor claim) | chromiecraft.com (search excerpt) | "at least 15,000 concurrent players" |
| T5-46 | VERIFIED | SS14 PVS budgets and range are hand-set constants | https://github.com/space-wizards/RobustToolbox (Robust.Shared/CVars.cs) | "CVarDef.Create("net.pvs_budget", 50" |
| T5-47 | VERIFIED | RobustToolbox code after 2019-03-13 is MIT | https://github.com/space-wizards/RobustToolbox (legal.md) | "contributed **after** 13 March, 2019 at 12:00:00 UTC are licensed under the MIT license" |
| T5-48 | SNIPPET | SS14 has about 3,500 concurrent Steam players | https://tracker.gg/population/steam/1482520 | "3,458 players online" |
| T5-49 | VERIFIED | Veloren record: 277 concurrent players (0.13, 2022) | https://gitlab.com/veloren/site (content/blog/2022-07-25-devblog-182.md) | "smashed our previous release party record with 277 players online at peak!" |
| T5-50 | VERIFIED | Veloren interest management is region subscription with 512-unit regions; GPL-3; 934 commits in the last year | https://gitlab.com/veloren/veloren (common/src/region.rs) | "const REGION_LOG2: u8 = 9;" |
| T5-51 | VERIFIED | Minestom exposes chunk-to-thread assignment as a one-method interface; Apache-2.0 | https://github.com/Minestom/Minestom (ThreadProvider.java) | "int findThread(T partition);" |
| T5-52 | VERIFIED | Folia's regionizer is inside a 1,090,131-byte patch; GPL-3 patches | https://github.com/PaperMC/Folia | "PATCHES-LICENSE file describes the license for api & server patches" |
| T5-53 | VERIFIED | Valence is near-dormant (5 commits in the last year); MultiPaper's last commit is 2025-05-22 ("public beta", 1.20.1) | https://github.com/valence-rs/valence ; https://github.com/MultiPaper/MultiPaper | "MultiPaper is in public beta." |
| T5-54 | VERIFIED | Luanti object-send and block ranges are settings | https://github.com/luanti-org/luanti (builtin/settingtypes.txt) | "active_object_send_range_blocks (Active object send range) int 8" |
| T5-55 | VERIFIED | Ryzom Core (AGPL-3, 1,671 commits on `core4`) has a separate position and vision service | https://github.com/ryzom/ryzomcore | file `ryzom/server/src/gpm_service/vision_delta_manager.cpp` |
| T5-56 | VERIFIED | Hypixel's scale comes from many independent instances | https://arxiv.org/abs/2112.06963 | "stitching together thousands of (independent) MLG instances" |
| T5-57 | SNIPPET | Random waypoint's average speed decays; metrics shift up to 40 % | https://doi.ieeecomputersociety.org/10.1109/INFCOM.2003.1208967 | "the average nodal speed consistently decreases over time" |
| T5-58 | SNIPPET | Floyd and Paxson: heterogeneity; search for invariants | https://ee.lbl.gov/papers/paxson.html | "judiciously exploring the simulation parameter space" |
| T5-59 | VERIFIED | MANET survey: simulation credibility decreased | https://api.crossref.org/works/10.1145/1096166.1096174 | "the credibility of the simulation results has decreased" |
| T5-60 | VERIFIED | CausalSim: trace replay under a new policy can be wrong | https://arxiv.org/abs/2201.01811 | "replaying traces under an intervention may lead to incorrect results" |
| T5-61 | VERIFIED | Puffer: simulator winners lost in the real world | https://arxiv.org/abs/1906.01113 | "notwithstanding good performance in network emulators or simulators" |
| T5-62 | VERIFIED | Donnybrook: 88-person user study; simulator calibrated on real Quake III behaviour; no real workload with hundreds of players | https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/donnybrook.pdf | "it is not possible for us to obtain a real workload with hundreds of players" |
| T5-63 | VERIFIED | Hetzner shared vCPUs are shared through the hypervisor; dedicated ones are not | https://docs.hetzner.com/cloud/servers/overview/ | "Every dedicated resources instance has its own dedicated CPU resources" |
| T5-64 | VERIFIED | Godot can run headless and at a fixed frame rate without real-time sync | https://github.com/godotengine/godot-docs (tutorials/editor/command_line_tutorial.rst) | "This setting disables real-time synchronization." |
| T5-65 | GUESS | Interest-management evaluator: 2–4 weeks for one person (week plan in 5.1) | own estimate | – |
| T5-66 | GUESS | Partitioning: toy simulator in 1–2 weeks; calibration and validation another 3–6 weeks; trust limited by bot realism | own estimate | – |
| T5-67 | GUESS | Session packing evaluator in 1–2 weeks once logs exist | own estimate | – |
| T5-68 | GUESS | Live bot evaluation of 1,000 candidates × 3 seeds × 5 min ≈ 10.4 days; offline at 2 s ≈ 1.7 h | arithmetic in 5.4 | – |
| T5-69 | GUESS | No MMO-scale public fine-position dataset, no maintained cross-game MMO benchmark suite, no game-fleet simulator found | searches listed in section 8 | – |
