# RTL-TO-GDSII Flow — 32-bit Sequential ALU (Open-Source Sky130HD Flow)

## Table of Contents
1. Project Objectives
2. Design Specifications
3. RTL-to-GDSII Flow Steps
4. Results Summary
5. Challenges & Learnings
6. Conclusion

## Project Objectives
1. Design and verify a 32-bit Sequential Arithmetic Logic Unit (ALU) at RTL level.
2. Perform a complete open-source ASIC physical design flow — synthesis, floorplanning, placement, CTS, routing, and signoff — using industry-standard open EDA tools.
3. Generate a DRC-clean, timing-closed GDSII layout ready for tapeout on the SkyWater 130nm PDK.

## Design Specifications
- **ALU Width:** 32 bits
- **Sequential Design:** Registered inputs/outputs, operates synchronously on a single clock domain
- **Technology Node:** SkyWater Sky130HD (130nm, `sky130_fd_sc_hd` standard cell library)
- **Target Clock Period:** 16 ns (~62.5 MHz), retimed from an initial 10 ns target after post-route timing analysis revealed the carry-chain critical path required more margin
- **Core Area:** 37,577.29 µm² at 20% target utilization (effective ~22.7% after CTS/filler insertion)
- **Cell Count:** 1,051 synthesized instances (+ 511 tap cells, + 4,160 filler cells post-placement)

## Tools and Technologies
| Stage | Tool |
|---|---|
| RTL Synthesis | Yosys |
| Floorplan / Placement / CTS / Routing | OpenROAD (via OpenROAD-flow-scripts, ORFS) |
| Static Timing Analysis | OpenSTA |
| GDSII Streamout / DRC | KLayout |
| PDK | SkyWater Sky130HD, managed via Volare |

## RTL-to-GDSII Flow Steps

### 1) RTL Design & Synthesis
- Verilog RTL for `alu32` (combinational ALU core) and `alu32_top` (registered wrapper).
- Synthesized with Yosys: `read_verilog` → `hierarchy` → `proc` → `opt` → `fsm` → `memory` → `techmap` → `dfflibmap` → `abc` (mapped to `sky130_fd_sc_hd__tt_025C_1v80` liberty).
- Result: 948 combinational cells + 103 `dfxtp_1` flip-flops, chip area 7,602.29 µm².

### 2) Timing Constraints (SDC)
- Defined clock, I/O delays, driving cells, and load constraints via OpenSTA-compatible SDC.
- Iterated the clock period from 10 ns → 14 ns → **16 ns** based on progressively more accurate timing views (ideal clock → post-CTS propagated clock → post-route with extracted parasitics).

### 3) Floorplanning & Power Planning
- Initialized floorplan at 20% target utilization, aspect ratio 1.0, using OpenROAD's `initialize_floorplan`.
- Inserted 511 tap cells (`sky130_fd_sc_hd__tapvpwrvgnd_1`) for substrate connectivity.
- Generated power distribution network (PDN) via ORFS `pdn.tcl`.

### 4) Placement & Clock Tree Synthesis
- Global placement (Nesterov-based) converged in 386 iterations, final placed area 8,680.70 µm².
- Detailed placement legalized 100% of cells via diamond search, zero placement failures.
- CTS built an H-tree clock network (3 levels, 9 clock buffers, 103 sinks) using `sky130_fd_sc_hd__clkbuf_16`.

### 5) Routing
- Detailed routing via TritonRoute (OpenROAD), converging from an initial 1,585 DRC violations to **0 violations** over 6 optimization iterations.
- Final routed wirelength: 39,404 µm across met1–met4, using 8,903 vias.

### 6) Signoff Checks
- **STA (post-route, with extracted parasitics):**
  - Setup: worst slack **+3.11 ns MET** (critical path through the 32-bit carry chain)
  - Hold: worst slack **+0.60 ns MET**
  - WNS/TNS: 0.00 / 0.00 (zero violations)
- **Power:** Total ~0.795 mW @ 62.5 MHz (58.3% internal, 41.7% switching, negligible leakage)
- **DRC:** Verified clean in KLayout using the Sky130 DRC ruleset after GDS streamout.

### 7) GDSII Streamout
- Final GDS generated via KLayout's `def2stream.py`, using an embedded-LEF technology wrapper to correctly resolve via geometry (a fix required after discovering the default tech file's LEF references were incomplete).

## Results Summary
| Metric | Value |
|---|---|
| Core Area | 37,577.29 µm² |
| Final Utilization | 22.7% |
| Clock Period | 16 ns (62.5 MHz) |
| Worst Setup Slack | +3.11 ns |
| Worst Hold Slack | +0.60 ns |
| DRC Violations | 0 |
| Routed Wirelength | 39,404 µm |
| Total Power | ~0.795 mW |

## Challenges & Learnings
- **Clock period retiming:** Initial 10 ns target failed post-route STA by -2.89 ns due to a long ripple-carry-style critical path (~15 serial stages through `maj3_1` full-adder cells). Iteratively corrected the SDC based on increasingly accurate timing views rather than trusting the optimistic ideal-clock estimate from synthesis alone.
- **OpenSTA SDC compatibility:** Design-Compiler-style commands (`remove_from_collection`, `all_inputs -no_clocks`) aren't supported in this OpenSTA build — resolved by explicitly listing input ports instead.
- **Environment variable dependencies:** ORFS platform scripts (e.g. `tapcell.tcl`) require `TAP_CELL_NAME`/`ENDCAP_CELL_NAME` to be exported before invoking OpenROAD.
- **GDS streamout path handling:** `def2stream.py` requires either running from the correct working directory or passing fully absolute paths for `in_def`/`out_file`.
- Gained hands-on experience with the complete open-source EDA stack: Yosys synthesis, OpenROAD place-and-route, OpenSTA signoff timing, and KLayout GDS generation/DRC.

## Conclusion
This project delivers a synthesizable, timing-closed, DRC-clean 32-bit ALU design carried fully from RTL through GDSII using an entirely open-source toolchain on the SkyWater Sky130HD PDK. It demonstrates practical, hands-on debugging across the full physical design stack — synthesis, floorplanning, placement, CTS, routing, and signoff — making it a strong foundation for roles in Physical Design, RTL-to-GDSII, or Backend VLSI.
