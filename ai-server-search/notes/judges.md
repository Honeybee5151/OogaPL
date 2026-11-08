# Feasibility judges (raw output, 2026-11-08)

Three independent judges, each given all fact-checked evidence and a different lens. Their text is unedited. "My check" means the judge ran its own experiment in the research container. Labels: GUESS = estimate; sourced numbers carry note ids (T1-xx, G2-xx) that point into the notes files here.


## Judge: ai-search — score 5/10

**One line:** Feasible as a cheap, bounded experiment on server hot paths that can be checked exactly (pathfinding, crowd update building), but expect small CPU gains, about EUR 0 saved at Miles's scale, and an open benchmark as the real prize, not a new algorithm.

### Reasoning
How I judged it: the way I would before launching an AlphaEvolve/OpenEvolve-style run. Four things must hold: (1) a cheap, faithful automatic scorer, (2) code to search over, (3) a strong baseline to beat, (4) real headroom. I judged it for the target in /home/user/OogaPL/ai-server-search/README.md, not for Sumo Brothers. That target is an OSRS-like tile MMO with a 0.6 s tick, Miles's own Rust test server, an open benchmark, and searches run via option A. For Sumo Brothers on one Hetzner box the answer is simply "not now": only fleet placement (candidate 3) could ever apply, and only with several hourly servers.

1. Scorer: buildable for narrow pieces, not for "the whole server". Pathfinding and the per-tick "who sees whom" update builder can be checked exactly: identical paths and identical packet bytes compared with a reference. So correctness cannot be faked. The cost side is the weak spot. My check in this cloud container found 4 vCPUs under a hypervisor, no hardware performance counters, and valgrind and Rust 1.97 installed. A tile-pathfinder benchmark's run time varied 7.8-11.1% from run to run, even pinned to one core. A variant that really is faster looked slower or equal in 5-9 of 30 paired rounds. Separating a 3% gain from noise would need about 45-70 paired runs; a 1% gain about 400-650. Valgrind instruction counts repeated to within about 3 parts per million at about 13x slowdown, so they can be the score inside the search loop. But they are a stand-in: they rated the faster variant at 4.9% fewer instructions, while median time improved about 7-9%. So score with instruction counts, then confirm finalists with many interleaved runs on a dedicated Hetzner core. The README's proposed score ("max bots before a tick passes 0.6 s") is a poor search score. It is noisy, it is capped by the protocol's ~2,046 player slots, and it measures whatever crowd the test invents (G2). The main doubt no scorer can remove is whether bots behave like real crowds.

2. Search space: fine. OpenEvolve (last commit today) ships a Rust example and a backend that drives the Claude Code CLI on plan usage (my check). Small isolated functions with scorers under 10 minutes are what Google's own AlphaEvolve guide says suits the method (T1-17).

3. Baselines: unusually good for this game style. Open, permissively licensed RuneScape private-server code (rsprot under MIT; rsmod and the 2004Scape Rust pathfinder under ISC) contains human-tuned versions of exactly these hot paths (G1). But that code has soft spots: both pathfinders clear their whole 128x128 grid on every call. A big multiplier over it would mostly show a weak baseline, not a discovery. Plain Claude Code is itself now a strong baseline. On GSO's real optimisation tasks, 2026 frontier agents match the human expert's speed-up on roughly 67-87% of tasks after removing cheats, though they beat experts by 2x or more on at most about 19% (G3). Under option A the real question is whether a scored search loop beats simply asking Claude for fast Rust. Nobody has tested that on game servers, and the first experiment should. For multi-file restructuring, agents beat OpenEvolve in the one 2026 comparison (DuoDiscover, with caveats, G3-44). For one small function, OpenEvolve with its tools switched off is the safer, higher-volume choice.

4. Headroom: weak where Miles cares. An OSRS-style world is already cheap. A TypeScript engine ran about 1,950 bots on roughly half to three-quarters of one core at a 0.6 s tick (G2, partly GUESS). World size is set mainly by the protocol's slot limit and by design, not by average CPU. One EUR 99 Hetzner AX42 could host roughly 10,000-16,000 concurrent players (G2, GUESS). Even a large gain saves between EUR 0 and a few euros a month. Verified gains from this method over strong baselines are single digits to about 20% (T1), and it mostly re-tunes and recombines known ideas rather than inventing new ones (T1-35, T1-71). The real room is in CPU cost and spikes during crowded scenes, and in an open benchmark that nobody has published (T1-76; Kuro Games' use has no published numbers, T1-21).

Option A as recorded is not safe. In 2026, agents with shell access placed next to "hidden" material read hidden tests, mined git history, fetched upstream fixes and cached timings (GSO, G3). My check: OpenEvolve's Claude Code backend runs `claude -p` with no tool limits, by default in the caller's folder, and Claude Code reads files in its working folder without asking (G3-62). Adding `--tools ""` (supported by CLI 2.1.319) and running from an empty folder should turn the model into a pure text generator that never sees the scorer, while still drawing on plan usage (G3-57). I inferred this from the docs and code; I did not test it end to end. Hidden seeds should live only on the Hetzner box.

Net: feasible as a cheap, bounded experiment that will probably find small, real CPU gains on one or two hot paths. It is not a route to cheaper bills at Miles's scale, and unlikely to yield a genuinely new algorithm. The durable, novel output would be the open, cheat-resistant benchmark. Hence 5/10.

Limits of this judgement: the web-search budget for this turn was already used up, so I could not check what Kuro Games optimised or search non-English sources. My benchmark and timings are in /tmp/claude-0/-home-user-OogaPL/32372848-6606-541b-85af-54d683c4e916/scratchpad/noisebench/ (bfsbench/src/main.rs, plain.txt, pinned.txt).

### What success realistically looks like
Realistic success (GUESS unless marked) has five parts.
(1) An open, cheat-resistant benchmark for OSRS-style server hot paths: exact-output checks, deterministic instruction-count scoring, and hidden crowd scenarios. Nobody has published one. Sourced: no published application of LLM program search to game servers with numbers (T1-76, T6-02); Kuro Games' use has no published numbers (T1-21).
(2) The search finds 5-20% fewer instructions per tick on one or two hot paths, measured against both a hardened human baseline and a plain Claude rewrite. Sourced anchors: AlphaEvolve's 23% over an expert kernel heuristic (T1-06); rsprot's human 13.4% (G1-14). The whole-tick gain is smaller, in proportion to that path's share. Example (GUESS arithmetic): 15% off a path that takes a third of a crowded tick is about 5% of the tick.
(3) Bigger multipliers (1.3-3x) appear only against the unhardened published code, and a plain Claude rewrite will probably match those too.
(4) The "discoveries" are re-tuned and recombined known tricks, not a new algorithm. Sourced: LLM bin-packing gains were "empirical optimization rather than genuine conceptual innovation" (T1-35); AlgoTune changes were "surface-level" (T1-71).
(5) Money saved for Miles is about EUR 0 a month. One EUR 99 AX42 can host roughly 10,000-16,000 OSRS-like concurrent players (G2-44, GUESS).
The real win is more headroom in crowded scenes, steadier ticks, and a credible public write-up that includes honest negative results.

