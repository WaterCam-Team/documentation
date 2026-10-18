# Command routing by observed transport capability: LoRa primary, IP fallback

Status: **design for review, nothing implemented.** No code in any repo changes
until this is agreed.

Code read for this document: API `main` at `60c541e` (#163), SU-WaterCam
`main` at `7e17c4f` (#193), mDot-AT-firmware `main` at `2ad3ce2d` (#10). File
references are repo-relative; line numbers are approximate and only there to
help find things. Statements marked **Assumption** could not be checked in the
code and need a bench check or a captured payload.

---

## Contents

1. Problem statement
2. Goals and non-goals
3. How commands are routed today
4. The capability model
5. Command classes
6. Routing decision table
7. Holding a command until the next uplink
8. Fallback timing
9. Expiry, superseding, cancellation
10. Sequence numbers: wire format and compatibility
11. Downlink size and data rate
12. Node changes (SU-WaterCam)
13. API changes
14. Dashboard changes
15. Deploy order and compatibility
16. Failure modes and their consequences
17. Testing plan
18. Rollout plan
19. Decisions (review 2026-10-17)
20. Where the code differs from the agreed direction

---

## 1. Problem statement

Each device row has one `transport_type` (`lora` or `ip`, column
`devices.transport_type`, `app/models_db.py`, default `lora`). Every command
route reads that one value to choose between ChirpStack (LoRa) and the IP
downlink queue (`downlink_queue` table, which a node only reads when it polls).

The value is set automatically: `POST /ip/uplink` (`app/routes.py`, `ip_uplink`)
sets `transport_type = "ip"` for any device that sends a routine IP uplink.
Only an upload flagged `ip_fallback` from a device with a `lora_eui` is exempt.
Nothing ever sets it back to `lora`.

**The incident.** A bench unit with both LoRa and IP sent routine IP uplinks
during an IP test phase, so it was flipped to `ip`. It then went back to
LoRa-first operation with routine IP upload off. From then on, emergency,
configuration and remote-debug commands for it went into the IP queue. The node
never polled (it polls only with `ip_upload.enabled` true, or on a rare fallback
upload), so the commands were never delivered and the dashboard just said
"queued". Remote start (`!`) and the watchdog (`1B 9B HH`) were refused with 422
("IP devices can't be woken remotely"), although the unit had a working mDot.
API #163 made `ip-upload` go over LoRa for any unit with an EUI, which fixed that
one command. The underlying problem is still there: the route depends on a
stored mode that drifts away from what the unit can do now.

**Fleet policy (from the user):** every unit uploads over LoRa whenever it can.
IP is a fallback, and routine only during an IP test phase or on a unit with
marginal LoRa coverage, which still prefers LoRa but in practice relies on WiFi
or cellular. Commands should follow the same policy: LoRa first, IP when LoRa
isn't working, decided from what the server has actually seen recently.

## 2. Goals and non-goals

Goals

- Route each command by **observed capability**, meaning what the server has
  recently heard from the unit over each transport, not by a stored mode.
- LoRa is the primary route. The IP queue is the fallback, used only when the
  LoRa attempt isn't confirmed and the unit is polling IP. A command never goes
  out on both routes at once.
- Don't spend a `!` wake (a full extra boot) on commands that can wait. Hold
  them and send them when the unit is next heard over LoRa, because it is awake
  then.
- Commands expire, so a stale command can't land days later.
- A copy that arrives late or twice must not be applied twice, or after a newer
  command of the same kind. This is what the 1-byte per-kind sequence numbers
  are for.
- Old nodes keep working with the new API, and new nodes with the old API.
- The dashboard shows the derived state (last heard on each transport), which
  route each command took, what is held, and why.
- Keep a manual override, using the existing `transport_type` column.
- Record RSSI, SNR and data rate per LoRa uplink, and fix the ChirpStack v4
  data-rate extraction.

Non-goals

- Sending both routes at once ("belt and braces"). This was explicitly ruled out.
- Changing the mDot firmware. `!` and `1B 9B` stay exactly as they are (a
  1-byte and an exact 3-byte frame).
- Downlink over cellular SMS, or any IP push. IP stays poll-only.
- LoRaWAN confirmed downlinks (`confirmed=True` in `send_downlink`). Their ack
  is a MAC-layer frame from the mDot, which says nothing about whether the Pi
  applied the command. Application-level confirmation stays.
- Changing the stream-gauge or vendor-gauge paths.

## 3. How commands are routed today

| Route (API `app/routes.py`) | Payload | When `transport_type == "ip"` | Otherwise (LoRa) | Confirmation |
|---|---|---|---|---|
| `POST /devices/{id}/emergency` (`device_emergency_command`) | `21 91 HH` on, `99 99` off (`app/chirpstack.py`, `emergency_on_bytes`, `EMERGENCY_OFF`) | IP queue (`queue_downlink`) | `node_delivery.deliver_in_background(kind="emergency")`; `!` wake only for "on" (`wake=enable and wake`) | status JSON `em` = 1 / 0 (`note_status`) |
| `POST /downlink?send=true` (`api_downlink_encode`) | `encode_downlink` frame: `10 90`, `11 91`, `12 92`, `13 93`, `14 94`, `16 96`, `17 97`, concatenated | IP queue | `node_delivery`, kind `config`, `wake` defaults to **true** | any status JSON after the send (`lambda st: True`) |
| `POST /devices/{id}/remote-debug` | `18 98`/`18 99`/`18 9A NN` (`app/remote_debug.py`) | IP queue | `remote_debug.start_lora` (node_delivery under the same slots); wake only for start | status JSON `rd` |
| `POST /devices/{id}/remote-start` | `!` (0x21) | **422** "IP devices can't be woken remotely" | `send_remote_start`, fire and forget | none |
| `POST /devices/{id}/watchdog` | `1B 9B HH` | **422** "LoRa units only" | `mdot_watchdog.start_lora` (node_delivery, never wakes) | mDot `WDOG:<h>` uplink |
| `POST /devices/{id}/ip-upload` | `17 97 NN` | LoRa if the device has a `lora_eui` (since #163); IP queue only without an EUI | `ip_fallback.request_lora` (node_delivery, `!` wake when `wake`) | next IP uplink, or `ipr` in the status |
| `gauge_scheduler._dispatch` (gauge triggers) | `21 91 00` and/or `!` | IP queue for emergency; remote start skipped | node_delivery for emergency (wake), raw `send_downlink` for `!` | as emergency |
| `weather_scheduler` | `start_signal_hex` (default `21`) | not consulted: sends to listed `dev_eui`s | raw `send_downlink` | none |
| `ip_fallback.check_auto` (automatic `17 97 01`) | `17 97 01` | not consulted: needs an EUI | node_delivery with `wake=True` | as ip-upload |
| `POST /ip/uplink` (`ip_uplink`) | — | **sets `transport_type = "ip"`** unless `ip_fallback` and the device has an EUI | — | — |
| `GET /ip/downlink/{id}`, `POST /ip/uplink` with `want_command` | — | `pending_command` → `get_pending_downlink`: the oldest undelivered row, atomically marked delivered | — | being handed over counts as delivered; nothing confirms it was applied |
| `POST /ip/device/{id}/transport` | — | sets `lora` / `ip` (manage_devices) | — | — |

How LoRa delivery works now (`app/node_delivery.py`, `deliver`):

1. Send the command (Class C, so the mDot gets it at once if listening).
2. Poll the stored last status every 5 s for up to `FIRST_ANSWER_S` = 60 s.
3. No answer and `wake` set: send `!`, wait `BOOT_WAIT_S` = 75 s, then resend
   every `WAKE_RESEND_S` = 35 s until `WAKE_WINDOW_S` = 240 s after the `!`.
   That is at most 7 downlinks and about 5 minutes in all.
4. `MAX_CONCURRENT` = 3 deliveries at a time; the rest show "queued".
5. Progress is kept per `device_id|kind` in `node_state.json` (`app/state_store.py`,
   section `progress`). A delivery that was running when the API stopped reads
   as `no_answer`, "API restarted during delivery".

What the node does with a LoRa downlink (SU-WaterCam
`tools/lora_handler_concurrent.py`, `decode`): it is forwarded by the mDot as
`PACKET:PORT=..,SIZE=..,DATA=<HEX>` only while the Pi is up, since the mDot's
`forwardPacketToSerial` writes to serial regardless and a powered-off Pi drops
it. `decode` tries these in order: bare `21` (logged), then
`parse_downlink_frame` (`tools/transmit_ip.py`, the `DOWNLINK_CODE_LENGTHS`
table), then the TLV parser `[ch][cmd][len][value]`, then an ASCII "digits"
parser `[2 chars][2 chars][rest]`. Applying `21 91`, `99 99`, `18 9x` or a
config change triggers `_reply_with_status`, the compact status JSON
(`tools/lora_debug_integration.py`), which is the API's confirmation.

The mDot (`CommandTerminal/CmdClassCPacketProcessor.cpp`,
`processIncomingPacket`) handles exactly two frames itself, Pi on or off:
`isEmergencyPacket` (exactly one byte `0x21`: pulse the Witty Pi switch if PA_6
is low, then send `EMERGENCY_ACK:IN=..,OUT=..`) and `pi_watchdog_downlink`
(exactly 3 bytes `1B 9B HH`: set the watchdog and answer `WDOG:<h>`). Everything
else is forwarded to serial.

Problems this shows, beyond the incident:

- The route is decided by a single stored mode that a routine IP uplink
  changes, and nothing changes it back.
- IP queue rows never expire and can't be cancelled. A command queued for a
  unit that never polls waits forever, and is then applied whenever the unit
  does poll, possibly weeks later and after newer commands.
- Delivered over IP means "handed to a poll", not "applied".
- Config and IP-upload requests use a `!` wake by default, which costs a full
  boot cycle for commands that could wait for the next scheduled wake.
- At DR0 the node can't send its status reply (`{"dbg":1,"em":1}` is already
  16 B, the DR0 uplink limit is 11 B; see `tools/lora_debug_integration.py` and
  API `app/ip_fallback.py`). Every LoRa delivery to a DR0 unit looks
  unconfirmed, wakes it with `!`, and ends `no_answer` although the command was
  applied.

## 4. The capability model

### 4.1 Observations (per canonical device id)

All are kept in a new `link` section of `node_state.json`. Section 13.1 covers
the choice between this file and SQLite.

| Field | Set when | Notes |
|---|---|---|
| `lora_at` | any LoRa uplink for the device (ChirpStack webhook or live stream), including mDot-only ones (`WDOG:`, `WD:`, `EMERGENCY_ACK`) | shows the radio link works |
| `lora_pi_at` | a LoRa uplink that came from the Pi: sensor, bitmap, status JSON, ack. Not `mdot_watchdog` or `EMERGENCY_ACK` | means the Pi is awake now and can take a command |
| `wakes` | ring of up to 128 "wake starts": a `lora_pi_at` more than 20 min after the previous Pi uplink; trimmed to 7 days | for the window (4.3) |
| `lora_rssi`, `lora_snr`, `lora_dr`, `lora_gw_n` | each LoRa uplink: best gateway's RSSI/SNR, the data rate, and how many gateways heard it | link quality; also written to Influx per uplink (13.8) |
| `ip_poll_at` | a **routine** IP contact that can carry a command: `GET /ip/downlink/{id}`, or `POST /ip/uplink` with `want_command` and without `ip_fallback` | today nothing records polls |
| `ip_fallback_at` | `POST /ip/uplink` with `ip_fallback: true` | shown on the dashboard; not used for routing (4.2) |
| `ip_wakes` | same ring as `wakes`, from `ip_poll_at` | window for IP |
| `unconfirmed` | consecutive LoRa deliveries that were sent while the Pi was evidently awake and not confirmed (4.4) | reset by any confirmed LoRa delivery |
| `doubtful_since` | when `unconfirmed` reached 3 | cleared with `unconfirmed` |
| `seq_capable` | the node's software version is at least the first release with sequence numbers, or it has sent an ack (10.6) | gates the sequenced format |
| `seq` | the API's next sequence number per kind | 10.5 |

The existing `ip_fallback` section already keeps `lora_at`, `ip_at` and
`bitmap_at` per device (`app/ip_fallback.py`, `note_uplink`, called from
`process_uplink_data` and `ip_uplink`). The new hook goes in the same two places
and the new section takes over the timestamps. `ip_fallback.ip_at` keeps its
meaning (any IP uplink), because its confirmation logic uses it.

### 4.2 Definitions

Let `now` be server time. Every timestamp is the server's receive time; the
node's clock is never used, so a wrong node RTC doesn't matter (see 16).

- **LoRa-active**: the device has a `lora_eui`, and `now - lora_at <= W_lora`.
  A unit with an EUI that has never been heard over LoRa (no `lora_at`) is
  **not** LoRa-active. An IP-only unit needs no override; it falls out of
  this naturally.
- **Pi-awake-on-LoRa**: `now - lora_pi_at <= 10 min`. Used only for timing
  sends (7) and counting doubtful (4.4), never to choose the route.
- **LoRa doubtful**: LoRa-active and `unconfirmed >= 3`, meaning three LoRa
  deliveries in a row went out while the unit's uplinks kept arriving, and none
  was confirmed. Uplinks work and downlinks apparently don't (gateway TX, RX2
  settings, mDot Class C off, antenna).
- **IP-active**: `now - ip_poll_at <= W_ip`. Only routine polls count, because
  routing to the IP queue only makes sense if the node will poll again on its
  normal schedule. A fallback upload (`ip_fallback: true`) shows the unit can
  reach the server, but it happens at most about twice a day while LoRa is
  failing, and never otherwise. It does pick up a queued command
  (`want_command` is always set, SU-WaterCam `ticktalk_main.py`
  `ip_uplink_transmit`), so a queued command may still arrive that way as a
  bonus. Decided in Q2.
- Only **authenticated** IP requests count (`device_auth.token_matches` passed),
  once the device has a token. Otherwise a spoofed uplink could make a unit look
  IP-active and pull its commands into a queue the attacker then drains.
  Strict from the start (Q8): `IP_AUTH` is `enforce`, so a contact without a
  valid token doesn't count.

### 4.3 Windows

Units run daylight schedules: wakes every 2 h by day, an overnight gap of about
14-16 h, and emergency mode waking every 2-10 min (`ALLOWED_EF_MIN`, and it
skips shutdown). The window must stay open across a normal night, or every unit
would count as not LoRa-active each morning and get its commands routed to IP.
It must also close soon enough that a unit which really lost LoRa gets its
commands over IP the same day.

Proposed formula (per device, per transport):

```
gaps        = differences between consecutive wake starts in the last 7 days
daily_max   = for each calendar day (server local) with >= 1 wake, its longest gap
normal_gap  = median(daily_max)            # needs >= 3 days with wakes
interval    = median(gaps)
W           = clamp(1.5 * normal_gap,
                    low  = max(3 * interval, 6 h),
                    high = 48 h)
W           = 26 h  while there are < 3 days of history
```

Why each part:

- **1.5 × the normal gap.** A unit that misses one scheduled wake (a brownout,
  a deferred recovery boot: SU-WaterCam `docs/WITTYPI_SCHEDULING.md` §6 lists
  a deferral of at least 2 h) is still LoRa-active. A unit that misses a whole
  night plus a morning is not. With a 15 h overnight gap that gives 22.5 h.
- **Median of daily maxima, not the longest gap.** The agreed direction said
  "longest normal gap". A single 3-day outage in the last 7 days would make the
  longest gap 72 h and the window 108 h: a dead unit would stay "LoRa-active"
  for 4.5 days. The median of daily maxima ignores up to three bad days.
- **Default 26 h.** About 1.6 × a 16 h night, plus slack. For comparison:
  `ALERT_SILENT_HOURS` defaults to 20 ("longer than a unit's normal overnight
  gap", `app/alerts.py`), and the mDot's watchdog defaults to 20 h ("longer than
  a daylight schedule's overnight gap", mDot README). 26 h sits above both, so a
  unit counts as "silent" and has had its watchdog press before it stops
  counting as LoRa-active.
- **Floor of 3 wake intervals, at least 6 h.** A unit on a 24 h schedule
  (2-hourly around the clock) has a normal gap of 2 h, which gives 3 h. The
  floor of 6 h stops a short ChirpStack or gateway hiccup from flipping routing.
- **Ceiling 48 h.** Keeps the window bounded on a unit with a very sparse
  schedule. No unit uses a gap over 32 h (the 72 h monitoring option is
  unused), so no special handling is needed (Q6).
- **Emergency mode** makes the interval short and the unit awake. The normal
  gap is still the overnight one from the days before, so the window doesn't
  shrink suddenly when emergency mode starts.

`W_ip` uses the same formula on `ip_wakes`. A unit that polls IP every wake has
the same gaps.

The window is recomputed when a wake start is added (cheap: at most 128
numbers). The dashboard shows the value used and how it was derived ("26 h,
default: less than 3 days of history" or "23 h = 1.5 × 15.3 h overnight gap").

### 4.4 Counting "doubtful"

A LoRa delivery attempt counts toward `unconfirmed` only when all of these hold:

- it was sent while the Pi was awake on LoRa (a Pi uplink within 10 min before
  the send, or one during the attempt's confirmation window). An attempt to a
  sleeping unit that didn't answer isn't evidence against the downlink path;
- the node can confirm at its current data rate. A seq-capable node always can
  (its ack is 6 B, 10.4). A legacy node can only when `lora_dr >= 1`, because
  its status JSON needs at least 16 B and DR0 carries 11 B;
- the command is a Pi command. mDot commands are confirmed by the mDot itself,
  so a missing `WDOG:` does count, but only once per watchdog request.

Clearing: any confirmed LoRa delivery resets `unconfirmed` to 0 and clears
`doubtful_since`. If the unit is doubtful, all its Pi commands go to IP, so
nothing would ever clear it. So the router **probes**: on a Pi uplink, at most
once every 6 h while doubtful, it sends a ping (seq-capable nodes: the empty
sequenced frame `1F 9F 02 00 SS`, 10.3) or the legacy debug status request
`5001` (ASCII digits, `tools/lora_debug_integration.py`; only when
`lora_dr >= 1`). An answer clears doubtful.

## 5. Command classes

| Class | Commands | Who acts on it | Urgency | `!` wake |
|---|---|---|---|---|
| **M**: mDot-handled | `!` remote start (Power On), `1B 9B HH` watchdog | mDot, Pi on or off | send now | `!` is itself the command; watchdog never wakes |
| **U**: urgent node command | emergency on `21 91 HH`, emergency off `99 99`, remote debug start `18 98/99/9A NN` and stop `18 9x 00` | Pi | send now | only for emergency **on** and remote debug **start** (today's rule, kept) |
| **N**: non-urgent node command | configuration (`10 90` ... `16 96`), IP upload request `17 97 NN` (dashboard and automatic rule) | Pi | hold until the unit is heard | never |

Notes

- "Power On" was listed both as urgent with IP fallback and as mDot-handled. It
  is mDot-handled: `!` is acted on by the mDot, and a sleeping Pi has no IP
  connection, so it has **no IP fallback**. It is LoRa only.
- Emergency **off** and remote debug **stop** don't get a `!`. A unit in
  emergency mode or in a debug session is awake: `call_shutdown` returns early
  while `emergency_mode` is on, and the session keeps the unit up. If it doesn't
  answer, it is most likely asleep, which means emergency already ended (its
  time limit) or the session timed out. The command is then held until the next
  uplink (N-style) and not woken. This matches today's code
  (`wake=enable and wake`; `wake and not ending`).
- Automatic `17 97 01` (`ip_fallback.check_auto`) already fires on a LoRa uplink
  (the node is awake) but passes `wake=True`. It becomes class N with no wake.
- `17 97` is also a field of `/downlink` configuration (`DownlinkCmd.ip_upload_wakes`).
  The router splits it into its own `ip_upload` command, because sequence numbers
  are per kind (10) and an IP upload request must not be superseded by a later
  threshold change.

## 6. Routing decision table

Override = the `transport_type` column, now with three values (13.2):
`auto` (default: this table), `lora` (force LoRa: never IP), `ip` (force IP).

States (with `auto`):

| # | LoRa-active | LoRa doubtful | IP-active | Typical unit |
|---|---|---|---|---|
| S1 | yes | no | yes | dual unit in an IP test phase |
| S2 | yes | no | no | normal field unit |
| S3 | yes | yes | yes | downlink broken, IP polling |
| S4 | yes | yes | no | downlink broken, no IP |
| S5 | no | — | yes | IP-only unit, or LoRa gone with IP working |
| S6 | no (has EUI) | — | no | silent unit |
| S7 | no EUI | — | no | IP-only unit that stopped polling |

Actions:

| State | M (`!`, `1B 9B`) | U (emergency, remote debug) | N (config, `17 97`) |
|---|---|---|---|
| S1 | LoRa now | LoRa now (wake per 5) → not confirmed after the delivery (~5 min) → **IP queue** | **hold** → send on next Pi uplink → not confirmed after 3 uplinks, or unit stops being LoRa-active → **IP queue** |
| S2 | LoRa now | LoRa now (wake) → not confirmed → **hold**: resend on next Pi uplink, no further `!`, until expiry | **hold** → send on next Pi uplink → retry on later uplinks until expiry |
| S3 | LoRa now, warn "downlinks doubtful" | **IP queue** directly | **IP queue** directly |
| S4 | LoRa now, warn | LoRa now (no `!`: the downlink path is the suspect, and a `!` the mDot did hear would only waste a boot) → hold, retry on Pi uplinks | hold → send on next Pi uplink (probe-like) |
| S5 | EUI: LoRa now, warn "not heard over LoRa for X h"; no EUI: 422 | **IP queue** | **IP queue** |
| S6 | LoRa now, warn | LoRa now with `!` (the only chance of reaching it now) → not confirmed → hold; released by the first contact on either transport | hold; released by the first contact on either transport |
| S7 | 422 (no EUI) | hold; released at the next IP poll (= queued) | same |

With overrides:

| Override | M | U | N |
|---|---|---|---|
| `lora` | LoRa now | as S2 (never IP) | as S2 |
| `ip` | 422 "this unit is set to IP only" | IP queue | IP queue |

Rules that apply in every state:

- **One route at a time.** A command has at most one live copy: in flight over
  LoRa, held, or in the IP queue. Moving it to the IP queue ends LoRa attempts.
  A held command released to LoRa is not also queued.
- **Late LoRa confirmation wins.** If a LoRa confirmation for a command arrives
  after the command was moved to the IP queue and before the node polled, the
  IP row is cancelled. If the node already polled it, the seq makes the second
  copy a no-op on new nodes (10). On old nodes it is applied twice (16, "duplicate
  delivery").
- **IP queue → LoRa is not automatic.** Once a command is in the IP queue it
  stays there until polled, expired, superseded or cancelled. Moving back would
  bring "both at once" back in through the side door.
- **Release on first contact (S6/S7).** A held command in a "no route" state
  goes out over whichever transport is heard first. That is LoRa for a Pi uplink
  and IP for a poll, where the router creates the queue row just before
  `pending_command` runs, so the same poll gets it.
- A poll from a LoRa-active, non-doubtful unit does **not** release held N
  commands to IP (policy: LoRa first), with one exception (Q3, decided):
  **per-wake fallback.** When an IP-active unit polls, any N command that was
  not confirmed over LoRa in that wake (none was sent because LoRa heard
  nothing, or it was sent and not confirmed) is handed over in that poll. A
  dual unit, and especially one with marginal LoRa, then gets each command
  within one wake either way. Duplicates are covered by the seq on new nodes;
  on old nodes a repeated config is harmless and a repeated `17 97` costs at
  most a few extra uploads. No extra cellular data: the poll rides on the
  wake's routine upload.

## 7. Holding a command until the next uplink

### 7.1 Why the API holds it, not ChirpStack

ChirpStack's device queue is not a hold: for a Class C device it schedules a
queued item as a Class C downlink as soon as the gateway is free, whether or not
the Pi is up. The mDot then forwards it to a powered-off Pi, which drops it.
That is exactly today's problem. So the API keeps held commands and only calls
`send_downlink` when it has evidence that the Pi is awake.

**Assumption:** ChirpStack v4 sends Class C queue items immediately and may
also use the RX1/RX2 window of an uplink that arrives first. Either way it
doesn't wait for the Pi. To be checked against the deployed ChirpStack.

### 7.2 Where the hook goes

`app/chirpstack.py`, `process_uplink_data`, next to the existing
`ip_fallback.note_uplink(...)` call. That spot already resolves the canonical
`device_id` (`resolve_device_id`), knows `format_detected`, and excludes
`mdot_watchdog`. The hook:

```
routing.on_lora_uplink(device_id, dev_eui,
                       from_pi = format_detected not in ("mdot_watchdog",) and not EMERGENCY_ACK,
                       rssi, snr, dr, gw_n, f_cnt)
```

1. Updates the `link` observations (4.1). This always happens, from both the
   webhook and the live stream.
2. If `from_pi`, and the device has held commands whose route is LoRa now
   (state S1, S2, S4, S6, or override `lora`), schedules a **release** on a
   background thread. The webhook must return fast; today's deliveries run on
   threads too.

The hook is idempotent per uplink. ChirpStack may hand the same uplink to the
webhook and to the live stream (`start_live_stream`), so it keeps the last
`fCnt` released per device and ignores a repeat. Only one release runs per
device at a time (a per-device flag under the router's lock).

The IP side has the matching hook, `routing.on_ip_contact(device_id, kind)`, in
`ip_uplink` (before `pending_command`) and `ip_downlink_poll`. It updates
`ip_poll_at` / `ip_fallback_at` and does the S6/S7 "first contact" release
(6).

### 7.3 Release timing (Class C)

- The uplink that triggers a release is often the first of a burst: the sensor
  packet, then the bitmap packets (SU-WaterCam `ticktalk_main.py`, sensor packet
  then bitmap). The mDot is half-duplex. While it transmits the next packet it
  can't hear the downlink. **Assumption:** after each uplink the mDot opens
  RX1/RX2 and then returns to continuous Class C receive (RXC), so a downlink
  sent between packets is heard. To be measured on the bench.
- So the release waits `RELEASE_DELAY_S` = 5 s after the uplink, sends, and
  waits up to 60 s for the confirmation, as `FIRST_ANSWER_S` does today. No
  answer: the command goes back to held, `attempts` + 1. It is sent again on the
  next Pi uplink in the same wake (at most 3 attempts per wake) or in a later
  wake.
- **Risk: the last uplink of a wake.** If the trigger is the wake's last uplink
  and the Pi halts within seconds, the downlink is lost. The next wake retries,
  which costs one wake interval of delay and no energy. **Assumption:** the
  first Pi uplink of a wake comes several minutes before shutdown (capture,
  segmentation and bitmap upload follow it). Releasing on the *first* Pi uplink
  of a wake (the one that starts a `wakes` entry) makes this case rare.
  Optional node change (Q9): stay up 30 s after the last LoRa uplink.

### 7.4 Several held commands

- At most one held command **per kind** per device. A new command of a kind
  supersedes the held one (9.2). Configuration merges: the held config's fields
  are overlaid with the new command's fields, and the result is one command.
- Release order on one uplink: emergency, remote debug, configuration, IP
  upload. Within a kind there is only one command. They go out one at a time:
  send, wait for confirmation (or 60 s), next. Each downlink takes about 0.3 s
  of gateway airtime (11), so 4 kinds is under 2 s of airtime.
- Ordering across kinds doesn't matter for correctness, because each kind has
  its own seq space and touches different settings. Emergency goes first because
  it changes the shutdown behaviour.
- A held command whose release is in progress (`sending`) can't be cancelled
  from the dashboard. The cancel waits for the attempt to finish (at most 60 s)
  and then applies.

### 7.5 The deferred command's lifecycle

```
created ─► held ─(Pi uplink)─► sending ─(confirmed)─► confirmed
              ▲                    │
              └──(no answer)───────┘
held / sending ─(fallback rule, 8)─► ip_queued ─(poll)─► ip_delivered ─(ack)─► confirmed
any non-final ─► expired | superseded | cancelled
```

Urgent commands start in `sending` (the existing node_delivery sequence)
instead of `held`.

## 8. Fallback timing

| Class | Primary attempt | Falls back to IP when | Worst-case IP pickup |
|---|---|---|---|
| U, S1 | node_delivery as today: 60 s, `!`, resends until 240 s after the `!`: about 5 min | the delivery ends unconfirmed **and** the unit is IP-active at that moment | the node's next poll: the end-of-cycle check (`tools/last_downlink_check.py`, before shutdown) if it is awake, else its next wake (2 h by day, up to the overnight gap) |
| U, S2/S4/S6 | as above | never directly. Held, and retried on each Pi uplink. If the unit becomes IP-active before expiry (state changes to S1/S3/S5), the next router tick moves it to IP | — |
| N, S1 | send on next Pi uplink | (0) the unit polls IP at the end of a wake in which it wasn't confirmed over LoRa (per-wake fallback, 6); else (a) 3 attempts in awake wakes unconfirmed (the unit is then also counted toward doubtful), or (b) the unit stops being LoRa-active (`now - lora_at > W_lora`): no LoRa uplink within the window | as above |
| N, S2 | as above | never; held until expiry | — |
| any, S3/S5 | — | immediately (IP is the route) | as above |

Rule (b) means the fallback comes at most `W_lora` after the last LoRa uplink,
not `W_lora` after the command was created. For a command created at the end
of a quiet night, that is sooner.

A **router tick** (background thread, every 60 s, like `downlink_scheduler`)
does the time-based transitions: fallback rule (b), expiry, state changes that
move held commands to IP, and the doubtful probe schedule. It is idempotent and
works only from the ledger (13.1), so after a restart it simply carries on.

## 9. Expiry, superseding, cancellation

### 9.1 Expiry

`expires_at` is set when the command is created. After it, the command is
`expired` wherever it is: held, or an IP row not yet polled (the IP queue query
filters on it, 13.5). A command already handed to a poll or sent over LoRa
can't be recalled. Its effect is covered by the seq and by the node's own limits.

| Command | Expires after | Why |
|---|---|---|
| Configuration | 7 days | settings don't age much; matches the node's `REQUEST_MAX_AGE_S` (7 days) for `17 97` |
| Emergency on | `HH` hours if `HH > 0`, else the node's `emergency_max_hours` (last reported, else 24, the node's default in `ticktalk_main.py`), whichever is shorter | after that the node would have ended it itself (`call_shutdown`'s time limit); turning it on later would be a different decision |
| Emergency off | the same deadline as the "on" it ends, else 24 h | after that emergency has ended anyway |
| Remote debug start | 1 h | an operator waiting at a terminal; later is useless and costs SIM data |
| Remote debug stop | 4 h (the longest session, `MAX_MINUTES` = 240) | after that the session has timed out |
| IP upload request (`17 97 NN`, `NN > 0`) | 48 h | the automatic rule re-asks after `IP_FALLBACK_COOLDOWN_HOURS` (24) anyway |
| IP upload cancel (`17 97 00`) | 48 h | same |
| `!`, `1B 9B` | not held, so no expiry; watchdog delivery as today | — |

Note: `21 91 HH` with `HH > 0` also **stores** `HH` as the node's new
`emergency_max_hours` (`_start_emergency` and `apply_downlink_command` both call
`set_param_fn("emergency_max_hours", ...)`). So the server can't assume the
node's default stays 24 after any timed emergency. It should use the last value
the node reported. Today the status JSON doesn't carry it; see 12.4.

An expired command raises a dashboard notice. For emergency on it also raises an
alert (13.7), because an emergency that never arrived matters.

### 9.2 Superseding

A new command of a kind supersedes every non-final command of the same kind for
the device: held, `sending` (after its current attempt), or `ip_queued` and not
yet polled (the row is cancelled). This alone prevents most out-of-order
problems on old nodes: an "on" left in the IP queue can't be picked up after a
later "off".

Configuration merges: fields from the superseded command not set in the new
one are carried over, so "area 30%" followed by "stage 1.2 m" ends as one
command with both.

### 9.3 Cancellation

Dashboard and API: `DELETE /devices/{id}/commands/{command_id}` for held or
`ip_queued` (not yet polled) commands. Permission `send_downlink`, the same as
creating it. For an `ip_queued` command the row's `cancelled_at` is set in one
UPDATE guarded by `delivered_at IS NULL`, so a cancel racing a poll either wins
or reports "already picked up".

## 10. Sequence numbers: wire format and compatibility

### 10.1 What they solve

With "primary, then fallback" a command can reach a node twice. The LoRa copy
may have been applied without the confirmation reaching the server (status lost
at DR0, gateway deaf for a moment), and then the IP copy is applied again. Or a
node can get copies out of order. The per-command effects of a duplicate today:

| Code | Effect of a duplicate on today's node |
|---|---|
| `21 91 HH` | restarts the emergency duration (`emergency_since = now`) |
| `99 99` | none |
| `18 98 NN` | extends the session to NN×10 min from now (`remote_debug.start`, "extend the running session") |
| `17 97 NN` | ignored only within `REPEAT_WINDOW_S` = 15 min (`tools/ip_fallback.py`, sized for the ~6 min wake + resend sequence); later it **restarts the count**, so NN more uploads at about 90 KB of SIM data each |
| config | none, unless a newer config arrived in between (then the old one reverts it) |

An IP fallback copy can arrive hours after the LoRa copy (the next poll), well
outside the 15 min window. The worst ordering case is emergency "on" applied
after a later "off".

### 10.2 Why not a trailing seq byte

The obvious format, the command with one extra byte at the end, is **not safe**
on today's nodes. A node that doesn't know the format must ignore the frame, but
today's `decode` falls through its parsers, and some of them accept it with a
different meaning. Traced through `tools/lora_handler_concurrent.py` `decode` and
`tools/transmit_ip.py` `parse_downlink_frame`:

| Frame (seq `SS` appended) | `parse_downlink_frame` | TLV parser | digits parser | Result on an old node |
|---|---|---|---|---|
| `21 91 06 25` | None (1 byte left over) | `vlen` = 6, truncated → None | channel `21`, command `91`, value `0625` → `_start_emergency("0625")` | **emergency on for 168 h** (0x625 clamped), and `emergency_max_hours` saved as 168 |
| `21 91 01 25` | None | `(0x21, 0x91, [0x25])` exactly fills the frame | — | **emergency on for 37 h** (0x25) |
| `18 98 06 03` | None | `vlen` = 6, truncated | `18`/`98`, value `0603` → `minutes_from_code` = 0x603×10 | **session of 240 min** (clamped) instead of 60 |
| `99 99 26` | None | `vlen` = 0x26, truncated | `99`/`99`: no branch | ignored: **emergency stays on** |
| `17 97 02 0A` | None | `vlen` = 2, truncated | no branch | ignored |
| `10 90 1E 11` | None | `vlen` = 0x1E, truncated | `10`/`90` doesn't match `10`/`9` | ignored |

So a trailing byte can misparse into a longer emergency or session on an old
node.

### 10.3 The sequenced frame: a TLV wrapper

```
1F 9F  LL  KK  SS  <legacy frame>
│  │   │   │   │   └─ the unchanged command bytes (one kind only)
│  │   │   │   └───── sequence number, 0-255, per kind
│  │   │   └───────── kind: 00 ping, 01 emergency, 02 remote debug, 03 IP upload, 04 configuration
│  │   └───────────── length of everything after LL (2 + len(legacy frame))
└──┴───────────────── new code pair 1F/9F (the NN, NN|0x80 convention; unused in the node table, the API encoders and the mDot)
```

Why this shape:

- **Old nodes ignore it, deterministically.** `1F 9F` isn't in
  `DOWNLINK_CODE_LENGTHS`, so `parse_downlink_frame` returns None. The TLV
  parser then reads `ch=0x1F, cmd=0x9F, len=LL`, and because `LL` is exactly the
  rest of the frame it parses one complete TLV and returns. `_apply_command_tlv`
  formats the bytes as decimal (`f"{0x1F:02d}"` = `31`, `f"{0x9F:02d}"` = `159`),
  matches no branch, and prints "Unknown TLV channel/command". `decode` returns
  after a successful TLV parse, so the digits parser never sees it. Nothing is
  applied and no status is sent. This holds for every inner frame, because the
  wrapper's own length byte makes the TLV parse succeed whatever follows.
- **The mDot forwards it.** It is neither exactly one byte `0x21` nor exactly
  3 bytes starting `1B 9B`.
- **The kind is explicit.** The node doesn't have to infer it, the ack can name
  it, and a ping (kind 00, empty inner frame) is possible.
- The inner frame is the legacy frame, so the node reuses
  `parse_downlink_frame` and `_apply_server_frame` unchanged.

Byte-level examples:

| Command | Legacy | Sequenced | Size |
|---|---|---|---|
| Emergency on 6 h, seq 37 | `21 91 06` | `1F 9F 05 01 25 21 91 06` | 8 B |
| Emergency on, node default, seq 38 | `21 91 00` | `1F 9F 05 01 26 21 91 00` | 8 B |
| Emergency off, seq 39 | `99 99` | `1F 9F 04 01 27 99 99` | 7 B |
| Remote debug 60 min cellular, seq 3 | `18 98 06` | `1F 9F 05 02 03 18 98 06` | 8 B |
| Remote debug end (WiFi code), seq 4 | `18 99 00` | `1F 9F 05 02 04 18 99 00` | 8 B |
| IP upload, 2 wakes, seq 10 | `17 97 02` | `1F 9F 05 03 0A 17 97 02` | 8 B |
| Config: area 30 %, stage 125 cm, seq 17 | `10 90 1E 11 91 00 7D` | `1F 9F 09 04 11 10 90 1E 11 91 00 7D` | 12 B |
| Config, all six settings | 19 B | 23 B | 23 B |
| Ping, seq 200 | — | `1F 9F 02 00 C8` | 5 B |

Trace of the first row on an old node: `decode("1F9F050125219106")` → not
`21` → `parse_downlink_frame`: code `1F 9F` unknown → None → TLV: `(0x1F, 0x9F,
len 5)`, value `01 25 21 91 06`, offset 8 = length → one TLV →
`_apply_command_tlv` → "Unknown TLV channel/command: 31/159" → return.

New-node parsing rules: the frame must be exactly `5 + (LL - 2)` bytes. `KK`
must be known. The inner frame must parse with `parse_downlink_frame` and
contain only codes of kind `KK` (ping: empty). Anything else is rejected with
ack result 3 (below). Not applied.

### 10.4 The ack uplink

The status JSON can't go out at DR0, so confirmation needs a compact binary
uplink. Proposed: a new uplink channel **`01 0E` command_ack**, 4 value bytes,
sent as its own LoRa packet:

```
01 0E  KK  SS  RR  EM
            │   │   └─ emergency mode now, 0/1 (so emergency on/off is confirmed at DR0 too)
            │   └───── result: 00 applied, 01 duplicate (already applied), 02 stale (SS = the node's last), 03 rejected
            └───────── the seq received (or the node's last, for 02)
```

6 bytes, which fits DR0's 11 B. Example, emergency seq 37 applied:
`01 0E 01 25 00 01`. `01 0E` is the next free code after `01 0D`
(`clock_unverified`) in the API's `CHANNEL_REGISTRY` (`app/decoders.py`).

After the ack, the node still sends the status JSON when it fits the current
size limit (`_reply_with_status`, unchanged), so the dashboard's `rd`, `ipr`
and so on stay current.

The node sends `01 0E` only in reply to a sequenced frame. An old API never
sends one, so it never receives an ack it can't decode.

**Assumption:** the API's LoRa decoder (`process_uplink_data`, the `tlv_only`
path) decodes a packet consisting only of a registered `01 xx` channel. To be
checked with a unit test before the node sends it.

Over IP, the ack goes to a new endpoint `POST /ip/downlink/{device_id}/ack`
`{queue_id, kind, seq, result, em}` (device token, like the poll). An old API
answers 404, and the node logs it and carries on.

### 10.5 Allocation and comparison

- The API keeps `next_seq[device][kind]` in `node_state.json` (`link.seq`) and
  assigns it when a command is **created**. Resends, wake resends, held retries
  and the IP fallback copy all carry the **same** seq. That is the point.
- The node keeps `last[kind]` (seq, time) and compares with serial arithmetic
  (RFC 1982 style):
  `d = (SS - last) mod 256`: `d == 0` duplicate, `1 <= d <= 127` newer (apply and
  store), `128 <= d <= 255` stale (don't apply).
- No stored value for a kind, or a stored value older than 30 days: accept and
  store. This covers a fresh SD card, and a counter that has long since moved on.
- **Resync.** If the API's counter is lost (state file gone) and restarts low,
  the node may answer `02 stale` with its own last seq. The API then sets
  `next_seq = last + 1`, issues the command again under the new number (same
  ledger entry, `attempts` + 1) and logs it. So losing API state costs one
  round trip, not a stuck unit.
- Per-kind counters, not one per device: with a single counter, a kind unused
  for 128 commands of other kinds would look stale (`d >= 128`) and its next
  command would be ignored.
- Ping (kind 00) is never stored or compared; the node always acks it with
  `RR = 00`.

### 10.6 Over IP

The IP row keeps the **legacy** `hex_payload` and `parts`, and gains top-level
fields in what `pending_command` returns: `"seq": 37, "kind": 1, "command_id": ...`.
Today's node (`apply_downlink_command`) reads only `parts[].code` and
`parts[].payload_hex` and ignores other keys, so an old IP node applies the
command as it does now, without dedupe. A new node checks the seq first, using
the same store and rules as LoRa, then applies, then posts the ack.

### 10.7 Who gets sequenced frames

The API sends the sequenced format only to devices marked `seq_capable`: the
device's last reported `software_version` (uplink channel `01 09`, CalVer,
`app/decoders.py` `decode_software_version`, or `"v"` in the status JSON) is
at least the first release with this feature, or the device has ever sent a
`01 0E` ack. Everyone else gets legacy frames and the old confirmation
predicates. A new node accepts legacy frames forever (manual tools, old API) and
treats them as today, without seq checks.

## 11. Downlink size and data rate

- **The 11-byte DR0 limit is an uplink limit** (US915 DR0 = SF10/125 kHz,
  maximum application payload 11 B; DR1 53, DR2 125, DR3 242; the node's
  `lora_debug_integration.py` gives the same numbers). It limits the **ack**,
  not the command.
- Class C downlinks use the RX2/RXC parameters. For US915 the default is
  923.3 MHz at **DR8** (SF12/500 kHz), maximum application payload **53 B**
  (33 B if repeater-compatible sizes are enforced; up to 15 B of FOpts MAC
  commands can also take room). The largest sequenced command (all six config
  fields) is 23 B, so every command fits at the slowest downlink rate whatever
  the node's uplink DR. **Assumption:** the deployed ChirpStack region config
  uses the default RX2 DR8. `app/node_delivery.py` already assumes DR8 for its
  airtime figures; to be checked in the ChirpStack region file.
- Airtime at DR8 (SF12, 500 kHz, CR 4/5, 8-symbol preamble, explicit header,
  CRC off on downlinks): a 3 B command (16 B PHY payload) takes about 0.29 s,
  and an 8 B sequenced one (21 B PHY) about 0.33 s. So the wrapper costs about
  40 ms of gateway airtime per downlink.
- **The uplink DR matters for confirmation:** at DR0 only the 6 B ack (new
  nodes) or `WDOG:<h>` (mDot, ≤ 8 B) get through. A legacy node at DR0 can't
  confirm anything. Hence 4.4.

### 11.1 Fixing ChirpStack v4 data rate and link metadata

Today (`app/chirpstack.py`, end of `process_uplink_data`):

```python
rssi_val = uplink_data.get("rxInfo", [{}])[0].get("rssi", 0)
snr_val  = uplink_data.get("rxInfo", [{}])[0].get("snr", 0)
"data_rate": uplink_data.get("txInfo", {}).get("dataRate", {}),
```

- `txInfo.dataRate` is a ChirpStack v3 field. In v4 the uplink event has a
  top-level `dr` (integer) and `txInfo.modulation.lora` =
  `{bandwidth, spreadingFactor, codeRate}`. So `data_rate` is always `{}`.
- Only `rxInfo[0]` is used. With several gateways the first isn't necessarily
  the best.
- A missing `rxInfo` gives `rssi = 0`, which looks like an excellent signal.
- The live-stream path builds `uplink_data` from `uplink_event` with only
  `deviceInfo`, `fPort`, `data` and `publishedAt` (`start_live_stream`), so
  stream uplinks have no radio metadata at all. **Assumption:** this branch
  actually runs (it depends on the protobuf message having `uplink_event`); to
  be checked on the server.

Proposed:

```
dr       = uplink_data["dr"] if an int
           else from txInfo.modulation.lora: US915 uplink SF10/125 → 0, SF9 → 1, SF8 → 2, SF7 → 3, SF8/500 → 4
           else None
best     = the rxInfo entry with the highest SNR (ties: higher RSSI)
rssi,snr = best's values, or None (never 0) when rxInfo is missing
gw_n     = len(rxInfo)
sf, bw   = from txInfo.modulation.lora
```

These are stored in the reading (`lora_dr`, `lora_sf`, `lora_rssi`, `lora_snr`,
`lora_gw_n`, as integers, or not written when unknown) and in the `link`
section. The old `data_rate` dict field is dropped. **Assumption:** the exact
v4 JSON layout above; to be confirmed with one captured webhook body
(`GET /chirpstack/raw` returns them) and pinned in a test fixture.

## 12. Node changes (SU-WaterCam)

### 12.1 Parser

- `tools/transmit_ip.py`: add `parse_sequenced_frame(data) -> (kind, seq, parts) | None`
  next to `parse_downlink_frame`, plus a `KIND_CODES` table (kind → allowed
  codes, from `DOWNLINK_CODE_LENGTHS`). `DOWNLINK_CODE_LENGTHS` is unchanged:
  `1F 9F` is **not** added to it, so the legacy parser keeps rejecting the
  wrapper.
- `tools/lora_handler_concurrent.py` `decode`: try `parse_sequenced_frame`
  first, right after the bare-`21` check and before `_parse_server_frame`. On a
  match, `seq_store.check(kind, seq)`:
  - newer: `_apply_server_frame(parts)`, then `seq_store.commit(kind, seq)`,
    then send the ack (`00`), then the status JSON as today;
  - duplicate: don't apply, ack `01`, and also the status JSON. A duplicate
    usually means the server missed the first answer;
  - stale: ack `02` with the stored seq;
  - rejected: ack `03`.
- Commit after applying, not before. A crash between the two then means a
  re-apply on the next copy, which is harmless for these commands, rather than
  a skip.
- `_apply_server_frame` replies with the status JSON per part today (emergency,
  remote debug, config each call `_reply_with_status`). For sequenced frames the
  reply moves to the wrapper handler, so one ack and one status go out per
  command.
- Remote debug start is applied on a thread (`_start_remote_debug`), and
  `remote_debug.start` already answers with a status at once. The ack is sent
  when the session request is accepted, not when Tailscale is up.

### 12.2 Per-kind seq store

`tools/command_seq.py`, state in `data/command_seq.json`:

```json
{"1": {"seq": 37, "at": 1760000000}, "2": {"seq": 4, "at": 1760000000},
 "3": {"seq": 10, "at": 1760000000}, "4": {"seq": 17, "at": 1760000000}}
```

Written atomically under a lock with `tools/config_io.py`, as
`data/ip_fallback.json` is. Both the LoRa daemon (LoRa path) and ticktalk (IP
path) write it, so the check and commit must happen under one lock acquisition
per command. It survives reboots; a missing or corrupt file means "accept
anything" (10.5).

### 12.3 IP path

`apply_downlink_command` gains an optional `seq` / `kind` from the command
dict. When present, it does the same check and commit, and
`handle_downlink_reply` posts the ack to `/ip/downlink/{id}/ack`. That is one
small request on a connection that is already up, and the node ignores a 404
from an old API. `tools/last_downlink_check.py` (up to 3 commands per check)
goes through the same function, so nothing else changes there.

### 12.4 Status fields

Appended to `collect_compact_status` (`tools/lora_debug_integration.py`), after
the existing fields so they are the first to be trimmed at small payload limits
(the ack already carries what confirmation needs):

- `"sq":[37,4,10,17]`: last applied seq per kind 1-4, `null` for none. Only once
  the node has received a sequenced frame, so an old API never sees it.
- `"emh":24`: the node's current `emergency_max_hours`, so the API can compute
  emergency expiry (9.1) after the node's stored default has changed. Low
  priority, near the end.

API side, `DEBUG_FIELD_LABELS` (`app/decoders.py`) gains labels for both.

### 12.5 Repeat windows

| Code | Today | With seq (sequenced frames) | Legacy frames (old API, manual tools) |
|---|---|---|---|
| `21 91 HH` | every copy restarts the duration | a duplicate is a no-op (no restart, no new `emergency_since`); a **new** "on" (new seq) still restarts it, which is the operator's intent | unchanged |
| `18 98/99/9A NN` | every copy extends the session from now | a duplicate is a no-op; a new start extends, as now | unchanged |
| `17 97 NN` | same NN within 15 min = resend, else restarts the count | the seq replaces the window: same seq = no-op at any age; a new seq always sets the count | keep the 15 min `REPEAT_WINDOW_S` |

The 15 min window stays for legacy frames. It is the only protection an old API
gives, and its 15 min was sized for the LoRa wake + resend sequence (about
6 min). It can't cover an IP fallback copy that arrives hours later, which is
why seq is needed.

### 12.6 Not changed on the node

- The mDot handling of `!` and `1B 9B`, and the firmware.
- The IP poll schedule: wake start, after an IP uplink, before shutdown.
- `ip_fallback`'s own rule (`plan`).
- Optional, Q9: wait up to 30 s before halting if a LoRa uplink went out in
  the last 30 s, to catch a held command released by that uplink.

### 12.7 Pickle rebuild

The changes touch `ticktalk_main.py` only through `tools/` imports. If any
`@GRAPHify` function changes, rebuild `output/ticktalk_main.pickle` with
`python compile.py ticktalk_main.py --out output` and commit both pickle files,
as usual. The LoRa daemon runs from source.

## 13. API changes

### 13.1 Data model: a command ledger in SQLite

New table `device_commands` (Alembic migration; the API runs
`_run_alembic_upgrade_head` at start):

| Column | Type | Meaning |
|---|---|---|
| `id` | int PK | command id, returned by every POST |
| `device_id` | str FK | canonical id |
| `kind` | str | `emergency`, `remote_debug`, `ip_upload`, `config`, `remote_start`, `watchdog` |
| `cls` | str | `M`, `U`, `N` |
| `legacy_hex` | str | the legacy frame |
| `parts` | JSON | as today's queue `parts` (with `name`, `hours`, ...) |
| `seq` | int null | per-kind seq (null for M and for non-capable nodes) |
| `state` | str | `held`, `sending`, `ip_queued`, `ip_delivered`, `confirmed`, `expired`, `superseded`, `cancelled`, `failed` |
| `route` | str | `lora`, `ip_queue`, `held` |
| `reason` | str | the routing explanation shown on the dashboard ("held: LoRa-active, sent at the next uplink") |
| `wake` | bool | `!` allowed for this command |
| `woke_at` | datetime null | when a `!` was sent for it |
| `attempts` | int | LoRa attempts |
| `created_at`, `created_by`, `source` | | `dashboard`, `gauge_trigger`, `weather`, `auto` |
| `expires_at` | datetime | 9.1 |
| `sent_at`, `fallback_at`, `delivered_at`, `confirmed_at` | datetime null | |
| `confirmed_via` | str null | `lora_ack`, `lora_status`, `mdot_wdog`, `ip_ack`, `ip_poll` (legacy IP) |
| `queue_id` | int null FK `downlink_queue.id` | the IP copy |
| `superseded_by` | int null | |
| `error` | str null | |

Why SQLite and not `node_state.json`: held commands must survive restarts (the
agreed list names "API restarts losing held commands"). They also need queries
(per device, per state, history) and transactions shared with `downlink_queue`
(cancel the IP row and update the command in one commit). `node_state.json` is
rewritten whole on every change, and the `link` observations, which change on
every uplink, stay there.

`node_delivery`'s `progress` section stays, mirrored from the ledger, so the
dashboard's `pollDelivery` and the `GET /devices/{id}/delivery/{kind}`,
`/emergency`, `/remote-debug`, `/watchdog`, `/ip-upload` status routes keep
working unchanged during the transition.

### 13.2 `transport_type` becomes the override

- Values: `auto` (new default), `lora` (force LoRa), `ip` (force IP).
  `crud.update_device_transport_type`, `POST /ip/device/{id}/transport` and
  `gauge_scheduler._dispatch` (which raises on unknown values) accept `auto`.
- Migration: set **every** existing row to `auto`. Today's values can't be told
  apart: most `ip` values were written by the auto-flip, and `lora` is the
  column default. After the migration the operator sets `ip` on any unit that
  must never use LoRa (Q1). None is planned: a unit with marginal LoRa that
  relies on WiFi or cellular stays on `auto`, so it still prefers LoRa and
  falls back to IP per wake (rule in 6).
- `ip_uplink` no longer writes `transport_type`. **This is the auto-flip
  removal.** `get_or_create_device` still creates unknown devices (with `auto`).
- `GET /devices` keeps `transport_type` (now the override) and adds `link`
  (13.4), so old dashboard code that reads `transport_type` sees `auto` and
  shows "unknown". The new dashboard uses `link`.

### 13.3 The router: `app/routing.py` (new)

- `route(device, command) -> Decision` implements the table in 6 as a pure
  function of (class, override, capability state, EUI, seq_capable). It returns
  the action and a one-line `reason`. It is pure, so the table can be tested
  exhaustively.
- `submit(db, device, command, source)` creates the ledger row, supersedes (9.2),
  assigns the seq, and acts: start a LoRa delivery (node_delivery), hold, or
  queue for IP.
- `on_lora_uplink`, `on_ip_contact`: hooks (7.2).
- `on_ack(device, kind, seq, result, em, via)`: matches the ledger row (same
  kind and seq, non-final), marks it confirmed, cancels its IP row if not polled,
  resets `unconfirmed`; handles `02 stale` resync (10.5).
- `tick()`: every 60 s (8).
- Every command route calls `routing.submit` instead of choosing a transport
  itself: `device_emergency_command`, `api_downlink_encode` (`send=true`),
  `device_remote_debug`, `device_remote_start`, `device_watchdog`,
  `device_ip_upload_request`, `gauge_scheduler._dispatch`,
  `ip_fallback.check_auto`, and `weather_scheduler` (which today sends `!` to a
  raw `dev_eui` list; it would resolve each to a device and submit `!`, class M).
- `chirpstack_emergency` (`POST /chirpstack/emergency`, the raw dev_eui path) is
  left as is, a raw tool. Its docstring already points to the device route.

### 13.4 Changes to `node_delivery`

- `deliver` keeps its sequence. It gains `on_unconfirmed` / `on_confirmed`
  callbacks to the router, so that the router, not the thread, decides the
  fallback.
- `answered` for seq-capable nodes becomes "a `01 0E` ack with this kind and seq
  (any result except 03), or the old predicate". Old predicates stay for legacy
  nodes (`em`, `rd`, any status for config, `ip`/`ipr` for IP upload).
- Config confirmation today is `lambda st: True`: *any* status after the send.
  That status could be an unrelated one (the node's own debug reply). With acks
  this becomes exact; for legacy nodes it stays as is.
- `!` budget: at most one `!` per command, and at most 4 per device per 24 h
  across all commands (gauge triggers and repeated clicks included). Over
  budget, the command is sent without a wake and held, and the dashboard says
  why (Q5).
- No `!` when the unit is doubtful (S4), or when a Pi uplink arrived in the last
  10 min (it is awake, so a missing answer isn't sleep).
- `MAX_CONCURRENT` = 3 is unchanged. Held releases use the same slots; they are
  short (one send, 60 s wait).
- **Restart.** A ledger row in `sending` at start-up: if it is under 10 min old
  and class U, the delivery resumes from "resend" without a new `!` (the first
  may already have booted the Pi). Otherwise it goes back to `held` and the
  router applies its normal rules. This replaces today's "no_answer, API
  restarted during delivery".

### 13.5 IP queue

`downlink_queue` gains `command_id` (FK, null for rows queued by older code),
`expires_at`, `cancelled_at`, `kind`, `seq`.

- `get_pending_downlink` filters
  `delivered_at IS NULL AND cancelled_at IS NULL AND (expires_at IS NULL OR expires_at > now)`,
  in the same atomic `UPDATE ... RETURNING`.
- `pending_command` adds `seq`, `kind`, `command_id` to the returned dict
  (ignored by old nodes, 10.6). It also tells the router the row was handed over
  (ledger → `ip_delivered`; for a legacy node this is the final confirmation,
  `confirmed_via = ip_poll`).
- New `POST /ip/downlink/{device_id}/ack` (device token, `check_device`).
- Rows left from before the migration (no `command_id`, never delivered): the
  migration gives them `expires_at = created_at + 7 days`. Older ones expire at
  once, which stops a weeks-old command landing. The migration logs how many
  (Q1).
- `downlink_scheduler`'s cleanup (30 days after delivery) also removes cancelled
  and expired rows after 30 days.

### 13.6 Routes: request and response

- Every command POST returns `command_id`, `route` (`lora`, `ip_queue`, `held`),
  `state`, `reason`, `expires_at`, and keeps today's fields (`transport`,
  `status`, `queue_id`), so existing scripts keep working. `transport` is
  `lora` for `route` `lora` or `held`, and `ip` for `ip_queue`.
- `remote-start` and `watchdog` no longer return 422 for `transport_type` `auto`
  or `lora`, only for override `ip` and for no EUI. When the unit isn't
  LoRa-active they still send, with a `warning`.
- New `GET /devices/{id}/link`: the capability state (4.1, plus the derived
  booleans, the windows and their derivation).
- New `GET /devices/{id}/commands?active=1` and `?limit=N`: the ledger.
- New `DELETE /devices/{id}/commands/{command_id}`: cancel (9.3).
- `wake` in request bodies stays accepted. It can only *remove* the wake (no
  `!` for this command), never add one to class N.

### 13.7 Alerts (`app/alerts.py`)

New kinds, change-only like the existing ones:

- `lora_downlink`: start when a unit becomes LoRa doubtful ("uplinks arrive,
  3 commands unconfirmed"), end when a probe or command is confirmed.
- `command_expired`: once per expired **emergency** or **remote debug**
  command. Config and IP-upload expiries show on the dashboard only.
- `ip_fallback` (existing) is unchanged.

`ALERT_SILENT_IGNORE` already handles bench units. The new kinds honour it too.

### 13.8 Readings

Per LoRa uplink, `lora_dr`, `lora_sf`, `lora_rssi`, `lora_snr`, `lora_gw_n` go
to Influx with the reading (11.1). The device panel can then plot link quality.
This also tells us whether "DR0 units" are a real population, which decides how
much the 6 B ack matters.

## 14. Dashboard changes (`templates/dashboard.html`)

### 14.1 Transport section (replaces "Current transport")

```
Transport  (override: Auto ▾  [Set])
LoRa   heard 1 h 20 min ago · window 23 h (1.5 × 15.3 h overnight gap) · active
       RSSI −112 dBm · SNR −7 dB · DR2 · 2 gateways
       downlinks: OK  (or: doubtful since 09:12, 3 unconfirmed; next probe at the next uplink)
IP     last routine poll 3 d ago · window 26 h (default) · not active
       last fallback upload 5 h ago
Commands go: LoRa first; IP fallback not available (no routine IP polls)
```

- Override select: Auto / Force LoRa / Force IP, each with one line on what it
  does. Changing it asks for confirmation when held or queued commands exist,
  and says what happens to them: they stay where they are, and the next tick
  re-routes held ones.
- The intro paragraph "A routine IP uplink sets a device to ip" is replaced.
- `renderPanelTransport`'s per-control hints (`footerHint`, `ipUploadHint`,
  `watchdogHint`, `remoteDebugHint`) are derived from `link` instead of
  `transport_type`. For example: "Not heard over LoRa for 30 h: Power On will
  be sent, but may not arrive". Or: "Held: sent when the unit next reports over
  LoRa (normally within 2 h by day)".

### 14.2 Per-command result (`showControlResult`)

- On submit: the route and reason from the response. "Configuration held: sent
  at the next LoRa uplink (last one 1 h ago). Expires in 7 days." "Emergency on:
  sending over LoRa…". "Emergency on: queued for the next IP poll (LoRa
  downlinks doubtful)".
- `pollDelivery` keeps working on `/delivery/{kind}` (13.1). A new
  `pollCommand(command_id)` shows the ledger state: held / sending (attempt n) /
  waking / in IP queue / picked up by the node / confirmed (via LoRa ack, IP ack,
  status) / expired / superseded / cancelled.
- "Confirmed" says how: "the node confirmed it" (ack or status) or "the node
  picked it up over IP" (legacy IP: handed over, not proven applied).

### 14.3 Held and queued commands

A "Pending commands" list in the panel: kind, what (e.g. "emergency on 6 h"),
route, created by / when, expires in, a **Cancel** button (9.3), and the reason
line. Superseded and expired commands stay in a collapsed "recent" list for
24 h, so an operator can see why a command didn't run.

The IP Upload status line (`ipUploadResult`, from `GET /ip-upload`) uses the
ledger instead of searching the last 50 queue rows.

### 14.4 Accessibility

Same patterns as the existing controls: status lines with `role="status"`,
errors with `role="alert"`, Cancel as real buttons with an accessible name
("Cancel held configuration for <unit>"). Re-check with axe as for API #120.

## 15. Deploy order and compatibility

| Combination | Behaviour |
|---|---|
| **Old node, new API** | `seq_capable` false → legacy frames and the old confirmation predicates. Routing, hold, expiry, superseding and the auto-flip removal all apply. Residual risk: a LoRa copy applied but unconfirmed, then the IP copy, is applied twice (effects in 10.1; bounded by superseding and expiry). At DR0 nothing confirms, so these units are excluded from doubtful counting (4.4) |
| **New node, old API** | The old API never sends `1F 9F`, so the node takes the legacy path. No acks, no `sq` (both gated on having received a sequenced frame). Auto-flip and today's routing as before |
| **New node, new API** | Sequenced frames, acks, dedupe |
| **Mixed fleet** | Per-device `seq_capable`. A node updated in the field becomes capable at its next `01 09` report |
| **API rolled back** | `transport_type = auto` would break the old `gauge_scheduler` (`ValueError`), and the old routes treat `auto` as LoRa (`!= "ip"`). Rollback therefore needs a down-migration mapping `auto → lora`. Ledger and new queue columns are ignored by old code; queue rows with `cancelled_at` set would be handed out by old code, so the down-migration deletes undelivered cancelled or expired rows |

Order:

1. **API phase A** (observe only): `link` observations, the DR/RSSI/SNR fix,
   `GET /devices/{id}/link`, the dashboard Transport section showing derived
   state next to the current `transport_type`. No routing change. Run for at
   least a week and compare the computed windows and states with what the units
   actually did.
2. **API phase B** (routing, legacy frames): ledger, router, holds, fallback,
   expiry, superseding, IP queue columns, auto-flip removal, `transport_type`
   migration to `auto`, `!` budget, alerts. Behind `COMMAND_ROUTING=observed`
   (default `legacy` for the first deploy, then switched).
3. **Node release**: parser, seq store, acks, status fields, IP ack. Harmless
   against API phase B, which still sends legacy frames.
4. **API phase C**: sequenced frames for capable nodes, behind `SEQ_FRAMES=1`;
   ack handling, resync, probes.

## 16. Failure modes and their consequences

| # | Failure | Consequence | Mitigation |
|---|---|---|---|
| 1 | **Stale window, unit dead or LoRa gone**: still "LoRa-active" for up to `W_lora` (≤ 48 h) after its last uplink | U commands go to LoRa, use a `!`, then fall back to IP after ~5 min if IP-active (fine). N commands are held until the unit stops being LoRa-active, then go to IP. Without IP they wait until expiry | dashboard shows "last heard"; `silent` alert at 20 h; the window uses the median of daily maxima so it doesn't stretch after outages |
| 2 | **Window too short** (a unit just changed schedule) | a healthy unit counts as not LoRa-active: N commands go to IP (if IP-active) or are held for first contact. No loss, just the wrong route for a while | floor 6 h and ceiling 48 h; a 72 h schedule exceeds the ceiling, so such units need override `lora` (documented; Q6) |
| 3 | **LoRa lost mid-delivery** | U: unconfirmed → IP queue (S1) or held (S2). N: back to held, retried on the next uplink | ledger state; resend uses the same seq |
| 4 | **DR0: replies can't be sent** | legacy nodes: no LoRa confirmation at all, so U deliveries end unconfirmed (and get an IP copy if IP-active: a duplicate, effects in 10.1), and N retries 3 times. New nodes: the 6 B ack fits | 4.4 excludes these from doubtful; 6 B ack; DR is now recorded, so the dashboard can say "DR0: the node can't confirm" |
| 5 | **Downlinks fail, uplinks fine** (gateway TX, RX2 mismatch, `AT+CPROC=0`) | counted as doubtful after 3 → S3/S4 | `lora_downlink` alert; probes every 6 h clear it once fixed |
| 6 | **Doubtful false positive** (the node was busy, e.g. the half-duplex mDot transmitting a bitmap burst) | commands go to IP needlessly, or are held | release delay 5 s and per-wake retries; only awake attempts count; a probe clears it at the next uplink |
| 7 | **Clock skew** | node RTC wrong (`01 0D clock_unverified`): no effect, all routing times are server receive times; seq doesn't use time. Server clock jump: windows and expiry shift by the jump | NTP on the server (assumed) |
| 8 | **Duplicate delivery** (LoRa copy applied, unconfirmed, then IP copy) | new nodes: a no-op (seq). Old nodes: per 10.1, emergency duration restarted, session extended, up to NN extra IP uploads, config harmless | seq; superseding; legacy `17 97` window |
| 9 | **Out-of-order** (an older command arrives after a newer one of the same kind) | new nodes: rejected as stale. Old nodes: prevented for IP copies by superseding (the older row is cancelled before it is polled). A residual race exists only if the older one was polled at the same moment | seq; superseding |
| 10 | **ChirpStack queue backlog** (Class C items not yet sent when resends add more) | a burst of copies later | **Assumption** that this happens: check `DeviceQueue` depth when resending; flush (ChirpStack `FlushQueue`) before each resend of the same command |
| 11 | **Energy: `!` wakes** | each `!` is a full extra boot and capture cycle on battery | class N never wakes; at most one `!` per command, 4 per device per day; no `!` when awake or doubtful |
| 12 | **Gateway half-duplex airtime** | about 0.3 s deaf per downlink; a U delivery is ≤ 7 downlinks (~2.3 s); held releases ≤ 4 per uplink; probes ≤ 1 per 6 h per doubtful unit | `MAX_CONCURRENT` = 3 kept; releases share the slots |
| 13 | **Emergency during an outage** (gauge trigger fires, units unreachable) | U to all units: `!` + resends (concurrency 3, so 10 units ≈ 17 min), then IP for IP-active units; others held until first contact or expiry; `command_expired` alerts | priority: emergency deliveries ahead of others in the slot queue (new: a priority queue in `_deliver_in_turn`) |
| 14 | **IP-only units** | never LoRa-active → IP queue; M commands 422 without an EUI | — |
| 15 | **IP-only unit stops polling** | commands held (S7) until the next poll or expiry | dashboard "waiting for contact" |
| 16 | **API restart** | ledger and queue are in SQLite, so held commands survive; `sending` rows resume (13.4). `link` and `seq` survive in `node_state.json`; lost file → seq resync (10.5), windows fall back to 26 h until 3 days of history | — |
| 17 | **`node_state.json` lost** | windows default, `unconfirmed` reset, seq resync once per kind | — |
| 18 | **Operator confusion** ("why didn't my command go?") | — | every command shows route + reason; held list with expiry and cancel; Transport section shows the evidence |
| 19 | **Operator sets Force IP on a LoRa unit** | commands wait for polls that may never come | the dashboard warns when Force IP is set on a unit that isn't IP-active |
| 20 | **Security, LoRa path** | LoRaWAN frames are encrypted and MIC'd with the session keys, and the downlink frame counter stops replays at the LoRaWAN layer. Seq adds no authentication, but nothing on this path needs it | — |
| 21 | **Security, IP path** | without a token (`IP_AUTH` not `enforce`, or a unit without a token), anyone can poll `/ip/downlink/{id}` and consume commands, or fake uplinks to make a unit look IP-active and draw its commands to the IP queue | only authenticated contacts count toward IP-active once a unit has a token (4.2); finishing the token rollout is a prerequisite for relying on IP fallback (Q8); the ack endpoint requires the token |
| 22 | **Two acks race** (LoRa ack and IP ack for the same command) | first wins, second is a no-op | match on (kind, seq, non-final) |
| 23 | **The node reflashed or got a new SD card** | seq store empty → accepts anything. A late copy of an old command could be applied once | expiry limits how old such a copy can be |
| 24 | **Seq wrap** | 256 per kind; stale needs 128+ commands of one kind in flight, which is impossible with one live command per kind | — |
| 25 | **Webhook and live stream both deliver an uplink** | double observation, which is harmless, but a double release would send twice | release keyed on `fCnt` (7.2) |
| 26 | **The release triggers on the last uplink of a wake** | downlink lost, retried next wake | release on the first uplink of a wake; optional node linger (Q9) |

## 17. Testing plan

### 17.1 API unit tests

- `routing.route`: every class × override × state (S1-S7) × EUI × seq_capable,
  table-driven from section 6, so the table and the code can't drift.
- Per-wake fallback: an IP-active unit's poll hands over N commands not
  confirmed over LoRa in that wake, and none confirmed in it.
- Windows: synthetic wake rings (daylight 2-hourly with a 15 h night, 24 h
  2-hourly, emergency 5 min, one 72 h outage in the 7 days, < 3 days of history)
  → expected `W`.
- Doubtful counting: only awake attempts; legacy at DR0 excluded; reset on
  confirmation; probe scheduling.
- Hold/release: a Pi uplink releases; an mDot uplink (`WDOG:`, `EMERGENCY_ACK`)
  doesn't; the same `fCnt` twice releases once; order of kinds; at most 3
  attempts per wake.
- Fallback with an injected clock (node_delivery already takes `sleep`/`clock`):
  U to IP after the sequence; N to IP when the unit stops being LoRa-active, and
  after 3 attempts.
- Expiry of held, `sending` (after its attempt) and queued rows; the IP poll
  never returns expired or cancelled rows.
- Superseding, including config merge; cancel racing a poll.
- Acks: match, duplicate, stale → resync, rejected; late LoRa ack cancels the IP
  row.
- Restart: ledger rows in each state survive `node_delivery.reset()` /
  `state_store.reset()` and resume correctly.
- ChirpStack metadata: v4 fixtures with `dr`, with only `txInfo.modulation.lora`,
  with several gateways, with no `rxInfo` (→ None, not 0).
- Decoding a LoRa packet that is only `01 0E ...` (10.4 assumption).
- Alembic upgrade and downgrade on a copy of a real database. `transport_type`
  → `auto`; old undelivered queue rows get `expires_at`.
- Existing tests that must keep passing: `test_emergency_transport.py`,
  `test_ip_transport.py`, `test_ip_fallback.py`, `test_remote_debug.py`,
  `test_mdot_watchdog.py`, `test_gauge_dispatch.py`, `test_downlink_cleanup.py`.
  Some assert today's routing (for example 422 for IP devices) and change on
  purpose.

### 17.2 Node unit tests

- `parse_sequenced_frame`: every example in 10.3; wrong `LL`; unknown `KK`; a
  code of the wrong kind; empty ping.
- **Old-parser compatibility golden test**: feed every sequenced example to
  today's `decode` (a copy of the current `LoRaHandler.decode`, or the real one
  with `update_config` mocked) and assert nothing is applied and no status is
  sent. Also pin the trailing-byte misparses in 10.2 as documentation of why the
  wrapper exists.
- Seq store: newer / duplicate / stale, wrap at 255 → 0, a missing file, a
  corrupt file, a 30-day-old entry, and concurrent LoRa + IP writers.
- Ack size ≤ 11 B; the ack is sent only for sequenced frames.
- IP path: `seq` in the command dict honoured; absent → legacy; the ack post
  ignores a 404.
- `17 97` legacy window still works; sequenced duplicates at any age are no-ops.
- Existing: `test_ip_command_handling.py`, `test_ip_fallback.py`,
  `test_last_downlink_check.py`, `test_remote_debug.py`.

### 17.3 Integration (no hardware)

API test client + fake `send_downlink` + a simulated node that runs the real
SU-WaterCam parser on the bytes and produces the ack / status uplink JSON
(ChirpStack v4 shaped). Scenarios: awake unit; sleeping unit (answers only after
`!` + 75 s); deaf downlink (doubtful, then IP); DR0 unit (6 B ack only);
IP-only unit; dual unit in an IP test phase; API restart mid-delivery; old node
(legacy parser) against API phase C.

### 17.4 Bench tests on the test unit

Use the bench procedure that keeps the unit from powering off where a test
needs it awake. Collect the node's LoRa daemon log (grep for `Server command`,
`Unknown TLV`, `ack`, `Emergency`) and `data/command_seq.json`, and the API
ledger (`GET /devices/{id}/commands`) and `link`.

| Test | Steps | Check |
|---|---|---|
| B1 hold and release | unit on its normal schedule; send a config change while it sleeps | ledger `held`; at the next wake released within ~5 s of the first Pi uplink; ack within 60 s; no `!` sent |
| B2 urgent with wake | emergency on 1 h while asleep | command, `!`, boot, ack; `em=1`; 1 `!` in the budget |
| B3 DR0 | force DR0 on the mDot (`AT+TXDR=0`, ADR off) for the test; send emergency off | the 6 B ack arrives; the status JSON is trimmed or not sent; confirmed via `lora_ack` |
| B4 deaf downlink → IP | IP routine polling on; block downlinks (e.g. `AT+CPROC=0` on the mDot for the session, so it doesn't forward; **note this also stops `!`**); send 3 config changes over 3 uplinks | doubtful after 3, alert; next command goes to the IP queue and is applied once; re-enable; the probe clears doubtful |
| B5 duplicate | with downlinks blocked as in B4, send emergency on; let it fall back to IP; re-enable LoRa and replay the same sequenced frame with a test script | node applies once; second copy acked `01 duplicate`; `emergency_since` unchanged |
| B6 out-of-order | emergency on queued for IP (unit not polling); then emergency off over LoRa; then let the unit poll | the "on" row superseded and not handed out; the node stays off |
| B7 API restart | restart the API while a command is held, and once while one is `sending` | held survives; `sending` resumes without a second `!` |
| B8 old node | one unit on the previous release; API phase C | gets legacy frames only; a hand-sent sequenced frame logs "Unknown TLV" and changes nothing |
| B9 Power On without the IP flip | a unit that has sent routine IP uplinks; `POST remote-start` | accepted (no 422); `!` sent |
| B10 expiry | remote debug start while the unit sleeps with no `!` budget left; wait 1 h | expired; alert; never applied at the next wake |

## 18. Rollout plan

1. Review this document; answer section 19.
2. API phase A to production (observe only). For a week, compare the
   dashboard's derived LoRa-active/IP-active against what the units did, the
   computed windows against the real schedules (is 1.5 × the overnight gap
   right?), and the DR distribution (how many DR0 uplinks).
3. Adjust the window constants if needed. Then API phase B with
   `COMMAND_ROUTING=legacy`, check the migration result (`transport_type` all
   `auto`, set overrides), then switch to `observed`. Watch the ledger for a few
   days; `legacy` is the rollback switch.
4. Node release to the test unit; bench tests B1-B10 (B3-B6 need phase C on a
   staging API or with `SEQ_FRAMES=1` for that device only: a per-device
   allow-list). Then the other bench unit. Then, first among field units, the
   unit with marginal LoRa (it is likely at DR0, where only the 6 B ack can
   confirm anything), then the rest at their normal
   update (over WiFi only, per the node update procedure).
5. API phase C (`SEQ_FRAMES=1`) once at least the bench units are capable.
6. Update the docs: SU-WaterCam `docs/IP_TRANSMISSION.md` (the "auto-sets
   transport_type" text, poll semantics), `docs/REMOTE_START_AND_EMERGENCY.md`
   (routing, the wrapper), API `docs/` for the new routes, and the dashboard
   help text.

## 19. Decisions (review 2026-10-17)

1. **Migration:** every device to `auto`; undelivered IP queue rows older than
   7 days expire at migration. No unit gets `ip`: the unit expected to have
   marginal LoRa should still prefer LoRa, relying on WiFi or cellular in
   practice, so it stays `auto` and gets the per-wake fallback (6).
2. **IP fallback uploads** don't count toward IP-active. They still pick up
   whatever is queued.
3. **Wake without LoRa confirmation:** yes, extended to per-wake fallback for
   IP-active units: commands not confirmed over LoRa in a wake go out in that
   wake's IP poll (6, 8).
4. **Emergency off / remote debug stop:** no `!`; held when unanswered.
5. **`!` budget:** 1 per command, 4 per device per 24 h. Gauge-triggered
   emergencies don't count toward it: a flood is never blocked by a budget.
6. **Window formula:** median of daily maxima, clamp 6-48 h. No unit uses gaps
   over 32 h (the 72 h option is unused).
7. **Expiry:** emergency off at the "on" deadline, else 24 h; debug stop 4 h.
8. **Security:** strict now. `IP_AUTH` is `enforce` and the units that use IP
   have tokens, so only authenticated contacts count toward IP-active. Check
   each returning unit's token before it relies on IP.
9. **Node linger:** not now. Release on the first uplink of a wake plus the
   end-of-cycle check should do; revisit if phase A shows misses.
10. **Capability gating:** by the `01 09` software version.
11. **`17 97` split** out of configuration into its own command: yes.
12. **Weather scheduler:** not built yet (planned). When it is, it sends
    through the router (transport checks, `!` budget, expiry), not raw
    `send_downlink`.
13. **Doubtful** after 3, probe every 6 h: yes; tune from phase A data.
14. **Reserved codes** (one list for both designs): `1F 9F` sequenced wrapper
    and `01 0E` ack (this design); `1D 9D` schedule command, `01 0F` schedule
    report and seq kind `05` (schedule control design). Check all against the
    ChirpStack codec and `DOWNLINK_CODE_LENGTHS` before the node work.

Added in review:

- **Marginal LoRa units:** the router shows recent RSSI, SNR and DR per unit,
  and Power On / watchdog (LoRa only) warn "LoRa marginal: Power On may not
  arrive" when the link is poor. A false "doubtful" on such a unit is harmless:
  its commands go over IP, which suits it.
- **Rollout:** that unit gets the node release first among field units (18).

## 20. Where the code differs from the agreed direction

- **Trailing seq byte is unsafe.** Old nodes misparse `21 91 HH SS` (digits or
  TLV path: emergency for 168 h or SS hours) and `18 98 NN SS` (240 min
  session), and ignore `99 99 SS`. Hence the `1F 9F LL KK SS` wrapper, which old
  nodes ignore deterministically (10.2, 10.3).
- **Power On (`!`) has no IP fallback.** It was listed as urgent with IP
  fallback, but the mDot handles it and a sleeping Pi has no IP. It is class M,
  LoRa only.
- **The DR0 11 B limit is on uplinks.** Commands are Class C downlinks at RX2
  (US915 DR8, 53 B), so every command fits, sequenced or not, and config frames
  of up to 22 B (all fields, with `17 97`) already go out today. What DR0 breaks is the **confirmation**:
  the status JSON is at least 16 B, so today every LoRa delivery to a DR0 unit
  ends unconfirmed (and wakes it). A naive "3 unconfirmed = doubtful" would
  flag healthy DR0 units, hence the 6 B ack and the counting rules in 4.4.
- **IP polls aren't recorded today.** `GET /ip/downlink` stores nothing, and
  `ip_fallback.ip_at` counts fallback uploads, which do pick up queued commands.
  IP-active needs a new observation that separates routine polls from fallback
  uploads.
- **"Longest normal gap" is outage-sensitive.** One multi-day outage in the 7
  days would stretch the window to days. The proposal is the median of daily
  maxima.
- **"No LoRa uplink within the window"** is implemented as "the unit stops
  being LoRa-active", which happens at most `W` after the last uplink, so for
  most commands sooner than `W` after creation.
- **Current defaults wake for non-urgent commands.** `/downlink` config,
  `ip-upload` and the automatic `17 97` rule all use `!` today (`wake=True`).
  The agreed class N means removing that.
- **`21 91 HH` changes the node's `emergency_max_hours` permanently**, so
  "emergency_max_hours" can't be assumed to be 24 in the expiry rule; the node
  should report it (`emh`).
- **ChirpStack data rate:** confirmed, `txInfo.dataRate` is always `{}` on v4.
  Also only `rxInfo[0]` is used, a missing `rxInfo` stores RSSI/SNR as 0, and
  the live-stream branch carries no radio metadata at all.
- **ChirpStack's queue can't hold commands** for a Class C device (it sends at
  once), so "send on next uplink" has to be held in the API.
- **Two more senders bypass transport routing:** `weather_scheduler` (raw `!` to
  a `dev_eui` list) and `gauge_scheduler._dispatch` (its own `transport_type`
  branch, which raises on unknown values and so must change with the migration).
- **The auto-flip already spares `ip_fallback` uploads** from units with an EUI.
  The incident came from routine IP uplinks during a test phase, which this
  design stops treating as a mode change.
