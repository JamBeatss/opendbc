# Audi A7 C8 (MLB evo) openpilot port - staging fork

**Status: Phase 0 - no measurements yet.** No code in this branch has been tested on a car. All additions below are placeholders marked `# TODO`. **Do not merge upstream. Do not install.**

## Car

- 2021 Audi A7, US-spec, C8 chassis (4K), Premium Plus trim, 3.0T V6 EA839.
- Platform: MLB evo. Steering (EPS) and driver-assist traffic on FlexRay.
- Factory Driver Assistance package (adaptive cruise + lane assist).

## Why this fork exists

Upstream opendbc already supports MLB (pre-evo, CAN) via `VolkswagenMLBPlatformConfig` with Porsche Macan Mk1 as the only entry. MLB evo (FlexRay) has no upstream support as of 2026-10-04. This branch is the staging area for an eventual PR that would add it: a new `VolkswagenMLBEvoPlatformConfig` class alongside the existing MLB one, with the Audi A7 as the first entry.

## Hardware approach

pico-flexray MITM module (https://github.com/dynm/pico-flexray) translates FlexRay to CAN so that above the panda-USB layer the car looks CAN. This lets us reuse the opendbc Volkswagen brand folder rather than start a new brand.

## Collaborators (comma.ai Discord)

- **Au7_Unleashed** - owns a 2019 A7 MLB evo. Found the FlexRay tap point at the J533 gateway between J1121 (driver assistance) and the gateway.
- **Nitrogen** (CAI) - author of pico-flexray. Universal tap-test: look for frame `0x44` on the BDC.
- **Dennis** - has MLB-Evo DBC files, publicly linked the rusefi B8_Q5_D4_C7_MLB.dbc.
- **CzokNorris** - producing v3 pico-flexray boards at JLCPCB.
- **gericho** - wrote the WIP BMW porting guide (https://github.com/dynm/pico-flexray/discussions/3).
- **Pravin K** / dolson8874 - have the first working FlexRay port on-road (JLR Evoque EVA2 2021+ via pico-flexray + comma 3x, https://youtu.be/DwmWbSW71b4).

## Phases

- Phase 0 (now): design, offline signal study against Robbe Derks' Q8 FlexRay dumps (https://github.com/robbederks/q8_flexray_dumps), community recon. **No hardware purchased yet.**
- Phase 0.5: offline signal identification against the Q8 dumps + MLB-evo DBC files (Dennis has some). Build a draft `vw_mlbevo.dbc` from an MLB-evo DBC, not from `vw_mlb.dbc` or the pre-evo rusefi MLB DBC (per Dennis, 2026-10-06, the MLB DBC is not correct for MLB evo).
- Phase 1: Pico 2 (RP2350) + TLE9222 transceiver + harness. Flash pico-flexray firmware. Four-resistor passive snooper first if Nitrogen shares the schematic.
- Phase 2: read-only tap at the J533 gateway. Confirm frame `0x44` on the BDC. Collect Cabana routes in varied driving states.
- Phase 3: signal identification in Cabana (demux cycle count, label steering/wheel speeds/ACC/buttons/doors).
- Phase 4: bench rig with EPS off the car, inject through pico bridge, pass the opendbc safety test suite.
- Phase 5: low-speed on-road, empty lot.
- Phase 6: PR to commaai/opendbc.

## What's in this branch

- `opendbc/car/volkswagen/values.py` - placeholder `AUDI_A7_MK2` entry under a new `VolkswagenMLBEvoPlatformConfig` class with a `VolkswagenFlags.MLB_EVO = 512` flag. All marked `# TODO`.
- `opendbc/dbc/vw_mlbevo.dbc` - placeholder, byte-identical copy of `vw_mlb.dbc` with a header comment marking it as a placeholder. Known not to match MLB evo (per Dennis, 2026-10-06); to be replaced with a real MLB-evo DBC.
- `RESEARCH.md` - compiled findings from the 2026-10-04 research session: the upstream opendbc MLB layout, VolkswagenFlags bit allocation, the pre-evo MLB signal map (reference only; does not carry over to MLB evo), pico-flexray architecture, candidate taps, DBC sources, template ports to copy, and the reading list.

## What is NOT here (TODO before any PR)

- Firmware fingerprints from the reference A7 (VCDS/OBDeleven scan pending).
- Actual MLB-evo frame IDs and signal layouts (Phase 2/3 captures pending).
- `fingerprints.py` entries.
- Any safety file changes (`opendbc/safety/modes/volkswagen_mlb.h` or a new `volkswagen_mlb_evo.h`).
- Any CARS.md documentation entry.
- A passing `test_volkswagen_mlb_evo.py`.
- Bench or on-road testing.

## References

- comma.ai Discord thread "Low cost FlexRay MITM module" in #car-port-projects.
- pico-flexray: https://github.com/dynm/pico-flexray
- pandad-pico-flexray: https://github.com/dynm/pandad-pico-flexray
- Modified Cabana (cabana-flexray branch): https://github.com/dynm/openpilot
- JLR port proof: https://github.com/dolson8874/opendbc
- Q8 FlexRay dumps: https://github.com/robbederks/q8_flexray_dumps
- rusefi MLB DBC: https://github.com/rusefi/rusefi_documentation/blob/master/OEM-Docs/VAG/B8_Q5_D4_C7_MLB.dbc

## License

Mirrors opendbc's MIT license. All original edits in this branch are MIT.
