# QoS-Aware Data Movement Subsystem — RTL Design and UVM Verification

**Revision:** 0.2 / 05 September 2026
**Supersedes:** RTL_UVM_Project_Plan v0.1
**Author context:** One engineer with RTL design experience, beginning structured verification. 20 focused hours per week.

---

## What changed from v0.1, and why

| Change | Reason |
| --- | --- |
| Reframed as a **QoS-aware data movement subsystem with bounded latency**, not "a DMA subsystem" | Same RTL, different positioning. The first reads as a student project; the second describes a research contribution. Decide this before writing the README, because it shapes what you build. |
| **CDC moved into the core path** (now M6), out of the optional extension slot | Single-clock-domain designs read as academic. Async FIFOs, synchronizer constraints and reset coordination are interview table stakes. Optional-at-the-end means it won't happen. |
| **Non-ideal memory model added at M2** | With single-cycle BRAM there is nothing for a scheduler to schedule around. Variable service time is what creates QoS problems in the first place. |
| **Arbitration made a swappable module at M4**, with priority classes and regulator hooks present from the start | If a fixed round-robin arbiter is frozen into the RTL, the research campaign becomes a redesign instead of a configuration sweep. |
| **Telemetry from day one** (M2), not retrofitted | You cannot research what you did not measure. Retrofitting observability is painful and gets skipped. |
| **Compute client reshaped as a realistic traffic generator** (M5) | A better math unit teaches nothing about memory subsystems. A GEMM-tile access pattern is a genuinely useful load on the fabric. |
| **Literature track runs from week 1**, not at M9 | Reading prior work at the end is the wrong order. It shapes the design and tells you 18 months earlier whether the idea is already published. |
| **Timeline extended to ~8–12 months** for the first release | First UVM environments always take longer than estimated. Padding now prevents the discouragement that kills projects at month six. |
| **Visibility and external review added as explicit tracks** | Reputation compounds slowly. Start the clock early. An hour of review from a working engineer redirects more effort than a month solo. |

---

## Claim boundary

This is a proposed plan. It does not claim that RTL has been implemented, verified, or made ready for ASIC tapeout. Effort figures are engineering planning estimates for one engineer who already knows RTL and is learning verification while building. Reforecast after M1 and M3 against actual logged hours.

---

## 1. What is being built

A data-movement subsystem where quality of service is a first-class design property rather than an afterthought:

- **Two DMA channels** with configurable priority classes, moving data over AXI4.
- **A banked scratchpad SRAM** (32 KiB, four 64-bit banks) with observable bank-conflict behavior.
- **A pluggable arbitration and QoS layer** — the research surface of the project.
- **A compute client** whose access pattern models GEMM tile consumption, acting as a realistic traffic load.
- **A second clock domain** for the compute client, crossed with properly constrained async FIFOs.
- **A non-ideal memory model** in simulation with variable service latency, bank conflicts and refresh stalls.
- **Telemetry throughout**: latency histograms, stall-cause counters, queue occupancy, dumpable traces.
- **Software control** via an unchanged Ibex RISC-V core, running C drivers.
- **One FPGA-qualified configuration** with documented operating limits.

### Frozen baseline (confirm feasibility in M0 before locking)

| Item | Proposed baseline |
| --- | --- |
| Control interface | 32-bit AXI4-Lite registers; aligned accesses, defined byte-strobe behavior |
| Data interface | 64-bit AXI4 data, 32-bit address, INCR bursts, max 16 beats |
| Channels | One in M2; two in M4. Bounded credit/ID scheme from M3, initially 4 outstanding read bursts globally |
| QoS classes | Two priority classes minimum (urgent / bulk); arbiter interface supports N classes |
| Transfers | Linear, 8-byte aligned, length divisible by 8. Split at burst limits and 4 KiB boundaries |
| Scratchpad | 32 KiB across four 64-bit banks; mapping and access latency frozen at M0 |
| Compute client | Signed INT8 dot product, length 1–256, signed 32-bit result — but issued in a tiled, strided access pattern |
| Clock domains | One domain M0–M5; second domain for compute client from M6 |
| Memory | Uncached, non-coherent shared buffers with explicit software ownership. Parameterizable latency model in simulation; FPGA BRAM on hardware |

### Explicitly out of scope for the first release

