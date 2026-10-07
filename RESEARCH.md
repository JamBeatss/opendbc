# Audi A7 C8 / MLB evo - research notes

This replaces an earlier, longer version of this file. That version restated public opendbc material and contained two errors, both corrected by Dennis in the comma Discord: the Macan (95B) is MLB "Cajun", not MLB evo, and the pre-evo MLB DBC does not carry over to MLB evo. This version keeps only what is specific to MLB evo and points at primary sources.

## The problem

MLB evo cars (for example the C8 A6/A7, D5 A8, Q8, e-tron, 2018+ Cayenne, Urus) carry steering and driver-assistance traffic on FlexRay. openpilot and its panda safety code speak CAN. Upstream opendbc supports pre-evo MLB (`VolkswagenMLBPlatformConfig`, Porsche Macan Mk1, safety mode `volkswagenMlb`) but nothing on MLB evo.

## What exists today (checked 2026-10-06)

| Piece | Where | State |
|---|---|---|
| FlexRay MITM hardware | [dynm/pico-flexray](https://github.com/dynm/pico-flexray) | Pico 2 (RP2350) plus FlexRay transceivers, panda-compatible over USB |
| openpilot with FlexRay support | [dynm/sunnypilot](https://github.com/dynm/sunnypilot) | sunnypilot fork with pico-flexray panda support and FlexRay-aware Cabana. Its opendbc fork has one FlexRay platform, `BMW_SP2018`. No Audi |
| A FlexRay port on the road | [JLR EVA2 demo](https://youtu.be/DwmWbSW71b4) | comma 3X plus pico-flexray, May 2026 |
| MLB-evo bus data | [robbederks/q8_flexray_dumps](https://github.com/robbederks/q8_flexray_dumps) | Saleae dumps and decoded CSV of the EPS FlexRay bus on a 2019 Audi Q8 |
| MLB-evo DBC | not public | Dennis (comma Discord) has MLB-evo DBC files |
| FlexRay porting workflow | [gericho's BMW guide](https://github.com/dynm/pico-flexray/discussions/3) | BMW connectors, but the Cabana capture and demux workflow carries over |

## A7-specific notes

- C8 chassis code `4K`, WMI `WAU`.
- Possible tap point: Au7_Unleashed (comma Discord, 2026-08-25) found FlexRay pins at the J533 gateway, on the bus between the driver assistance unit (J1121) and the gateway, from the A7 wiring diagram. Not verified on this branch.
- Do not reuse `vw_mlb.dbc` signal layouts. `vw_mlbevo.dbc` on this branch is a placeholder copy only.

## Unknowns

- Which FlexRay frames and bits carry steering torque request, driver torque, steering angle, wheel speeds, ACC state and buttons on the A7.
- Whether the A7's EPS FlexRay layout matches the Q8 captures.
- ECU firmware fingerprints for the A7.

## Next steps for this branch

1. Get an MLB-evo DBC.
2. Listen-only tap on the car; record drives in Cabana.
3. Verify each signal against the DBC and the car.
4. Add `volkswagenMlbEvo` to `SafetyModel` in `opendbc/car/car.capnp`, register the safety hooks, and make `test_volkswagen_mlb_evo.py` run instead of skip.
5. Bench test with the EPS off the car before any on-road use.
