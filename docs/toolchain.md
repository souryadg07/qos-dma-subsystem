# Toolchain — Final Decision

**Status:** Decided 06 September 2026. Supersedes §4, §5 and §7 tooling notes in `plan.md` v0.2.
**Principle:** Every tool below is free at the scale this project needs. Nothing blocks on a license negotiation. One paid escape valve exists (DSim Cloud) and is optional until M7.

---

## 1. The stack at a glance

| Role | Tool | Cost | Used in |
| --- | --- | --- | --- |
| **Primary simulator** | Altair DSim (Free Individual License) | Free | M1–M9 |
| **Secondary simulator** | AMD Vivado XSim | Free (Vivado BASIC) | M1–M9, portability cross-check |
| **Burst regressions** | DSim Cloud (usage-based) | Pay-per-minute | M7+, optional |
| **Formal** | SymbiYosys + Yosys | Free | M4–M7, leaf proofs |
| **Structural CDC** | Vivado `report_cdc` + documented manual review | Free | M6 |
| **Lint / style** | Verible + `verilator --lint-only` | Free | Continuous |
| **Waveform debug** | GTKWave / Surfer, Vivado waveform viewer | Free | Continuous |
| **Dependency mgmt** | Bender (PULP), FuseSoC (Ibex) | Free | M0, M8 |
| **Build / orchestration** | GNU Make, Python, Tcl | Free | Continuous |
| **Modelling & analysis** | Python 3.11+ (numpy, pandas, matplotlib) | Free | M2, M5, M9 |
| **DPI-C** | GCC / Clang | Free | M0 probe, M5 |
| **Software (M8)** | RISC-V GCC toolchain | Free | M8 |
| **FPGA implementation** | Vivado BASIC; 60-day EVAL held in reserve | Free | M8 |
| **Version control** | Git + GitHub | Free | Continuous |
| **Editor** | VS Code + DSim Studio extension | Free | Continuous |

**Total cost to reach M8: zero.**

---

## 2. Simulation

### Primary — Altair DSim

Free Individual License: individual use, one concurrent simulation at a time. Supports UVM, constraint solver, SystemVerilog assertions, code and functional coverage. Now a Siemens product (Siemens acquired Altair March 2025; Altair acquired Metrics/DSim June 2024) — the same vendor as Questa.

Install via **DSim Studio**, a VS Code extension. Requires an Altair One account. Windows and Linux supported; macOS is not.

> **DSim Studio is the editor front-end. DSim is the simulator.** Use Studio for interactive work and debug. Drive batch regressions from `make` on the command line — that is what M6 onward needs, and it keeps the flow portable.

### Secondary — Vivado XSim

Not a fallback. A **portability check**. Running the same testbench on two independent simulators proves the code is LRM-clean rather than leaning on one vendor's behaviour. This is a genuine credibility signal in the release package and costs almost nothing once the Makefile supports both.

Policy: every test must pass on DSim. The smoke suite must additionally pass on XSim. Deep randomized regressions run on DSim only.

### Escape valve — DSim Cloud

The free tier's single-concurrent-simulation limit is the one real constraint. It is fine through M5. It hurts at M7 when hundreds of seeds are needed overnight. DSim Cloud bills per minute with no on-prem license required — spin it up only for closure sweeps and the M9 campaign.

**Budget this as the project's only likely cash cost.** Estimate at M6 once regression runtimes are known.

---

## 3. Formal — SymbiYosys + Yosys

Scope: bounded proofs on small leaves only. Queue conservation, one-hot grant, response routing, resource bounds, and (M9) the arbiter latency-bound property.

**Known limitation:** the open frontend's SystemVerilog and SVA support is narrower than commercial tools. Write formal properties in a deliberately restricted subset, and keep formal harnesses separate from the simulation SVA so a frontend limitation never blocks simulation.

Declare in the release notes: *proofs are bounded, on selected leaves, with stated assumptions. This is not commercial formal signoff.*

---

## 4. CDC (M6)

No free commercial CDC signoff tool exists. The plan:

1. **Vivado `report_cdc`** on the synthesized design — structural classification of every crossing. Confirm coverage and report format at M0.
2. **Manual crossing inventory** in `docs/` — every crossing listed with its type, synchronizer, and justification.
3. **Explicit synchronizer constraints** (`set_max_delay -datapath_only`, false paths) written and reviewed.
4. **Simulation stress** across clock ratios and reset phases.