Cache coherence, atomics, a DDR controller, Linux, scatter/gather, 2D transfers, unaligned transfers, CPU microarchitecture changes, source/destination alias races across clients, and a full neural network accelerator. Each is a separately estimated extension, not a stretch goal.

---

## 2. The three new technical pillars

These are the additions that turn a competent project into a research vehicle. Each must be built early, because each is expensive to retrofit.

### 2.1 Non-ideal memory model (introduced M2)

Build a parameterizable memory endpoint model in SystemVerilog, driven by a configuration object:

- Base read and write latency, with a configurable distribution (fixed, uniform, or trace-driven).
- Bank state: row-buffer hit / miss / conflict with distinct service times.
- Periodic refresh stalls.
- Bounded request queue with backpressure.
- A "hostile mode" that maximises conflicts, for worst-case testing.

Calibrate the distribution against published DRAM timing or traces from Ramulator or DRAMSim. You do not need cycle-accurate DRAM fidelity. You need *credible variable service time*, so the arbiter faces a scheduling problem rather than a fixed pipeline.

The FPGA target still uses BRAM. State this clearly in the release notes: the simulation model provides the QoS research environment; the hardware run demonstrates functional correctness and timing closure.

### 2.2 Pluggable arbitration and QoS layer (introduced M4)

Define an arbiter as a module behind a stable interface, not as inline logic:

```
Inputs per requester:  request valid, priority class, deadline hint (optional),
                       bytes remaining, age counter
Outputs:               grant one-hot, grant class, regulator state
Config:                per-class weights, per-class rate limit,
                       max consecutive grants, tie-break policy
```

Ship at least three implementations behind this interface:

1. **Round-robin** — the naive baseline.
2. **Strict priority with starvation guard** — the common industrial baseline.
3. **Weighted / deficit round-robin with per-class bandwidth regulators** — modelled on Arm AMBA QoS regulator behaviour.

The research candidate becomes a fourth implementation. Because the interface and the regression suite already exist, evaluating it is a configuration sweep with correctness already proven, not a rebuild.

### 2.3 Telemetry and observability (introduced M2, extended throughout)

Instrument from the first transfer. Minimum set:

| Counter / structure | Purpose |
| --- | --- |
| Per-channel latency histogram (request accept → final response) | Tail latency is the whole research question |
| Stall-cause counters (no credit, arbiter loss, bank conflict, downstream backpressure, response queue full) | Attribution — knowing *why* something stalled |
| Queue occupancy high-water marks | Resource bounds and sizing evidence |
| Bank conflict counts, per bank | Scratchpad mapping experiments |
| Grant counts per class, regulator state | Verifying the arbiter does what it claims |
| Bytes moved, per channel and per class | Throughput and fairness accounting |

Add a trace dump mechanism: timestamped events written to file in simulation, readable by a Python analysis script. Keep the same counters available as CSRs on hardware so the FPGA run is measurable too.

---

## 3. Ownership: what you write versus what you reuse

| Category | Content |
| --- | --- |
| **Reuse and pin** | PULP iDMA backend (selected configuration), applicable PULP AXI components, unchanged supported Ibex configuration |
| **Design and verify yourself** | Command frontend, channel scheduler, **arbitration and QoS layer**, credit and ID tracking, completion accounting, scratchpad integration and bank decode, **CDC infrastructure**, compute client, **telemetry**, **memory model**, all glue logic |
| **Replace deliberately** | One bounded upstream block, only when a learning or measured performance goal justifies it. Keep the original for comparison |
| **Package separately** | Portable RTL and interfaces; upstream dependencies; verification; software; FPGA target wrappers and constraints |

Write a short behaviour contract before touching any module: accepted inputs, output meaning, latency and throughput, reset behaviour, error cases, parameter limits. Derive the checker from the contract, not from the RTL. This is the single habit that separates the durable half of this project from the perishable half.

---

## 4. Verification architecture

| Component | Responsibility |
| --- | --- |
| **Agents and RAL** | AXI4-Lite control, AXI memory endpoint behaviour, competing-client traffic generation, clock and reset, interrupt monitoring. Register prediction observes bus effects |
| **Reference models** | Independent byte-addressable memory and transfer model; integer compute oracle; **QoS service model** predicting which requester should win under the declared policy |
| **Scoreboards** | Descriptor / channel / ID context, permitted ordering, data conservation, destination correctness, error and terminal status |
| **Assertions** | Handshake stability, resource accounting, one-hot grants, routing invariants, no-starvation properties, CDC-specific properties from M6 |
| **Coverage** | Requirement-linked functional coverage; reviewed structural coverage; **QoS-specific crosses** (class × contention level × memory pressure) |
| **Formal** | Small leaves first: queue conservation, one-hot grant, response routing, resource bounds. Later: bounded-latency properties under stated assumptions |

