# can2wifi-pcb

Custom PCB for the [can2wifi](https://github.com/avtotor/can2wifi) firmware — a
**CAN → WiFi bridge** on ESP32-S3 that plugs into a vehicle OBD-II port, reads the
CAN bus via TWAI and streams frames over WiFi (Telnet :23, GVRET / SavvyCAN) and
UART. Targeted at **mass production via JLCPCB** (full PCBA).

> ⚠️ **Status: engineering draft — NOT yet fab-ready.** The schematic is complete
> and ERC-clean, footprints are assigned, components are placed and ground pours
> added. **Copper routing is NOT done** and several values need datasheet
> verification. See [Before you order](#before-you-order). Open the project in
> KiCad and finish/review it before sending anywhere.

## Design decisions

| Topic | Choice |
|---|---|
| Power | **12 V only** from OBD-II (pin 16) → DC-DC buck → 5 V → LDO → 3.3 V. USB is data-only. |
| Manufacturing | Full PCBA at JLCPCB (Gerber + BOM + CPL, LCSC parts). |
| Vehicle connector | **OBD-II J1962 male**, board-mounted, THT (hand-solder). |
| Protection | Extended: reverse-polarity, 12 V load-dump TVS, CAN common-mode choke + TVS, USB ESD. |
| CAN transceiver | **TJA1051T/3** (3.3 V VIO) instead of TJA1050 — removes the 1k/2k divider and the marginal 5 V TXD threshold. Firmware-compatible. |
| USB | CH340C USB-UART on UART0 (GPIO43/44) + dual-transistor auto-program. Native USB unavailable (GPIO20 used by CAN). |
| Board | 70 × 52 mm, 2-layer FR4 1.6 mm, GND pours both sides. |

## Architecture

```
OBD-II 12V ─[F1 PTC]─[TVS SMBJ24A]─[D2 SS56 revpol]─ +12V
                                                       │
                                          [U4 LM5164 buck 100V] → +5V
                                                       │            │
                                                  [U5 AMS1117] → +3V3
                                                       │
  USB-C ─[D4 USBLC6 ESD]─ D+/D- ─[U3 CH340C]─ UART0 ─ [U1 ESP32-S3]
                                   DTR/RTS ─[Q1/Q2]─ EN / IO0
                                                       │ GPIO21/20
                                          [U2 TJA1051]─[L2 CMC]─[D3 TVS]─ CANH/CANL → OBD 6/14
```

## Signal map (from firmware)

| Net | ESP32-S3 pin | Notes |
|---|---|---|
| CAN_TX | IO21 | → TJA1051 TXD |
| CAN_RX | IO20 (module pin 14 "USB_D+") | ← TJA1051 RXD |
| LED_STAT | IO2 | active-high status LED (D7) |
| ESP_TX | TXD0 / IO43 | → CH340 RXD |
| ESP_RX | RXD0 / IO44 | ← CH340 TXD |
| EN | EN | RST button + RC + Q1 |
| IO0 | IO0 | BOOT button + Q2 |

> Native USB (GPIO19/20) is sacrificed: GPIO20 is the CAN RX pin, so programming
> goes through the CH340C on UART0. **On the bench, apply 12 V while flashing**
> (USB does not power the board). Optional DNP diode **D5** (VBUS→+5V) can be
> populated to power the board from USB for bench work.

## BOM (51 placements)

Assembly BOM with LCSC part numbers: [doc/can2wifi-pcb-bom-jlcpcb.csv](doc/can2wifi-pcb-bom-jlcpcb.csv)
(JLCPCB columns: Comment / Designator / Footprint / LCSC Part #). Plain BOM:
[doc/can2wifi-pcb-bom.csv](doc/can2wifi-pcb-bom.csv).

**44 of 51 placements have a verified in-stock LCSC number.** Still TBD (pick
before ordering): **D6/D7** LEDs (green/blue 0603), **F1** PPTC fuse (0.5 A 1812),
**R4** 31.6 k (set by buck feedback — verify), **R8** 120 Ω (DNP, not fitted),
**SW1/SW2** tactile buttons (match the PTS645 footprint), **J1** OBD-II connector
(hand-soldered, not JLCPCB-assembled).

| Ref | Value | Footprint |
|---|---|---|
| U1 | ESP32-S3-WROOM-1-N16R8 | RF_Module:ESP32-S3-WROOM-1 |
| U2 | TJA1051T/3 | SOIC-8 |
| U3 | CH340C | SOIC-16 |
| U4 | LM5164DDA (100 V sync buck) | SOIC-8-1EP ThermalVias |
| U5 | AMS1117-3.3 | SOT-223 |
| Q1, Q2 | MMBT3904 | SOT-23 |
| D1 | SMBJ24A (load-dump TVS) | SMB |
| D2 | SS34 (reverse-polarity, C115205) | SMA |
| D3 | PESD2CAN (CAN TVS) | SOT-23 |
| D4 | USBLC6-2SC6 (USB ESD) | SOT-23-6 |
| D5 | SS34 — **DNP** (USB→5V bench) | SMA |
| D6/D7 | Power / Status LED | 0603 |
| L1 | 33 µH (buck) | 12×12 mm |
| L2 | CAN common-mode choke | Coilcraft 1812CAN |
| F1 | PTC 0.5 A | 1812 |
| J1 | OBD-II J1962 male | can2wifi:OBD2_J1962_Male (custom) |
| J2 | USB-C 2.0 | HRO TYPE-C-31-M-12 |
| R1/R3 100k, R2 13k, R4 31.6k | LM5164 UVLO/RON/FB | 0603 |
| R5–R7 10k | EN/IO0 pull-ups | 0603 |
| R8 120Ω — **DNP** | CAN termination (bus already terminated) | 0805 |
| R9/R10 5.1k | USB-C CC | 0603 |
| R11/R12 1k | LED series | 0603 |
| R13 0Ω | CH340 V3↔3V3 link | 0603 |
| SW1/SW2 | RST / BOOT | PTS645 |
| C1–C17 | decoupling / bulk / buck | 0603/0805/1210 |

## Before you order

This board **must be finished and reviewed in KiCad** first. Checklist:

1. **Route the board.** No copper traces exist yet. Either install Java + the
   Freerouting JAR and run the MCP `autoroute`, or route manually in Pcbnew.
   Gerbers/CPL are only meaningful after routing.
2. **Clean up placement.** Auto-placement left courtyard overlaps (esp. around U1
   — the WROOM-1 footprint reserves a large keep-out/antenna area). Re-arrange so
   the ESP32 antenna sits over a board edge with copper keep-out beneath it.
3. **Verify the LM5164 buck values** against the TI datasheet / WEBENCH:
   `R3 (RON)` switching frequency, `L1` inductor, `R4/R5` feedback divider for
   5.0 V, `C4` feed-forward, input/output caps. These are reference-style
   placeholders, not silicon-verified.
4. **Verify the J1962 footprint** (`can2wifi.pretty/OBD2_J1962_Male`) against the
   exact OBD-II male connector you buy — pin pitch, row spacing and mounting holes
   are a generic 2×8 layout and will differ per part.
5. **Finish LCSC assignment** — 44/51 done; assign the 7 TBD lines (LEDs, fuse,
   buttons, R4 31.6k) and regenerate the BOM + CPL for assembly.
6. **Confirm the USB-C MPN** matches the `HRO TYPE-C-31-M-12` footprint.
7. Run **DRC** until clean, refill zones, then export Gerbers, drill, BOM and CPL.

## Files

| Path | Contents |
|---|---|
| `can2wifi-pcb.kicad_sch` | Schematic (ERC-clean: 0 errors) |
| `can2wifi-pcb.kicad_pcb` | Board: outline, placement, GND pours (unrouted) |
| `can2wifi.pretty/` | Custom J1962 footprint |
| `doc/can2wifi-pcb-schematic.pdf` | Schematic PDF |
| `doc/can2wifi-pcb-bom.csv` | BOM (grouped by value) |

## License

Proprietary — see [LICENSE](LICENSE).