Declare in the release notes: *CDC verified by structural analysis, constraint review and simulation stress; not by a commercial CDC signoff flow.* Simulation assertions alone cannot establish metastability robustness — state this rather than implying more than was done.

---

## 5. Lint, style and static checks

- **Verible** — formatting (enforce one style, commit the config) and style lint.
- **`verilator --lint-only`** — catches a different class of issue than Verible; run both.
- Run both on every commit via a Makefile target. Add to CI once CI exists.

---

## 6. Debug

- DSim and XSim both emit standard waveform formats; view in **GTKWave** or **Surfer**.
- Vivado's own waveform viewer for XSim runs and all M8 hardware debug.
- **Do not commit waveforms.** Preserve the failing seed and a minimal reproducer — that is the reproducible artifact. `.gitignore` already enforces this.

---

## 7. Versions to pin at M0

Record every one of these in `docs/decisions/`. A version not written down is a version you will not reproduce.

| Item | Decision |
| --- | --- |
| DSim version | Pin exact build |
| UVM library version | DSim ships a recent UVM; Ibex DV assumes UVM 1.2. **Decide at M0 which the project targets** and do not casually upgrade borrowed environments |
| Vivado version | Pin; XSim behaviour varies across releases |
| Yosys / SymbiYosys | Pin commit |
| Verible / Verilator | Pin release |
| PULP iDMA + AXI | Pin commit hashes; commit `Bender.lock` |
| Ibex | Pin commit; keep the configuration unchanged |
| Python | Pin version and commit `requirements.txt` |
| RISC-V GCC | Pin release (M8) |

---

## 8. M0 capability probe — mapped to tools

All six must pass on **DSim** before M1 begins. Items 1–4 should also be attempted on **XSim** to establish the dual-simulator baseline early.

| # | Probe | Tool target |
| --- | --- | --- |
| 1 | Small constrained-random UVM test runs | DSim, XSim |
| 2 | Covergroup samples and reports | DSim (XSim via `xcrg`) |
| 3 | A deliberately failing SVA property is reported | DSim, XSim |
| 4 | DPI-C function call works | DSim, XSim |
| 5 | Two coverage runs save and merge | DSim |
| 6 | Batch regression reports pass/fail correctly | DSim, via `make` |

Record simulator versions, UVM library version, and any feature that did not work. **A probe item that fails is a scope decision, not a footnote** — resolve it before building on top of it.

---

## 9. What this toolchain does NOT provide

State these plainly in every release note. Overclaiming is the fastest way to lose a reviewer's trust.

- **No commercial formal signoff.** Bounded proofs on selected leaves only.
- **No commercial CDC signoff.** Structural analysis, constraints and simulation stress only.
- **No parallel regression on the free tier.** Seed counts are limited by wall-clock time unless DSim Cloud is used.
- **No power analysis.** Energy claims require measurement or a qualified estimation flow; FPGA resource counts are insufficient.
- **No ASIC flow.** No technology-specific physical implementation, DFT, or product qualification.
- **No commercial UVM VIP.** All agents are hand-written, which is the point.

---

## 10. Install order

1. VS Code (already installed)
2. Altair One account → DSim Studio extension → DSim
3. **Run the M0 capability probe on DSim** ← the real gate
4. Python 3.11+ and a virtual environment
5. Verible, Verilator, Yosys/SymbiYosys
6. GTKWave or Surfer
7. Vivado BASIC (large download; start it in the background during step 3)
8. Bender
9. Re-run probe items 1–4 on XSim; commit a dual-simulator Makefile
10. *(M8 only)* RISC-V GCC, FuseSoC, board vendor flow

Steps 1–3 are today's work. Everything else follows.

---

## 11. Open decisions

| Question | Resolve by |
| --- | --- |
| Which UVM library version does the project target? | M0 |
| Does Vivado `report_cdc` give sufficient structural coverage? | M0 (confirm), M6 (use) |
| Is a Linux workstation/VM needed, or is Windows sufficient throughout? | M0 |
| What does DSim Cloud cost for a realistic M7 sweep? | M6 |
| Which FPGA board? | M8 preparation |
| Any university affiliation available for a Questa/Enterprise seat? | Ongoing — pursue in parallel, but the project does not depend on it |

---

*Toolchain decision record. Update this file when any tool choice changes, and note the reason.*