Use OpenTitan as a methodology reference, not as code to copy onto AXI. The reference model must be derived from the specification independently — if it shares an assumption with the RTL, a shared mistake passes silently. Validate the checker itself with deliberate fault injection at every milestone.

---

## 5. Milestones

Exit evidence is the gate. Calendar estimates are for planning only.

| ID | Milestone | Hours | Exit artifact |
| --- | --- | --- | --- |
| M0 | Scope, baseline, **tool gate** | 25–35 | Pinned build, capability probe passed, ownership contracts, decision log |
| M1 | Reusable UVM foundations | 40–60 | Independent checking on one project leaf; injected bugs detected |
| M2 | Single-channel DMA + **memory model** + **telemetry v1** | 60–85 | Byte-correct transfers under variable memory latency, with measurable behaviour |
| M3 | Outstanding work, ordering, recovery | 60–90 | ID tracking, error handling, stop/drain, completion and reset contract |
| M4 | Two channels, shared SRAM, **pluggable arbitration** | 60–95 | Contention handling, three arbiter implementations, QoS service model checking |
| M5 | Compute client as **traffic generator** | 40–60 | Numerical correctness plus realistic tiled access pattern loading the fabric |
| M6 | **Clock domain crossing** | 50–80 | Second domain for compute client, structural CDC clean, constrained synchronizers |
| M7 | Subsystem qualification | 60–90 | Requirements closure, coverage, assertions, selected formal, release candidate |
| M8 | Ibex and FPGA integration | 70–110 | Software workload, timing closure, sustained hardware runs |
| M9 | Research campaign | 100–180 | Reproducible baseline comparisons, measured tradeoffs, writeup |

**Dependency note:** M0→M8 is the first-release path. M6 (CDC) follows M5 so that the compute client is verified in a single domain before the crossing is introduced — one hard thing at a time, and it gives you a clean before/after comparison.

### Schedule

| Outcome | Active hours | At 20 h/week with 30% reserve |
| --- | --- | --- |
| Standalone verified subsystem (M0–M7) | 395–595 | ~26–39 weeks (**6–9 months**) |
| FPGA-qualified first release (M0–M8) | 465–705 | ~30–46 weeks (**8–11 months**) |
| Through research campaign (M0–M9) | 565–885 | ~37–58 weeks (**9–14 months**) |

Add three to six months beyond M9 for writeup, revision, feedback and iteration if you are targeting a publication. A realistic total for "verified subsystem plus a research result someone else takes seriously" is **18 to 24 months** at this pace.

The 30% reserve (higher than the 25% in v0.1) reflects that this is your first UVM environment. Recompute it after M1 and M3 using actual logged hours — those two reforecast points are mandatory, not optional.

---

## 6. Parallel tracks (run from week 1, not at the end)

### 6.1 Literature track — 2 to 4 hours per week

Do not defer this to M9. Reading shapes the design and tells you early whether your idea already exists.

**Start here:**
- Arm AMBA AXI and ACE protocol specification (the QoS signalling chapters specifically)
- Arm AMBA QoS regulator and network interconnect documentation
- Onur Mutlu's memory systems lecture series (freely available)
- Rixner et al., *Memory Access Scheduling* — the foundational paper, and its citation tree
- MICRO / ISCA / HPCA / DAC papers on memory scheduling, QoS and fairness in shared memory systems
- Network-on-chip QoS literature — the arbitration problems are structurally similar
- Real-time systems literature on worst-case latency analysis and network calculus (this is where the bounded-latency angle lives)

**Keep a running annotated bibliography** in `docs/literature/`. One paragraph per paper: what it claims, what it assumes, what it measures, and whether it forecloses or enables your direction. By M4 you should be able to state precisely what has already been done and where the gap is.

### 6.2 Visibility track — from month 1

