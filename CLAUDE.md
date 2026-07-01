# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A **hardware/KiCad project**, not software. It is the PCB for the [can2wifi](https://github.com/avtotor/can2wifi)
firmware — a CAN→WiFi OBD-II bridge on ESP32-S3. There is no build/lint/test in the
software sense; "the build" is *route the board → DRC-clean → export fab files*.
Targeted at full PCBA at JLCPCB.

Source of truth is the KiCad data, edited via the **`mcp__kicad__*` MCP server** (swig
backend), with Freerouting driven over Bash. The three files that matter:

| File | Role |
|---|---|
| `can2wifi-pcb.kicad_sch` | Schematic (ERC-clean) |
| `can2wifi-pcb.kicad_pcb` | Board: outline, placement, GND pours, routing |
| `can2wifi-pcb.kicad_pro` | Project / design rules / net classes |
| `can2wifi.pretty/OBD2_J1962_Male.kicad_mod` | Only custom footprint (registered via `fp-lib-table`) |

`README.md` describes the *original design intent* (it still says 70×52 mm and
"NOT routed"). The board on disk is **ahead of the README**: re-placed, enlarged to
~98×101 mm, autorouted (2-layer, ~382 tracks / 28 vias) and DRC = 0. Trust the
`.kicad_pcb` and the project memory, not the README, for current layout state.

## Common workflow

Always start by opening the project, then verify — the MCP backend's in-memory state
and the disk can diverge (see Gotchas).

- Open: `mcp__kicad__open_project` with the `.kicad_pro` path.
- Checks: `mcp__kicad__run_erc`, `mcp__kicad__run_drc`, `mcp__kicad__refill_zones`,
  `mcp__kicad__check_courtyard_overlaps`, `mcp__kicad__query_zones`.
- Save: `mcp__kicad__save_project` (explicit save is more reliable than auto-save).
- Fab export: `mcp__kicad__export_gerbers`, `mcp__kicad__export_drill`,
  `mcp__kicad__export_bom`, `mcp__kicad__export_pos` (CPL). Exclude DNP parts (**D5**, **R8**).

### Routing (the one Bash step)

The MCP `autoroute` call times out (~30 s cap); Freerouting needs longer. Run it manually:

```bash
# 1. export_dsn (MCP) → board.dsn
java -jar ~/.kicad-mcp/freerouting.jar -de board.dsn -do board.ses -mp 30   # run in background
# 2. import_ses (MCP) ← board.ses
```

## Architecture (the board)

12 V-only supply from OBD-II pin 16 → DC-DC buck (LM5164) → 5 V → LDO (AMS1117) →
3.3 V. USB-C is **data-only** (CH340C USB-UART on UART0, GPIO43/44, with Q1/Q2
auto-program). Native USB is sacrificed because GPIO20 is the CAN RX pin. CAN via
TJA1051T/3 (3.3 V VIO) → common-mode choke → TVS → OBD pins 6/14. Extended protection
chain (PTC, reverse-polarity, load-dump TVS, ESD). U1 is fixed at (35,16); its antenna
keep-out band is baked into the WROOM-1 footprint (board coords x[11,59], y[-11.75,9.25])
and must stay copper-free — the top board edge is at y=-13 so the keep-out sits on-board.

## Gotchas (MCP swig backend — verify after every reopen)

These cost real debugging time; honor them:

- `move_component`, zone geometry / zone `connect_pads` mode, and Edge.Cuts edits made
  directly in `.kicad_pcb` **persist** across reopen+save.
- `set_board_size` and `set_design_rules` are **in-memory only** — a later `open_project`
  reverts them. Set rules, then `save_project`, and do **not** reopen.
- Editing design rules directly in `.kicad_pro` does **not** take: `open_project` reads
  stale values and a later MCP save overwrites the file. So `min_resolved_spokes` and
  project rules can't be changed by hand-editing `.kicad_pro`.
- Both GND zones use **solid** pad connection (`connect_pads yes`) to avoid
  `starved_thermal` DRC errors (the backend can't set `min_resolved_spokes=1`).
- `refill_zones`/auto-save sometimes refuse with "disk changed externally" (mtime race
  after render/import_ses): `open_project` to resync, re-apply, then save.
- `check_courtyard_overlaps` treats U1's whole 48×43 bbox as courtyard and false-flags
  neighbours in x[11,59]; real KiCad DRC only blocks the module body + keep-out band.
- `get_board_2d_view` and reading the PDFs fail on this machine (no pdftoppm). Validate
  via `run_drc` / `check_courtyard_overlaps` / `query_zones`, not visuals. The PDF in
  `doc/` is plotted from a temp copy shifted (dx99.5, dy67.5) to center on A4; the real
  board keeps its own coords near (0,0)/negative-Y.

## Conventions

- `fab/` is git-ignored (regenerated output); `*.kicad_prl`, `*-backups/`, `.history/`
  too. Commit only the editable source (`.kicad_sch`, `.kicad_pcb`, `.kicad_pro`, the
  `.pretty` footprint, docs).
- Out of scope unless asked: verifying LM5164 buck values (R3/R4/L1…), validating the
  custom J1962 footprint against the real connector, the 7 missing LCSC numbers
  (LEDs/fuse/buttons/R4), and JLCPCB rotation conventions in the CPL.
- License is **proprietary** (`LICENSE`) — do not add open-source headers.
