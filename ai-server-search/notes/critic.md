# Completeness critic (raw output, 2026-11-08)

The top 3 gaps were researched further (notes G1, G2, G3); gaps 4 and 5 went to the judges unresearched.

## Gaps

### Gap 1
Nobody looked at open-source RuneScape private servers (RSPS). For an OSRS-style server, that is where the hot paths and the strong human-written baselines are. RSProt, rsmod, Lost City/2004Scape and 2009scape appear nowhere in the six notes files or in the earlier tick-lang research. I checked RSProt (github.com/blurite/rsprot, MIT licence). It ships JMH benchmarks for the OSRS 'player info' and 'NPC info' builders, the per-tick code that works out who is near each player and what changed (choosing what each player sees, then encoding the update). Player info for 2,046 players inside a 13x13-tile box takes 170 ms per 600 ms game cycle on one thread, or 11.5 ms multithreaded (Ryzen 3950X, JDK 8). NPC info with 500 NPCs takes 64 ms on one thread. Each viewer's range shrinks to keep at most 250 high-resolution players, and the protocol shows at most 250 NPCs. Lost City (github.com/2004Scape/Server, now LostCityRS/Server and LostCityRS/Engine-TS) is an open TypeScript engine with a built-in web client. My copies of RSProt's README and BENCHMARKS.md are in /tmp/claude-0/-home-user-OogaPL/32372848-6606-541b-85af-54d683c4e916/scratchpad/critic-dl/.

**Why it matters:** Miles's recorded target is not Sumo Brothers. /home/user/OogaPL/ai-server-search/README.md says it is an OSRS-like tile MMO with a 0.6 s tick and thousands of players, built as his own Rust test server. For that target the measured hot path is the per-tick update builder, and its cost is CPU. The evidence aimed elsewhere: T5's interest-management scorer counts bytes, which cost about EUR 0 on Hetzner, and T3 calls interest management irrelevant for small arenas. These projects supply three things the evidence says AI search needs:
1. Strong human-optimised baselines. T1 shows that big 'wins' come from weak baselines, and under option A the baseline would otherwise be Claude's own first draft.
2. Harnesses whose output (packet bytes, paths) can be compared exactly, so every candidate is correct by construction.
3. Pull-request histories that show how much speed-up humans already found.
Player and NPC info, tile pathfinding and NPC target scans could replace fleet placement and snapshot compression as the recommended targets.

**Instruction given:** 1. Read rsprot's README.md and BENCHMARKS.md, then PlayerInfoBenchmark.kt and NpcInfoBenchmark.kt (linked from BENCHMARKS.md) and the player-info code under protocol/osrs-*/. Record the algorithm (high/low resolution, view-range shrinking, appearance caching), whether any test compares encoded bytes with a reference, and any TODO or performance comments.
2. In rsmod/rsmod and LostCityRS/Engine-TS (and LostCityRS/Server), find the tick loop, player info, tile pathfinding and NPC hunt/aggression code. Record language, licence and any benchmarks.
3. For each repo, run: gh api -X GET search/issues -f q='repo:OWNER/REPO performance'. Repeat with benchmark, pathfinding, 'player info' and tick. Record before/after numbers from merged PRs as evidence of how much speed-up humans found.
4. Note the licence and Jagex intellectual-property limits on reusing any of this in an open MIT/Apache benchmark.
Deliver a ranked list of OSRS-style hot paths with measured costs. For each, name the strongest existing baseline and harness, and say whether it has an exact correctness check. If time is short, do steps 1 and 3 first.

### Gap 2
The capacity numbers behind the cost model and the proposed score are unchecked guesses. T2's OSRS-like model rests on '100 players per vCPU. This is a pure GUESS' and '2,000-3,000 CCU per box (GUESS)'. Nobody measured, or found measurements of, how much CPU and RAM an OSRS-style world uses when players are spread normally versus crowded at hotspots. Nobody asked whether the ~2,000-player world size is a protocol or design cap rather than a CPU limit. RSProt's worst case has all 2,046 players crowded together, teleporting and chatting every cycle, and needs about 28% of one core for player info. Its NPC benchmark notes call a similar NPC setup 'extremely unrealistic'. That hints T2's guess may be an order of magnitude too pessimistic for normal play, and that crowds, not average load, set the limit.