- **Public repository from week 1.** Real README, real documentation, honest status.
- **Write up debugging war stories** as you go. A well-documented failure teaches more than a clean success, and it is what gets read.
- **Submit early and often**, before the work is finished: CARRV, DATE Work-in-Progress, ORConf, RISC-V Summit, FPGA-focused workshops. Rejection at this stage costs nothing and the feedback is valuable.
- **Post progress** where practitioners are. Consistency matters more than polish.

### 6.3 External review track — from M2 onward

Get your architecture document and testplan in front of a working DV or fabric engineer around M2. Informally is fine — a former colleague, someone from a conference, an open-source maintainer whose code you are using.

An hour of their review will redirect more effort than a month of solo work. It also starts the relationships that lead to the team you eventually need to join. Repeat at M4 and M7.

---

## 7. Tooling

### Required at M0 (this is a hard gate)

| Tool | Purpose |
| --- | --- |
| **Full Questa / equivalent commercial simulator** | SV/UVM, constrained randomization, SVA, DPI-C, coverage, debug. **Non-negotiable.** |
| SystemVerilog + UVM | Pick a simulator-supported UVM library and lock the version |
| Git | RTL, specs, scripts, dependency locks, decision log |
| Python + venv | Reference models, memory model calibration, telemetry analysis, regression orchestration |
| Make, Tcl, C/C++ | Build, simulator control, DPI-C, software tests |
| Bender | PULP dependency manifests and locks |
| Verible, Verilator | Formatting, lint, secondary smoke simulation |

**M0 capability probe — all six must pass:** run a small constrained-random UVM test; sample and report a covergroup; deliberately fail an SVA property and see it reported; call a DPI-C function; save and merge two coverage runs; confirm a batch regression reports pass/fail correctly. Record simulator version, UVM library version and enabled license features.

> **This is the project's single point of failure.** The plan assumes full Questa with coverage, SVA and DPI-C. EDA Playground with Riviera-PRO will not carry you past M1. The Altera FPGA/Starter edition lacks the required verification features. **Resolve simulator access before writing any RTL.** If you cannot, the entire timeline is fiction and you should restructure around what you actually have.

### Later

| When | Tool |
| --- | --- |
| M4–M7 | Formal: Questa Verify Property if licensed, otherwise SymbiYosys/Yosys for supported leaves. Prove small contracts first |
| M6 | Structural CDC analysis tool. Simulation assertions alone cannot establish metastability robustness |
| M8 | FuseSoC and RISC-V GCC for Ibex; FPGA vendor synthesis, P&R, STA, embedded logic analyzer |
| M9 | Python analysis and plotting; optional board power measurement |

**Spending order:** simulator access first, existing compute hardware second, FPGA board selected at M8 readiness, formal licenses or measurement equipment only for a defined gate.

**Workstation:** 16 GB RAM works for early exercises; 32 GB and an SSD is the useful target. No GPU needed. Use a vendor-supported Linux distribution — do not assume WSL is supported.

### On using AI tools

Use them aggressively for what they are good at, and spend the reclaimed hours on what they are not.

**Delegate:** UVM boilerplate (agents, drivers, monitors, sequence items), assertion drafting from spec text, directed test scaffolding, coverage exclusion drafts, documentation, regression triage, standard blocks like FIFOs and synchronizers.

**Do not delegate:** deciding *what* to verify, the risk model behind the testplan, spec ambiguity resolution, root-cause analysis of hard bugs, architectural tradeoffs, and the choice of what the research question should be.

The templated work is what gets automated first. The judgment work is the reason you are doing this project. Do not spend your scarce hours typing SystemVerilog that a tool can generate.

---

## 8. Detailed milestone content

### M0 — Scope, baseline and tool gate (25–35 h)

**Build:** Pass the capability probe. Choose one upstream iDMA configuration and dependency revision; reproduce its native simulation. Record reused versus owned modules. Freeze address map, alignment, burst limits, resource bounds, ownership rules, reset scope, command acceptance and error semantics. Identify the controlling AXI specification revision and supported subset. Draft the QoS class definitions.

**Exit evidence:** Clean-checkout build reproducing a known transfer. Capability probe log. Dependency lock file. Architecture and interface draft. Requirements with stable IDs. First testplan. Open-issue list. **Decision log** (see §11).

### M1 — Reusable UVM foundations (40–60 h)