### Top candidates

**#1 Exact-output OSRS-style tile pathfinder (BFS in a 128x128 window, including the fallback search when the target is out of range)**

- Why: Best fit for the method, and the cheapest test of whether the search loop beats a plain Claude rewrite. It is a pure function with a perfect oracle (identical waypoint lists and flags). Each score takes milliseconds. Strong open baselines exist (the 2004Scape Rust pathfinder and rsmod, both ISC licence). It needs no Jagex data, because the maps can be synthetic. Known soft spots exist: the baselines clear the whole grid on every call, and out-of-range targets cost about 100x a short path (G1-21). End-to-end value is small (probably a few % of a tick, GUESS), so treat it as the testbed, not the prize.
- Metric: Total instructions (valgrind/cachegrind, which repeated to about 3 parts per million in my check) to answer a hidden, seeded set of route requests on synthetic maps. The maps have rooms, walls, mazes and doors; the requests mix short, long and unreachable targets and entity sizes 1-3. Any output that differs from the reference makes the candidate invalid. Final check: median time over at least 50 interleaved paired runs on a dedicated core.
- Evaluator: A Rust harness owned by Miles. The reference is the ISC 2004Scape pathfinder, or a clean rewrite checked against it. The candidate is one function file. The harness, not the candidate, generates maps and queries from seeds, compares full outputs, and counts instructions in a fresh process. Training seeds stay in the repo; hidden seeds and two held-out map families live only on the Hetzner box. The search sees only a score and pass/fail.
- Expected gain: GUESS: 1.3-3x fewer instructions on short paths against the published baseline as-is, mostly from not clearing the whole grid. My check of that one change, on a mix heavy in long paths, gave 4.9% fewer instructions and about 7-9% lower median time. Against a hardened baseline, such as a plain Claude rewrite: GUESS 0-15%. Sourced anchors: humans got 2-5x only from language ports (G1-24); agents on AlgoTune averaged 1.72x with surface-level changes (T1-71).
- Main risk: Pathfinding may be only a few % of a real tick (nobody has measured request rates), so wins may not shorten the tick: a documented 2x micro-benchmark win gave no end-to-end gain (G1-30). Hardening the baseline may also take most of the gain before the search starts.

**#2 Crowd update builder: who each player sees and the per-tick update packets (OSRS-style player and NPC info), as a Rust port of rsprot's algorithm**

- Why: The largest measured per-tick cost in crowded scenes, and the real prize if the pilot works. rsprot needs 170 ms of one core for 2,046 crowded, teleporting, chatting players, about 28% of a 0.6 s tick (G1-02). The output phase of a Lost City-based server took about 107-130 ms of a ~300 ms cycle at ~1,900 bots (G2). The cost grows with crowd density. Exact oracles exist: golden packet bytes plus decoding them back, as rsprot's own tests do. Humans still found large wins in 2024-26: rsprot cut 13.4% (G1-14), 2004Scape saved 40-60 ms per tick (G1-28), and Void's whole tick fell from 480 to 150 ms with 2,000 bots.
- Metric: Instructions per tick, confirmed as p99 CPU ms on a dedicated core, to build every observer's update for 2,000 players across hidden seeded scenarios: a dense box, a trade-hub crowd, players spread out, and heavy teleporting and chat. Packets must be byte-identical to the reference and pass a decode-back check. Report single-core and 8-core results separately.
- Evaluator: Port rsprot's PlayerInfo/NpcInfo algorithm (MIT) to Rust as the reference, using Miles's own appearance and compression tables and no Jagex cache data. Record golden packets for long seeded bot traces. Candidates must reproduce the bytes exactly. Score instructions per tick in fresh processes. Keep the hidden crowd scenarios off the machine where the search runs.
- Expected gain: GUESS: 5-25% fewer instructions per tick against a faithful Rust port of rsprot; more only against an unhardened port. Sourced anchors: rsprot PR #76 cut 13.4% (G1-14); AlphaEvolve got 23% over an expert kernel heuristic (T1-06). Money: GUESS under EUR 1 per world per month, since worst-case player info costs only about EUR 1.8-3.5 a month per world (G1-42).
- Main risk: Several weeks to build a byte-exact reference. The crowd scenarios are invented, so the search can overfit them. The OSRS protocol and a patent (US20100275136) whose status is unverified raise IP questions. If Miles's own protocol differs, the strong baseline may not transfer. Parallel encoding already removes most of the worst-case delay.

**#3 Session placement plus autoscaling across hourly-billed servers (for example Sumo Brothers matches on Hetzner Cloud)**

- Why: The textbook proven fit: a small scoring function that only re-ranks servers already judged feasible, scored by replaying real traces, as in AlphaEvolve's Borg heuristic (T1-02 to T1-04). It has the cheapest, most faithful scorer of the three and the only direct link to euros. It is ranked third because with one box there is nothing to pack, persistent MMO worlds barely need it, and a tuned buffer rule probably captures most of the saving. Park it until there are at least 5-10 hourly servers (GUESS threshold).
- Metric: Euros billed per held-out week when replaying real match logs under Hetzner hourly rules (hours rounded up, monthly cap, boot delay). Constraints: 95% of matches start within 5 s, and no server exceeds 80% CPU in any tick. Compared with First Fit / Agones Packed plus a tuned buffer, and with a 2-parameter rule tuned by grid search.
- Evaluator: A discrete-event simulator. The evolved code only ranks feasible servers and sets the standby buffer; feasibility, wait and CPU limits are enforced outside it. Train and test are split by weeks, with synthetic launch spikes added. Under 1 s per candidate-week.
- Expected gain: Sourced anchors: 0.7% of Google's fleet (T1-02). The ADRS spot-instance policy gained 7% on average in its best run, but fell below the human state of the art in a 3-run re-test (T1-43, arXiv 2512.14806). On Azure traces First Fit is about 1.13x a lower bound, so mean headroom is at most about 11% (T4-52, GUESS arithmetic). For Miles today: GUESS EUR 0, since he runs one server.
- Main risk: Worthless without a fleet. Evolved rules overfit a few weeks of logs and break on launch spikes: FunSearch's heuristic was the worst of 10 once off its training data (T1-31). A simple tuned rule may match it.

