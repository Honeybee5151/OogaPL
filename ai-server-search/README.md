# AI-driven search for cheaper MMO servers (new idea, 2026-11-08)

Status: **feasibility verdict done (2026-11-08), see `feasibility.md`.** Nothing is built yet;
waiting for Miles to choose the direction. Raw evidence is in `notes/`.

The idea: use AI-driven algorithm search (in the style of DeepMind's AlphaEvolve and FunSearch,
or open-source tools like OpenEvolve) to find new algorithms and rules that make MMO servers
cheaper and more efficient. This follows the tick-lang research in `research/`, which was stopped.

## Decisions and facts from Miles so far

- **Goal:** cheaper MMO servers overall (cost per player), not only for crowded events.
- **Share the results:** open benchmark, scripts and write-ups so other developers can use them.
  A permissive licence (MIT/Apache) is the likely choice; to be decided later.
- **Target game style:** like RuneScape but pixel art. Tile-based movement, a slow server tick
  (Old School RuneScape uses 0.6 s), aiming for thousands of players. The client is light
  (pixel art), so the server is the bottleneck.
- **Miles is joining that game's dev team** (quest design). We do **not** use the team's code or
  data. We build our own original test server and keep the two separate unless the team agrees
  in writing.
- **Test server language:** Rust (if the research says go).
- **Where it lives:** this folder in the OogaPL repo.
- **AI budget:** Miles has the Claude Ultra plan, which includes $200 a month of API credit
  (expires monthly, no rollover; API calls stop when it runs out). Claude Code itself is also
  covered by the plan.
- **How searches run (decided 2026-11-08): option A, inside Claude Code** (agents in Claude Code
  propose and test candidates), using the plan's Claude Code usage. Cloud sessions cannot store a
  key for Anthropic's own API host, so the $200 API credit can only be used from a machine Miles
  controls (his computer or Hetzner server, key in an environment variable there). That is
  option B, kept for later, bigger runs.
- **Check with Miles before the first run that spends API credit.**

## Proposed score (my proposal, not yet tested)

How many bot players one server can hold before a tick takes longer than its time budget
(0.6 s), at a fixed quality bar. This converts directly into players per euro of server.
Hidden test scenarios the AI never sees guard against overfitting to our own simulator.
