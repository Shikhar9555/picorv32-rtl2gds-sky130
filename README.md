# PicoRV32 RTL-to-GDSII on SkyWater 130nm (OpenLane)

Physical implementation of the **PicoRV32 RISC-V CPU** from RTL to GDSII using the open-source **OpenLane** flow and the **SkyWater Sky130** PDK.

The goal was not just to complete the flow, but to **analyze the signoff reports, find the root cause of each violation, attempt fixes, and measure their effect**. Across **5 runs**, this project investigates antenna, fanout and slew violations, documents a clear trade-off between them, and identifies why router-stage antenna repair could not fix the remaining violations. It also documents a config setting that was silently ignored, traced through OpenLane's source scripts.

> **Note:** The PicoRV32 RTL was not written by me. It comes from [YosysHQ/picorv32](https://github.com/YosysHQ/picorv32) (see that repository for its license). My work in this project is the **physical design**: floorplan, placement, CTS, routing, signoff, and analysis of the results.

---

## ⚡ Summary

| Run | Approach | Antenna violations (pin + net) | Diode cells | Max fanout violations | Max slew violations | Timing |
|---|---|---|---|---|---|---|
| run1 | Default settings | 104 | 208 | 54 | 0 | ✅ Met |
| run2 | Heuristic diode insertion (90 µm threshold) | **14** | 8,848 | 834 | 3 | ✅ Met |
| run3 | CTS clustering fix only | 90 | 194 | **25** | **0** | ✅ Met |
| run4 | Heuristic diode insertion (200 µm threshold) | 26 | 5,305 | 505 | 2 | ✅ Met |
| run5 | run3 + outer antenna repair loop | 90 | 194 | 25 | 0 | ✅ Met |

All runs: **DRC = 0, LVS = 0, no setup/hold violations**, die area **0.30 mm²** at **40 MHz**.

**Key findings:**
1. ✅ Setting `CTS_SINK_CLUSTERING_SIZE` to 10 eliminated all clock-buffer fanout violations (**19 → 0**), confirmed across every run that used it.
2. ⚖️ With heuristic diode insertion, **antenna and fanout violations move in opposite directions**. Changing the threshold only moves the result along the trade-off curve; it does not remove the trade-off.
3. 🔍 Router-stage antenna repair works on **estimated global routes** and reports 0 violations, but after **detailed routing** the real wires still violate on 46 pins. This is why config tuning could not fix them.
4. 🐛 A setting I used (`GRT_ANTENNA_ITERS`) **does not exist** in this OpenLane version and was silently ignored for three runs. I found this by tracing the variable through OpenLane's scripts.

---

## 🛠️ Tools

| Component | Used |
|---|---|
| OS | Windows + **WSL2 (Ubuntu 24.04)** |
| Container | **Docker Desktop** with WSL integration |
| Flow | **OpenLane v1.0.2** (OpenROAD, Yosys, Magic, KLayout, Netgen) |
| PDK | **SkyWater Sky130A**, `sky130_fd_sc_hd` standard cell library |
| Layout viewer | **KLayout** |

### Setup
```bash
sudo apt update
sudo apt install -y build-essential python3-venv python3-pip make curl

git clone https://github.com/The-OpenROAD-Project/OpenLane
cd OpenLane
make            # pulls the OpenLane Docker image and the Sky130 PDK
make test       # should end with "Basic test passed"
```

---

## 📂 Design Setup

PicoRV32 is not bundled with this OpenLane version, so I created the design directory manually:

```bash
git clone --depth 1 https://github.com/YosysHQ/picorv32.git /tmp/picorv32
mkdir -p designs/picorv32/src
cp /tmp/picorv32/picorv32.v designs/picorv32/src/
```

### Base configuration (`config.json`)
```json
{
  "DESIGN_NAME": "picorv32",
  "VERILOG_FILES": "dir::src/picorv32.v",
  "CLOCK_PORT": "clk",
  "CLOCK_PERIOD": 25,
  "FP_CORE_UTIL": 35,
  "PL_TARGET_DENSITY": 0.45
}
```

| Setting | Meaning |
|---|---|
| `CLOCK_PERIOD: 25` | 25 ns clock = **40 MHz**, a relaxed target for the baseline |
| `FP_CORE_UTIL: 35` | 35% of the core area filled with cells, leaving room for routing |
| `PL_TARGET_DENSITY: 0.45` | How tightly global placement packs the cells |

### Running the flow
```bash
make mount                                  # enter the OpenLane container
./flow.tcl -design picorv32 -tag run1       # full RTL-to-GDSII flow
```

The exact resolved settings for each run are in `configs/runN_config.tcl`.

### Flow stages
1. **Synthesis** (Yosys): RTL → gate-level netlist of Sky130 cells
2. **Floorplan**: die size, IO placement, power distribution network (PDN)
3. **Placement**: global and detailed placement of standard cells
4. **CTS**: clock tree synthesis
5. **Routing**: global and detailed routing
6. **Signoff**: parasitic extraction, multi-corner STA, DRC, LVS, antenna check, GDSII

---

## 📊 Run 1: Baseline (default settings)

| Metric | Value |
|---|---|
| WNS / TNS | 0 / 0 (timing met at 40 MHz) |
| DRC / LVS | 0 / 0 |
| Die area | 0.30 mm² (~535 × 533 µm) |
| Synthesized cells | 9,406 |
| Total cells (incl. decap, fill, welltap) | 34,905 |
| Total wire length | 566,247 µm |
| Runtime | ~40 min on a laptop |

### Issues found

**1. Antenna violations: 54 pin, 50 net.**
Long metal wires can accumulate charge during fabrication and damage the transistor gates they connect to. These must be fixed before tapeout.

**2. Max fanout violations: 54** (limit = 10). Three distinct causes:
- **Clock tree leaf buffers** (`clkbuf_leaf_*`): **19 violations**. CTS groups sinks using its own setting, `CTS_SINK_CLUSTERING_SIZE` (default 25), independently of the fanout limit.
- **An input buffer** (`input30`): fanout 18.
- **Regular logic gates and flip-flops**: fanout 11–17.

Max slew and max capacitance violations were both **0**, so all signals were electrically within limits. Max fanout is a design-rule guideline.

**3. 106 unconstrained endpoints (harmless).**
Outputs such as `eoi`, `trace_data`, `pcpi_*` and `mem_addr[1:0]` had no timing checks. PicoRV32's **PCPI, trace and IRQ features are disabled by default**, so these outputs are tied to constants by **tie cells**, and `mem_addr[1:0]` is always 0 because memory accesses are word-aligned. A constant output has no timing path to check.

> ❌ What I did **not** do: raise `MAX_FANOUT_CONSTRAINT` to make the violations disappear from the report. That hides the problem without fixing it.

---

## 📊 Run 2: Heuristic diode insertion + CTS fix

Added to the config:
```json
"RUN_HEURISTIC_DIODE_INSERTION": true,
"GRT_ANTENNA_ITERS": 10,
"CTS_SINK_CLUSTERING_SIZE": 10
```
> ⚠️ `GRT_ANTENNA_ITERS` turned out not to exist and was ignored (see Run 5). The effective changes in this run were heuristic diode insertion and CTS clustering.

| Metric | Run 1 | Run 2 | Change |
|---|---|---|---|
| Pin antenna violations | 54 | **7** | ✅ −87% |
| Net antenna violations | 50 | **7** | ✅ −86% |
| Clock-buffer fanout violations | 19 | **0** | ✅ Fixed |
| Diode cells | 208 | 8,848 | ⚠️ 42× |
| Total cells | 34,905 | 47,309 | +36% |
| Wire length (µm) | 566,247 | 589,965 | +4% |
| Vias | 84,216 | 100,828 | +20% |
| **Max fanout violations** | 54 | **834** | ❌ |
| **Max slew violations** | 0 | **3** | ❌ |
| WNS / TNS | 0 / 0 | 0 / 0 | ✅ |
| DRC / LVS | 0 / 0 | 0 / 0 | ✅ |

### Why did fanout violations go from 54 to 834?

- Many nets in the fanout report had a fanout of **exactly 20**.
- Heuristic insertion places a diode next to almost every sink pin on qualifying nets, and **STA counts every diode as a load**.
- A net that drove ~10 gate inputs now drove 10 gates + 10 diodes = **20**, so its fanout **doubled**.
- The slew violations share the same cause: all three entries belong to **one net** (`_08009_/Y` → `_08010_/A2`) with `ANTENNA__08010__A2/DIODE` on the sink pin. The extra diode load raised the slew to 0.80 ns against a 0.75 ns limit.

**Conclusion:** broad heuristic insertion fixed most antenna violations but degraded fanout and slew. Fixing one problem created another.

**CTS fix:** with `CTS_SINK_CLUSTERING_SIZE: 10`, CTS built clusters of at most 10 flip-flops, eliminating all clock-buffer fanout violations (**19 → 0**).

---

## 📊 Run 3: CTS fix without heuristic diodes

**Plan:** disable heuristic insertion, rely on targeted router-stage repair (`GRT_ANTENNA_ITERS: 10`), keep the CTS fix.
**Expectation:** antennas close to run 2, fanout around 35, slew 0.

| Metric | Run 1 | Run 3 |
|---|---|---|
| Antenna violations (pin / net) | 54 / 50 | 46 / 44 ❌ |
| Diode cells | 208 | 194 |
| Max fanout violations | 54 | **25** ✅ |
| Max slew violations | 0 | **0** ✅ |
| Clock-buffer fanout violations | 19 | **0** ✅ |

**Result:** the best signal integrity of all runs (better than expected), but antenna violations barely changed. Run 5 explains why: `GRT_ANTENNA_ITERS` was ignored, so this run was effectively **run 1 + the CTS fix**.

---

## 📊 Run 4: Middle ground (200 µm threshold)

Re-enabled heuristic insertion, but only for longer nets. Before running, I confirmed the setting exists:
```bash
grep -rn "HEURISTIC_ANTENNA_THRESHOLD" configuration/
# configuration/routing.tcl:21:set ::env(HEURISTIC_ANTENNA_THRESHOLD) 90
```
The default of **90 µm** explains run 2's 8,848 diodes. Run 4 raised it to **200 µm**.

| Metric | Run 2 (90 µm) | Run 4 (200 µm) |
|---|---|---|
| Antenna violations (pin + net) | 14 | 26 |
| Diode cells | 8,848 | **5,305** (−40%) |
| Max fanout violations | 834 | 505 |
| Max slew violations | 3 | 2 |

**Result:** run 4 landed in the middle of the trade-off curve. Fewer diodes meant fewer fanout violations but more antenna violations. **No threshold value reduces both at once.**

---

## 📊 Run 5: Finding the ignored setting 🐛

### Discovery
Listing every antenna-related setting:
```bash
grep -rn -i "antenna\|diode" configuration/
```
`GRT_ANTENNA_ITERS` was **not** in the list. The real setting is `GRT_MAX_DIODE_INS_ITERS` (default 1).

The run's resolved `config.tcl` contained both keys, but that file records every key from `config.json`, used or not. So I checked the scripts that actually run the flow:
```bash
grep -rn "GRT_ANTENNA_ITERS\|GRT_MAX_DIODE_INS_ITERS" scripts/
# Only GRT_MAX_DIODE_INS_ITERS is used:
# while {$iter <= $::env(GRT_MAX_DIODE_INS_ITERS) && $minimum_antennae > 0} { ...
```
**Confirmed:** `GRT_ANTENNA_ITERS` had been silently ignored in runs 2–4.

### Run 5 = Run 3 + `GRT_MAX_DIODE_INS_ITERS: 10`

**Result: identical to run 3 in every metric** (46/44 antennas, 194 diodes, same wire length and via count). The flow is deterministic, and the extra iterations never ran.

### 🔍 Root cause, confirmed from the global routing log
```
[INFO GRT-0006] Repairing antennas, iteration 12.
[INFO GRT-0012] Found 1 antenna violations.
[INFO GRT-0015] Inserted 1 diodes.
[INFO GRT-0006] Repairing antennas, iteration 13.
[INFO GRT-0012] Found 0 antenna violations.
```
- OpenROAD's antenna repair has its **own internal loop** (13 iterations here), which ended with **0 violations**.
- OpenLane's `GRT_MAX_DIODE_INS_ITERS` is an **outer loop** that only repeats while violations remain. Since the inner loop reached 0, the outer loop never repeated.
- **Signoff still found 46 violating pins**, because repair runs on **estimated global routes**. **Detailed routing** later produces the real wire shapes, which differ from the estimate, and some nets end up violating.

**This is why only heuristic insertion, which places diodes *before* routing, reduced the real violations, at the cost of fanout.**

### Remaining 46 antenna pins (runs 3 and 5)

| Ratio (actual / limit) | Pins |
|---|---|
| > 2.0× | 4 |
| 1.5–2.0× | 8 |
| 1.0–1.5× | 34 |

- About **3/4 of the pins are less than 50% over** the limit.
- By layer: **met1: 30**, met2: 11, met3: 5.
- **11 violations are on output buffers** (`output127/A`, `output211/A`, …). Output buffers sit at the die edge, so the wires from the core logic to them are long. These have the largest ratios.

---

## 🏁 Best run

No run is fully signoff-clean; the results show a trade-off:

- **For tapeout: Run 2.** Antenna violations are a manufacturing risk and the most serious issue here, and run 2 has the fewest (14). Its fanout count is inflated by diode loads; the electrical checks are nearly clean (0 capacitance violations, and slew only 0.05 ns over the limit on a single net).
- **For signal integrity: Run 3 / Run 5.** Fanout 25 and slew 0, but 90 antenna violations remain.

**Industry next step:** the remaining violations would be fixed with a manual **ECO (Engineering Change Order)**: inserting diodes only at the specific violating pins, or rerouting those nets on higher metal layers.

---

## 🖼️ Layout

| Full chip | Zoomed in |
|---|---|
| ![Full chip](images/picorv32_full_chip.png) | ![Zoomed](images/picorv32_zoomed.png) |

The zoomed view shows:
- **Standard cell rows**, e.g. `sky130_fd_sc_hd__dfxtp_1` (D flip-flop), `sky130_fd_sc_hd__clkbuf_16` (clock buffer), `sky130_ef_sc_hd__decap_12` (decap cell)
- **Signal routing**: adjacent metal layers run in alternating directions so wires can cross without shorting
- **Vias** connecting metal layers
- **Power straps** distributing VPWR/VGND across the die

---

## 🧠 Lessons Learned

1. **"Flow complete" does not mean "design clean".** The real work is reading and understanding the reports.
2. **Find the reason behind every warning.** Some are harmless (unconstrained endpoints), some are critical (antenna).
3. **A fix can create a new problem.** Heuristic diodes fixed antennas but increased fanout violations 15×.
4. **Change one variable per run** so each effect can be attributed.
5. **Relaxing a constraint to hide violations is not a fix.**
6. **Verify that every setting actually takes effect.** The tool does not always reject unknown config keys; I used an invalid setting for three runs before tracing it through the scripts.
7. **A wrong hypothesis is still useful data.** Runs 3 and 5 disproved my expectation, and that led to the real root cause.
8. **Know when to stop tuning.** Config tuning reached its limit; the remaining fix is an ECO.

---

## 🚀 Future Work

- [ ] Core utilization sweep (35% / 50% / 65%): effect on area, congestion and wire length
- [ ] Clock period sweep (25 → 15 → 10 → 8 ns) until timing fails, then analyze the **critical path**
- [ ] Python script to generate the cross-run comparison table from `metrics.csv` automatically
- [ ] Manual ECO for the remaining antenna pins
- [ ] Analyze the remaining logic-net fanout violations

---

## 📁 Repository Structure

```
├── README.md
├── configs/
│   └── run1_config.tcl ... run5_config.tcl          # exact settings of each run
├── reports/
│   ├── run1_metrics.csv ... run5_metrics.csv        # area, timing, violation metrics
│   └── run1_sta_checks.rpt ... run5_sta_checks.rpt  # max slew / fanout / cap reports
└── images/
    ├── picorv32_full_chip.png
    └── picorv32_zoomed.png
```

---

## 🙏 Credits

- [PicoRV32](https://github.com/YosysHQ/picorv32): YosysHQ
- [OpenLane](https://github.com/The-OpenROAD-Project/OpenLane): The OpenROAD Project / Efabless
- [SkyWater Sky130 PDK](https://github.com/google/skywater-pdk): Google and SkyWater