**Build:** Learn UVM on a leaf that stays in the project — a command queue with a CSR wrapper. Transaction classes, sequencer, driver, passive monitor, agent configuration, scoreboard, test. Register model. Exercise backpressure, reset, simultaneous push/pop. Make the checker consume observed handshakes and independently computed expected results.

**Exit evidence:** Directed and constrained-random tests pass. Boundary cases covered. **Injected bugs are detected** — data corruption, missing events, duplicate events — each with a reproducible failing seed and a waveform that explains it. UVM_ERROR and UVM_FATAL treated as regression failures.

**Reforecast point.** Log actual hours against the 40–60 estimate and adjust everything downstream.

### M2 — Single-channel DMA, memory model, telemetry (60–85 h)

**Build:** Integrate the chosen backend. Implement command frontend, burst segmentation, buffer and control wrappers. Support 4 KiB splitting and maximum burst length. Start with one outstanding operation. **Build the parameterizable memory model (§2.1).** **Build telemetry v1 (§2.3)** — latency histogram, basic stall counters, trace dump. Specify command rejection rules and reserve terminal-result capacity before accepting work.

**Exit evidence:** Byte-correct transfers at minimum and maximum lengths, page and burst boundaries, under random delays and backpressure. Destination data exact, surrounding memory untouched. Injected faults (wrong address, lost beat, premature completion) caught by independent checks. **Latency histogram produced and sane under three different memory latency profiles.**

### M3 — Outstanding work, ordering and recovery (60–90 h)

**Build:** Bounded credits and IDs. Read-response association, write address/data ordering. Read and write error handling, completion status, interrupt and W1C races, graceful STOP, coordinated reset.

**Recovery semantics to specify explicitly:** STOP halts new issuance and drains accepted transactions before reporting a stopped or partial result. A failed write response leaves destination data uncertain — no rollback is promised; mark the range invalid until software repairs it. A stalled endpoint may prevent drain completion; a watchdog can report the fault but cannot make accepted AXI traffic disappear. Reset invalidates in-flight work through a visible reset epoch and is an explicit exception to exactly-once completion.

**Exit evidence:** Legal interleaving and reordering across IDs, required ordering within an ID. Credits exhausted. Responses delayed. Errors injected. Stop at issue, data and response boundaries. No premature success, no orphaned accepted transaction, no duplicate terminal event.

**Reforecast point.** Second mandatory recalibration.

### M4 — Two channels, shared SRAM, pluggable arbitration (60–95 h)

**Build:** Channel scheduling, scratchpad banks, response routing, contention counters. **Implement the arbiter interface of §2.2 and all three baseline policies.** Per-class bandwidth regulators. Build the **QoS service model** in the testbench that independently predicts which requester should win. Extend the scoreboard for channel/descriptor/ID context and memory visibility.

**Exit evidence:** Simultaneous submission, same-bank and cross-bank access, queue and credit saturation, stalled clients, error plus contention, interrupt races. Routing and data preserved. **Arbiter behaviour matches the independent service model for all three policies.** Starvation-freedom assertions pass. Telemetry distinguishes stall causes correctly. All M2–M3 regressions still pass.

**External review point.**

### M5 — Compute client as traffic generator (40–60 h)

**Build:** Signed INT8 dot product, 1–256 elements, signed 32-bit result — but issued as a **tiled, strided access pattern** modelling GEMM tile consumption: fetch a tile, compute, stall if the next tile is not ready, repeat. Specify lane packing, addresses, legal lengths, accumulator initialization, result visibility, ownership transitions. Use the scratchpad. DMA moves data while compute works on other owned buffers.

**Why this shape:** the arithmetic is trivial and that is deliberate. What matters is that the fabric now faces a bursty, latency-sensitive consumer that stalls when starved — which is exactly the traffic pattern QoS exists to serve.

**Exit evidence:** Numerical agreement with an independent integer model across zero, extrema, alternating signs and random operands. DMA/compute overlap on disjoint buffers. Illegal launch rejected. Reset and delayed memory handled. **Compute stall cycles attributable to data starvation are measured and reported** — this becomes a primary research metric.

### M6 — Clock domain crossing (50–80 h)

**Build:** Move the compute client into a second clock domain. Asynchronous FIFOs with proper gray-code pointers. Reset and flush coordination across domains. Clock-stop behaviour. Physical synchronizer constraints. Give CDC its own requirement and verification matrix.