**Why it matters:** Suppose a Rust OSRS-style world with 2,000 players uses only a small share of one core except in crowds. Then the server cost per player is far below T2's estimate. The README's proposed score ('how many bot players one server can hold before a tick takes longer than its time budget') would mostly measure a player cap or an invented crowd. The only things worth searching would be crowd-spike handling or how many worlds fit on one box, where RAM may be the limit. That changes whether AI search can make this game cheaper at all, and what the score should be.

**Instruction given:** 1. Search LostCityRS/Server, LostCityRS/Engine-TS, rsmod/rsmod, and any Rust engine found with: gh api -X GET search/repositories -f q='runescape server language:rust'. In each repo, use gh api search/code over READMEs, docs, run scripts, configs and issues for: tick, 'players online', stress, bot, Xmx, memory. Record measured tick times against player counts, and RAM per world.
2. Grep rsprot and rsmod for the player-index width or limit (2047, 2048, MAX_PLAYERS) to see whether the ~2,000 cap comes from the protocol.
3. Find public crowd data: GitHub repos that log the OSRS world list (player counts per world over time; search 'osrs world population'), and RuneLite (runelite/runelite) code on player render limits. If WebSearch has budget, add excerpts of Jagex posts on crowding at the Grand Exchange or on busy worlds.
4. Redo T2's OSRS-like cost per CCU-month using sourced players-per-core ranges for normal and crowd cases.
Deliver: players per core (normal and crowd, with sources), RAM per world, the cause of the per-world cap, and the revised cost per CCU-month.

### Gap 3
No evidence covers the search method Miles chose. /home/user/OogaPL/ai-server-search/README.md records 'option A, inside Claude Code (agents in Claude Code propose and test candidates)'. The notes never mention benchmarks that test coding agents at making real code faster. I confirmed three on arXiv:
- GSO (2505.23671): leading SWE-agents achieve 'less than 5% success rate' against expert optimisations, with failure modes that include 'difficulties with low-level languages' and 'lazy optimization strategies'.
- SWE-fficiency (2511.06090): agents 'achieve less than 0.23x the expert speedup'.
- SWE-Perf (2507.12415): 'a substantial capability gap'.
The GSO repo's news (3 Nov 2025) adds a 'HackDetector that catches models reward hacking'. These are 2025 results, and how current models do in 2026 is unchecked. Two more things are unexamined: reward hacking by agents that have shell access, and how many candidates the plan's usage allows.

**Why it matters:** The success rates and costs in the evidence come from evolutionary frameworks. They score hundreds to thousands of candidates against an evaluator the LLM cannot touch. Option A agents work in the same repository and container as the evaluator and the 'hidden test scenarios', so both the risk of gaming the score and the number of candidates differ. If 2026 agents still reach only a small fraction of expert speed-ups, option A should be limited to building the evaluator and the baselines. The search itself would then run as option B (OpenEvolve on the $200/month API credit). That would change the feasibility answer for the setup Miles actually has.

**Instruction given:** 1. Open arxiv.org/abs/2505.23671, /2511.06090 and /2507.12415, and check each for a 2026 version (v2, v3). Record the headline numbers, the models tested and the failure modes.
2. In github.com/gso-bench/gso and gso-bench/gso-experiments, grep for leaderboard or results files that include 2026 models (opus, claude, gpt-5, opt@). Also find the HackDetector code and the hack rates it reports.
3. Query api.semanticscholar.org/graph/v1/paper/search for 2026 papers on coding-agent performance optimisation, and on coding agents gaming tests (for example ImpossibleBench). Record the hack rates and the isolation they recommend, such as keeping the evaluator outside the agent's sandbox.
4. Estimate candidates per month for option B: $200 divided by the cost per iteration. Take Claude prices from raw.githubusercontent.com/BerriAI/litellm/main/model_prices_and_context_window.json and tokens per iteration from OpenEvolve's docs/llm-providers.md. Do the same for option A if plan usage limits are documented.
Deliver: the expected 2026 success rate of agent-driven optimisation against expert baselines, the main failure and hacking modes, the isolation needed, and candidates per month for each option.