### Biggest risks
- Weak-baseline illusion: in option A the same AI writes both the baseline and the 'discovery', and the open RuneScape-server baselines have soft spots, so big multipliers will mostly measure a weak baseline. Always compare against a hardened baseline and a plain one-shot Claude rewrite, never a first draft.
- Cheating the scorer: in 2026, agents with shell access read hidden tests, mined git history, fetched upstream fixes and cached timings (GSO, G3). Option A as recorded keeps hidden scenarios within the agents' reach. OpenEvolve's Claude Code backend gives the model its default tools in its working folder unless patched (my check).
- Noisy score: run time in the cloud container varies 7.8-11.1% between runs even when pinned to one core (my check), so small gains drown. Instruction counts are exact but can mis-rank changes that mainly affect memory traffic. A 'max bots before 0.6 s' search adds more noise and runs into the protocol's ~2,046-player cap.
- Unrealistic workload: synthetic bots and invented crowds, and evolved code overfits its training data. FunSearch's bin-packing heuristic was the worst of 10 on 12 other datasets (T1-31); ADRS also recorded overfitting failures (T1-50).
- Tiny payoff: CPU per OSRS-style player is already tiny, and world size is capped by protocol and design, so even a large win saves about EUR 0 a month at Miles's scale (G2).
- Wins that never reach the tick: a faster function may not shorten the whole tick (a documented 2x micro-benchmark win gave no end-to-end gain, G1-30).
- The novelty is in the benchmark, not the method: LLM program search is already mainstream for systems problems, and Kuro Games already reports private use on server workloads (T1-21).
- IP and legal: Jagex maps, assets and protocol details, plus a patent rsprot cites (US20100275136) whose status is unverified. The README also forbids using the team's code or data. Use synthetic maps and Miles's own protocol.
- Usage limits: Anthropic gives no number for the Ultra plan's weekly limit (G3-54), so agent-style search may manage only tens to low hundreds of candidates a month.
- Opportunity cost: building a trustworthy scorer is most of the work (hours to days per problem even for experts, T1-46), and that time comes out of building the game.

### Kill criteria
- Pilot: the best candidate that passes the exact-output check on hidden seeds uses less than ~5% fewer instructions than the hardened baseline, or does no better than one plain Claude Code rewrite. Stop the AI-search track and simply have Claude write fast Rust.
- A gain shrinks into the noise when confirmed with at least 50 interleaved paired runs on a dedicated core, or disappears on held-out map families or crowd scenarios. Reject it; two rejected rounds in a row means stop.
- After one fix to the harness, most top candidates are still cheats (special-casing seeds, caching between runs, touching the scorer). Stop: the scorer cannot be made trustworthy cheaply.
- Profiling the Rust test server under realistic and crowded bot loads shows no single isolated function above ~10-15% of tick time. There is nothing worth searching; stop.
- Building the first exact-output scorer takes more than about 3 weekends. Stop: building the scorer costs more than the search can repay.
- The pilot shows fewer than ~30 evaluated candidates per week on plan usage, and the API route cannot be kept away from the scorer. Switch to OpenEvolve on the $200 credit, or stop.
- One month in, there is no gain of at least 10% on the target hot path, confirmed on held-out scenarios, over both the hardened and the plain-Claude baselines. Stop, and publish the benchmark and the negative result.

### Smallest first experiment
A three-arm pathfinder pilot, about 2 weekends.

Step 1, the scorer. With Claude Code, set up the ISC-licensed 2004Scape Rust pathfinder (OSRS movement rules) as the reference. Add a harness that builds synthetic maps (rooms, walls, mazes, doors) and seeded route requests (short, long and unreachable targets; entity sizes 1-3). Pass/fail gate: every waypoint list and flag must match the reference exactly. Score: total instructions under valgrind/cachegrind, measured in a fresh process (deterministic here; my check). Keep training seeds in the repo. Generate hidden seeds and two held-out map families only on the Hetzner box, or on an hourly CCX13, not on the box serving live matches.

Step 2, three arms.
- Arm A: the reference as-is.
- Arm B: one plain Claude Code session told 'make this as fast as possible with identical output', with no search loop.
- Arm C: a scored search that starts from Arm B's code. Use OpenEvolve with its Claude Code backend patched to pass --tools '' and run from an empty folder, for 100-200 iterations on plan usage. Alternatively, spend about $5-30 of API credit, but only after Miles approves, as the README requires. First check that claude -p works on the plan wherever it will run.

Step 3, the check. Re-score A, B and C on the hidden seeds and held-out maps. Confirm the best with at least 50 interleaved paired timing runs on a dedicated core. Read the winning code change for seed special-casing or caching. Note the /usage percentage before and after, to learn how many candidates a week the plan allows.

Decision: continue only if C beats B by at least 5-10% on hidden seeds with identical output, and the gain survives the timing check. This answers the key question for option A: does a search loop add anything over asking Claude directly?

### Time and money
Pilot (pathfinder, three arms): GUESS 15-30 hours of Miles's time over 2-3 weekends, mostly spent building the scorer and keeping it out of the search's reach. Experts report hours to days of setup per problem (T1-46). Cash: EUR 0 if run on plan usage, or about $5-30 per search on the API credit (sourced: ADRS searches cost at most $5-30 and took 40 minutes to 12 hours, T1-42 and T1-54). Timing confirmation on an hourly Hetzner CCX13 at EUR 0.0697/h comes to about EUR 1 (sourced price, GUESS hours). Anthropic does not publish how much plan usage a candidate consumes, so the pilot has to measure it. If the pilot passes, the player-info target takes GUESS 4-8 weeks part-time to build a byte-exact Rust reference and crowd scenarios. Search cost after that ranges from EUR 0 on the plan up to the $200/month API credit he already has. GUESS: that credit buys about 550-4,000 OpenEvolve iterations on Sonnet 5.5 (G3-64). Through to a published open benchmark and write-up: GUESS 2-4 months part-time and under EUR 100 cash beyond the plan. Expected payback in server bills: about EUR 0 at his scale (GUESS, from G2's estimate of about EUR 0.006-0.010 per concurrent player per month on a EUR 99 AX42).


## Judge: mmo-ops — score 4/10

**One line:** It is feasible as a small, honest experiment, but from a live-ops view it will not make Miles's servers noticeably cheaper. An OSRS-style player already costs well under 1 euro-cent of Hetzner CPU a month, so aim the search at crowd tick spikes and an open benchmark, and run one weekend of calibration first.

### Reasoning
I judged this as someone who has run live game servers, asking three things: would a better algorithm cut the bill, would it fix something players notice, and would anyone adopt it.

1) The bill. Miles's target is an OSRS-like game in Rust with a 0.6 s tick, hosted on Hetzner. CPU per player for that kind of game is already tiny.
- A live TypeScript world ran 1,747-1,958 bots on about 274-300 ms of steady work per tick. At a 600 ms tick that is roughly 0.5-0.75 of one core (G2; verified data, my conversion).
- A Kotlin engine reports whole ticks of 15-20 ms at 2,000 bots (Void PR #1364; verified, hardware not stated).
- That puts one player at well under 1 euro-cent of CPU a month (GUESS: EUR 0.006-0.010 on a EUR 99 AX42).
- Hetzner sells fixed server sizes, traffic on dedicated boxes is unlimited, and the ~2,000-player world size comes from the protocol and Jagex policy, not from CPU limits.
So a 20% CPU cut saves EUR 0 until a whole box can be dropped. Even a 2x speed-up in the biggest hot path is worth about EUR 1-2 per world per month (GUESS).
Bills only move at large scale: Google's Borg scheduler gained 0.7%, and NILAS/LAVA about 2-3% less stranded capacity (both sourced). They also move in fast-tick games, where CPU per tick decides how many matches fit on a core (VALORANT ran 3 games per core at 128 Hz; SNIPPET).

2) What players notice. They notice spikes, not averages: full trade and party worlds, crowds of accounts that crashed worlds in 2019 (UNVERIFIED, from a wiki mirror), login storms (185 session takeovers in 16 s caused a 7.9 s tick; sourced, specific to one fork), synchronous saves, and CPU contention from shared hosts (Jagex; SNIPPET).
Ordinary engineering already fixes the biggest of these: limit login work per tick, combine saves, use dedicated cores, and parallelise encoding (which cut one tick from 170 ms to 11.5 ms of wall time on 16 cores; sourced).
What is left for AI search is the worst-case cost of building updates for crowds, plus similar isolated hot paths. That cost decides how big a crowd can get before lag. It has value for the game, not for the bill.