**Exit evidence:** Structural CDC analysis clean, with every crossing classified and justified. Reset and clock-ratio stress passed across a range of frequency relationships. All pre-CDC functional regressions still pass. Synchronizer constraints written and reviewed. Metastability injection where the simulator supports it.

### M7 — Subsystem qualification (60–90 h)

**Build:** Close the requirements-to-test-to-checker-to-coverage map. Review functional and structural coverage gaps, assertions, lint. Apply formal to tractable leaves: queue conservation, one-hot grants, response routing, resource bounds. Check assertion antecedents to avoid vacuous passes. Package simulation, models, scripts, limits and known issues.

**Exit evidence:** All planned tests pass or carry reviewed, justified exclusions. No unexplained failures or critical lint. Every major invariant has an effective checker. Injected faults caught. Formal results state their assumptions and bounds. A clean checkout reproduces the release candidate.

**External review point.** Also the natural point for a first workshop submission.

### M8 — Ibex and FPGA integration (70–110 h)

**Build:** Integrate an unchanged Ibex configuration with a protocol and width adapter, program and data memory, timer, UART. Uncached shared memory with explicit ownership. Preserve upstream FuseSoC and Bender flows behind thin project wrappers. C driver tests. Clock and reset constraints. Expose telemetry counters as readable CSRs.

**Exit evidence:** Software submits jobs, handles errors, verifies results. Timing closed at a declared target with constrained clocks and reviewed exceptions. Known-answer and sustained mixed-workload runs repeated. Logs, versions, resource reports and board configuration archived.

**Select the board after an early synthesis estimate.** Check BRAM for scratchpad plus CPU memory, LUT and DSP headroom, clocking, JTAG, UART. Do not buy hardware to begin UVM work. Procurement and license lead times are calendar dependencies outside the active-hour estimates.

**Scope of the claim:** one parameter set, one tool profile, one target, tested conditions. ASIC adoption would additionally require technology-specific physical implementation, DFT, power and reset analysis, and product qualification. Reusing a proven CPU does not transfer its verification evidence to your glue logic.

### M9 — Research campaign (100–180 h)

**Primary direction — formally bounded latency for QoS classes.**

Most work in this area establishes QoS behaviour by simulation and reports percentiles. Proving a *bounded worst-case latency* for a priority class under explicitly stated service assumptions is harder, genuinely useful, and sits exactly where design meets verification — which is your specific advantage. Safety-critical automotive and aerospace silicon needs this and largely does not have it. Smaller field, fewer strong competitors.

**Method:** Profile the baseline first. State the service assumptions precisely (downstream memory guarantees, maximum burst, credit bounds, arbitration policy). Derive a latency bound analytically. Encode it as a formal property on the arbiter and prove it under those assumptions. Then measure how often the real system meets, approaches or violates the bound when assumptions are relaxed. Compare unchanged baseline, credible tuned alternatives, and your design under identical workloads and implementation conditions.

**Secondary direction — open reproducible verification infrastructure for accelerator memory subsystems.** OpenTitan did this for security IP and became the reference everyone cites. Nothing equivalent exists for data movement. Less glamorous, higher adoption potential, and it makes you known.

**Other candidate experiments:** bank mapping or buffering to reduce scratchpad conflicts for a specific access pattern; transfer scheduling matched to the compute client's consumption pattern; fault isolation and diagnosis with bounded overhead and tested recovery.

**Exit evidence:** Reproducible data and scripts for throughput, latency distribution, stall attribution, resources, frequency and end-to-end task time. Adverse cases and negative results included. The improvement explained along with its cost and its failure modes. **Correctness regressions run on every candidate before any performance number is collected.**

> A measured tail-latency improvement is not a worst-case bound. A bound requires explicit service assumptions and appropriate proof. Energy claims require credible measurement or a qualified estimation flow; FPGA resource counts alone are insufficient.

---

## 9. What verification closure means here

**Requirements and checking.** Every supported feature has a requirement ID, a behaviour statement, a test or proof, a checker, and an evidence reference. Data, routing, accepted-transaction counts, ordering, completion, reset and QoS semantics are independently checked. Protocol compliance reviewed against the selected AXI specification revision.