### Gap 4
Nobody has tested whether a trustworthy CPU score can be measured where option A will run. The README's score is wall-clock tick time against the 0.6 s budget. T5 recommends counting operations or using dedicated cores, but nobody checked the real environment. I checked this cloud container:
- 4 vCPUs ('Intel(R) Xeon(R) Processor @ 2.10GHz') under a hypervisor (kernel 6.18.44-fc).
- /sys/bus/event_source/devices has no 'cpu' entry (only breakpoint, msr, power, software, tracepoint, uprobe), and the CPU flags lack arch_perfmon. So there are no hardware performance counters.
- perf is not installed, and perf_event_paranoid=2.
So hardware instruction counts are unavailable. The only options left are valgrind (installed), which is deterministic but slow, and noisy wall-clock timing. Meterstick found the spread of tick times (IQR) 1.09-5.61x larger on cloud machines than on a dedicated node.

**Why it matters:** The evidence expects single-digit-percent gains over strong baselines. If run-to-run noise in this container is that large or larger, option A cannot tell a real improvement from noise, and a binary search for 'max players' will give different answers from run to run. Scoring would then have to move to deterministic instruction counts (cachegrind, much slower) or to dedicated cores on Hetzner (CCX or AX). A cloud session may not be able to reach those, which would push the search onto Miles's own machine or into option B.

**Instruction given:** In a Claude Code cloud container, about 15 minutes:
1. Record nproc, the CPU model, the contents of /sys/bus/event_source/devices, and perf_event_paranoid.
2. Use cargo new to make a small benchmark: 20,000 BFS path queries on a 104x104 tile grid with random walls and a fixed seed. Build with --release. Run it 30 times plain and 30 times under taskset -c 1. Report min, median, p90 and the coefficient of variation.
3. Run it twice under valgrind --tool=cachegrind. Check that the instruction counts are identical, and measure the slowdown.
4. Check on crates.io that iai-callgrind (or gungraun) and criterion install.
5. Read docs.hetzner.com on dedicated versus shared vCPUs. If the read_documentation tool is available, read its environment.network topic to see whether a user's own server can be added to a cloud session's allowlist, and whether SSH works through the proxy.
Deliver: the noise level compared with the expected gain size, whether instruction counts are deterministic, evaluations per hour for each method, and a recommended way to score.

### Gap 5
The search for novelty and prior attempts used only English sources and never looked at patents. T6 searched arXiv, Semantic Scholar, GitHub and one web query. Its web budget ran out, and Hacker News, Reddit and GDC were blocked. It ran no Chinese or Korean queries, although the only known studio user, Kuro Games, is Chinese, and AOI (area-of-interest) algorithms are a common topic in Chinese game-server writing. It also ran no patent search. I ran a GitHub code search for the OpenEvolve marker 'EVOLVE-BLOCK-START' together with the Chinese word for 'game' (7 hits) and with 'AOI' (12 hits). Both returned only blog posts about search frameworks and copies of the ShinkaEvolve README, so public code is unlikely. Talks, blogs and patents are still unchecked, and what Kuro actually optimised is still unknown.

**Why it matters:** The verdict calls the application 'new in public', and part of Miles's motivation is his belief that few people have tried this. A Chinese or Korean studio talk, blog post or patent describing LLM-evolved server heuristics would weaken that claim. It would also give the game-server gain numbers the evidence lacks, since all it has now is Kuro's quote without numbers.

**Instruction given:** This turn's web-search budget is used up, so WebSearch may need a new turn.
1. With WebSearch, run these queries: 'AlphaEvolve 游戏 服务器', '库洛 AlphaEvolve', 'Kuro Games AlphaEvolve', '大模型 进化 启发式 游戏服务器 AOI', 'OpenEvolve 游戏', 'AlphaEvolve 게임 서버', 'Google Cloud Next 2026 Kuro Games AlphaEvolve', and the patent query 'patent large language model generated heuristic game server allocation'. Record titles, URLs, dates and any numbers.
2. Run gh api -X GET search/repositories -f q='AlphaEvolve game server', and a gh api code search for OpenEvolve together with '服务器'.
3. Query api.semanticscholar.org/graph/v1/paper/search for 'large language model evolutionary heuristic game server', limited to 2025-2026.
If WebSearch is unavailable, do steps 2-3 and report the patent and non-English web parts as not done.
Deliver every prior attempt found (studio, problem, metric, gain, date), or a list of what was searched with no hits.

