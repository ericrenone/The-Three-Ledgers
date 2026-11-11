# The Three Ledgers

### An architecture strategy for AI, 2027–2032: why FLOPs stopped being the product, what replaced them, and twelve dated predictions

> **What this is.** A top-down strategy document on AI compute architecture, written from the evidence assembled in this folder — the dissection of e-means (`README.md`), the rotation-native analysis (`README_ROTATION.md`), the three hardware frameworks it was tested against, and the market state as of November 2026. It is written for people who allocate capital, pick silicon, or design the algorithms that will run on it. It makes claims that can be wrong, dates them, and says what would falsify each.
>
> **What it is not.** It is not a market-sizing exercise. The numbers quoted are the public ones, cited, and used only to locate the argument in time. The argument itself is structural and would survive a 30% error in any of them.

---

## 0. The thesis in one page

For forty years the semiconductor industry sold one thing: multiplications per second per dollar. That product is now in surplus. The 2026 accelerator market — roughly $730 billion of hyperscaler capital expenditure, two-thirds of it serving inference rather than training, custom ASIC shipments growing at 44.6% against 16.1% for GPUs — is not short of FLOPs. It is short of three other things, and strategy for the next five years is the strategy of those three scarcities.

**Ledger 1 — bytes.** Every HBM wafer that will exist through 2027 is already sold. New capacity arrives in 2028–2029. Each HBM wafer displaces three DDR5 wafers, so the shortage propagates into ordinary memory. The binding constraint on inference is not compute; it is how many bytes a token must move. Any architecture that moves fewer bytes per token is worth more than any architecture that multiplies faster. This is why decode has already left the GPU for SRAM-first and in-memory-compute ASICs, why the heterogeneous prefill/decode split is in production, and why "eliminate the traffic" (compute transcendentals on-chip instead of fetching them, compress KV state, quantize shape-aware) is now a higher-leverage lever than "widen the bus".

**Ledger 2 — joules.** Power delivery, not transistor count, is what stops a data center. The sub-7nm interconnect bottleneck is thermal. Fixed-point shift-and-add datapaths that do in picojoules what DRAM does in hundreds of picojoules are not a nostalgia item; they are the only arithmetic that scales with the power envelope. Joules and bytes are the same ledger seen from two sides: moving a byte costs more energy than computing on it.

**Ledger 3 — bits of trust.** This is the one the industry has not priced yet. The same prompt at temperature zero returns ~80 distinct completions out of 1,000 runs on a production server, because kernels are not batch-invariant; a deterministic mode costs 1.6–2.1× today. Reinforcement-learning post-training breaks when the inference path and the training path compute different numbers. Regulated decisions (credit, clearing, medical triage) cannot run on a function whose output depends on what else was in the batch. Trust — *the same bits, every time, attestable* — is a product, it has a cost today, and that cost is falling. The evidence in this folder shows the point most people miss: **arithmetic determinism is not algorithmic determinism.** A 2026 state-of-the-art clustering algorithm run on bit-exact arithmetic still returns six different answers for six batch orders. Trust requires deterministic arithmetic *and* either order-invariant algorithms or a logged transcript. That is a design discipline, and it is one the hardware frameworks tested here (fixed-point, residual monitors, direction records) already possess.

**The structural claim.** The accelerator market is not splitting into two (training GPU vs. inference ASIC). It is splitting into **three**: a *multiply engine* for prefill and training where FLOPs still matter; a *bandwidth engine* for decode where bytes are the constraint; and a *state engine* — fixed-point, exact, order-invariant, self-attesting — for everything that must be reproducible: codebooks and quantizers, risk and compliance, RL rollouts, agent memory, audit. The third engine does not exist as a product category today. By 2030 it will.

**The algorithmic claim.** Every learning system is a choice of two things: *which sufficient statistics it carries* and *how it forgets*. Algorithms that carry less and forget by schedule (the e-means pattern: counts and sums, cosine-annealed) win on the multiply engine and lose on shape, drift, exactness, and small data. Algorithms that carry the second moment and forget by rate (the rotation-native pattern: a triangular factor updated by Givens rotations) win on the state engine and are *built from the primitives it has* — vectoring, rotation, division — with no inverse ever formed. The preconditioner is the product. Who owns the metric owns the margin.

**Twelve predictions** follow in §8, each dated, each with a falsifier. The three with the highest conviction: attested (bitwise-reproducible, transcript-carrying) inference becomes a priced SKU at every major cloud by end-2028; memory-bytes-per-token replaces peak FLOPs as the headline metric in hyperscaler silicon announcements by 2028; and hyperscaler capex growth falls below 20% year-on-year in 2028 not because demand falls but because bytes-per-token falls faster than tokens rise.

---

## 1. Where the money is, November 2026

Locate the argument before making it.

**Capital.** The four largest hyperscalers guide to roughly $730 billion of 2026 capital expenditure — Amazon ~$220B, Alphabet $195–205B, Microsoft ~$170B, Meta $135–145B — up about 78% from ~$410B in 2025, with every guidance revision in 2026 moving up and Microsoft's CFO citing $25B of component price inflation. Gartner puts total 2026 AI spending at $2.67 trillion, $1.48 trillion of it infrastructure. One investment bank projects $7.6 trillion of hyperscaler AI capex across 2026–2031. The 2026 number is roughly 1.5% of world GDP spent by four companies on one category of equipment.

**Memory.** All three merchant DRAM suppliers describe 2026 HBM output as committed; Samsung's HBM4 was reported sold out for the year on its Q1 call and SK hynix's full 2026 DRAM and NAND offering was spoken for in October 2025. New HBM capacity — SK hynix Indiana, the Cheongju lines, Micron Idaho — is not expected at volume before 2028–2029. The conversion ratio is ~3 DDR5 wafers per HBM wafer, so commodity DRAM is being starved: lead times above 40 weeks on large orders, DDR5 contract prices up multiples, Samsung fulfilling ~70% of incoming DRAM orders, and the two Korean suppliers signalling to investors that they will *not* chase volume. Memory is the first ledger because it is the one with a physical wall and an 18–24-month build time.

**Workload mix.** Inference is roughly two-thirds of all AI compute in 2026, up from one-third in 2023 and half in 2025; Gartner has inference at 55% of AI-optimized IaaS spend in 2026 and 59% in 2027; over a model's life, inference is 80–90% of its compute cost. Training is a plateau, not a decline — it grows at ~20% a year — but every incremental dollar is tilting to serving, and the thing being served is increasingly agentic: multi-step, tool-using, long-context, and therefore decode-heavy and memory-bound.

