# Audi A7 MLB evo port - compiled research (2026-10-04)

## Corrections (2026-10-06)

Two claims in the first version of this file were wrong, as pointed out by Dennis in the comma.ai Discord:

1. **The Macan is not MLB evo.** All combustion Macans (95B) are on the MLB "Cajun" platform, very similar to MLB B8 (A4, A5, Q5), non-FlexRay, and supportable by openpilot. The post-2018 **Cayenne** is the MLB-evo / FlexRay Porsche; it was conflated with the Macan.
2. **The pre-evo MLB DBC is not correct for MLB evo.** Do not use `vw_mlb.dbc` signal names or layouts as a starting point for an MLB-evo port. MLB-evo DBC files exist; start from those.

The sections below have been edited accordingly.

A one-file dump of what was learned during the 2026-10-04 research session. Everything in this file is a reference, not a plan. The project is paused as of that date; this file exists so that the next person to revisit the fork - the fork owner, or anyone they later share the fork with - doesn't start from zero.

## Why MLB evo is the gap

Upstream `opendbc/car/volkswagen/` already supports three VAG platforms:

| Flag bit | Platform | DBC | First car entry |
|---|---|---|---|
| `MLB = 8` | MLB (pre-evo), CAN-based | `vw_mlb.dbc` (144 messages) | `PORSCHE_MACAN_MK1` (2017-24) |
| `MQB` (default, no flag) | MQB, CAN | `vw_mqb.dbc` | VW/Audi/Skoda/CUPRA/SEAT MQB cars |
| `PQ = 2` | PQ, CAN | `vw_pq.dbc` | VW/Audi PQ cars |
| `MEB = 16` | MEB, CAN (electric) | `vw_meb.dbc` | VW ID.* |

The DBC family also includes `vw_mqbevo.dbc` for the newer MQB evo generation, which is the precedent for the "evo as its own thing" pattern we would mirror for MLB.

**MLB evo (2018+ A6/A7/A8/Q7/Q8/e-tron, Porsche Cayenne (2018+), Bentley Bentayga, Lamborghini Urus and others) is unsupported** because its ADAS traffic runs on **FlexRay** instead of CAN. All openpilot safety and panda code assumes CAN.

Dennis in #volkswagen-audi-porsche summed it up: "MLB-Evo uses Flexray. So same deal unfortunately." Jason Young (`jyoung8607`, ultra openpilot contributor) in #general: "Most Porsche hasn't really been investigated... but many of them are likely to use MLB (non-evo) Audi Q5, and is purely CAN, so we can drive it. ... Certain older ones, but not the current gen MLB evo ones unfortunately."

## The hardware bridge: pico-flexray

