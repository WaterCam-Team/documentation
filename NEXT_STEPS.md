# WaterCam next steps

**Updated:** 2026-10-04. In rough priority order. Each item names the repo and
PR or doc to start from.

## 1. Flash and test the emergency-packet firmware fix
**Repo:** mDot-AT-firmware, PR #2 (open)

The mDot firmware currently treats **any** one-byte downlink as the emergency
message `!` and powers the Pi on. PR #2 accepts only `!` (0x21).

- [ ] Build in Mbed Studio and flash one mDot (unit 005).
- [ ] With the Pi off, send `21` on f_port 10: PB_1 should pulse and the Pi should boot.
- [ ] Send another single byte, e.g. `22`: it should be forwarded, with no power-on.
- [ ] Send `5001`: the debug-status reply should still arrive.
- [ ] Merge, then flash the rest of the fleet.

## 2. Test emergency start end to end
**Repos:** mDot-AT-firmware, SU-WaterCam

When the Pi is off, the firmware prints its `EMERGENCY:` lines before the Pi
has booted, so the node may never see them. If so, a unit powered on by an
emergency runs a normal cycle and shuts down again.

- [ ] With the Pi off, send `21` and check whether `emergency_mode` is set after boot.
- [ ] If it isn't, pass the emergency on after boot: for example, the firmware re-reports it once the Pi is up, or the node asks the mDot at start-up.

## 3. New segmentation masks, then retrain
**Repos:** photo-masks, segformer_5band (`NEXT_CHECKPOINT.md`), SU-WaterCam #91 (draft)

The deployed model (`iter_100.pth`, 100 training iterations) is effectively
untrained: water IoU against hand-labelled masks was 0.14 and 0.01 on the two
example scenes. Segmentation runs on the units, but the flood bitmaps aren't
meaningful yet.

- [ ] Label new masks in the annotator. Label evaluation scenes from scratch, not over an auto seed.
- [ ] Train the next checkpoint and validate it against the held-out scenes.
- [ ] Decide on #91's two settings, the raw NIR band and divisible input sizing, using the new checkpoint. Merge #91 if they help.
- [ ] Export to ONNX through the metadata-aware exporter, and deploy to the units.

## 4. Update the field units (007–011)
**Doc:** SU-WaterCam `docs/FIELD_DEPLOYMENT_GUIDE.md`

Use WiFi on the bench where possible, to save cellular data. For each unit:
- [ ] Pull SU-WaterCam `main`.
- [ ] Install and enable `lora_daemon`, `segformer_daemon`, `wittypi-recovery` and `wittypi-boot-mark`.
- [ ] Copy `config/wittypi/beforeScript.sh`, and set `recovery_boot.enabled=true` for the field.
- [ ] Install `config/journald-volatile.conf` (journal in RAM).
- [ ] Give the unit its own machine ID (guide step 1.2).
- [ ] Use stock clocks (`arm_freq=1800`) to lower peak current.
- [ ] Keep Witty Pi "Default ON" with the 10 s delay; press the button at deployment.

## 5. Resolve the `14 94` command collision
**Repos:** API, SU-WaterCam

The dashboard sends "flood code frequency" as an index (0–5) on `14 94`, but
the node maps `14 94` to `photo_interval`, in seconds, minimum 30. The node
rejects those values, so neither setting can currently be changed from the
dashboard.

- [ ] Decide what `14 94` means, give the other setting its own code, and make the API encoder and the node's LoRa and IP parsers agree.

## 6. Measure power properly
**Doc:** SU-WaterCam `docs/POWER_ANALYSIS.md`

- [ ] Measure the off-state draw: the Witty Pi plus the mDot listening in Class C, with the Pi off. It may be as large as all the wakes combined.
- [ ] Measure the energy per wake with a logger sampling at least once a second. The two captures were measured at 5 Hz on 2026-10-06 (75–87 mWh); boot and shutdown still weren't.
- [ ] Update `POWER_ANALYSIS.md` with both.

## 7. Open field-power tests
**Doc:** SU-WaterCam `docs/UNIT006_POWER_FAILURE.md`

- [ ] Run a load test on 006 while the V50 charges from the real panel through its side solar port, in sun.
- [ ] Do a full drain with only the panel connected, the power logger running and `recovery_boot` on, and check that the unit recovers by itself.
- [ ] Summarise the Witty Pi logs from 008–011: wakes after a power loss, and boots that never set a next wake. This costs a few KB of cellular data per unit, so get approval first.
- [x] Decide 006's configuration: stock clocks and the packaged kernel, applied 2026-10-06.

## 8. Measure battery charge
**Doc:** SU-WaterCam `docs/POWER_ANALYSIS.md`, section *Battery state of charge*

Units have no state-of-charge sensor, and since SU-WaterCam #122 they send no battery percentage. The voltage-based one was noise. Plan: read the V50's own charge signal (about half the cell voltage on its USB-C D+ pin) through the mDot's free `PB_0` ADC.

- [ ] Confirm the field V50s are the "Always On" model the D+ signal is documented for.
- [ ] Prototype on one unit: a USB-C breakout in the V50's top port with only D+ and GND connected, D+ to mDot `PB_0` through about 10 kΩ, and 100 nF to GND.
- [ ] mDot-AT-firmware: an AT command that reads `AnalogIn(PB_0)` (Mbed Studio build).
- [ ] SU-WaterCam `battery_manager.py`: a first path that asks the mDot through the LoRa daemon; fix `CELL_V_MIN` (V50 cuts out near 3.45 V/cell) and the INA260 path's coulomb counting.
- [ ] Calibrate D+ against charge during the full drain test (section 7), logging D+ once a minute.

## 9. Finish the v6 HAT board
**Repo:** pcb-designs, PR #3 (draft); see `SESSION_HANDOFF.md`

- [ ] Route the remaining 26 connections and clear the 17 DRC violations.
- [ ] **Verify BNO055 (U2) pad 1 against the physical breakout.** The footprint has no orientation marking, and a reversed part puts 3.3 V on RST.
- [ ] Fix the `MDOT:MTDOT` footprint: it has a duplicate pad 24 and no pad 28.
- [ ] Settle the open assembly choices (BNO055 JP2, R6/R7, R3's footprint, HAT EEPROM), build the fab package, and order samples from OSHPark.
- [ ] Add a connector for the V50 D+ charge signal to mDot `PB_0` (section 8). The mDot symbol's pin numbers don't match MultiTech's guide (PB0/PB1 and PA0/PA7 are swapped), so lay out by pin name.

## 10. Later
- [ ] Cap how long emergency mode can run (e.g. 6–8 h), then fall back to frequent scheduled wakes. Emergency mode keeps a unit on continuously, about 70–110 Wh/day.
- [ ] Investigate updating the segmentation model over the air, within the cellular data limits.
- [ ] Test ONNX export and inference for the recent SegFormer 3-band model.
- [ ] Cache the camera's white-balance gains across boots, saving about 5 s per wake.
- [ ] Reduce Tailscale's cellular data while keeping it for remote access (parked 2026-10-07). A cellular wake used 126–474 KB, about 10 KB of it ours. Options and estimates are in SU-WaterCam `docs/CELLULAR_DATA.md`.
