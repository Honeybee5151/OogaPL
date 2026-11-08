# T2: What makes MMO and online-game servers expensive, and how big the bills are

Research date: 2026-11-08. Written for Miles (Sumo Brothers on Godot, a Hetzner Linux box; next project: an Old School RuneScape-like pixel-art MMO with a 0.6 s tick and thousands of players, per `/home/user/OogaPL/ai-server-search/README.md`).

Status labels: **VERIFIED** (read in the primary source, with a quote), **VERIFIED_FIG** (read off a chart or table), **SNIPPET** (seen only in a search-engine excerpt because the page was blocked), **UNVERIFIED** (secondary source only, or memory), **WRONG** (a common claim that the primary source contradicts), **GUESS** (my own estimate; arithmetic shown).

Currency: Hetzner prices are in EUR excluding VAT; AWS, GCP and Azure prices are in USD. I do not convert. Any exchange rate between 1.0 and 1.3 USD per EUR leaves every conclusion below unchanged.

---

## 0. Bottom line (plain language)

1. **For Miles's kind of game the binding resource is CPU time per tick.** Old School RuneScape (OSRS) worlds are limited by CPU contention and tick processing time. EVE slows the game clock when a node's CPU is overloaded. Albion runs each map cluster on one thread. Minecraft-like servers are "poorly parallelized" and top out at roughly 90 players. Riot had to pack three VALORANT games onto one core to make 128-tick affordable.
2. **Bandwidth is only expensive on the big clouds.** AWS charges $0.09/GB (first 10 TB), GCP $0.12/GiB (Premium tier), Azure about $0.087/GB. Hetzner dedicated servers have unlimited traffic on a 1 Gbit/s port, and Hetzner Cloud includes 20 TB per server in the EU and then charges €1/TB. Since June 2026 AWS GameLift bundles bandwidth for free on newer instances, at about a 27–30% higher instance price.
3. **For an OSRS-like game, bandwidth is tiny anyway.** Jagex says OSRS Mobile uses "500kb to 2MB of data per hour" (SNIPPET). That is about 4.4 kbit/s per player at most.
4. **Prices per vCPU-month** (on-demand, Nov 2026): AWS $26–37, GCP $33–35, Azure about $31 (SNIPPET). Hetzner Cloud dedicated-vCPU (CCX) €21.75. Hetzner dedicated root servers cost €3.3–5.0 per hardware thread.
5. **Hetzner raised prices twice in 2026** (1 April for everyone; 15 June for new orders and rescales). Secondary sources say CCX/CPX cloud plans rose by roughly 2.5×. Many cloud plans currently show "This product is currently unavailable." Servers Miles already rents keep their price.
6. **Cost per concurrent player per month** (my model, GUESS): OSRS-like about $0.31–0.53 on AWS, GCP or Azure, versus €0.03–0.05 on Hetzner dedicated. An action MMO is about $2.9–5.2 versus €0.38–0.56. A fast action game at around 220 kbit/s is about $7.4–10.5 (78–88% of it bandwidth) versus €0.17–0.25.
7. **How much server cost matters.** The only clean audited number is Roblox's. In 2025 its "infrastructure and trust & safety" line was $1.15B (SNIPPET), about 24% of revenue and 19% of total costs. That line also includes moderation and support staff. Everything else is thin or anecdotal. For an OSRS-like game, world hosting is probably under 2% of revenue (GUESS).
8. **The X% answer.** Make the binding resource X% cheaper per player, and the server bill falls by about (that resource's share of the bill) × X%. The company's total costs fall by that amount × the server share of total costs.
   - On Hetzner, CPU is about the whole bill, so a 20% CPU gain gives up to about a 20% bill cut. You only see it once you run several servers, or can step down a server tier.
   - A single box sees no saving until it changes tier.
9. **Best target: CPU per player per tick.** It is the binding constraint, and you pay for it on every provider. Bandwidth only matters on clouds that meter egress. Memory is rarely the limit, though 2026 RAM prices make it pricier.
10. **The money at stake for Miles is small.** At thousands of players on Hetzner the bill is tens to low hundreds of euros a month (GUESS). The bigger payoffs are more players per world, steadier ticks, and an open benchmark that large operators could use. At Google scale, AlphaEvolve's scheduler heuristic recovered "0.7%" of fleet compute. That is huge for Google and pennies for a single €79 box.

---

## 1. Prices that matter to a small developer (primary sources)

### 1.1 Hetzner (www.hetzner.com, docs.hetzner.com; fetched 2026-11-08)

Raw files: `/tmp/claude-0/-home-user-OogaPL/32372848-6606-541b-85af-54d683c4e916/scratchpad/dl/t2-hetzner/` (`dedicated-rootserver_matrix-ax.html(.txt)`, `cloud_*.html(.txt)`), `.../dl/t2-hetzner-docs/`, `.../dl/t2-hetzner-news/`.

**Dedicated root servers, AX line** (https://www.hetzner.com/dedicated-rootserver/matrix-ax/), VERIFIED:

| Model | CPU | RAM | From €/month (excl. VAT) | Setup | Quote |
|---|---|---|---|---|---|
| AX41 | Ryzen 5 3600 (6-core Zen 2) | 64 GB DDR4 | 59.00 | 0.00 | "AX41 1 Type starting from max/mo. € 59.00 per month" |
| AX42 | Ryzen 7 PRO 8700GE (8-core Zen 4) | 64 GB DDR5 ECC | 79.00 (max 99.00) | 49.00 | "AX42 2 Types starting from max/mo. € 79.00 per month, excl. VAT max/mo. € 99.00 per month" |
| AX102 | Ryzen 9 7950X3D (16-core) | 128 GB DDR5 ECC | 159.00 (max 259.00) | 129.00 | "AX102 4 Types starting from max/mo. € 159.00" |
| AX162 | EPYC 9454P (48-core Genoa) | 128 GB DDR5 ECC reg. | 319.00 (max 614.00) | 304.00 | "AX162 4 Types starting from max/mo. € 319.00" |

- Traffic, from the same page: "Traffic usage is unlimited and free of charge. Please note that our unlimited traffic policy does not apply to servers that have the 10G uplink addon." The connection is listed as "Guaranteed bandwidth 1 GBit/s". VERIFIED.
- Traffic docs (https://docs.hetzner.com/robot/general/traffic/): "All root servers have a dedicated 1 GBit uplink by default and with it unlimited traffic. Inclusive monthly traffic for servers with 10Gbit uplink is 20TB. There is no bandwidth limitation. We will charge € 1 ($1.20)/TB for overusage." Also "We calculate monthly traffic only using outgoing traffic." VERIFIED.
- DDoS: the AX page lists "DDoS protection" under Infrastructure (no extra price shown). VERIFIED.
- Locations: the AX matrix shows "Germany, FSN1" and "Finland, HEL1". The cloud page says "Our cloud servers are available to you in Germany, Finland, Singapore, and the USA." VERIFIED. So US players are served from Hetzner Cloud (Ashburn and Hillsboro), not from AX dedicated servers.

**Hetzner Cloud** (https://www.hetzner.com/cloud/cost-optimized/, /regular-performance/, /general-purpose/), VERIFIED:

| Plan | vCPU | RAM | €/month (EU) | Included traffic (EU / US / SG) | Status on 2026-11-08 |
|---|---|---|---|---|---|
| CX23 (Cost-Optimized, shared) | 2 | 4 GB | 5.99 | 20 TB / n/a / n/a | "not available" |
| CPX22 (Regular Performance, shared) | 2 | 4 GB | 19.99 | 20 TB / n/a / 1 TB | "currently unavailable" |
| CPX42 | 8 | 16 GB | 69.99 | 20 TB | "currently unavailable" |
| CCX13 (General Purpose, dedicated vCPU) | 2 | 8 GB | 43.49 (US 43.99, SG 54.49) | 20 TB / 1 TB / 1 TB | "currently unavailable" |
| CCX23 | 4 | 16 GB | 86.49 | 20 TB / 2 TB / 2 TB | "currently unavailable" |
| CCX33 | 8 | 32 GB | 138.99 | 30 TB / 3 TB | (see page) |

- Quotes: "CCX13 2 AMD 8 GB 80 GB NVMe from /month € 43.49 per month, excl. VAT"; "CPX22 2 AMD 4 GB 80 GB NVMe from /month € 19.99 per month, excl. VAT"; "CX23 not available 2 Intel®/AMD 4 GB 40 GB NVMe from /month € 5.99 per month, excl. VAT"; "This product is currently unavailable. Please check back later."
- Traffic quotes: "Cloud servers located in the EU include at least 20 TB of included traffic." "Cloud servers located in the U.S. include at least 1 TB of included traffic." "Cloud servers at AP locations include at least 0.5 TB of included traffic. For each additional TB at AP locations we charge € 7.40 ..."
- The page says "€ 1.00 per hour" for extra TB. It is marked "Automatically translated by AI", and the docs say €1/TB, so this is a translation slip.
- The General Purpose page itself suggests these plans for games: "Classic areas of use include ... public game servers ...". VERIFIED.
- Hetzner's own sales claim on its cloud page: "expensive egress traffic at hyperscalers such as AWS, Azure, and Google can quickly drive up total costs, with Hetzner you benefit from generous included traffic and very low prices for additional data transfer." VERIFIED, but it is a vendor claim.

**2026 price rises.** Primary statements from the pressroom, VERIFIED:

- 23 Feb 2026, https://www.hetzner.com/pressroom/statement-price-adjustment/: "There have been drastic price increases in various areas in the IT sector recently. That is why, unfortunately, we must also increase the prices of our products." Also: "The price changes take effect on 1 April 2026 and are for both new orders and existing products."
- 27 May 2026, edited 15 June 2026, https://www.hetzner.com/pressroom/standardization-and-price-adjustment-of-our-server-products/: "we are increasing our monthly prices for new orders. This is due to the ongoing challenges in the hardware procurement market." Also: "All dedicated servers and cloud plans at all locations are affected by this adjustment." And: "Currently rented servers are not affected by the price adjustment."
- How big the June change was (SNIPPET, secondary sources; Hetzner's docs table returned 404):
  - CCX13 went "€15.99 before and €42.99 after" (+169%), and CPX22 went "€7.99 to €19.49" (https://wz-it.com/en/blog/hetzner-price-increase-june-2026-cpx-ccx-alternatives/ and https://northflank.com/blog/hetzner-cloud-server-price-increases).
  - The live prices (€43.49, €19.99) are exactly €0.50 above these. The docs say "We charge an additional cost of € 0.50/month (excl. VAT) for our Primary IPs of type IPv4" (https://docs.hetzner.com/cloud/servers/overview/), so the website price probably includes an IPv4 address (GUESS).
  - April 2026 dedicated changes were reported as about +20% (for example "AX42 €47.30 → €57.30", SNIPPET, https://webhosting.today/2026/05/29/hetzner-has-now-raised-prices-three-times-in-2026-this-one-is-different/). The current AX42 is €79.00, which is VERIFIED.
  - The stated cause is DRAM and NAND prices driven by AI data-centre demand (SNIPPET, Tom's Hardware and heise via search). Reports say memory add-ons rose by about 575% (SNIPPET).

### 1.2 AWS: official AWS Price List API (pricing.us-east-1.amazonaws.com, fetched 2026-11-08)

Raw files: `.../dl/t2-aws/` (filtered CSV rows plus JSON). Parser scripts: `/tmp/claude-0/-home-user-OogaPL/32372848-6606-541b-85af-54d683c4e916/scratchpad/scripts/t2_ec2parse.py` and `t2_dtparse.py`.

The EC2 offer file (version 20261107002852, "Publication Date","2026-11-07T00:28:52Z") gives on-demand Linux, shared tenancy prices. VERIFIED:

| Instance | vCPU / RAM | us-east-1 $/h | eu-central-1 $/h | Quote |
|---|---|---|---|---|
| c7i.large (Sapphire Rapids) | 2 / 4 GiB | 0.08925 | 0.10185 | "$0.08925 per On Demand Linux c7i.large Instance Hour" |
| c7g.large (Graviton3) | 2 / 4 GiB | 0.0725 | 0.0825 | "$0.0725 per On Demand Linux c7g.large Instance Hour" |
| c8g.large (Graviton4) | 2 / 4 GiB | 0.07976 | 0.0907 | "$0.07976 per On Demand Linux c8g.large Instance Hour" |
| c7a.large (EPYC 9R14) | 2 / 4 GiB | 0.10264 | 0.11713 | |
| m7i.large | 2 / 8 GiB | 0.1008 | | "$0.1008 per On Demand Linux m7i.large Instance Hour" |
| c5.2xlarge / c5.9xlarge | 8/16, 36/72 GiB | 0.34 / 1.53 | | "$1.53 per On Demand Linux c5.9xlarge Instance Hour" |
| c7g.large, 1-yr standard RI, no upfront | | 0.0478 effective | | "Linux/UNIX (Amazon VPC), c7g.large reserved instance applied" |

**Internet egress** (AWSDataTransfer offer 20261103133147; us-east-1 and eu-central-1 identical). VERIFIED:

- "$0 for 100GB of data transfer out to the internet, aggregated globally, each month"
- "$0.090 per GB - first 10 TB / month data transfer out beyond the global free tier"
- "$0.085 per GB - next 40 TB / month data transfer out"
- "$0.070 per GB - next 100 TB / month data transfer out"
- "$0.050 per GB - greater than 150 TB / month data transfer out"

**GameLift** (AmazonGameLift offer 20261013153044, us-east-1). VERIFIED:

- "$ 0.094 per c7g.large GameLift Linux On Demand Instance Hour with free network bandwidth in US East (N. Virginia)"
- "$0.114 per Instance hour for GameLift On Demand Linux c7i.large with free network bandwidth"
- c8g.large is $0.103 and c7a.large $0.13.
- 576 of the 1,072 price lines carry the words "with free network bandwidth". Older c5.large ("$0.109 ... with 50 GB EBS") does not.
- AWS's announcement (SNIPPET, https://aws.amazon.com/about-aws/whats-new/2026/06/amazon-gamelift-servers-free-network-bandwidth/): bandwidth free from 15 June 2026 on generation 6 and newer instances, in all regions except China, for On-Demand and Spot.
- GameLift premium over plain EC2 (GUESS, from the VERIFIED prices):
  - c7g.large: +29.7% (0.094/0.0725 − 1)
  - c7i.large: +27.7%
  - c8g.large: +29.1%
  - c7a.large: +26.7%
- The extra $0.0215/h on c7g.large buys 0.239 GB/h of EC2 egress at $0.09/GB. That is about 0.53 Mbit/s per instance. GameLift wins whenever a 2-vCPU instance sends more than about 0.5–0.7 Mbit/s on average (GUESS; arithmetic in `t2_costmodel.py`).

**Other AWS lines.** VERIFIED:

- Shield Advanced: "$3,000 for First Subscription" per month, plus per-GB Shield data-transfer fees (AWSShield offer, published 2026-09-16).
- DynamoDB us-east-1: "$0.625 per million write request units (N. Virginia)"; "$0.00065 per hour for units of write capacity beyond the free tier"; storage "$0.25 per GB-Month".

**AWS's own cost model for game servers** (SNIPPET):

- https://aws.amazon.com/blogs/gametech/how-to-host-your-unreal-engine-game-for-under-1-per-player-with-amazon-gamelift/ estimates "a total monthly Amazon GameLift cost of $81,396.67, or $0.81 per player per month" for 100,000 peak CCU.
- Its assumptions are average CCU = 50% of peak, 16 sessions per instance, 8 players per session, a 10% idle buffer, and "DTO representing 20% of total costs".
- This dates from before the June 2026 free-bandwidth change.

### 1.3 Google Cloud: official pricing pages (cloud.google.com, fetched 2026-11-08)

Raw files: `.../dl/t2-gcp/`.

**Network** (https://cloud.google.com/vpc/network-pricing; the region selector lists "Iowa (us-central1)" first). VERIFIED:

- Premium Tier to North America: "1 gibibyte to 1,024 gibibyte $0.12 / 1 gibibyte ... 1,024 gibibyte to 10,240 gibibyte $0.11 ... 10,240 gibibyte and above $0.08".
- Premium Tier to Europe uses the same tiers, but $0.085 above 10 TiB.
- Standard Tier: "200 gibibyte to 10,240 gibibyte $0.085 / 1 gibibyte ... 10,240 gibibyte to 153,600 gibibyte $0.065 ... 153,600 gibibyte and above $0.045", with "First 200 GiB per month free".
- Cloud Armor Enterprise (DDoS) data processing: "The charge is $0.05 per GiB data transferred for the first 100 TiB and $0.04 per GiB for the next 400 TiB."

**VMs** (https://cloud.google.com/products/compute/pricing/general-purpose and /compute-optimized, us-central1). VERIFIED:

- "c4-standard-2 | 2 | 7 GiB | $0.096866 / 1 hour"
- "c4d-standard-2 | 2 | 7 GiB | $0.091680438 / 1 hour"
- "e2-standard-2 | 2 | 8 GiB | $0.06701142 / 1 hour"
- "n2d-standard-2 | 2 | 8 GiB | $0.084492 / 1 hour"
- "c2d-standard-2 | 2 | 8 GiB | $0.090798 / 1 hour"
- Third-party trackers give the same $0.0969 for c4-standard-2 in us-central1, which confirms the region.

### 1.4 Azure (azure.microsoft.com and prices.azure.com blocked)

- Zone 1 (North America and Europe) internet egress is about **$0.087/GB for the first 10 TB**, with 100 GB/month free (SNIPPET from third-party pages such as https://egresscost.com/azure, "verified 2026-06"). One source disagrees on the size of the tier.
- D2as v5 (2 vCPU, 8 GiB), East US, Linux pay-as-you-go: about **$0.086/h** (SNIPPET, https://calculator.holori.com/azure/vm/standard-d2as-v5/eastus). Another tracker shows $0.071, so treat this as approximate.

### 1.5 Summary: price per vCPU-month and per TB of egress

GUESS. These are VERIFIED or SNIPPET prices × 730 h, divided by vCPUs. Script: `/tmp/claude-0/-home-user-OogaPL/32372848-6606-541b-85af-54d683c4e916/scratchpad/scripts/t2_costmodel.py`.

| Option | per vCPU (or thread) per month | per TB egress |
|---|---|---|
| AWS c7i.large on-demand, us-east-1 / Frankfurt | $32.58 / $37.18 | $90 (first 10 TB), down to $50 (>150 TB) |
| AWS c7g.large on-demand / 1-yr RI | $26.46 / $17.45 | $90 → $50 |
| AWS GameLift c7g.large (bandwidth included) | $34.31 | $0 |
| GCP c4-standard-2 / c2d-standard-2 | $35.36 / $33.14 | $120 per TiB (Premium, first TiB) / $85 (Standard) |
| Azure D2as v5 (SNIPPET) | about $31.39 | about $87 (SNIPPET) |
| Hetzner Cloud CCX13 (dedicated vCPU) | €21.75 | 20 TB included (EU), then €1 |
| Hetzner Cloud CPX22 / CX23 (shared) | €10.00 / €3.00 (CX23 unavailable) | 20 TB included (EU), then €1 |
| Hetzner AX42 / AX102 / AX162 dedicated, per hardware thread | €4.94 / €4.97 / €3.32 | €0 (unlimited at 1 Gbit/s) |

Caveats:
- An x86 "vCPU" on AWS and GCP is one hyperthread. A Graviton vCPU is a whole core.
- The Hetzner dedicated per-thread price assumes every thread is used.
- Egress on the big clouds costs 50–120 times Hetzner's €1/TB overage.

**WRONG (with nuance):** the common belief that "Hetzner Cloud is a fraction of AWS's price". For dedicated-vCPU cloud plans after June 2026 this no longer holds:
- CCX13 at €21.75 per vCPU-month is close to AWS c7g on-demand ($26.46).
- It is above AWS c7g with a 1-year reservation ($17.45).

It does still hold for Hetzner dedicated root servers (about 5–7× cheaper per thread) and for traffic (about 90× cheaper per GB).

---

## 2. What does it cost per concurrent player?

This is a GUESS model built on the prices above.

Formula:
- cost per concurrent user (CCU) per month = (price per vCPU-hour × 730) / players per vCPU + (MB per player-hour / 1000 × 730 × price per GB)

Three workload archetypes. Each input is labelled:

- **A. OSRS-like tile MMO (0.6 s tick).**
  - 2 MB per player-hour (4.4 kbit/s). This is the top of Jagex's OSRS Mobile figure (SNIPPET, §3.2).
  - 100 players per vCPU. This is a pure GUESS: an OSRS world holds 2,000 players, and I do not know how many cores Jagex gives it.
- **B. WoW/New World-like action MMO.**
  - 8.8 players per vCPU, derived from New World's published per-world setup (§3.1).
  - 13.5 MB per player-hour (30 kbit/s). This is a GUESS between the WoW average of about 17 kbit/s and raid peaks of 50 kbit/s (§3.2).
- **C. Fast action or physics game (60 Hz, Sumo-bout-like or Fortnite-like).**
  - 20 players per vCPU. GUESS; VALORANT packs about 30 players per core at 128 Hz (§3.1).
  - 100 MB per player-hour (222 kbit/s). This comes from UNVERIFIED third-party Fortnite figures of 50–150 MB/h.

Cost per CCU-month. The bandwidth share of the bill is in brackets.

| Platform | A. OSRS-like | B. action MMO | C. fast action |
|---|---|---|---|
| AWS EC2 c7i on-demand (us-east-1) | $0.46 (bw 29%) | $4.59 (bw 19%) | $8.20 (bw 80%) |
| AWS EC2 c7g on-demand | $0.40 (33%) | $3.89 (23%) | $7.89 (83%) |
| AWS EC2 c7g 1-yr RI | $0.31 (43%) | $2.87 (31%) | $7.44 (88%) |
| AWS GameLift c7g (bandwidth free) | $0.34 (0%) | $3.90 (0%) | $1.72 (0%) |
| GCP c4-standard-2, Premium egress | $0.53 (33%) | $5.20 (23%) | $10.53 (83%) |
| GCP c2d-standard-2, Standard egress | $0.46 (27%) | $4.60 (18%) | $7.86 (79%) |
| Azure D2as v5 (SNIPPET prices) | $0.44 (29%) | $4.42 (19%) | $7.92 (80%) |
| Hetzner Cloud CCX13 | €0.22 (0%) | €2.47 (0%) | €1.09 (0%) |
| Hetzner Cloud CPX22 | €0.10 | €1.14 | €0.50 |
| Hetzner AX42 dedicated | €0.05 | €0.56 | €0.25 |
| Hetzner AX162 dedicated | €0.03 | €0.38 | €0.17 |

How to read this (GUESS):

- **Hetzner dedicated is about 7–9× cheaper than AWS on-demand per player for the CPU-heavy archetypes.** For the bandwidth-heavy archetype it is about 30–40× cheaper.
- **Egress volume per CCU-month**: 4.4 kbit/s → 1.45 GB; 30 kbit/s → 9.9 GB; 100 kbit/s → 32.9 GB; 222 kbit/s → 72.9 GB. At $0.09/GB that is $0.13, $0.89, $2.96 and $6.56.
- **A Hetzner 1 Gbit/s port is never the limit for an MMO.** It carries about 227,000 players at 4.4 kbit/s, 33,000 at 30 kbit/s, or 4,500 at 222 kbit/s.
- **Hetzner US Cloud includes only 1–8 TB per server**, but overage is €1/TB. 10 GB per CCU-month therefore costs €0.01 per CCU-month beyond the allowance. That is still negligible.
- **What the table leaves out:**
  - peak-versus-average capacity (you buy for the peak);
  - idle buffers;
  - databases, backups and logging;
  - DDoS products (AWS Shield Advanced is $3,000/month);
  - staff time.
- **The OSRS-like column is very sensitive to the players-per-vCPU guess.** At 1,000 players per vCPU, compute on AWS falls to about $0.03 per CCU-month, and bandwidth becomes about 80% of a still-tiny bill.

Cross-checks against published bills (GUESS arithmetic, SNIPPET inputs):

- **AWS GameLift blog example:** $81,396.67 / 100,000 peak CCU = $0.81 per peak-CCU-month. That is $0.0022 per player-hour at 50% average/peak.
- **New World, compute only, at today's list price:**
  - 4 × c5.2xlarge ($0.34) + 7 × c5.9xlarge ($1.53) = $12.07/h per world.
  - 2,500 players gives $0.0048 per player-hour.
  - Amazon's internal cost is surely lower.
- **Roblox:** $1,153.5M / 123.9B engaged hours = **$0.0093 per engaged hour**, about **$6.7 per average-CCU-month**.
  - Average CCU ≈ 127M DAU × 2.7 h / 24 h = 14.3M.
  - This line includes trust & safety staff, moderation, support, storage and AI/GPU.
- **Hypixel (UNVERIFIED):** "$100,000 per month" in 2015 at about 16,000 concurrent players. That would be about $6 per CCU-month.

---

## 3. Which resource is the bottleneck per player?

### 3.1 CPU: the usual hard limit on players per world or server

- **OSRS** (SNIPPET; Jagex blog "More Worlds, More Power: The Road To Greater Capacity", 7 Aug 2025, via the OSRS wiki copy https://oldschool.runescape.wiki/w/Update:More_Worlds,_More_Power:_The_Road_To_Greater_Capacity and the RuneNation mirror):
  - Each world simulates the whole game world, with "tens-of-thousands of NPCs, and up to 2,000 players".
  - Worlds run on "VMWare vSphere with Linux Virtual Machines".
  - Over-allocating vCPUs ("overcommitting") hurts world performance: "contention for CPU time can be a contributing factor, one we aim to minimise".
  - "Beyond scripts, we are always on the lookout for optimisations throughout the core engine. ... algorithms and improvements to CPU cache efficiency, that save several milliseconds off of each tick!"
  - "a game tick lasts 0.6 seconds, but for the game to feel responsive, we aim for the actual processing time to be much lower. We also aim for that processing time to be consistent each tick."
  - AWS was "confined to our core infrastructure, covering services like accounts and player saves". Cloud-hosted worlds were starting with US East.
  - The OSRS wiki says "each members world allows 2000 players to be connected at once" (SNIPPET, https://oldschool.runescape.wiki/w/Server).
- **EVE Online** (SNIPPET): time dilation is CCP's response when a node's CPU is overloaded. CCP's 2011 post says the goal is to keep the queue of pending tasks near zero (https://eveonline.com/article/introducing-time-dilation-tidi). A CCP director called it a "band-aid" (TechRadar). Whether a node is strictly single-threaded is disputed between sources.
- **Albion Online** (SNIPPET; 2016 Quo Vadis talk by David Salz, https://es.slideshare.net/davidsalz54/albion-online-software-architecture-of-an-mmo-talk-at-quo-vadis-2016-berlin):
  - The world is split into clusters of about 1 km².
  - Each cluster holds "up to 300 players", about 500 mobs and more than 10,000 interactive objects.
  - Each cluster's game logic runs on a single thread, to avoid locking.
- **Minecraft-like games** (VERIFIED):
  - Servo (arXiv 2305.00032, https://arxiv.org/abs/2305.00032): "Servo scales to 150 players per instance, whereas Minecraft and Opencraft support up to 90 and 10 players". Commercial services partition players into small instances, "recommending as few as 10 (+1) maximum concurrent players per instance".
  - Yardstick (ICPE 2019; abstract via Semantic Scholar, DOI 10.1145/3297663.3310307): "Minecraft-like services are poorly parallelized".
  - Meterstick (arXiv 2112.06963): "peak-latency degrades sharply to 20.7 times the arithmetic mean". Cloud deployment is a significant source of performance variability.
- **VALORANT** (SNIPPET; Riot tech blog "VALORANT's 128-Tick Servers", https://technology.riotgames.com/news/valorants-128-tick-servers):
  - Using one full core per game would make it "prohibitively expensive to host game servers at scale".
  - Servers were tuned to fit 3 games into about 7.8 ms, so each frame must average under 2.6 ms.
  - Frame time went from about 50 ms at the start to 2.34 ms.
  - GUESS: 3 games × 10 players ≈ 30 players per core at 128 Hz.
- **New World** (SNIPPET; AWS blog https://aws.amazon.com/blogs/gametech/the-unique-architecture-behind-amazon-games-seamless-mmo-new-world):
  - "Each of the four REPs is a separate Amazon EC2 C5 Instance, specifically an Amazon EC2 C5.2xlarge Instance".
  - "seven computational processing units, each of which is an Amazon EC2 C5.9xlarge Instance".
  - "Collectively, the Amazon EC2 instances for a single world in Aeternum can simulate more than 7,000 artificial intelligence entities and hundreds of thousands of objects for 2,500 players."
  - It simulates about 30 times per second, against about 5 for "a traditional MMO".
  - DynamoDB "handles about 800,000 writes every 30 seconds". It is not clear whether this is per world or for the whole game.
  - GUESS: 4×8 + 7×36 = 284 vCPUs per world gives 2,500 / 284 = **8.8 players per vCPU**. The vCPU and RAM numbers are VERIFIED from the AWS price list.
  - GUESS on persistence cost: if the 800k writes/30 s figure were per world, on-demand DynamoDB would cost 26,667 writes/s × 3,600 × $0.625/M = **$60/h**, which is more than the $12/h of compute. With provisioned capacity it would be $17/h. The scope is unknown, but persistence writes can be a large hidden cost.
- **World of Warcraft, 2009** (SNIPPET; Blizzard GDC Austin keynote as reported by TechCrunch, https://techcrunch.com/2009/09/18/blizzard-reveals-some-technical-data-about-world-of-warcraft/, and Data Center Knowledge):
  - "13,250 server blades, 75,000 CPU cores, and 112.5 terabytes of blade RAM".
  - 20,000 systems, 1.3 PB of storage, 10 data centres.
  - "the Blizzard network is managed by a staff of 68 people".
  - Subscribers were "more than 11.5 million" (Dec 2008, SNIPPET, https://investor.activision.com/node/17971).
  - GUESS: if peak CCU was 5–15% of subscribers, that is **about 8–23 CCU per core** and **68–205 MiB RAM per CCU**.

### 3.2 Bandwidth per player: measured numbers

- **WoW** (VERIFIED_FIG; Saldana et al., arXiv 2007.15373, Table II, https://arxiv.org/abs/2007.15373):
  - Server-to-client: E[P] = 314 bytes average TCP payload at 6.05 packets/s.
  - Client-to-server: 8.74 bytes at 9.51 pps.
  - The paper defines "E[P]: The expected value of the TCP payload". The values come from the Svoboda et al. model (ICC 2007, DOI 10.1109/ICC.2007.270).
  - GUESS: (314 + 40 header) × 6.05 = 2,142 B/s, or about **17 kbit/s** down per player. Up is (8.74 + 40) × 9.51 ≈ 3.7 kbit/s.
  - The same table gives ShenZhou Online s2c 114 B at 8 pps (about 9.9 kbit/s, GUESS) and Runes of Magic 99 B at 5.17 pps (about 5.7 kbit/s, GUESS).
- **WoW peaks** (VERIFIED quote of a cited study; arXiv 2007.16070, https://arxiv.org/abs/2007.16070): "traffic demands may rise up to 50 kbps in downlink and 5 kbps in uplink [21], but that the average demands are typically lower". [21] is Suznjevic, Dobrijevic and Matijasevic, Multimedia Tools & Applications 45 (2009).
- **Lineage II** (VERIFIED, abstract via OpenAlex; Kim et al., NetGames 2005, DOI 10.1145/1103599.1103619):
  - "the average payload size of server packets is about 318 bytes".
  - "the bandwidth usage of the server traffic is about ten times larger than that of the client traffic".
  - "there is a linear relationship between the number of users and the bandwidth usage".
- **OSRS** (SNIPPET; Jagex "OSRS Mobile: Data and Battery Blog", via https://oldschool.runescape.wiki/w/Update:OSRS_Mobile:_Data_and_Battery_Blog): "Old School Mobile currently uses, on average, 500kb to 2MB of data per hour of full gameplay."
  - GUESS: 2 MB/h = 556 B/s ≈ 4.4 kbit/s.
  - This was an early mobile test, so the current figure may differ.
- **Carlini's PhD thesis on MMOs** (VERIFIED; arXiv 1510.08940, 2012, https://arxiv.org/abs/1510.08940):
  - The server-to-client update rate "represents the major source of cost of the infrastructure".
  - "a medium-sized server can manage around 1000 clients before slowing down".
  - "With 1000 nodes, and Ts = 0.25 the bandwidth requirement is 3MB/s. Using the prices of a current commercial on-demand platform, the deployment would cost 30$ per day only considering bandwidth" (footnote: "0.12$ per GB, Amazon EC2 prices, July 2012").
  - GUESS: 3 MB/s / 1000 = 24 kbit/s per client at 4 updates/s.
- **Physics-heavy state sync** (VERIFIED; Glenn Fiedler, "Snapshot Compression", 2015, source at https://raw.githubusercontent.com/gafferongames/gafferongames/master/content/post/snapshot_compression.md):
  - "Our target bandwidth is __256 kilobits per-second__."
  - The naive 60 Hz snapshot of 901 cubes was "**17.37 megabits per-second!**".
  - "I wrote a short ruby script to find the best encoding with a greedy search."
  - Fabian Giesen's arithmetic coder "beat my best bit-packed result by 25%".
- **Fortnite** (UNVERIFIED; third-party pages such as https://www.firsty.app/help/general/how-much-data-does-fortnite-use): about 50–150 MB per hour of gameplay, i.e. 111–333 kbit/s. No official Epic figure was found.

### 3.3 Memory per player

- **New World** (GUESS): 4×16 + 7×72 = 568 GiB per world / 2,500 players ≈ **233 MiB allocated per player**. That is what was provisioned, not what was used.
- **WoW 2009** (GUESS): **68–205 MiB per peak CCU** (see §3.1).
- **RAM is cheap next to CPU on AWS** (GUESS from VERIFIED prices):
  - m7i.large − c7i.large = $0.01155/h for 4 GiB more, so about $0.0029 per GiB-hour, about **$2.11 per GiB-month**.
  - One vCPU-hour costs about the same as 15 GiB-hours.
  - For New World-like loads, RAM would be about 11% of the compute cost.
- **Hetzner dedicated** bundles 64–128 GB. For a tile MMO the RAM is very unlikely to be the limit (GUESS).
- **The 2026 DRAM shortage raised the price of RAM-heavy configurations** (Hetzner cites "hardware procurement" costs, VERIFIED; memory add-ons up about 575%, SNIPPET). This matters only if a design keeps very large world state in RAM.

### 3.4 Verdict on the bottleneck

- **CPU per tick** limits how many players a world or server can hold in every MMO example found: OSRS, EVE, Albion, Minecraft, VALORANT, New World.
- **Bandwidth** is the main *cost* only when egress is metered at $0.09–0.12/GB and the game is action-heavy (about 100+ kbit/s per player).
- **Memory** is neither the limit nor the main cost in any example.
- All three points are GUESS-level synthesis of the sources above.

---

## 4. How big are real bills, and what are they made of?

- **Roblox** (SNIPPET; FY2025 10-K, https://www.sec.gov/Archives/edgar/data/1315098/000131509826000024/rblx-20251231.htm; sec.gov is blocked, so these are search excerpts):
  - Infrastructure and trust & safety was **$1,153,454 thousand in 2025** against $915,418 thousand in 2024 (+26%). "The increase was primarily due to an increase of $186.3 million related to data center and technical infrastructure expenses", plus $47.5M in personnel.
  - What the line covers: "expenses related to the operation of our data centers and technical infrastructure". That includes "third-party service provider costs, such as cloud computing or other hosting and data storage", co-located and edge data-centre facilities, network and bandwidth, and depreciation ($184.6M in 2025). Since FY2023 it also includes "personnel costs and moderation and customer support related costs".
  - "users spent 123.9 billion hours engaged on our Platform, or an average of 2.7 hours per DAU each day"; 127M average DAU.
  - "The virtual environments and assets used by Roblox Clients are simulated on servers running in 25 regional data centers"; "As of December 31, 2025, the Roblox Cloud uses over 150,000 servers."
  - Revenue was about $4.9B (+36%) and bookings about $6.8B (SNIPPET, news coverage). Total costs and expenses were about $6,123M (SNIPPET, secondary).
  - GUESS: I&T&S ≈ **23.6% of revenue** and **18.8% of total costs**; ≈ $0.0093 per engaged hour; ≈ $7,700 per server per year, including staff.
  - Roblox on public cloud, 2023 (SNIPPET; O'Reilly talk slides as covered by DCD and SDxCentral, https://datacenterdynamics.com/en/news/roblox-spends-nearly-400m-on-ashburn-data-center-rolls-out-edge-facilities): "overall the cost of using [public] cloud is too much for what we need, networking would be a huge cost for us due to game server traffic. For some of our compute use cases cloud costs up to 10x more."
- **Star Citizen / Cloud Imperium** (SNIPPET; https://cloudimperiumgames.com/blog/corporate/cloud-imperium-financials-for-2019 and -2021):
  - Hosting is not reported on its own line. It sits inside "Publishing Operations, Community, Events and Marketing", which "includes the costs of running our platform, publishing, data hosting, and server costs" (2021 wording).
  - 2019: "this cost line moves directly with income and unsurprisingly showed a large growth in 2019, up 26% to $11.4M". The 2020 post says "costs increased by 10% over 2019 to $74.8M".
  - GUESS: that implies about $68M total in 2019, so the whole category was about 17%, and hosting itself is some unknown part of that 17%.
  - Headcount is over 1,100 (SNIPPET). Salaries dominate.
- **Jagex / RuneScape** (UNVERIFIED; secondary page citing Companies House): 2024 revenue £151M and adjusted EBITDA £78M. Hosting is not broken out.
  - GUESS: OSRS peak of about 240,000 CCU (SNIPPET, GamesRadar) / 2,000 per world = 120 worlds.
  - At €100–2,000 per world-month, that is €0.14–2.9M/yr, or about **0.1–1.7% of revenue** (assuming £1 ≈ €1.16).
- **EVE Online** (SNIPPET; CCP dev blog "A History of EVE Database Server Hardware", 2022): CCP chose "a high-end AMD server with 2 TB of memory and a single CPU rocking 128 cores", but "the team had one problem: the cost". No figures. The database is a notable single expensive box.
- **Palworld** (SNIPPET; PC Gamer, TechSpot, GGRecon): server costs were projected at over ¥70M for February 2024, about **$478k/month**, at launch peak (19M players). The CEO had ordered 100% uptime "at any cost". This is a co-op survival game, not an MMO, but it shows launch-peak bills.
- **Worlds Adrift on SpatialOS** (SNIPPET):
  - Bossa closed it in July 2019 as "no longer commercially viable". "Worlds Adrift just hasn't reached the level of popularity it needs to continue." Bossa said SpatialOS was not to blame (https://www.pcgamesinsider.biz/news/69671/bossa-doesnt-blame-improbable-for-worlds-adrifts-cancellation/).
  - A Steam post says "these costs are not compatible with the amount of players currently playing" (SNIPPET; probably a forum post, not the official statement).
  - A Steam user recalled "100k a month for a game that was doing 200 concurrents" (UNVERIFIED).
  - Improbable, the SpatialOS vendor, in the year to May 2019: "Cost of sales also rose by 62 per cent for the year, resulting in gross loss rising by 59 per cent, hitting £16m" (SNIPPET; pcgamesinsider). The platform cost more to run than it earned.
  - A 2018 Reddit thread cites SpatialOS worker pricing of "$2.44 per hour" to "$9.96 per hour" (SNIPPET).
- **Indie MMOs** (SNIPPET):
  - Project Gorgon: shards "may cost as much as $2000/month per shard"; the developer hoped "to get my monthly costs down to $500/month or less" (wiki copy of Elder Game's "How does Project Gorgon make money?", https://wiki.projectgorgon.com/wiki/Elder_Game:_How_does_Project_Gorgon_make_money%3F).
  - Wild Terra 2: the studio asked players for "$1800 per month for server costs" (MassivelyOP headline).
- **Ops staff:** I found no clean primary split.
  - WoW's network was "managed by a staff of 68 people" in 2009 (SNIPPET).
  - Roblox's 2025 I&T&S growth was mostly infrastructure (+$186.3M) rather than personnel (+$47.5M) (SNIPPET).
  - On unmanaged Hetzner servers, Miles's own ops time is likely worth more than the server bill (GUESS).

## 5. Does server cost matter to MMO economics?

The evidence is **thin**. Only Roblox reports an audited line for it, and that line mixes in safety staff.

- **Platform scale (Roblox):** about 19% of total costs and about 24% of revenue (GUESS from SNIPPET inputs). Here 1% efficiency on infrastructure is worth about $11.5M a year (GUESS: 1% × $1.153B).
- **Big subscription or premium MMOs (OSRS-like):** probably 0.1–2% of revenue for game-world hosting (GUESS). Salaries, marketing and platform fees are the real costs.
- **Crowdfunded or AAA development (Star Citizen):** the category that contains hosting was about 17% of costs in 2019, together with marketing and events (GUESS from SNIPPET).
- **Failed MMOs (Worlds Adrift):** the stated cause was too few players, not the server bill. With 200 CCU any per-player cost is high. Better algorithms cannot fix a lack of players.
- **Indies:** hundreds to a few thousand dollars a month (SNIPPET examples). That matters to a one-person studio, but it is often less than one salary.

---

## 6. If an algorithm makes one resource X% more efficient, how much does the bill drop?

GUESS throughout.

**Formula.** Suppose resource r is the binding constraint, and the new algorithm needs X% less of r per player. Then:
- Server bill drop ≈ s_r × X%, where s_r is r's share of the server bill.
- If instead "X% more efficient" means "X% more throughput", the drop is s_r × X / (100 + X). For example, +25% throughput gives 20% fewer servers.
- Company-wide saving ≈ (server share of total costs) × (server bill drop).
- A resource that is not binding saves nothing on bundled servers. Cutting bandwidth on a CPU-bound Hetzner box saves €0.

**Worked examples:**

1. **Miles today, one Hetzner box.**
   - The bill is fixed at €59–159/month for AX41–AX102, or his existing contract.
   - A 30% CPU cut saves €0 unless it lets him step down a tier. For example, AX42 €79 → AX41 €59 is −25%, if the older Zen 2 chip is fast enough (GUESS).
   - The real gain is about 1.4× more players on the same box (1/0.7).
2. **OSRS-like at 10,000 CCU on Hetzner AX42s.**
   - Say 2,000–3,000 CCU per box (GUESS), so 4–5 boxes ≈ €316–395/month.
   - A 30% CPU cut removes about 1 box, saving about €79/month. Rounding makes the saving lumpy.
   - Bandwidth savings are worth €0.
3. **Action MMO on AWS EC2 on-demand** (CPU about 77–83%, bandwidth about 17–23% of the bill; §2 table):
   - 20% less CPU per player → bill −15 to −17%.
   - 20% less bandwidth → bill −3.5 to −4.6%.
4. **Fast action game on AWS, GCP or Azure VMs** (bandwidth 78–88% of the bill):
   - 20% fewer bytes per player → bill −16 to −18%.
   - 20% less CPU → −2.4 to −4.4%.
   - Moving to GameLift (free bandwidth) or Hetzner beats any compression algorithm: about −79% for the same workload (AWS c7i $8.20 → GameLift $1.72 per CCU-month).
5. **Roblox scale.**
   - If half of I&T&S were compute (pure GUESS), 20% less compute per player ≈ 0.5 × 0.2 × $1.153B ≈ **$115M a year**, about 1.9% of total costs.
   - For comparison, AlphaEvolve's production scheduling heuristic at Google "continuously recovers on average 0.7% of Google's fleet-wide compute resources, which would otherwise be stranded" (VERIFIED, arXiv 2506.13131, https://arxiv.org/abs/2506.13131). Percent-level AI wins pay off at that scale.

**Best target: CPU time per player per tick** (simulation, NPC AI, pathfinding, collision, interest management):
- It is what caps players per world in every example.
- You buy it on every provider.
- It also improves the game itself: bigger worlds, steadier ticks, no time dilation.
- Jagex already reports hand-made algorithm and cache wins of "several milliseconds" per tick.

**Second target: bytes per player**, only for action games on clouds that meter egress, or for mobile players' data plans. Changing provider is a cheaper fix for egress cost than any algorithm.

**Memory and database writes:** only if profiling shows them binding. New World's write volume is a hint that persistence can be expensive.

---

## 7. Evidence against, and caveats

- **Server bills are small for most games next to salaries.** An algorithm that cuts them by 20% changes indie economics by a few euros to a few hundred euros a month (GUESS).
- **Savings are lumpy.** With one or two servers you save nothing until you can drop a tier or avoid buying the next server.
- **Bandwidth cost can be removed by a business choice** (Hetzner dedicated, GameLift gen6+, Roblox-style own data centres), not by an algorithm.
- **Hetzner's 2026 increases and "currently unavailable" plans** show that hardware supply and price, not efficiency, are moving the 2026 bill the most. Existing contracts keep their prices.
- **My cost model rests on guesses** for players per vCPU and bytes per player. Miles should measure his own game. Bytes on the wire and CPU per tick are both easy to measure.
- **Many key sources were blocked** (Roblox 10-K, AWS blogs, Riot, Jagex, CCP, CIG). Those claims are SNIPPET-level. The search tool's summaries sometimes paraphrase, so the quoted snippets may not be word-for-word.
- **Prices change.** AWS, GCP and Hetzner prices were read on 2026-11-07/08.

## 8. Candidate problems for AI search, seen through the cost lens

1. **Players per core for an OSRS-style tick.**
   - Metric: the most scripted bots one pinned core sustains with p99 tick time ≤ 10% of 600 ms, with identical game outcomes against a reference.
   - This maps directly to € per player on Hetzner.
2. **State-update encoder.**
   - Metric: mean bytes per second per client on replayed traces, at bounded quantisation error and lossless discrete state.
   - Headroom is proven (17.37 Mbit/s → 256 kbit/s; an arithmetic coder beat hand bit-packing by 25%), but it is worth about €0 on Hetzner.
3. **Interest management and update-rate policy.**
   - Metric: bytes and CPU per player at bounded staleness or position error.
   - Carlini shows bandwidth falls roughly in step with longer update intervals (3 MB/s at 0.25 s against about a third of that at 1 s for 1,000 clients).
4. **Packing zones and sessions onto servers.**
   - Metric: servers or core-hours needed for a demand trace with no tick-budget violations.
   - FunSearch already published online bin-packing heuristics with an evaluation suite (VERIFIED, https://github.com/google-deepmind/funsearch: "`bin_packing` contains heuristics discovered by FunSearch for online 1D bin packing problems, and an evaluation suite"). AlphaEvolve did the same for Borg (0.7%).
   - The gains are probably small and matter only at scale.

## 9. Open questions

- How many cores does one OSRS world actually use? That number decides the players-per-vCPU input.
- What are Miles's own game's bytes per player per second and CPU per player-tick? Measure them.
- How does Roblox's I&T&S split between hardware/network and people?
- Is New World's 800k DynamoDB writes per 30 s per world or global?
- Will Hetzner's "currently unavailable" cloud plans and its 2026 prices persist? Would OVH or other unmetered hosts be cheaper now? (Not researched; ovhcloud.com was blocked.)
- What do ops staff and DDoS mitigation (beyond Hetzner's included protection) really cost for UDP game traffic?

## 10. Hosts that were blocked

These failed to resolve or returned 403 from the proxy:
- technology.riotgames.com, aws.amazon.com, docs.aws.amazon.com, repost.aws, b0.p.awsstatic.com
- www.sec.gov, efts.sec.gov, data.sec.gov
- azure.microsoft.com, prices.azure.com
- ir.roblox.com, corp.roblox.com, blog.roblox.com, www.roblox.com
- www.gamedeveloper.com, gafferongames.com (worked around via raw.githubusercontent.com), mas-bandwidth.com, www.gdcvault.com
- ccpgames.com, www.eveonline.com, forums.eveonline.com
- secure.runescape.com, oldschool.runescape.wiki, runenation.org
- robertsspaceindustries.com, www.cloudimperiumgames.com, improbable.io
- www.companieshouse.gov.uk, find-and-update.company-information.service.gov.uk, web.archive.org
- developer.valvesoftware.com, www.epicgames.com, news.blizzard.com, www.albiononline.com
- steamcommunity.com, www.reddit.com, old.reddit.com, news.ycombinator.com, hn.algolia.com
- wiki.projectgorgon.com, www.eldergame.com
- www.pcgamer.com, massivelyop.com, www.datacenterknowledge.com, www.datacenterdynamics.com, www.sdxcentral.com, www.techcrunch.com, www.tomshardware.com, wz-it.com
- lup.lub.lu.se, portal.research.lu.se, core.ac.uk, research.spec.org
- export.arxiv.org
- website-price-api.hetzner.com, cloudpricingcalculator.appspot.com, www.gstatic.com, www.ovhcloud.com

These were reachable but throttled: api.openalex.org (daily budget used up) and api.semanticscholar.org (HTTP 429 at times).

## 11. Files

- Cost model script: `/tmp/claude-0/-home-user-OogaPL/32372848-6606-541b-85af-54d683c4e916/scratchpad/scripts/t2_costmodel.py`. Run it with `python3 -I`; it prints every derived number used above.
- Parsers: `.../scratchpad/scripts/t2_ec2parse.py`, `.../scratchpad/scripts/t2_dtparse.py`.
- Downloads, one folder each: `.../scratchpad/dl/t2-hetzner/`, `t2-hetzner-docs/`, `t2-hetzner-news/`, `t2-aws/`, `t2-gcp/`, `t2-papers/` (arXiv PDFs and text: 2007.15373, 2007.16070, 1510.08940, 2305.00032, 2112.06963), `t2-gaffer/`, `t2-funsearch/`.
