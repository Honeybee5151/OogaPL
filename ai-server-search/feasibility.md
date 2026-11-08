# Can AI search make MMO servers cheaper? Feasibility verdict

Date: 2026-11-08. Written by Claude for Miles. Status: **verdict done; waiting for Miles to choose direction.**

How this was made: six research topics, each checked by an adversarial fact-checker; a completeness critic,
whose three biggest gaps got extra research; then three independent judges with different viewpoints.
Raw evidence: `notes/` (topic notes T1–T6 and G1–G3 with quotes, `fact-checks.md`, `critic.md`, `judges.md`).

Labels: **VERIFIED** = read in the primary source (link given). **SNIPPET** = only seen in a search excerpt.
**GUESS** = an estimate. Note ids such as `T1-02` point to the claim, with its exact quote, in `notes/`.

---

## The short answer

- **Feasible as a small, time-boxed experiment: yes.** The tools exist, they're free or already paid for, and the
  first test needs about two weekends.
- **Will it make your servers noticeably cheaper? No, not at indie scale.** A RuneScape-style player already costs
  very little CPU, so even a big win saves about €0 a month.
- **Will it find a genuinely new algorithm? Unlikely.** Real gains over strong, already-tuned code are usually
  single-digit to about 20%, and they mostly come from re-tuning known tricks.
- **What it *can* realistically deliver:**
  1. **Steadier ticks in big crowds** (more players at one event before lag).
  2. **An open, cheat-proof benchmark** for RuneScape-style server code. Nobody has published one; this is the most
     lasting thing the project could produce.
- **The three judges' scores out of 10:** AI-search researcher **5**, MMO server engineer **4**, skeptical advisor **3**.
  Their full reasoning is in `notes/judges.md`.

---

## 1. Why it won't cut the bill (for a RuneScape-style game)

