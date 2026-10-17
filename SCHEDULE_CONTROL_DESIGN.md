# Remote schedule control: one unit or a whole neighbourhood

Status: **design for review, nothing implemented.** No code in any repo changes
until this is agreed.

Code read for this document: SU-WaterCam `main` at `7e17c4f` (#193), API `main`
at `60c541e` (#163), mDot-AT-firmware `main` at `2ad3ce2d` (#10), upstream
Witty-Pi-4 at `e833c8d`, and the companion design
[COMMAND_ROUTING_DESIGN.md](COMMAND_ROUTING_DESIGN.md) ("the routing design",
not yet merged). File references are repo-relative; line numbers are left out
because they move. Statements marked **Assumption** could not be checked in the
code and need a bench check, a captured payload or a unit test.

---

## Contents

1. Problem statement
2. Goals and non-goals
3. How schedules work today
4. Schedule representation
5. Wire format
6. Node: validation
7. Node: installing a schedule
8. Node: special states (mid-wake, emergency, remote debug, bench hold, clock)
9. Node: persistence, reporting, revert
10. The existing `12 92` monitoring frequency
11. Fit with the routing design (kind 05)
12. Before the routing design exists
13. Neighbourhood operation
14. LoRa capacity and collisions
15. Energy budget and guard
16. Failure modes and safety nets
17. API changes
18. Dashboard changes
19. Deploy order and compatibility
20. Testing plan
21. Rollout plan
22. Decisions (review 2026-10-17)
23. Where the code differs from what the plan assumed

---

## 1. Problem statement

A unit's wake schedule is the Witty Pi schedule file `~/wittypi/schedule.wpi`.
Today it is copied by hand at deployment (SU-WaterCam
`docs/FIELD_DEPLOYMENT_GUIDE.md` 1.13, normally
`config/wittypi/watercam_daylight_2h.wpi`) and only changes when someone logs
in to the unit. There is no way to change it from the dashboard, so:

- a neighbourhood can't be switched to more frequent wakes ahead of a storm
  and back afterwards;
- units in a neighbourhood can't be made to capture at the same moments;
- a unit whose schedule is wrong (wrong time zone, a bench schedule left on) can
  only be fixed by a site visit or a remote debug session.

The dashboard has a "Monitoring Frequency" control (1, 3, 6, 24 or 72 h, downlink
`12 92 II`). It looks like a schedule control but does not change when the unit
wakes (3.5).

A schedule is also the most dangerous thing the server can change. A bad one can
leave a unit that never wakes again (the "stranded" failure in
`docs/WITTYPI_SCHEDULING.md` §6), or one that wakes so often it runs the
battery flat. Most of this design is about making that hard to do and easy to
recover from.

## 2. Goals and non-goals

Goals

- Set a unit's schedule from the dashboard, from a small table of presets or as
  a compact custom specification, and set the same schedule on every unit of a
  neighbourhood in one action.
- Units on the same schedule wake at the same absolute times (simultaneous
  capture), with an optional per-unit offset of seconds to spread LoRa traffic.
- The node validates every schedule itself and refuses unsafe ones, whatever
  the server sent.
- Installing is atomic. A power cut at any moment leaves either the old schedule
  or the new one, and a next wake armed.
- Every existing safety net (mDot watchdog, recovery boot, shutdown-time
  next-wake check, `!`) keeps working, and the mDot watchdog is kept longer
  than the longest gap of the schedule in force.
- The node reports the schedule in force in a few bytes that fit the DR0 uplink.
- A temporary schedule reverts by itself on the node after a set time, even if
  the server is gone.
- One-step revert and "back to default".
- Fits the routing design as a non-urgent class N command, and also works
  before the routing design is implemented.
- Old nodes ignore the new command safely; new nodes work with the old API.

Non-goals

- Changing the Witty Pi firmware or the upstream Witty Pi scripts.
- Battery-aware automatic scheduling. There is no battery measurement yet
  (`docs/POWER_ANALYSIS.md`, "Battery state of charge"); this design leaves room
  for it (22, Q12).
- Changing what happens inside a wake (captures per wake, `photo_interval`).
  Those stay configuration parameters.
- Wakes more often than every 15 minutes. Emergency mode (unit stays awake) is
  the tool for that.
- Schedules computed on the server and pushed as `.wpi` text. They are too large
  for LoRa and the node couldn't check them as well as a structured spec.

## 3. How schedules work today

### 3.1 Witty Pi 4 (firmware 0x07)

From `docs/WITTYPI_SCHEDULING.md` and the upstream scripts:

- Two one-shot alarms, startup (registers 27-30) and shutdown (32-35), each
  "day of month, hour, minute, second" in BCD. There is no month: an alarm means
  the next time the RTC shows that day and time, so nothing can be armed more than
  about 28 days ahead.
- The firmware checks once a second and fires only when `now - alarm` is 0 or
  1 s. **A missed alarm never fires late.**
- A startup alarm while the Pi is on is cached and fires about 3 s after power
  is cut (reason 8), unless MISC bit 0 is set. The cache is RAM, so an ATtiny
  reset loses it.
- "Default ON" (register 17) powers the Pi only when the ATtiny itself starts
  (power applied, or a brownout reset), reason 10. It is not a periodic retry.
- The RTC holds **local** time and is ±2 ppm (manual: about 1 minute a year).

### 3.2 `runScript.sh` and the schedule file

- Syntax: `BEGIN`, `END`, then `ON`/`OFF` lines with `D H M S` durations
  (`WAIT` leaves an edge to something else). The cycle of states repeats from
  `BEGIN` until `END`.
- `BEGIN` and `END` are parsed with `date -d` in the system time zone, and slots
  are `BEGIN + n × period` in absolute seconds. So a 24 h schedule is fixed in
  UTC and moves an hour on the wall clock across a daylight-saving change (the
  daylight preset's header says so).
- It finds the ON state that ends after now and arms **shutdown = end of that
  ON** and **startup = end of the following OFF**. Booted inside an OFF state it
  treats the next ON as current, so it arms the shutdown for the end of the next
  ON and the startup for the slot after that, skipping one slot. Run with no
  arguments (as `witty_pi_4.apply_schedule()` does) it never "revises".
- An empty file clears both alarms. A file that has ended, has no ON or no OFF,
  or has no BEGIN/END arms nothing.
- `daemon.sh` runs it at every boot, after `beforeScript.sh` (ours runs
  `syncTime.sh`, in the background when the RTC had good time).

### 3.3 What our software adds

| Piece | When | What it does with the schedule |
|---|---|---|
| `tools/wait_for_wittypi_schedule.py` | `ticktalk.service` ExecStartPre | waits up to 120 s for the daemon's "Schedule next startup at:" line, so the load doesn't start before a wake is armed |
| `tools/recovery_boot.py` (`wittypi-recovery.service`) | boot, before ticktalk | on start reasons 5, 9, 10 outside [armed − 2 min, armed + 20 min], arms the first `schedule.wpi` slot at least 120 min away and powers off; leaves `/run/wittypi-recovery-wake` |
| `tools/ensure_next_wake.py` (`wittypi-next-wake.service`, ExecStop) | every shutdown, whatever caused it | if no startup is armed, it is under 120 s away, or it is later than the next slot, arms the first slot ≥ 120 s away (alarm 1 direct, alarm 2 cleared). Never moves a wake later. Skips if the recovery mark exists |
| `tools/max_awake.py` (`max-awake.timer`, 5 min) | while up | powers off after `max_awake_minutes` (60), except in a remote debug session, with the hold flag `/etc/watercam/hold-awake`, or in emergency mode |
| `tools/wittypi_control.py` `apply_emergency_schedule` | emergency on/off | on: clears the shutdown alarm. Off: `rearm_schedule()` (runScript.sh on the installed file); only if no file exists, generates one from `wittypi_*` parameters (08:00, eight 30-min slots) |
| `tools/remote_debug.py` | session start/stop | start: clears the shutdown alarm and reads it back. Stop: `rearm_schedule()` from the installed file. A reboot during a session clears it again |
| `ticktalk_main.py` `call_shutdown` | end of each capture | powers off after `shutdown_iteration_limit` captures (default 2 in `_DEFAULT_PARAMS`, 60 s apart by `photo_interval`), after the last downlink check; in emergency mode never, until `emergency_max_hours` (24) |

`recovery_boot.parse_schedule` and `next_on_start` are a Python mirror of
`runScript.sh` (WAIT states are "unusable"). `ensure_next_wake` uses them. This
design reuses them, so the node checks a new schedule with the same code that
later arms it.

A wake takes about 2 minutes and 0.12 Wh with current software (POWER_ANALYSIS,
"One wake"). The ON window (10 or 15 min in the presets) is only a maximum.

### 3.4 The mDot watchdog

`AT+WDOG=<h>` (0-168, default 20, saved in `piwdog.cfg`): if the Pi has had no
power (PA_6 low) for that many hours, the mDot presses the Witty Pi switch
(1.5 s), then retries after 1 h, doubling, at most 24 h apart. It sends
`WD:<press>,<hours>` and answers `AT+WDOG?` with
`<hours>,<hours since the Pi last had power>,<presses since>`. The server sets it
with the downlink `1B 9B HH` (`POST /devices/{id}/watchdog`), handled by the mDot
with the Pi on or off, answered by `WDOG:<h>`. The firmware README states the
rule this design enforces: the watchdog must exceed the longest scheduled gap.

### 3.5 What `12 92` (monitoring frequency) does today

Traced through the code:

- API: `encode_downlink` maps `monitoring_freq_h` (1, 3, 6, 24, 72) to its index
  in `ALLOWED_MF_HOURS` and emits `12 92 II`. The dashboard's "Monitoring
  Frequency" select sends it.
- Node: `apply_downlink_command` (`tools/transmit_ip.py`) maps the index back
  through `_MF_HOURS` and calls `set_param_fn("monitoring_frequency", hours * 60)`.
  The LoRa daemon's TLV and digit parsers route `12 92` to the same function.
- The parameter is then **only stored and reported**: in the sensor data
  (`09 39`), the IP uplink parts, `debug_status_command.py`, and the status
  dicts of `adaptive_monitoring` and friends. The update callbacks registered in
  `ticktalk_main.py` and `tools/lora_runtime_integration.py` only print a line.
  Nothing reads it to change `schedule.wpi`, an alarm, `photo_interval` or
  `shutdown_iteration_limit`.
- `emergency_workflow` would set it to 5, but that function is not wired into the
  `ttmain` graph.

**Confirmed: `12 92` changes nothing about when a unit wakes.** Section 10
recommends what to do with it. The same holds for `13 93` (emergency frequency):
it is stored and reported only, and emergency mode captures every
`photo_interval` (applied by `tools/photo_interval_watcher.py`).

## 4. Schedule representation

### 4.1 Two forms

- **(a) Preset:** an index into a table that the node and the API both carry.
- **(b) Custom:** a structured specification of a daily pattern of wakes.

Both become the same internal "spec". A preset is a named spec, not a `.wpi` file
installed byte for byte (4.4 explains why). The node generates the `.wpi` from
the spec.

### 4.2 The custom spec

| Field | Size | Range | Meaning |
|---|---|---|---|
| `first` | u16 | 0-1439 | minute of the day, **UTC**, of the first wake of the daily window |
| `interval` | u16 | 15-1440 | minutes between wake starts in the window |
| `count` | u8 | 0-96 | wakes per window; **0 = all day** (then `interval` must divide 1440) |
| `on` | u8 | 10-60 | ON window in minutes: the most a wake may last before the Witty Pi shuts the Pi down |
| `days` | u8 | see below | which days run the window |
| `offset` | u8 | 0-255 | seconds added to every wake time, for spreading a neighbourhood's LoRa traffic (13.3) |
| `revert` | u8 | 0-168 | 0 = permanent; else hours after install at which the node goes back to the previous schedule by itself |

`days`:

- `0x7F` = every day (the usual case; period 24 h).
- `0x01`-`0x7E` = a weekday mask, bit 0 Monday ... bit 6 Sunday, of the UTC day
  in which the window's **first** wake falls (period 7 days).
- `0x80 | N`, N = 2-7 = every N days (period N days), counted from the epoch
  below. For example `0x83` is the old "every 72 h" idea, done as a real schedule.

The **epoch** is fixed: 2026-01-05 00:00:00 UTC, a Monday. Every unit computes
its slots from the same instant, so units with the same spec wake at the same
absolute times. Multi-day periods have the same phase on every unit, and the
weekday mask has a fixed meaning.

Derived values, computed the same way on node and API (shared test vectors, 20.1):

- the list of wake starts over one period;
- `max_gap`: the longest time between consecutive wake starts, including the
  wrap from the end of one period to the start of the next;
- `wakes_per_day`: average over the period;
- `off_min = interval - on` for windowed schedules.

### 4.3 UTC, not local time

Decision: **the wire format is UTC. The dashboard shows local time.**

- `runScript.sh` already makes every schedule fixed in absolute time: slots are
  `BEGIN + n × period` seconds. A local-time spec would need the node to rewrite
  the file twice a year, and the slots would jump an hour on the change days.
- The sun doesn't observe daylight saving. A camera that needs light wants fixed
  solar times, which UTC gives within the seasonal drift of sunrise. The
  daylight preset already behaves this way (08:00-16:00 standard time, so
  09:00-17:00 local in summer).
- A command in UTC means the same thing on every unit, whatever its system time
  zone. The node converts the epoch plus `first` into a local `BEGIN` string with
  its own zone, and `runScript.sh` converts it back, so the result doesn't depend
  on that setting as long as the RTC and the system zone agree.
- **Choice of `BEGIN` text:** the node writes `BEGIN` as the local time of a
  January instant (epoch + `first` + `offset`). In January there is no
  daylight-saving ambiguity in the northern hemisphere. The node parses the
  string back with `recovery_boot.parse_schedule` and checks it gives the same
  UTC instant. If it doesn't (an ambiguous or skipped local time in some zone),
  it moves `BEGIN` one period later and checks again. **Assumption:** units run
  with a northern-hemisphere zone; the check covers others anyway.
- The dashboard shows each wake in the neighbourhood's time zone for the date
  shown, with the daylight-saving shift spelt out ("09:00-17:00 from 8 March").

What UTC does not fix: the **RTC keeps local time** (`system_to_rtc`). On the
first boot after a daylight-saving change, the RTC is an hour off until
`syncTime.sh` corrects it, so one wake per change lands an hour early or late.
That happens today with every schedule. Keeping the RTC in UTC would remove it,
but it touches the upstream scripts (out of scope; Q8).

### 4.4 The preset table

Presets are specs, not shipped files. The shipped `.wpi` files have a local
`BEGIN`, so their real times depend on the unit's time-zone setting. No setup
step in the field guide sets the zone, and a unit left on UTC would wake at
08:00 UTC, before dawn in the Americas, with the daylight file. **Assumption:** the units' zone
is US Eastern, which is what the preset headers assume; the node will report its
UTC offset (9.3) so this can be checked across the fleet.

Initial table (append-only; an index never changes meaning):

| Index | Name | Spec (`first` UTC, `interval`, `count`, `on`, `days`) | Equals today's file on a UTC−5 standard-time unit |
|---|---|---|---|
| 00 | Daylight, every 2 h (**default**) | 780 (13:00), 120, 5, 15, `7F` | `watercam_daylight_2h.wpi` |
| 01 | Day and evening, every 2 h | 660 (11:00), 120, 9, 15, `7F` | `watercam_on15_off1h45_offnight.wpi` |
| 02 | Hourly, all day | 0, 60, 0, 10, `7F` | `watercam_10minutes_per_hour.wpi` |
| 03 | Storm: hourly, daylight | 780 (13:00), 60, 9, 15, `7F` | — (new) |
| 04 | Storm: every 30 min, daylight | 780 (13:00), 30, 17, 15, `7F` | — (new) |

`offset` and `revert` are not part of a preset: they come with the command (5.3).
New presets need a node release and an API release with the same table. The API
only offers an index the unit's reported software version knows (19). The
shipped `.wpi` files stay for manual installs.

### 4.5 Schedule identity

`hash` = CRC-32 (zlib) of the 8 canonical bytes
`first(2) interval(2) count(1) on(1) days(1) offset(1)`. It identifies the
**wake pattern**: a preset and the equal custom spec have the same hash, and the
API can compute it without the node. `revert` is not part of it, since it says
what happens later, not when the unit wakes.

A schedule the node didn't install itself (copied by hand) is identified by the
CRC-32 of the file's bytes, reported with source "manual" (9.3), so the two can't
be confused.

### 4.6 Time zones, daylight saving and local daylight

Decided 2026-10-17:

- **Units keep a local system zone.** Units in US Eastern stay on
  `America/New_York`, so file names, capture folders and logs stay in local time
  (easier during development). Units deployed in another zone get that zone set
  at installation (setup mode). The Witty Pi RTC keeps local time, so the
  DST-hour error of 4.3 stays: one wake per change lands an hour off until
  `syncTime.sh` corrects the RTC. Running nodes in UTC (which would remove it
  without touching the Witty Pi scripts) is not adopted for now.
- **Neighbourhoods get a time zone** (an IANA name, e.g. `America/New_York`) in
  the API. The dashboard shows schedules, wakes and the DST shift in it, and
  warns when a unit's reported offset (`tzo`, 9.3) doesn't match its
  neighbourhood's offset for that date.
- **Local daylight is matched by the schedule editor (18.5)**, not by the
  node. The wire format stays UTC. A fixed schedule slides against daylight
  through the year (at mid-northern latitudes about 3 h at each end of the day
  between winter and summer), so the dashboard warns when a unit's schedule
  misses more than about 1 h of daylight at either end or wakes in darkness,
  and the operator applies an updated schedule a few times a year.
- **The built-in presets (4.4) are US Eastern schedules.** The dashboard
  offers them only for neighbourhoods whose zone they suit, or shows what they
  mean in local time and daylight before they are applied elsewhere.
- **Later:** a sunrise/sunset schedule type computed on the node (NOAA sunrise
  formula, the window rewritten once a day through the normal install, the
  neighbourhood's reference position in the command so every unit computes the
  same times). Only if seasonal re-applying becomes a chore.

## 5. Wire format

### 5.1 Code and shape

New downlink code pair **`1D 9D`** (the `NN, NN|0x80` convention). Unlike the
other codes it is **length-prefixed**:

```
1D 9D  LL  OP  <body>
│  │   │   │   └─ op-specific, LL-1 bytes
│  │   │   └───── 00 preset, 01 custom, 02 revert, 03 query
│  │   └───────── length of everything after LL
└──┴───────────── schedule command
```

The length byte is what makes it safe on old nodes (5.5). A schedule command is
always sent alone, never concatenated with other codes.

**Why `1D 9D`.** It has never been used anywhere, which matters because units on
older software keep whatever meanings their decoder had:

- `15 95` (the obvious next free pair) is **not** usable. It was a legacy LoRa
  code, removed from the node's decoder only in SU-WaterCam #186 (2026-10-15).
  Units still on a pre-#186 commit decode it with the old meaning, and the IP
  fallback work avoided it for the same reason.
- Also excluded: `1F 9F` (the routing design's wrapper), `17 97` (IP upload),
  `1B 9B` and a lone `21` (intercepted by the mDot), and every code in today's
  `DOWNLINK_CODE_LENGTHS` (`10`-`14`, `16`-`18`, `21 91`, `99 99`).
- Searched for `1D 9D` in every spelling (`1D 9D`, `1d 9d`, `1D9D`, `1d9d`) and
  as `0x1D`: no hit in the full git history of SU-WaterCam (`tools/`,
  `ticktalk_main.py`, all branches), of the API (`app/`, encoders and decoders),
  or of the mDot firmware's `CommandTerminal/`. The only `0x1D`/`0x9D` hits in
  the mDot repo are in vendored mbed-os sources.
- The old LoRa TLV parser compares channel and command **as decimal strings**
  (`f"{ch:02d}"`), so a code byte can also collide through its decimal value.
  `0x1D` is `29` and `0x9D` is `157`. Neither `'29'` nor `'157'` appears as a
  channel or command in any version of the decoder (searched the history too).

### 5.2 Ops

| Op | Body after OP | LL | Frame size |
|---|---|---|---|
| `00` preset | `PP` index, `OO` offset s, `RR` revert h | 04 | 7 B |
| `01` custom | `FF FF` first, `II II` interval, `CC` count, `NN` on, `DD` days, `OO` offset, `RR` revert | 0A | 13 B |
| `02` revert | `TT` target (00 previous, 01 baseline, 02 preset 00), `HH HH HH HH` if-hash | 06 | 9 B |
| `03` query | — | 01 | 4 B |

All integers big-endian, like the other server frames.

- **Revert targets.** *Previous* is the schedule in force before the current one.
  *Baseline* is the schedule the unit had when the feature first ran on it (the
  deployment schedule, saved once). *Preset 00* is the default, without needing
  the preset op.
- **If-hash** (compare-and-swap): the revert applies only if the current
  schedule's hash equals it. `00000000` means "whatever is current". A rollback
  of a neighbourhood (13.5) uses it, so a unit that never applied the batch isn't
  reverted to an older schedule.
- **Query** changes nothing. The node answers with its report (9.3). It is
  useful for checking a neighbourhood's real state.

### 5.3 Examples

| Command | Frame (bare) | Size |
|---|---|---|
| Default preset, permanent | `1D 9D 04 00 00 00 00` | 7 B |
| Preset 01, offset 20 s, permanent | `1D 9D 04 00 01 14 00` | 7 B |
| Custom: 13:00 UTC, every 60 min, 9 wakes, ON 15, every day, offset 0, revert after 48 h | `1D 9D 0A 01 03 0C 00 3C 09 0F 7F 00 30` | 13 B |
| Custom: every 3 days, one wake at 16:00 UTC, ON 15 | `1D 9D 0A 01 03 C0 05 A0 01 0F 83 00 00` | 13 B |
| Revert to previous if current hash is `1A2B3C4D` | `1D 9D 06 02 00 1A 2B 3C 4D` | 9 B |
| Query | `1D 9D 01 03` | 4 B |

(For the 3-day example, `interval` is 1440 with `count` 1. The period comes from
`days`.)

Inside the routing design's sequenced wrapper (kind `05`, seq `SS`):

| Command | Sequenced frame | Size |
|---|---|---|
| Default preset, seq 7 | `1F 9F 09 05 07 1D 9D 04 00 00 00 00` | 12 B |
| Custom (row 3 above), seq 8 | `1F 9F 0F 05 08 1D 9D 0A 01 03 0C 00 3C 09 0F 7F 00 30` | 18 B |
| Revert, seq 9 | `1F 9F 0B 05 09 1D 9D 06 02 00 1A 2B 3C 4D` | 14 B |
| Query, seq 10 | `1F 9F 06 05 0A 1D 9D 01 03` | 9 B |

### 5.4 Sizes against LoRa limits

- **Downlink:** Class C downlinks use RX2, US915 default DR8 (SF12/500 kHz),
  53 B maximum application payload (33 B if repeater-compatible sizes are
  enforced; FOpts MAC commands can take up to 15 B). The largest frame is 18 B
  sequenced, so it fits either way. Airtime at DR8 is about 0.37-0.41 s for 13-18 B
  (13 B LoRaWAN overhead, CR 4/5, CRC off on downlinks). **Assumption**, as in
  the routing design 11: the deployed ChirpStack region config uses the default
  RX2 DR8. To be checked in the region file; this design needs nothing more than
  that.
- **Uplink:** DR0 (SF10/125 kHz) carries 11 B. The routing design's ack
  (`01 0E`, 6 B) and the schedule report (`01 0F`, 9 B, 9.3) each fit; together
  (15 B) they need DR1 or better, otherwise they go as two packets.

### 5.5 Old nodes: traced through the parser

Traced through two decoders: **current** `main` (`7e17c4f`), and **pre-#186**
(`e92b5ee`, the parent of #186 `28e7699`, which still has the legacy codes
`15 95`, `22`, `31`, `32`, `40`-`44`). `decode` in
`tools/lora_handler_concurrent.py` gets `1D 9D 04 00 00 00 00` as the hex string
`"1D9D0400000000"`:

1. Not `"21"` (both versions).
2. `_parse_server_frame` → `tools/transmit_ip.py` `parse_downlink_frame`: code
   `1D 9D` isn't in `DOWNLINK_CODE_LENGTHS` in either version → None.
3. TLV parser (the same code in both): `ch=0x1D, cmd=0x9D, vlen=0x04`, 4 bytes
   follow, offset = length → exactly one TLV. `_apply_command_tlv` formats the
   bytes as decimal (`f"{0x1D:02d}"` = `29`, `f"{0x9D:02d}"` = `157`).
   - Current: its decimal branches are `10/90`, `11/91`, `12/92`, `13/93`,
     `99/00`, its hex branches `ch == 0x21 and cmd == 0x91` and
     `ch == 0x18 and cmd == 0x98`. None match.
   - Pre-#186: the same, plus the legacy decimal branches `15/95`, `22/00`,
     `31/00`, `32/00`, `40/00` to `44/00`. None match `29/157`.
   - Both print "Unknown TLV channel/command: 29/157". `decode` returns after a
     successful TLV parse, so the digit parser (whose branches test the first two
     hex characters, e.g. `'15'` with `'9'` in the pre-#186 version) never sees
     it. It wouldn't match there either: the first two characters are `1D`.
4. Nothing applied, no reply, in both versions.

The runtime manager's own parser (`lora_runtime_integration.process_lora_payload`,
present before #186) isn't on the downlink path: in `e92b5ee` it is called only
from the module's self-test.

The same holds for every op, because `LL` is by construction the exact rest of
the frame. The danger is a wrong `LL`. The TLV parser would then go on to read
body bytes as further TLVs, and a body containing `21 91 xx` or `18 98 xx` could
start an emergency or a debug session on an old node. Two guards:

- the API encoder computes `LL` from the body and a unit test runs every
  encodable frame (property-based: random valid specs) through a copy of the old
  `decode` and asserts that no handler runs;
- old nodes shouldn't receive the frame at all: the API sends `1D 9D` only to
  units whose reported release has the feature (19).

The mDot forwards it: it is neither exactly one byte `0x21` nor exactly three
bytes starting `1B 9B`.

Over IP, an old node's `apply_downlink_command` (current and pre-#186) gets a
part `{"code": "1D 9D", "payload_hex": "0400000000"}`. The code isn't in
`_expected_len`, so there's no length check, and the final `else` logs
"unrecognised code '1D 9D'" and skips it. In the routing design's wrapper it is
ignored as in its 10.3.

### 5.6 New nodes: parsing rules

- `parse_downlink_frame` learns one length-prefixed code: for `1D 9D` the width
  is `1 + data[i+2]`. (`DOWNLINK_CODE_LENGTHS` gets a sentinel for it; `1F 9F` is
  still not added, as the routing design requires.)
- A `1D 9D` part must be the only part of the frame, `LL` must match, `OP` must
  be known and its body exactly the right length. Otherwise: rejected (ack
  result 03 if sequenced; report reason 01) and nothing applied.
- In a sequenced frame, kind `05` may contain only `1D 9D`.

## 6. Node: validation

The node checks every schedule, preset or custom, before touching anything.
Validation is a pure function in a new `tools/schedule_spec.py`, with the same
rules, numbers and test vectors as the API's preview (17.4).

| # | Rule | Why | Reject reason |
|---|---|---|---|
| V1 | field ranges as in 4.2; `days` valid | — | 02 |
| V2 | windowed: `(count − 1) × interval + on ≤ 1440`; all-day: `1440 mod interval = 0` | the window, including its last ON, fits in a day and doesn't overlap the next day's | 03 |
| V3 | `interval − on ≥ 5` min (windowed and all-day) | the Pi must be off before the next alarm: halt, the 7 s power-cut delay, and `ensure_next_wake`'s 120 s margin, or the slot is skipped or cached | 04 |
| V4 | `on ≥ 10` min | the worst normal wake: about 2 min of cycle, plus up to about 2 min if the RTC was bad (`beforeScript.sh` then blocks on `syncTime.sh`: 30 s sleep, up to 100 s of GPS), plus a cellular fallback upload (`cellular_up` allows 90 s, Tailscale more), the last downlink check, and a recovery boot's up to 2 min wait for the daemon | 02 |
| V5 | `on ≤ max_awake_minutes` (60) | above that `max_awake` ends the wake anyway; a warning on the API, a hard limit on the node | 02 |
| V6 | `max_gap ≤ 16 h` (decided, Q4) | the default preset's overnight gap is exactly 16 h and nothing uses longer gaps. Up to 16 h the watchdog stays at its 20 h default (6.1), and the 20 h silent alert and the routing design's LoRa window need no change. The wire format keeps multi-day `days` values for later; the node rejects them for now | 05 |
| V7 | the mDot watchdog can be set to `wdog_needed` (6.1), or the unit has no mDot | otherwise a stranded unit would be pressed too early (every night) or never | 07 |
| V8 | permanent schedules: `wakes_per_day ≤ schedule_max_permanent_wakes` (default 48); temporary (`revert` 1-72 h): up to 96 | energy (15). A dense schedule is allowed only if it ends by itself | 06 |
| V9 | the generated file parses with `recovery_boot.parse_schedule`, `next_on_start(now + 120 s)` exists and is within `max_gap` of now, and the parsed slots equal the spec's | self-check: the code that will arm the wake agrees with the spec | 08 |
| V10 | preset index known to this node | append-only table | 0B |
| V11 | revert: target exists; if-hash matches (or is 0) | 5.2 | 09 / 0A |
| V12 | weekday mask (`days` 01-7E): `count ≤ 24` | keeps the generated file under about 340 states, so `runScript.sh` stays fast at boot (F26) | 02 |

Not checked on the node: whether wakes fall in darkness. That needs the site's
position and is a dashboard warning (17.4).

### 6.1 The mDot watchdog rule

The watchdog counts hours **without Pi power**. The longest such stretch on a
schedule is about `max_gap` minus one wake (about 2 min). A recovery boot can
stretch it: it defers to the first slot at least 2 h away, so the worst case is
about `max_gap + 2 h`.

```
wdog_needed = clamp(ceil(max_gap_h) + 4, 20, 168)
```

- **+4 h** covers one recovery deferral and a wake that started late.
- **Floor 20** (today's default): dense schedules keep 20 h for now rather than
  dropping to a few hours. A lower watchdog would recover a stranded unit
  sooner, but a too-low value presses the switch at night on a healthy unit.
  Q5 asks whether to go lower.
- For the default preset: `max_gap` 16 h → 20 h, today's value.
- **With V6 at 16 h (decided), `wdog_needed` is always 20 h**, today's default,
  so the first release doesn't adjust the watchdog: install steps 5 and 9 (7.1)
  are skipped, and V7 only checks that the mDot reports 20 h or more. The rest
  of this section is the design for when longer gaps are allowed.

Who sets it: **the node, through the LoRa daemon, as part of the install**,
using a new daemon socket action `{"action": "pi_watchdog", "params": {"hours":
N}}` that sends `AT+WDOG=N` and reads back `AT+WDOG?`. Why the node and not the
API's `1B 9B`:

- it is atomic with the install, so it can't be forgotten or arrive out of
  order;
- it works when the command arrived over IP;
- it covers schedules the node changes by itself (revert, rollback).

Order: **raise before installing** a sparser schedule, **lower after** a denser one
is in place. A watchdog that is too high only slows a rescue; one that is too low
causes needless presses. Neither strands a unit, but raising first means there
is never a moment with a sparse schedule and a short watchdog.

If the daemon can't set it (no answer, mDot busy): a schedule that needs a
**higher** value is rejected (reason 07). One that would only lower it is
installed, with flag bit 4 "watchdog not adjusted" set (9.3). A unit with no mDot
(IP only) has no watchdog. V7 passes with the same flag set, and the dashboard
warns that nothing would rescue a stranded unit.

The API can still read the value: the mDot's `WDOG:<h>` answer and the report's
flag. The existing `POST /devices/{id}/watchdog` stays for manual use. The
dashboard warns when a manual value is below the schedule's `wdog_needed`.

The firmware (v4.1.5-wc2610) answers `WDOG:<h>!` when the new value was set but
**not saved** to flash. The daemon treats a trailing `!` as a failure.

**Assumption:** sending `AT+WDOG` while the Class C processor runs is safe. It is
an ordinary AT command the firmware accepts at any time (`CmdPiWatchdog.cpp`), but
the daemon sends it between its own transmits. To be checked on the bench.

### 6.2 Interaction with other node parameters

| Parameter | Interaction | Rule |
|---|---|---|
| `bitmap_refresh_hours` (24) | a schedule with gaps over 24 h sends a bitmap every wake (refresh due); a dense one sends far fewer than one per capture | none needed; the dashboard shows "bitmap at least every max(gap, 24 h)" |
| `ip_fallback_after_hours` (12), `ip_fallback_min_interval_hours` (6) | with gaps over 12 h, every wake of a unit whose bitmaps LoRa holds back (DR0) qualifies for a fallback IP upload (about 90 KB of cellular each) | warn in the preview when `max_gap > ip_fallback_after_hours` on a unit that has used fallback recently |
| `max_awake_minutes` (60) | caps any wake | V5 |
| `emergency_max_hours` (24) | emergency mode is independent of the schedule; when it ends, the schedule in force (possibly a new one) is armed | 8.3 |
| `shutdown_iteration_limit`, `photo_interval` | the length of a wake, not the schedule | V4 assumes the defaults (2 captures, 60 s); a unit with more captures per wake needs a longer `on`. The API preview checks `on` against the unit's reported values when known |
| `recovery_boot.min_delay_minutes` (120) | a deferral picks the next slot of the schedule in force | covered by the +4 h in 6.1 |

Server side (17.6): `ALERT_SILENT_HOURS` (20) alerts every night on a unit with a
gap over 20 h, and the routing design's LoRa window has a 48 h ceiling. Both
become schedule-aware.

## 7. Node: installing a schedule

New module `tools/schedule_control.py`, called by whichever process received the
command: the LoRa daemon (`decode`, LoRa path) or ticktalk
(`apply_downlink_command`, IP path). Both go through one function,
`install(spec, source, seq=None)`, under an exclusive `fcntl` lock
(`data/schedule.lock`) that `ensure_next_wake.py` also takes (with a 10 s
timeout), so a shutdown never interleaves alarm writes with an install.

In the LoRa daemon, `decode` runs while the listener holds `transmit_lock`, so the
install runs on a helper thread (as the debug reply does), and the ack and report
are sent once it finishes. An install takes under about 3 s, including the
watchdog round trip. **Assumption:** to be timed on the bench.

### 7.1 Steps

```
 1. parse + validate (6)                                  → reject: report reason, ack 03
 2. same hash as the schedule in force and same revert?   → no-op: ack 00 (01 if same seq), report
 3. generate the .wpi text; self-check (V9)
 4. state.json: {"phase": "installing", new, old}         (atomic write, tools/config_io.py)
 5. wdog_needed > current?  raise it now (6.1; skipped while V6 ≤ 16 h)           → fail: reject 07, phase back to "idle"
 6. backup: schedule.wpi → data/schedule/previous.wpi (+ meta)
           first run only: → data/schedule/baseline.wpi
 7. write ~/wittypi/schedule.wpi.tmp, fsync, rename over schedule.wpi, fsync the directory
 8. arm (7.2), read the alarm back, check with ensure_next_wake.decide()
                                                          → fail: restore previous.wpi (rename), re-arm it,
                                                            reject 08
 9. wdog_needed < current?  lower it now (skipped while V6 ≤ 16 h)
10. state.json: {"phase": "idle", current = new, previous = old, installed_at,
                 revert_at, probation: 3}
11. log to wittyPi.log (7.3); ack 00; report (9.3); status JSON as today
```

The seq (routing design) is committed after step 10, as that design says
("commit after applying").

### 7.2 Arming: not `runScript.sh`

Install arms **only the startup alarm**, to the first slot of the new schedule
at least 120 s from now (`recovery_boot.next_on_start(new, now + 120)`). It writes
registers 27-30 the way `recovery_boot.arm_startup` does and reads them back. It
**does not set or clear the shutdown alarm** (a new `clear_shutdown=False`
argument to `arm_startup`, which clears alarm 2 today).

Why not `rearm_schedule()` (`runScript.sh`):

- Mid-wake in an OFF state of the new schedule, `runScript.sh` treats the next ON
  as current. It arms a shutdown for the end of the **next** ON and a startup for
  the slot **after** it, skipping a slot (`ensure_next_wake` would repair the
  startup at shutdown, but only if it runs).
- It **sets a shutdown alarm**, which would cut power in the middle of an
  emergency or a remote debug session (both clear that alarm on purpose), and on a
  bench unit kept up with the hold flag. The field guide already says not to run
  `runScript.sh` on the bench.
- Its output is parsed by scraping the second and third lines
  (`witty_pi_4.apply_schedule`), and it can take seconds on a long file.

Leaving the shutdown alarm as it is keeps the current wake's backstop (the end of
the old ON window). The Pi halts itself long before that. At the next boot,
`daemon.sh` runs `runScript.sh` on the new file as usual.

`ensure_next_wake` at the same wake's shutdown then finds the startup armed at
the next slot and leaves it ("next slot or earlier"). If the install armed
nothing (crash between steps 7 and 8), `ensure_next_wake` arms the new file's next
slot, or leaves an earlier armed one, which costs at most one wake at the old
time.

### 7.3 Logging

Two lines in `~/wittypi/wittyPi.log`, in the daemon's format:

```
[2026-10-20 14:02:11] Schedule install: preset 00 (hash 1A2B3C4D), previous 5E6F7081, revert none, wdog 20 h.
[2026-10-20 14:02:11] Schedule next startup at:  2026-10-20 16:00:00
```

The second line uses the same wording as `runScript.sh` (with two spaces, as
`ensure_next_wake` and `recovery_boot` write it). This matters:
`recovery_boot.previously_armed_startup` finds the last such line, so a boot that
browns out at the new wake time is still recognised as a scheduled wake, not an
outage.

### 7.4 Power loss during install

| Lost after step | On disk | Armed | Next boot |
|---|---|---|---|
| 1-4 | old file | old next slot (armed at this boot) | old schedule; reconcile: phase "installing", file = old → report "install interrupted", reason 08 |
| 5 | old file; watchdog raised | old | as above; the watchdog stays high (harmless) until the next install or reconcile lowers it |
| 6 | old file + backup | old | as above |
| 7 (before rename) | old file + a `.tmp` | old | the `.tmp` is deleted; as above |
| 7 (after rename) | **new** file | old next slot | wakes at the old slot (or the new one, if `ensure_next_wake` ran at shutdown); `runScript.sh` arms from the new file. Reconcile: file = new → phase "idle", current = new, report it |
| 8-10 | new file | new slot | reconcile finishes step 9-10 |

`rename` within one filesystem is atomic, and `fsync` before and after makes it
durable across a power cut. **Assumption:** `~/wittypi` and `data/` are on the
same ext4 root filesystem (stock Raspberry Pi OS layout), so the temporary file
is in the target directory, not `data/`.

**Reconcile** runs at boot, in the same unit as recovery boot or right after it
(a new `schedule_control.py reconcile`, ordered before ticktalk). It compares the
file's CRC with `state.json` and finishes or abandons an interrupted install. It
then checks for a due revert (9.4) and for a manual edit (9.3). It never powers
off and never arms anything, because `runScript.sh` has just armed from the file.

### 7.5 File ownership

**Assumption:** `~/wittypi/schedule.wpi` is writable by user `pi`. The Witty Pi
installer runs as root and the guide copies the file as `pi`, so it depends on how
the unit was set up. If it is root-owned, the install goes through a fixed
`doas` rule (like the shutdown command), running
`schedule_control.py install-staged` on a spec staged under `data/`. To be
checked on each unit with `ls -l` before the first remote install.

## 8. Node: special states

### 8.1 Command during a normal wake

The usual case: held commands are released on the wake's first uplink (routing
design 7.3). The install finishes within seconds. The rest of the wake carries
on unchanged, with the same captures and the usual shutdown, and the next wake
follows the new schedule. No reboot.

If the command arrives in the last seconds before halt, the install either
finishes before the lock is taken by `ensure_next_wake`, or `ensure_next_wake`
goes first and the install then runs into a stopping system. Then: no ack, the
command stays held, the server resends at the next wake, and the node reports
duplicate or no-op if the first copy did land.

### 8.2 Command arrives on a boot between slots

For example a `!` remote start, a watchdog press or a button press: the same as
8.1. `runScript.sh` at this boot treated the next ON as current; the install
re-arms the startup to the new schedule's next slot, which also repairs the skip
described in 3.2.

### 8.3 Emergency mode

Emergency mode keeps the unit awake and has cleared the shutdown alarm
(`apply_emergency_schedule(True)`). Rule: **emergency overrides; the new schedule
becomes the one restored.**

- Install steps 1-7 and 9-10 run as usual. Step 8 (arming) is **skipped**; flag
  bit 1 "armed when emergency or session ends" is set.
- Both ways emergency ends already re-arm from the installed file:
  `apply_emergency_schedule(False)` calls `rearm_schedule()`, both from the LoRa
  daemon's emergency callback and from `call_shutdown`'s time limit. So the new
  schedule is armed then, with no change to that code.
- `apply_emergency_schedule(False)` has a fallback that generates a schedule
  from the `wittypi_*` parameters (08:00, eight 30-min slots) when no file
  exists. It is changed to install preset 00 through `schedule_control` instead,
  so there is one generator (23).
- A schedule change doesn't end emergency mode and doesn't change
  `emergency_max_hours`.

### 8.4 Remote debug session

A session clears the shutdown alarm and stops ticktalk. `stop()` calls
`rearm_schedule()` from the installed file. Rule: the same as emergency: install,
don't arm, flag bit 1, and the session's stop arms the new schedule. A reboot
during the session re-establishes it (`_establish` clears the shutdown alarm
again), and the reboot's `runScript.sh` has already armed from the new file.

An operator logged in during a session can also install a schedule by hand. The
node then reports source "manual" (9.3), and the dashboard shows that it differs
from the desired one.

### 8.5 Bench units (hold flag)

With `/etc/watercam/hold-awake` present, the install behaves as in a normal
wake. Because it never sets the shutdown alarm (7.2), it can't power a held
bench unit off. That is the point of not using `runScript.sh`.

### 8.6 Unverified clock

`tools/capture_time.py` marks a boot "clock unverified" when neither network nor
GPS nor the server set the time (`/run/watercam-time-source` missing; uplink
`01 0D`). The schedule's wake times are only as right as the RTC, which is
already true of any schedule. A new schedule doesn't make a wrong clock worse,
but its "next wake" arithmetic (V9, 7.2) uses the RTC.

Rule: **install anyway**, set flag bit 2 "clock unverified at install", and let
the API decide. The API shows it and, for a neighbourhood batch, counts the unit
as "applied, timing unverified". Refusing would leave a unit with a wrong
schedule as well as a wrong clock. The server's time correction
(`transmit_ip.py`, `watercam-set-clock`) is the fix for the clock, and it is
separate.

## 9. Node: persistence, reporting, revert

### 9.1 What is stored

`data/schedule/state.json` (atomic, under the lock):

```json
{"phase": "idle",
 "current":  {"src": "preset", "preset": 0, "spec": "030c0078050f7f00", "hash": "1a2b3c4d",
              "installed_at": 1792504931, "revert_at": null, "seq": 7},
 "previous": {"src": "manual", "hash": "5e6f7081", "file": "previous.wpi"},
 "baseline": {"src": "manual", "hash": "5e6f7081", "file": "baseline.wpi"},
 "probation": 3, "wdog_set": 20, "last_reject": {"reason": 0, "at": null},
 "report_enabled": true}
```

plus `previous.wpi` and `baseline.wpi`. All survive reboots. `schedule.wpi`
itself stays the single source of truth for the Witty Pi and for
`ensure_next_wake`/`recovery_boot`. `state.json` only describes it.

A missing or corrupt `state.json` means "unknown": reconcile rebuilds it from
the file (source "manual", file CRC) and reports that. It never changes the
schedule because of it.

### 9.2 Probation and automatic rollback

A unit can't see that it failed to wake, but at the next boot it can see how
long it was off. After an install, the new schedule is **on probation** for 3
scheduled wakes (boots with start reason 1, "scheduled startup", or a recovered
dip inside the window).

At each boot during probation, reconcile computes the off-time: now minus the
last shutdown time (the shutdown mark in `wittyPi.log`, also used by
`rtc-early-clock`). If the off-time is longer than the probation schedule's
`max_gap` + 1 h, the schedule missed at least one wake, whatever then woke the
unit (mDot watchdog press, `!`, button, power restore). Reconcile **rolls back
to the previous schedule** before ticktalk starts: steps 6-10 with source
"rollback", flag bit 3. The next report tells the server.

Why not the mDot's press count (the first draft): the firmware resets
`presses` the moment the Pi has power (`PiWatchdog.h`, `pi_watchdog_step`), so
`AT+WDOG?` at boot always reads 0 presses. The off-time works without a
firmware change and also catches rescues that weren't watchdog presses.

The server adds its own check: an alert when a unit misses its first expected
wake on a new schedule (±10 min, 17.6).

### 9.3 Reporting the schedule in force

**LoRa, compact:** new uplink TLV channel **`01 0F` schedule report**, 7 value
bytes, 9 B in all (fits DR0):

```
01 0F  SC  HH HH HH HH  FL  RS
       │   │            │   └─ last reject reason (6), 00 none
       │   │            └───── flags: 0 revert pending, 1 armed when emergency/session ends,
       │   │                   2 clock unverified at install, 3 rolled back (probation or revert),
       │   │                   4 watchdog not adjusted, 5 on probation
       │   └────────────────── hash (4.5)
       └────────────────────── source: 00-3F preset index, 40 custom, 80 manual (file CRC),
                               C0 no schedule.wpi, C1 file unusable (parse_schedule says no)
```

Example: default preset, on probation, no reject: `01 0F 00 1A 2B 3C 4D 20 00`.

When it is sent:

- after every schedule command (with the routing ack, or alone before routing
  exists), as its own packet at DR0 or together with the ack at DR1 and above;
- after reconcile changed something (rollback, revert, manual edit detected,
  install finished after a power cut);
- once every `lora_full_send_hours` (24) with the full sensor send, through
  `tools/sensor_changes.py` as an exact field.

It is sent only when `report_enabled` is set, which happens on the first `1D 9D`
the node receives. An old API never sends one, so it never gets a channel it
can't decode. **Assumption,** as for the routing design's `01 0E`: the API's
TLV detector (`app/decoders.py`) stops at an unknown pair, so an old API sees a
packet that starts with `01 0F` as undecodable, not as corrupt sensor data. That
is moot if it is never sent to an old API, but to be pinned by a unit test.

**Status JSON** (`collect_compact_status`): three fields appended at the end
(first to be trimmed): `"sc":"1a2b3c4d"` (hash), `"nw":1792519200` (the armed
startup, UTC epoch) and `"tzo":-300` (the node's current UTC offset in minutes, so
a unit with a wrong zone shows up, 4.4). **Assumption:** the
API stores unknown status keys without error. `DEBUG_FIELD_LABELS` gains labels.

**IP:** the uplink JSON gains `"schedule": {"src", "hash", "flags", "reason",
"next_wake"}`. **Assumption:** the API's IP uplink model ignores extra keys (to be
checked: it must not use `extra="forbid"`).

`next_wake` is in the IP report and the status JSON (`"nw"`) only, not in `01 0F`,
which keeps it within 11 B. The API can compute it from the hash anyway.

### 9.4 Temporary schedules and revert

- `revert` hours > 0: `revert_at = installed_at + revert × 3600` (node RTC).
- Checked in two places: reconcile at boot, and `ensure_next_wake` at shutdown,
  before it decides on the next wake (under the lock). If due, the node
  reinstalls `previous` (steps 6-10, source "revert", flag bit 3), so the wake it
  arms is already the restored schedule's.
- After a revert, `previous` becomes the temporary one, so "revert to previous"
  undoes a revert.
- A temporary schedule that replaces another temporary one: `previous` is the
  last **permanent** one, so a chain of storm schedules always ends on the normal
  one. Q6.
- A wrong RTC moves the revert by the same error as everything else, and the
  `revert` field is capped at 168 h.

### 9.5 Manual edits

If reconcile finds `schedule.wpi` whose CRC matches neither `current` nor its
generated text, someone changed it by hand. It records source "manual" with the
file's CRC, keeps the old `current` as `previous`, reports it, and **changes
nothing**: an operator on site may have done it on purpose. The dashboard shows
the drift (18.2).

## 10. The existing `12 92` monitoring frequency

Recommendation: **retire it as a control, don't map it onto schedules.**

- Mapping 1/3/6/24/72 h onto presets would make old scripts, the old dashboard
  and any queued `12 92` suddenly change real wake times on new nodes. None of the
  five values expresses a daylight window, so a mapping would also have to invent
  one.
- Repurposing the code would give old and new nodes different meanings for the
  same bytes.
- Code space is cheap: `1D 9D` is a clean new code.

Steps:

1. Dashboard: remove the "Monitoring Frequency" select from the configuration
   card. The schedule card (18) replaces it.
2. API: `DownlinkCmd.monitoring_freq_h` keeps being accepted for one release and
   returns a `warning` ("has no effect on wake times; use /schedule"), then
   returns 422 with that message.
3. Node: keeps accepting `12 92` (it is harmless and an old API may send it) and
   logs "12 92 is deprecated and changes nothing; the wake schedule is set by
   1D 9D". `monitoring_frequency` stays a stored, reported number.
4. The dashboard labels the reported `sr_monitoring_frequency` as legacy, or
   stops showing it.

The same applies to `13 93` emergency frequency, which also changes nothing
(3.5). It is out of scope here; Q10.

## 11. Fit with the routing design (kind 05)

| Item | Value |
|---|---|
| Class | **N**, non-urgent: held until the unit is heard, never a `!` wake |
| Kind | **`05` schedule**, its own seq space (`next_seq[device][5]`); the node's `data/command_seq.json` gains `"5"` |
| Legacy frame inside the wrapper | `1D 9D LL OP ...` (5). Reserved alongside the routing design's `1F 9F` and `01 0E` (its 10.3, 10.4, Q14); `1D 9D` is not `15 95`, which pre-#186 nodes still decode (5.1) |
| Ack | `01 0E 05 SS RR EM`, results as in the routing design 10.4: 00 applied (including "same hash, nothing to do"), 01 duplicate seq, 02 stale, 03 rejected (the reason is in the `01 0F` report that follows) |
| Confirmation for the ledger | ack with this seq (any result except 03); or, for a node without seq, a report or status `sc` equal to the desired hash |
| Expiry | permanent: **7 days** (as configuration). Temporary: **min(7 days, revert hours)**, since after that the temporary schedule would already be over. Query: 1 day |
| Superseding | a new schedule command for a device supersedes every non-final one (held, `sending` after its attempt, `ip_queued` not yet polled). **No merging**: a schedule is replaced whole |
| Release order on one uplink | emergency, remote debug, configuration, IP upload, **schedule last**: the least urgent, and the longest frame |
| Release spacing | 14.4: at most one schedule downlink per gateway every 3 s, and a neighbourhood batch released over several uplinks |
| Wake budget | not applicable (class N never wakes) |
| Fallback to IP | as class N in the routing design 8: S1 after 3 unconfirmed attempts or LoRa inactivity; S2 never; S3/S5 directly |
| Dedupe | seq first; then content: an identical spec is a no-op whatever the seq |

Two points where a schedule needs more than other class N commands:

- **The capability windows (routing design 4.3)** must widen *before* a unit
  goes onto a sparser schedule, or the unit counts as "not LoRa-active" overnight
  and its next commands get routed to IP. Once a unit reports a hash, the router
  uses the schedule's own `max_gap`:
  `W = clamp(1.5 × max_gap, max(3 × interval, 6 h), max_gap + 12 h)`. That replaces
  the 48 h ceiling for units with a known schedule. Until three days of history
  exist it uses that instead of the 26 h default, and when a schedule command is
  confirmed, the wake history is cleared so the old pattern doesn't distort the
  median.
- **The routing design's failure mode 2 and Q6 assume `monitoring_freq_h = 72`
  makes 72 h gaps.** It doesn't (3.5). With this design, long gaps come only from a
  `days = 0x8N` schedule, and the router knows them from the report.

## 12. Before the routing design exists

The schedule feature can ship first. It needs:

1. **A hold:** the routing design's `on_lora_uplink` hook (its 7.2), reduced to
   one job: if the device has a pending schedule command and this uplink came from
   the Pi, send it after a delay (14.4) on a background thread. Pending state
   lives in the `device_schedules` table (17.1), not a general ledger.
2. **Bare frames** (`1D 9D ...`), no wrapper, no seq.
3. **Confirmation by content:** the `01 0F` report (or `sc` in the status JSON)
   with the desired hash. The node sends the report after every `1D 9D`, so DR0
   units can confirm (the report is 9 B).
4. **Ordering without seq:** the API has at most one schedule command in flight
   per device and only ever sends the **current** desired spec. A copy that
   arrives late or twice is either the current spec (a no-op on the node, since
   same hash) or was superseded before it was sent. The one gap is an IP queue row
   picked up after a newer LoRa delivery. So until routing exists: schedule
   commands go to the IP queue **only** for units with `transport_type == "ip"`,
   and superseding a schedule **deletes** its undelivered queue row in the same
   transaction.
5. **Retries:** resend on each later wake's first Pi uplink, at most 3 attempts
   per wake, until confirmed, rejected or expired (7 days).
6. **No `!` wakes.** If an operator wants it faster, they press "Power On"
   themselves, and the held command goes out on the resulting uplink.

When routing arrives, the `device_schedules` pending state becomes a ledger
row of kind `schedule`, frames for seq-capable nodes get the wrapper, and the
reduced hook is removed. The node accepts bare `1D 9D` forever (manual tools, old
API), without seq checks, as the routing design does for other legacy frames.

## 13. Neighbourhood operation

### 13.1 Fan-out

`POST /neighborhoods/{id}/schedule` creates a **batch**: one desired schedule for
the neighbourhood. For each `watercam` device in it (as `gauge_scheduler` selects
them: `neighborhood_id`, `sensor_type == "watercam"`), it creates a per-device
desired schedule, with the batch's spec and the device's own `offset` (13.3).
Each device then goes through the normal per-device path (11 or 12). The batch
itself sends nothing.

Devices that can't take it are listed, not silently skipped:

- the release is too old (no `1D 9D`): "update node software first";
- no LoRa EUI and not IP-active;
- a manual schedule on the unit (9.5): included only if the operator ticks
  "overwrite manual schedules";
- flagged as bench (`ALERT_SILENT_IGNORE` or a new device flag): included only if
  ticked.

### 13.2 Simultaneity

Units on the same spec arm the same absolute instants (fixed epoch, UTC spec,
4.2-4.3). How close their captures really are:

| Source of skew | Size | Basis |
|---|---|---|
| RTC drift between syncs | ≤ 0.2 s/day (±2 ppm) | Witty Pi manual |
| Clock setting at boot | about 1 s (HTTP `Date` has 1 s resolution; GPS similar) | `syncTime.sh`, field guide |
| Alarm resolution | 0-1 s (fires when `now − alarm` is 0 or 1) | WITTYPI_SCHEDULING §1 |
| Boot to first capture | 11 s boot, about 15 s to the armed alarm, then sensors; varies by a few seconds | POWER_ANALYSIS. **Assumption:** spread a few seconds; to be measured (20.4, B6) |
| A power-on dip | +20-30 s on the affected unit (the ATtiny restarts, waits Default ON's delay) | WITTYPI_SCHEDULING §3; seen on 30-35 % of a bench unit's wakes |
| Unit without any time source | grows ~1 min/year, plus an hour after a DST change until corrected | field guide "How the unit keeps time" |

So: typically within about 5 s, and within about 30 s when a unit dips. A flood
changes over minutes, so that is simultaneous enough. A unit flagged clock
unverified is shown as "timing unverified" in the batch. Its captures may be at
the right interval but at the wrong times.

### 13.3 Spreading LoRa traffic: `offset`

Simultaneous wakes mean simultaneous uplinks (14). The cheap fix is to move the
**wake** of each unit by a few seconds, not to delay its transmissions:

- Delaying transmits inside a wake keeps the Pi up longer (about 2.5 W × the
  delay, +35 % for 60 s on a 0.12 Wh wake).
- Moving the wake costs nothing. The Witty Pi alarms are to the second, and
  `offset` (0-255 s) is part of the spec.

For a batch the API assigns `offset_i = round(i × spread / N)` over the units
sorted by device id, so they are evenly spaced. `spread` is a batch option:
0 (exactly simultaneous), or by default `min(120, 10 × N)` seconds. Each unit's
spec, and so its hash, differs only in `offset`. The batch view groups them as
one schedule. A unit added to the neighbourhood later gets the next free offset.

### 13.4 Partial application

A batch is complete when every included device is `confirmed`, `rejected` or
`expired`. Until then the dashboard shows per unit (18.3): desired, reported,
state (held since, sent N times, confirmed at, rejected with reason, expired),
last heard, next expected wake, watchdog value, offset.

A neighbourhood **can** run mixed schedules for a while, and that is shown, not
hidden. A batch never blocks on one unit. Expiry ends the attempt and raises a
dashboard notice. The unit's desired schedule then stays, marked "not
delivered", until the operator re-sends or changes it (no automatic resend
after expiry; 16, F16).

**Canary rollout** (batch option, default on for permanent schedules): stage 1
sends to one unit (the operator's choice, or the one with the best link). Stage 2
starts when that unit has confirmed **and** has been heard at its first wake on
the new schedule (±10 min of the expected time). Temporary storm schedules default
to "all at once", because the point is speed, and they revert by themselves.

### 13.5 Rollback of a neighbourhood

`POST /neighborhoods/{id}/schedule/rollback` (with the batch id):

- devices whose batch command is still held or IP-queued: the command is
  cancelled; their desired schedule goes back to the pre-batch one;
- devices that confirmed: a revert command, `1D 9D 06 02 00 <batch hash>`. The
  if-hash makes it a no-op (reject 0A, reported) on a unit that has since been
  changed to something else;
- devices that rejected or expired: nothing to undo.

The API records each device's reported schedule before the batch, so the
dashboard shows what "rolled back" means per unit.

## 14. LoRa capacity and collisions

### 14.1 Uplink airtime

US915, 125 kHz, CR 4/5, 8-symbol preamble, explicit header, CRC on, 13 B LoRaWAN
overhead:

| Data rate | Max payload | 11 B | 40 B | 230 B (bitmap) |
|---|---|---|---|---|
| DR0 SF10 | 11 B | 0.37 s | — | — |
| DR1 SF9 | 53 B | 0.21 s | 0.33 s | — |
| DR2 SF8 | 125 B | 0.11 s | 0.19 s | — |
| DR3 SF7 | 242 B | 0.06 s | 0.10 s | **0.38 s** |

A bitmap (about 230 B, `tools/bitmap_policy.py`) only fits at DR3, at 0.38 s, close
to the 400 ms dwell limit. Below DR3 bitmaps wait in store-and-forward
(`tools/ip_fallback.py`). **Assumption:** units run ADR and are mostly at DR3 when
they send bitmaps. The routing design's DR recording (its 11.1, 13.8) will show
the real mix.

### 14.2 Collisions when a neighbourhood wakes at once

The gateway listens on sub-band 1: 8 × 125 kHz channels (plus one 500 kHz channel)
(`tools/mdot_subband.py`). Each uplink picks a channel at random. Uplinks are
**unconfirmed**, and a bitmap counts as "sent" once the mDot sent it, so a lost
bitmap is not retried until the next change or the 24 h refresh.

Estimate: pure ALOHA per channel, each unit sends `k` = 2 packets of 0.38 s (both
captures send a bitmap: the worst case, as in emergency mode or while water
changes), spread uniformly over a window `W`, all at the same SF (worst case: no
SF orthogonality and no capture effect). Probability that a given packet
collides:

| Units N | W = 5 s | W = 30 s | W = 120 s |
|---|---|---|---|
| 5 | 16 % | 3 % | 0.7 % |
| 10 | 30 % | 6 % | 1.5 % |
| 20 | 52 % | 12 % | 3 % |
| 40 | 78 % | 22 % | 6 % |

The natural spread of a wake's uplinks across units (boot variation, segmentation,
dips) is perhaps 20-40 s (**assumption**, measured in B6). So without an offset,
10 units lose about one bitmap in 16-20, and 20 units about one in 8. An offset
spread of 120 s brings 20 units to about 3 %. Sensor packets are shorter at DR3
(about 0.1 s) and collide less. At DR0 they are 0.37 s each, like a bitmap.

Gateway capacity is not the limit: an 8-channel concentrator demodulates up to 8
packets at once, one per channel. The limit is two packets on the same channel at
the same time.

### 14.3 Downlink contention

The gateway is **half-duplex**: while it sends a Class C downlink (about 0.4 s at
DR8 for a schedule frame) it receives nothing on any channel. If a batch's
commands were released 5 s after each unit's first uplink (the routing design's
`RELEASE_DELAY_S`), 20 units would put about 8 s of deafness into the busiest 30 s
of the wake: about a quarter of that wake's uplinks lost.

### 14.4 Mitigations

1. **`offset` spread** (13.3) for batches: spreads uplinks and so also releases.
2. **Gateway pacing in the router** (new, applies to all held releases): at most
   one downlink in flight per gateway, and **schedule** releases at least 3 s
   apart and not within the first 30 s after any Pi uplink from another unit of
   the same neighbourhood (a "busy" signal). The release delay for schedules is
   randomised in 20-60 s after the uplink, still well before shutdown (the wake's
   second capture is 60 s after the first).
3. **Batch release cap:** at most 5 schedule releases per neighbourhood per wake
   slot. The rest go at the next slot. A non-urgent command can wait 2 h, and it
   doubles as a gentle rollout.
4. **Acks and reports** go out right after the install, so they follow the
   (paced) downlinks and are spread with them.
5. Emergency (class U) is not paced by any of this.

## 15. Energy budget and guard

### 15.1 Model

From `docs/POWER_ANALYSIS.md`:

- a clean wake ≈ **0.12 Wh** (0.11-0.13, partly measured);
- off-state ≈ 0.05-0.1 W, **1.2-2.4 Wh/day** (estimate, unmeasured);
- a wake that runs to the end of its ON window costs about `on/60 × 2.3 W`
  (0.58 Wh for 15 min). That is the pessimistic case;
- battery 48 Wh usable; harvest about 20-30 Wh on a clear day and 3-6 Wh on an
  overcast one (estimates, summer).

```
daily_low  = wakes_per_day × 0.12 + 1.2      (Wh)
daily_high = wakes_per_day × 0.12 + 2.4
daily_pess = wakes_per_day × on/60 × 2.3 + 2.4
autonomy   = 48 / max(daily_high − 3, 0.1)   days on overcast days (3 Wh harvest)
```

| Schedule | Wakes/day | Estimate (Wh/day) | Pessimistic |
|---|---|---|---|
| Preset 00, daylight every 2 h | 5 | 1.8-3.0 | 5.3 |
| Preset 01, every 2 h, 9 wakes | 9 | 2.3-3.5 | 7.6 |
| Hourly, daylight (9 wakes) | 9 | 2.3-3.5 | 7.6 |
| Every 30 min, daylight (17) | 17 | 3.2-4.4 | 12.2 |
| Preset 02, hourly all day (ON 10) | 24 | 4.1-5.3 | 11.6 |
| Every 15 min, daylight (33, ON 10) | 33 | 5.2-6.4 | 15.0 |
| Every 15 min, all day (96, ON 10) | 96 | 12.7-13.9 | 39.2 |
| One wake every 3 days | 0.33 | 1.2-2.4 | 2.5 |

For comparison: one unit used 12-15 Wh/day in the field on older software, when
few wakes ended cleanly. Emergency mode is 70-110 Wh/day.

### 15.2 Guard

| Level | Condition (`daily_high`) | API | Node |
|---|---|---|---|
| Green | ≤ 4 Wh/day | allowed | allowed |
| Amber | 4-8 Wh/day | warning, operator confirms ("about N days of autonomy without sun") | allowed if `wakes_per_day ≤ 48` |
| Red | > 8 Wh/day | allowed **only** as temporary (`revert` 1-72 h), confirmed | V8: permanent > 48 wakes/day rejected; temporary ≤ 96 |

The API's levels are advice that the operator can confirm past. The node's V8 is
a hard limit that the server can't override (`schedule_max_permanent_wakes` is a
local runtime parameter, changed only on the unit). So a server bug, a stolen
dashboard session or a typo can't put a unit permanently on a draining schedule.

Once there is a real battery reading (the planned V50 D+ via the mDot ADC), the
guard can use it: refuse amber and red schedules below some charge, and revert
temporary ones early (Q12).

## 16. Failure modes and safety nets

### 16.1 The safety nets, and what each covers

| Net | Acts when | Covers | Doesn't cover |
|---|---|---|---|
| Node validation (6) | every command | malformed, unsafe, unfittable schedules | a valid schedule with a wrong clock |
| Self-check V9 | every install | generator bugs: the file must parse with the arming code and give the spec's slots | bugs shared by generator and parser |
| Atomic install + reconcile (7.4) | power loss | half-written files, lost state | — |
| `ensure_next_wake` (every shutdown) | no startup armed, armed too soon or past, or later than the next slot | an install that died between rename and arming; slots skipped by `runScript.sh` | a halt that skips ExecStop (power cut, brownout) |
| `wait_for_wittypi_schedule` | boot | load starting before a wake is armed | — |
| `recovery_boot` | reasons 5/9/10 outside the window | power returning after an outage; arms a slot of the schedule in force | it can't tell a bad schedule from an outage |
| mDot watchdog (`AT+WDOG`, kept ≥ `wdog_needed`) | no Pi power for WDOG hours | **stranding**, whatever caused it; then retries 1 h, 2 h, ... ≤ 24 h | the mDot without power; a unit without an mDot |
| Probation rollback (9.2) | a watchdog-rescued boot within 3 wakes of an install | a new schedule that stranded the unit: back to the previous one, automatically | a schedule that wakes but at useless times |
| Temporary `revert` (9.4) | `revert_at` passed | dense storm schedules left on | — |
| `max_awake` (60 min) | wake too long | hung wakes, whatever the schedule | emergency, debug sessions, hold flag (by design) |
| Default ON | ATtiny power-up or brownout | power returning | **not** a stranded unit with power present |
| `!` remote start | operator, LoRa | a unit that is off with a working mDot and link | no LoRa |
| Server: missed first wake (17.6) | no Pi uplink within expected first wake + 30 min after a confirmed change | tells the operator early, who can `!` and revert | — |
| Button on site | always | everything | needs a visit |

### 16.2 Failure modes

| # | Failure | Consequence | Mitigation |
|---|---|---|---|
| F1 | **A schedule that never wakes** (no slot, END passed, BEGIN far ahead, WAIT state) | stranded until the watchdog | can't be generated: fixed epoch in the past, END 2037-12-31, no WAIT; V9 requires a slot within `max_gap`; watchdog; probation rollback |
| F2 | **A slot armed too close** (< 2 min) | the Pi isn't off in time: the alarm is cached (reason 8) or missed | V3 (OFF ≥ 5 min); arming uses `now + 120 s`, as `ensure_next_wake` does |
| F3 | **Arming > 28 days ahead** (no month field) | the alarm fires on the wrong month's day | V6 caps gaps at 16 h |
| F4 | **Too frequent / always on** | battery drain | OFF ≥ 5 min and interval ≥ 15 min (can't express always on); energy guard 15.2; node hard limit V8; temporary revert |
| F5 | **ON window too short** | the Witty Pi shuts the Pi down mid-cycle; data lost, wakes continue | V4 (≥ 10 min); preview warns if below the unit's reported cycle length |
| F6 | **Watchdog shorter than the gap** | a healthy unit is pressed during its gap (an extra night wake, energy) | the node raises it first (6.1); dashboard warns about manual values |
| F7 | **Watchdog much longer than needed** | slow rescue of a stranded unit | the node lowers it after a denser schedule |
| F8 | **RTC lost or wrong** | wakes at wrong real times (true of any schedule); `revert` and probation counts shifted | flag bit 2; `01 0D`; server clock correction; the dashboard compares uplink receive times with expected wakes (17.6) and flags a skew over 10 min |
| F9 | **Wrong node time zone** | presets and UTC specs are still right (4.3); only shipped `.wpi` files used by hand are off | `tzo` in the status (9.3) |
| F10 | **DST change** | one wake an hour off at the first boot after the change, until `syncTime.sh` (existing) | UTC spec avoids the twice-yearly file change; Q8 for an RTC in UTC |
| F11 | **Power loss during install** | old or new schedule, never neither | 7.4 |
| F12 | **Command during emergency / debug / hold** | must not cut power | install without arming; existing restore paths arm the new file (8.3-8.5) |
| F13 | **Old node gets `1D 9D`** | ignored ("Unknown TLV channel/command: 29/157"; IP: "unrecognised code") | 5.5; gating on release; property test against the old decoder |
| F14 | **New node, old API** | nothing changes: no `1D 9D` sent, no `01 0F` sent | `report_enabled` only after a `1D 9D` |
| F15 | **API restart** | desired schedules, batches and (with routing) the ledger are in SQLite; nothing lost | a pending command resumes at the next uplink |
| F16 | **Duplicate or late delivery** | same seq: no-op (01). Same hash, other seq: no-op (00). Older spec after a newer one: rejected stale (02) with seq; without seq, impossible by construction (12, item 4) except across the IP queue, which is kept out | seq; superseding; content hash |
| F17 | **Expired, never delivered** | the unit keeps its old schedule; desired ≠ reported | dashboard "not delivered"; no auto-resend after expiry, so a stale storm schedule can't land days later |
| F18 | **Two operators at once** | lost update | `If-Match` on the desired schedule's version (17.3); 409 with "changed by X at T"; audit |
| F19 | **Gauge emergency while a schedule change is pending** | none: emergency is class U, released first; the install during emergency doesn't arm (8.3); emergency's end arms the new schedule | — |
| F20 | **A neighbourhood partly applies** | mixed schedules | shown per unit; canary; rollback with if-hash (13.5) |
| F21 | **Batch causes uplink or downlink collisions** | lost bitmaps/acks | offset spread, pacing, release cap (14.4) |
| F22 | **Rollback chain** (revert of a revert, temporary over temporary) | surprising target | `previous` is always the last permanent schedule for temporary installs (9.4); the dashboard shows the target before sending |
| F23 | **Manual edit on the unit** | drift from desired | reported as "manual"; not overwritten silently (9.5) |
| F24 | **`state.json` corrupt** | unknown description | rebuilt from the file; schedule unchanged |
| F25 | **AT command clash** (AT+WDOG during a transmit) | one fails | through the daemon's single serial owner; retried; reject 07 only for raises |
| F26 | **A long `.wpi`** (all-day 15 min = 192 states; weekday mask × 96 = 1344) | `runScript.sh` walks states in bash with a subshell per duration; at boot it must finish well before `wait_for_wittypi_schedule`'s 120 s, and `apply_schedule` has a 15 s timeout | the generator merges all-day uniform patterns into one ON/OFF pair (as preset 02); the weekday mask limited to `count ≤ 24`; **assumption:** a 400-state file runs in under 5 s on a Pi 4. Bench check (B11) |
| F27 | **Field unit vs bench unit** | a bench schedule left on a deployed unit, or a field schedule powering off a held bench unit | bench flag on the device; install never sets the shutdown alarm; batch excludes bench units by default |
| F28 | **Unauthorised change** | stranding or drain on purpose | permission `manage_schedules` (17.3); node hard limits; LoRa encrypted and replay-protected; IP needs the device token (routing design F21) |

## 17. API changes

### 17.1 Data model (Alembic migration)

`device_schedules` (desired state and history, one row per change):

| Column | Type | Meaning |
|---|---|---|
| `id` | int PK | |
| `device_id` | str FK | canonical id |
| `batch_id` | int null FK | neighbourhood batch |
| `src` | str | `preset`, `custom`, `revert`, `query` |
| `preset` | int null | |
| `spec` | JSON | the fields of 4.2 |
| `spec_hash` | str(8) | 4.5 |
| `frame_hex` | str | bare `1D 9D ...` |
| `revert_hours` | int | 0 = permanent |
| `version` | int | per-device counter for `If-Match` |
| `state` | str | `pending`, `sent`, `confirmed`, `rejected`, `expired`, `superseded`, `cancelled` |
| `reject_reason` | int null | 6 |
| `warnings_acked` | JSON | which preview warnings the operator confirmed |
| `created_by`, `created_at`, `confirmed_at`, `expires_at` | | |
| `command_id` | int null | routing ledger row, once routing exists |
| `prev_reported_hash` | str null | the unit's report before this change (for rollback, 13.5) |

`schedule_batches`: `id`, `neighborhood_id`, `spec`, `spread_s`, `rollout`
(`canary`/`all`), `canary_device_id`, `stage`, `state`, `created_by`,
`created_at`, `rolled_back_at`.

Reported state per device (in `node_state.json`, section `schedule`, like the
other observations): `src`, `hash`, `flags`, `reason`, `next_wake`, `tzo`,
`wdog_h` (from `WDOG:`), `reported_at`, `via`.

`neighborhoods` gains `timezone` (IANA name, null = the server's `SITE_TZ`
setting) for the previews.

The preset table and the spec functions (validation, slots, `max_gap`, hash,
energy, `wdog_needed`, frame encoding) live in a new `app/schedules.py`. It is pure
and tested against the same vectors as the node (20.1).

### 17.2 Routes

| Route | Permission | What |
|---|---|---|
| `GET /schedule-presets` | `read_dashboard` | the table, with per-preset energy and next wakes |
| `POST /schedules/preview` | `read_dashboard` | body: spec or preset, `device_id` or `neighborhood_id`, `revert_hours`, `spread_s`; returns next 10 wakes (UTC and local), wakes/day, `max_gap`, `wdog_needed`, energy (low/high/pessimistic, autonomy), errors (same reasons as the node), warnings (17.4) |
| `GET /devices/{id}/schedule` | `read_dashboard` | desired, reported, drift, history (last 20) |
| `PUT /devices/{id}/schedule` | `manage_schedules` | set (preset or custom, `revert_hours`, `ack_warnings`); header `If-Match: <version>` |
| `POST /devices/{id}/schedule/revert` | `manage_schedules` | `{target: previous|baseline|default}` |
| `POST /devices/{id}/schedule/query` | `send_downlink` | ask the unit for its report (held until heard) |
| `POST /neighborhoods/{id}/schedule` | `manage_schedules` | create a batch (13) |
| `GET /neighborhoods/{id}/schedule` | `read_dashboard` | current batch and per-unit status |
| `POST /neighborhoods/{id}/schedule/rollback` | `manage_schedules` | 13.5 |
| `DELETE /devices/{id}/commands/{command_id}` | `send_downlink` | cancel (routing design 9.3); before routing, `DELETE /devices/{id}/schedule/pending` |

Each response carries `state`, `reason`, `expires_at`, and (with routing)
`command_id`, `route`.

### 17.3 Permission, concurrency and audit

- New permission **`manage_schedules`**, granted to the admin role by the
  migration, not to the roles that only have `send_downlink`. Changing a
  schedule can strand a unit or drain it, which is more than a threshold change.
- `PUT` needs `If-Match` with the current `version`; a mismatch returns 409 with
  who changed it and when. A batch takes a per-neighbourhood lock; a second batch
  for the same neighbourhood supersedes the first per device and is shown as
  such.
- Every change, batch, revert and rollback writes `audit_logs`
  (`app/auth.py` `audit_log`), action `schedule.set` / `schedule.batch` /
  `schedule.revert` / `schedule.rollback`, with the spec, hash, warnings
  acknowledged and the previous reported hash.

### 17.4 Preview warnings

- Energy level amber/red (15.2).
- Wakes in darkness: from the neighbourhood's (or device's) position and a solar
  position calculation, count wakes per day with the sun below the horizon at the
  winter and summer solstice. Positions stay in the database and never in
  responses that leave the server.
- `max_gap > ALERT_SILENT_HOURS`: shown, since alerts become schedule-aware (17.6).
- `max_gap > ip_fallback_after_hours` on a unit that used IP fallback in the
  last 7 days (6.2).
- `on` shorter than the unit's last measured wake + 3 min (from Pi uplink times),
  or `shutdown_iteration_limit × photo_interval` on its reported values.
- The unit has no mDot (no watchdog), or a manual watchdog value below
  `wdog_needed`.
- The unit is clock unverified or its `tzo` differs from the neighbourhood's.
- Errors (blocking) are the node's reject reasons, computed the same way, so the
  operator sees a rejection before sending.

### 17.5 Delivery

As 11 (routing) or 12 (before routing). Pacing (14.4) is a small addition to the
release path, keyed by gateway id (the uplink's best `rxInfo` gateway) and
neighbourhood.

### 17.6 Alerts and expectations

- `check_silent` uses a per-device threshold:
  `max(ALERT_SILENT_HOURS, reported max_gap + 4 h)`. Unknown schedule: as today.
- New alert `schedule_first_wake`: after a confirmed change, no Pi uplink within
  30 min of the first expected wake → alert once. That is the early warning for
  F1 on units where the watchdog would take 20+ h.
- New alert `schedule_rollback`: a report with flag bit 3 from a probation
  rollback.
- Expected-wake tracking: per device, the next expected wake from the reported
  hash; the dashboard shows "late by N min" when a wake is missed, and flags a
  consistent skew of over 10 min (F8).

## 18. Dashboard changes (`templates/dashboard.html`)

### 18.1 Device panel: Schedule card

Replaces the "Monitoring Frequency" select (10):

- **In force (reported):** name ("Daylight, every 2 h", or "Custom", or
  "Manual on unit"), the next wakes today in local time, wakes/day, ON, last
  report time, flags as plain words ("on probation", "reverts to Daylight at 18:00
  on 22 Oct", "watchdog 20 h").
- **Desired** if different: state ("held since 14:02: sent at the next uplink",
  "rejected: OFF time under 5 minutes", "expired 27 Oct: not delivered"), with
  cancel.
- **Change:** preset dropdown or custom editor (first wake in local time,
  converted to UTC with the date's offset shown; interval; number of wakes or
  "all day"; ON; days; temporary: revert after N hours). A live preview: the
  next 10 wakes, a 24 h strip showing wakes against daylight, the energy level with
  its numbers, warnings with confirm boxes, errors that disable "Send".
- **Revert:** previous / deployment baseline / default, each with the schedule
  it would restore.

### 18.2 Drift

When reported ≠ desired after the command is final (manual edit, rollback,
expiry), the card says so and offers "make desired = reported" or "send desired
again". Nothing is resent automatically.

### 18.3 Neighbourhood view

A "Schedule" tab per neighbourhood: the batch spec and preview, spread and
rollout options, and a table with one row per unit: unit, offset, desired,
reported, state, last heard, next expected wake, watchdog, warnings. Plus a
summary ("7 of 10 confirmed, 2 held: not heard since 06:10, 1 rejected"),
"Rollback", and for canary batches, "Continue to all units".

### 18.4 Confirmations and accessibility

Sending needs an explicit confirmation naming the units and the energy level;
red-level and neighbourhood sends need typing the neighbourhood name. Controls
follow the existing WCAG 2.2 AA work: labelled inputs, status in text not only
colour, the preview strip with a text equivalent.

### 18.5 Schedule editor and saved schedules

A dashboard editor to define a schedule and apply it to selected units or
neighbourhoods, so an operator can match local daylight hours (4.6).

- **Editing in local time.** The operator enters the window in the
  neighbourhood's time zone; the API converts it to the UTC custom spec (4.2).
  A schedule applied across zones shows its times in each zone, with a warning.
- **Daylight shown while editing.** A 24 h bar per neighbourhood with the
  site's sunrise and sunset (from the devices' stored positions) for today, the
  next solstice and the next DST change, the wakes marked on it. Wakes in
  darkness and uncovered daylight are visible at once.
- **Live checks.** The node's validation rules (6), the energy guard (15) and
  the preview warnings (17.4) run in the editor; a schedule the node would
  reject can't be saved.
- **Saved schedules are a library on the server**, named and versioned. They
  are custom specs, so a new one needs no node release (unlike a preset) and
  goes out as the 13 B custom op. A saved version is never edited in place:
  editing creates a new version, and units move to it only when the operator
  applies it as a batch.
- **Applying:** to a neighbourhood, or to any selection of units, through the
  batch path of 13 (canary for permanent schedules, pacing, offset spread,
  per-unit progress, rollback). The device panel shows the saved schedule's
  name when the unit's reported hash matches it (offset aside).
- **Drift warning** (4.6) on units whose schedule no longer fits the season.
- **Limit:** one daily window per schedule, as in the spec. Two windows a day
  would need a spec extension (later).
- API: a `saved_schedules` table (id, name, version, spec without offset,
  created_by, created_at, archived) and `neighborhoods.timezone`, in the
  migration of 17.1. Permission `manage_schedules` (Q15) covers both.

## 19. Deploy order and compatibility

| Combination | Behaviour |
|---|---|
| Old node, new API | the API doesn't offer schedules for releases without the feature (reported `01 09`); if sent anyway, ignored (5.5) |
| New node, old API | nothing sent, nothing reported; manual installs work as today; `12 92` still accepted (and logged as doing nothing) |
| New node, new API, no routing | bare frames, reduced hold hook, content confirmation (12) |
| New node, new API, routing | kind 05, wrapper, seq, ack (11) |
| Mixed fleet | per device, by reported release |
| API rolled back | desired schedules unused; nodes keep what they have; temporary schedules still revert on the node |
| Node rolled back | the installed `schedule.wpi` stays and works (it is an ordinary file); reports stop; `ensure_next_wake`/`recovery_boot` keep using it |

Order:

1. **Node release A**: `tools/schedule_spec.py`, `tools/schedule_control.py`
   (install, reconcile, revert, probation), the `1D 9D` parser, `01 0F`, status
   `sc`/`tzo`, the daemon's `pi_watchdog` action, the lock in `ensure_next_wake`,
   `apply_emergency_schedule`'s fallback through the new generator, the `12 92`
   log line, a reconcile service ordered before ticktalk. The `@GRAPHify` graph
   is untouched (installs run from `tools/` in the daemon and in
   `apply_downlink_command`). **Assumption:** no pickle rebuild needed; if any
   graph function changes, rebuild with
   `python compile.py ticktalk_main.py --out output` and commit both pickle files.
   Harmless against today's API.
2. **API phase 1** (read only): `app/schedules.py`, presets, preview, the
   Schedule card showing reported state, schedule-aware silent alerts. No sending.
3. **API phase 2**: per-device sending behind `SCHEDULE_CONTROL=1`, with the
   reduced hold hook (or the router if it exists), permission, audit; the
   monitoring-frequency control removed.
4. **API phase 3**: neighbourhood batches, offsets, pacing, canary, rollback.
5. With the routing design's phase C: kind 05 in the ledger, wrapped frames.

## 20. Testing plan

### 20.1 Shared test vectors

A JSON file of specs → expected validity and reason, slot list (UTC epoch values
over a period), `max_gap`, `wdog_needed`, hash, frame bytes, energy, with the
same file in both repos (copied, and a test that fails if the copies differ in a
checkout of both). It includes month ends (day-of-month alarms), the 2026 and
2027 DST changes (US and one southern-hemisphere zone), windows crossing UTC
midnight, weekday masks, `0x8N`, offsets 0 and 255.

### 20.2 Node unit tests

- Spec validation: every rule, each boundary.
- Generator round trip: spec → `.wpi` → `recovery_boot.parse_schedule` →
  `next_on_start` reproduces the spec's slots for 400 days.
- **Differential test against `runScript.sh`:** run the real upstream script in
  a sandbox with a stub `utilities.sh` (`set_startup_time` and `set_shutdown_time`
  echo their arguments, `current_timestamp` returns a fixed time, `TZ` set) for
  generated files at many "now" values. Assert its startup equals our
  `next_on_start` (or the slot after, where it skips: 3.2).
- `install` with fault injection after each step (7.4), on a temp directory and
  stubbed I2C: the next boot's reconcile ends in "old" or "new", never neither,
  and a next wake is always armed.
- Emergency, debug session and hold flag: arming skipped or startup-only; the
  shutdown alarm never written.
- Revert: due at boot and at shutdown; chain rules.
- Probation: a boot after an off-time longer than `max_gap` + 1 h → rollback,
  whatever woke the unit; normal boots → probation counts down.
- Old-decoder property test: random valid frames through a frozen copy of the
  pre-feature `decode` and `apply_downlink_command` → no handler called (5.5).
- Parser: length mismatches, unknown ops, `1D 9D` concatenated with other codes →
  rejected.
- `01 0F` encoding and the 11 B limit; status fields at the end of the list.

### 20.3 API unit tests

- `app/schedules.py` against the shared vectors.
- Preview warnings and errors; local-time conversion across DST.
- Desired-state transitions, `If-Match`, superseding (and deleting the IP row
  before routing), expiry, rollback with if-hash.
- Batch offsets, canary stage transitions, pacing (no two schedule downlinks
  within 3 s per gateway; release cap).
- Report decoding (`01 0F` alone, with `01 0E`, at 9 and 15 B) and status `sc`.
- Silent-alert threshold from the reported schedule.

### 20.4 Bench tests on the test unit (and a second bench unit for B6)

Follow the bench units' safe-testing procedure. Tests that need the unit to
really power off (B4, B5) are run attended, with the "never power off" stand-in
removed only for the test and restored afterwards.

| # | Test | Pass |
|---|---|---|
| B1 | Preset and custom install over LoRa mid-wake | ack + report; registers 27-30 show the new slot; wittyPi.log shows both lines; the next wake at the new time; `ensure_next_wake` at shutdown "left as it is" |
| B2 | Install during emergency mode | shutdown alarm stays cleared; flag bit 1; ending emergency arms the new schedule |
| B3 | Install during a remote debug session | the session isn't cut; stop arms the new schedule |
| B4 | **Deliberately bad schedule**, proving the nets: with validation bypassed (a bench-only `--unsafe` flag), install a file with no slot ahead and set `AT+WDOG=1` | the unit stays off; the watchdog presses after 1 h; reconcile sees the off-time is longer than `max_gap` + 1 h and rolls back; the report has flag bit 3; the API raises `schedule_first_wake` before that. Then the same with `!` instead of waiting |
| B5 | **Power cut mid-install**: a pause hook (`SCHEDULE_INSTALL_PAUSE_AT=<step>`) and pull the supply at steps 5, 7a, 7b, 8 | every case boots with old or new, a next wake armed, reconcile reporting correctly |
| B6 | **Two-unit simultaneity**: same spec, offset 0, for 2 days (≥ 10 wakes); then offset 30 s on one | alarm skew from wittyPi.log/powerlog boot lines ≤ 2 s; boot-to-first-uplink skew measured (server receive times); with the offset, uplinks 30 ± a few s apart |
| B7 | Install over IP (IP unit) | ack/report over IP; queue row handling |
| B8 | Old release receives `1D 9D` | "Unknown TLV channel/command: 29/157"; no change; no reply |
| B9 | Temporary schedule with `revert` 2 h | reverts at the first shutdown after 2 h; report |
| B10 | Energy: hourly all-day for 24 h with the powerlog | daily energy within the estimate's range; refines 15.1 |
| B11 | Long file: a weekday-mask schedule with `count` 24 (336 states) | `runScript.sh` at boot under 5 s; `wait_for_wittypi_schedule` well within 120 s |
| B12 | `AT+WDOG` through the daemon during transmits | read back right; no lost uplinks |

### 20.5 Integration (no hardware)

API with a fake ChirpStack that loops downlinks into an in-process node decoder
(stubbed I2C and daemon), covering: hold → release on uplink → install → report →
confirmed; a batch of 20 simulated units with pacing; API restart mid-batch.

## 21. Rollout plan

1. Node release A on the test unit, then the second bench unit; B1-B12.
2. API phases 1-2. The test unit gets a custom schedule from the dashboard, then
   reverts, for a week of normal use.
3. One field unit, canary: preset 00 sent to a unit already on it (a no-op install,
   which proves the path and the report), then a small change (offset 10 s), for a
   week.
4. Field units on the release report their schedules (query), so drift
   (manual files, wrong zones, `tzo`) is found before anything is changed.
5. API phase 3 for one neighbourhood: a temporary storm schedule with
   `revert` 24 h, canary off, measuring collisions (bitmaps received vs sent),
   then a permanent change with canary.
6. Then generally. The routing design's kind 05 follows its own phase C.

## 22. Decisions (review 2026-10-17)

1. **UTC on the wire**, local time on the dashboard, slots fixed in UTC: yes.
2. **Presets:** 00-02 as shipped (00 the default), plus two storm presets now:
   03 hourly by day, 04 every 30 min by day (4.4).
3. **Limits:** interval ≥ 15 min, ON 10-60 min, OFF ≥ 5 min, `count` ≤ 96: OK.
   Presets use ON 15, which also suits units relying on cellular.
4. **Longest gap: 16 h for now** (V6), not 72 h. Nothing uses longer gaps. The
   watchdog then stays at 20 h, and the silent alert and the routing window
   are unchanged. Multi-day `days` values stay in the wire format for later.
5. **Watchdog floor 20 h:** keep (moot under the 16 h cap).
6. **Temporary over temporary:** revert to the last permanent schedule.
7. **Energy guard** 4 / 8 Wh/day and the node's 48 permanent wakes/day: OK; the
   node limit is local only (not settable remotely). Off-state figures to be
   updated after the off-state measurement with an mDot fitted.
8. **RTC in UTC:** not now.
9. **Clock unverified:** install and flag.
10. **`13 93`:** retire it the same way as `12 92`.
11. **Spread** `min(120, 10 × N)` s; canary on for permanent, off for temporary:
    OK.
12. **Battery-aware guard:** parked until the V50 battery reading exists.
13. **Expiry** 7 days, no automatic resend: OK (the routing design's per-wake
    IP fallback covers units on IP).
14. **Codes:** reserve `1D 9D`, `01 0F` and kind `05`, kept in one list with the
    routing design's `1F 9F` and `01 0E`.
15. **Permission:** a separate `manage_schedules`, admin only by default.
16. **Probation:** 3 scheduled wakes; rollback triggered by the off-time at
    boot (9.2), not the mDot's press count; plus a server alert on a missed first
    wake.

Added 2026-10-17:

17. **Time zones:** units keep a local system zone (US Eastern units stay on
    `America/New_York`, local file and folder names); neighbourhoods get a time
    zone in the API (4.6).
18. **Schedule editor:** saved, versioned schedules defined in local time with
    daylight shown, applied to units or neighbourhoods as batches (18.5). A
    node-side sunrise/sunset type is later, if needed.

## 23. Where the code differs from what the plan assumed

- **`monitoring_frequency` / `12 92` changes nothing about wakes.** It is stored
  and reported (`09 39`, IP parts, status), and its callbacks only print. The
  dashboard control and the routing design's failure mode 2 and Q6 (72 h
  schedules via `monitoring_freq_h`) assume otherwise. `13 93` emergency frequency
  is the same. `emergency_workflow`, which would set it to 5, isn't in the graph.
- **"Remote debug clears the schedule"**: it clears only the **shutdown** alarm,
  and leaves `schedule.wpi` and the startup alarm alone. `stop()` re-arms from the
  installed file. That makes "install now, armed when the session ends" simple.
- **`runScript.sh` is unsafe for arming a new schedule mid-wake**: it sets a
  shutdown alarm, which would end an emergency, a debug session or a held bench
  unit, and skips a slot when run in an OFF state. Install therefore arms the
  startup alarm only, as `ensure_next_wake` does.
- **Presets as shipped files depend on the unit's time zone** (`BEGIN` is local),
  and no documented setup step sets the zone. Hence presets as UTC specs and the
  `tzo` report.
- **`watercam_10minutes_per_hour.wpi`'s header** says "5 minutes, in every 20
  minutes"; the file is ON 10, OFF 50.
- **Log wording:** `docs/WITTYPI_SCHEDULING.md` says upstream `runScript.sh` logs
  "Schedule next startup at:" with one space and ours with two. The upstream
  script read here (`e833c8d`) uses **two** spaces too (its shutdown line has one).
  The parsers use `\s+`, so nothing breaks, but the log can't tell who armed a
  wake from the spacing. **Assumption:** units may have a different upstream copy;
  check one unit's `runScript.sh`. The new "Schedule install:" line (7.3) gives an
  unambiguous marker either way.
- **`witty_pi_4.generate_schedule`** (the fallback in
  `apply_emergency_schedule` when no file exists) writes non-atomically, defaults
  to eight 30-min slots from 08:00 local, not the deployment schedule, and leaves
  a 2-min OFF state. It is replaced by the new generator installing preset 00.
- **The mDot watchdog has no link to the schedule today**: the 20 h default
  matches only the daylight preset's 16 h gap. Nothing raises it for a sparser
  schedule.
- **`ALERT_SILENT_HOURS` is global (20 h)** and the routing design's window
  ceiling is 48 h. Both must become schedule-aware before gaps over about 16 h are
  allowed.
- **Uplinks are unconfirmed and a bitmap counts as delivered once the mDot sent
  it**, so collisions from simultaneous wakes silently lose bitmaps until the next
  refresh (24 h). That makes the offset spread more than a nicety for
  neighbourhoods of 10+ units.
- **`15 95`, the next pair in the code sequence, is not free in practice.** It was
  a legacy code that units on software from before SU-WaterCam #186 still decode
  with its old meaning. Hence `1D 9D`, a pair never used in any repo (5.1), traced
  through both the current and the pre-#186 decoder (5.5).
- **The mDot's press count can't detect a rescue.** `presses` resets as soon as
  the Pi has power, so the node always reads 0 at boot. Probation uses the
  off-time instead (9.2).
- **Checked on the test unit:** Witty Pi firmware revision 0x07 (register 12;
  0x26 in register 0 is the board ID), zone America/New_York, and its
  `runScript.sh` logs "Schedule next startup at:" with two spaces.
- **`DOWNLINK_CODE_LENGTHS` has only fixed widths.** `1D 9D` is the first
  length-prefixed code; the new parser needs one special case (5.6).