`dynm/pico-flexray` (https://github.com/dynm/pico-flexray) is the FlexRay-to-CAN translator that closes the gap. Nitrogen (CAI badge, OP of the "Low cost FlexRay MITM module" thread) built it:

- **Hardware:** Raspberry Pi **Pico 2 (RP2350)**, not the original Pico. "BREAKING CHANGE: Please use the Pico 2 RP2350. I found it can simplify DMA triggers, which the RP2040 does not support." Plus 1 to 4 FlexRay transceivers (TLE9222 preferred; TJA1082 or NCV7383 pin-compatible).
- **Firmware architecture:** PIO at 100 MHz, 10x oversampling of FlexRay, three modules - BSS Streamer (reads RXD, pushes FlexRay data to CPU), Detector (originally planned, now dropped), Forwarder (RXD_0 to TXD_1, or CPU-generated override on TXD_1).
- **USB side:** panda-compatible, so Cabana sees it as a comma panda.
- **Modified Cabana:** `dynm/openpilot` branch `cabana-flexray`, with per-message demux dropdown and bit-level mux assignment in Message Detail View.
- **Test rig built in:** jumper pins 13 and 15 on the Pico 2. Pin 15 plays back Q8 FlexRay data from `robbederks/q8_flexray_dumps`, pin 13 reads it. Validates the whole BSS Streamer and Cabana pipeline off-car.
- **Four-resistor passive snooper (theoretical, untested):** Nitrogen in #Low cost FlexRay MITM module 2026-08-25: "You can even build a passive FlexRay snooper with just four resistors, no transceiver required. Use PIO as a simple comparator to recover the differential signal. I think it should work, but haven't tried it yet." First builder validates.

## Known-good proof: the JLR port drives on-road

Pravin K (`pravink2089` on YouTube, Discord credits "dolson/defender/flexray") posted "JLR steer with Comma 3X + MITM FlexRay" on 2026-05-15 (https://youtu.be/DwmWbSW71b4). Video description: "This should work on all EVA2 vehicles model year 2021 onwards. Includes Defender, Discovery +Sport, Velar, RR +Sport. Pico-FlexRay MITM is very cheap, so if you have a 3X it shouldn't be hard to hook up. The steering is a jittery at low speeds, which should get resolved as time progresses." sunnyhaibin (ultra openpilot contributor) forwarded it in #dev-opendbc 2026-05-19 crediting Nitrogen, dolson8874, CzokNorris. adeeb (CAI moderator) replied: "very cool to see the flexray ports rolling in!"

**dolson8874's public repos** (platform-identical pattern to copy):

| Repo | Purpose |
|---|---|
| `dolson8874/LandRover-Harness` | Physical harness design for the JLR tap. 16 stars, pushed 2025-09-25. |
| `dolson8874/flexray-interceptor` | Fork of `pd0wm/flexray-interceptor` with 2025-09-24 updates. |
| `dolson8874/opendbc` | opendbc fork. **Note (verified 2026-10-04): master branch has no `car/landrover/` folder; the JLR car-port code pravink2089 pointed people at in the YouTube comments must live in a draft branch or private tree. Worth asking dolson or Pravin where.** |
| `dolson8874/q8_flexray_dumps` | Mirror of Robbe Derks' Q8 captures. |

## Where the signals live on MLB (pre-evo) - reference only, not MLB evo

Upstream `opendbc/safety/modes/volkswagen_mlb.h` already enforces these frames for MLB pre-evo. Per Dennis (2026-10-06), these do not carry over to MLB evo. Use an MLB-evo DBC instead.

| Message | Hex ID | Role in safety file | Signal of interest |
|---|---|---|---|
| `ESP_03` | 0x100 | Wheel-speed vehicle_moving check | Four 12-bit fields: `ESP_VL_Radgeschw`, `ESP_VR_Radgeschw`, `ESP_HL_Radgeschw`, `ESP_HR_Radgeschw` (FL/FR/RL/RR) |
| `LH_EPS_03` | 0x11D | Driver input torque sample | `volkswagen_mlb_mqb_driver_input_torque(msg)` |
| `ESP_05` | 0x106 | Brake pressure detected | Bit 26 |
| `ACC_05` | 0x10D | Stock cruise engage/disengage | `ACC_Status_ACC` bits 7-1 (states 3/4/5 = engaged, 2 = main-on standby) |
| `Motor_03` | 0x121 | Gas pedal and brake switch | `MO_Fahrpedalrohwert_01` (byte 6), `MO_Fahrer_bremst` (bit 35) |
| `LS_01` | 0x12B | Cancel button (and resume) | `LS_Abbrechen` (bit 13), `LS_Tip_Wiederaufnahme`, `LS_Hauptschalter` |
| `HCA_01` | 0x126 | openpilot steering output | `HCA_01_LM_Offset`, `HCA_01_LM_OffSign`, `HCA_01_Status_HCA` (7 engaged / 3 standby) |

(Decimal IDs from `vw_mlb.dbc`: `ESP_03 = 256`, `LH_EPS_03 = 285`, `ESP_05 = 262`, `ACC_05 = 269`, `Motor_03 = 289`, `LS_01 = 299`, `HCA_01 = 294`.)

Dennis publicly linked a richer pre-evo DBC in #volkswagen-audi-porsche (2025-11-10):

> https://raw.githubusercontent.com/rusefi/rusefi_documentation/master/OEM-Docs/VAG/B8_Q5_D4_C7_MLB.dbc

That file is GPL-licensed and should not be copied into this MIT-licensed fork. Download it to a scratch folder for signal-ID work; do not commit it.

Dennis also offered MLB-Evo DBCs directly (#flexray 2026-02-16): "Do you have the DBC files for that car? I actually have DBC files for MLB-Evo." Reach out to him (`patatjes` on Discord) for those.

## The universal tap test: frame 0x44

Nitrogen's go/no-go check after wiring up pico-flexray (#Low cost FlexRay MITM module 2026-06-30 and 2026-08-07):

> "You can tap any FlexRay line on the BDC with pico-flexray, open cabana, and see if frame 0x44 shows up."

Frame `0x44` is not in the pre-evo `vw_mlb.dbc`. Treat it as MLB-evo-specific and a required addition to any eventual `vw_mlbevo.dbc`.

## Candidate taps on the Audi A7 C8

Au7_Unleashed (2019 A7 MLBevo per his words - note the "4G" vs the real C8 "4K" chassis-code question to resolve) posted the Audi A7 electrical circuit diagram page 20/3 in the thread on 2026-08-25: "found best spot is thru the gateway where there is pins to access the flexray bus between driving assist module and GW". The diagram shows:

- `J533` = Data bus diagnostic interface (gateway)
- `J1121` = Driver assistance systems control unit (= address **00A5**, the module Audi recall 90TV flashes on 2021-2023 A6/A7)
- `J775` = Chassis control unit
- Connectors on J533: `T40a` (40-pin black), `T54c` (54-pin black), `T81a` (81-pin black), `T81b` (81-pin black)
- Variants on the diagram: *2 EPS, *3 no ride-height ctrl, *4 active steering, *5 driver assistance, *6 additional

Nitrogen's alternative suggestion (same thread, 2026-06-30): "You can tap any FlexRay line on the BDC with pico-flexray". **BDC = Body Domain Controller.** Which is the right tap on a C8 A7 is an open question for Au7 or Nitrogen.

## What a port adds to opendbc

Infrastructure-wise, this is additive, not a brand-new brand folder. The pattern is:

1. New DBC `opendbc/dbc/vw_mlbevo.dbc`, built from MLB-evo DBC sources (e.g. the files Dennis has), **not** from a copy of `vw_mlb.dbc`. It must include MLB-evo-specific frames like `0x44`.
2. New `VolkswagenMLBEvoPlatformConfig` class in `opendbc/car/volkswagen/values.py` with a new `VolkswagenFlags.MLB_EVO` bit.
3. Per-car entry alongside `PORSCHE_MACAN_MK1` (which is the only MLB entry today).
4. Firmware fingerprints in `fingerprints.py` from a real car via VCDS/OBDeleven.
5. Either reuse `mlbcan.py` or add parallel `mlbevocan.py` depending on bit-layout verification.
6. Either extend `volkswagen_mlb.h` safety file with MLB_EVO cases, or clone to `volkswagen_mlb_evo.h`.
7. Matching `test_volkswagen_mlb_evo.py` must pass.
8. `opendbc/docs/CARS.md` entry.

### VolkswagenFlags bit allocation (verified upstream 2026-10-04)

| Bit | Flag | Status |
|---|---|---|
| 1 | STOCK_HCA_PRESENT | taken |
| 2 | PQ | taken |
| 4 | KOMBI_PRESENT | taken |
| 8 | MLB | taken |
| 16 | MEB | taken |
| 32 | ALT_GEAR | taken |
| 64 | STOCK_KLR_PRESENT | taken |
| 128 | MEB_GEN2 | taken |
| 256 | HAS_BSM | taken |
| **512** | **(available - MLB_EVO uses this in this branch)** | - |

Important: Python's `IntFlag` silently makes a duplicate-value entry an alias instead of raising at import. The first `MLB_EVO = 16` commit in this branch aliased MEB. The follow-up commit `7962b7b` bumped MLB_EVO to 512.

## The methodology to copy

Pravin K's methodology for JLR (works, demonstrated on-road):
1. Wiring diagram to find the right ECU bus.
2. Harness build (dolson8874's LandRover-Harness is the template).
3. pico-flexray in read-only mode first.
4. Cabana captures across all relevant car states (manual, cruise on/off, lane-keep on/off, left/right steer input, cruise button presses, gear shifts, door and belt switches).
5. Demux the FlexRay cycle count in cabana-flexray to split messages that carry different functions on different cycles.
6. Build the DBC incrementally - signal by signal, verified visually in Cabana.
7. opendbc port against that DBC.
8. Bench test with EPS off the car before anything on-road.
9. Low-speed on-road in an empty lot.
10. Submit upstream.

gericho's BMW porting guide (https://github.com/dynm/pico-flexray/discussions/3) is the step-by-step companion. Required reads:

- Ubuntu 24 LTS as the OS for Cabana.
- `picotool` udev rules on first setup (gericho's exact commands are in that discussion).
- Cabana setting: Drag Directions = "Always Little Endian".
- Routes are logged as soon as Cabana opens; stored in `/home/user/`, split into 1-minute segments.
- Demux: pick 2 or 3 bits at position 0 of the frame, set them to "multiplexor signal", then drag the actual payload bits as "multiplexed signal" with mux value 0 to 2^n - 1.
- FlexRay specifics explained in gericho's own words: "cycle repetition = 2 means two mux slots. Even cycles might carry braking, odd cycles might carry steering."

## Who to talk to in the comma.ai Discord (2026-10-04)

| Handle | Why |
|---|---|
| `Au7_Unleashed` (user tag `au7_unleashed`) | Owns an A7 MLB evo. First mover on the A7 tap-point question. |
| `Nitrogen` (CAI badge) | pico-flexray author. Advises on tap points, hardware, snooper schematics. |
| `Dennis` (user tag `patatjes`) | Has MLB-Evo DBC files. Publicly linked the rusefi pre-evo MLB DBC. |
| `CzokNorris` | Producing pico-flexray v3 boards at JLCPCB. Possible source of assembled hardware. |
| `sunnyhaibin` (SP, ultra openpilot contributor) | Promotes FlexRay ports in dev-opendbc. |
| `jyoung8607` (crown, ultra openpilot contributor) | Jason Young, author of the comma_con car-port talk; MLB expert. |
| `adeeb` (CAI, moderator) | Comma moderator, supportive of FlexRay work. |
| `gericho` | Author of the WIP BMW porting guide. |
| `pravink2089` / `dolson8874` (not Discord handles - GitHub/YouTube) | JLR port authors. |
| `fff111` (CAI) | Comma, iterating on the VW_MLB DBC in dev-opendbc. |

## Reading list for the next session

1. `commaai/opendbc` README and `opendbc/car/volkswagen/` (values.py, mlbcan.py, carcontroller.py, carstate.py).
2. `opendbc/safety/modes/volkswagen_mlb.h` and `opendbc/safety/tests/test_volkswagen_mlb.py`.
3. `opendbc/dbc/vw_mlb.dbc` - 144 messages, pre-evo baseline.
4. https://github.com/dynm/pico-flexray README and `src/main.c`.
5. https://github.com/dynm/pandad-pico-flexray `main.cc` and `panda_safety.cc`.
6. https://github.com/dynm/pico-flexray/discussions/3 (gericho's BMW guide).
7. https://blog.comma.ai/hacking-an-audi-performing-a-man-in-the-middle-attack-on-flexray/ (2020-03-03, the original Q8 proof of concept).
8. https://github.com/robbederks/q8_flexray_dumps (the signal-labeling gold mine).
9. Jason Young's car-port talk: https://www.youtube.com/watch?v=XxPS5TpTUnI
10. zsec blog "Enabling Car Play and Android Auto for free on VW and Audi MMI 2026" https://blog.zsec.uk/audi-carplay-2026/ (background on MIB3 vs MH2p; unrelated to the port but useful general reference).

## Status at the time this file was written

Research-only. Nothing in this fork has been tested on a car. The `AUDI_A7_MK2` entry in `values.py` has placeholder specs that must be verified. `vw_mlbevo.dbc` is a copy of `vw_mlb.dbc`. **Do not merge upstream. Do not install.**

Project paused 2026-10-04. The next trigger to revisit is whichever of: (a) a public MLB evo port appears from Au7_Unleashed, Dennis, or the comma community; (b) Dennis's MLB-Evo DBCs become public; (c) a different approach (EPS swap, factory ADAS integration via activation) becomes attractive.