**Coverage.** Track requirement-linked coverpoints and crosses: channel count × burst boundary; outstanding depth × response delay; bank conflict × client mix; error type × job state; stop/reset × outstanding work; **priority class × contention level × memory pressure**; **clock ratio × crossing direction × reset phase**. Review branch, FSM, toggle and assertion coverage. Explain unreachable bins. A percentage or seed count alone is not the gate.

**Oracle validation.** Known-answer tests and deliberate faults validate the checker path itself. Successful loopback alone proves nothing. Inject at least one fault per milestone and confirm detection.

**Release evidence package.**

| Artifact | Minimum content |
| --- | --- |
| Specifications | Architecture, ports, registers, legal parameters, ownership, errors, stop/reset, QoS policy, performance envelope |
| Verification | DV architecture, testplan, models, assertions, coverage report and exclusions, regression seeds, issue closure |
| Implementation | RTL and dependency tags, tools, license and attribution inventory, timing and resource reports, constraints and reviewed exceptions |
| Integration | Driver examples, build and run instructions, hardware logs, known limitations, reproducible benchmark procedure |
| Research | Baseline data, comparison methodology, analysis scripts, negative results |

Maintain a short dated gate-review note at each milestone stating what evidence was reviewed and what exclusions remain. A green upstream badge, successful synthesis, or a large randomized run count does not establish closure for your changes.

---

## 10. First four weeks

At 20 hours per week the first four weeks target M0 and M1 and readiness to start M2. Do not expect a working DMA in this window.

| Week | Main work | Concrete result |
| --- | --- | --- |
| 1 | **Confirm simulator access and pass the capability probe.** Select upstream revision, reproduce smoke simulation. Begin literature track. Create public repo. | Tool manifest, probe log, dependency lock, baseline tag, repo live |
| 2 | Map data and control flow. Write scope, reset and ownership rules. Draft requirement and test IDs. Define QoS classes. | Architecture sketch, module ownership list, reviewed first-leaf contract, decision log started |
| 3 | Build UVM agent, monitor and independent scoreboard around the queue/CSR leaf. | Directed and random tests with reproducible pass/fail |
| 4 | Add assertions, functional coverage, reset and backpressure cases. Inject bugs. Review hours spent. | M1 evidence, bug journal, **revised M2 estimate**, first DMA testplan |

**Weekly rhythm:** approximately 3 h specification and review, 6 h RTL and integration, 7 h verification and debug, 2 h analysis and documentation, 2 h literature. Shift toward DV during closure milestones.

**Regression policy:** small deterministic smoke suite on every change; deeper seed and configuration sweeps on the licensed machine. Preserve failing seeds and minimal reproducers. Keep large wave databases out of Git history.

---

## 11. Repository structure

```
docs/
  architecture/      Block diagrams, interface specs, register maps
  requirements/      Requirement IDs, traceability matrix
  testplan/          Per-milestone testplans, coverage plans
  decisions/         Decision log (see below)
  literature/        Annotated bibliography, reading notes
  milestones/        Gate-review notes, release notes
rtl/                 Owned RTL
third_party/         Pinned upstream code and manifests, clearly separated
dv/
  agents/            AXI, AXI-Lite, interrupt, clock/reset
  env/               Environment, config, virtual sequencer
  models/            Reference models, memory model, QoS service model
  tests/             Directed and random tests
  assertions/        SVA modules and bind files
  formal/            Formal harnesses and property sets
  coverage/          Covergroups, exclusion files with justifications
sw/                  C drivers, workloads
fpga/                Target wrappers, constraints, build scripts
scripts/             Build, regression, report, analysis
analysis/            Python telemetry analysis, plotting, benchmark harness
results/             Evidence summaries, artifact pointers
```

### Decision log — record at M0 and maintain

Simulator edition and actual enabled features; UVM library version; upstream commit hashes; owned module list; AXI specification revision; supported parameter matrix; address map and SRAM bank mapping; QoS class definitions and their meanings; accepted-command and result-capacity rules; stop, error and reset semantics; buffer ownership contract; weekly available hours. FPGA selection can stay open until M8 preparation.

Every subsequent significant decision gets a dated entry: what was decided, what alternatives were considered, why. This document is what makes the project defensible to a reviewer eighteen months from now — including you.

---

## 12. Risks