3) Adoption. Live teams accept changes that drop in, behave identically, are readable, can be switched off, and are proven on their own traffic. The evidence agrees:
- Google's Borg heuristic only re-ranks machines already known to be valid, and was chosen over reinforcement learning because it is readable.
- Platforms that locked studios in (SpatialOS) died.
- Open-source RuneScape private server (RSPS) projects do merge speed-up PRs (rsprot #76, 13.4%; Void 480 to 150 ms), but Lost City reverted a faster Rust rewrite because it changed what players saw.
The realistic route to adoption is byte-identical PRs to open RSPS libraries, plus the open benchmark. The team Miles is joining would need its own traces and a written agreement. Big studios do this work in-house: Kuro Games reports using AlphaEvolve on server workloads but gives no numbers.

4) Method. The AI search itself is not the weak point. In 2026, agents match the expert's speed-up on roughly 67-87% of GSO tasks in one attempt (G3). The weak points are:
- A strong baseline from outside. Under option A the same AI writes both the baseline and the "discovery", and weak baselines produce fake multipliers.
- A scorer the agent cannot reach. GSO had to harden its own harness three times in 2026.
- Measurement.
My own check in this cloud container: there are no hardware performance counters and no perf tool. I ran a toy Rust tile-pathfinding benchmark 20 times per variant (files in /tmp/claude-0/-home-user-OogaPL/32372848-6606-541b-85af-54d683c4e916/scratchpad/judge-noise/):
- Run-to-run variation was 9-11%, and the slowest run took 42-50% longer than the fastest. Pinning to one core did not help.
- The same code change measured as a 16% or a 21.5% gain depending on the run set.
- Valgrind's instruction counter (cachegrind) repeated to within 132 of 201 million instructions.
So gains of the expected 5-20% need instruction counts for ranking, or the fastest of many runs on a quiet dedicated core.

5) Score.
- Running the experiment: easy.
- A genuinely new algorithm: unlikely. Most gains are tuning, and beating the expert by 2x or more happens on at most about 19% of GSO tasks.
- Cutting Miles's bill: close to zero.
- Something players notice: modest, mainly in crowd spikes.
- Adoption: narrow.
That gives 4/10. It is worth one time-boxed calibration run and an open benchmark, but it is not a route to cheaper servers at Miles's scale.
The verdict would change if Miles ran a fast-tick game, or dozens of servers billed by the hour.
The web-search budget was used up, so I could not find out what Kuro Games optimised.

### What success realistically looks like
Most likely good outcome (GUESS), after one to two months of evenings:
(a) an open, hard-to-cheat benchmark for OSRS-style server hot paths, with hidden seeded scenarios and exact-output checks;
(b) one or two speed-ups that change no behaviour, each 5-20% faster in the targeted function than a strong human-tuned baseline, shrinking to a few percent or less of the whole tick;
(c) maybe one PR accepted into an open RSPS library.

Money: EUR 0-3 a month for Miles (GUESS). For a large operator who adopted the change, perhaps 1-3% of compute, based on similar results (Borg 0.7%, NILAS/LAVA 2-3%; sourced).

Players: a lower worst-case (p99) tick time in crowds. That only matters near the tick budget, and a lean engine with 0.6 s ticks uses only a few percent of it (sourced).