**Silicon.** Custom ASIC shipments are projected to grow 44.6% in 2026 against 16.1% for merchant GPUs; ASIC-based AI servers reach ~28% of the market; one research house expects custom ASIC *unit* volume to exceed merchant GPU volume by 2028. Google's TPU 8t/8i pair (April 2026), Microsoft's Maia 200 (TSMC 3nm, >140B transistors, 216 GB HBM3e, >10 PFLOPS FP4), Amazon's Trainium 3 (UltraClusters to one million chips), Meta's MTIA 300 (March 2026), and OpenAI's Broadcom-built inference part (10 GW planned by 2029) are all in production or sampling. Broadcom reported $8.4B of AI semiconductor revenue in its February-2026 quarter, up 106%; Marvell guides to $11B of AI ASIC revenue for 2026; the two control ~95% of the co-design market. NVIDIA still holds ~70% of AI chip revenue (over 90% of merchant accelerators) and is expected to keep training while ceding inference share.

**Heterogeneity is already in production.** d-Matrix's Corsair (digital in-memory compute, SRAM-first, no liquid cooling) entered full volume production in June 2026; Parasail deployed it alongside Hopper and Blackwell in July 2026 with GPUs on prefill and Corsair on decode — the first commercial-scale disaggregated inference of its kind. NVIDIA's own answer is the Groq licensing deal (closed December 2025) that puts an LPU decode rack beside its GPUs. The industry has already conceded the point that one engine does not serve both phases.

**Trust.** Thinking Machines' late-2025 result — 1,000 identical temperature-zero queries, ~80 distinct completions, root cause the absence of batch invariance in reductions — reframed determinism from an academic nicety into an engineering gap. Their batch-invariant vLLM path cost 2.1× unoptimized, 1.6× optimized. Follow-on work extends determinism across tensor-parallel sizes and achieves bitwise identity between the vLLM rollout and FSDP training paths (Zhang et al., Nov 2025), and reduces the verification overhead further (MarginGate, May 2026). Anthropic's mid-2026 API change rejecting non-default temperature and top-p values is a signal that providers are moving to control the sampling surface. Determinism is a product under construction.

Those are the facts. Now the argument.

---

## 2. Three ledgers, one accounting identity

Strategy follows scarcity. The three scarcities are not independent; they are three views of one accounting identity, and seeing that is what makes the strategy coherent rather than a list.

**Bytes and joules are the same ledger.** Moving a 32-bit word across a chip costs more energy than a 32-bit multiply; moving it off-chip to DRAM costs 100× more. HBM exists because bandwidth is the constraint, and HBM is sold out because every token moves its weights and its KV cache through it. So "reduce bytes per token" and "reduce joules per token" are one objective with two names. Everything that reduces traffic — on-chip transcendentals instead of LUT fetches, KV compression, speculative decode, weight quantization, SRAM-resident decode — is simultaneously a memory play and a power play. The hardware frameworks in this folder (Topological Rotation §I.2, HELICON §11) make the same observation from the fabric side: a CORDIC transcendental costs ~1–2 pJ and no memory fetch; a LUT miss cascades into cache-line misses. Whether the specific CORDIC claims hold at every node is secondary; the *direction* is correct and it is where the industry is already moving.

**Trust is the ledger that converts the other two into revenue.** A token that cannot be reproduced cannot be audited, cannot be used as an RL rollout without train/inference mismatch, cannot be the basis of a regulated decision, and cannot be sold with a service-level guarantee stronger than "usually similar". Trust therefore sets the *price* at which bytes and joules are sold. Today the trust premium is paid as a 1.6–2.1× slowdown on the same hardware. The strategic question is who collapses that premium first and where it ends up as margin.

**The identity.** Value per token = (useful bits delivered) / (bytes moved × joules per byte) × (fraction of tokens that can be trusted). FLOPs do not appear. That is the one-line summary of why the industry's forty-year product has stopped being the product.

---

## 3. The algorithmic lens: every system is a sufficient statistic and a forgetting rule

The companion analyses established, on one algorithm, a principle that generalizes to the whole stack.

### 3.1 What e-means taught

