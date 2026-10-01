# PicoRV32 RTL-to-GDSII on SkyWater 130nm (OpenLane)

Is project mein maine ek real **RISC-V CPU (PicoRV32)** ko open-source tools ki madad se **RTL (Verilog code) se GDSII (final chip layout)** tak le gaya hoon, **SkyWater Sky130** process mein.

Sirf flow chalana goal nahi tha. **5 runs** mein maine **reports padhi, violations dhoondhe, unka reason samjha, fix try kiya**, aur har change ka effect data se measure kiya. Ek jagah meri apni galti (ek config setting jo silently ignore ho rahi thi) bhi pakdi aur OpenLane ke scripts padh ke confirm ki.

> **Note:** PicoRV32 ka RTL maine nahi likha. Ye [YosysHQ/picorv32](https://github.com/YosysHQ/picorv32) se liya gaya hai (original license ke liye unka repo dekho). Mera kaam **physical design** hai: floorplan, placement, CTS, routing, signoff aur violations ka analysis.

---

## ⚡ Short mein: kya mila

| Run | Approach | Antennas (pin+net) | Diodes | Fanout viol. | Slew viol. | Timing |
|---|---|---|---|---|---|---|
| run1 | Default settings | 104 | 208 | 54 | 0 | ✅ Met |
| run2 | Heuristic diodes (90 µm threshold) | **14** | 8,848 | 834 | 3 | ✅ Met |
| run3 | Sirf CTS fix | 90 | 194 | **25** | **0** | ✅ Met |
| run4 | Heuristic diodes (200 µm threshold) | 26 | 5,305 | 505 | 2 | ✅ Met |
| run5 | run3 + outer antenna repair loop | 90 | 194 | 25 | 0 | ✅ Met |

Saare runs mein: **DRC = 0, LVS = 0, setup/hold violations = 0**, die area **0.30 mm²**.

**Main findings:**
1. ✅ `CTS_SINK_CLUSTERING_SIZE: 10` se clock buffer fanout violations **19 → 0** (har run mein confirm).
2. ⚖️ Heuristic diode insertion mein **antenna aur fanout ek-dusre ke ulte chalte hain**. Threshold badalne se sirf trade-off curve par jagah badalti hai, trade-off khatam nahi hota.
3. 🔍 Router ki antenna repair **estimated (global) routes** par chalti hai aur 0 violations dikhati hai, lekin **detailed routing** ke baad asli wires mein 46 pins phir bhi violate karte hain. Isliye config tuning se ye fix nahi hua.
4. 🐛 Ek setting (`GRT_ANTENNA_ITERS`) jo maine use ki, wo **OpenLane mein exist hi nahi karti**. Silently ignore ho rahi thi. Scripts mein `grep` karke pakdi.

---

## 📌 Physical Design kya hota hai? (simple mein)

- **RTL Design (front-end):** Verilog mein likhna ki chip *kya karegi*.
- **Physical Design (back-end):** Us Verilog ko asli **layout** mein convert karna, yaani transistors aur metal wires kahan rakhe jayenge, taaki chip factory mein ban sake.

Ek example se samjho: architect ghar ka **plan** banata hai (RTL designer), aur construction engineer us plan se **asli ghar** banata hai (Physical Design engineer). Ye project construction wala kaam hai.

---

## 🛠️ Tools aur Setup

| Cheez | Kya use kiya |
|---|---|
| OS | Windows + **WSL2 (Ubuntu 24.04)** |
| Container | **Docker Desktop** (WSL integration ke saath) |
| Flow | **OpenLane v1.0.2** (OpenROAD, Yosys, Magic, KLayout, Netgen andar hi hain) |
| PDK | **SkyWater Sky130A** (`sky130_fd_sc_hd` standard cell library) |
| Layout viewer | **KLayout** |

### Setup steps (short mein)
```bash
# 1. Basic tools
sudo apt update
sudo apt install -y build-essential python3-venv python3-pip make curl

# 2. OpenLane install (Docker image + Sky130 PDK download hota hai)
git clone https://github.com/The-OpenROAD-Project/OpenLane
cd OpenLane
make
make test        # "Basic test passed" aana chahiye
```

**Setup mein aaye problems aur unke fix:**
- `sudo` password galat → `apt update` chala hi nahi → purani package list se **404 errors**. Fix: sahi password ke saath `apt update` dobara chalaya.
- **"Could not get lock"**: Ubuntu background mein updates install kar raha tha. Fix: wait kiya (lock files delete nahi karni chahiye).
- **`git clone` fail (connection reset)**: slow/unstable internet ki wajah se. Fix: dobara try kiya (`--depth 1` bhi option hai).
- **KLayout window freeze**: Fix: `docker stop <id>` aur `wsl --shutdown`. Seekh: KLayout ko `&` ke saath background mein kholo.

---

## 📂 Design Setup

PicoRV32 OpenLane ke andar example ke roop mein nahi tha, isliye design folder khud banaya:

```bash
git clone --depth 1 https://github.com/YosysHQ/picorv32.git /tmp/picorv32
mkdir -p designs/picorv32/src
cp /tmp/picorv32/picorv32.v designs/picorv32/src/
```

### Base config (`config.json`)
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

| Setting | Matlab |
|---|---|
| `CLOCK_PERIOD: 25` | 25 ns clock = **40 MHz** (relaxed target, pehle run ke liye) |
| `FP_CORE_UTIL: 35` | Core area ka 35% cells se bhara, baaki routing ke liye jagah |
| `PL_TARGET_DENSITY: 0.45` | Placement kitna tight pack karega |

### Flow chalana
```bash
make mount                                  # OpenLane container mein jao
./flow.tcl -design picorv32 -tag run1       # pura RTL-to-GDSII flow
```

Har run ki exact settings `configs/runN_config.tcl` mein hain (OpenLane khud save karta hai).

---

## 🔄 Flow ke steps (kya-kya hota hai)

1. **Synthesis (Yosys):** Verilog → gate-level netlist (Sky130 cells)
2. **Floorplan:** Chip ka size decide karna, IO pins rakhna, power grid (PDN) banana
3. **Placement:** Saare standard cells ko rows mein rakhna
4. **CTS (Clock Tree Synthesis):** Clock ko saare flip-flops tak barabar delay se pahunchana
5. **Routing:** Metal wires se cells ko jodna
6. **Signoff:** DRC, LVS, Antenna check, STA (timing), aur final **GDSII**

---

## 📊 Run 1: Default settings

**Result: Flow complete ✅**

| Metric | Value | Matlab |
|---|---|---|
| WNS / TNS | 0 / 0 | Timing meet hui (40 MHz par) ✅ |
| DRC violations | 0 | Layout manufacturing rules follow karta hai ✅ |
| LVS errors | 0 | Layout aur netlist match ✅ |
| Die area | 0.30 mm² (~535 × 533 µm) | Chip ka size |
| Synthesized cells | 9,406 | Logic cells |
| Total cells | 34,905 | Decap, filler, welltap cells milake |
| Wire length | 566,247 µm | ~0.57 m metal wiring! |
| Runtime | ~40 min | Laptop par |

### Lekin problems bhi mile ⚠️

**1. Antenna violations: 54 pin + 50 net**
Manufacturing ke time lambi metal wires charge collect kar leti hain, jo transistor ke gate ko damage kar sakta hai. Real tapeout mein ye fix karna zaroori hai.

**2. Max fanout violations: 54** (limit = 10)
Kuch signals 10 se zyada inputs ko drive kar rahe the. Report padhne par 3 types mile:
- **Clock tree buffers** (`clkbuf_leaf_*`): **19 violations**. CTS apne clustering setting (`CTS_SINK_CLUSTERING_SIZE`, default 25) se kaam karta hai, fanout limit 10 se nahi.
- **Input buffer** (`input30`): fanout 18
- **Normal logic gates / flip-flops**: fanout 11–17

Slew aur capacitance violations **0** the, yaani electrically signals theek the. Fanout ek design-rule guideline hai.

**3. 106 unconstrained endpoints: ye harmless hain ✅**
`eoi`, `trace_data`, `pcpi_*`, `mem_addr[0:1]` jaise outputs par timing check nahi tha. Reason: PicoRV32 mein **PCPI, trace aur IRQ features default mein disabled** hain, isliye ye outputs constant (0/1) hain, jo **tie cells** drive karte hain. Aur `mem_addr[1:0]` hamesha 0 hota hai (word-aligned access). Constant signal ka koi timing path nahi hota.

> ❌ Jo **nahi** kiya: `MAX_FANOUT_CONSTRAINT` ko 20 karke violations "gayab" kar dena. Usse problem sirf chhupti hai, fix nahi hoti.

---

## 📊 Run 2: Heuristic diode insertion + CTS fix

Config mein add kiya:
```json
"RUN_HEURISTIC_DIODE_INSERTION": true,
"GRT_ANTENNA_ITERS": 10,
"CTS_SINK_CLUSTERING_SIZE": 10
```
> ⚠️ Baad mein pata chala ki `GRT_ANTENNA_ITERS` OpenLane mein exist hi nahi karta, ye ignore ho gaya (Run 5 dekho). Is run ka asli effect sirf heuristic diodes aur CTS setting ka hai.

### Run 1 vs Run 2

| Metric | Run 1 | Run 2 | Effect |
|---|---|---|---|
| Pin antenna violations | 54 | **7** | ✅ −87% |
| Net antenna violations | 50 | **7** | ✅ −86% |
| Clock buffer fanout violations | 19 | **0** | ✅ Fully fixed |
| Diode cells | 208 | **8,848** | ⚠️ 42× zyada |
| Total cells | 34,905 | 47,309 | +36% |
| Wire length (µm) | 566,247 | 589,965 | +4% |
| Vias | 84,216 | 100,828 | +20% |
| **Max fanout violations** | 54 | **834** | ❌ Bahut badh gaye |
| **Max slew violations** | 0 | **3** | ❌ Naye aa gaye |
| WNS / TNS | 0 / 0 | 0 / 0 | Timing abhi bhi meet ✅ |
| DRC / LVS | 0 / 0 | 0 / 0 | Clean ✅ |

### 🔍 Analysis: fanout 54 se 834 kyun hua?

- Fanout report mein bahut saare nets ka fanout **exactly 20** tha. Ye random number nahi hai.
- **Heuristic diode insertion lagbhag har input pin ke paas ek diode laga deta hai.** STA har diode ko ek extra **load** ki tarah count karta hai.
- To jo net pehle ~10 loads drive kar raha tha, ab 10 gates + 10 diodes = **20** drive kar raha hai. Fanout **double**.
- Slew violation bhi isi se aaya: report mein ek hi net (`_08009_/Y` → `_08010_/A2`) tha, aur us pin par `ANTENNA__08010__A2/DIODE` laga hua tha. Extra diode load se slew 0.80 ns ho gaya (limit 0.75 ns). Teeno "violations" asal mein **ek hi net** ke hain.

**Conclusion:** Heuristic diode insertion ek "hathoda" approach hai. Antenna fix ho gaye, lekin signal quality (fanout/slew) kharab ho gayi. Ek problem fix karne se dusri aa gayi.

**CTS fix kaam kar gaya:** `CTS_SINK_CLUSTERING_SIZE: 10` se CTS ne max 10 flip-flops ke clusters banaye, isliye clock buffer fanout violations **19 → 0**.

---

## 📊 Run 3: Heuristic band, sirf CTS fix

**Plan:** heuristic diodes band karke router ki targeted repair try karna (`GRT_ANTENNA_ITERS: 10`), aur CTS fix rakhna.

**Expectation:** antennas run2 jaisa kam, fanout ~35, slew 0.

| Metric | Run 1 | Run 3 |
|---|---|---|
| Antennas (pin / net) | 54 / 50 | 46 / 44 ❌ |
| Diode cells | 208 | 194 |
| Max fanout violations | 54 | **25** ✅ |
| Max slew violations | 0 | **0** ✅ |
| Clock buffer fanout violations | 19 | **0** ✅ |

**Result:** Signal quality sabse acchi (fanout 25, slew 0, expectation se bhi behtar), lekin **antennas lagbhag same** rahe. Run 5 mein pata chala kyun: `GRT_ANTENNA_ITERS` ignore ho gaya tha, to ye run asal mein "run1 + CTS fix" hi tha.

---

## 📊 Run 4: Beech ka raasta (threshold 200 µm)

Heuristic diodes wapas on, lekin sirf **lambi nets** par. Pehle confirm kiya ki setting exist karti hai:
```bash
grep -rn "HEURISTIC_ANTENNA_THRESHOLD" configuration/
# configuration/routing.tcl:21:set ::env(HEURISTIC_ANTENNA_THRESHOLD) 90
```
Default **90 µm** hai, yaani 90 µm se lambi har net par diode. Isliye run2 mein 8,848 diodes lage the. Run4 mein ise **200** kiya.

| Metric | Run 2 (90 µm) | Run 4 (200 µm) |
|---|---|---|
| Antennas (pin + net) | 14 | 26 |
| Diode cells | 8,848 | **5,305** (−40%) |
| Max fanout violations | 834 | 505 |
| Max slew violations | 3 | 2 |

**Result:** Run4 trade-off curve par **beech mein** aaya. Threshold badalne se diodes kam hue, lekin antennas badhe. **Koi bhi threshold dono (antenna aur fanout) ko ek saath kam nahi kar sakta.**

---

## 📊 Run 5: Galti pakdi aur sahi setting test ki 🐛

### Galti kaise mili
Saari antenna-related settings dekhi:
```bash
grep -rn -i "antenna\|diode" configuration/
```
List mein `GRT_ANTENNA_ITERS` **tha hi nahi**. Asli setting `GRT_MAX_DIODE_INS_ITERS` thi (default 1).

Run ke resolved config (`config.tcl`) mein dono dikh rahe the, lekin wo file har key record karti hai, chahe use ho ya nahi. Isliye **scripts mein check kiya**:
```bash
grep -rn "GRT_ANTENNA_ITERS\|GRT_MAX_DIODE_INS_ITERS" scripts/
# Sirf GRT_MAX_DIODE_INS_ITERS use hota hai:
# while {$iter <= $::env(GRT_MAX_DIODE_INS_ITERS) && $minimum_antennae > 0} { ...
```
**Confirm:** `GRT_ANTENNA_ITERS` silently ignore ho raha tha.

### Run 5 = Run 3 + `GRT_MAX_DIODE_INS_ITERS: 10`

**Result: Run 3 se bilkul identical**, har metric same (46/44 antennas, 194 diodes, same wire length, same vias). Flow deterministic hai, aur extra iterations kabhi chale hi nahi.

### 🔍 Root cause (log se confirm)

Global routing log mein:
```
[INFO GRT-0006] Repairing antennas, iteration 12.
[INFO GRT-0012] Found 1 antenna violations.
[INFO GRT-0015] Inserted 1 diodes.
[INFO GRT-0006] Repairing antennas, iteration 13.
[INFO GRT-0012] Found 0 antenna violations.
```
- OpenROAD ki antenna repair ka **apna andar ka loop** hai (13 iterations chale) aur wo **0 violations** par khatam hua.
- OpenLane ka `GRT_MAX_DIODE_INS_ITERS` ek **bahar ka loop** hai, jo tabhi repeat hota hai jab antennas bache hon. Andar wala 0 par pahunch gaya, to bahar wala repeat hi nahi hua.
- **Lekin signoff mein 46 pins violate kar rahe the.** Kyunki repair **global routing ke estimated wires** par hoti hai. **Detailed routing** baad mein asli wires banata hai jinka shape alag hota hai, aur kuch nets wahan violate karne lagti hain.

**Isliye sirf heuristic (routing se *pehle* diodes) hi asli violations kam kar paya, aur wo fanout ki keemat par.**

### Run 3/5 ke bache hue 46 antenna pins ka analysis

| Ratio (actual / limit) | Pins |
|---|---|
| > 2.0× | 4 |
| 1.5–2.0× | 8 |
| 1.0–1.5× | 34 |

- **~3/4 pins limit se 50% se kam upar** hain.
- Layers: **met1: 30**, met2: 11, met3: 5
- **11 violations output buffers par** (`output127/A`, `output211/A`, …). Output buffers chip ke edge par hote hain, to core logic se unki wire **lambi** hoti hai, aur sabse bade ratios yahi hain.

---

## 🏁 Best run kaunsa?

Koi bhi run poori tarah clean nahi hai. Ye ek trade-off hai:

- **Tapeout ke liye: Run 2.** Antenna violations manufacturing risk hain (transistor damage), isliye ye sabse important hain, aur run2 mein sabse kam hain (14). Uske 834 fanout violations zyadatar diodes ki wajah se "inflated" hain; electrical checks lagbhag theek hain (cap = 0, slew sirf 1 net par 0.05 ns upar).
- **Signal integrity ke liye: Run 3 / Run 5.** Fanout 25, slew 0, lekin 90 antennas bache.

**Real industry mein agla kadam:** bache hue violations **manual ECO (Engineering Change Order)** se fix hote hain: sirf un specific pins par diode lagana, ya un nets ko upar ke metal layers par reroute karna.

---

## 🖼️ Layout Screenshots

| Full chip | Zoomed in |
|---|---|
| ![Full chip](images/picorv32_full_chip.png) | ![Zoomed](images/picorv32_zoomed.png) |

**Zoomed view mein kya dikh raha hai:**
- **Standard cell rows**, jaise `sky130_fd_sc_hd__dfxtp_1` (D flip-flop), `sky130_fd_sc_hd__clkbuf_16` (clock buffer), `sky130_ef_sc_hd__decap_12` (decap cell)
- **Metal wires**: alag layers alternate direction mein chalti hain (horizontal / vertical), taaki ek-dusre ko cross kar sakein bina short hue
- **Vias**: chhote squares jahan do metal layers judti hain
- **Power straps**: moti lines jo poore chip mein VPWR/VGND le jaati hain

---

## 🧠 Is project se kya seekha

1. **Flow chalana easy hai, results samajhna asli skill hai.** "Flow complete" ka matlab "design perfect" nahi hota.
2. **Har warning ka reason dhoondo.** Kuch harmless hote hain (unconstrained endpoints), kuch serious (antenna).
3. **Ek fix dusri problem la sakta hai.** Heuristic diodes ne antenna fix kiye, par fanout 15× badha diya.
4. **Ek run mein ek hi cheez badlo**, taaki pata chale kis change ka kya effect hua.
5. **Constraint relax karke violations chhupana fix nahi hai.**
6. **Har setting ko verify karo ki wo sach mein lagi.** Tool unknown config keys ko hamesha reject nahi karta. Maine ek galat setting 3 runs tak use ki, jab tak scripts padh ke confirm nahi kiya.
7. **Expectation galat ho sakti hai, aur wo bhi useful data hai.** Run 3 aur Run 5 ne meri hypothesis galat sabit ki, aur usi se asli root cause mila.
8. **Kab rukna hai, ye bhi skill hai.** Config tuning ki limit mil gayi; aage ka fix ECO hai, aur tuning nahi.

---

## 🚀 Aage kya karna hai

- [ ] `FP_CORE_UTIL` experiments (35% vs 50% vs 65%): area, congestion, wire length par effect
- [ ] Clock period kam karte jaana (25 → 15 → 10 → 8 ns) jab tak timing fail na ho, phir **critical path** analyse karna
- [ ] Python script jo saare runs ka `metrics.csv` padh ke comparison table automatically banaye
- [ ] Bache hue antenna pins par manual ECO try karna
- [ ] Bache hue logic-gate fanout violations analyse karna

---

## 📁 Repo Structure

```
├── README.md
├── configs/
│   ├── run1_config.tcl ... run5_config.tcl   # har run ki exact settings
├── reports/
│   ├── run1_metrics.csv ... run5_metrics.csv # area, timing, violations
│   └── run1_sta_checks.rpt ... run5_sta_checks.rpt  # slew/fanout/cap reports
└── images/
    ├── picorv32_full_chip.png
    └── picorv32_zoomed.png
```

---

## 🙏 Credits

- [PicoRV32](https://github.com/YosysHQ/picorv32): YosysHQ
- [OpenLane](https://github.com/The-OpenROAD-Project/OpenLane): The OpenROAD Project / Efabless
- [SkyWater Sky130 PDK](https://github.com/google/skywater-pdk): Google + SkyWater