## Framing problems
- Wrong workload. Five of the six topic verdicts judge payoff and relevance by Sumo Brothers on one Hetzner box:
- T1: 'With a single Hetzner box the payoff is cents to a few euros a month'.
- T3: 'No value for small-arena games like Sumo Brothers'.
- T4: 'For Sumo Brothers on one Hetzner box this family is worth about EUR 0'.
- T5: 'On one box it saves little'.
- T6: 'aim ... at Miles's own servers'.
But /home/user/OogaPL/ai-server-search/README.md, under 'Decisions and facts from Miles so far', records a different target: an OSRS-like pixel-art tile MMO with a 0.6 s tick, aiming for thousands of players, built as his own original Rust test server, with an open benchmark and write-ups. Only T2 modelled that kind of game. For that target, fleet placement and autoscaling barely matter, since there are only a few worlds. Interest management is a per-tick CPU cost (OSRS-style player and NPC info), not a bandwidth cost. Success means capacity per world, steady ticks in crowds and a reusable benchmark, not Miles's euro bill. The final answer should be re-aimed at that target.
- Method mismatch. The evidence assesses AlphaEvolve/OpenEvolve-style evolution: 100 to 1,000+ scored candidates, $5-30 of LLM fees per run, and an evaluator the LLM cannot touch. The recorded decision is option A, where Claude Code agents with shell access propose and test candidates using the plan's usage. So pay-back sums in LLM fees ('below the cost of one search plus setup') do not apply. What decides feasibility is how many candidates the usage limits allow, how well agents optimise real code, and keeping the evaluator out of the agent's reach. The README guards against overfitting with 'hidden test scenarios the AI never sees', but those cannot stay hidden from an agent working in the same repo and container unless they are stored somewhere else.
- Self-made baseline. In option A the same AI writes both the baseline Rust server and the 'discoveries'. T1's own finding is that big multipliers come from weak baselines, so a speed-up over Claude's first draft says nothing about a new algorithm. Under option A, 'conventional engineering first' and 'AI search' become the same activity: Claude writing the server. The real question is whether a scored search loop adds anything beyond Claude implementing known strong designs. Answering it needs an outside strong baseline, such as RSProt's player-info builder or published OSRS-style designs, plus a classical tuning baseline (grid or Bayesian search over the same knobs).
- No success criterion. 'Cheaper', 'more efficient' and 'a new algorithm' pull in different directions. At Hetzner dedicated prices an OSRS-like player costs a few euro cents a month (T2, a guess), so euros are a weak goal at indie scale. People who optimise live game servers aim for the same behaviour, only faster: Minecraft's Lithium mod says it 'doesn't change any game mechanics or visuals, it just makes the game run faster', and OSRS-style games depend on exact tick behaviour. Before any search, define what counts as 'finding something'. Examples: at least X% fewer CPU-milliseconds per tick than a strong baseline on unseen crowd scenarios with the same final game state, or a benchmark that other developers adopt.
- The proposed score is shaky. The README scores 'how many bot players one server can hold before a tick takes longer than its time budget' and says this 'converts directly into players per euro'. But:
- The count may hit protocol or design caps before CPU runs out: OSRS-style worlds hold about 2,000 players, and RSProt limits each viewer to 250 high-resolution players and 250 NPCs.
- It depends on how realistic the bots are at crowd hotspots.
- It is wall-clock time on one particular machine.
- Players per euro also depends on how many worlds fit on one box (RAM as well as cores) and on Hetzner's fixed tiers.
A better search score is a deterministic cost per tick on fixed, replayed scenarios (instructions or CPU-milliseconds per player per tick, with the same final game state), plus a separate crowd-spike test. Capacity on a real Hetzner box can be the final check.
- The data plan does not fit the constraints. T5 makes 'Miles's own trace logging' the first deliverable and later switches to 'Sumo Brothers' own recordings'. But for the OSRS-like target, Miles's test server has no players, and the README rules out using the team's code or data without written agreement. Realistic crowd workloads must therefore come from public sources, from open-source private-server operators (nobody has looked for these), or from bot models checked against real play. That makes the realism of the evaluator the main risk for the real target.
- Novelty is framed in the wrong place. The user believes few people have tried using AI for this, and the evidence partly agrees ('new in public'). In fact, LLM program search for systems heuristics was mainstream in 2025-26 (ADRS; AlphaEvolve generally available on Google Cloud), and a game studio (Kuro Games) publicly reports using it on server workloads. What would be new and useful is an open, hard-to-game evaluator and benchmark for OSRS-style servers. The summary should say this plainly, not imply that using AI is the novel part.
