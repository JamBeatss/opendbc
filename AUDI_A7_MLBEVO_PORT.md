# Audi A7 C8 (MLB evo) port - draft

Draft of an openpilot/opendbc port for the Audi A7 C8 (chassis `4K`, MLB evo platform, FlexRay translated to CAN via https://github.com/dynm/pico-flexray). Reference car: 2021 US A7 Premium Plus 3.0T.

## Status

Skeleton. Compiles. The structural safety test skips until `volkswagenMlbEvo` is added to `SafetyModel` in `opendbc/car/car.capnp`. **Signals not verified against the car. Do not install.**

## In this branch

- `opendbc/car/volkswagen/values.py` - `AUDI_A7_MK2` entry under new `VolkswagenMLBEvoPlatformConfig` + `VolkswagenFlags.MLB_EVO = 512`
- `opendbc/car/volkswagen/fingerprints.py` - `AUDI_A7_MK2` ECU slot skeleton, all empty
- `opendbc/safety/modes/volkswagen_mlb_evo.h` - safety file, cloned from `volkswagen_mlb.h` with WIP banner
- `opendbc/safety/tests/test_volkswagen_mlb_evo.py` - structural test, skips until `volkswagenMlbEvo` is added to `SafetyModel` in `opendbc/car/car.capnp` and the safety hooks are registered
- `opendbc/dbc/vw_mlbevo.dbc` - placeholder, byte-identical to `vw_mlb.dbc`, to be replaced with a real MLB-evo DBC
- `RESEARCH.md` - short notes: what exists for FlexRay and MLB evo today, and what is unknown for the A7

## Open blockers

- Firmware fingerprints from the car (VCDS/OBDeleven scan)
- MLB-evo DBC (pre-evo MLB does not carry over; MLB-evo DBCs exist in the community)
- Per-signal verification via Cabana captures
- Bench test with EPS off the car + safety suite pass before any on-road use

## License

MIT, inherited from upstream opendbc.