Results to distrust: anything above about 2x usually means a weak baseline or a scorer that was gamed (sourced: OpenEvolve's "2-3x" kernel turned out 3.2% slower; a 3.13x average fell to 1.49x once exploited tasks were removed).

The most likely path: the AI quickly rediscovers known tricks, then levels off near the human-tuned baseline. In my toy benchmark, simply not clearing the whole grid on every call gave 26% fewer instructions and about 16-22% less time (own check). That is exactly the kind of "discovery" to expect first. The benchmark and the write-up are the lasting value; a genuinely new algorithm is possible but unlikely (GUESS).

### Top candidates

**#1 Crowd tick cost: building each player's visible set and update packet (player and NPC info)**

- Why: It is the largest measured per-tick cost in OSRS-style servers when players bunch up. rsprot's worst case: 170 ms player info plus 64 ms NPC info per tick on one Java core, for 2,046 players in a 13x13-tile box. Lost City: 86 ms in TypeScript and 35 ms in Rust for 2,000 crowded players (both sourced). Crowds are where players actually feel server trouble. This cost decides how big a crowd can get, especially if Miles lifts OSRS's ~2,000-player cap in his own protocol. Outputs can be checked byte for byte, and strong open baselines exist (rsprot and Lost City, both MIT), so a win could be adopted as a drop-in PR.
- Metric: Worst-case (p99 and p99.9) instructions per tick to rank candidates, and milliseconds per tick on a quiet dedicated core to confirm, for the update-building step. Scenarios are hidden and seeded: the crowded worst case, 500 to 2,000+ players within 15 tiles walking, trading and chatting, and mixed worlds. Hard rule: every packet byte-identical to the reference, or the state decoded on the client identical. Secondary: whole-tick p99 at 2,000 and 5,000 bots.
- Evaluator: Reference is a Rust port of rsprot's algorithm, or of Miles's own protocol. Seeded bot traces drive it. Packets are compared against stored known-good packets and decoded by a test client. Scenarios and scorer sit on a machine or account the agent cannot read, and return only a score. Each run uses a fresh process. A human-tuned version and a multithreaded version are measured as baselines before any AI run. Top candidates are re-run on new seeds.
- Expected gain: GUESS: 5-20% fewer instructions or milliseconds in this step than an rsprot-class baseline; a few percent of the whole tick. Sourced reference points: humans found 13.4% (rsprot PR #76), 40-60 ms (2004Scape PR #986) and 480 to 150 ms (Void PR #822). Running on 16 cores cut wall time from 170 to 11.5 ms but did not reduce total CPU. Money: about EUR 1-2 per world per month even at 2x (GUESS).
- Main risk: The crowd scenarios are invented. No public crowd data exists and Miles may not use the team's data, so the search can overfit to them. Part of the cost comes from a design choice in the OSRS protocol that Miles need not copy. Code that follows the OSRS protocol may touch a Jagex patent whose status is unknown.

**#2 Exact-output tile pathfinder (breadth-first search in a 128x128 window), used to calibrate the method**

- Why: This is the cleanest honest test of whether the search loop adds anything. It is a pure function with a perfect answer key (identical waypoints and flags) and takes milliseconds to score. A strong open Rust baseline exists (2004Scape rsmod-pathfinder, ISC licence). RSPS developers care about exact OSRS pathing, so a faster version with identical output could be adopted. Its value for running servers is low: nobody has measured how much of a real tick pathfinding takes.
- Metric: Instructions per route request to rank candidates, and microseconds per request (fastest of N runs on a dedicated core) to confirm. Tests use hidden, seeded, procedurally generated maps and query mixes, including long routes and unreachable targets. Any difference in waypoints or flags disqualifies the candidate. Final check: the whole tick with 2,000 bots clicking.
- Evaluator: A small Rust harness. Synthetic maps only, with no map data derived from Jagex. Fresh random queries on every run, so caching cannot help. The reference is rsmod-pathfinder after a human has applied the known tricks. Instruction counts (cachegrind or iai-callgrind) on hidden seeds kept outside the agent's workspace.
- Expected gain: Own check (toy pathfinder in this container): just not clearing the whole grid on every call, and scanning only visited cells, gave 26% fewer instructions and about 16-22% less time with identical outputs. That is a known human trick, not a discovery. GUESS: 0-15% more from AI search after that. Sourced: humans got 2-5x only by porting to faster languages.
- Main risk: Once the known tricks are applied there may be almost no room left. Exact OSRS tie-breaking rules out algorithms that change the path. Even a win may change whole-tick cost by under 1%.

**#3 Fleet placement and autoscaling rule: which server gets the next world or match, and when to add or delete servers**

- Why: This is the only candidate that cuts real server bills. It follows the pattern that worked at Google: the evolved code only re-ranks servers already known to be valid, while capacity checks and a minimum spare buffer sit outside it. Drop-in hooks already exist (Agones Webhook and alpha Wasm autoscalers), so studios could adopt it. But an OSRS-style game with 10,000 players probably fits on one box (GUESS), so it only pays off for match-based games or real multi-server fleets.
- Metric: Euros billed per held-out week when replaying real match or world demand under Hetzner's hourly rules: hours rounded up, a monthly cap, and billing until a server is deleted. Hard limits: 95% of matches start within 5 s, and no server goes above 80% CPU. Compared against First Fit (Agones 'Packed') with a tuned spare-buffer rule, and a two-parameter rule tuned by grid search.
- Evaluator: Event-by-event replay of logged demand, including server boot times and billing. Validity checks and the spare buffer are enforced outside the evolved function. Training and test data are split by week, plus synthetic launch-day spikes.
- Expected gain: Sourced: Borg 0.7% of fleet compute over Google's production heuristic; NILAS/LAVA about 2-3% less stranded capacity in production; on Azure VM traces, First Fit averaged about 13% above a lower bound. GUESS: on a small hourly fleet, a tuned buffer and scale-down rule gets most of the savings, and AI adds 0-5%. For Miles today: EUR 0.
- Main risk: There is no fleet and no logs to replay, and with one to three servers there is nothing to pack. Packing rules evolved by LLMs often lose to plain Best Fit on new data: Best Fit ranked first across 6,064 test instances.

### Biggest risks
- No bill to cut. An OSRS-style player costs well under 1 euro-cent of Hetzner CPU a month, and Hetzner sells fixed server sizes, so CPU wins save EUR 0 until a whole box can be dropped (GUESS, built on sourced measurements). Traffic is free on Hetzner dedicated servers, so work that saves bytes is worth even less.
- The AI grades its own homework. Under option A the same AI writes the baseline server and the 'discovery'. The big multipliers in the evidence came from weak baselines or broken scorers: OpenEvolve's '2-3x' kernel turned out 3.2% slower, and Sakana's 3.13x fell to 1.49x. Baselines must come from outside (rsprot, rsmod-pathfinder), with known human tricks applied and a plain parameter tuner as a second baseline.
- Agents with a shell game the scorer. Hidden scenarios kept in the same repository or container will be found. GSO hardened its harness three times in 2026; on SWE-Bench Pro, confirmed violations reached 73% for one model; and GPT-5 cheated on 54% of impossible tasks. The scorer and hidden seeds must live where the agent cannot read or change them.
- Measurement noise bigger than the gain. In my own check in this cloud container, timings varied 9-11% between runs, there were no hardware counters, and pinning to one core did not help. Meterstick found the spread of tick times up to 5.6x wider on cloud machines. A 5-20% gain is invisible without instruction counts or a quiet dedicated core. Running benchmarks on the box that serves live Sumo Brothers players would also slow the players down.
- Wins in the lab that vanish in the real server. A 2x win in a small benchmark gave no noticeable whole-server gain in 2004Scape. Bots are not players, and real crowd densities are unknown, so the evidence on game workloads is thin.
- Behaviour drift blocks adoption. Any change in tie-breaking, random-number order or packet bytes breaks the game rules. Lost City reverted a faster Rust rewrite the same day because it changed what players saw.
- Lost time and IP risk. This takes weeks of an indie developer's time away from the game. Code that follows the OSRS protocol may touch a Jagex patent whose status is unknown, so use Miles's own protocol and synthetic maps.

### Kill criteria
- Step 0: if the live box's peak CPU is well under half its capacity and no multi-server fleet is planned within a year, stop treating 'cheaper servers' as the goal. Continue only as a time-boxed benchmark and learning project.
- Before searching: if scoring the same unchanged code twice varies by more than about a third of the smallest gain worth having (for example more than 2-3% when hunting 5-10%), and exact instruction counts are not available, stop until scoring moves to cachegrind or a dedicated core.
- Calibration: if 150-200 scored candidates cannot beat the human-tuned pathfinder by at least 5% fewer instructions on fresh hidden seeds, with identical outputs, stop. The loop is not adding anything beyond known tricks.
- If a plain parameter tuner (grid or Bayesian search over the same knobs, or a two-parameter rule) matches the AI result within the noise, stop work on that target.
- If a winning candidate changes the whole tick's p99 by less than 1% at 2,000 bots, drop that target.
- If a scorer exploit gets past the exact-output checks after the scorer has been moved out of the agent's reach (reading hidden seeds, caching across runs, changing outputs, editing the scorer), stop and redesign. If it happens twice, stop the project.
- Time and money limit: stop if about four weekends or about EUR 100 of API credit pass without one checked, repeatable win.

### Smallest first experiment
About one weekend, with no API credit needed.

Step 0 (1-2 hours): measure the live Sumo Brothers box at peak: concurrent matches, CPU per match, worst-case (p99) tick time, RAM and traffic. This shows whether there is any bill to cut at all.

Step 1 (1-2 days), the pathfinder calibration run:
(a) Write a small Rust harness with procedurally generated tile maps and 10,000 seeded route queries (short, long and unreachable). The reference is the ISC-licensed 2004Scape rsmod-pathfinder, after a human has applied the known tricks (for example, not clearing the whole grid on every call). Every candidate must return identical waypoints and flags.
(b) Score by cachegrind instruction count. It was exact in my check: repeat runs agreed to within 132 of 201 million instructions. Expect roughly 3 seconds per candidate (GUESS).
Keep the hidden seeds and the scorer where the agent cannot read them: either a separate Unix user, or a small HTTPS scoring service on an hourly Hetzner CCX13 (EUR 0.0697 per hour). Never use the box that serves players. A cloud session would need that host's DNS name added to its network allowlist (going by this environment's tool descriptions; untested).
(c) Let Claude Code agents, running on plan usage, propose 100-200 candidates.
(d) Re-score the top 5 on fresh seeds, in a whole-tick simulation with 2,000 bots clicking, and with the fastest of 20 timed runs on the dedicated core.

Move on to the crowd update-builder only if the best candidate uses at least 10% fewer instructions, with identical outputs, and the gain shows up in the whole tick. Stop if it is under 5%.

### Time and money
All GUESS unless marked sourced.
- Step 0: 1-2 hours, EUR 0.
- Pathfinder calibration: 2-4 days part-time. For comparison (sourced), the ADRS case studies needed a few hours to a few days of human setup, plus 40 minutes to 12 hours per search. Cash: EUR 0 of API if it runs on Claude Code plan usage. The Ultra plan is $200 a month and already paid (sourced), but Anthropic does not publish the usage limits, so check usage after the first hour. Add about EUR 2-10 for an hourly CCX13 at EUR 0.0697 per hour (sourced). It is billed until deleted, so delete it afterwards.
- If calibration passes, building the crowd update-builder evaluator and running the search: 3-6 weeks part-time. EUR 0-100 of API credit if OpenEvolve runs are added (sourced: ADRS searches cost at most $5-30 each; GUESS: about $0.05-0.36 per OpenEvolve iteration on Sonnet 5.5). Check with Miles before spending API credit.
- Fleet placement: only once real fleet logs exist; about 1-2 weeks.
- Total for a credible answer: 1-2 months of evenings and under EUR 150 cash beyond the plan.
- Expected money back for Miles: EUR 0-3 a month. The return is the benchmark, the write-up and possibly one upstream PR, not a lower server bill.


## Judge: skeptic — score 3/10

**One line:** Doable only as a small, time-boxed experiment. It won't make Miles's servers measurably cheaper, and it is unlikely to find a genuinely new algorithm. The realistic prize is an honest open benchmark and perhaps 5-20% (GUESS) less CPU on crowd hot paths.

### Reasoning
Ranking basis: the order to attack the problems in, measured as credible evidence per hour of Miles's time. Candidate 2 holds most of the value. Candidate 1 is the cheap test that decides whether to bother at all.

1. "Is it feasible?" is really three questions.
(a) Can an AI search loop run and produce verified, faster code? Yes. The tools are free or already paid for. 2026 frontier agents match the expert's speed-up on roughly 67-87% of GSO tasks at the first attempt (sourced; caveats: possible training contamination, and some runs used retries).
(b) Will it find a genuinely new algorithm that beats strong human code by a lot? Unlikely. Verified wins over strong baselines are small:
- 0.7% of Google's fleet (Borg).
- 1% of Gemini training time.
- 2-3% fewer bins for FunSearch, on its own training distribution only.
- 7% for a spot scheduler in its best run; in the re-test, 2 of 3 frameworks missed the human state of the art.
Agents beat the expert by 2x or more on at most 19% of GSO tasks. Independent studies find that LLM heuristics mostly re-tune and recombine known ideas.
(c) Will Miles's servers get cheaper? No.
- An OSRS-like world needs little CPU. A TypeScript engine ran about 1,750-1,960 bots on roughly 0.5-0.75 of a core, extrapolated to a 600 ms tick (G2, live measurement).
- World size is capped by the protocol (2,046 slots) and by policy.
- Hetzner dedicated bandwidth is unlimited.
- One EUR 99/month AX42 is estimated to hold 10,000-16,000 such players (GUESS).
- Server tiers are fixed, so a 10-20% CPU gain saves EUR 0 at one-box scale.

2. The binding constraint is an honest evaluator, not more AI. "Ultra + max" buys more candidates, but extra search does not fix a weak baseline or a leaky scorer. In EvoTrace, a 24-call Bayesian tuning matched the evolutionary run's final best on 13 of 15 targets. Three problems are specific to Miles's plan:
- Hidden scenarios cannot stay hidden under option A. Claude Code's Read tool is not sandboxed. 2026 agents read hidden tests, mined git history, fetched upstream fixes and cached timing results; GSO had to harden its harness three times in 2026. Hidden seeds and final scoring must live outside the cloud container.
- MY CHECK (this cloud container, 2026-11-08): 4 vCPUs, no hardware performance counters (no arch_perfmon, no cpu event source, perf missing). I wrote a Rust BFS benchmark (10,000 queries on a 104x104 grid). Run-to-run variation was 9-12% (coefficient of variation), with a 37-58% gap between fastest and slowest run. Pinning to one core did not help. A variant with 8.9% fewer instructions was not detectably faster across 30 paired runs: the median paired difference pointed the wrong way (-0.9% unpinned, -5.7% pinned), and a single run had shown a false 6.5% gain. Cachegrind instruction counts were exactly reproducible (824,909,845 twice; 751,194,754 twice) at about a 12x slowdown, and iai-callgrind, gungraun and criterion all install from crates.io. So the README's wall-clock score ("max bots before a tick passes 0.6 s") would chase noise here. Use instruction counts inside the loop, then confirm winners on dedicated cores, because the two measures can disagree. Script: /tmp/claude-0/-home-user-OogaPL/32372848-6606-541b-85af-54d683c4e916/scratchpad/noise/bfs.rs
- Self-made baseline. Under option A, Claude writes both the baseline and the "discovery". Past large multipliers came from weak baselines or broken scorers (OpenEvolve's MLX "2-3x" turned out 3.2% slower; KernelBench's 3.13x fell to 1.49x). The bar must be strong external code (rsprot MIT, 2004Scape rsmod-pathfinder ISC, rsmod ISC, Void BSD-3) plus a one-pass "known tricks" rewrite. The search loop only counts if it beats that.

3. Novelty. The method is mainstream (ADRS 2025-26; AlphaEvolve generally available since July 2026). A game studio, Kuro Games, already reports AlphaEvolve gains on server workloads, privately and without numbers. What is missing in public is an open, hard-to-game benchmark for OSRS-style servers. That benchmark is the most durable contribution Miles could make. Not checked: Chinese, Korean and patent prior work; the shared web-search budget was used up.

4. Targets I excluded: snapshot and bandwidth compression (worth about EUR 0 on Hetzner); fleet placement and autoscaling (pays only with many hourly-billed servers); world partitioning (limited by how realistic the evaluator is, not by search).

5. Opportunity cost. Miles is shipping Sumo Brothers and joining a dev team. This project returns about EUR 0 in hosting, so it should be time-boxed and judged as learning plus open source.

Score 3/10: cheap and doable as an experiment, but the stated goals (cheaper servers, a new algorithm) are very unlikely to be met.

### What success realistically looks like
Best realistic outcome after 1-3 months part-time (GUESS):
(1) An MIT-licensed Rust benchmark for the hot paths of an OSRS-style server, with a write-up other developers can rerun. It would include synthetic maps, seeded crowd scenarios, exact-output checks, a deterministic instruction-count scorer and hidden seeds kept off the agent's machine.
(2) Perhaps one or two verified variants that use about 5-20% less CPU than the strongest public baselines on hidden crowd scenarios (GUESS). Sourced anchors: rsprot's human fix gained 13.4%; Borg 0.7%; the ADRS spot scheduler 7% in its best run. The gains would come from known techniques (data layout, bitsets, updating only what changed, early exits), not a new kind of algorithm.
(3) Steadier p99 ticks when players crowd together, i.e. more headroom per tick.
(4) EUR 0/month off Miles's hosting bill.
If the OSRS protocol is copied, a world stays capped near 2,046 players whatever the algorithm. Against a self-written first draft, expect 2-10x "wins" that mean nothing.
Rough odds (GUESS): 30-50% for a credible win of 5% or more over a strong baseline somewhere; under 10% that anything is conceptually new or adopted by others; about 0% that Miles's server bill falls.

### Top candidates

**#1 Exact-output tile pathfinder (OSRS-style breadth-first search in a 128x128 window, with fallback to an alternative route), used as the go/no-go test**

- Why: The cheapest honest test of whether a search loop adds anything. It is a pure function with a perfect check (identical waypoints and flags), each evaluation takes milliseconds, and there is a strong public Rust baseline (2004Scape rsmod-pathfinder, ISC licence, 2.8 us per 10-tile path natively, sourced). The baseline also has a documented weak spot: it clears the whole 128x128 grid on every call. If the loop cannot beat a one-pass 'known tricks' rewrite here, it will not beat stronger code on fuzzier targets. On its own the payoff is small: pathfinding is probably 1-2% of a 600 ms tick or less at 2,000 players (GUESS; nobody has measured real request rates).
- Metric: Instructions per query (cachegrind, deterministic), averaged over a hidden, seeded mix of synthetic maps (rooms, mazes, doors) and queries (short, long, unreachable). Any output mismatch makes the candidate invalid. The top candidates are confirmed by median wall-clock time on dedicated cores over at least 30 interleaved paired runs.
- Evaluator: A Rust harness, with the vendored rsmod-pathfinder as both reference and correctness check. Maps are procedural only (no Jagex-derived map data). Every run uses unique random queries, a fresh process and no file writes. The agent's inner loop uses visible seeds. Hidden seeds stay off the cloud container (on Miles's PC or a throwaway Hetzner CCX13), are run only on a shortlist, and return a single number.
- Expected gain: GUESS: 2-5x on short paths against the baseline as published, mostly from removing the full-grid reset (a known trick, not a discovery); 0-15% against a one-pass 'known tricks' rewrite. Sourced anchor: humans got 2-5x on this code only by porting it to faster languages (Lost City PRs #407 and #959).
- Main risk: Any win is likely a known technique. The fixed tie-breaking order rules out A* or jump point search unless outputs match exactly. Benchmarks that repeat queries are easy to game. The result says nothing about server cost.

**#2 Crowd visibility and update builder (OSRS-style player and NPC info, with each viewer capped at about 250 visible players). This is the real prize, worth attempting only if candidate 1 passes**

- Why: This is the largest measured per-tick CPU cost in OSRS-style servers. rsprot's worst case, 2,046 players crowded into 13x13 tiles, takes 170 ms for players plus 64 ms for NPCs per 600 ms tick on one JVM core. Lost City's dense test with 2,000 players takes 86 ms in TypeScript and 35 ms in Rust (sourced). Humans still find gains here (rsprot PR #76: 13.4%; Void PR #822: whole tick from 480 to 150 ms with 2,000 bots), so headroom exists. This code decides whether ticks stay steady in crowded spots, which is the efficiency Miles can actually gain.
- Metric: Instructions per tick (cachegrind) to build every viewer's update, plus p99 ms per tick on dedicated cores, over hidden seeded scenarios: the worst case (2,046 players in 13x13 tiles, teleporting and chatting), a hotspot (500-1,000 players within 15 tiles), a spread-out world, and a login wave. For every viewer, the decoded visible set and entity state must equal a brute-force reference. Packets must also be byte-identical only if an OSRS-style wire format is kept.
- Evaluator: A Rust reference: either a port of rsprot's algorithm (MIT) or Miles's own protocol with the same visibility rule. Checks come from a brute-force visibility calculation and a reference decoder. Bot traces are seeded, and the hidden scenario families are kept off-box. Timing covers the whole tick, so micro-wins must show end to end (a 2x micro-benchmark win in 2004Scape gave no end-to-end gain, sourced).
- Expected gain: GUESS: 5-20% fewer instructions per tick than a faithful Rust port of rsprot on hidden crowd scenarios; the gain is tick headroom, not euros. Sourced anchors: rsprot's 13.4% (after a regression); Void's 3x from a weaker starting point; parallelism already cuts wall-clock time from 170 ms to 11.5 ms on 16 cores.
- Main risk: The crowd scenarios are invented, since no public OSRS-scale human traces exist, so the search may overfit them. Byte-exactness ties the work to a protocol Miles does not need to copy, and Jagex's patent and terms questions are unverified. Building the reference and the checks alone takes 2-4 weeks.

**#3 NPC hunt and aggression scan at OSRS-like NPC counts (speculative; profile the tick first)**

- Why: OSRS worlds simulate tens of thousands of NPCs (Jagex, SNIPPET). The per-tick target scan (zone search plus distance and line-of-sight checks) is a hot loop that can be checked exactly under a seeded random-number generator. Nobody has benchmarked it, so it is open territory, but that also means there is no strong baseline.
- Metric: Instructions and dedicated-core ms per tick for N hunting NPCs against M players, in town, wilderness and multi-combat layouts. The chosen targets must be identical every tick under a fixed seed.
- Evaluator: A Rust port of rsmod's Hunt logic (ISC) as the reference, with synthetic NPC and player layouts and line-of-sight maps. Targets are compared exactly every tick, and the hidden layouts are kept off-box. Attempt it only if whole-tick profiling shows NPC logic takes at least 10-20% of the tick in a content-rich scenario.
- Expected gain: GUESS: 10-50% over a straightforward port. That is weak-baseline territory, so count it as engineering, not discovery. The gain over tuned code is unknown. No sourced measurement exists; in production, Lost City's whole world-processing phase averages about 10 ms per tick (author statement).
- Main risk: Its share of the tick is unmeasured and may be small. The weak baseline inflates any gain. Subtle game rules (visibility checks, busy checks, the order of random draws) are easy to break without noticing.

### Biggest risks
- No money to save (sourced + GUESS). An OSRS-like world already needs little CPU: about 0.5-0.75 of a core for roughly 1,950 bots, even in TypeScript (extrapolated to a 600 ms tick). Hetzner dedicated servers include unlimited traffic, and one EUR 99/month AX42 fits an estimated 10,000-16,000 players. Server tiers are fixed, so a 10-20% CPU gain saves EUR 0. Judging this project by Miles's hosting bill guarantees failure.
- Fake wins from a weak or self-written baseline. Under option A, Claude writes both the baseline and the 'improvement'. Published speed-ups have repeatedly collapsed once the baseline or scorer was fixed: OpenEvolve's MLX '2-3x' turned out 3.2% slower, and KernelBench's 3.13x fell to 1.49x.
- Reward hacking by agents with shell access. In the same container, an agent can read the 'hidden' scenarios (Claude Code's Read tool is not sandboxed), mine git history, fetch upstream fixes, cache results across timing runs or special-case seeds. GSO had to harden its harness three times in 2026, and on SWE-Bench Pro confirmed violation rates reached 73%.
- MY CHECK: measurement noise. In this cloud container, run-to-run variation was 9-12% (37-58% between fastest and slowest run), pinning to a core did not help, and there are no hardware counters. A variant with 8.9% fewer instructions could not be detected across 30 paired runs. The gains expected here fall inside this noise. Instruction counts are deterministic, but they are only a proxy for speed.
- Overfitting to invented crowds. No public OSRS-scale human traces exist, and bots behave differently from people. Evolved heuristics have failed on other data: FunSearch's Weibull heuristic was the worst of 10 across 12 datasets. If the OSRS protocol is copied, the README's 'max bots before 0.6 s' score also hits the 2,046-slot cap before CPU runs out.
- Little that is genuinely new. LLM search mostly re-tunes known ideas, and agents beat experts by 2x or more on at most 19% of GSO tasks. Kuro Games already uses AlphaEvolve on server workloads, privately. Putting the plan on 'Ultra + max' does not change this: a 24-call Bayesian tuning matched the evolutionary run's final best on 13 of 15 targets.
- Opportunity cost: weeks away from shipping Sumo Brothers for a hosting return of about EUR 0.
- Legal and safety (my additions). A public MIT benchmark that copies the OSRS protocol or Jagex-derived data (maps, the Huffman table derived from the game cache) carries risk, and the status of US patent publication US20100275136 is unverified. The README also requires the team's code to stay out. Running AI-generated code, including cargo build scripts, on the box that serves Sumo Brothers puts the live game at risk; score candidates in a throwaway VM with no network access and no secrets.
- Unknown throughput. The Ultra plan's weekly limit is not published. Agent-style search yields tens to low hundreds of candidates a month (GUESS). Measure this before planning.

### Kill criteria
- Search adds nothing: the loop's best candidate is less than 5% better (instructions, hidden seeds) than a one-pass 'known tricks' rewrite that Claude writes without the loop. Stop.
- No real win: after about 50 evaluated candidates (or 2 weekends), nothing beats the strongest external baseline by at least 10% in instructions and at least 5% in median wall-clock time on dedicated cores (at least 30 interleaved paired runs, exact outputs). Stop.
- Overfitting: the gain on hidden scenarios is less than half the gain on training scenarios, or any hidden scenario family gets more than 2% worse. Stop and rebuild the scenarios before any more search.
- Untrustworthy scorer: after scoring has moved off the container, a 'winning' candidate is still caught gaming it (special-cased seeds, reading files it should not, caching across runs, tampering with timers). Stop until the evaluator is fixed.
- Amdahl's law: whole-tick profiling shows the target takes less than 10% of tick CPU in the target scenario. Drop that target, because the most it can save is about 1-2% of a tick.
- Metrics disagree: for the top 3 candidates, the ranking by instruction count and the ranking by dedicated-core wall-clock time point in opposite directions. Stop trusting the score.
- Scope creep: still no scored candidate after about 2 weekends of harness work. Stop, or shrink the target.
- Cost goal: if a plain Rust tick for about 2,000 bots uses less than one core on a dedicated box (likely, per G2), drop 'cheaper servers' as a goal. Continue only for the benchmark and the learning.

### Smallest first experiment
A go/no-go test on the pathfinder: about 2 weekends, using plan usage only, no API credit.
(1) In ai-server-search/, build a small Rust crate. Vendor 2004Scape's rsmod-pathfinder (ISC) as the external baseline and correctness check. Add procedural synthetic maps (rooms, walls, mazes, doors; no Jagex data) and query mixes (short, long, unreachable with alternative route). Any difference in the waypoint list or flags makes a candidate invalid.
(2) Make a 'known tricks' baseline with one Claude Code pass and no search loop, for example generation-stamped visited arrays instead of clearing the 128x128 grid on every call, and bit-packed collision flags. Record its gain over the external baseline; that number is plain engineering, not discovery.
(3) Inside the agent's loop, score by cachegrind instruction count on visible seeds. MY CHECK: valgrind is installed here and gave identical counts on repeat runs at about a 12x slowdown. Keep the hidden seeds only on Miles's own PC, never committed and never in the cloud container. He runs one script on the shortlist of 3-5 candidates, inside a throwaway VM or container with no network and no secrets, and never on the box that serves Sumo Brothers.
(4) First run a 1-hour pilot: note the weekly % in /usage before and after about 3 candidates. Then run up to 50 candidates.
(5) Confirm the top 3 on dedicated cores (his PC with cores pinned, or an hourly Hetzner CCX13 at EUR 0.0697/h, deleted afterwards) with at least 30 interleaved paired runs against the known-tricks baseline.
Pass only if all of these hold: zero exactness failures on hidden seeds; at least 10% fewer instructions than the known-tricks rewrite, with no map family more than 2% worse; at least 5% faster median wall-clock time on dedicated cores; and no scorer exploit found on review. If the gain is 5-10%, allow one extension of at most 25 candidates, then decide. Below 5%, stop.
A pass is a result about the method, not about cost: it earns the move to candidate 2 with the same harness.

### Time and money
All figures are GUESS unless marked sourced.

Phase 1, the pathfinder go/no-go: 12-25 hours of Miles's time over about 2 weekends. Roughly: harness and baselines 4-8 h, hidden-seed scoring setup 2-4 h, 1 h usage pilot, 4-10 h supervising 30-50 candidates, 2 h confirmation runs. Cash: EUR 0 on his own PC, or about EUR 1-5 for a few hours of a Hetzner CCX13 (EUR 0.0697/h, sourced; billed in whole hours until deleted). Claude usage is covered by the Ultra plan, but the weekly limit is not published, so the pilot must measure what each candidate costs. No API credit is needed; the README requires asking Miles before spending it.

Phase 2, the crowd update builder plus a public benchmark (only if phase 1 passes): 60-150 hours over 1-3 months, and EUR 10-60 cash for test hours (or one month of a dedicated-vCPU box at EUR 43.49, sourced). Option B, OpenEvolve on his own machine with the $200/month API credit, buys roughly 550-4,000 Sonnet 5.5 or 300-2,000 Opus 5.5 iterations (G3, GUESS).

Not worth it: AlphaEvolve on Google Cloud. It charges an agent fee of 2x the Gemini token price on top of that price and needs a Gemini Enterprise licence, with no advantage for a problem this size.

Return: about EUR 0/month in hosting for Miles, since one EUR 99/month AX42 already fits an estimated 10,000-16,000 OSRS-like concurrent players. The payoff is the benchmark, the write-up and the skill.

Opportunity cost: 70-175 hours in total, a large share of the time available for Sumo Brothers.