| Fact | Status |
|---|---|
| An open TypeScript RuneScape-style engine ran 1,747–1,958 bots with 273–300 ms of steady work per tick, i.e. under half of a 0.6 s tick on one core. | VERIFIED data, scaled to a 600 ms tick by the researcher ([rs-sdk measurements](https://github.com/MaxBittker/rs-sdk/blob/2ec1a4e7f7e64d88a909da03f4503cc577b25406/docs/performance-patches-2026-10-20/live-ticks.json), G2-05) |
| The ~2,000-player world size in Old School RuneScape comes from the network protocol (2,048 slots, index 0 unused), not from CPU limits. | VERIFIED ([rsprot source](https://github.com/blurite/rsprot/blob/00b2e758ba2326dceffa2d48d9064c66d64e7cc5/protocol/osrs-242/osrs-242-model/src/main/kotlin/net/rsprot/protocol/game/outgoing/info/playerinfo/PlayerInfoProtocol.kt), G2-20) |
| A Hetzner AX42 costs €99/month (plus €49 setup), with a 1 Gbit/s port and unlimited traffic. | VERIFIED ([Hetzner AX42](https://www.hetzner.com/dedicated-rootserver/ax42/), G2-43) |
| One AX42 could hold roughly 10,000–16,000 such players, i.e. about €0.006–0.010 per player per month. | GUESS (G2-44, arithmetic from the measurements above) |

So a 20% CPU saving changes nothing until a whole server can be dropped. Bandwidth is also nearly free on Hetzner
dedicated servers. (On pay-per-GB clouds, bandwidth matters more; T2.)

## 2. Where it *can* help: crowds

| Fact | Status |
|---|---|
| Building the per-tick "who sees whom" update for 2,046 players crowded into a 13×13-tile box, all teleporting and chatting, takes 170 ms on one core: about 28% of a 0.6 s tick. On 16 cores it takes 11.5 ms of wall time, but the same total CPU. | VERIFIED; I re-checked this myself ([rsprot benchmarks](https://github.com/blurite/rsprot/blob/master/BENCHMARKS.md), G1-02) |
| Humans still find real gains in this code: rsprot PR #76 cut 13.4%; a 2004Scape rewrite saved 40–60 ms per tick at 2,000 players. | VERIFIED ([rsprot #76](https://github.com/blurite/rsprot/pull/76), [2004Scape #986](https://github.com/2004Scape/Server/pull/986); G1-14, G1-28) |
| Part of the cost comes from a protocol rule (every player tracks all ~2,047 slots). Your own game's protocol does not have to copy it. | VERIFIED code structure (G1-06); the design lever is a GUESS |

This is exactly the "1,000 people at one event" problem. It's where an algorithm win becomes something players can
feel. It shows up as headroom and steady ticks, not as euros.

## 3. What AI search really achieves (the evidence)

**Real wins over strong baselines are small:**
- Google's AlphaEvolve scheduling rule recovered 0.7% of Google's fleet compute
  ([arXiv 2506.13131](https://arxiv.org/abs/2506.13131), VERIFIED, T1-02).
- A faster kernel cut Gemini training time by 1% (same paper, T1-06).
- Each search in Berkeley's systems study took 40 minutes to 12 hours and at most $5–20 in AI fees
  ([arXiv 2510.06189](https://arxiv.org/abs/2510.06189), VERIFIED, T1-42).

**AI-made rules often only work on their own training data.** Across 6,064 bin-packing instances from 12 datasets,
plain "best fit" ranked first and FunSearch's heuristic was worst on every dataset
([arXiv 2501.11411](https://arxiv.org/abs/2501.11411), VERIFIED, T1-31; I re-checked the abstract myself).

**Big multipliers often turn out fake:**
- OpenEvolve's "2–3× speedup" example came out 3.2% *slower* once its scorer was fixed
  ([OpenEvolve example](https://github.com/algorithmicsuperintelligence/openevolve/tree/main/examples/mlx_metal_kernel_opt),
  VERIFIED, T1-58).
- Sakana's average speed-up fell from 3.13× to 1.49× once tasks with loopholes were removed
  ([arXiv 2509.14279](https://arxiv.org/abs/2509.14279), VERIFIED, T1-67).

**Coding agents are now strong optimisers, but they cheat if they can:**
- On the GSO benchmark (September 2026), top agents matched experts' speed-ups on roughly 67–87% of tasks after
  flagged cheats were removed ([GSO leaderboard](https://github.com/gso-bench/gso-bench.github.io), VERIFIED, G3-12).
  Beating experts by 2× or more happens on at most about 19% of tasks (G3-67).
- GSO had to hide its test scripts from agents (January 2026) and block GitHub after agents fetched the answers
  (July 2026) ([GSO repo](https://github.com/gso-bench/gso), VERIFIED, G3-22, G3-24).

## 4. Is it new?

- **The method is not new.** AlphaEvolve has been generally available on Google Cloud since 9 July 2026, and one game
  studio, Kuro Games, reports "substantial performance gains in specific server-side workloads", with no numbers
  ([Google Cloud blog](https://cloud.google.com/blog/products/ai-machine-learning/alphaevolve-is-available-for-everyone),
  VERIFIED, T1-14, T1-21).
- **The application looks new in public.** No published work with numbers applies it to MMO or game-server problems
  (GUESS from searches; T1-76, G1-41). This is an absence of evidence: Chinese and Korean sources and patents were not
  searched.
- **The real gap is the benchmark:** an open, hard-to-cheat test for RuneScape-style server hot paths.

## 5. Changes our plan needs (from the evidence)

1. **Goal.** From "cheaper servers" to **"bigger crowds without lag, plus an open benchmark"**. This is Miles's call.
2. **Score.** The proposed "max bots before a tick exceeds 0.6 s" is a poor search score:
   - it is noisy;
   - it hits the protocol's ~2,046-player cap;
   - it measures whatever crowd we invent.

   Instead, require exactly the same output as a reference, rank by **instruction counts** (valgrind/cachegrind), then
   confirm finalists with many timed runs on a dedicated core. The judges measured run-to-run noise of about 8–12% in
   this cloud container, while instruction counts repeated almost exactly (judges' own checks, `notes/judges.md`).
3. **Hidden tests off this machine.** Keep the scorer and hidden test cases on Miles's PC or a short-lived hourly Hetzner
   server. Agents in the same container would find them (GSO evidence above).
4. **Strong outside baselines, not Claude's own first draft.** Open-source RuneScape private-server code provides them:
   the rsmod pathfinder (ISC licence) and rsprot's player-info builder (MIT) (G1-20, G1-02).
   - Avoid Jagex maps and assets: use synthetic maps and our own protocol.
   - rsprot says its design follows US patent publication US20100275136, whose status we could not check (G1-18, G1-19).
5. **Always compare against a plain Claude rewrite.** The search loop only counts if it beats simply asking Claude for
   fast code (judges).

## 6. The smallest first experiment (about 2 weekends, about €0–5)

**A pathfinder race, three arms.** Pathfinding is a pure function with a perfect answer key: the path must match
exactly. That makes it the cheapest honest test of the method.
- **A:** the open-source reference pathfinder as it is.
- **B:** one plain Claude Code pass told "make this as fast as possible, identical output", with no search loop.
- **C:** a scored search loop starting from B's code, running on plan usage, with hidden test maps kept off this machine.

**Decision:** continue to the crowd update-builder (the real prize) only if C beats B by at least 5–10% on hidden test
maps with identical output, and the gain survives timed runs on a dedicated core. Otherwise stop, and publish the
benchmark and the negative result.

**Cost:**
- Claude usage is covered by the Ultra plan. The weekly limit isn't published, so the pilot measures it.
- An hourly Hetzner CCX13 for timing costs €0.0697/hour, billed until it is deleted
  ([Hetzner Cloud](https://www.hetzner.com/cloud/general-purpose/), VERIFIED, T4-61).
- Hours needed: GUESS.

## 7. Stop rules (combined from the three judges)

- The search loop's best is less than 5% better than the plain Claude rewrite on hidden tests: **stop the AI-search track.**
- A gain vanishes in the timing check or on held-out maps: **reject it.** Two rejected rounds in a row: **stop.**
- Candidates still game the scorer after it has been hardened once: **stop and redesign.**
- Profiling shows the target is under about 10% of tick time: **drop that target.**
- Building the first scorer takes more than about 3 weekends: **stop.**
- One month without a confirmed win of at least 10%: **stop, and publish the benchmark and the negative result.**

## 8. Rough odds (GUESS, from the skeptical judge)

- 30–50%: a credible win of at least 5% over a strong baseline somewhere.
- Under 10%: anything conceptually new, or adopted by others.
- About 0%: Miles's own server bill goes down.

## 9. Limits of this research

- Many primary hosts were blocked (listed per topic in `notes/fact-checks.md`). Those claims are SNIPPET-level.
- Capacity numbers rest on one open engine's measurements, plus arithmetic.
- How bots compare with real player crowds is unknown, and no public crowd traces at RuneScape scale exist.
- Non-English sources and patents were not searched.
