# WaterCam next steps

**Updated:** 2026-10-22. In rough priority order. Each item names the repo and
PR or doc to start from.

## Now: get the units deployable
**Repos:** SU-WaterCam, mDot-AT-firmware, pcb-designs

**State on 2026-10-22.** Node release v2026.10.7 (SU-WaterCam, signed; needs the API at `3138fda` or later, which production runs). 005 and 006 are on v2026.10.6 and v2026.10.7; 008–011 are still on v2026.10.2 and switched off.

- [ ] **006: 24 h run on v2026.10.7** with the dashboard's sun-following schedule (sunrise+30 to sunset−15, hourly, 10 min), delivered and acked over IP on 2026-10-22. Check the wakes, uploads, USB-off and the daily GPS window from its logs.
- [ ] **mDot firmware: the reset loop when it can't join** (mDot-AT-firmware #22, draft). With an unreachable sub-band, sw6 and sw7 both restart every 20 s without reaching the join, so the self watchdog isn't being kicked; the likely cause is a crash early in the join path, whose report goes to the unwired debug port. A diagnostic build with crash capture is ready for 005, which first needs a power cycle (its bootloader hung during a flash). Then the fix, then flash 006 once, before it deploys.
- [ ] mDot flash tool: `flash --reset-loop` for an mDot that keeps restarting (mDot-AT-firmware #24).
- [ ] mDot firmware, after the fix: report the reset reason and any crash address on the AT port at start-up, so a field unit can say why its mDot restarted.
- [ ] Emergency mode sleeps between captures (SU-WaterCam #243): about 15–22 Wh/day at 10 minutes instead of 70–110. Bench test on a unit before release.
- [ ] Measure the mDot's receive current with the Pi off (SU-WaterCam #244): the datasheet's 43 mA would add about 3 Wh/day to every schedule.
- [ ] Tilt test of the pose levels on 005 (by hand: about 2°, 7° and 25°).
- [ ] 008–011: power on over WiFi and update to the current release, with v2026.10.6's install steps.

Found on 2026-10-19: unit power cuts logged by the Witty Pi as "power supply is
newly connected" have two hardware causes, and neither shows a voltage sag first.
- **The Witty Pi's USB-C socket.** On 006, pushing or wiggling the plug cut power
  with two different adapters and cables, and tilting the box did the same. The
  fix is to feed 5 V through the Witty Pi's P2 header instead
  (SU-WaterCam `docs/WITTYPI_P2_POWER_HARNESS.md`).
- **The mDot's jumper wiring.** The mDot runs from the Witty Pi's own 3.3 V rail,
  so disturbing its wires reset the whole unit (005). A lifted mDot ground also
  left the mDot hung until it was power-cycled.

- [ ] 006: fit the P2 harness, secure the mDot wiring, then wiggle- and tilt-test with the lid closed.
- [ ] 006: gyro calibration stage, deployment config (ticktalk, schedule, next wake), then magnetometer, mount angle and a LoRa check at the site.
- [x] Release v2026.10.4 (and since then v2026.10.5–v2026.10.7).
- [ ] Every other unit: the same USB-C push/wiggle test (ideally the P2 feed), secure mDot wiring with PA_6 connected, IMU calibration, update by tag.
- [ ] mDot firmware `4.1.5-wc2610-sw2` (mDot-AT-firmware #12, #14): restarts the mDot if its firmware stops, or if the Pi is on and no AT input arrives for 30 minutes. Units need v2026.10.4's keep-alive **before** this firmware. Tested on 005 on 2026-10-19 (flash, join, remote wake, an hour idle without a restart).
- [ ] Decide whether a small cap on the Witty Pi's 3.3 V rail helps: one unit ran 100 µF across P3 3V3/GND overnight on 2026-10-19; compare with a night without it.

## 1. Flash and test the emergency-packet firmware fix
**Repo:** mDot-AT-firmware, PR #2 (open)

The mDot firmware currently treats **any** one-byte downlink as the emergency
message `!` and powers the Pi on. PR #2 accepts only `!` (0x21).

- [x] Build and flash one mDot (unit 005): builds on the command line now; flashed over serial (mDot-AT-firmware `docs/BUILD_AND_FLASH.md`).
- [x] With the Pi off, send `21` on f_port 10: PB_1 should pulse and the Pi should boot. Remote wake works on 005 (2026-10-10, and twice on 2026-10-19).
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

008–011 were prepared and tested on 2026-10-15 and run release v2026.10.2; the current release is v2026.10.7. Update units by release tag rather than `main` (SU-WaterCam `docs/RELEASES.md`); the list below is what a unit needs, for checking one.

Use WiFi on the bench where possible, to save cellular data. For each unit:
- [ ] Pull SU-WaterCam `main`.
- [ ] Install and enable `lora_daemon`, `segformer_daemon`, `wittypi-recovery`, `wittypi-boot-mark` and `wittypi-next-wake` (SU-WaterCam #156: re-arms the next schedule slot at every shutdown, so a boot between slots can't skip one).
- [ ] Copy `config/wittypi/beforeScript.sh`, and set `recovery_boot.enabled=true` for the field.
- [ ] Install `config/journald-volatile.conf` (journal in RAM).
- [ ] Give the unit its own machine ID (guide step 1.2).
- [ ] Use stock clocks (`arm_freq=1800`) to lower peak current.
- [ ] Keep Witty Pi "Default ON" with the 10 s delay; press the button at deployment.
- [ ] Switch DNS to systemd-resolved (guide 1.3, SU-WaterCam #145). Without it, each Quectel up/down drops MagicDNS and the uplink falls back to cellular on every wake. Restart tailscaled detached, since it drops a Tailscale SSH session. Install `config/systemd-resolved/90-watercam.conf` too (SU-WaterCam #153: LLMNR and multicast DNS off).
- [ ] Keep cellular on demand and filtered (guide 1.5.1): `Quectel` autoconnect off, the polkit rule, and `watercam-cellular-filter.service` enabled (SU-WaterCam #152). The filter keeps Tailscale's private-address probes off the modem: 19–28 KB per cellular wake down to 0, measured on 006 on 2026-10-12.
- [ ] Remove packages a node doesn't need (guide 1.3.2, SU-WaterCam #161): `neovim`, `wget`, `lrzsz`, the bookworm leftovers (`gcc-12`, `python3.11-minimal`), leftover config, and the venv's dead `lib/python3.11` (about 300 MB). Done on 005 and 006 on 2026-10-12.
- [ ] Run `segformer_daemon` from the venv (SU-WaterCam #142): pip-install the pinned onnxruntime, flatbuffers and protobuf from `requirements.txt`, install pip's `opencv-python-headless` from `requirements.txt` and purge apt's `python3-opencv` (SU-WaterCam #162: same results, about 450 MB less; done on 005 and 006 on 2026-10-12), and copy `config/segformer_daemon.service`. Compare one capture's mask before and after.
- [ ] Issue the unit an IP token in Management and put it in `runtime_config.json` as `ip_upload.api_key` (API #122). Once every unit sends one (`GET /ip/auth-status`), set `IP_AUTH=enforce` on the server.

## 5. Check the registration orientation flag in the field
**Repos:** SU-WaterCam #143, API #123

After co-registration, the node compares the IMU pose with the one stored with the cached transform. It sends `registration_status` on `01 08` (0 ok, 1 shifted, 2 adopted, 3 no cache, 4 no IMU), and the dashboard shows "Check alignment" while it is 1. It flags only; the transform is never re-solved automatically. Tested on 005 (LoRa) and 006 (IP) on 2026-10-11: adopted, then ok, and both reached the server.

- [ ] Do IMU stage-1 calibration (`bno055_calibration.json`) on 005, 006 and the field units. Until then heading isn't trusted and only tilt counts. Use `--mode calibrate --stages gyro,accel` at the lab and `--stages mag` at the site; on these units accel usually reaches 1/3 or 2/3 (the antennas block the side positions). 006 has accel saved (1/3).
- [ ] Look at the "Check alignment" pill in a browser; it has only been syntax-checked.
- [ ] When a unit flags, check its alignment by eye, then re-seed or run `python -m tools.registration_orientation --reset`.

## 6. Build a custom node image (pi-gen)
**Doc:** `IMAGE_BUILD_FEASIBILITY.md`

pi-gen on Trixie is feasible. Hostname `ufo-01-01-NNN`, Tailscale and Tailscale SSH are set at first boot from files the flashing script writes to the boot partition, with a per-card auth key, never one built into the image.

- [ ] Remove conda from 005 and 006 once the venv daemon has run for a while (both switched 2026-10-11).
- [ ] Publish the int8 model as a versioned release asset.
- [ ] Write the pi-gen stage (including systemd-resolved, installed last) and the first-boot service; build in Docker or on an arm64 runner.
- [ ] Write the flashing script: asks for NNN and the API key, mints the Tailscale key, writes the boot-partition files.
- [ ] Flash a spare card for 006 and compare it with the current image.

## 7. Resolve the `14 94` command collision
**Repos:** API, SU-WaterCam

The dashboard sends "flood code frequency" as an index (0–5) on `14 94`, but
the node maps `14 94` to `photo_interval`, in seconds, minimum 30. The node
rejects those values, so neither setting can currently be changed from the
dashboard.

- [x] Decide what `14 94` means, give the other setting its own code, and make the API encoder and the node's LoRa and IP parsers agree. Done in SU-WaterCam #134/#136: `14 94` is the flood-code frequency everywhere (index into 10–60 min). `photo_interval` has no remote command; add one only if it's needed.

## 8. Measure power properly
**Doc:** SU-WaterCam `docs/POWER_ANALYSIS.md`

- [ ] Measure the off-state draw: the Witty Pi plus the mDot listening in Class C, with the Pi off. It may be as large as all the wakes combined. USB meters read nothing below 10–20 mA, so use a multimeter in series with the mDot's VDD (SU-WaterCam #244).
- [x] Measure the energy per wake with a logger sampling at least once a second. The two captures were measured at 5 Hz on 2026-10-06 (75–87 mWh); boot (25–35 mWh) and shutdown (10–15 mWh) on 005 on 2026-10-22.
- [ ] Update `POWER_ANALYSIS.md` with both.

## 9. Open field-power tests
**Doc:** SU-WaterCam `docs/UNIT006_POWER_FAILURE.md`

- [ ] Run a load test on 006 while its V75 charges from the real panel through the solar jack (3.5 × 1.1 mm), in sun.
- [ ] Check the V75 in the cold and the heat: does the output come back by itself (Always On) after a cutoff below −15 °C or above 60 °C, and how hot does the enclosure get in sun (charging stops above 45 °C)?
- [ ] Do a full drain with only the panel connected, the power logger running and `recovery_boot` on, and check that the unit recovers by itself.
- [ ] Summarise the Witty Pi logs from 008–011: wakes after a power loss, and boots that never set a next wake. This costs a few KB of cellular data per unit, so get approval first.
- [x] Decide 006's configuration: stock clocks and the packaged kernel, applied 2026-10-06.

## 10. Measure battery charge
**Doc:** SU-WaterCam `docs/POWER_ANALYSIS.md`, section *Battery state of charge*

Units have no state-of-charge sensor, and since SU-WaterCam #122 they send no battery percentage. The voltage-based one was noise. Plan: read the pack's own charge signal with the mDot's ADC. All current units have the new-design Voltaic V75, which puts it on the SBU pins (A8/B8) of its USB-C PD port; old-design packs used D+ (about half the cell voltage).

- [x] Confirm the field packs are Always On models: all units have new-design V75s, wired to both USB-A ports (2026-10-17).
- [ ] Measure the V75's SBU signal against the cell voltage on the bench (the scale isn't published) and check it fits under the mDot's 3.0 V ADC reference.
- [ ] Prototype on one unit: a USB-C breakout in the V75's top port with only SBU and GND connected (no VBUS, no CC), SBU to mDot `PB_0` through about 10 kΩ, and 100 nF to GND. First check that this doesn't switch Always On off on the A ports.
- [ ] mDot-AT-firmware: an AT command that reads `AnalogIn(PB_0)` (Mbed Studio build).
- [ ] SU-WaterCam `battery_manager.py`: a first path that asks the mDot through the LoRa daemon; fix `CELL_V_MIN` (the V50 cut out near 3.45 V/cell; find the V75's point) and the INA260 path's coulomb counting.
- [ ] Calibrate the signal against charge during the full drain test (section 9), logging it once a minute.

## 11. Finish the v6 HAT board
**Repo:** pcb-designs, PR #3 (draft); see `SESSION_HANDOFF.md`

- [ ] Route the remaining 26 connections and clear the 17 DRC violations.
- [ ] **Verify BNO055 (U2) pad 1 against the physical breakout.** The footprint has no orientation marking, and a reversed part puts 3.3 V on RST.
- [ ] Fix the `MDOT:MTDOT` footprint: it has a duplicate pad 24 and no pad 28.
- [ ] Settle the open assembly choices (BNO055 JP2, R6/R7, R3's footprint, HAT EEPROM), build the fab package, and order samples from OSHPark.
- [ ] Add a connector for the pack's charge signal (SBU on the V75) to mDot `PB_0` (section 10). The mDot symbol's pin numbers don't match MultiTech's guide (PB0/PB1 and PA0/PA7 are swapped), so lay out by pin name.
- [ ] Add design change 19 (a polymer bulk cap at the Witty Pi's 5 V input) and 20 (a separate mDot supply, and NRESET on GPIO25 through a MOSFET) from `WaterCam_PCB_Design_Changes_Required.md` to the schematic. Both follow the 2026-10-19 power-cut findings. `Schematic_Remaining_Work.md` #18–#20 (pcb-designs #4, #5) refine 20: feed the mDot the 5 V directly through a Schottky rather than a 3.3 V LDO, which would leave it at MultiTech's 3.3 V minimum. Update change 20 on the v6 branch to match.
- [ ] A debug UART header for the mDot (PA_9/PA_10/GND) and a series resistor on its AT TX (`Schematic_Remaining_Work.md` #19). The units are hand-wired, and these aren't practical to add by hand.

## 12. Later
- [x] Cap how long emergency mode can run: `emergency_max_hours` (default 24) with a hardware backstop. Sleeping between captures (SU-WaterCam #243) cuts its cost about fourfold.
- [ ] Emergency mode's low-battery guard (longer interval at low charge), once the charge signal (section 10) is readable.
- [ ] Lower the mDot's always-on draw, the largest item in a unit's daily energy. Measure it first; options that keep Class C, and a Class B or Class A + heartbeat variant for deployments that can accept a slower remote wake, are in `LORAWAN_CLASS_B_OPTION.md` (not planned).
- [ ] Investigate updating the segmentation model over the air, within the cellular data limits.
- [ ] Test ONNX export and inference for the recent SegFormer 3-band model.
- [ ] Watch a real command arrive over cellular. Since SU-WaterCam #144 the node polls for IP commands after every delivered uplink, so a cellular-only unit gets them after the capture. Tested on 006 with an empty queue only.
- [ ] Cache the camera's white-balance gains across boots, saving about 5 s per wake.
- [ ] Reduce Tailscale's cellular data while keeping it for remote access (parked 2026-10-07). A cellular wake used 126–474 KB, about 10 KB of it ours. Options and estimates are in SU-WaterCam `docs/CELLULAR_DATA.md`.
