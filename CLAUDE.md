# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

This is a **KiCad 9 hardware design project**, not software: a circular carrier board ("RPi5 Circular Design", by Dussat Global Technologies) for the Raspberry Pi Compute Module 5 (CM5). There is no source code, build system, or test suite — the deliverables are the schematic and PCB layout files, and correctness is checked via KiCad's Electrical Rules Check (ERC) and Design Rules Check (DRC), not unit tests.

Key files:
- `Circular_RPi5.kicad_pro` — project file: DRC rules, net classes, teardrop/via/track defaults.
- `Circular_RPi5.kicad_sch` — schematic (component connectivity).
- `Circular_RPi5.kicad_pcb` — PCB layout (copper, stackup, board outline).
- `Circular_RPi5.kicad_prl` — local/UI project settings (layer visibility, view state); not design-meaningful.
- `Module.pretty/` — local footprint library folder containing a generic collection of dev-board footprints (Arduino, Feather, Pico, etc.); most are unused leftovers from a bulk import, not specific to this design.
- `Circular_RPi5-backups/` — zipped snapshots of the project auto-saved by KiCad; binary, not meant for manual editing or diffing.
- `fp-info-cache`, `~Circular_RPi5.kicad_pcb.lck` — KiCad-generated transient/cache files, not meaningful design state.

## Working with the files

These are KiCad S-expression files, normally edited through the KiCad GUI (Eeschema for `.kicad_sch`, Pcbnew for `.kicad_pcb`). Avoid hand-editing them directly — they contain UUID cross-references between schematic symbols, PCB footprints, and net connectivity that are easy to break outside the GUI. If a scripted change is genuinely needed, prefer KiCad's Python API (`pcbnew` module) or `kicad-cli`, and always re-run DRC/ERC afterward to confirm nothing broke.

## Commands

KiCad 9 is installed at `C:\Program Files\KiCad\9.0`; the CLI is at:
```
"C:\Program Files\KiCad\9.0\bin\kicad-cli.exe"
```

Common checks (run from the repo root):
```
# Design Rule Check on the PCB (the closest thing this project has to a test suite)
"C:\Program Files\KiCad\9.0\bin\kicad-cli.exe" pcb drc Circular_RPi5.kicad_pcb --output drc-report.json --format json

# Electrical Rule Check on the schematic
"C:\Program Files\KiCad\9.0\bin\kicad-cli.exe" sch erc Circular_RPi5.kicad_sch --output erc-report.json --format json
```
Other `kicad-cli pcb export ...` / `kicad-cli sch export ...` subcommands generate gerbers, drill files, BOM, netlist, or PDF/SVG plots when needed.

Opening the project for interactive editing is done via the KiCad GUI on `Circular_RPi5.kicad_pro`.

## Board architecture

**Stackup**: 6 copper layers, 1.6 mm total thickness — `F.Cu` / `In1.Cu` (Ground) / `In2.Cu` / `In3.Cu` (Power) / `In4.Cu` (Ground) / `B.Cu`, all FR4 prepreg dielectric. The design was reduced from an original 8-layer stackup to 6 layers (see commit history) — keep this in mind if re-adding layers or reasoning about impedance/return paths, since only two dedicated plane layers (In1 ground, In4 ground) sandwich a shared power layer (In3).

**Net classes** (`Circular_RPi5.kicad_pro`): `Default` (0.127 mm track/clearance) and `Power` (same track width, applied to `GND` and `/+3.3V` nets by pattern match).

**Schematic subsystems** (by symbol library used):
- **CM5 module socket** — `CM5IO:ComputeModule5-CM5` (footprint `CM5IO:Raspberry-Pi-5-Compute-Module`). This is a **local/global library dependency not checked into this repo** — the `CM5IO` symbol/footprint library must be resolvable via the user's KiCad library tables (global `sym-lib-table`/`fp-lib-table` or another project's), since no `fp-lib-table`/`sym-lib-table`/`.kicad_sym` file exists here. If the project fails to resolve `CM5IO:*` parts, that's a missing library configuration issue outside this repo, not a bug in these files.
- **Ethernet** — UDE `RB1-125B8G1A` Gigabit RJ45 magjack (`Connector:RJ45_RB1-125B8G1A`), all four differential pairs wired. It replaced an Amphenol `RJMG1BD3B8K1ANR`, which was 10/100-only and physically had no contacts for pairs 2 and 3.
- **HDMI** — Micro-D HDMI connector (Molex 46765-0xxx footprint).
- **USB3 Type-A host port** (J1, `USB3_A_Molex_48393-001`) — powered/switched through a TI `TPS2069CDBV`, with `USBLC6-2SC6`/`USBLC6-4SC6` ESD protection, a 22 uF bulk cap (C9) on VBUS, and TVS/polyfuse protection. The USB2 D+/D- pair is on CM5 pins 134/136 so it shares the USB3-0 controller with the SuperSpeed lanes.
- **USB-C power input** — power-only receptacle (no data lines) for board power.
- **High-density interconnect** — a 22-pin FFC/FPC connector (Amphenol `F32Q` series) plus a 5-pin header, likely for a display/camera or expansion interface.
- **Port protection** — recurring pattern of `Device:D_TVS`, `Device:Polyfuse`, and TI `TPD4E05U06DQA`/`USBLC6-x` ESD arrays on externally-exposed connectors (USB, Ethernet, HDMI).

**Design rules live in two places.** `Circular_RPi5.kicad_pro` holds the netclasses (`Power_5A` 3.0 mm, `HS_100` 0.20/0.13 mm, `HS_90_USB3` 0.22/0.11 mm), but a netclass `track_width` is only the router's default -- DRC never checks it. The enforcing rules are in **`Circular_RPi5.kicad_dru`**, which also explains why each limit is what it is. The 5 A width rule is deliberately scoped to two named rule areas called `HC_5V`; +5V is distributed by a 4614 mm2 plane on In3.Cu, so most +5V surface copper is a milliamp branch and a net-wide width rule produces false alarms.

**Known outstanding risk (not a DRC error):** eight differential pairs -- USB3 TX/RX, HDMI TX0/TX1/TX2, MIPI D0/D1/CLK -- are routed as two separate traces rather than coupled pairs, giving roughly 147 ohm differential against a 90/100 ohm target. Return paths were verified solid (max 0.4 mm antipad-scale gaps over the reference GND planes). Ethernet is unaffected because those runs are electrically short. Fixing it requires interactive coupled re-routing in KiCad; it cannot be scripted, as `pcbnew` exposes no routing API.

The board outline (`Edge.Cuts`) is circular, per the project name — factor this into any placement/routing/keepout reasoning (e.g., component and via placement near the board edge relative to a circular boundary, not a rectangular one).