| Risk | Response |
| --- | --- |
| **Simulator or license gap** | Resolve at M0 through actual feature probes. This gates everything. Hosted tools are for exercises only |
| Memory model too idealized to create real QoS problems | Build variable-latency modelling at M2, calibrate against published DRAM behaviour, include a hostile mode |
| Arbitration frozen too early, research becomes a rebuild | Define the swappable arbiter interface at M4 with three baseline implementations before any research candidate |
| Missing observability discovered at M9 | Telemetry at M2, extended at every milestone. Never defer instrumentation |
| CDC deferred and then skipped | It is M6, on the core path, with its own gate. Not optional |
| Reference model shares RTL mistakes | Derive behaviour independently from spec. Observe handshakes. Independent known-answer vectors. Fault injection every milestone |
| Research idea already published | Literature track from week 1. Know the gap by M4, not M9 |
| Ambiguous memory races | Explicit non-coherent ownership, disjoint write regions, visibility defined before concurrency testing |
| Stop/reset corrupts transactions | Drain on STOP, coordinate reset, invalidate partial output per the contract |
| Performance tuning breaks correctness | Full correctness regression on every candidate before collecting any performance data |
| Board or DDR integration dominates | Board selected after synthesis estimate. FPGA RAM first. DDR deferred |
| Feature expansion prevents release | M0–M8 scope frozen. Coherence, 2D, scatter/gather, CPU changes behind separate milestones |
| Isolation — working alone too long | External review at M2, M4, M7. Submit to a workshop before the work is finished. **If still solo at two years, something has gone wrong** |
| Discouragement at month six | The padded schedule exists for this. Milestone gates give visible progress. The bug journal shows how far you have come |

---

## 13. Progress signals

**On track if:**
- You are finding bugs your own checkers missed, and understanding why they missed them
- You can explain a design tradeoff without reaching for the spec
- Someone you do not know has used, cited or commented on your work
- Your testplan is growing at least as fast as your test count
- The literature notes let you state precisely where your gap is

**Drifting if:**
- You are mostly making tests pass rather than deciding what should be tested
- The repository is growing but the requirements matrix is not
- You have not spoken to anyone doing this professionally
- Telemetry exists but you have never analysed the output
- Milestone estimates keep being met exactly — it means you are padding, or not attempting anything hard enough to surprise you

---

## 14. Honest framing of what this achieves

This project makes you **credible**. That is the necessary first step and the one most people never take. It gets you into a team where harder problems are available.

It does not, by itself, make you a leader in the field. No single project does. Solo work has a ceiling that effort does not raise: you will not see how a large team coordinates, how review culture catches what individuals miss, what happens when a spec ambiguity surfaces two weeks before tapeout, or what a silicon bug costs. That gap closes only inside a real organization.

Rough clock for the larger ambition: two years to credible contributor, five to eight to a recognized name in a subfield, ten or more to what you described. The mechanism is a loop — build, measure, write up, get into a stronger room, repeat. Not any single output of it.

The design-plus-verification combination is the right thing to be building. Most architects cannot verify at depth; most DV engineers cannot architect. People who do both well end up owning subsystem definition, which is where the interesting decisions get made. Protect that as your differentiator.

---

## 15. References

Pin actual revisions and confirm target configurations at M0. Upstream documents and tool capabilities change.

**Upstream RTL**
- PULP iDMA — architecture, prerequisites, native simulation: https://github.com/pulp-platform/iDMA
- PULP AXI — SystemVerilog communication components: https://github.com/pulp-platform/axi
- Ibex — requirements and integration: https://ibex-core.readthedocs.io/en/latest/02_user/system_requirements.html

**Methodology**
- OpenTitan design verification methodology: https://opentitan.org/book/doc/contributing/dv/methodology/index.html

**Tools**
- Siemens Questa One Sim: https://www.siemens.com/en-us/products/ic/questa-one/simulation/questa-one-sim/
- EDA Playground FAQ (hosted limits): https://eda-playground.readthedocs.io/en/latest/faq.html
- Verilator language support: https://verilator.org/guide/latest/languages.html

**Domain reading** — start the literature track with Arm AMBA AXI/ACE specifications and QoS regulator documentation, Onur Mutlu's memory systems lectures, Rixner et al. on memory access scheduling and its citation tree, MICRO/ISCA/HPCA memory scheduling and fairness papers, NoC QoS literature, and real-time systems work on worst-case latency analysis and network calculus.

---

*Plan revision 0.2. Estimates are engineering judgments, not commitments. Reforecast at M1 and M3.*