e-means (Meta FAIR, November 2026) is Lloyd's k-means with two modifications: it keeps a *decayed* count and sum per cluster instead of exact ones, and it anneals the decay on a cosine schedule. Rewritten as an optimizer it is gradient descent with an isotropic, stale-count preconditioner, $c_j' = c_j - (N_j')^{-1} I\,\nabla L$. On its home turf — a million to a billion embedding vectors in 96–3,000 dimensions on a GPU — it reaches converged-Lloyd quality in 1.4–2.6 epochs, 10–45× faster than the best GPU Lloyd. Those are real and large.

Outside that turf it fails in five specific ways, each traceable to the preconditioner: it has no notion of cluster shape (ARI 0.12 vs 0.47 for a Fisher-metric method at condition number 100); its scaling laws contain no dimension term and clip to full-replacement outside their fitted range (37% worse than Lloyd++ at $d = 4, 8$; destroys a good k-means++ init half the time); its annealed forgetting rate is an anti-tracking schedule under drift (2.2× the error of an exact online tracker); its output depends on batch order, revival RNG and BF16 ties (165% loss spread across six orders on identical data); and its revival heuristic is blind to merged clusters and produces identical children at a zero coordinate.

### 3.2 The grid

Lay the alternatives on two axes — *what sufficient statistics do you carry* and *how do you forget* — and the entire design space of online estimation appears:

```
                      forget: never        by epoch          constant rate        adaptive rate
count + sum           MacQueen/Sculley     Lloyd             e-means              (open)
count + sum + scatter —                    Mahalanobis k-means  Givens-QR tracker  Givens w/ adaptive λ
```

The first row is *multiply-native*: its hot loop is a GEMM, its state is a ratio of EMAs, it needs floating-point range, it has no invariant until a final Lloyd step. The second row is *rotation-native*: its state is a triangular factor updated by Givens rotations (CORDIC vectoring + rotation), it needs mantissa not range, it is positive-semidefinite by construction, every update is unimodular and therefore reversible, and it carries conservation laws (mass, first moment, PSD) that can be wired as a zero-cost residual monitor. Measured: in float16 the Givens factor holds 1.5–3.7% error while the Sherman–Morrison inverse overflows at step 0; in bfloat16 SM goes indefinite at half its checkpoints and diverges while Givens stays bounded.

### 3.3 Why this is a strategy lens and not a clustering footnote

Because the same two axes describe every component of an AI serving stack:

| system | sufficient statistic carried | forgetting rule | row |
|---|---|---|---|
| transformer KV cache | every key/value ever seen in context | never (until window end) | row 1, "never" — the most expensive cell |
| sliding-window / compressed KV | recent keys + summary | by window | row 1, "by epoch" |
| state-space / linear-attention models | a fixed-size recurrent state | constant rate (decay) | row 1, "constant rate" — e-means for sequence models |
| retrieval-augmented memory | an index (count + sum + *structure*) | adaptive (reindex on drift) | row 2 |
| per-channel / per-head quantization | first moment per channel | by calibration run | row 1 |
| Fisher-aware quantization (Hessian/covariance per channel) | second moment per channel | by calibration run | row 2 |
| first-order optimizer (Adam) | diagonal second moment | constant rate (β₂) | row 1.5 |
| Shampoo / Muon / natural-gradient | full or Kronecker second moment | constant rate | row 2 |
| online risk engine (HELICON-class) | Greeks, PDE residual | by tick | row 2 with invariants |

The pattern is exact. The industry's 2024–2026 movement — from Adam toward Shampoo and Muon in frontier training, from naive KV caches toward structured and compressed state, from per-tensor toward Hessian-aware quantization — is a migration from row 1 to row 2, driven by the same three ledgers: the second moment costs more to carry (bytes) but it is what lets you quantize harder, forget smarter and attest. **The preconditioner is the product.** Whoever carries the right metric at the lowest byte cost wins each cell, and the cells are being contested one at a time.

### 3.4 Scaling laws as products, and their failure class

e-means' most commercially attractive feature is that it ships with fitted scaling laws and therefore no hyperparameters. This is the right product instinct — a buyer wants a function, not a tuning job — and the right warning. The laws were fit at $n = 10^6$, $k = 1024$, $d = 96$, and silently extrapolate outside that envelope to a step of exactly 1 (full replacement, no memory). Nothing in the API says so. The algorithm does not refuse; it degrades.

This is a general failure class and it will be regulated. A model card describes a model; an *algorithm card* must describe the fitted envelope of any empirically tuned law inside a component, and the component must refuse or fall back outside it. The hardware frameworks already have the idiom: HELICON's convergence window guard asserts $w' \in [0.125, 8.0]$ at ingress and raises FAULT otherwise; QARE's ε-frontier states $C_c$, $C_q$ and the crossover in basis points so that a listener cannot assume the advantage is universal. The software world will adopt the same discipline, first voluntarily (Faiss-class libraries), then by standard (§8, P5).

---

## 4. The hardware lens: three engines, not two

### 4.1 The split that already happened

The 2026 consensus — GPUs for training and prefill, custom ASICs for decode — is correct and incomplete. It is organized around one ledger (bytes). Decode is memory-bound; an SRAM-first part with 10× the effective bandwidth per watt serves it better; the hyperscalers that own their models have built such parts and the merchant vendors (d-Matrix, Groq-via-NVIDIA, Qualcomm AI200, Positron) are selling them to everyone else. Prefill remains compute-bound and stays on the FLOP engine. This split is in production at Parasail, at Microsoft (Copilot traffic to Maia), at Google (Claude inference on Ironwood at seven-figure volume), and it will be the default pattern by 2027.

### 4.2 The third engine

What the two-engine model omits is the trust ledger, and the workloads that live on it:

- **Codebooks and quantizers.** Every IVF/PQ index, every VQ bottleneck, every KV-cache quantizer has a codebook, and a codebook is a contract: the same vector must map to the same code forever. e-means builds codebooks fast on the multiply engine and returns them without a transcript. The companion analysis shows why that is a liability and what the fix is (return the permutation seed, revival log, dtype and finalization with the codebook; or build the codebook on an order-invariant algorithm).
- **RL rollouts.** Post-training is now ~30× the compute of pre-training by some estimates, and it depends on the rollout path and the training path computing the same numbers. Zhang et al.'s bitwise vLLM/FSDP parity is the first demonstration; it is a software fix on hardware that was not designed for it, and it costs throughput.
- **Regulated decisions.** Credit, insurance, clearing, medical triage, public-sector eligibility. A clearing house switched its entire options model in one week in April 2020 because the market went negative and a logarithm has no value there (HELICON §6). These buyers do not want a faster model; they want an attestable one. HELICON's residual monitor and QARE's direction record are the pattern: a per-result invariant, computed beside the output, at line rate.
- **Agent memory and state.** Long-running agents accumulate state across sessions. That state is a sufficient statistic with a forgetting rule (§3.3), and the moment it is used to make a consequential decision it needs the same properties: reproducible, order-invariant or transcribed, attestable.
- **Evaluation itself.** An eval suite whose seed the provider quietly ignores is "always within one batch of being a different benchmark". The judge, the eval harness and the production path need identical bits.

These workloads share four requirements — exact arithmetic (fixed-point or batch-invariant float), order invariance or a logged transcript, conservation-law monitors, and bounded dynamic range — and none of them needs peak FLOPs. The hardware that satisfies all four is the hardware the three frameworks describe: fixed-point, rotation-native, pipelined, branchless, with a residual tap. Call it the **state engine**. It does not exist as a product category today; its components exist as FPGA designs, as HFT risk pipelines, as the deterministic-mode kernels now shipping in vLLM and SGLang, and as the `contrib/` directories of vector-search libraries.

### 4.3 What the state engine is and is not

It is: a fixed-point or batch-invariant datapath; rotation (Givens/CORDIC) and division native; triangular solves native; per-result invariants wired as hardware; a transcript output (direction bits, seed, schedule) as a first-class port; dynamic-range guards at ingress; and a tier placement (QARE's D0–D4) that says explicitly what latency class it serves. It owns *state*: codebooks, precision factors, risk registers, agent memories, eval references.

It is not: a training chip, a decode chip, or a general-purpose accelerator. It will not beat the multiply engine on GEMM (the frameworks concede ~13% on flat matmul and that is the honest number) and it will not beat the bandwidth engine on decode. Its throughput claim is *attested results per second per watt*, and on a speed-bumped or regulated path — where the mandated delay is a thousand times the pipeline (HELICON §12.3) — that is the only figure of merit left.

The cleanest division of labour the evidence supports: the multiply engine computes the *assignment* (GEMM-shaped), the state engine owns the *metric* (rotation-shaped). The rotation-native analysis costed this at $d = 96$, $k = 1024$: 4.7 thousand rotations per sample on the state engine against 4.8 million MACs on the multiply engine, with the state fitting in a few million words. The state engine is never the bottleneck; it is the authority.

### 4.4 Where it lives in the rack

Not as a fourth accelerator in every node. As the *D0 register* of QARE's tier table: the small, exact, always-on authority that the big engines read from and write back to on a cycle. In a 2029 inference rack: GPU prefill → ASIC decode → state engine for codebook lookup, KV-quantizer metric, agent-memory commit, and the per-result attestation that gets logged. Physically it is likely a chiplet or an FPGA lane beside the decode ASIC, on the same TileLink/UCIe fabric, with its own SRAM and no HBM — which is exactly what makes it buildable in a memory-starved market.

---

## 5. Memory strategy: the only ledger with a wall

### 5.1 The captive-customer problem

Three suppliers, 18–24-month fab lead times, a supplier base that has told investors it will not overbuild after the last three boom-bust cycles, and demand guidance of trillions. Every company building above the memory layer is a captive customer through 2028. The hyperscalers know this; it is why Microsoft's Maia 200 carries 216 GB of HBM3e (the largest in its class — they bought the memory, the chip is where they put it), why Amazon is scaling LPDDR and SRAM-heavy designs, and why Meta is "buying everything it can from both sources".

### 5.2 The four responses, ranked by leverage

1. **Eliminate traffic.** The highest-leverage response because it compounds and needs no supplier. On-chip transcendentals (no LUT fetch; the frameworks estimate 0.5% of traffic but disproportionate cache pollution), speculative and multi-token decode, KV compression and quantization, and — the one nobody is pricing — *shape-aware* quantization that uses the second moment to spend bits where the Jacobian says they matter (HELICON Law II applied to activations). Each of these reduces bytes per token by a constant factor, and constant factors are what you buy when the exponent is fixed by physics.
2. **Change the memory class.** SRAM-first decode (d-Matrix, Groq), LPDDR5X for capacity (AWS), GDDR7 for mid-tier bandwidth, on-package SRAM for codebooks and state. The HBM share of inference accelerator *units* will fall even as HBM revenue rises, because the units that can avoid it will.
3. **Own the allocation.** Multi-year take-or-pay contracts, co-investment in packaging lines (SK hynix Indiana, the TSMC co-development), and qualification of all three suppliers. This is table stakes, not strategy; it determines who survives 2027, not who wins 2030.
4. **Reprice the product.** Pass the $25B of component inflation through to token prices. This is happening and it is the least interesting response because it is the one everyone can do.

### 5.3 The metric that will replace FLOPs

Because bytes are the wall, **memory bytes per generated token** (weights + KV + auxiliary traffic, at the deployed precision) will become the headline number in accelerator announcements, the way "tokens per second per watt" is already displacing "TFLOPS". The frameworks' 137 GB/token traffic budget for a GPU-centric design (§III.3 of Topological Rotation) is one worked example; the correct way to read any 2027 datasheet will be: what is the byte count per token, what fraction of it is HBM, and what is the attested fraction.

---

## 6. Trust strategy: pricing the third ledger

### 6.1 What determinism costs now and why it falls

Batch invariance is a kernel property: reductions must use the same tree regardless of how many requests share the pass. Enforcing it costs 1.6–2.1× today on GPUs because the fast paths (split-K, tiling by batch dimension) are precisely the ones that break it. The cost falls through three mechanisms, in order: (i) optimized batch-invariant kernels (already 2.1× → 1.6×); (ii) verification rather than enforcement — compute fast, verify sparsely, repair on disagreement (LLM-42 and MarginGate, with the latter cutting the verification increment roughly in half); (iii) hardware that is batch-invariant by construction, i.e. fixed-point or systolic datapaths whose reduction order is fixed at synthesis. The third mechanism is the state engine, and it is where the premium goes to near zero.

### 6.2 Arithmetic determinism is not algorithmic determinism

The single most important finding in this folder for the trust ledger. e-means on bit-exact arithmetic returns six different results for six batch orders on identical data. Deterministic kernels fix the *arithmetic* source of irreproducibility; they do nothing about the *algorithmic* sources — data order, RNG, tie-breaking, revival. Those require either an order-invariant algorithm (Lloyd, whose M-step is a permutation-invariant sum) or a transcript sufficient for exact replay (QARE's direction record; e-means' permutation seed + revival log + dtype + finalization).

Reproducibility = deterministic arithmetic ∧ (order-invariant algorithm ∨ logged transcript).

Any "reproducible AI" claim that does not state which disjunct it satisfies is incomplete, and most current claims are. The hardware frameworks tested here get it half right (bit-exact arithmetic) and the e-means evidence supplies the other half. The combination — exact arithmetic plus transcript-carrying algorithms — is what the state engine sells.

### 6.3 Who pays, and how much

Three buyer classes with three willingness-to-pay profiles:

- **RL labs.** Train/inference mismatch is load-bearing; they will pay the full current premium (1.6–2×) on their rollout fleet now and migrate to batch-invariant-by-construction hardware as it appears. Size: a fraction of post-training compute, which is itself a multiple of pre-training compute. This is the anchor customer.
- **Regulated enterprises.** Will pay a modest premium (10–40%) for an attested SKU with a transcript, because the alternative is not being allowed to deploy. Size: the long tail of enterprise inference, growing with agentic adoption.
- **Everyone else.** Will not pay, and will receive determinism for free around 2030 when the hardware makes it the default. The premium collapses to a feature.

The strategic consequence: the trust premium is a *window*, 2027–2030, during which attested inference is a differentiated product. Clouds that ship it first capture the RL-lab and regulated-enterprise demand; silicon that makes it free ends the window. Both are good businesses; they are different businesses.

### 6.4 The attestation primitive

HELICON's residual monitor and QARE's `Estimated` type generalize to a single primitive that the whole stack lacks: every result carries (value, invariant residual, transcript handle). For an LLM token: the token, the batch-invariance witness (hash of the reduction tree used), and the seed/schedule handle. For a clustering output: the codebook, the mass/first-moment residuals, the permutation and revival log. For a risk figure: the Greeks, the PDE residual, the sequence number. The primitive is cheap — adders beside the output stage — and it is what converts a fast pipeline into a sellable one. Expect it to be standardized (§8, P5, P12).

---

## 7. Playbooks by actor

### 7.1 Hyperscaler with captive models

You own the model, the serving stack and the chip. Your binding constraint is bytes, your opportunity is trust.

- Split the fleet three ways (multiply / bandwidth / state) and route by *phase and trust class*, not by model. Prefill to GPU; decode to the SRAM-first part; codebook, quantizer metric, agent-memory commit and attestation to a fixed-point lane beside the decode part.
- Make batch-invariant kernels the default on the decode part by 2027 and sell the difference as an SKU until it is free.
- Ship second-moment-aware quantization (per-channel covariance, Fisher-weighted bit allocation) for KV and weights; this is the "eliminate traffic" lever that does not need a supplier.
- Treat every empirically fitted law inside your stack (batch-size heuristics, learning-rate schedules, quantization calibrations) as an algorithm with an envelope; guard the envelope at ingress; log the guard. This is cheap now and mandatory by 2029.
- Do not build row-2 state at codebook scale ($k \ge 10^6$, $d \sim 10^2$): the second moment does not fit. Build it at $k \le 10^4$, where every non-embedding workload lives.

### 7.2 Merchant silicon (GPU incumbent)

Your product is FLOPs and your customers are leaving the part of the market that no longer needs them.

- The licensing-in of a decode architecture was correct; the next move is a *state* architecture, and it is cheaper: a fixed-point, rotation-native, batch-invariant-by-construction chiplet on the same fabric, sold on attested-results-per-watt. You have the fabric and the software; what you lack is the discipline of invariants, and that is purchasable.
- Re-headline. The datasheet that leads with bytes-per-token at deployed precision and the attested fraction will be read; the one that leads with PFLOPS at FP4 will be discounted by every buyer who has read a 2026 memory supply note.

### 7.3 Inference-ASIC challenger

You win decode on bandwidth per watt. Your risk is that the hyperscalers' captive parts take the volume and leave you the neoclouds.

- Own the *heterogeneous deployment layer* (you already know this: the GigaIO acquisition, SquadRack, Aviator). The customer's barrier is operational complexity, not performance.
- Add the state engine as the third element of your rack before the incumbents do. A fixed-point attestation lane is small, needs no HBM, and turns "faster decode" into "faster *and* auditable decode" — which is the only pitch that gets you into regulated enterprises, where captive hyperscaler silicon cannot be rented anyway.
- Publish your crossover. QARE's ε-frontier discipline — state the classical cost, your cost, and the regime where you win in the customer's unit — is more persuasive than a 10× headline and survives contact with a procurement engineer.

### 7.4 Enterprise buyer

You rent compute and you are about to be asked by a regulator what your model did.

- Buy by trust class, not by throughput. Classify every workload D0–D4 (authoritative / real-time / cycle / batch / fan-out) and demand the attested SKU for D0–D2.
- Require algorithm cards: fitted envelope, failure modes, transcript format. Refuse components that cannot state their envelope. The e-means laws are the cautionary example; they are also the template for what a good card looks like once the envelope is added.
- Standardize features before clustering, quantizing or embedding — the cheapest defence against the shape failure, and the one e-means and Lloyd both lack.
- Keep an order-invariant reference (a Lloyd, a batch solve, a fixed-point replay) for every stochastic fast path. Reconcile nightly. The reconciliation residual is your audit artefact.

### 7.5 Financial institution, clearing house, exchange

The frameworks in this folder were written for you and they are mostly right about you.

- The 2020 negative-oil episode is the proof that model assumptions embedded in a datapath do not update on an advisory; the metric mux (log-space / price-space / Lorentz) is the hardware form of model-agnosticism, and it is two percent of area.
- Mandate the per-result invariant. A PDE residual or a ν/Γ identity beside every Greek set costs four adders and makes a line-rate pipeline a different risk object. This will become a clearing-rule requirement (§8, P12); be the venue that writes it.
- Place quantum where the ε-frontier says and nowhere else (D2 collateral, one expectation, intraday); place fixed-point deterministic engines at D0/D1; place GPUs at D3/D4. QARE's placement is correct and it is unusually honest about where quantum loses.

### 7.6 Sovereign and public buyer

Your constraint is supply security and auditability; your advantage is patience.

- Memory is the choke point with three suppliers and no fourth entrant before 2029. Co-invest in packaging and qualification, not in fabs you cannot fill.
- Fund the state engine as public infrastructure. Attested, fixed-point, order-invariant compute is what public-sector decisions legally require and what no vendor is yet selling as a product. It is small, it does not need HBM, and it is the one tier that can be domestically produced on trailing nodes.
- Write the algorithm-card standard before the vendors write it for you.

---

## 8. Twelve predictions

Stated so they can be wrong. Confidence is subjective and honest.

**P1 — Attested inference becomes a priced SKU.** By 31 December 2028, each of the three largest public clouds lists an inference option with bitwise reproducibility and a replay transcript, at a price premium of at least 1.3× over standard. By end-2030 the premium is below 1.1× because batch-invariance has become the hardware default. *Falsifier:* no such SKU at two of the three clouds by end-2028. *Confidence:* high. *Why:* the RL-lab anchor demand exists now, the kernels exist, the cost is already down to 1.6×, and the regulated-enterprise pull is structural.

**P2 — Three engines, not two.** By 2030 a recognizable product category exists — fixed-point or batch-invariant-by-construction, rotation/division-native, invariant-monitored, transcript-emitting — sold on attested results per watt rather than FLOPs, with at least two merchant vendors and one hyperscaler captive part. Its share of inference-adjacent silicon spend is 5% or more. *Falsifier:* by end-2030 no accelerator datasheet from a top-ten vendor headlines reproducibility or invariant monitoring as a primary spec. *Confidence:* medium-high.

**P3 — Bytes per token replaces FLOPs as the headline.** By end-2028, at least two of the four hyperscaler custom-silicon announcements lead with a memory-traffic-per-token or tokens-per-watt figure and relegate peak FLOPS to a secondary line. *Falsifier:* all four 2028 announcements lead with PFLOPS. *Confidence:* high. *Why:* memory is sold out through 2027 and every buyer knows it; FP4 PFLOPS already reads as a vanity number.

**P4 — HBM's unit share of inference accelerators falls below 50% by 2029.** Revenue share rises (price), unit share falls (SRAM-first, LPDDR, GDDR7, on-package SRAM for state). *Falsifier:* HBM on more than half of inference accelerator units shipped in 2029 by a credible tracker. *Confidence:* medium. *Risk:* HBM4 capacity arrives faster than guided and prices fall.

**P5 — Algorithm cards with validity envelopes become a standard requirement.** By end-2029 at least one regulator or standards body (an EU AI Act implementing act, a NIST profile, a financial supervisor) requires, for ML components in high-risk decisions, a statement of the fitted envelope of any empirically tuned law and the component's behaviour outside it. Extrapolation of fitted scaling laws without a guard — the e-means $\alpha_0 = 1$ clip — is a named defect class. *Falsifier:* no such requirement in any binding instrument by end-2029. *Confidence:* medium.

**P6 — Codebooks ship with transcripts.** By end-2028 the dominant vector-search and quantization libraries return, with every learned codebook, the information sufficient for exact replay (seeds, schedule, precision, finalization) and refuse to serialize without it; order-invariant finalization (a full Lloyd pass or equivalent) is the default for codebooks that will be deployed. *Falsifier:* Faiss-class libraries still return bare centroid arrays as the default in 2029. *Confidence:* medium-high. *Why:* it is cheap, the failure mode is now documented, and the first production incident traced to an irreproducible codebook will make it mandatory.

**P7 — Hyperbolic and mixed-curvature inference stays niche through 2029.** Fewer than 2% of production tokens are generated by models with non-Euclidean geometry in their main path, despite the primitives being cheap and the Lorentz centroid being a one-step hyperbolic vectoring op (demonstrated in this folder). *Falsifier:* a hyperbolic or mixed-curvature architecture in the top ten of open-weight downloads, or in a frontier lab's production API, by end-2028. *Confidence:* medium. *Why:* the geometry is real but the Euclidean stack's momentum, tooling and quantization ecosystem are overwhelming; curvature will arrive first in retrieval and memory (the state engine), not in the main transformer path. This is a deliberate disagreement with the Topological Rotation framework's P3.

**P8 — Train/inference parity becomes a stated property of every frontier RL pipeline.** By end-2027, every frontier lab's public description of its post-training stack claims bitwise or verified parity between rollout and training numerics. *Falsifier:* two or more frontier labs publicly describing post-training in 2028 without such a claim. *Confidence:* high. *Why:* the mismatch is a known source of RL instability, the fix exists, and the cost is tolerable at post-training scale.

**P9 — The capex curve bends in 2028.** Combined hyperscaler capex growth falls below 20% year-on-year in 2028 (from ~78% in 2026 and an expected 30–40% in 2027) — not because token demand slows but because bytes per token fall faster than tokens rise, and because the memory wall caps what can physically be bought. *Falsifier:* 2028 growth above 30%. *Confidence:* medium. *Risk:* an agentic demand step-change that outruns efficiency; or memory relief arriving early and unlocking deferred purchases.

**P10 — Fixed-point returns to the data center.** By end-2029 at least one hyperscaler custom part or one merchant decode part publicly documents an integer/fixed-point-only execution path with a bit-exactness guarantee across batch size and parallelism degree. *Falsifier:* no such documented path by end-2029. *Confidence:* medium. *Why:* it is the cheapest way to make batch invariance free, and the HFT and risk world has run it at line rate for a decade.

**P11 — The second moment moves into production quantization by 2027.** KV-cache and weight quantizers in at least two frontier serving stacks allocate bits using per-channel (or per-head) second-moment / Fisher information rather than first-moment ranges alone, and advertise it. *Falsifier:* none of the top serving stacks document Hessian- or covariance-aware KV quantization in production by end-2027. *Confidence:* high. *Why:* this is the lowest-cost "eliminate traffic" lever that exists, the research is mature, and the migration from row 1 to row 2 is already under way in optimizers.

**P12 — Per-result invariant attestation becomes a clearing or venue rule.** By end-2030 at least one clearing house, exchange or prudential supervisor requires, for automated pricing or risk systems on its critical path, a per-result conservation-law check (PDE residual, identity check or equivalent) logged at line rate. *Falsifier:* no such rule in force anywhere by end-2030. *Confidence:* medium-low. *Why:* it is the only correctness evidence that exists at line rate, the hardware cost is adders, and the first large loss traced to a silent pipeline fault will make it unavoidable — but regulators are slow, and this one may land in 2031–32.

A thirteenth, offered without a date because it is the thesis: **the accelerator industry's unit of account changes from the multiplication to the attested byte.** When a vendor can tell you how many bytes a trusted token costs, and nothing else, the transition is complete.

---

## 9. Unit economics of one token across three ledgers

Strategy without a cost model is opinion. Here is the model, with the arithmetic shown so that every assumption can be replaced.

### 9.1 The 2026 token

Take a 70B-parameter dense model served at 8-bit weights, 4k-token context, batch of 64 concurrent requests, on a GPU-centric rack with HBM at ~8 TB/s aggregate across the node.

*Bytes.* Decode is one forward pass per token. Weights: 70 GB at 8-bit, read once per decode step and shared across the batch, so ~1.1 GB per token at batch 64. KV cache: for 70B-class architectures roughly 0.3–0.5 MB per token of context at 8-bit (depends on grouped-query head count); at 4k context that is ~1.2–2 GB per request per step, *not* shared across the batch — call it 1.6 GB per token generated. Auxiliary (activations, LUT fetches, attention scratch): ~5–10% on top. Total ≈ **2.8 GB moved per generated token**, roughly 60% of it KV cache, 40% weights. The frameworks' 137 GB/token figure (Topological Rotation §III.3) is for batch 1; at batch 64 the weights amortize and the KV term dominates, which is the more important observation: **KV traffic is the part that does not amortize**, and it grows linearly with context.

*Joules.* HBM3e moves data at roughly 3–4 pJ/bit at the package boundary, more once you include the controller and on-die fabric; call it 5 pJ/bit end-to-end. 2.8 GB × 8 × 5 pJ ≈ **0.11 J of memory traffic per token**, before any compute. The FLOPs for the same token (2 × 70B ≈ 140 GFLOP) at ~0.5 pJ/FLOP effective for a dense FP8 pipeline cost ~0.07 J. Memory movement already exceeds arithmetic in the energy budget of a single decode token, and the ratio worsens with context length. This is the quantitative content of "bytes and joules are the same ledger".

*Trust.* On the default serving path, batch invariance is absent; the deterministic path costs 1.6× throughput. If the token is sold at $p$ per million, the attested token costs $1.6p$ in compute-equivalent today. The fraction of tokens *that need* attestation is small (RL rollouts, regulated decisions, evals) — perhaps 5–10% of volume — but they are the tokens with the highest willingness to pay.

### 9.2 The 2029 token

Same model class, same context, with the levers of §5.2 applied and the three-engine split of §4.

*Bytes.* Weights at 4-bit with shape-aware (second-moment) allocation: 35 GB, amortized at batch 64 → 0.55 GB. KV at 3–4 bits effective with per-head covariance-weighted quantization plus eviction of low-information keys (a row-2 statistic deciding what to forget): ~0.4 GB per token. On-chip transcendentals and fused attention remove most auxiliary traffic. Total ≈ **1.0 GB per token** — a 2.8× reduction with no change to the model, all of it from carrying a better statistic (the second moment) and forgetting by information rather than by position.

*Joules.* SRAM-first decode moves the hot KV window at ~0.1–0.3 pJ/bit instead of 5; weights still stream from off-package memory (LPDDR or HBM) but at a third the volume. Blended: ~1.5 pJ/bit × 1.0 GB × 8 ≈ **0.012 J per token**, roughly a 9× reduction from 2026 on the memory line, with compute roughly flat. The decode token is now compute-bound again at the margin, which is exactly the regime where the bandwidth engine's SRAM investment pays.

*Trust.* The state engine's fixed-point attestation lane and batch-invariant-by-construction decode kernels bring the deterministic path to ~1.05× the fast path. The attested token's premium collapses from 60% to 5%, and its share of volume rises because the price no longer excludes it.

### 9.3 What the model says about strategy

- Of the 2.8× byte reduction, roughly 1.7× comes from *precision* (8→4 bit) and 1.6× from *what you carry and forget* (covariance-aware allocation and information-based eviction). The second lever is the one that requires the second moment — row 2 of the grid — and it is the one no supplier controls. It is pure architecture.
- Of the ~9× joule reduction, most comes from *where the bytes live* (SRAM vs HBM), i.e. the bandwidth engine. That lever is capital-intensive and supply-constrained; the precision/statistics lever is neither.
- Trust is the only ledger whose cost goes to near zero rather than to a physical floor, because its cost today is a software artefact (reduction-order variance) and not a physical one. That is why it is the ledger with the largest *margin* change over the period even though it is the smallest in absolute energy.

Replace any number above with your own; the ranking of levers does not change unless HBM energy per bit falls by 5× or KV traffic stops dominating, and neither is on any public roadmap through 2029.

---

## 10. Roadmap, 2027–2032, and what to measure

### 10.1 Year by year

**2027 — the heterogeneous default.** Prefill/decode disaggregation becomes the standard rack pattern at hyperscalers and neoclouds; merchant decode parts (d-Matrix, Groq-in-NVIDIA, Qualcomm, Positron) ship in volume; HBM remains fully allocated; DDR5 prices stay elevated. Batch-invariant kernels ship as an opt-in mode in the major serving stacks. Second-moment-aware KV quantization enters production (P11). Frontier RL pipelines claim train/inference parity (P8). Capex grows 30–40% to around $1 trillion. *Watch:* the first production incident publicly attributed to an irreproducible learned component — a codebook, a quantizer, a router — which will be the catalyst for P6.

**2028 — the metric flips.** At least two hyperscaler silicon announcements headline bytes-per-token or tokens-per-watt (P3). Attested inference is a priced SKU at the major clouds (P1). Codebooks ship with transcripts by default (P6). First meaningful HBM capacity relief late in the year; prices stop rising, do not fall. Capex growth falls below 20% (P9) as the §9 byte reductions land across the installed base. *Watch:* whether the attested SKU premium is above or below 1.3× — above means the state engine has a market; below means GPUs closed the gap in software and P2 weakens.

**2029 — the third engine appears.** A fixed-point or batch-invariant-by-construction part with invariant monitoring is documented by at least one hyperscaler or merchant vendor (P10). HBM unit share of inference accelerators drops below half (P4). A regulator names algorithm envelopes in a binding instrument (P5). Hyperbolic architectures remain under 2% of production tokens (P7) but appear in retrieval and memory subsystems on the state engine. *Watch:* whether the state engine is a chiplet beside the decode ASIC (this document's expectation) or a standalone card (slower adoption, higher margin).

**2030 — trust becomes free, then mandatory.** The attested premium falls below 1.1×; attestation moves from SKU to default. The state engine reaches ~5% of inference-adjacent silicon spend (P2). A clearing house or supervisor mandates per-result invariant logging on a critical path (P12, with slippage risk to 2031–32). *Watch:* the first accelerator datasheet whose primary spec is "attested results per second per watt".

**2031–2032 — the unit of account changes.** Vendors quote bytes per trusted token. Memory supply has caught up and the bytes ledger loosens; the joules ledger (power delivery, cooling) becomes the binding one; the trust ledger is priced into every token as a default property. The multiply engine is a training appliance; the bandwidth engine is the decode commodity; the state engine is the authority tier that the other two read from. The accounting identity of §2 is the industry's P&L.

### 10.2 The scorecard

Twelve quantities that, tracked quarterly, will tell you whether this document is right before the predictions resolve.

| # | metric | 2026 reading | direction if thesis holds | source |
|---|---|---|---|---|
| 1 | Bytes moved per generated token, blended fleet | ~2.8 GB (model of §9.1) | falls 2–3× by 2029 | your own telemetry; vendor disclosures |
| 2 | Share of decode tokens served off-GPU | low single digits | >30% by 2028, >50% by 2030 | cloud and neocloud disclosures |
| 3 | Attested-inference price premium | 1.6–2.1× (compute-equivalent) | <1.3× by 2028, <1.1× by 2030 | cloud price lists; serving-stack benchmarks |
| 4 | Share of tokens served under a deterministic/attested mode | ~0 (opt-in, rare) | 5–10% by 2028, default by 2030 | serving-stack telemetry |
| 5 | HBM unit share of inference accelerators | majority | <50% by 2029 | analyst trackers |
| 6 | DRAM contract-price trend and lead times | rising; >40 weeks | plateau 2028, ease 2029 | supplier calls; distributor lead-time reports |
| 7 | Hyperscaler capex growth, y/y | ~78% | 30–40% (2027), <20% (2028) | 10-Qs |
| 8 | Custom ASIC unit share vs merchant GPU | ~28% of AI servers | crosses 50% ~2028 | TrendForce / Omdia |
| 9 | Serving stacks with second-moment-aware KV quantization in production | 0–1 | ≥2 by end-2027 | release notes; papers |
| 10 | Learned components shipped with a replay transcript (codebooks, quantizers, routers) | rare | default in major libraries by 2028 | library APIs |
| 11 | Binding instruments naming algorithm envelopes or per-result attestation | 0 | ≥1 by 2029; ≥1 venue rule by 2030 | regulatory registers |
| 12 | Production tokens on non-Euclidean main paths | ~0 | stays <2% through 2029 | model-release tracking |

If metrics 1, 3 and 7 move as stated by end-2028, the thesis is substantially right regardless of how the silicon category question (metric 2 and P2) resolves. If metric 3 collapses *without* new hardware, the three-engine claim is wrong in form but the three-ledger claim is right in substance: trust was still the product; it was just delivered in software.

---

## 11. What would break this

**Memory relief arrives early.** If HBM4 yields and the new packaging lines come up in 2027 rather than 2028–29, the bytes ledger loosens, the "eliminate traffic" premium falls, and P4 and P9 slip by a year or two. The three-engine thesis survives (trust does not depend on memory) but its timing softens.

**Determinism gets cheap on GPUs without new hardware.** If batch-invariant kernels reach parity with the fast path (the 1.6× becomes 1.05×) by 2027, the state engine loses its first anchor customer and P2 weakens to "a software SKU, not a silicon category". This is the most plausible breaker. The counter-argument is algorithmic determinism (§6.2): kernels fix arithmetic, not order, and the transcript/invariant discipline still needs a home.

**Agentic demand outruns efficiency.** If tokens per user-task grow by more than the byte-per-token efficiency gains — plausible with long-horizon agents — capex does not bend in 2028 (P9 fails) and the memory wall becomes a hard ceiling on deployment rather than a cost. In that world the "eliminate traffic" levers are more valuable, not less; P3 and P11 strengthen.

**Geometry wins.** If a mixed-curvature architecture delivers the 4–8% iso-parameter gains the Topological Rotation framework predicts and it reproduces at 70B scale, P7 fails and the rotation-native engine gets a *main-path* workload, which would be a far larger market than the state-engine thesis assumes. This document bets against it on timing, not on physics.

**Regulators do nothing.** P5 and P12 are the ones that depend on institutions rather than engineering. If no binding instrument names algorithm envelopes or per-result attestation by 2030, the trust ledger stays a private premium rather than a public requirement. The engineering thesis is unchanged; the market is smaller.

---

## 12. A closing frame

Three documents in this folder test a 2026 algorithm against three 2026 hardware frameworks, and the result is not that one side wins. It is that they describe the same two axes from opposite ends. The algorithm chose to carry less and forget by schedule, and bought speed on the multiply engine at the price of shape, drift, exactness and small data. The frameworks chose to carry invariants and forget nothing, and bought determinism at the price of 13% on flat matmul. Neither choice is wrong. Each is wrong *outside its envelope*, and the industry's strategic error for the next five years will be the same error e-means' packaging makes: calling a regime-specific win a replacement.

The three ledgers are the envelope. Bytes say where the wall is. Joules say what it costs to approach it. Trust says what the result is worth once you get there. FLOPs are not on the list because they are no longer scarce, and a strategy built on an abundant input is a strategy built on last decade's product.

---

## 13. Sources

Market and supply (accessed November 2026)
- Hyperscaler 2026 capex guidance: Amazon ~$220B (Q3 release, 29 Oct 2026); Alphabet $195–205B; Microsoft ~$170B incl. finance leases (FY27 Q1 call, 28 Oct 2026); Meta $135–145B (Q3 release, 21 Oct 2026). Trackers: walnutinvest.com/stats/ai-capex-statistics; valueaddvc.com/ai-spending; nextmsc.com hyperscaler capex comparison 2026; analysis-atlas.com hyperscaler-ai-capex-2026.
- Gartner 2026 AI spending ($2.67T; infrastructure $1.484T; inference 55% of AI-optimized IaaS in 2026, 59% in 2027): rcpmag.com, Sept 2026.
- HBM/DRAM supply: intuitionlabs.ai HBM-DRAM analysis (Samsung HBM4 sold out, SK hynix 2026 committed, new capacity 2028–29); nand-research.com May 2026 and Dec 2025 updates (3:1 HBM:DDR5 wafer ratio; lead times >40 weeks; supplier capital discipline); fusionww.com CoWoS/HBM constraints; sourceability.com memory shortage.
- Inference share of compute (~⅔ in 2026, Deloitte; 80–90% lifecycle): metavert.io, saturncloud.io, Futurum Research (inference 71.7% of AI platforms infrastructure spend by 2030).
- Custom ASIC vs GPU (44.6% vs 16.1% shipment growth; ASIC servers ~28%; Omdia unit crossover 2028; Broadcom $8.4B AI revenue Q1 FY26; Marvell $11B 2026 guide; Maia 200, TPU 8t/8i, Trainium 3, MTIA 300, OpenAI–Broadcom 10 GW): Tom's Hardware "custom AI ASIC state of play" (May 2026); introl.com custom-silicon inflection (Feb 2026); tech-insider.org custom silicon race.
- d-Matrix Corsair full production (June 2026) and Parasail heterogeneous deployment (July 2026): d-matrix.ai; HPCwire; Converge Digest.
- Determinism: Thinking Machines Lab batch-invariance result (late 2025; 1,000 runs → ~80 completions; 2.1× → 1.6×); Zhang et al., *Deterministic Inference Across Tensor Parallel Sizes That Eliminates Training–Inference Mismatch*, arXiv:2511.17826; Chu, Zhou et al., *MarginGate*, arXiv:2605.30218; tianpan.co on provider seed handling (June 2026); hidekazu-konishi.com on where bitwise identity is required (Aug 2026).

Algorithmic evidence (this folder)
- `README.md` — e-means as preconditioned gradient descent; five failure modes; decision matrix; `experiments.py`, `results.json`.
- `README_ROTATION.md` — Sherman–Morrison vs Givens under reduced precision; conservation laws; Lorentz-centroid EMA; GYRE design; `rotation_experiments.py`, `rotation_results.json`.
- Bessa et al., *e-means: outperforming k-means (in one-tenth the epochs)*, arXiv:2611.09501 (Nov 2026).
- Lederman et al., *The Catastrophic Failure of the k-Means Algorithm in High Dimensions*, arXiv:2602.09936 (ICML 2026).
- Chen, Bayrooti, Morad, *Natural Gradient Expectation Maximization*, arXiv:2602.10602 (2026).

Hardware frameworks tested
- *Topological Rotation: The Weyl-CORDIC-Semimetal Unified Framework*, v2.0 (June 2026).
- *HELICON: Hyperbolic · Euclidean · Linear Iterative CORDIC Option Network*.
- *QUADRANT + QARE: Reversible CORDIC Ladder … for Coupled-Path Amplitude Estimation*.
